# ToppyTaskbar requirements

Status: **decision-complete** (no open questions). Date: 2026-10-19.

## 1. Problem and desired outcome

### Problem
- The user needs the Windows 11 taskbar at the top of the screen (accessibility, neck problems).
- Windows 11 removed this option. The user now uses Start11 for it. Start11 has many features the user does not need.
- Microsoft restored a native **Taskbar position** setting in KB5124008 (September 2026, Build 26100.9445 / 26200.9445). Microsoft releases it by Controlled Feature Rollout (CFR).
- The user has Windows 11 25H2, Build **26200.9457**. The update is installed, but the setting is **not visible**. Microsoft has not switched it on for this PC.

### Desired outcome
- The native Windows setting puts the taskbar at the top.
- ToppyTaskbar is a small helper that switches on the hidden native feature. It does not move the taskbar itself.
- After this, the user does not need Start11 to get a top taskbar.

## 2. Confirmed requirements

### R1. Form and delivery
- R1.1 One Windows `.exe` with a small dialog window (no console, no tray icon, no background process).
- R1.2 The user starts it on demand. It does not start with Windows and does not install a scheduled task or service.
- R1.3 A GitHub Actions workflow builds the `.exe` and attaches it to a GitHub release.
- R1.4 The `.exe` is unsigned. A SmartScreen warning on first run is accepted.
- R1.5 The `.exe` requests administrator rights at start (UAC prompt, manifest `requireAdministrator`).

### R2. Checks at start
- R2.1 Read the Windows build and revision (UBR).
- R2.2 Supported: Windows 11 client edition (product type workstation, not Server) with Build 26200 and UBR 9445 or higher. Do not query or write overrides on any other OS or build.
- R2.3 Read the current state of the taskbar-position feature ID (see R9). Read the ownership record (R4.5) to find out if this program wrote the override.
- R2.4 Find out if Start11 is running (process check).
- R2.5 Find out if a restart is pending: a pending marker (R4.4) exists and its boot session is the current boot session.

### R3. Status display
The window shows one clear status line and only the buttons that apply:

| State | Status text (meaning) | Buttons |
|---|---|---|
| Not supported (wrong OS, edition or build) | This Windows version is not supported. On Windows 11 25H2, install the September 2026 update first. No changes made. | Close |
| Supported, feature off | The top taskbar option is hidden on this PC. | Enable, Close |
| Restart pending (after Enable or Undo) | Restart to finish. | Restart now, Later |
| Feature on by this program | The option is available. If it is missing in Settings, Microsoft may have changed the feature. Use Undo. | Open taskbar settings, Undo, Close |
| Feature on by Microsoft (no override) | The option is available. This program changed nothing. | Open taskbar settings, Close |
| Override set by another tool (no ownership record) | The option was changed by another tool. This program does not change it. | Open taskbar settings, Close |

- R3.1 "Restart pending" has priority over all other states except "Not supported".
- R3.2 Show Undo only when the ownership record (R4.5) exists.

### R4. Enable
- R4.1 "Enable" sets the feature ID(s) in R9 to **enabled**, with the same mechanism ViVeTool uses (persistent override that survives restart).
- R4.2 After success, ask: **Restart now** or **Later**. "Restart now" restarts Windows. "Later" closes the dialog.
- R4.3 The program does not restart Windows without the user's confirmation.
- R4.4 After a successful Enable or Undo, store a pending marker with the current boot session identifier (for example the system boot time) under `HKLM\SOFTWARE\ToppyTaskbar`. At start, delete the marker if its boot session is not the current one.
- R4.5 Before Enable writes, store a durable ownership record under `HKLM\SOFTWARE\ToppyTaskbar`: each feature ID it writes and that ID's previous override value (or "none"). The record stays until Undo succeeds. If a record already exists (for example after a Windows update removed the override), keep its original previous values.
- R4.6 Enable is all or nothing. If any write fails, restore the previous value of every ID already written, delete the ownership record and show the error. If the restore also fails, keep the ownership record, show the error with the IDs and their previous values, and show Undo so the user can try again.

### R5. Set the taskbar to top
- R5.1 The program does **not** write the taskbar position itself.
- R5.2 "Open taskbar settings" opens `ms-settings:taskbar`. The user selects **Top** under Taskbar behaviors > Taskbar position.
- R5.3 When the user starts the program after the restart and the feature is on, the program shows the "Open taskbar settings" button (R3).

### R6. Undo
- R6.1 "Undo" restores, for each ID in the ownership record, its previous value: it removes the override if the previous value was "none", or writes the previous override back. It does not force IDs to "disabled". Then it deletes the ownership record.
- R6.2 After success, ask: **Restart now** or **Later** (same as R4.2).
- R6.3 The status text tells the user to set the taskbar position back to Bottom in Settings first, if they want the bottom position.
- R6.4 Undo never changes an ID that is not in the ownership record.

### R7. Start11
- R7.1 If Start11 is running, show a warning: "Start11 is running. Turn off its taskbar features, or the taskbar can behave incorrectly."
- R7.2 The warning does not block Enable or Undo.
- R7.3 The program does not change, stop or uninstall Start11.

