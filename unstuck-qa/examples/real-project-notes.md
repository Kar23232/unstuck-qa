# 🗺️ Notes from a real project

These traps come from testing a real data-entry web app: long forms, lab-result tables, AI-suggested values that a human confirms, and a lot of save buttons. Names and URLs are anonymized. The patterns are what matter.

## 🧭 Navigation

- 🔗 **Deep links beat clicking.** A URL like `/cases/123/forms/labs?series=r1&row=2#lab-series-r1` lands exactly where you need. Clicking breadcrumbs sometimes doesn't navigate at all (see the next item).
- 🚧 **Unsaved-changes guard.** With unsaved edits, every link opens a modal: *"You have unsaved changes: Cancel / Leave anyway"*. The URL doesn't change, so the agent thinks the click failed.

## 💬 Native dialogs (`window.confirm`)

These showed up on:

- delete
- "restore the original AI suggestion"
- "discard local edits"
- "cancel draft transfer"
- "move & confirm" during a partial save

No handler means the click never returns.

## 📌 Things covering the button

- A sticky bottom bar (*Cancel / Partial save / Save all & confirm*) sat on top of the last few fields.
- A floating "Candidates (N)" drawer button in the top-right corner covered content.

## 🔄 Pages that never sit still

- An AI-task panel polled the server every 2.5–5 s and re-rendered the page, so "wait until stable" never finished.
- After each save, the page re-rendered itself, so element handles went stale. Locate elements again after every save.

## 🧩 AI suggestions, one by one

Each row or field had suggestion chips like `#0 item: … result: … ref range: …`, plus a "Confirm no data" button. Confirming them is a long chain. If one step errors, note where you stopped and continue from there. Never restart from the top.

## 🧾 Errors that are results, not flakiness

- ⚔️ **409 "version changed, please refresh"** on "Confirm & assign" when two edits raced. That's expected optimistic locking. Record it as the result.
- 🧨 **`An error occurred in the Server Components render…`** is an uncaught server exception. Record the text and the digest. It's a bug, not the network.
- 🔁 **"Please refresh and try again"**: retry once. If it fails again, it's a fail.
