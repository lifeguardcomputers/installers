# App Updater test guide

For the technician testing on a **Windows 11 x64** PC and a **Windows 11 ARM64** PC.
Channel `test`, sequence 2. This test manifest expires on 22 February 2027; after that the
PCs refuse it until a new one is published.

What gets installed:
- PDF Reader (pdf-reader) version 3.6.1
- Documents and Spreadsheets (office-suite) version 26.8.3.1
- Photo Album (photo-album) version 0.7.34-pa.1

Use a test PC, not a customer's. Below, `<arch>` is `x64` or `arm64` to match the PC
(Settings, System, About, "System type").

## Quick way: OfficeApps-Setup.exe

In person, the simplest way is the setup program: download https://raw.githubusercontent.com/lifeguardcomputers/installers/office-apps-test/LG-OfficeApps/OfficeApps-Setup.exe and run it
(it is not code-signed: if Windows shows "Windows protected your PC", click **More info**, then **Run anyway**).
Click through it; it asks for administrator rights, lets you untick apps, shows its progress, and
offers to open the log at the end. It runs the same script as the RMM (section 7), so it does
steps 1 to 3 below in one go, with the same result. `OfficeApps-Setup.exe /S` runs it without any
windows and returns the same exit codes as the script.

The steps below do the same by hand and show more of what happens.

## 1. Download the updater

In the browser, download the file for this PC:

- x64: https://raw.githubusercontent.com/lifeguardcomputers/installers/office-apps-test/LG-OfficeApps/app-updater/0.1.0-test.14/appupdater-x64.exe
- ARM64: https://raw.githubusercontent.com/lifeguardcomputers/installers/office-apps-test/LG-OfficeApps/app-updater/0.1.0-test.14/appupdater-arm64.exe

It is a test build and is not code-signed. If the browser warns, choose Keep. If Windows shows
"Windows protected your PC" (SmartScreen) when it runs, click **More info**, then **Run anyway**.
If it is blocked without that choice, right-click the file, choose Properties, tick **Unblock**, and click OK.

## 2. Set it up

Open an **elevated** Command Prompt (Start, type `cmd`, right-click, Run as administrator),
go to the Downloads folder (`cd %USERPROFILE%\Downloads`) and run (one line):

    appupdater-<arch>.exe setup --manifest-url https://raw.githubusercontent.com/lifeguardcomputers/installers/office-apps-test/LG-OfficeApps/manifest.test.json --channel test --enroll pdf-reader,office-suite,photo-album

