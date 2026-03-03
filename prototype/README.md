# Rescue ongoing recording — Prototype

Single-page prototype: **user can create a new recording**; **user cannot access the stuck recording**. The only escape is a **Rescue** button on the stuck “Ongoing” dictation row (Web Page Design folder style).

## Problem (correct in all files)

- **User can create (or add) a new recording** anytime.
- **User cannot access or play** the recording that is stuck (e.g. started on another device, stop never sent).
- **Rescue** on the stuck row closes that recording and makes partial notes available.

## What the prototype does

1. **INPUTS FOR AI FOR JOHN DOE** — List of dictations. One row is **“3. Ongoing Dictation by Nicolas at 2:25 PM”** (dashed underline = stuck). You cannot access or play it; you can use **Rescue** on that row.
2. **DICTATION** — Start a new recording (always allowed; no blocker).
3. **Rescue** — Button only on the stuck row. Click to close the stuck recording and get partial notes.
4. **Reset simulation** — Restores the stuck “Ongoing” row.

## UI

Styled to match the **Web Page Design** folder (Consult / Athena-style): sidebar, section title, purple DICTATION button, green ADD CONSULT / PROCESS. No “MARVIX” file was found in the folder; if you have MARVIX UI specs or assets, they can be swapped in.

## Run

```bash
open prototype/index.html
# or
npx serve prototype
```
