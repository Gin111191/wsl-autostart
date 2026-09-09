# wsl-autostart

Make WSL2 start automatically when you log into Windows — **and actually stay
running**, so you can SSH straight into it without touching the Windows desktop.

One file. One command. No admin rights.

---

## The problem

You want `ssh my-wsl-box` to work right after the PC boots. So you add
`wsl.exe -d Ubuntu -u root /bin/true` to Task Scheduler at logon.

It appears to work. `wsl -l -v` says `Running`. Twenty seconds later the distro
is dead.

## Why

**WSL shuts a distro down a few seconds after its last session closes.**

`/bin/true` exits immediately, so WSL counts zero open sessions and powers the
distro off. Enabling `systemd` does *not* prevent this — the shutdown is issued
by the Windows side of WSL, not from inside Linux. `journalctl` shows a clean
`systemd-poweroff.service` at the end of the boot.

Here is `journalctl --list-boots` from a machine with the broken setup:

```
-2  16:23:19 → 16:23:38   ← the scheduled task fired here. Alive 19 SECONDS.
-1  16:24:46 → 16:28:46   ← a manual terminal, died when the window closed
 0  16:30:45 → running    ← only alive because a session was attached
```

So the autostart command must be one that **never exits**:

```
WRONG   wsl.exe -d Ubuntu -u root /bin/true          → boots, dies ~19s later
RIGHT   wsl.exe -d Ubuntu -u root -e sleep infinity  → session stays, distro lives
```

---

## Install

Open a **normal** PowerShell — Administrator is *not* required — and run:

```powershell
'CreateObject("WScript.Shell").Run "C:\Windows\System32\wsl.exe -d Ubuntu -u root -e sleep infinity", 0, False' |
  Set-Content "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\start-wsl.vbs"
```

That's the entire install. Windows runs everything in the Startup folder at login.

### What each part does

| Part | Why |
|---|---|
| `.vbs` + `0` | Hidden window — no black console flashing at login. A `.bat` file *will* flash. |
| `False` | Don't wait for the command; login isn't delayed. |
| `-u root` | Runs as root, independent of your user's shell config. |
| `-e sleep infinity` | The process that never exits. **This is the whole fix.** |
| `-d Ubuntu` | Your distro name — run `wsl -l -v` and change it if yours differs. |

To open that folder by hand: <kbd>Win</kbd>+<kbd>R</kbd> → `shell:startup`

---

## Verify

### Without rebooting

```powershell
wsl --shutdown
wscript.exe "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\start-wsl.vbs"
wsl -l -v          # → Running
```

Now **wait two minutes** and check again:

```powershell
wsl -l -v          # must STILL be Running
```

> The two-minute wait *is* the test. The broken version also reports `Running`
> immediately — the two are indistinguishable until the timeout elapses.

### From inside WSL

```bash
pgrep -u root -x sleep -a       # → <pid> sleep infinity
```

That process is what holds the distro open. If it's missing, the script never ran.

### The real acceptance test

Reboot Windows → log in → wait a minute → from your other machine:

```
ssh your-wsl-host
```

...without typing `wsl.exe` on Windows even once.

---

## Requirement: you must log into Windows

The Startup folder only fires on an **interactive login**. Boot the machine and
leave it at the lock screen and nothing starts.

If you need WSL up on a headless box that nobody logs into, enable Windows
auto-login. Do **not** reach for a `-AtStartup` scheduled task running as
`SYSTEM`: WSL creates a separate VM per Windows user, so you end up with two
distro instances in parallel, each entitled to whatever `.wslconfig` allocates.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Distro dies ~20s after starting | The command still exits. Check the `.vbs` really says `-e sleep infinity`. |
| Nothing starts at all | Machine booted but nobody logged in. Startup only fires on login. |
| Black window flashes at login | You used a `.bat`. Switch to `.vbs` with the `0` argument. |
| Wrong distro starts | `wsl -l -v`, then fix `-d <name>` in the `.vbs`. |
| Distro dies after ~3 days | You used the scheduled-task variant without `-ExecutionTimeLimit 0` (see below). |
| `wsl -l -v` says Running but SSH is refused | Not this project — check `sshd` and your network/VPN layer inside WSL. |

## Uninstall

Delete `start-wsl.vbs` from the Startup folder. That's all — nothing else was touched.

---

## Alternative: Task Scheduler

The Startup folder is simpler and needs no elevation, so prefer it. If you want a
scheduled task anyway, use **Administrator** PowerShell:

```powershell
Register-ScheduledTask -TaskName "Start WSL" -Force `
  -Action   (New-ScheduledTaskAction -Execute "C:\Windows\System32\wsl.exe" -Argument "-d Ubuntu -u root -e sleep infinity") `
  -Trigger  (New-ScheduledTaskTrigger -AtLogOn) `
  -Settings (New-ScheduledTaskSettingsSet -Hidden -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit (New-TimeSpan))
```

`-ExecutionTimeLimit (New-TimeSpan)` means zero, i.e. no limit. **Don't omit it.**
Scheduled tasks default to a 72-hour cap, which would kill `sleep infinity` and
take WSL down with it every three days.

To remove it later:

```powershell
Unregister-ScheduledTask -TaskName "Start WSL" -Confirm:$false
```

---

## Tested on

WSL 2.7.12, Windows 11 build 26200, Ubuntu with `systemd=true` in `/etc/wsl.conf`.
The behaviour it works around has been in WSL2 for years and is not version-specific.
