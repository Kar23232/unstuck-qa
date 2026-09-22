---
name: unstuck-qa
description: Use when reproducing or regression-testing bugs in a real web app through a browser (clicking, filling forms, saving, very long data-entry pages) and collecting screenshot evidence for each issue. Built-in guardrails stop the agent from hanging or looping - one image per turn, viewport-only screenshots, native-dialog handling, unsaved-changes guards, and a one-click verdict rule for save actions. Also prevents the "No tool output found for tool call" error that can permanently break a Codex session on third-party model providers.
---

# unstuck-qa: reproduce web app bugs, collect evidence, never get stuck

**Use this when** you have a bug list (Issue 1..N) and need to reproduce each one in a real browser, ending with pass / fail + screenshot evidence.

**Don't use this for** pure API tests or code review. Those go through unit tests or HTTP scripts (see §10).

## 1. Hard rules (breaking one = the run failed)

1. **One image per message.** Never view two images in the same turn: no parallel `view_image` calls or image reads. See "Why rule 1 exists" below.
2. **Viewport screenshots only.** No full-page captures. For a long page, take 2–3 viewport shots while scrolling, or read structured text first.
3. **Shrink before you look.** Any image you will view must be at most 1500 px wide, JPEG preferred. Resize it yourself first:
   - macOS: `sips -s format jpeg -Z 1500 shot.png --out shot.jpg`
   - ImageMagick: `magick shot.png -resize '1500x>' shot.jpg`

   Never pass a raw full-page image (think 392 × 18,788 px).
4. **Never dump the whole page.** No full `innerText`, no full accessibility tree. Query only the selectors you need and return counts, states, and key text.
5. **Never wait for `networkidle` or "DOM stable".** Always wait for a specific element or specific text, with a timeout.
6. **If a tool call doesn't come back, stop.** Don't stack more calls in the same session. Tell the user what happened, then retry that single action once. If every request starts failing, see §3.

### Why rule 1 exists

Observed with Codex CLI + DeepSeek, September 2026:

1. The agent viewed two screenshots in parallel.
2. Both results were recorded. But the first image was large, so the client auto-resized it and inserted a short note ("image was resized") **between** the two tool results.
3. The provider expects every tool result to come right after its call, back to back. The note broke that order, so it rejected the request with `No tool output found for tool call call_01_...`.
4. The broken turn stays in the history, so every later request fails the same way. The session is dead.

Rule 1 removes the parallel images. Rule 3 removes the auto-resize. Keep both.

## 2. Keep a progress log

A dead session should cost nothing. Keep `progress.md` in the evidence folder:

- **At the start:** if `progress.md` exists, read it and continue from the first unfinished issue. Don't redo issues already marked pass / fail / blocked.
- **After every issue:** append its result right away, in the §9 format.
- **Also record:** the environment (base URL, test account, data state) and anything flaky or blocked.

## 3. If the session is already broken

Symptom: every request fails with `No tool output found for tool call ...`, always with the same call ID.

- Retrying in the same session does not work. The broken history is sent again every time.
- Resuming that session does not work either. Same history, same error.
- What works: start a new session. It reads `progress.md` (§2) and continues from the next unfinished issue.

## 4. Before opening the page

- **Kill leave-page blockers.** Stop `beforeunload` handlers from registering (init script), and set `window.onbeforeunload = null` after load.
- **Register a dialog handler before any click.** Log the type and message, then accept. A native `confirm` / `alert` blocks the page: with no handler, the click never returns. With a default dismiss, the action is silently cancelled.
- **For key buttons that open a dialog,** run once with accept and once with dismiss, and record both results.
- **Lower the default timeout** (for example 15 s), so a hang fails fast instead of blocking for minutes.

```js
// Playwright example
await context.addInitScript(() => {
  const add = window.addEventListener;
  window.addEventListener = function (type, ...rest) {
    if (type === 'beforeunload') return;
    return add.call(this, type, ...rest);
  };
});
page.on('dialog', async (dialog) => {
  console.log(`[dialog] ${dialog.type()}: ${dialog.message()}`);
  await dialog.accept();
});
page.setDefaultTimeout(15_000);
// after each navigation:
await page.evaluate(() => { window.onbeforeunload = null; });
```

## 5. Navigation and page state

- **Navigate by exact URL,** including query and `#anchor`. Don't click through links or breadcrumbs to get somewhere.
- **Watch for unsaved-changes guards.** Many apps turn every link into a "You have unsaved changes" modal while a form is dirty. The click looks like it did nothing and the URL doesn't change. Handle the modal explicitly, or save / discard first.
- **After every navigation, confirm where you landed.** Check the URL and one landmark element before doing anything else.

## 6. Save actions: one click, one verdict (most important)

1. Before clicking, capture a baseline: URL, key field values, status text.
2. Click once.
3. Capture three things together:
   - the toast / banner text, verbatim;
   - that request's HTTP status and response body, including any error code or digest;
   - console errors.
4. Give the verdict: **success** / **error** (with verbatim text and code) / **no reaction**.

Discipline:

- **"Please refresh and try again"**: retry at most once. A second failure is a fail. No refresh-and-retry loops.
- **A raw framework error in English** (for example `An error occurred in the Server Components render...`) is not network flakiness. It is an uncaught server exception. Record it verbatim as evidence and grab the digest.
- **"Data has changed" / "updated elsewhere" / version conflict** (often HTTP 409) is the result of the test. Record it. Don't refresh your way around it.

## 7. When a click fails, check in this order

1. **Covered.** Sticky bottom action bars and floating drawers or buttons can sit on top of the target. Scroll, or use another entry point for the same function.
2. **Stale handle.** The page re-rendered after a save. Locate the element again and never reuse old handles.
3. **Background polling.** Panels that refresh every few seconds mean "wait until stable" never happens. Use the exact waits from rule 5.
4. **Native dialog.** See §4. The handler must exist before the click.

## 8. Evidence

- **Name files** `<seq>-<issue>-<state>.png`, for example `07-p8-save-error.png`.
- **Each piece of evidence must stand alone:** URL, time, steps, and verbatim key text (toast, error, field value).
- **Things the UI can't show** (was it saved to the database? which record? why the conflict?): if you can query the database or logs, attach a text note as well. Don't rely on a screenshot alone.

## 9. Report format

Per issue, exactly three lines:

```
Issue N <title>
Result: pass / fail / blocked (+ one-line reason)
Evidence: <absolute path to png> (+ for fail: verbatim text and code)
```

End with a list of what was not verified in this run or needs a human check.

## 10. If it doesn't need the UI, don't use the UI

Priority, high to low:

1. Unit / function tests in the repo (isolated temp database, mocked AI / OCR / external services).
2. Direct HTTP / API scripts (with CSRF if needed). These give you status codes, error codes, and digests.
3. The real browser UI (this skill), mainly to produce screenshot evidence.

For the same bug, a UI run usually takes 10× or more tool calls than the API route, and the chance of getting stuck grows with it. Bugs about extraction, classification, or normalization belong in 1–2. Only UI behavior, interaction, and on-screen wording must go through 3.

## 11. Quick trap list

- Native `window.confirm` on delete, restore, discard, and move actions.
- An unsaved-changes guard turns links into a modal.
- A sticky bottom bar hides the last few fields.
- Floating drawer buttons cover content.
- Long chains of per-field confirmations (for example AI suggestions): if one step errors, note where you stopped and continue from there. Don't restart from the top.
- An optimistic-concurrency 409 ("please refresh") is expected under concurrent edits. It is a result, not a flaky retry.

See `examples/real-project-notes.md` for traps from a real project.
