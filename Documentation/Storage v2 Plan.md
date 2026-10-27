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
- [x] Tests passing: `Tests\Run Storage Tests.vi` reports 34/34 (the earlier "35" was a miscount; the tests hold 34 checks). **These are plain-LabVIEW tests, not Caraya, because Caraya is not installed in LabVIEW 2026 (32-bit).** Port them once Caraya is installed through VIPM.
- [x] Migration run on the live folder 2026-10-17. Backup is in `Backup v1 2026-10-17 212116\`. The 4 projects include `Default` as an inactive project, because a v1 entry in 2025-08 used it.
- [x] `Run Storage Tests.vi`: `Work Folder` now defaults to empty, which means `<temp>\LVTimeTracker Tests` (2026-10-18; regenerated, so its hand layout was lost, with Erich's OK)
- Known deviations: private data uses typedef arrays with linear lookup (the tooling cannot author maps); INI files are read and written by our own text parser in `Storage\Support\`; the wizard accessors use NI's pane, not 7/5; the test VIs have no icons. Class controls and three array controls are not Fuse (Fuse has no equivalents).
- [x] Commit the data layer (`0953bd3`); Erich is reviewing it in LabVIEW
- [x] `UI Support\` helpers for `Main.vi` (Initialize Stores, Project/Task List, Load Day, Format Entry Row, Start/Stop Entry, Save Edited Entry, Set Task Active, Save Project) and `UI State.ctl`; `Tests\Run UI Support Tests.vi` reports 40/40 (2026-10-18). Agreed with Erich: no day cache (re-read the day instead); Erich rewires `Main.vi` himself, one frame at a time, following `Documentation/Main.vi Rewiring Guide.md`.
- [x] Erich: rewire `Main.vi` per the guide (`81f91d4`, `835d1e7`, `edb3a86`). Checked 2026-10-25: every frame in the guide is on the new layer, including Load Data for Month (cache removed), Set to Current Day (bug 3), Add/Edit Project and Save Project(s). No frame calls the old classes; the old fields are gone from `Main Loop Data.ctl`; the old classes are out of the `.lvproj`. `Main.vi` is executable; Storage 34/34, UI Support 40/40.
  - `UI Support\Modify Entry.vi` no longer has a caller, because its body was inlined into the Modify Time Entry frame. Keep it until the real Edit Entry dialog exists, or delete it.
- [x] `New Project.vi` on the Project Store (`865e33b`; closing the window with the X is untested)
- [x] **Edit Entry dialog** (2026-10-25): `UI Support\Edit Entry Dialog.vi` is the real dialog now, generated fresh with the placeholder's pane, so Main.vi's call is unchanged. It has Project and Task rings, Start Time, End Time (greyed for a running entry, which is saved as 0), Description, and OK and Cancel. OK checks that a task is selected and that End Time is not before Start Time. Closing the window counts as Cancel. An entry whose project or task is inactive still shows it; the new `UI Support\Find or Add List Item.vi` adds it to the list. Driven through VI Server against the sample data, five cases passed (OK with a new description, a different task, a different project, a running entry, Cancel).
  - Correction: the plan said to build it on `Additional UIs\Edit Time Entry.vi`. That VI is an unused 2025-08 stub (Comment, Start, End, no buttons). The real old dialog was `Time Entries\Time Entry\Edit Time Entry.vi`, deleted in `c5de36e`, and it is still in git. Erich chose a fresh panel. `Additional UIs\Edit Time Entry.vi` can now be deleted.
  - 2026-10-26, rebuilt: the composed VI crashed LabVIEW, or failed to save with error 42, after any control Replace, in the IDE too. A copy and a bug write-up are in `C:\Apps\LVTimeTracker\Crash Repro - Edit Entry Dialog\` (outside the repo). The VI was rebuilt by grafting: LabVIEW pasted the composed diagram and the new controls into a VI holding only the pane controls. The rebuilt VI saves after Replace. Its controls are now Fuse, Time Entry / Edited Entry are linked to `Time Entry.ctl`, and the five cases pass again.
  - **Open (Erich):** the panel layout, and the window-close event. Closing the window with the X does not end the dialog yet. The graft could not carry a Panel Close frame, and Erich asked to add the event himself rather than have it polled. Still untested: the invalid-input message, and an entry with an inactive project or task.
- [ ] Bug 1 (day cache): closed by the rewiring, since there is no cache; confirm in the app
- [x] Deleted the old `Projects/`, `Time Entries/` and `Task_Management/` folders (47 files, including `CleanUp.vi`, `INI_Init.vi` and `Time Entries\Time Entry\Edit Time Entry.vi`), 2026-10-25. Before deleting, a byte search found no file outside them that referenced their classes. Afterwards `Main.vi` was still executable, and the suites still gave 34/34 and 40/40.- [ ] Update the status columns in `Code Review.md`

## Resuming

If work stops partway: run `git status` on the branch. Then use `lvai_describe_class` and `lvai_describe_ctl` to see which files under `Storage\` exist and are complete, and check whether the files under `Tests\` run. Finish the first unchecked item above. Nothing from this work has been committed yet unless `git log` shows a `[Claude]` commit for it.
