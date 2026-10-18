# Storage v2 (schema version 2): plan and progress

Branch: `feature/data-structures`. The goal is to move the app to the storage format in `Documentation/Sample Data/README.md` and to fix bugs 2 and 8 and the decimal-separator issue from `Documentation/Code Review.md`.

## Decisions (agreed 2026-10-17)

- **Migration:** automatic. On startup, if `Projects.ini` has no `[_Meta] Schema Version = 2`, back up every `*.ini` into `Backup v1 <timestamp>\` and convert the files.
- **Classes:** new classes are built alongside the old ones. `Main.vi` is switched over afterwards, and only then are `Projects/` and `Time Entries/` deleted. The app keeps working throughout.
- **Scope:** first build and test the data layer, then check in with Erich, then wire `Main.vi` and the dialogs.
- **`[Default]` project:** this is not a real project. It is the default-task template, and it migrates to `Settings.ini` `[Default Tasks]`.
- **Running entry:** an `End Time` of timestamp 0 is written to the file as an empty string.
- **Tests:** Caraya tests live under `Tests\` on disk and in their own top-level **Tests** virtual folder in the `.lvproj`, separate from the source. The runner sits in `Tests\`. No test VI goes inside a class or library.

## Target layout

| Item | Location |
|---|---|
| Row typedefs `Project.ctl`, `Task.ctl`, `Time Entry.ctl` (Fuse) | `Storage\Rows\` |
| `Project Store.lvclass`: maps ID→Project and ID→Task | `Storage\Project Store\` |
| `Project Store-INI.lvclass`: reads and writes `Projects.ini` + `Tasks.ini`, full rewrite | `Storage\Project Store-INI\` |
| `Time Entry Store.lvclass`: storage gateway (Read Month, Read Entries for Day, Save, Read, Delete, Get Running Entry) | `Storage\Time Entry Store\` |
| `Time Entry Store-INI.lvclass`: month file chosen from each entry's UTC start time | `Storage\Time Entry Store-INI\` |
| `Timestamp to ISO 8601 UTC.vi`, `ISO 8601 UTC to Timestamp.vi` | `Utilities\Time Management\` (in `Time Management.lvlib`) |
| `Migrate v1 Data.vi` | `Storage\Migration\` |
| Caraya tests and runner | `Tests\`, `Tests\Storage\` |

Live user data is in `%LOCALAPPDATA%\LVTimeTracker`: v1 format, 4 projects including `Default`, 17 entries from Jul–Oct 2025. Erich says these are throwaway test files from earlier examples (2026-10-17), so they may be modified or migrated in place when that is easier. Automated tests still use copies, so that they can be repeated.

## Progress

- [x] Read the code review, create the branch, agree on the decisions
- [x] Data layer: typedefs, classes, time helpers, migration (all built 2026-10-17)
- [x] Tests passing: `Tests\Run Storage Tests.vi` reports 35/35. **These are plain-LabVIEW tests, not Caraya, because Caraya is not installed in LabVIEW 2026 (32-bit).** Port them once Caraya is installed through VIPM.
- [x] Migration run on the live folder 2026-10-17. Backup is in `Backup v1 2026-10-17 212116\`. The 4 projects include `Default` as an inactive project, because a v1 entry in 2025-08 used it.
- [ ] Before committing: the `Work Folder` default in `Run Storage Tests.vi` points at a session scratch folder and needs changing (for example, to an empty path that falls back to the system temp folder)
- Known deviations: private data uses typedef arrays with linear lookup (the tooling cannot author maps); INI files are read and written by our own text parser in `Storage\Support\`; the wizard accessors use NI's pane, not 7/5; the test VIs have no icons. Class controls and three array controls are not Fuse (Fuse has no equivalents).
- [x] Commit the data layer (`0953bd3`); Erich is reviewing it in LabVIEW
- [ ] Switch `Main.vi`, `Edit Time Entry.vi` and `New Project.vi` to the new classes; change `Main Loop Data` to hold the active entry as a `Time Entry` row and to cache each day as rows
- [ ] Fix bug 1 (day cache) on top of that
- [ ] Delete the old `Projects/` and `Time Entries/` classes, and the broken `CleanUp.vi` / `INI_Init.vi`
- [ ] Update the status columns in `Code Review.md`

## Resuming

If work stops partway: run `git status` on the branch. Then use `lvai_describe_class` and `lvai_describe_ctl` to see which files under `Storage\` exist and are complete, and check whether the files under `Tests\` run. Finish the first unchecked item above. Nothing from this work has been committed yet unless `git log` shows a `[Claude]` commit for it.
