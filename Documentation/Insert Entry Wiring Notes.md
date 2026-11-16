# Insert entry: notes for wiring it into Main.vi (2026-11-11)

The split logic and the dialog mode are built and tested outside `Main.vi`. Claude does not edit `Main.vi`; these are the hand changes to hook it up.

## What exists

| VI | What it does |
|---|---|
| `UI Support\Edit Entry Dialog.vi` | New input **Insert Mode? (F)**. With it TRUE, the dialog opens on the entry you right-clicked (the original), shows its times for reference, defaults to a 15-minute slot in the middle of it, an empty description, and lets you pick another project/task. OK only accepts a time inside the original (up to now for a running entry). **Edited Entry** is then the *inserted* entry. With it FALSE (unwired) the dialog works exactly as before, so the Modify Time Entry frame needs no change. |
| `UI Support\Insert Entry.vi` | Splits the original around the inserted time and saves the pieces: piece 1 (original ID, original description, ends at the inserted start), the inserted entry, piece 3 (original project/task, empty description, starts at the inserted end, ends at the original end). Zero-length outer pieces are dropped, except that a running original always keeps a running piece 3, which comes out on **Running Entry**. Invalid times give error 5104 and nothing is written. Tested in `Tests\Run UI Support Tests.vi` ("== Insert Entry"). |

## 0. First: put the dialog's window X back (Edit Entry Dialog.vi)

The rebuild of `Edit Entry Dialog.vi` could not carry your **Panel Close?** frame (the graft tool refuses filter events), so pressing the window X now closes the window without ending the dialog, in both Edit and Insert mode. Please re-add it as before: a **Panel Close?** event frame, Discard? = TRUE, that does what Cancel does (Updated? FALSE, entry and Project Store passed through, stop the loop). The new Insert Mode? input reaches the event structure through its own tunnel; the frame doesn't need it. Then update the sentence in the VI description that says the X is not handled yet.

`Insert Entry.vi` terminals: **Time Entry Store in** top-left (11) / **out** top-right (3), **Project Store in** top-middle (7) / **out** (5), **Original Entry** (left, 10), **Inserted Entry** (left, 9), **Now** (bottom-middle, 6), **Pieces** (right, 2), **Running Entry** (right, 1), error in / out.

`Edit Entry Dialog.vi` gained **Insert Mode? (F)** on the left, third terminal (9). The dialog's window title reads "Insert Time Entry" in Insert mode.

## 1. A message for it

As for the Daily Roll-up:

1. `Internal Message Library\Controls\Message.ctl`: add an item, e.g. **Insert Time Entry**.
2. Make `Internal Message Library\Queue-Insert Time Entry.vi` from `Queue_Template.vit`, Message constant **Insert Time Entry**, and add it to `Internal Messaging Library.lvlib`.

## 2. The right-click item (event loop)

1. Add an item to the TimeCard listbox's shortcut menu, e.g. **Insert Entry…** (its tag must match the case label exactly, including case).
2. In the **"TimeCard Listbox": Shortcut Menu Selection (User)** frame, add a case for that tag. It should find the clicked row the same way Make Inactive does (Point To Row on the event's Coords; skip when the row is -1 / not in bounds) and send `Queue-Insert Time Entry.vi` with **Data = the row** (I32). Wire its error out to the frame's error output.

## 3. The frame (main loop)

New case **"Insert Time Entry"**, built like Modify Time Entry:

1. `Variant To Data` (I32) on the message data → **row**. If the row is outside `UI State.Day Entries`, do nothing.
2. **Original Entry** = `UI State.Day Entries[row]`.
3. `Edit Entry Dialog.vi` with Project Store in, Time Entry = Original Entry, **Insert Mode? = TRUE** → Edited Entry (the inserted entry), Updated?.
4. Case on **Updated?**, True:
   - `Insert Entry.vi`: Time Entry Store in, Project Store in, Original Entry, Inserted Entry = Edited Entry, Now = Get Date/Time In Seconds.
   - If `Original Entry.ID` = `UI State.Active Entry.ID`, bundle **Running Entry** into `UI State.Active Entry` (the timer carries on as piece 3).
   - Bundle both stores back into Main Loop Data.
   - Queue **Load Data for Month** with Data = the **Displayed Day** local variable, so the TimeCard redraws with the new rows.
5. False: pass everything through. Wire the error to the frame's error output.

## Things to know

- If the original was the running entry, step 4's Active Entry update matters: without it the TimeCard's timer keeps pointing at piece 1, which has now ended.
- The dialog's new controls (Insert Mode? on the pane, the "Original" indicator, and any title change) were placed by LabVIEW; arrange the panel by hand.
