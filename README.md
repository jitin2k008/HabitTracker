# Habit Tracker

A minimal, offline-first habit tracker that lives in a single HTML file. Add habits, choose the weekdays they matter, and check in each day — streaks, stats, and a 14-day history are computed for you.

No frameworks, no build step, no dependencies, no network requests. Open `index.html` and go.

> **Vibecoded project.** This app was built through a structured, AI-assisted workflow: a human-authored specification, phase-gated implementation by an AI coding agent, and human review at every step. [Read how it was built.](#how-this-was-built)

## Features

- **Habit scheduling** — choose the weekdays each habit is active (Mon–Sun); off-days show a disabled "Rest day" button.
- **One-tap check-in** — mark today done, or undo it; backfill or remove any past scheduled day from the history strip.
- **Streaks** — current streak per card, plus current, best, and total completions for each habit. Off-days are skipped, and an unfinished today never breaks a streak.
- **History** — 14 days per habit, with `‹ Earlier` / `Later ›` paging back up to 52 weeks. Click any past scheduled day (or a stray off-day completion) to toggle it.
- **Stats dashboard** — today's progress, highest current streak, highest best streak, and total completions.
- **Edit & delete** — rename a habit or change its active days without touching its history; delete with a confirmation prompt.
- **Light & dark themes** — follows your OS by default; the header toggle remembers your choice.
- **Local & private** — everything is stored in your browser's `localStorage`. Nothing leaves your device.
- **Accessible** — native buttons, inputs, and a `<dialog>`, with visible keyboard focus, ARIA labels, and pressed states.

## Quick start

1. Clone or download this repository.
2. Double-click `index.html` (or open it in your browser).

That's it — no install, server, or build step. The app works from `file://`.

## Deploy to GitHub Pages

1. Push the repository to GitHub.
2. Go to **Settings → Pages**.
3. Under **Build and deployment**, choose **Deploy from a branch**, select `main` and `/ (root)`, then **Save**.
4. Your tracker will be live at `https://<username>.github.io/<repository>/`.

## How this was built

This is a **vibecoded** project, and that was a deliberate choice rather than an accident. The application was written by an AI coding agent working from a human-authored specification (`SPEC.md`) instead of being typed out keystroke by keystroke. The workflow was structured to keep quality high:

- **Specification first.** Requirements, data formats, streak math, UI structure, and edge cases were written down before any code existed.
- **Phase-gated delivery.** The build was split into three explicit phases — core, depth, polish — with a review checkpoint between each one. No phase started until the previous one was confirmed.
- **Human review at every step.** The human author defined the requirements, inspected each change, and validated behavior with a browser test checklist after every phase.
- **Boring, portable technology.** The result is dependency-free vanilla HTML, CSS, and JavaScript in one file — easy to read, audit, host, and maintain.

Vibecoding, in this project, means faster iteration on a well-specified problem — not a shortcut on correctness. The app has no third-party dependencies, no build tooling, and no hidden network calls, and its data model is documented below so anyone can verify or extend it.

## Data model

State is stored under the key `habitTracker.v1`:

```json
{
  "habits": [
    {
      "id": "m1x2y3z4abc",
      "name": "Read",
      "activeDays": [1, 3, 5],
      "createdAt": "2026-10-29",
      "completions": ["2026-10-29"]
    }
  ]
}
```

- `activeDays` — unique weekday numbers, `0` = Sunday … `6` = Saturday, sorted ascending.
- `createdAt` and `completions` — local `YYYY-MM-DD` dates, never UTC.
- `completions` — unique dates, sorted ascending.

The theme preference is stored separately under `habitTracker.theme` as `"light"` or `"dark"`; when absent, the app follows the system setting.

If saved data is invalid, the raw string is copied to `habitTracker.v1.corrupt` and the app starts empty rather than crashing.

## How streaks work

- A day is **scheduled** when its weekday is in the habit's active days.
- A day is **done** when it appears in `completions`.
- Current streak: count backward from today; a missed scheduled day breaks the streak, off-days are skipped, and today not being done yet does not break it.
- Best streak: the longest run of completed days since the habit was created (or its earliest backfilled completion).
- Completions on off-days still count as +1.
- Streaks are always computed on the fly — never stored.

## Project structure

| File | Purpose |
| --- | --- |
| `index.html` | The entire app: markup, styles, and logic. |
| `SPEC.md` | The specification the app was built against. |
| `README.md` | This document. |

## Browser support

Current versions of Chrome, Edge, Firefox, and Safari. The app uses standard HTML, CSS custom properties, `<dialog>`, and `localStorage`.

## Privacy & limitations

- Data is per browser and per profile; there are no accounts, sync, or backups.
- Clearing site data or `localStorage` deletes your habits.
- If storage is blocked (for example, by browser settings), a banner appears and changes are kept in memory for the session only.
- The app makes no network requests and loads no external fonts, scripts, or styles.

## Contributing

Issues and pull requests are welcome. If you contribute, please keep the project's core constraint intact: the app stays a single, dependency-free `index.html` that works when opened directly from disk. Update `SPEC.md` alongside any behavioral change so the documentation and the implementation stay in sync.
