# OLED Shift

**AutoHotkey v2 Window Drift Script — burn-in protection for OLED displays**

![Platform](https://img.shields.io/badge/platform-Windows%2010%2F11-blue)
![AutoHotkey](https://img.shields.io/badge/AutoHotkey-v2.0%2B-green)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## Overview

OLED Shift is a background AutoHotkey v2 script that silently moves windows on a target monitor in a slow, repeating orbit. Its purpose is to prevent OLED burn-in — the permanent pixel degradation that occurs when a static image such as a taskbar, sidebar, or pinned application window remains in the exact same screen position for extended periods.

The script runs invisibly in the system tray and requires no interaction during normal use. Shifts are small enough (5 pixels per step) that they are imperceptible during regular work. Drift runs on a fixed schedule regardless of whether the user is active — windows always move when they are due, ensuring continuous protection during long sessions.

> **Requirements** — AutoHotkey v2.0 or later must be installed. The script will prompt for administrator privileges at launch — elevation is required to move windows owned by other elevated processes such as Task Manager or system utilities.

---

## Installation

1. Install **[AutoHotkey v2](https://www.autohotkey.com/)** if you haven't already.
2. Download this repository and keep the folder structure intact:

```
OLED_Shift\
    OLED_Shift.ahk
    ICON\
        OLED_Shift.ico
    Logs\              ← created automatically on first run
```

3. Double-click `OLED_Shift.ahk` to run it.
4. Accept the UAC prompt for administrator rights.
5. A tray icon appears — the script is now active.

**Optional — run at startup:**
Press `Win + R`, type `shell:startup`, press Enter, and drop a shortcut to `OLED_Shift.ahk` in that folder.

---

## How It Works

### The Drift Orbit

Each tracked window follows a sixteen-step square orbit around its home position. The orbit walks all eight compass directions — up, up-right, right, down-right, down, down-left, left, and up-left — twice before repeating. At the default step size of 5 pixels, the orbit covers a 10-pixel radius square, which is sufficient to meaningfully shift static UI content off its worn zone while remaining imperceptible during use.

Every window maintains its own independent position in the orbit with its own randomised schedule, so windows never all move at the same time.

### Timing and Randomisation

The script scans for windows every 10 seconds. Each window drifts on its own timer — once a window has been still for at least `driftInterval` (default 10 minutes), it is moved on the next scan. After each shift, the next drift is scheduled at `driftInterval` plus a random additional delay of 0 to 10 minutes.

This randomisation keeps windows permanently out of phase with each other. Even if several windows are opened at the same time, their drift schedules quickly diverge so moves are spread across the session rather than clustering together.

### Always-On Drift

A window that has been sitting in the same position for 10 minutes will be moved regardless of what the user is doing. This ensures continuous burn-in protection during long uninterrupted sessions such as video watching, coding, or meetings.

The only condition that pauses drift is a genuine fullscreen application on the target monitor (a game, video player, etc.). Drift is held until the fullscreen app exits.

### Manual Move Detection

If you manually reposition a window by dragging it, the script detects the position change on the next scan and resets that window's home position and drift timer. The orbit restarts cleanly from the new location after the next drift interval.

### Window Filtering

Not every window on the system is tracked. The script applies the following filters before registering a window:

- Shell and desktop infrastructure is excluded — Program Manager, wallpaper worker, and taskbar classes are never moved.
- Windows without the `WS_VISIBLE` style flag are skipped.
- Windows smaller than 100 × 50 pixels are ignored (tooltips, ghost windows, etc.).
- Untitled windows are skipped — typically background or system processes.
- A window is only managed by the monitor whose bounds contain the window's centre point, preventing dual-monitor overlap issues.

### Monitor Targeting

The script targets a single monitor. By default it resolves the primary display automatically using the Windows monitor API, which correctly handles setups where the primary monitor is not assigned index 1. A specific monitor index can be hardcoded in the configuration if needed.

---

## Configuration

All settings are at the top of `OLED_Shift.ahk`. No other part of the script needs to be edited for normal use.

| Setting | Default | Description |
|---|---|---|
| `targetMonitor` | `0` | Monitor to protect. `0` = always resolve to the primary monitor. Set to a specific index (`1`, `2`, `3`...) to hardcode a display. Use **Ctrl+Shift+F10** to list available indices. |
| `checkInterval` | `10000` | How often (ms) the script scans for windows due for a drift step. Default is 10 seconds. |
| `driftInterval` | `600000` | How long (ms) a window must be still before it is moved. Drift fires regardless of user activity. Default is 10 minutes. |
| `driftStep` | `5` | Pixels moved per drift step. At 5 px the orbit covers a 10 px radius. Keep between 1–8 for imperceptible shifts. |
| `maxLogSizeKB` | `1000` | Log file size limit in KB. When exceeded the log is archived to `.bak` and a fresh file is started. |
| `logFile` | `A_ScriptDir\Logs` | Path to the log file. Defaults to a `Logs\` subfolder next to the script — no path setup needed on other machines. |

> **Tip — finding your monitor index:** Press **Ctrl+Shift+F10** or use the tray menu to open the monitor list. It shows every monitor's index, name, pixel bounds, and which is the primary.

---

## User Interface

### System Tray

Right-click the tray icon to open the menu:

| Menu item | Action |
|---|---|
| **OLED Shift — Monitor N** | Disabled header showing which monitor is currently targeted. |
| **Force drift now** | Immediately moves all tracked windows one orbit step, bypassing the fullscreen guard. Useful for testing. |
| **Open log file** | Opens the current log in the default text editor. |
| **List monitors** | Opens a message box listing all connected monitors with index, name, and bounds. |
| **Exit** | Stops the script. |

### Hotkeys

| Hotkey | Action |
|---|---|
| `Ctrl + F10` | Force drift — immediately moves all tracked windows one orbit step. |
| `Ctrl + Shift + F10` | List monitors — opens the monitor index helper. |

---

## Log File

All activity is written to a plain-text log with 24-hour timestamps, stored in a `Logs\` subfolder next to the script. It is created automatically on first run. When the file exceeds 1 MB it is rotated: renamed to `.bak` and a fresh log is started. Only one backup is kept at a time.

### Log Events

| Event | What it means |
|---|---|
| `SESSION START` | Script launched. Shows the resolved monitor index. |
| `STARTUP` | Monitor auto-detection result — which index the primary resolved to. |
| `REGISTERED` | A new window was found on the target monitor and added to the tracking database. Shows process, position, and size. |
| `DRIFT` | A window was moved. Shows process, step number, source and destination coordinates, and distance. |
| `FORCE DRIFT` | Same as `DRIFT` but triggered by the force drift action rather than the timer. |
| `MANUAL MOVE` | A window moved more than 2 px without the script moving it. Home position reset and drift timer restarted. |
| `BLOCKED: FULLSCREEN` | Drift suppressed because a fullscreen window covers the target monitor. Logged once per suppression period. |
| `ERROR` | A window failed to move (e.g. a sandboxed or UWP app). Logged once per window to avoid flooding. |
| `LOG ROTATED` | The log exceeded the size limit and was archived to `.bak`. |

---

## Notes

### OLED Effectiveness

At 5 px per step the orbit covers a 10-pixel radius square over 16 steps. This is sufficient to prevent static burn-in from most desktop UI elements — taskbars, sidebars, notification areas, and docked application windows. The default 10-minute drift interval with up to 10 minutes of additional random variance means any given pixel will be shifted off its position regularly throughout a long session.

### Admin Elevation

Administrator rights are requested at startup so the script can move windows owned by elevated processes such as Task Manager, registry editors, and system utilities. If elevation is denied the script continues running but may be unable to move those specific windows. Failures are logged once per window.

### UWP and Sandboxed Apps

Some Universal Windows Platform (UWP) applications and sandboxed processes reject `WinMove` calls regardless of elevation level. The script detects this on the first failed attempt, logs a single error for that window, and does not attempt to move it again. This does not affect other tracked windows.

> **First Run** — On first launch the script creates the `Logs\` directory automatically and validates that the configured monitor index exists. If the monitor is not found the script exits with a clear error message rather than running silently against the wrong display. Use **Ctrl+Shift+F10** to confirm your monitor index before adjusting the `targetMonitor` setting.

---

## License

MIT — do whatever you want with it.
