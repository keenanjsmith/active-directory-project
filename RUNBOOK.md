# AD Lab Runbook

Build a working Active Directory domain in VirtualBox: one Windows Server 2022 domain controller
running AD DS, DNS, DHCP, and NAT routing, plus a Windows 11 Enterprise client that joins the
domain and authenticates against it.

**Reference video:** Josh Madakor, https://youtu.be/MHsI8hJmggI
Timestamps throughout are real, taken from the transcript. Scrub directly to them.

**This document is not a transcript of that video.** It builds the same lab on newer software
(Server 2022 and Windows 11 instead of Server 2019 and Windows 10, VirtualBox 7 instead of an
older release) and every place the newer software behaves differently is called out inline. Those
callouts are marked **PITFALL** and come from an actual build, not from anticipating problems.

---

## What differs from the video

|  | Video | This build |
|---|---|---|
| Server OS | Server 2019 | Server 2022 |
| Client OS | Windows 10 | Windows 11 Enterprise |
| Hypervisor | Older VirtualBox | VirtualBox 7.x |
| Domain | `mydomain.com` | `smithlab.local` |
| DC name | `DC` | `DC-1` |
| Client name | `client1` | `CLIENT-1` |
| Admin account | `a-jmadakor` | `keenan_admin` |
| Password | `Password1`, on camera | `<LabPassword>` |

Internal subnet is `172.16.0.0/24` with the DC static at `172.16.0.1`, same as the video.

> **Never use a real password anywhere in this build.** It appears in plaintext in several
> configuration steps and in screenshots. The video uses `Password1` on camera and explicitly says
> not to do that in real life.

**Time:** roughly four to six hours across two sittings if this is your first time. The two
Windows installs are most of it and are mostly waiting.

---

## Before you start: keep a build log

Open a second window with a scratch file and record anything that surprises you: the exact error
text, what you tried, what actually fixed it. Two lines is enough. This costs almost nothing
during the build and is the difference between documentation that shows a happy path and
documentation someone can actually use.

Everything marked **PITFALL** below came out of that file.

---

# Part 0. Host preparation

Do all of this before creating a single VM.

## 0.1 Move the VirtualBox machine folder off C:

Open VirtualBox, then **File > Preferences > General > Default Machine Folder**.

**Type the path directly into the field**, for example `D:\VirtualBox VMs`. Click OK. Reopen
Preferences to confirm it saved.

> **PITFALL: the default location will fill your system drive.**
> The default is `C:\Users\<user>\VirtualBox VMs`. This lab creates two 60 GB dynamic disks. On a
> system drive with only a few gigabytes free, the install fills the drive partway through and
> fails.
>
> **Do not use the folder browser to pick the new location.** On Windows it opens inside
> OneDrive by default. Never put VMs in a synced folder. OneDrive will attempt to sync 60 GB disk
> files, cause sync conflicts while the VM is running, and can corrupt them. Type the path in.

## 0.2 Clear out dead VM entries

If VirtualBox lists any VM as **Inaccessible**, remove it now so it does not confuse you later.

Right-click the entry > **Remove** > **"Remove only"**.

> **PITFALL: "Remove only" versus "Delete all files".**
> Symptom:
> ```
> Runtime error opening '...\.vbox' for reading
> VERR_PATH_NOT_FOUND (Path not found)
> Result Code: E_FAIL (0x80004005)
> ```
> This means VirtualBox has a registration entry pointing at files that no longer exist. Choose
> **Remove only**, which clears the registration. **Delete all files** attempts to delete files
> that are already gone and is the wrong choice here.

## 0.3 Download and organize the ISOs

Both are free Microsoft evaluation editions.

- Windows Server 2022 Evaluation, from the Microsoft Evaluation Center
- Windows 11 Enterprise Evaluation, from the Microsoft Evaluation Center

