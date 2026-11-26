# Adding a network printer to a remote user's session, when every obvious method fails

A service desk problem I spent a while solving properly instead of walking to the desk one more time.

## The problem

A user calls. They need a network printer added. You are already connected to their machine over PowerShell remoting, you have admin rights, and you have the print server name and the queue name.

You cannot do it.

`Get-Printer` on their machine shows you printers that the user has never heard of, and none of the ones they do have. `Add-Printer` appears to succeed and the user still sees nothing. So you give up, walk over, sit at their desk, and add it in two clicks.

Multiply that by every printer request in a week.

## Why the obvious method fails

Per-user printer connections are not machine state. They live in the user's own registry hive, in three places:

```
HKCU\Printers\Connections
HKCU\Software\Microsoft\Windows NT\CurrentVersion\Devices
HKCU\Software\Microsoft\Windows NT\CurrentVersion\PrinterPorts
```

That is the whole problem in one line. `Invoke-Command` runs as the account you authenticated with, which is your admin account, so `HKCU` resolves to *your* hive on their machine, not theirs. `Get-Printer` is reading a different user's printers. `Add-Printer` is adding a printer for an admin account that will never log in interactively.

Both commands are working correctly. They are just working on the wrong person.

## What I tried that did not work

**Running the command as the user with `-Credential`.** You do not have their password, and you should not want it.

**A scheduled task registered to run in the user's context.** This looks like the clean answer and it is the one most forum posts suggest. It fails on credential delegation: the task runs as the user but with no network credentials, so the moment it reaches out to `\\PRINTSERVER` to pull the driver, it gets denied. You have moved the failure, not fixed it.

**Logon scripts and Group Policy preferences.** These work, and they are the right answer for printers that a whole department needs. They are useless for "this one person needs this one printer in the next five minutes," which is what the ticket actually says.

**`Add-Printer -ConnectionName` as admin, then hoping.** Adds nothing for the user. It does do one genuinely useful thing, which turned out to be part of the answer.

## What works

Four layers, none of which is clever on its own.

**1. Make the connection at machine level, in admin context.**

`printui.dll` has a per-machine printer connection mode that works fine as admin, because it is not writing to anybody's `HKCU`:

```
rundll32 printui.dll,PrintUIEntry /ga /c\\TARGETPC /n\\PRINTSERVER\QUEUENAME
```

`/ga` is the per-machine add. This is the piece that survives the admin context problem.

**2. Get the driver on the machine.**

This is where `Add-Printer` as admin earns its place. The connection is useless if the driver is not present, and adding the connection as admin pulls the driver down from the print server and installs it machine-wide:

```powershell
Add-Printer -ConnectionName "\\PRINTSERVER\QUEUENAME"
```

The printer it creates for the admin account is throwaway. The driver it installs is the point.

**3. Write the connection into the target user's hive.**

Now go and do deliberately what `HKCU` would not do for you. If the user is logged on, their hive is already mounted under `HKU\<their SID>`. If they are not, load it:

```powershell
reg load "HKU\TempHive" "C:\Users\<username>\NTUSER.DAT"
```

Resolve the SID properly rather than guessing from the folder name:

```powershell
$sid = (New-Object System.Security.Principal.NTAccount("DOMAIN","username")).Translate(
         [System.Security.Principal.SecurityIdentifier]).Value
```

Then write the three sets of values that a real printer connection consists of, under `Printers\Connections`, `Devices` and `PrinterPorts`. Copying them from a machine where the printer already works is far more reliable than constructing them from documentation.

If you loaded the hive, unload it when you are done:

```powershell
reg unload "HKU\TempHive"
```

Do not skip that. A hive left loaded locks the profile and the user's next logon is your next ticket.

**4. Make it appear without a reboot.**

`/ga` connections normally materialise at next boot, which is not an answer the user wants to hear. Restarting the spooler picks it up immediately:

```powershell
Restart-Service -Name Spooler -Force
```

The shell sometimes still shows a stale printer list. Restarting Explorer fixes the display, at the cost of closing their File Explorer windows, so it is worth asking first rather than doing it to them.

## Removal

Same steps, reversed, and in this order:

1. Delete the values you added from the user's hive, all three locations.
2. Remove the machine-level connection: `rundll32 printui.dll,PrintUIEntry /gd /c\\TARGETPC /n\\PRINTSERVER\QUEUENAME`
3. Restart the spooler.

Skipping step 1 and only doing step 2 leaves orphaned entries that produce a printer the user can see and cannot print to, which is a worse ticket than the one you started with.

## Things that will bite you

- **Point and Print restrictions.** If policy blocks non-admin driver installation you are fine here, because the install in layer 2 happens as admin. If policy blocks it outright, nothing in this approach helps.
- **Type 3 versus Type 4 drivers.** Package-aware drivers behave differently on install. Test with the drivers your print server actually serves, not with one you picked.
- **`/ga` is per machine.** On a shared or multi-user machine you have just added the printer for everyone who logs on there. If that is not acceptable, drop layer 1, pre-install the driver and rely on the hive write alone.
- **The target machine needs line of sight to the print server.** If it does not, none of this applies and you have a different problem.
- **Writing to another user's hive is a sharp tool.** Validate the SID, change only the values you mean to change, and unload cleanly.

## What it changed

Printer requests stopped being a deskside visit. I never tracked the count, so I am not going to invent one, but it was a recurring request type on a service desk covering a head office plus plants and remote sites, and each one previously meant leaving the queue to walk to someone's desk. Afterwards it was handled from the same remote session as everything else in the ticket.

The thing I took away from it is smaller than the technique. The reason this problem survives in so many service desks is not that it is hard, it is that the first three sensible approaches all fail in ways that look like the approach was wrong rather than the context. Once you know the connection lives in the user's hive and your session is not that user, the fix stops being clever and becomes obvious.

---

*Written up from a tool I built for internal use. The approach is general, the code here is generic, and nothing in it is specific to any employer's environment.*
