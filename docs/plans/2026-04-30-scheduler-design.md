# Mahjong Copilot Scheduler Design

Date: 2026-04-30
Scope: Add scheduling for autoplay/autojoin with browser restart. No architecture changes.

## Goals
- Support two scheduling modes: fixed daily window and rotating cycle.
- Allow configuration from GUI with persistence.
- When schedule closes, stop autoplay/autojoin and close Playwright after the current game ends.
- When schedule opens, restart Playwright and re-enable autoplay/autojoin.

## Architecture Overview
Scheduling lives in the GUI layer and runs via tkinter `after` checks. It triggers BotManager actions to
start or stop autoplay and autojoin, and to open or close the Playwright browser. For closing, BotManager
uses a pending shutdown flag to defer closing the browser until the current game ends.

Key modules and responsibilities:
- `gui/main_gui.py`: schedule state, timer loop, user input, status display.
- `bot_manager.py`: enable/disable autoplay/autojoin, start browser, and handle deferred shutdown.
- `game/browser.py`: Playwright lifecycle (`start`, `stop`).
- `common/settings.py`: schedule configuration persistence.

## Scheduling Modes

### Fixed Daily Window
- User sets `start_time` and `end_time` (HH:MM, local time).
- Supports cross-day windows (e.g., 22:00 -> 06:00).
- Open condition:
  - If `start <= end`: open when `start <= now < end`.
  - If `start > end` (cross-day): open when `now >= start` OR `now < end`.

### Rotating Cycle
- User sets `on_hours` and `off_hours`.
- Scheduler keeps `next_switch_at` timestamp and `current_state` (on/off).
- If current time passes `next_switch_at`, immediately switch and recompute next time.
- `next_switch_at` persists so restarts do not reset the cycle.

## Behavior Rules

### Schedule Open
1) `start_browser()`
2) `enable_automation()`
3) `enable_autojoin()`

### Schedule Close
1) `disable_automation()`
2) `disable_autojoin()`
3) Set `pending_browser_shutdown = True`
4) If not in game: `browser.stop(True)` immediately
5) If in game: close browser in `_process_end_game()` when the game ends

### Deferred Browser Shutdown
- Add `pending_browser_shutdown` to BotManager.
- In `_process_end_game()`, if flag set, stop Playwright and clear flag.

## Settings (Persisted)
Recommended fields in `common/settings.py`:
- `schedule_enabled: bool`
- `schedule_mode: str`  # none | fixed | rotate
- `fixed_start_time: str`  # HH:MM
- `fixed_end_time: str`    # HH:MM
- `rotate_on_hours: float`
- `rotate_off_hours: float`
- `rotate_next_switch_at: str`  # ISO timestamp
- `rotate_state_on: bool`

## GUI Plan
- Add a small scheduling panel near existing AutoJoin timer controls.
- Controls:
  - Mode selector (None / Fixed / Rotate)
  - Fixed: start HH:MM, end HH:MM
  - Rotate: on hours, off hours
  - Enable checkbox
- Status line showing: current schedule state (on/off/waiting shutdown), next switch time.

## Validation Checklist
- Fixed window works in same-day and cross-day intervals.
- Rotate mode switches correctly even after sleep or restart.
- Close during an active game waits for end-game before closing browser.
- Open starts browser then enables autoplay/autojoin.
- Settings persist and restore scheduler state on restart.
