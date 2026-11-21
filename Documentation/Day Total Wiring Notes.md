# Day total: notes for wiring it into Main.vi (2026-11-19)

The day's tracked time is built and tested outside `Main.vi`. Claude does not edit `Main.vi`; these are the hand changes.

## What exists

| VI | What it does |
|---|---|
| `UI Support\Day Total Entries.vi` | Reads files. Returns the entries that count toward a day: the ones that start on it, plus any from the previous day that run past midnight or are still running. Call it **once when a day loads**. Terminals: **Time Entry Store in** top-left / **out** top-right, **Day** (left, 2nd), error in / out; **Total Entries** (right, 2nd). |
| `UI Support\Day Total.vi` | Arithmetic only, no files. Sums each entry clipped to the day's local midnight-to-midnight (DST-correct); a running entry counts up to **Now**. Gives **Total Seconds** and **Total Text** (`HH:MM:SS`, hours not wrapped at 24). Call it **on every timer tick**. Terminals: **Entries** (left, top), **Day** (left, 2nd), **Now** (left, 3rd), error in / out; **Total Seconds** (right, 2nd), **Total Text** (right, 3rd). |
| `UI Support\UI State.ctl` | New last element **Total Entries** (`Time Entry List.ctl`), to hold Day Total Entries' result between ticks. |
| `UI Support\Daily Rollup Dialog.vi` | New indicator **Time Tracked** (HH:MM:SS, worked out once when the dialog opens) and a new optional input **Total Entries** (top-middle). Unwired, it falls back to Day Entries. The rounded-hours **Total** is unchanged. |

All of it is tested in `Tests\Run UI Support Tests.vi` ("== Day Total").

## 0. First: put the roll-up dialog's window X back

The rebuild of `Daily Rollup Dialog.vi` could not carry its **Panel Close?** frame, so the window X currently does not end the dialog. Please re-add a **Panel Close?** event frame with Discard? = TRUE that stops the loop, as before. Also move the new **Time Tracked** indicator: LabVIEW put it at the top, above Total.

## 1. The indicator

Add a **Time Tracked** string indicator (Fuse String Indicator) to the main window, e.g. next to the TimeCard or the date.

## 2. Load Data for Month (main loop)

After `Load Day.vi`:

1. `Day Total Entries.vi`: Time Entry Store in = the store out of `Load Day.vi`, Day = the same day → bundle **Total Entries** into `UI State.Total Entries`, and the store back into Main Loop Data.
2. `Day Total.vi`: Entries = that Total Entries, Day = the same day, Now = Get Date/Time In Seconds → write **Total Text** to the **Time Tracked** indicator.

## 3. Update Time Entry (the timer tick)

1. `Day Total.vi`: Entries = `UI State.Total Entries`, Day = the **Displayed Day** local variable, Now = the Get Date/Time In Seconds this frame already uses → write **Total Text** to **Time Tracked**.

This only does arithmetic, so it is fine on every tick. Every frame that changes entries (Add, End, Modify, and Insert once wired) already queues Load Data for Month, which refreshes Total Entries.

## 4. Daily Roll-up frame

Wire `UI State.Total Entries` into `Daily Rollup Dialog.vi`'s new **Total Entries** input, so the dialog's Time Tracked includes an overnight entry too.

## Things to know

- **DST-change days:** `Storage\Support\Read Day Entries.vi` now returns the entries that start between local midnight and the next local midnight (from `Increment_Decrement by one Day.vi`), for both bounds (NO CHANGE for the day start, Increment for the next), so the 23-hour spring day and the 25-hour autumn day are covered exactly (tested in `Tests\Storage\Test Read Day Entries DST.vi`). Earlier it read 24 hours from a day start that could be an hour off on those days (it kept the DST flag of the given time), so 8 Mar also showed 7 Mar 23:00-24:00 and 9 Mar 00:00-01:00, and 1 Nov missed 00:00-01:00 and 23:00-24:00. The de-duplication by ID in `Day Total Entries.vi` is no longer needed for that case but does no harm.
- The DST test cases assume US Central time (this PC's time zone).
