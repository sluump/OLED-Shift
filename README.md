OLED Shift
AutoHotkey v2 Window Drift Script
Burn-in protection for OLED displays
Overview
OLED Shift is a background AutoHotkey v2 script that silently moves windows on a target monitor in a slow, repeating orbit. Its purpose is to prevent OLED burn-in — the permanent pixel degradation that occurs when a static image such as a taskbar, sidebar, or pinned application window remains in the exact same screen position for extended periods.

The script runs invisibly in the system tray and requires no interaction during normal use. Shifts are small enough (5 pixels per step) that they are imperceptible during regular work. Drift runs on a fixed schedule regardless of whether the user is active — windows always move when they are due, ensuring continuous protection during long sessions.

Requirements
AutoHotkey v2.0 or later must be installed. The script will prompt for administrator privileges at launch — elevation is required to move windows owned by other elevated processes such as Task Manager or system utilities.

How It Works
The Drift Orbit
Each tracked window follows an eight-step square orbit around its home position. The orbit visits all eight compass directions — up, up-right, right, down-right, down, down-left, left, and up-left — before repeating. At the default step size of 5 pixels, the total orbit footprint is 40 pixels, which is large enough to meaningfully shift static UI content off its worn zone while remaining imperceptible during use.

Every window maintains its own independent position in the orbit with its own randomised schedule, so windows never all move at the same time.

Timing and Randomisation
The script scans for windows every 10 seconds. Each window drifts on its own timer — once a window has been still for at least driftInterval (default 10 minutes), it is moved on the next scan. After each shift, the next drift is scheduled at driftInterval plus a random additional delay of 0 to 10 minutes.

This randomisation keeps windows permanently out of phase with each other. Even if several windows are opened at the same time, their drift schedules quickly diverge so moves are spread across the session rather than clustering together.

Always-On Drift
A window that has been sitting in the same position for 10 minutes will be moved regardless of what the user is doing on the machine. This ensures continuous burn-in protection during long uninterrupted sessions such as video watching, coding, or meetings.

The only condition that pauses drift is a genuine fullscreen application on the target monitor, such as a game or video player. In that case moving windows underneath would be invisible anyway, so drift is held until the fullscreen app exits.

Manual Move Detection
If the user manually repositions a window by dragging it, the script detects the position change on the next scan and resets that window's home position and drift timer. The orbit restarts cleanly from the new location after the next drift interval.

Window Filtering
Not every window on the system is tracked. The script applies the following filters before registering a window:

•	Shell and desktop infrastructure is excluded — Program Manager, wallpaper worker, and taskbar classes are never moved.
•	Windows without the WS_VISIBLE style flag are skipped.
•	Windows smaller than 100 × 50 pixels are ignored, covering tooltips, ghost windows, and similar transient elements.
•	Untitled windows are skipped as they are typically background or system processes.
•	A window is only managed by the monitor whose bounds contain the window's center point, preventing dual-monitor overlap issues.

Monitor Targeting
The script targets a single monitor only. By default it resolves the primary display automatically using the Windows monitor API, which correctly handles setups where the primary monitor is not assigned index 1. A specific monitor index can be hardcoded in the configuration if needed.

Configuration
All settings are at the top of the script file. No other part of the script should need to be edited for normal use.

Setting	Default	Description
targetMonitor	0	Monitor to protect. 0 = always resolve to the primary monitor. Set to a specific index (1, 2, 3...) to hardcode a display. Use Ctrl+Shift+F10 to list available indices.
checkInterval	10000	How often (ms) the script scans for windows due for a drift step. Default is 10 seconds.
driftInterval	600000	How long (ms) a window must be still before it is moved. Drift fires regardless of user activity. Default is 10 minutes (600,000 ms).
driftStep	5	Pixels moved per drift step. At 5px the orbit covers 40px total. Keep between 1–8 for imperceptible shifts.
maxLogSizeKB	1000	Log file size limit in KB. When exceeded the current log is archived to .bak and a new file is started. Default is 1 MB.
logFile	A_ScriptDir\Logs	Path to the log file. Defaults to a Logs\ subfolder next to the script so no path configuration is needed on other machines.

Tip: Finding your monitor index
If you are not sure which index corresponds to your display, press Ctrl+Shift+F10 or use the tray menu to open the monitor list. It shows every monitor's index, name, pixel bounds, and which is the primary.

User Interface
System Tray
OLED Shift runs in the system tray. Right-clicking the tray icon opens a menu with the following options:

Menu item	Action
OLED Shift — Monitor N	Disabled header showing which monitor is currently targeted. Not clickable.
Force drift now	Immediately moves all tracked windows one orbit step, bypassing the fullscreen guard. Useful for testing.
Open log file	Opens the current log in the default text editor. Shows a message if no log has been written yet.
List monitors	Opens a message box listing all connected monitors with their index, name, and screen bounds.
Exit	Stops the script.

Hotkeys

Hotkey	Action
Ctrl + F10	Force drift — immediately moves all tracked windows one orbit step, bypassing the fullscreen guard.
Ctrl + Shift + F10	List monitors — opens the monitor index helper.

Log File
All activity is written to a plain text log file with 24-hour timestamps. The log is stored in a Logs\ subfolder next to the script by default and is created automatically on first run. When the file exceeds 1 MB it is rotated: renamed to .bak and a fresh log is started. Only one backup is kept at a time.

Log Events
Event	What it means
SESSION START	Script launched. Shows the resolved monitor index.
STARTUP	Monitor auto-detection result showing which index the primary resolved to.
REGISTERED	A new window was found on the target monitor and added to the tracking database. Shows title, process, position, and size.
DRIFT QUEUED	A window is about to be moved. Shows title, process, step number, source and destination coordinates, and distance.
DRIFT STEP	Confirms a window was moved and shows the new step number and distance.
FORCE DRIFT QUEUED	Same as DRIFT QUEUED but triggered by the force drift action rather than the timer.
FORCE DRIFT DONE	Confirms a forced move completed and shows the new step position.
MANUAL MOVE	A window moved more than 2px without the script moving it. Home position has been reset and the drift timer restarted.
BLOCKED: FULLSCREEN	Drift suppressed because a fullscreen window is covering the target monitor. Only logged once per suppression period.
FULLSCREEN DETECTED	Details of the window that triggered the fullscreen block — title, process, class, position, and size vs monitor dimensions.
ERROR	A window failed to move (e.g. a sandboxed or UWP app). Only logged once per window to avoid flooding.
LOG ROTATED	The log exceeded the size limit and was archived to .bak.

Notes
OLED Effectiveness
At 5px per step the total orbit covers a 40-pixel square footprint. This is sufficient to prevent static burn-in from most desktop UI elements — taskbars, sidebars, notification areas, and docked application windows. The default 10-minute drift interval with up to 10 minutes of additional random variance means any given pixel will be shifted off its position regularly throughout a long session.

Admin Elevation
Administrator rights are requested at startup so the script can move windows owned by elevated processes such as Task Manager, registry editors, and system utilities. If elevation is denied the script continues running but may be unable to move those specific windows. Failures are logged once per window.

UWP and Sandboxed Apps
Some Universal Windows Platform (UWP) applications and sandboxed processes reject WinMove calls regardless of elevation level. The script detects this on the first failed attempt, logs a single error for that window, and does not attempt to move it again. This does not affect other tracked windows.

First Run
On first launch the script creates the Logs\ directory automatically and validates that the configured monitor index exists. If the monitor is not found the script exits with a clear error message rather than running silently against the wrong display. Use Ctrl+Shift+F10 to confirm your monitor index before adjusting the targetMonitor setting.