### R8. Accessibility
- R8.1 All functions work with the keyboard only (Tab order, Enter, Esc closes).
- R8.2 Screen readers read the status text and all buttons (standard Win32 or WinForms/WPF controls with names).
- R8.3 The window supports high contrast and display scaling up to 200% (per-monitor DPI aware).
- R8.4 Large, clear buttons. Plain text without technical terms in the main status line.

### R9. Feature IDs
- R9.1 Default: override only **59213768** (Taskbar position setting).
- R9.2 Do not override bundle or unrelated IDs: **61161244** (update bundle, many features) and **61754985** (Start menu customization).
- R9.3 Exception: if the release test on the target PC (R9.4) shows that 59213768 alone does not show the option, add 61161244 as a dependency. Document this in the release notes. Never add Start menu IDs.
- R9.4 Before the first release, test on the target PC (Build 26200, the only supported build): enable, restart, confirm that "Taskbar position" shows, then Undo and confirm that it is hidden again.
- R9.5 Keep all IDs as constants in one place in the code.

## 3. Preferences and examples (not requirements)
- **ViVeTool**: an example of the mechanism. The program can use its own code or a library. If it bundles ViVeTool code, its license (GPL-3.0) must be compatible with this repository (AGPL-3.0).
- **Start11, Windhawk "Taskbar on top", ExplorerPatcher**: references only. Do not copy their approach (code injection into Explorer).
- **Language and UI framework**: free choice of the coding agent (for example C# WinForms or C++ Win32). It must produce one `.exe` that runs on a standard Windows 11 PC without extra installs.
- **Exact wording** of status texts: examples. Keep the meaning and the plain-language style.

## 4. Non-goals
- No code injection or hooks into `explorer.exe`.
- No own taskbar or shell replacement.
- No automatic write of the taskbar position value (undocumented storage).
- No auto start, scheduled task, service or tray icon.
- No re-check after Windows updates in the background.
- No taskbar size, look, Start menu or multi-monitor settings.
- No change to Start11 (no detection beyond "is it running").
- No installer. No code signing.
- No support for Windows 10, Windows Server, Windows 11 24H2 (Build 26100) or builds below R2.2.

## 5. Edge cases and failure behavior

| Case | Behavior |
|---|---|
| User declines UAC | Program does not start. Windows handles this. |
| Build below the minimum | Show "update Windows" status. Make no changes. |
| Feature already on (CFR reached the PC) | Show "Feature on by Microsoft". Do not write overrides. No Undo. |
| Started again after "Later", before restart | Show "Restart pending" (R2.5). Do not show "available". |
| Pending marker from an earlier boot | Delete it. Continue with normal checks. |
| Writing the overrides fails | Roll back per R4.6. Show an error with the system error code. |
| Windows Server or other non-client OS | Show "not supported". Make no changes. |
| After restart, the option is still missing | No reliable automatic check exists. The "Feature on by this program" status text tells the user what to do (use Undo). |
| A Windows update removes the overrides | User runs the program again. It shows "feature off". |
| Microsoft changes the feature IDs in a later build | Program shows "feature off" or the missing-option status. The IDs are constants in one place in the code, so a new release can change them. |
| Start11 running | Warning (R7). Continue. |
| Restart now fails | Show an error and tell the user to restart manually. |
| Program started twice | Second instance brings the first window to the front, or exits. |

## 6. Confirmed constraints
- Target PC: Windows 11 25H2, Build 26200.9457.
- Feature ID: 59213768 for Taskbar position (sources: Allthings.how, Pureinfotech, phantomofearth). Windows Latest names 61161244 and 61754985 instead; R9 resolves this. Verify per R9.4.
- Admin rights and a restart are necessary to change feature overrides.
- Repository license: AGPL-3.0.
- Build in GitHub Actions. Output: unsigned `.exe` in a GitHub release.
- Architecture: x64 (default chosen for the target PC; see deferred ideas).
- No network access at run time.

## 7. Deferred ideas
- ARM64 build.
- Support for Windows 11 24H2 (Build 26100), after a separate test on that build.
- Decision to keep or uninstall Start11 (user decides after the native top taskbar works).
- Optional check at sign-in that re-enables the feature after Windows updates.
- Automatic write of the "Top" position, if Microsoft documents it.
- Code signing to remove the SmartScreen warning.
- Remove the program when Microsoft enables the feature for all PCs (the program then shows "available" and has no more use).

## 8. Open questions
None.

## Sources
- [Windows Latest: Windows 11 restores the movable taskbar (KB5124008)](https://www.windowslatest.com/2026/09/09/windows-11-finally-restores-a-windows-10-feature-after-five-years-the-movable-taskbar/)
- [NTCompatible: KB5120998 preview, taskbar position](https://www.ntcompatible.com/story/windows-11-august-2026-update-kb5120998-preview-taskbar-finally-moves-24h2-eol-looms/)
- [Allthings.how: KB5120998 hidden features and flag list](https://allthings.how/windows-11-kb5120998-unlock-the-update-s-hidden-features-early/)
- [phantomofearth on X: taskbar position flag 59213768](https://x.com/phantomofearth/status/2055393218703835163)
- [Ramen Software: Windhawk mod "Taskbar on top for Windows 11"](https://ramensoftware.com/windhawk-mod-taskbar-on-top-for-windows-11)