Create `D:\ISOs\` and move both there with clear names:

```
D:\ISOs\WindowsServer2022_Eval.iso
D:\ISOs\Windows11_Enterprise_Eval.iso
```

ISOs are read-only install media and can live anywhere. Keeping them out of the VirtualBox machine
folder avoids mixing them with VM disk files, and keeping them out of Downloads protects them from
cleanup tools.

> **Note:** if WinRAR is installed, Windows Explorer may list `.iso` files with Type "WinRAR".
> Cosmetic only. VirtualBox reads them fine.

> **On evaluation expiry:** Server 2022 Standard Eval runs 180 days, Windows 11 Enterprise Eval
> runs 90. Neither is a practical constraint for this lab. See Appendix B if you need longer.

---

# Part 1. Build DC-1

## 1. Create the DC-1 VM (video 4:35)

VirtualBox > **New**.

- **Name:** `DC-1`
- **ISO Image:** the Server 2022 ISO
- **Tick "Skip Unattended Installation"** (see pitfall below)
- **Memory:** 4096 MB, **CPUs:** 2, **Disk:** 60 GB

> **PITFALL: the ISO dropdown looks empty and it is supposed to.**
> It is not a list of ISOs VirtualBox has detected. It is a file picker that starts blank. Click
> **Other...** and browse to the `.iso` file.
>
> Nothing needs converting, extracting, or importing first. An ISO is already a single
> ready-to-use file. Once attached, the Version field should auto-correct from "Windows 10
> (64-bit)" to "Windows 2022 (64-bit)" and Edition should populate. That auto-detection is your
> confirmation the ISO is readable.

> **PITFALL: VirtualBox 7 hijacks VM creation with an unattended installer.**
> If "Skip Unattended Installation" is left unchecked, clicking Next lands you on an "Unattended
> Guest OS Install Setup" page pre-filled with username `vboxuser`, password `changeme`, and
> domain name `myguest.virtualbox.org`. The prior screen says "This OS type can be installed
> unattendedly. The install will start after this wizard is closed."
>
> Go **Back** and tick the checkbox. Next then proceeds to Hardware as expected.
>
> **Why not just use it:** it creates an account you do not want, applies a DNS suffix you do not
> want, and runs the entire Windows setup without you. You lose the OOBE screens, which on the
> client is where a step you need to document lives.
>
> With Skip ticked, the Edition dropdown greys out. Expected. You pick the edition inside the
> Windows installer instead.
>
> The video predates this wizard, so it never appears on camera. Anyone on VirtualBox 7.x will
> hit it on **both** VMs.

## 2. Configure DC-1's two network adapters (video 5:30)

**Settings > Network.**

- **Adapter 1:** Enable, attached to **NAT**
- **Adapter 2:** Enable, attached to **Internal Network**, name `intnet`

Adapter 1 is the path to the outside world. Adapter 2 is the isolated lab network. This split is
the whole point of the design: the client will have no route to the internet except through the
DC, so when the client reaches the internet you have proven the DC is routing.

📸 `01-dc1-vm-settings.png`
📸 `02-dc1-network-adapters-adapter-1.png`
📸 `02-dc1-network-adapters-adapter-2.png`

> **Two screenshots, not one.** The adapters live on separate tabs and cannot be captured
> together.

## 3. Install Server 2022 (video 7:40)

Boot the VM and run through Windows Setup.

- Edition: **Windows Server 2022 Standard Evaluation (Desktop Experience)**. Do not pick the
  non-Desktop-Experience option unless you want to do the entire lab in a command prompt.
- Custom install, use the whole disk.
- Set the Administrator password to your `<LabPassword>`.

To send Ctrl+Alt+Del into the VM: **Input > Keyboard > Insert Ctrl-Alt-Del**. Pressing the keys
directly goes to your host.

📸 `04-dc1-server-installed.png`

## 4. Install Guest Additions on DC-1 (video 10:00)

Full click path:

1. VM menu bar: **Devices > Insert Guest Additions CD image**
2. Inside Windows: **File Explorer > This PC > CD Drive (D:) VirtualBox Guest Additions**
3. Run **VBoxWindowsAdditions-amd64.exe**
4. Next through the prompts, then **Install**
5. If Windows Security prompts about an Oracle device driver, choose **Install**
6. Reboot when offered

**What it does:** fixes mouse capture so you no longer need Right Ctrl to release the cursor,
enables dynamic screen resolution when you resize the VM window, and enables shared clipboard.
Quality of life, not functionality. The lab works without it but taking screenshots is much worse.

> **PITFALL: the installer's C:\ is not your C:\.**
> The Destination Folder defaults to `C:\Program Files\Oracle\VirtualBox Guest Additions`. If your
> host system drive is nearly full, this looks like it is about to undo all the storage planning
> from Part 0.
>
> It is not. That `C:\` is the **guest's** virtual C: drive, which lives inside the VM's disk file
> on D:. Every Windows guest labels its own system drive C: regardless of what the host uses.
>
> Confirmation: the installer reports available space matching the VM's 60 GB disk, not the host's
> free space. If those numbers differ, you are looking at the guest.
>
> **Rule of thumb: once you are clicking inside the VM window, every drive letter is the guest's.**
> Host paths only matter in the VirtualBox Manager and settings dialogs.

## 5. Identify and rename the NICs (video 12:25)

Click the network icon in the system tray > **Network** > **Change adapter options**.

Two adapters, both with meaningless names. Identify them by address, not by number:

- Right-click the first > **Status > Details**. If it shows a normal home-network address
  (`10.x.x.x` or `192.168.x.x`), that is the NAT adapter. Rename it **internet**.
- The other will show **169.254.x.x**. That is APIPA, meaning it looked for a DHCP server and
  found none. Correct, since nothing exists on the internal network yet. Rename it **internal**.

Do this before anything else touches the adapters. You select them **by name** during RRAS setup,
and assigning a static IP to the wrong one is a hard error to spot afterward.

📸 `05-dc1-nics-renamed.png`

## 6. Static IP on the internal NIC (video 15:35)

Right-click **internal** > Properties > **Internet Protocol Version 4** > Properties > *Use the
following IP address*:

- IP address: **172.16.0.1**
- Subnet mask: **255.255.255.0**
- Default gateway: **leave blank.** The DC is the gateway for this network, so it has nothing to
  point at.
- Preferred DNS: **127.0.0.1.** Loopback. The DC becomes its own DNS server once AD is installed.

Leave the **internet** adapter on DHCP. Do not touch it.

📸 `06-dc1-static-ip.png`

## 7. Rename the computer (video 14:33)

Right-click Start > **System** > **Rename this PC** > `DC-1` > Restart.

Do this before promoting to a domain controller. Renaming a DC afterward is possible but
unnecessarily painful.

📸 `07-dc1-renamed.png`

## 8. Install AD DS and promote to a domain controller (video 17:45)

**Server Manager > Add roles and features** > Next > Next > select your server > Next > check
**Active Directory Domain Services** > Add Features > Next > Next > Next > **Install**.

Be careful not to grab one of the other Active Directory roles in the list. You want Domain
Services.

📸 `08-adds-role-installed.png`

Then click the **yellow flag** in the top bar > **Promote this server to a domain controller**:

- **Add a new forest**
- Root domain name: `smithlab.local`
- Next, set a DSRM password (use your `<LabPassword>`, you will never need it in this lab), Next,
  Next, Next, Next, **Install**

The server restarts itself. Log back in as **`smithlab\Administrator`**.

DNS is installed automatically during promotion. You do not add it as a separate role. This is why
step 6 pointed the DC at `127.0.0.1`.

📸 `09-domain-promoted.png`

## 9. Create a dedicated domain admin account (video 20:00)

**Start > Windows Administrative Tools > Active Directory Users and Computers.**

Right-click `smithlab.local` > New > **Organizational Unit** > name it `ADMINS`. Uncheck
*Protect container from accidental deletion*, which only makes cleanup annoying in a lab.

Inside `ADMINS`: right-click > New > **User**

- Name: your name
- User logon name: `keenan_admin`
- Password: `<LabPassword>`
- Uncheck *User must change password at next logon*
- Check *Password never expires*

Then right-click the new user > **Properties > Member Of > Add** > type `Domain Admins` > Check
Names > OK > Apply.

**Sign out and log back in as `smithlab\keenan_admin`.** Use this account for everything from here
forward.

📸 `10-admins-ou-and-user.png`
📸 `11-domain-admins-membership.png`
📸 `11b-logged-in-as-domain-admin.png`

> **Why a separate admin account.** The built-in Administrator is the first account an attacker
> tries, and its actions are hard to attribute in logs. A named administrative account gives you
> an audit trail: the log shows who did the thing, not just that Administrator did it. This is
> standard practice and worth a sentence in your README.
>
> The supplementary screenshot is worth taking. On a domain controller, ordinary domain users are
> denied local logon by default. Reaching a desktop as this account is independent evidence that
> it carries elevated rights, separate from what the group membership dialog claims.

## 10. Install RRAS and configure NAT (video 23:20)

**Add roles and features** > Next > Next > server > Next > check **Remote Access** > Next > Next >
Next > under role services check **Routing** (LPR and Web Server get pulled in automatically) >
Next > **Install**.

Then **Tools > Routing and Remote Access** > right-click your server > **Configure and Enable
Routing and Remote Access** > Next > select **Network address translation (NAT)** > Next > select
the **internet** interface > Next > **Finish**.

Selecting the right interface here is the payoff for renaming them in step 5.

A green up-arrow on the server icon means it worked.

> **PITFALL: the interface list can come up empty.** The video hits this at 24:55. Cancel out of
> the wizard, reopen Routing and Remote Access, and run it again. It usually populates on the
> second attempt. Occasionally it needs a reboot.

📸 `12-rras-nat-configured.png`

## 11. Install DHCP and create the scope (video 26:00)

**Add roles and features** > through to role selection > check **DHCP Server** > Add Features >
Next > Next > **Install**.

**Tools > DHCP** > expand your server > right-click **IPv4** > **New Scope**:

- Name: `172.16.0.100-200`
- Start IP: **172.16.0.100**
- End IP: **172.16.0.200**
- Length: **24**, Subnet mask: **255.255.255.0**
- Exclusions: none
- Lease duration: 8 days (the default is fine for a lab)
- **Yes, I want to configure these options now**
- **Router (default gateway): 172.16.0.1**, the DC's internal NIC
- DNS: should already be populated with the DC. Leave it.
- WINS: skip
- **Yes, I want to activate this scope now**

Then right-click the **DHCP server node > Authorize**, then right-click > **Refresh**. The IPv4
node should turn green.

> **PITFALL: "The address range you have specified is too large for a single scope."**
> The wizard then offers to create a superscope containing three scopes of 256 addresses each.
>
> **Answer No.** The real cause is almost always a digit that landed in the wrong octet of the End
> IP. The IP entry boxes auto-advance between octets as you type, which makes this very easy to do
> without noticing. If the third octet differs between start and end, you have crossed into
> another network and the address count multiplies by 256 per step, which is how a 101-address
> range becomes roughly 768.
>
> Accepting the superscope would silently build the wrong topology instead of surfacing the typo.
> A superscope serves multiple subnets from one DHCP server. This lab has one subnet.
>
> **Fix:** go Back and retype all four fields from scratch rather than hunting for the stray
> digit. Faster.
>
> Also note that **Length and Subnet mask are linked fields.** Editing one rewrites the other. Set
> Length to 24 and confirm the mask reads `255.255.255.0`.

## 12. Verify the DHCP options actually saved

This is the single highest-value verification in the build. If the Router option did not save,
the client comes up with no default gateway and nothing after this point works. The video hits
exactly this failure at 50:00 and loses several minutes to it.

Expand the scope, then click **Scope Options** (it sits below Reservations). You should see:

```
003 Router            172.16.0.1
006 DNS Servers       172.16.0.1
015 DNS Domain Name   smithlab.local
```

> **PITFALL: look in Scope Options, not Server Options.**
> **Server Options will be empty and that is correct.** The New Scope Wizard writes to Scope
> Options and never touches Server Options. Clicking Server Options shows only descriptive help
> text in the right pane, which looks exactly like the router option is missing entirely.
>
> The difference: Server Options are server-wide defaults inherited by all scopes. Scope Options
> apply to one scope and override the server default. A single-subnet lab only needs it in one
> place, and the wizard chooses Scope Options.
>
> **If Scope Options is also empty,** the router page in the wizard genuinely did not save.
> Fix: right-click **Scope Options > Configure Options >** check **003 Router >** enter
> `172.16.0.1` **> Add > OK**.

Also confirm here:

- The **scope icon is green** with an up arrow, not red. Red means created but not activated.
- The **server node icon is green**. Red means not authorized. Right-click the server >
  **Authorize** > Refresh.

📸 `13-dhcp-scope-active.png`

## 13. Disable IE Enhanced Security Configuration (video 31:20)

**Server Manager > Local Server > IE Enhanced Security Configuration > Off** for both
Administrators and Users.

Without this, every page load on the server throws security prompts and you cannot download the
script in the next step.

**Lab only.** This removes a hardening control on the most privileged machine in the environment.
Never do it in production. It belongs in your README as an honestly declared shortcut.

## 14. Bulk-create users with PowerShell (video 32:15)

Script: https://github.com/joshmadakor1/AD_PS

Download the repo zip on DC-1 and extract it to the Desktop.

**Edit `names.txt` first.** Add your own name at the top so an account gets created for you. You
will use it in the final verification step.

Open **Windows PowerShell ISE as Administrator**: Start > Windows PowerShell > right-click
PowerShell ISE > More > Run as administrator.

In the console pane:

```powershell
Set-ExecutionPolicy Unrestricted
```

Answer **Yes to All**. This is a security control you are deliberately disabling for the lab.
Production uses `RemoteSigned` at minimum, with signed scripts. Declare this in your README.

**You must `cd` into the script's folder** or it cannot find `names.txt`:

```powershell
cd C:\Users\keenan_admin\Desktop\AD_PS-master
```

Run `ls` to confirm `names.txt` is present. Then open the `.ps1` in ISE and click **Run**.

> **Three things to expect:**
>
> 1. The script hardcodes the OU name `_USERS` in two places, once creating the OU and once as the
>    `-Path` for each new user. If you want a different name you must change **both** or it fails.
>    Simplest is to leave it.
> 2. It hardcodes the password `Password1` for all 1000 accounts. Fine for a lab. Flag it in your
>    README as something you would never do in production, because a single shared credential
>    across 1000 accounts with no rotation is exactly the failure mode PAM exists to prevent.
> 3. **You will see red errors.** The video addresses this at 43:47: `names.txt` contains duplicate
>    names, so some creations fail. Expected, not your fault, and the script completes anyway.

Takes several minutes. Refresh ADUC to watch the `_USERS` OU fill.

📸 `14-powershell-script-running.png`
📸 `15-aduc-users-created.png`

## 15. Snapshot DC-1 before continuing

VirtualBox Manager > right-click DC-1 > **Snapshots > Take**. Name it something like
`DC-1 fully built, pre-client`.

Everything up to this point represents hours of work. The snapshot is cheap insurance.

> **PITFALL: `VBOX_E_INVALID_VM_STATE` when restoring a snapshot.**
> Restoring with **"Create a snapshot of the current machine state"** ticked can throw:
> ```
> Cannot delete the current state of the running machine (machine state: Snapshotting).
> Result Code: VBOX_E_INVALID_VM_STATE (0x80BB0002)
> Component: SessionMachine
> ```
> The error text saying "running machine" is misleading. This happens with the VM fully powered
> off, and each failed attempt leaves behind another stray snapshot.
>
> **Cause:** restore-with-safety-snapshot is a composite operation. VirtualBox takes the snapshot
> first, then restores. The machine state transitions to "Snapshotting" for part one, and part two
> runs its state validity check before that clears. It sees "Snapshotting" and aborts. The
> snapshot half has already committed, which is why each attempt still produces a new snapshot.
>
> **Fix:** do not use the checkbox. Either restore without it, or take a snapshot manually and
> then restore as a separate second action.
>
> **The restore may have worked despite the error.** Check the snapshot tree: if the newer
> snapshot appears as a **sibling** of the earlier one rather than a child, the machine was
> restored to a parent state and then snapshotted again, which means the restore succeeded. Better
> test regardless: boot the VM and confirm the environment is where you left it.

> Separately, VirtualBox will not restore a snapshot while the VM is running. Power it off first.

> **On snapshotting domain controllers generally:** this is safe here because there is a single DC
> with no replication partners. In production, reverting a DC causes USN rollback, where the
> restored DC reuses update sequence numbers its partners have already seen and quietly stops
> replicating. Additionally, once CLIENT-1 is domain-joined, rolling back DC-1 alone breaks the
> computer account password and the client throws *"The trust relationship between this
> workstation and the primary domain failed."* Snapshot both machines together, or rejoin the
> client after a rollback.

---

# Part 2. Build CLIENT-1

Everything above had to exist first. The client depends on the DHCP server for its address, the
DC for its gateway, and the domain for something to join. Building it earlier produces a machine
sitting on an APIPA address that can do nothing.

Video 45:00.

## 16. Create the CLIENT-1 VM

VirtualBox > **New**:

- **Name:** `CLIENT-1`
- **ISO Image:** Other... > the Windows 11 Enterprise ISO
- **Tick "Skip Unattended Installation".** Same wizard as step 1. It appears on this VM too, and
  the video does not show it.
- Version: Windows 11 (64-bit)
- **Memory:** 4096 MB, **CPUs:** 2, **Disk:** 60 GB

Then, before booting:

- **Settings > Network > Adapter 1 > Internal Network > `intnet`.** This is its only adapter.
  Adapter 2 stays disabled.
- **Settings > System > Motherboard > Enable EFI**, and set **TPM v2.0**.

Windows 11 refuses to install without EFI and TPM 2.0. This is a VM hardware setting, not
something the installer can work around.

📸 `16-client1-vm-settings.png`
📸 `17-client1-network-adapter.png`

## 17. Install Windows 11 Enterprise

The video installs Windows 10 here and does not apply. Two things differ.

**Product key:** choose "I don't have a product key."

**Edition: Windows 11 Enterprise or Pro. Never Home.** Home cannot join a domain. The video calls
this out at 48:06 for Windows 10 and it is still true.

Custom install, use the whole disk.

> **PITFALL: creating a local account on Windows 11 Enterprise. Do not use `OOBE\BYPASSNRO`.**
> The widely documented workaround (Shift+F10, then `OOBE\BYPASSNRO`, or `start ms-cxh:localonly`)
> targets a "Let's connect you to a network" screen. **On Enterprise that screen never appears.**
> Setup goes straight to a sign-in prompt, and the command-line bypass has nothing to act on.
>
> **Instead:** click **"Sign-in options."** You get two choices, one for face/fingerprint/PIN/
> security key, and one labeled **"Domain join instead."**
>
> Click **"Domain join instead."** Despite the name it does **not** join a domain at this stage.
> It drops you into the standard local account creation screen, "Who's going to use this device?"
> Create a local user there, for example `labuser`.
>
> **Why the difference:** `OOBE\BYPASSNRO` and `ms-cxh:localonly` are workarounds for Windows 11
> **Home and Pro**, which hide the local account option and force a network connection plus a
> Microsoft account. **Enterprise** exposes the local path directly, because enterprise
> deployments are expected to join a domain rather than use consumer accounts.
>
> **Takeaway: on Windows 11 Enterprise, check Sign-in options first.** The command-line bypass is
> unnecessary.

> **PITFALL: the security questions are credentials. Do not answer them honestly.**
> After setting the local password, setup requires three security questions. The prompt says "make
> sure your answers are unforgettable," which nudges you toward real personal answers.
>
> Answer `lab` to all three, or something equally fake and consistent.
>
> Three reasons. First, security question answers **are** credentials. First pet, childhood city,
> and first school are the same questions used for password recovery on real bank and email
> accounts, and reusing real answers on a throwaway lab VM spreads them somewhere they do not
> belong. Second, this build is being documented publicly and OOBE screenshots can capture them.
> Third, this local account is used for about fifteen minutes. Every login after the domain join
> uses a domain account.
>
> The same principle applies to `<LabPassword>` throughout this build. It appears in plaintext in
> configuration steps and must never be a password used anywhere real.

> **PITFALL: the update download near the end of OOBE.**
> Setup enters "Step 1 of 3: Downloading" with a progress bar and an "Update later" link. Clicking
> it produces a second warning that the PC will restart immediately and the newest features and
> security updates will not be available.
>
> **Click "Update later."** The restart is a normal part of finishing OOBE, not a penalty. Nothing
> in this lab depends on those updates, and the download runs through DC-1's NAT over a
> virtualized link, so it is slow with no payoff before the domain join. You can run Windows
> Update from the desktop afterward if you want.
>
> **This is also good news, and worth understanding rather than just waiting on.** CLIENT-1's only
> adapter is on the isolated `intnet` network. For it to download anything at all, four things had
> to be working simultaneously: DC-1 running, DHCP issuing the client an address, DNS resolving
> Microsoft's servers, and RRAS/NAT routing the traffic out through DC-1's internet adapter. This
> is effectively the step 18 verification happening early, without running a command. Worth a
> screenshot as evidence of the routing chain end to end.

📸 `18-client1-installed.png`

## 18. Install Guest Additions on CLIENT-1

Full click path again, deliberately. You did this on DC-1 in step 4, but that was several hours
and fourteen steps ago.

1. VM menu bar: **Devices > Insert Guest Additions CD image**
2. Inside Windows: **File Explorer > This PC > CD Drive (D:) VirtualBox Guest Additions**
3. Run **VBoxWindowsAdditions-amd64.exe**
4. Next through the prompts, then **Install**
5. If Windows Security prompts about an Oracle device driver, choose **Install**
6. Reboot when offered

The installer again defaults to `C:\Program Files\Oracle\VirtualBox Guest Additions`. That `C:\`
is the guest's virtual drive, same as in step 4.

## 19. Verify networking (video 50:00)

Open Command Prompt on CLIENT-1:

```
ipconfig
```

You want:

- An IPv4 address in **172.16.0.100 to 172.16.0.200**
- **Default Gateway: 172.16.0.1**

> If the gateway is missing, that is the Scope Options problem from step 12. Fix it on DC-1, then
> on the client run `ipconfig /renew`.

Then:

```
ping google.com
```

A reply proves the entire chain at once: client, to DC gateway, through NAT, out to the internet,
and DNS resolved the name. A single successful ping validates four separate pieces of
configuration.

```
ping smithlab.local
```

Should resolve to `172.16.0.1`.

📸 `19-client1-ipconfig.png`
📸 `20-client1-ping-google.png`

## 20. Rename and join the domain (video 54:00)

Right-click Start > **System** > **Rename this PC (advanced)** > **Change**:

- Computer name: `CLIENT-1`
- Member of > **Domain** > `smithlab.local`
- Credentials: `keenan_admin` and `<LabPassword>`

You get a welcome message, then restart.

If the domain is not found, DNS is the cause. Confirm `ipconfig /all` shows `172.16.0.1` as the
DNS server. Domain controllers are located through DNS SRV records, so a client pointed at the
wrong resolver cannot find a domain that is running fine.

📸 `21-client1-domain-join.png`

## 21. Verify from the DC side (video 56:26)

On DC-1:

- **DHCP > IPv4 > Scope > Address Leases.** CLIENT-1's lease is listed.
- **ADUC > Computers container.** CLIENT-1 appears.

📸 `22-dhcp-lease.png`
📸 `23-aduc-computers.png`

Optional: create a `_CLIENTS` OU and drag CLIENT-1 into it.

## 22. Log in as a domain user (video 58:33)

On CLIENT-1, click **Other user**. The screen should read *Sign in to: SMITHLAB*.

Log in as one of the generated accounts, formatted `<firstinitial><lastname>`, with the script's
password. Use the account generated from your own name if you added it to `names.txt`.

First login builds the profile and takes a minute. Then:

```
whoami
```

Returns `smithlab\<username>`.

📸 `24-client1-domain-login.png`

**That is the lab working end to end.** That one command output depends on AD DS, DNS, DHCP,
RRAS/NAT, the domain join, and Kerberos authentication all functioning at once.

---

# Appendix A. Troubleshooting index

Everything that went wrong during the build, in one place.

| Symptom | Step | Cause and fix |
|---|---|---|
| VM creation offers "Unattended Guest OS Install" with `vboxuser` / `changeme` | 1, 16 | "Skip Unattended Installation" unchecked. Go Back and tick it. |
| ISO dropdown is empty | 1 | It is a file picker, not a list. Click **Other...** |
| Host system drive filling during install | 0.1 | Default machine folder is on C:. Change it in Preferences and type the path. |
| VM listed as Inaccessible, `VERR_PATH_NOT_FOUND` | 0.2 | Stale registration entry. Remove > **Remove only**. |
| Guest Additions wants to install to `C:\Program Files` | 4, 18 | That is the guest's C:, not the host's. Proceed normally. |
| RRAS interface list is empty | 10 | Known bug. Cancel, reopen Routing and Remote Access, run again. Reboot if needed. |
| "The address range you have specified is too large for a single scope" | 11 | Digit in the wrong octet of the End IP. Answer **No** to the superscope offer and retype all four fields. |
| `003 Router` not visible under Server Options | 12 | Wrong node. It is under **Scope Options**. Server Options being empty is correct. |
| Scope icon red | 12 | Created but not activated. Right-click scope > Activate. |
| Server node red | 12 | Not authorized. Right-click server > Authorize > Refresh. |
| Red errors during the user-creation script | 14 | Duplicate names in `names.txt`. Expected. The script completes. |
| `VBOX_E_INVALID_VM_STATE` restoring a snapshot | 15 | The "Create a snapshot of the current machine state" checkbox. Restore without it. |
| Cannot restore a snapshot at all | 15 | The VM is running. Power it off first. |
| "This PC can't run Windows 11" | 16 | EFI and TPM 2.0 not enabled in **Settings > System**. |
| `OOBE\BYPASSNRO` does nothing on Windows 11 Enterprise | 17 | Wrong workaround for this edition. Use **Sign-in options > "Domain join instead."** |
| Client has an APIPA (169.254.x.x) address | 19 | DHCP is not reachable. Confirm DC-1 is running, the scope is active and authorized, and both VMs are on `intnet`. |
| Client has an address but no default gateway | 19 | `003 Router` missing from Scope Options. Fix on DC-1, then `ipconfig /renew`. |
| `ping google.com` fails but the client has a gateway | 19 | RRAS/NAT not configured or not started. Recheck step 10. |
| Domain not found during join | 20 | DNS. `ipconfig /all` must show `172.16.0.1` as the DNS server. |
| "The trust relationship between this workstation and the primary domain failed" | 15 | DC-1 was rolled back to a snapshot taken before the client joined. Rejoin the client. |

---

# Appendix B. Evaluation licenses

Both operating systems are free Microsoft evaluations with time limits.

| | Length |
|---|---|
| Windows Server 2022 Standard Evaluation | 180 days |
| Windows 11 Enterprise Evaluation | 90 days |

Not a practical constraint. The build, extension labs, and documentation take days, not months.

**What happens at expiry.** Neither is destructive and your data and configuration stay intact.
Server 2022 begins shutting down roughly every hour and displays expiry notices. Windows 11 nags
and disables personalization features.

**Workarounds, cheapest first:**

1. **Rearm.** From an elevated prompt: `slmgr /rearm`. Server 2022 permits up to five rearms,
   roughly three years total. Windows 11 Enterprise permits fewer. This is Microsoft's own
   supported mechanism.
2. **Snapshot before expiry and roll back.** The eval clock lives inside the guest, so restoring a
   pre-expiry snapshot restores the remaining days. Observe the DC snapshot caveats in step 15.
3. **Rebuild from a fresh ISO.** Free and unlimited.

**Downgrading to Server 2019 or Windows 10 is not a fix.** Those evaluations expire identically,
and Windows 10 reached end of support in October 2025.

---

# Appendix C. What this lab intentionally does wrong

Declare these honestly in your README. Knowing why they are wrong is more valuable than pretending
they were not done.

| Shortcut | Why it is acceptable here | Why it is not acceptable in production |
|---|---|---|
| IE Enhanced Security Configuration disabled | Isolated lab, needed to download the script | Removes a hardening control on the most privileged server in the environment |
| `Set-ExecutionPolicy Unrestricted` | Needed to run an unsigned script | Any script can then run unsigned. Use `RemoteSigned` at minimum |
| All 1000 generated users share one hardcoded password | Filler accounts with no real access | One shared credential, no rotation, no complexity variance |
| Single domain controller | Nothing to replicate with | No redundancy. One failure takes down authentication for the whole domain |
| `.local` domain suffix | Long-standing lab convention | Microsoft discourages it for new deployments. `.local` collides with mDNS/Bonjour. Use a subdomain of a domain you own |
| Windows Firewall left at defaults, no GPOs | Out of scope for a first build | A domain with no Group Policy is not actually managing anything |

---

# Next

Extension labs reuse `DC-1` and `CLIENT-1` with nothing new to install. Each is roughly one
session, and each maps onto A+ Core 2 Security and Operating Systems objectives, so they double
as exam preparation.

Each lab has its own runbook in [`extensions/`](extensions/), written to the same standard as
this one: every step verified by hand, every failure documented.

| Lab | Runbook | Status |
|---|---|---|
| DNS records, the local resolver cache, and CNAME aliases | [`extensions/01-dns-records-and-cache.md`](extensions/01-dns-records-and-cache.md) | Complete |
| Network file shares, NTFS versus share permissions, and a security group | | Planned |
| Account lockout policy via Group Policy, enabling and disabling accounts, and log review | | Planned |
| Windows Firewall rules and a Wireshark capture | | Planned |

The DNS lab in particular produced four findings that do not appear in any tutorial version of
it, because they only exist in a local, multihomed, snapshot-driven environment. Those are written
up in the [README](README.md) and in [`BUILD-LOG.md`](BUILD-LOG.md).
