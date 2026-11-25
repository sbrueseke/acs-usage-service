# Design

Status: **draft** – design phase, no implementation yet.

This document records the design decisions for acs-usage-service and the
reasoning behind them. Topics that are intentionally postponed are listed
at the end.

## Goal

Replace the built-in Apache CloudStack usage server, which has proven to be
error-prone, with a separate service that:

1. imports raw usage events from CloudStack into its own database,
2. computes usage (e.g. hourly resource runtime) from these raw events, and
3. exposes the results via an API.

Pricing and invoicing are **out of scope**; they are handled by an external
billing system. This project delivers quantities and durations only.

**CloudStack domains are the basis for billing.** Callers must be able to
request usage for a given domain, identified by its CloudStack domain UUID.
A parameter controls whether the result includes all subdomains
recursively (analogous to CloudStack's `isrecursive`).

- The domain UUID is stored with every event during import (see UUID
  resolution).
- The domain hierarchy is not stored; for recursive queries the subdomains
  are resolved at query time via the CloudStack API.

## Why the built-in usage server is not used

The built-in usage server copies events into helper tables and aggregates
them in a periodic batch job. Typical failure modes are jobs stuck on a
lock, helper tables becoming inconsistent after missing or duplicate events,
and no way to self-correct or recompute past periods.

This project keeps the raw events unchanged and derives everything else from
them, so results can always be recomputed.

## Data source

- Target CloudStack version: **4.22.1**.
- Source tables (MySQL, database `cloud`):
  - `usage_event`
  - `usage_event_details` (e.g. `cpuNumber`, `cpuSpeed`, `memory` for
    custom offerings, `vmSnapshotId`, `hidden`)
- Read directly from the **production** database:
  - dedicated database user with `SELECT` privileges on the required
    tables only,
  - short transactions with `READ COMMITTED`, to avoid holding InnoDB
    read views open,
  - statement timeouts.
- The `processed` column belongs to the built-in usage server and is
  neither read nor written by this project.
- Out of scope: network traffic (`user_statistics`), disk I/O
  (`vm_disk_statistics`), and any history of master data (accounts,
  offerings, ...).

## Event types and column semantics

`usage_event` reuses its columns with different meanings depending on the
event type. The following table was derived from
`UsageManagerImpl.createHelperRecord()` and `EventTypes.java` at tag
`4.22.1.0`. `account_id` (account) and `zone_id` (zone) have the same
meaning for all types. Entries marked ⚠ deviate from what the column name
suggests.

| Group | Event types | `resource_id` | `offering_id` | `template_id` | Other |
|---|---|---|---|---|---|
| VM | `VM.CREATE`, `VM.DESTROY`, `VM.START`, `VM.STOP`, `VM.UPGRADE`, `VM.DYNAMIC.SCALE` | VM | service offering | template | `resource_type` = hypervisor; details `cpuNumber`, `cpuSpeed`, `memory` |
| Volume | `VOLUME.CREATE`, `.DELETE`, `.RESIZE`, `.ATTACH`, `.DETACH`, `.UPLOAD` | volume | disk offering | template | `vm_id` = VM, `size` |
| IP address | `NET.IPASSIGN`, `NET.IPRELEASE` | IP address | – | ⚠ flag: is system (0/1) | `resource_name` = IP, ⚠ `size` = source NAT flag; detail `hidden` |
| Template | `TEMPLATE.CREATE`, `.COPY`, `.DELETE` | template | – | source template | `size`, `virtual_size` |
| ISO | `ISO.CREATE`, `.COPY`, `.DELETE` | ISO | – | – | `size` |
| Snapshot | `SNAPSHOT.CREATE`, `.DELETE` | snapshot | – | – | `size` = physical, `virtual_size` = allocated |
| Load balancer | `LB.CREATE`, `LB.DELETE` | LB rule | – | – | |
| Port forwarding | `NET.RULEADD`, `NET.RULEDELETE` | PF rule | – | – | |
| Network offering | `NETWORK.OFFERING.CREATE`, `.DELETE`, `.ASSIGN`, `.REMOVE` | ⚠ VM | network offering | – | ⚠ `resource_name` = NIC id, `size` = is default |
| VPN user | `VPN.USER.ADD`, `.REMOVE` | VPN user | – | – | `resource_name` = user name |
| Security group | `SG.ASSIGN`, `SG.REMOVE` | ⚠ VM | ⚠ security group | – | |
| VM snapshot | `VMSNAPSHOT.CREATE`, `.DELETE` | ⚠ VM | disk offering | ⚠ volume | detail `vmSnapshotId` |
| VM snapshot on primary | `VMSNAPSHOT.ON_PRIMARY`, `.OFF_PRIMARY` | ⚠ VM | – | – | `size`, `virtual_size`; detail `vmSnapshotId` |
| Backup | `BACKUP.OFFERING.ASSIGN`, `BACKUP.OFFERING.BACKUPS.DEL`, `BACKUP.USAGE.METRIC` | ⚠ VM | backup offering | – | METRIC: `size`, `virtual_size` |
| Network | `NETWORK.CREATE`, `.DELETE`, `.UPDATE` | network | network offering | – | `resource_type` = network state (UPDATE) |
| VPC | `VPC.CREATE`, `VPC.DELETE` | VPC | to be verified | – | |

## UUID resolution

- `usage_event` only contains internal numeric IDs, while the CloudStack API
  works with UUIDs.
- During import, the internal IDs are resolved to the **original CloudStack
  UUIDs** by reading them from the CloudStack database (e.g.
  `vm_instance.uuid`, `account.uuid`). No UUIDs are generated by this
  project.
- `usage_event` has no domain column. The **domain UUID** of the resource
  owner is derived during import via `account.domain_id` and stored with
  every event, because the domain is the billing basis. This also covers
  accounts that are deleted later.
- The ID-to-UUID mapping is immutable, so storing the UUID with the event
  is not a history of master data.
- Which column is resolved against which table is defined **per event
  type** (see table above). Flag values are stored as flags and never
  resolved.
- If a UUID cannot be resolved (e.g. the row was hard-deleted), the event is
  still stored, the UUID is left empty and the event is marked as
  `unresolved`. Nothing is guessed.
- Unknown event types are imported and flagged, not dropped.
- Names and other details are fetched at query time via the CloudStack API
  using the UUID. For deleted objects only the UUID and the
  `resource_name` from the event are available; this is an accepted
  limitation.

Open: the exact lookup tables per event type and the time zone of
`usage_event.created` still have to be verified against the 4.22.1 source.

## Target database

- **PostgreSQL**, storing the **raw events only**.
- Hourly usage is computed at query time from the raw events and is not
  stored. This keeps a single source of truth, and late events or logic
  fixes take effect immediately.
- TimescaleDB was considered. It only pays off once derived hourly data is
  stored, so it is postponed until measurements with real data show that
  query-time computation is too slow. The schema is kept compatible with a
  later migration.
- Reproducibility of already issued invoices is the responsibility of the
  external billing system.

## Import process

- The importer is a command that performs **one run and exits**.
- Progress is tracked with a **watermark** (highest imported
  `usage_event.id`). The watermark is updated in the same transaction as
  the imported events.
- Because auto-increment IDs are not necessarily committed in order, each
  run re-reads an **overlap window** (e.g. the last 1000 IDs). Inserts use
  `ON CONFLICT DO NOTHING`; raw events are immutable and never updated.
- A run:
  1. reads new events from `usage_event` (with overlap, ordered by `id`),
  2. reads the matching `usage_event_details`,
  3. resolves UUIDs per event type,
  4. in one transaction: inserts events and details and updates the
     watermark.
- A PostgreSQL **advisory lock** prevents concurrent runs.

### Scheduled operation

- A **systemd timer** starts the importer every minute.
- Each run processes a bounded number of events.

### Initial import

- Started **manually**. One invocation processes exactly **one batch** of
  the size given on the command line and exits.
- The import can be stopped at any time and continues from the watermark.
- The systemd timer is enabled only after the initial import has caught up.

## Runtime and deployment

- **Python ≥ 3.12**, no ORM (plain SQL with explicit transactions).
- Configuration via a TOML file (`tomllib`), CLI via `argparse`. Credentials
  are never part of the repository or the image; they are provided at
  runtime.
- Runs in **Docker**. The systemd timer runs `docker run --rm` in the
  foreground, so the container output goes to the systemd journal.
- Target hosts: Ubuntu 24.04 or 26.04.
- Development uses Docker Compose with PostgreSQL and a MySQL instance
  containing test data in the `usage_event` schema.

## Observability

### Logs

- Written to stdout and collected in the systemd journal
  (`journalctl -u <unit>`).
- Each run logs a one-line summary.

### Metrics and alerting

- After each run the importer atomically writes a file in the Prometheus
  text format for the textfile collector of the host's metrics agent.
- Draft metrics:
  - `acs_usage_import_last_run_timestamp_seconds`
  - `acs_usage_import_last_success_timestamp_seconds`
  - `acs_usage_import_last_run_success`
  - `acs_usage_import_last_run_duration_seconds`
  - `acs_usage_import_events_imported`
  - `acs_usage_import_backlog_events`
  - `acs_usage_import_unresolved_events`
  - `acs_usage_import_watermark_id`
- Alerts (Prometheus rules, delivered via Alertmanager):
  - run failed (`last_run_success == 0`),
  - no run happened (last success too old; this also covers a stopped
    timer or a broken Docker setup),
  - backlog or unresolved events too high.

### Run log table `import_run`

One row per run, kept indefinitely for now.

| Column | Type | Content |
|---|---|---|
| `run_id` | `bigint` identity | primary key |
| `started_at`, `finished_at` | `timestamptz` | run start and end |
| `mode` | `text` | `scheduled` or `manual` |
| `status` | `text` | `running`, `success`, `failed`, `skipped`, `aborted` |
| `batch_size` | `int` | batch size used |
| `watermark_before`, `watermark_after` | `bigint` | progress before and after the run |
| `first_event_id`, `last_event_id` | `bigint` | ID range read, including overlap |
| `events_read` | `int` | events read |
| `events_inserted` | `int` | newly inserted events |
| `events_skipped` | `int` | events already present (overlap) |
| `details_inserted` | `int` | newly inserted detail rows |
| `events_unresolved` | `int` | events with at least one unresolved UUID |
| `source_max_id` | `bigint` | highest `usage_event.id` at run time |
| `backlog` | `bigint` | `source_max_id - watermark_after` |
| `error_type`, `error_message` | `text` | exception type and message on failure |
| `tool_version` | `text` | importer version |
| `hostname` | `text` | host or container running the import |

- The row is inserted with status `running` in its **own transaction** at
  the start of the run and updated at the end, so failures are recorded
  even if the data transaction is rolled back.
- Rows left in `running` after a hard termination are marked `aborted` by
  a later run.
- If PostgreSQL itself is unreachable, the failure is visible through the
  metrics file and the journal.

## Postponed topics

- Schema of the raw event tables in PostgreSQL.
- Hourly usage computation.
- API design (including the default value of the recursive parameter).
