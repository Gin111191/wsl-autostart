# wsl-autostart

Make WSL2 start automatically on Windows — **and actually stay running** — so you can
SSH straight into it without touching the Windows desktop.

One file, one command, no admin rights if you log in.
[One more step](#if-nobody-logs-into-windows) if you want it up before anyone does.

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

## If nobody logs into Windows

The Startup folder only fires on an **interactive login**. Boot the machine, leave it
at the lock screen, and nothing above has run. `schtasks /query` says as much about the
task variant too — `Logon Mode: Interactive only`.

So for a box you want to reach by SSH right after power-on, without walking over to it,
the autostart has to be triggered by the boot rather than by the login.

### Do not use SYSTEM

The obvious move is a `-AtStartup` task running as `SYSTEM`. Don't.

WSL keys a distro instance to the **Windows user SID**, so a `SYSTEM` instance and your
own instance are two separate utility VMs against the same `ext4.vhdx`, each entitled to
whatever `.wslconfig` hands out. You get the memory twice and the disk contended.

### Run it at startup as yourself instead

Same trigger, same SID as your interactive session, so it is the *one* instance — the one
you also attach to when you do eventually log in. Windows stores the credential the way it
stores any saved task credential; nothing lands in the registry in clear text, which is the
part auto-login gets wrong.

**Administrator** PowerShell, once. Paste these **one line at a time** — a legacy conhost
window drops characters out of multi-line pastes, and backtick continuations are the first
thing it mangles, so there are none here:

```powershell
$c = Get-Credential -UserName (whoami) -Message "wsl-boot"
```
```powershell
$t = New-ScheduledTaskTrigger -AtStartup
```
```powershell
$t.Delay = 'PT30S'
```
```powershell
$a = New-ScheduledTaskAction -Execute "C:\Windows\System32\wsl.exe" -Argument "-d Ubuntu -u root -e sleep infinity"
```
```powershell
$s = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit (New-TimeSpan)
```

Check all four before registering — this catches both failures below at once:

```powershell
$c.UserName; $t.CimClass.CimClassName; $a.Execute; $s.ExecutionTimeLimit
```

Four lines back, or something is null:

```
gin-pc\admin
MSFT_TaskBootTrigger
C:\Windows\System32\wsl.exe
PT0S
```

`MSFT_TaskTimeTrigger` instead of `MSFT_TaskBootTrigger` means `-AtStartup` was lost and
the task would fire once at a clock time. Then:

```powershell
Register-ScheduledTask -TaskName "wsl-boot" -Force -User $c.UserName -Password $c.GetNetworkCredential().Password -Action $a -Trigger $t -Settings $s
```

And confirm what was actually saved, which is not always what you asked for:

```powershell
$x = Get-ScheduledTask -TaskName "wsl-boot"; $x.Triggers.CimClass.CimClassName; $x.Settings.ExecutionTimeLimit; $x.Principal.LogonType
```

```
MSFT_TaskBootTrigger
PT0S
Password
```

`LogonType: Password` is the one that matters — it means the task runs with nobody logged
in. `Interactive` means the credential did not stick and you have rebuilt the thing you
were trying to replace.

### The two ways this fails

**`You cannot call a method on a null-valued expression`** on the register line: `$c` is
null because `Get-Credential` was cancelled. `Ctrl+C` or an empty Enter at its prompt both
leave it null and it says nothing about it. Re-run that line and check `$c.UserName` before
moving on.

**`The user name or password is incorrect`**: it wants the account password — what you
would type at the lock screen — not your PIN. A PIN is device-local and a task cannot store
it. If you sign in with a Microsoft account, it is that account's password.

```powershell
Get-LocalUser | Select-Object Name, Enabled, PasswordRequired, PrincipalSource
```

`PrincipalSource: MicrosoftAccount` may need `-User "MicrosoftAccount\you@example.com"`
instead of the `whoami` form. `PasswordRequired: False` with PIN-only sign-in means there
is no password to store at all — see below.

**Accept it before you trust it.** Reboot, do **not** log in, wait a minute, and SSH in
from another machine. Only once that works, remove the login-triggered copies:

```powershell
Unregister-ScheduledTask -TaskName "Start WSL" -Confirm:$false
Remove-Item "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\start-wsl.vbs"
```

Keep them until then — they are what works today.

### If your account has no password

An account that only ever unlocks with a PIN or Windows Hello has no password for the task
to store, and registration fails. That is the case where auto-login is the remaining option:
enable it, then put `rundll32 user32.dll,LockWorkStation` in the Startup folder next to
`start-wsl.vbs` so the desktop does not sit unlocked. The password lives in the registry
either way, so treat it as the weaker choice, not the default.

### The cost: it takes WSLg down with it

A boot-triggered instance is created before anyone has a desktop, and WSL binds a distro
instance to the session that created it. WSLg's Windows half, `msrdc.exe`, therefore
lands in **session 0**, where there is no desktop to draw on. It connects to the weston
RDP server inside the WSLg system distro, then drops the peer ~100 s later; weston's
`rdp-backend.so` NULL-derefs on the disconnect path and dies with SIGSEGV, WSLGd
restarts it, and the loop runs all day — about 850 crashes per 24 h on the box this was
written on:

```
$ grep -c "terminated with signal 11" /mnt/wslg/stderr.log
$ dmesg | grep weston
weston[3773]: segfault at 218 ... in rdp-backend.so[1d17b,...]
$ tasklist.exe | findstr msrdc      # Session 0, while your desktop is session 1
$ query.exe session                 # console  ADMIN  1  Active
```

Logging in later does not rescue it: the instance already exists, and every terminal you
open afterwards attaches to that same one. No Linux GUI app works for as long as the
machine is up.

So pick one, they do not combine:

| You want | Use |
|---|---|
| SSH before anyone logs in | the boot task above — and no GUI apps, ever |
| Linux GUI apps (WSLg) | the Startup-folder `.vbs` only, so the first instance is yours |

If you take the boot task and never wanted GUI apps anyway, turn WSLg off outright and
save the crash loop and its log churn — `%USERPROFILE%\.wslconfig`:

```ini
[wsl2]
guiApplications=false
```

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Distro dies ~20s after starting | The command still exits. Check the `.vbs` really says `-e sleep infinity`. |
| Nothing starts at all | Machine booted but nobody logged in. Startup only fires on login — see [If nobody logs into Windows](#if-nobody-logs-into-windows). |
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

Measured: the 19-second death, the `sleep infinity` fix, `Logon Mode: Interactive only`
on the logon-task variant, and the `wsl-boot` task registering with
`MSFT_TaskBootTrigger` / `PT0S` / `LogonType: Password` on a real machine. Both failure
modes in *The two ways this fails* are transcripts, not guesses.

Still unmeasured: whether a distro started from **session 0** stays up and answers SSH
with nobody logged in. That is the one claim here reasoned from how WSL keys instances to
the user SID rather than observed. Run the acceptance test above before deleting the
`.vbs`, and if session 0 turns out not to hold a distro open, open an issue — auto-login
is then the only answer and this section is wrong.
