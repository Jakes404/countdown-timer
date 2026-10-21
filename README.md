# AHMP Aftelhorlosie (countdown timer)

An exam countdown timer for AHMP classrooms and exam venues, shown on a
projector. It's a single self-contained `index.html`, with no build step,
no server and no dependencies apart from Google Fonts (Nunito, DM Sans), which
fall back to system fonts when offline.

The interface is in Afrikaans; this README and the code comments are in English.

Current version: **1.8** (shown bottom left on the page, `#version-tag`).

## Where it lives

| Copy | Status |
|---|---|
| This repo, `index.html` | **Source of truth.** Make changes here. |
| Kompas Tools page `/t/aftelhorlosie` | Live copy, embedded on the school website. Re-upload `index.html` in Kompas after every change. |
| `AHMP/tools/aftelhorlosie-standalone.html` (local, not in git) | Copy of this repo's `index.html`. Don't edit; copy over it after changes. |
| [`vibe-code-tools/countdown-timer.html`](https://github.com/Jakes404/vibe-code-tools) | Copy of this repo's `index.html` with CRLF line endings. Don't edit; copy over it after changes. |
| `tools-platform/public/countdown.html` | Separate, older rewrite. Not kept in sync. |

## Features

- **Exam time**: presets 30–180 min, or a custom number of minutes.
- **Reading time** (Leestyd): optional, 5/10/15/20 min or custom. It counts down
  first (pink), then the exam time starts automatically.
- **Second group** (Tweede groep): for two groups in one venue with different
  exam times, e.g. English HL with 30 min more than English FAL. One Begin
  starts both; pause pauses both; reading time is shared. Each group gets its
  own clock, name and "Eindig om HH:MM" end time. When one group finishes, its
  clock shows "Tyd Verstreke!" while the other keeps running.
- **Extra time** (+5 min / −5 min): during exam time, +5 min adds 5 minutes and
  −5 min takes them off again after a mistaken click. One group: the buttons sit
  either side of the pause button. Two groups: each card has its own pair. They
  also show in fullscreen (and hide with the other controls). The buttons
  disappear once that group's time is up. −5 only works while there is added
  time to take off and the group would still have time left. The clock shows
  "+5 min bygevoeg" next to the end time. The "Nog … minute" warnings and the
  orange/red colours follow the new time left; whether the exam counts as
  short (under 45 min) still goes by the planned length. Herstel drops it.
- **Fullscreen** (Volskerm): clock only, sized for a projector. The exit and
  pause buttons and the cursor hide after 3 s without movement; moving the mouse
  or tapping shows them again.
- **Warnings**: "Nog 30 / 15 / 5 minute" flashes for 20 s. The clock turns
  orange in the last 15 min and red in the last 5 min (for exams shorter than
  45 min: 5 min and 1 min).
- **Safety during an exam**:
  - Time settings are locked once Begin is pressed (until Herstel). Extra time
    goes through the +5 / −5 buttons instead.
  - Herstel needs a second click within 4 s while a timer is running. This is an
    in-page confirmation, not `confirm()`, because sandboxed iframes may block
    modals.
  - The screen is kept awake (Screen Wake Lock API) while running.
  - The timer survives a refresh, a crash or a closed tab (see *Saved state*).
- **Help**:
  - The Hulp button opens an 8-step guide with a colour key.
  - "i" buttons explain Toetstyd, Leestyd and Tweede groep (hover, or tap on
    touchscreens).
  - Most controls show a hover tip (`data-tip` attribute).
- **Keyboard and accessibility**:
  - Space starts and pauses; Escape closes tips, the help dialog and the fallback
    fullscreen.
  - The switches are `role="switch"` buttons.
  - Warnings are announced through an `aria-live` region.
- **Layout**: at 1000px and wider, the clock is on the left and the settings are
  in a 400px column on the right (it scrolls on its own if needed). Narrower
  screens stack them. The clocks scale with the clock area through container
  query units (`cqi`).

## How it works

Everything is in the one `<script>` block at the bottom of `index.html`.

**Timing.** Time is measured from the wall clock, never by counting ticks, so it
can't drift when the browser throttles timers:

- `banked`: ms elapsed in earlier runs (before the last pause).
- `runStartedAt`: wall-clock ms when the current run started.
- `elapsed() = banked + (running ? Date.now() - runStartedAt : 0)`.

Reading time and every group share that one elapsed value. A group ends at
`readingMs + group.total + group.extra`, and its time left is that end minus
`elapsed()`. `extraSecs` holds the time added per group; it's kept apart from
`examTotal` / `group2Total` so Herstel drops it and the presets stay as set.
`tick()` runs every 250 ms: it stops the timer once every group is done, then
calls `render()`. The display is fully derived from state, and `setText` /
`setClass` / `setHTML` only touch the DOM when a value changes.

**Phases** are derived, not stored:

| Phase | Condition |
|---|---|
| Idle | `!started` |
| Reading | `elapsed() < readingMs` |
| Exam | between reading and a group's end |
| Done | `elapsed()` is past every group's end |

**Views.** `render()` fills two views with the same data: the normal page and
the fullscreen overlay (`views[]`). The single big clock shows reading time, or
the only group's time. The two-card layout (`.cards`) is used when the second
group is on and reading time is over or off.

**Saved state.** Settings, names, timer state and extra time are saved to `localStorage`
under `ahmp-aftelhorlosie` on every change. They're restored on load and ignored
if older than 12 hours. All storage access is wrapped in `try/catch`, so the
timer still works where storage is blocked (e.g. a sandboxed iframe without
`allow-same-origin`). The same goes for the wake lock.

## Changing things

The settings are constants near the top of the script:

| Constant | Meaning |
|---|---|
| `PRESETS` | Exam time buttons (minutes); used for group 1 and group 2. |
| `READING_PRESETS` | Reading time buttons (minutes). |
| `WARNING_MARKS` | Minutes left that show a "Nog … minute" warning. |
| `WARNING_SECS` | How long each warning shows. |
| `EXTRA_MINS` | Minutes per +/− click (button labels and tips follow it). |
| `STORE_KEY`, `STALE_MS` | Saved-state key and maximum age. |
| `warnLevel()` | Orange and red thresholds. |

Other places to change:

- **Help text**: the Hulp dialog is the `#help` markup; the tips are the
  `data-tip` attributes in the HTML.
- **Colours**: CSS variables in `:root`. `--pink` is the reading-time accent.
- **Version**: update the `#version-tag` text when you release.

## Testing checklist

Serve the file locally (e.g. `python3 -m http.server`) and check:

1. Single group: Begin, pause, resume and Herstel (needs two clicks while running).
2. Reading time on: the pink reading countdown flows straight into the exam time.
3. Two groups with reading time: the cards appear after reading, both end times
   are right, and the first group shows "Tyd Verstreke!" while the second keeps
   going.
4. Refresh the page while running and while paused: the time carries on correctly.
5. Extra time: +5 moves the clock and end time on; −5 takes it back and is
   greyed out with no added time or under 5 min left; the buttons go when the
   time is up; with two groups each card's buttons only change that group;
   Herstel clears it and the preset stays as set.
6. Volskerm: the controls (including +5 / −5) and cursor hide after 3 s; Esc and
   Verlaat exit.
7. Window widths of about 1440, 800 and 375px: nothing overflows, and the
   settings sit beside the clock only at 1000px and wider.
8. After uploading to Kompas, repeat steps 1, 4 and 5 inside the embed.

To fast-forward in the browser console: `runStartedAt -= 10 * 60000` jumps
10 minutes ahead.

## Known gaps

- It isn't on the AHMP design kit (v1.2): it uses green, orange, Nunito and
  DM Sans instead of the kit's colours and Proxima Nova.
- It hasn't been tested inside the Kompas sandbox; saved state and the wake lock
  may be blocked there and quietly turn off.
- The two-group mode supports exactly two groups. The state model
  (`groups()` + `groupEndMs`) generalises, but the UI and the card markup
  assume two.
