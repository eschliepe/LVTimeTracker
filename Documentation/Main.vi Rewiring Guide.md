# Main.vi rewiring guide (storage v2)

How to move `Main.vi` from the old `Projects/` and `Time Entries/` classes to the new storage layer, one frame at a time, with the app still running after each step. The helpers are in `UI Support\` (project folder **UI Support**); each one is tested by `Tests\Run UI Support Tests.vi`.

## Step 0: add the new state to `Main Loop Data.ctl`

Add three fields and **keep the old ones for now** (Bundle/Unbundle By Name keep working, so nothing breaks):

| New field | Type |
|---|---|
| `Project Store` | `Project Store.lvclass` (drag the class from the project into the cluster) |
| `Time Entry Store` | `Time Entry Store.lvclass` |
| `UI State` | `UI Support\UI State.ctl` |

Use the **parent** classes. `Initialize Stores.vi` puts the INI children on those wires.

`UI State` holds:

| Field | Replaces |
|---|---|
| `Active Entry` (Time Entry row; empty `ID` = nothing running) | `Time Entry.lvclass` + `Time Entry Active?` |
| `Displayed Day` | `Displayed Day` |
| `Project IDs`, `Task IDs` | nothing; they are parallel to the Projects/Tasks listbox items, so a selected row maps to an ID |
| `Row Entry IDs`, `Day Entries` | `Time Entries by Day` (the cache is dropped; see bug 1) |

## Frame by frame

Selected IDs come from indexing the parallel arrays with the listbox `Value`: `Project IDs[Projects.Value]`, `Task IDs[Tasks.Value]`, `Row Entry IDs[TimeCard row]`.

Terminals: every helper is 4-2-2-4 with error in/out at the bottom corners. A store comes in on the top middle (7) and goes out on 5; when a helper takes both stores, the second is on 6 in / 4 out. **Start Entry, Stop Entry and Save Edited Entry also take the Project Store**: Save Entry uses it to write the `Project Name`/`Task Name` labels into the month file. Rows keep the old seven columns, **ID first**: ID, Start, End, Elapsed, Project, Task, Description (same order as the old `Update Time Entry.vi`).

| Frame | Take out | Put in |
|---|---|---|
| **Initialize, Default** | `Projects-INI:Initialize`, `Time Entry-INI:Initialize` | `Initialize Stores.vi` (Data Folder empty = `%LOCALAPPDATA%\LVTimeTracker`; creates it if missing) → bundle `Project Store`, `Time Entry Store`; put `Running Entry` into `UI State.Active Entry` when `Entry Running?` is TRUE (and fire `Mark Active` as now) |
| **Load Data** | `Projects:Read from Data Source` | nothing (Initialize Stores already read the data); just queue Update Project List. Keep the frame for the menu's Load Projects; there, call `Project Store:Read from Data Source` |
| **Update Project List** | the `Projects` property node, the For loop, Sort 1D Array | `Project List.vi` → `Item Names` = Project Names; store `Project IDs`. Re-select by ID: search the old selected ID in the new `Project IDs` |
| **Update Task List** | `Projects` property node, Look In Map, the For loop | `Task List.vi` with `Project IDs[Projects.Value]` → `Item Names` = Task Names; store `Task IDs` |
| **Load Data for Month** (rename to *Load Day* if you like) | the whole map/cache case structure | `Load Day.vi` (both stores; Day = `Displayed Day`, Now = Get Date/Time) → MCLB `Item Names` = Rows; store `Row Entry IDs`, `Day Entries`. Keep the Top Left Visible Cell write |
| **Add Time Entry** | `Time Entry-INI:Initialize` (and its class constant), `Create Time Entry`, `Calculate Time Entry` (and its `Recalculate Override` constant), `Save Time Entry`; the two Index Arrays on the Projects/Tasks `Item Names`; the `Time Entry Active?` / `Time Entry.lvclass` Bundle By Name; and all the TimeCard Listbox code: the read and write Property Nodes (`Item Names`, `Top Left Visible Cell`, `Number of Rows`), Build Array, Array Size, Index Array, To Unsigned Long Integer, Greater Or Equal?, Select, the `Row` Unbundle/Bundle By Name and Increment. Keep the two listbox `Value` reads. | `Start Entry.vi` (both stores; Project ID, Task ID from the parallel arrays, Description, Start Time = now) → `UI State.Active Entry` = New Entry; then queue Load Data for Month. **Don't write the TimeCard listbox in this frame**: Load Data for Month (via `Load Day.vi`) redraws the day from the file, new running entry included, and refreshes `Row Entry IDs` / `Day Entries` together, so the three can't get out of step. Error 5101 means nothing was selected, so show it or ignore it, but don't mark active |
| **End Time Entry** | `Set Stop Time`, `Save Time Entry`; the `Time Entry.lvclass`/`Time Entry Active?` Unbundle and Bundle By Name (and the `Time Entry Active?` FALSE constant goes to the local variable only); all the TimeCard code: the read and write `Item Names` Property Nodes, Array Size, Index Array, Decrement, Replace Array Subset. Keep the `Task Active` local variable write. | `Stop Entry.vi` (both stores; Active Entry, End Time = now) → `Active Entry` = Active Entry out; then queue Load Data for Month (redraws the day correctly, which fixes the "overwrites the last row" bug) |
| **Update Time Entry** (timer tick) | `Calculate Time Entry` (and its `Recalculate` constant), the `Time Entry.lvclass` Unbundle/Bundle By Name, Array Size, Index Array, Decrement. Keep the TimeCard read/write `Item Names` Property Nodes, the Replace Array Subset (you give it a new row index) and the Variant To Data / Equal? day check around it. | Only if `Active Entry.ID` is in `Row Entry IDs`: `Format Entry Row.vi` (Active Entry, Now, Project Store) → replace **that** row (index from Search 1D Array), not the last one |
| **Modify Time Entry** | `Time Entry-INI:Initialize` (and its class constant), `Read Time Entry`, `Edit Time Entry`, both `Update Time Entry`, `Calculate Time Entry`, `Save Time Entry`, the `Time Entry.lvclass`/`Projects.lvclass` Unbundle and Bundle By Name, the `Active Time Entry GUID` property node and Equal?, the 2D Index Array on `Item Names`, and the TimeCard read/write Property Nodes and Replace Array Subset. Keep the `Variant To Data` (message data = row). | 1. Original Entry = Index Array(`UI State.Day Entries`, row): `Day Entries` is parallel to the rows. 2. Edit Time Entry dialog → Edited Entry + Updated?. This is `UI Support\Edit Entry Dialog.vi` (Project Store, Time Entry → Edited Entry, Updated?); it shows its own panel and closes it again. 3. Case on Updated?, True: `Save Edited Entry.vi` (both stores; Original, Edited); if Edited Entry.ID = `Active Entry.ID`, bundle Saved Entry into `Active Entry`; queue Load Data for Month (don't write the TimeCard here). False: nothing. |
| **Inactivate Task** | `Projects:Set Task Inactive`, the `Projects.lvclass` Unbundle/Bundle By Name, the Projects listbox Property Node and both Index Arrays on `Item Names`, the Tasks `Item Names` read, and the whole Diagram Disable Structure with the `Write to Data Source` inside. The message data (Variant To Data → row) stays: Task ID = `UI State.Task IDs[row]` | `Set Task Active.vi` (Task ID, FALSE) — it writes to disk itself (bug 7); queue Update Task List |
| **Add Project / Edit Project** | The old `Projects.lvclass` Unbundle/Bundle By Name, the Equal? on Projects in/out and its case, and `Queue-Save Project(s)` (the dialog saves to disk itself). In Edit Project, the empty-string constant / name wiring into `New Project Name` (that input is gone). | `New Project.vi` (rebuilt): Project Store in (top-left) / out (top-right), Project ID (left, 2nd), Project ID out (right, 2nd), Saved? (right, 3rd). **Add:** Unbundle `Project Store.lvclass` → New Project.vi with Project ID unwired/empty → Bundle `Project Store out` back → case on Saved?: True = queue Update Project List. **Edit:** the same, with Project ID = `UI State.Project IDs[Projects.Value]` (the message data used to carry the project **name**; send the row or the ID instead, or ignore the data and read `Projects.Value` here) → if Saved?, queue Update Task List (and Update Project List if the name may have changed). Default Tasks now come from `Settings.ini` `[Default Tasks]` via `UI Support\Read Default Tasks.vi`, inside the dialog. |
| **Save Project(s)** | `Projects:Write to Data Source` | `Project Store:Write to Data Source` (every helper that changes projects already writes, so this is only for the menu item) |
| **"Set to Current Day": Mouse Up** (event loop) | — | also queue Load Data for Month (bug 3: writing the local variable fires no event) |
| **Exit / Panel Close?** | `Queue-End Time Entry` | unchanged; End Time Entry now uses `Stop Entry.vi` |

## Step last: remove the old fields

When no frame reads `Task_Management.lvclass`¹, `Projects.lvclass`, `Time Entry.lvclass`, `Time Entry Active?` or `Time Entries by Day`, delete them from `Main Loop Data.ctl`. Then the `Projects/` and `Time Entries/` classes, `CleanUp.vi` and `INI_Init.vi` can be deleted (plan checklist).

¹ Only if `Task_Management` is unused; it was not part of this work.

## Known limits

- The old Add Time Entry frame scrolled the TimeCard down so the new row was visible. Load Data for Month sets `Top Left Visible Cell` to the top row, so on a day with more rows than fit, the new entry can be below the visible area. If that matters, set `Top Left Visible Cell.Row` to the active entry's row there (Search 1D Array of `Active Entry.ID` in `Row Entry IDs`).

- `Get Running Entry` only looks in the current and previous UTC month, so a timer left running longer than that is not resumed at startup.
- `Initialize Stores.vi` with an empty Data Folder (the real `%LOCALAPPDATA%` folder) has no automated test; the tests always pass a copy.

## Bugs this closes

| Code Review item | How |
|---|---|
| 1 (stale day list) | no cache; every redraw re-reads the day |
| 1-related (last row overwritten) | Update/End Time Entry touch only the active entry's row, found by ID |
| 2 (earlier month uses wrong file) | entries are read/written by their own start time |
| 3 (Set to Current Day) | Mouse Up queues Load Data for Month |
| 7 (inactive task not saved) | `Set Task Active.vi` writes |
| 8 (writer never removes) | full rewrite in the new store |
| `:60` seconds | `Format Entry Row.vi` rounds first |
| empty project/task on Start | `Start Entry.vi` refuses with 5101 |