**In PowerShell instead** (right-click Start, then Terminal (Admin)), put `.\` in front of the file name:

    .\appupdater-<arch>.exe setup --manifest-url https://raw.githubusercontent.com/lifeguardcomputers/installers/office-apps-test/LG-OfficeApps/manifest.test.json --channel test --enroll pdf-reader,office-suite,photo-album

The commands are plain words (`setup`, `check`, `status`), not `-setup`. Double-clicking the file does nothing useful:
it is a command-line tool. If setup stopped partway on an earlier try, just run it again.

It prints each step and should end with exit code 0 (`echo %errorlevel%`). It installs itself to
`C:\Program Files\AppUpdater\appupdater.exe`, keeps its settings in `C:\ProgramData\AppUpdater`, and adds
two scheduled tasks, `AppUpdater` (weekly at 03:00) and `AppUpdater-Logon` (15 minutes after logon).

## 3. Run a check, then look at the status

In the same elevated prompt:

    "C:\Program Files\AppUpdater\appupdater.exe" check
    "C:\Program Files\AppUpdater\appupdater.exe" status

In PowerShell, put `&` in front of a quoted path:

    & "C:\Program Files\AppUpdater\appupdater.exe" check
    & "C:\Program Files\AppUpdater\appupdater.exe" status

The first check downloads and installs all three apps, which can take a while (the office suite
is large). Plug a laptop into power. `status` shows `lastSequence: 2`, the last result and
the last 20 log lines. Run `check` a second time: every app should now be `up-to-date`, with no reinstall.
To run the check the way the PC will on its own (as SYSTEM), use `schtasks /Run /TN AppUpdater` instead.

## 4. What to look at for each app

Per-app results are in the log (`app-result` lines: `installed`, then `up-to-date`) and in
`type C:\ProgramData\AppUpdater\state.json` (`postInstallOk: true` for the two apps that have a postinstall).

- **PDF Reader** (SumatraPDF): appears in the Start menu, opens a PDF, and has the simplified settings.
  Its postinstall ran (`postInstallOk`).
- **Documents and Spreadsheets** (LibreOffice): opens with the simplified menus and toolbar, and
  Bigger/Smaller work. Its postinstall ran (`postInstallOk`).
- **Photo Album** 0.7.34-pa.1: installed in `C:\Program Files\Photo Album`, in the all-users Start menu,
  and starts for a normal user. The log says `publisher-unsigned` for it: expected, it is our own unsigned build.
- **On the ARM64 PC:** in Task Manager, Details tab, add the Architecture column: each running app
  should say ARM64, not x64. The exception is Photo Album: this test has no ARM64 build of it, so it
  runs under emulation on ARM64 (it says x64), and `appupdater status` or the log shows `arch-fallback`
  for it. That is expected.

Anything ending in `-failed` or `retry-later` is worth a screenshot and a copy of the log.

## 5. Where the log is

`C:\ProgramData\AppUpdater\appupdater.log` (open it from the elevated prompt, for example
`notepad C:\ProgramData\AppUpdater\appupdater.log`). Installer logs for the office suite are in
`C:\ProgramData\AppUpdater\logs\`.

## 6. How to undo

1. Remove the apps: Settings, Apps, Installed apps: uninstall SumatraPDF, LibreOffice and Photo Album.
2. Delete the tasks (elevated prompt):

       schtasks /Delete /TN AppUpdater /F
       schtasks /Delete /TN AppUpdater-Logon /F

3. Delete the updater's folders and its hidden Uninstall entry:

       rmdir /S /Q "C:\ProgramData\AppUpdater"
       rmdir /S /Q "C:\Program Files\AppUpdater"
       reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\AppUpdater" /f

## 7. Install or remove through the RMM (no clicks)

`install-officeapps.ps1` does steps 1 to 3 in one go: it picks the x64 or ARM64 updater, downloads it,
checks its SHA-256, runs `setup`, the first `check` and `status`. The updater's download addresses and
SHA-256 values are pinned inside the script, so the script is the one thing to trust: compare it with
`SHA256SUMS` if you copy it around by hand. Run it **as SYSTEM** (the RMM's normal way to run a script)
or from an elevated PowerShell, in Windows PowerShell 5.1 or later. It never asks anything.

To have the RMM fetch and run it (one line, for an RMM "run command" step running as SYSTEM):

    powershell -NoProfile -ExecutionPolicy Bypass -Command "[Net.ServicePointManager]::SecurityProtocol='Tls12'; $p=Join-Path $env:ProgramData ('install-officeapps-'+[guid]::NewGuid()+'.ps1'); Invoke-WebRequest -UseBasicParsing -Uri 'https://raw.githubusercontent.com/lifeguardcomputers/installers/office-apps-test/LG-OfficeApps/install-officeapps.ps1' -OutFile $p; & $p; $c=$LASTEXITCODE; Remove-Item -LiteralPath $p; exit $c"

Options: `-Enroll pdf-reader,photo-album` installs only those apps (ids as listed at the top); `-SkipFirstCheck`
sets up the updater only and leaves the installs to the scheduled task.

Exit codes: 0 every app installed or up to date; 1 an app had a problem (see the log); 2 setup failed or
not elevated; 3 download, SHA-256 or processor problem. Its own log is
`C:\ProgramData\AppUpdater\install.log` (before setup succeeds, in a `C:\ProgramData\AppUpdater-install-...` folder,
which it names).

To remove everything (tasks, the three apps, the updater's folders and its Uninstall entry), run
`uninstall-officeapps.ps1` (https://raw.githubusercontent.com/lifeguardcomputers/installers/office-apps-test/LG-OfficeApps/uninstall-officeapps.ps1) the same way. Add `-KeepApps` to
remove only the updater. Exit code 0 means nothing is left; 1 lists what is.

## 8. Deeper tests

Each project's README has a full Windows checklist:

- App Updater: `projects/app-updater/README.md`
- PDF Reader: `projects/pdf-viewer/README.md`
- Documents and Spreadsheets: `projects/senior-office-suite/README.md`
- Photo Album: `projects/photo-organizer/README.md`

Every file here is listed with its SHA-256 in `SHA256SUMS`.
