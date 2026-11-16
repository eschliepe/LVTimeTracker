# Sample data: proposed storage layout (schema version 2)

These files illustrate a proposed replacement for the current INI layout. They are
hand-written examples, not output from the application, and the application does not
read this format yet.

## The rule

Each file is a table. Each section is a row. The section name is the row's primary
key (a GUID). Each key is a column. A database backend can later store the same rows
without reshaping them.

| File | Table | Row key | Foreign keys |
|---|---|---|---|
| `Projects.ini` | Projects | project GUID | none |
| `Tasks.ini` | Tasks | task GUID | `Project ID` |
| `YYYY-MM.ini` | Time Entries | entry GUID | `Project ID`, `Task ID` |

## Columns

**Projects:** `Name`, `Active`, `Sort Order`

**Tasks:** `Project ID`, `Name`, `Active`, `Sort Order`, plus the label `Project Name`

**Time Entries:** `Project ID`, `Task ID`, `Start Time`, `End Time`, `Description`,
plus the labels `Project Name` and `Task Name`

## Conventions

- **IDs are the authority.** `Project Name` and `Task Name` on tasks and entries are
  labels for people reading the file. Write the current name on save; ignore the label
  on read and look the name up by ID. A database would not have these columns.
- **Timestamps are ISO 8601 in UTC**, for example `2025-10-06T13:02:11.250Z`.
- **An empty `End Time` means the entry is still running.** It maps to NULL in a
  database.
- **The month file is chosen by the entry's `Start Time`** (in UTC), even if the entry
  ends in the next month.
- **`[_Meta]` is not a row.** It carries `Schema Version` so old files can be detected
  and migrated. Code that lists section names must skip it.
- Strings are quoted and booleans are `TRUE`/`FALSE`, matching what LabVIEW's
  configuration file VIs write.

## Cases the samples cover

| Case | Where |
|---|---|
| Inactive project | `Internal` in `Projects.ini` |
| Inactive task with an existing entry | `Testing` in `Tasks.ini`; fourth entry in `2025-10.ini` |
| Same task name in two projects | `Development` under both `TimeTracker` and `Client Website` |
| Entry crossing a month boundary | last entry in `2025-10.ini` |
| Stale name label after a rename | first entry in `2025-11.ini` (`Admin` vs `Administration`) |
| Running entry | last entry in `2025-11.ini` |
| Empty description | second entry in `2025-10.ini` |

## Equivalent database tables

```sql
CREATE TABLE projects (
  id         TEXT PRIMARY KEY,
  name       TEXT NOT NULL,
  active     INTEGER NOT NULL,
  sort_order INTEGER NOT NULL
);

CREATE TABLE tasks (
  id         TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(id),
  name       TEXT NOT NULL,
  active     INTEGER NOT NULL,
  sort_order INTEGER NOT NULL
);

CREATE TABLE time_entries (
  id          TEXT PRIMARY KEY,
  project_id  TEXT NOT NULL REFERENCES projects(id),
  task_id     TEXT NOT NULL REFERENCES tasks(id),
  start_time  TEXT NOT NULL,
  end_time    TEXT,
  description TEXT NOT NULL DEFAULT ''
);
```
