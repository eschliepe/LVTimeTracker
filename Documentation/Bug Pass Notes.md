# Bug pass: notes for Main.vi (2026-10-31)

Fix notes for the bugs still open in `Main.vi` after the storage v2 rewiring (see `Code Review.md`). Claude does not edit `Main.vi`; each section says what to change by hand. Node names are the ones in `Main.vi` at `05b4413`.

The order below is a suggestion. Each section stands on its own, and the app stays runnable after each one.

## 1. Route errors from both loops (Code Review bug 6)

New helper: `Utilities\Error Logging\Route Error to Queue.vi` (no class, so the queue reference is on the top-middle terminals):

| conIdx | Terminal |
|---|---|
| 7 / 5 | Queue In / Queue Out (the main message queue) |
| 8 / 0 | error in / error out |
| 3 | Stop Loop? (bool) |

- No error in: does nothing, Stop Loop? FALSE.
- An error that is not fatal: sends it to the main loop as an **Error** message (through `Queue-Error.vi`). error out is cleared and Stop Loop? is FALSE. The Error frame then logs it with `Log Queued Error.vi`.
- A fatal error: the queue reference is invalid or released (codes 1, 1122, 1556), or the Error message itself cannot be queued. Then Stop Loop? is TRUE and error out carries that error.

**Main loop ("Main Loop - All the action happen here"):**
1. `DeQueue State.vi`'s **error in** comes in through a plain input tunnel from `Queue-Initialize.vi`'s error out, so every pass starts with the startup error. Replace that tunnel with a **shift register**: initialize the left side from `Queue-Initialize.vi`'s error out.
2. After the case structure, wire its **error out** tunnel (today it leaves the loop through a plain output tunnel) into `Route Error to Queue.vi`. Queue In = `DeQueue State.vi`'s queue out. Wire the helper's error out into the right side of the new shift register.
3. Loop stop: **Or** the case structure's stop output with the helper's **Stop Loop?**.
4. **Error** frame: don't wire `Log Queued Error.vi`'s error out to the frame's error output; wire the frame's incoming error through instead. Otherwise a log file that cannot be written would send an Error message, fail to log it, and send another, forever.

This also fixes the startup worst case. If `Create Queue.vi` or `Queue-Initialize.vi` fails, the first pass gets an invalid queue, the helper says Stop Loop?, and the loop ends instead of re-initializing in a tight loop.

**Event loop:**
1. Add an error **shift register** to the event loop, initialized from `Queue-Initialize.vi`'s error out (the plain tunnel it uses today).
2. Give the event structure an error **output** tunnel. In each frame, wire the last `Queue-*.vi` call's error out to it, or wire the error straight through in frames with no call.
3. After the event structure: error → `Route Error to Queue.vi` (Queue In = the event loop's queue tunnel) → right side of the shift register.
4. Event loop stop: **Or** the existing stop with **Stop Loop?**.

## 2. Separate Initialize from Default (Code Review bug 5)

The main case structure's frame is labelled `"Initialize", Default`, so any message without its own frame re-runs initialization.

1. Edit that frame's label to just `"Initialize"`.
2. Add a new frame and make it **Default**. Pass Main Loop Data, the queue, the timeout and the user event straight through, and leave the loop running (stop = FALSE).
3. In it, build an error and wire it to the frame's error output:
   - **Format Into String** `Main.vi: unhandled message %s`, with the **Message** enum. Bring `DeQueue State.vi`'s Message into the case as a data tunnel too; the selector terminal can't be read inside a frame.
   - **Error Cluster From Error Code.vi**: error code 5200, error message = that string.
4. Section 1's helper then logs it as a normal error.

## 3. Release what the app creates (Code Review: "Queue never released, user event never destroyed")

After the loops end:
1. **Release Queue**: queue = `DeQueue State.vi`'s queue out, brought out of the main loop through an output tunnel. error in = the main loop's error output.
2. **Unregister For Events**: the event registration refnum, brought out of the event loop through an output tunnel (it goes in through a tunnel today).
3. **Destroy User Event**: the user event wire (it already leaves the main loop through an output tunnel).
4. Merge the three error outs and pass them to `Utilities\Error Logging\Log Error.vi`, so a shutdown error is written to the log.

## 4. Add / End Time Entry reload the wrong day

**Add Time Entry** and **End Time Entry** queue Load Data for Month with **Get Date/Time In Seconds** ("now") as Data. When you are viewing another day, the TimeCard jumps to today while the date display does not.

- In both frames, replace the Get Date/Time In Seconds that feeds **Queue-Load Data for Month.vi**'s Data with a **Displayed Day** local variable (read), as Modify Time Entry already does.
- Keep Get Date/Time In Seconds where it feeds `Start Entry.vi`'s Start Time or `Stop Entry.vi`'s End Time.

**Add Time Entry**, minor: `Start Entry.vi`'s New Entry is bundled into `UI State.Active Entry` even when Start Entry returns an error (5101 when no project or task is selected). In that case New Entry is empty, so the active entry is cleared.
- Put a **Select** in front of that Bundle By Name: s = Start Entry's error out (a Select takes an error cluster as its selector), t = the current `UI State.Active Entry` (unbundled), f = New Entry.

## 5. Make Inactive: find the row with the listbox method

The **Make Inactive** frame works out the Tasks row from the mouse position, the listbox's Top Row, Number of Rows and ContentRectBoundsHeight (Divide, Quotient & Remainder). That breaks if the row height or a scroll offset changes.

- Use an **Invoke Node** on the Tasks listbox: method **Point To Row Column**, Point = the shortcut-menu event's **Coords** (panel coordinates).
- Use its **Row Number** as the row. When the click is not inside a row (Inside? FALSE / row -1), do nothing.
- Delete the Property Node (Top Row, Number of Rows, ContentRectBoundsHeight), the Divide and the Quotient & Remainder.
- Check the method's exact name and outputs in your LabVIEW version's Invoke Node list. This note was written from the documentation, not tried here.

## 6. DST day stepping (Code Review bug 4): no Main.vi change

`Utilities\Time Management\Increment_Decrement by one Day.vi` now steps by calendar date, not 86,400 s, and builds its result exactly as `Get Current Day as Timestamp.vi` does, so the two compare Equal on every date, DST changes included. Its connector pane is unchanged, so the two calls in `Main.vi` need no rewiring.

Follow-up (2026-11-01): both VIs used a date/time record with DST = 0, so on dates inside daylight saving time they returned 01:00 instead of 00:00. Both now set DST = -1 (automatic) and return true local midnight on every date; they still compare Equal.

## 7. Clean-up candidates

Nothing calls these any more:
- `Additional UIs\Edit Time Entry.vi`: an unused 2025-08 stub (the real dialog is `UI Support\Edit Entry Dialog.vi`).
- `UI Support\Modify Entry.vi`: its body was inlined into the Modify Time Entry frame.

Delete them when convenient (remove them from the project too).
