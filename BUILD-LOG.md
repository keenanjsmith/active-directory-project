# Build Log

Every point in this build where something broke, behaved unexpectedly, or where the reference
video and my own runbook turned out to be wrong.

These were written during the build, not reconstructed afterward. The wording is mostly as I
typed it at the time. Most write-ups of this lab document the path that worked. This is the one
that actually happened.

Each entry is cross-referenced to the step in [RUNBOOK.md](RUNBOOK.md) where it occurred.

Reference video: Josh Madakor, https://youtu.be/MHsI8hJmggI
It was recorded on Server 2019, Windows 10, and an older VirtualBox. Several entries below exist
purely because this build used Server 2022, Windows 11, and VirtualBox 7.

---

### VirtualBox default machine folder points at nearly-full C: drive
**Runbook step 0.1**

WHAT I WAS DOING:
About to create the first VM after installing/opening VirtualBox.

WHAT HAPPENED:
Default Machine Folder was C:\Users\<user>\VirtualBox VMs. C: had only 4.7 GB free.
Two 60 GB dynamic disks would have filled the system drive mid-install.

WHAT I TRIED:
Checked free space with dir before creating anything.

WHAT ACTUALLY FIXED IT:
File > Preferences > General > Default Machine Folder, typed D:\VirtualBox VMs
directly into the Folder field instead of using the browser. Clicked OK to commit.
Verified by reopening Preferences.

NOTE: the folder browser opened inside OneDrive - Personal by default. Do NOT put VMs
in a synced OneDrive folder. It will try to sync 60 GB disks, cause sync conflicts on
running VMs, and risk corruption.

---

### Old VMs showing "Inaccessible" with VERR_PATH_NOT_FOUND
**Runbook step 0.2**

WHAT I WAS DOING:
Opening VirtualBox for the first time in a while.

WHAT HAPPENED:
Two old VMs (Kali Linux, Metasploitable2) listed as Inaccessible.
Error: Runtime error opening '...\.vbox' for reading VERR_PATH_NOT_FOUND (Path not found).
Result Code: E_FAIL (0x80004005)

WHAT I TRIED:
Ran dir on the VM folder. Came back 0 files. The VM files were already gone.

WHAT ACTUALLY FIXED IT:
Right-click each dead entry > Remove > "Remove only" (NOT "Delete all files").
Registration entry cleared, list clean.

TIME LOST: ~3 min

---

### ISO organization
**Runbook step 0.3**

WHAT I WAS DOING:
ISOs downloaded to the browser default location.

WHAT HAPPENED:
Not an error, but ISOs scattered in Downloads make VM creation slower and risk being
deleted by cleanup tools.

WHAT ACTUALLY FIXED IT:
Created D:\ISOs\ and moved both files there with clear names:
  WindowsServer2022_Eval.iso
  Windows11_Enterprise_Eval.iso
ISOs are read-only install media, they can live anywhere. Keeping them out of the
VirtualBox machine folder avoids mixing them with VM disk files.

TIME LOST: ~1 min

---

### ISO Image dropdown appears empty when creating a VM
**Runbook step 1**

WHAT I WAS DOING:
Creating DC-1, reached the "Virtual machine Name and Operating System" screen.

WHAT HAPPENED:
Clicked the ISO Image dropdown expecting to see the ISOs I had just downloaded.
It was blank. Assumed the ISOs needed to be exported, extracted, or converted first.

WHAT I TRIED:
Looked for an import/export option.

WHAT ACTUALLY FIXED IT:
Nothing needed converting. An ISO is already a single ready-to-use file. The dropdown
is not a list of detected ISOs, it is a file picker that starts empty.
ISO Image dropdown > Other... > browse to the .iso file.

Once the Server 2022 ISO was attached, Version auto-corrected from "Windows 10 (64-bit)"
to "Windows 2022 (64-bit)" and the Edition field populated. That auto-detection is a good
confirmation the ISO is readable.

NOTE: Windows Explorer may list .iso files with Type "WinRAR" if WinRAR is installed.
Cosmetic only. VirtualBox reads them fine.

TIME LOST: ~2 min

---

### Guest Additions installer defaults to C:\ and it looks like it is filling the host system drive
**Runbook step 4**

WHAT I WAS DOING:
Installing VirtualBox Guest Additions inside DC-1 after the Server 2022 install finished.

WHAT HAPPENED:
The installer's Destination Folder defaulted to:
  C:\Program Files\Oracle\VirtualBox Guest Additions

My host C: drive is nearly full (about 4.7 GB free) and all lab files were deliberately
moved to D:. It looked like the install was about to write to the wrong drive and undo
the earlier storage planning.

WHAT I TRIED:
Stopped before clicking Next to check whether the path needed changing.

WHAT ACTUALLY FIXED IT:
Nothing needed changing. That C:\ is the GUEST's virtual C: drive, not the host's.
It lives inside DC-1's 60 GB virtual disk file at D:\VirtualBox VMs\DC-1\.
Every Windows guest labels its own system drive C: no matter what the host uses.

Confirmation: the installer reported "Space available: 49.7 GB". The host C: only had
4.7 GB free, so the number could only be coming from the VM's own virtual disk.

RULE OF THUMB: once you are clicking inside the VM window, all drive letters are the
guest's. Host paths are only relevant in the VirtualBox Manager and settings dialogs.

TIME LOST: ~2 min

---

### DHCP rejects scope: "address range too large for a single scope"
**Runbook step 11**

WHAT I WAS DOING:
Creating the DHCP scope on DC-1 for the internal network (intended range
172.16.0.100 - 172.16.0.200).

WHAT HAPPENED:
The New Scope Wizard threw:
"The address range you have specified is too large for a single scope."
It then offered to create a superscope containing 3 scopes with 256 addresses each,
Yes or No.

WHAT I TRIED:
Answered No, went Back, and retyped all four fields.

WHAT ACTUALLY FIXED IT:
A digit had landed in the wrong octet of the End IP address, so DHCP was reading a range
spanning multiple /24 networks (roughly 768 addresses) instead of 101. The IP entry boxes
auto-advance between octets as you type, which makes this very easy to do without noticing.

Correct values:
  Start IP:    172.16.0.100
  End IP:      172.16.0.200
  Length:      24
  Subnet mask: 255.255.255.0

Do NOT accept the superscope. A superscope is for serving multiple subnets from one DHCP
server, which this lab does not need. Accepting it would have silently built the wrong
topology instead of surfacing the typo.

ALSO NOTE: Length and Subnet mask are linked fields. Editing one rewrites the other.
Set Length to 24 and confirm the mask reads 255.255.255.0.

TIP: retype all four fields from scratch rather than editing them. Faster than hunting
for the stray digit.

TIME LOST: ~1 min

---

### Router option (003) is under Scope Options, not Server Options
**Runbook step 12**

WHAT I WAS DOING:
Verifying after creating the DHCP scope that the default gateway option had actually saved,
since this is a known failure point that breaks client networking later.

WHAT HAPPENED:
Clicked IPv4 > Server Options and the right pane showed only descriptive help text with no
options listed. Looked like the router option was missing entirely.

WHAT I TRIED:
Looked around Server Options for a 003 Router entry. Nothing there.

WHAT ACTUALLY FIXED IT:
Nothing was wrong. The New Scope Wizard writes DHCP options to SCOPE Options, not SERVER
Options. Server Options was empty because the wizard never touches it.

Correct place to look: expand the scope, then click Scope Options (below Reservations).
Expected entries:
  003 Router            172.16.0.1
  006 DNS Servers       172.16.0.1
  015 DNS Domain Name   smithlab.local

DIFFERENCE: Server Options are server-wide defaults inherited by all scopes. Scope Options
apply to one scope and override the server default. A single-subnet lab only needs it in one
place and the wizard chooses Scope Options.

If Scope Options is ALSO empty, the router page in the wizard did not save. Fix:
right-click Scope Options > Configure Options > check 003 Router > enter 172.16.0.1 > Add.

ALSO VERIFY HERE:
  - Scope icon green (up arrow), not red. Red = created but not activated.
  - Server node icon green. Red = not authorized. Right-click server > Authorize > Refresh.

TIME LOST: ~1 min

NOTE: this one was my runbook's error, not a VirtualBox or Windows quirk. The runbook sent me
to the wrong node. Corrected in the published version.

---

### VirtualBox throws VBOX_E_INVALID_VM_STATE when restoring a snapshot with the safety checkbox enabled
**Runbook step 15**

WHAT I WAS DOING:
Verifying that a snapshot of DC-1 would restore correctly, before relying on it as a rollback
point for the rest of the build.

WHAT HAPPENED:
Restoring the snapshot with "Create a snapshot of the current machine state" ticked threw:
  Cannot delete the current state of the running machine (machine state: Snapshotting).
  Result Code: VBOX_E_INVALID_VM_STATE (0x80BB0002)
  Component: SessionMachine

The VM was fully Powered Off the entire time. The error text claiming "running machine" is
misleading. Repeated attempts each left behind an additional stray snapshot.

WHAT I TRIED:
Confirmed in VirtualBox Manager that DC-1 showed Powered Off, not Running or Snapshotting.
Retried, got the identical error and another stray snapshot.

WHAT ACTUALLY FIXED IT:
The checkbox is the cause. Restore-with-safety-snapshot is a composite operation: VirtualBox
takes the snapshot first, then restores. The machine state transitions to "Snapshotting" for
part one, and part two runs its state validity check before that clears. It sees "Snapshotting"
and aborts. The snapshot half has already committed, which is why each failed attempt still
produced a new snapshot.

FIX: do not use the checkbox. Either
  (a) restore without it, or
  (b) take a snapshot manually, then restore as a separate second action.

THE RESTORE ACTUALLY WORKED DESPITE THE ERROR. Evidence is in the snapshot tree: the later
snapshot appeared as a SIBLING of the earlier one rather than a child. That branching only
occurs when the machine has been restored to a parent state and then snapshotted again.

BETTER TEST ANYWAY: boot the VM and confirm the environment is where you left it. Reading the
tree structure and booting the machine tell you more than the dialog does.

---

### VirtualBox hijacks CLIENT-1 creation with the Unattended Guest OS Install wizard
**Runbook step 16**

WHAT I WAS DOING:
Creating the CLIENT-1 VM. Expected the wizard to go from the name/ISO screen straight to
hardware settings (memory, CPU) the way it did for DC-1 and the way the video shows.

WHAT HAPPENED:
After clicking Next, VirtualBox jumped to an "Unattended Guest OS Install Setup" page instead,
pre-filled with:
  Username: vboxuser
  Password: changeme
  Hostname: CLIENT-1
  Domain Name: myguest.virtualbox.org
The prior screen had also noted: "This OS type can be installed unattendedly. The install will
start after this wizard is closed."

WHAT I TRIED:
Stopped, since the video never shows this screen and none of those values are wanted.

WHAT ACTUALLY FIXED IT:
The "Skip Unattended Installation" checkbox on the previous page was unchecked. VirtualBox
auto-detects a recognized ISO and offers to run Windows setup for you. Go Back, tick
"Skip Unattended Installation," and Next proceeds to Hardware as expected.

WHY NOT USE IT: unattended install creates the vboxuser/changeme account, applies an unwanted
DNS suffix, and runs the entire Windows setup without interaction. That skips the OOBE screens
where the Windows 11 local-account bypass is required, and produces no documentation of the
install. It also does nothing for the TPM requirement, which is a VM hardware setting.

NOTE: with Skip ticked, the Edition dropdown greys out. Expected. Edition gets selected inside
the Windows installer instead.

VIDEO DIVERGENCE: Josh's VirtualBox version predates this wizard, so the video never shows it.
Anyone following along on VirtualBox 7.x will hit this on both VMs.

TIME LOST: ~1 min

---

### Windows 11 Enterprise: use "Domain join instead" for a local account, not OOBE\BYPASSNRO
**Runbook step 17**

WHAT I WAS DOING:
Reaching the account setup screen during Windows 11 Enterprise OOBE on CLIENT-1, trying to
create a local account instead of signing in with a Microsoft account.

WHAT HAPPENED:
The documented workaround (Shift+F10, then OOBE\BYPASSNRO) did not apply. Setup never presented
a "Let's connect you to a network" screen to bypass. Instead it went straight to a sign-in
prompt. Clicking "Sign-in options" offered only two choices:
  - Face / fingerprint / PIN / security key
  - Domain join instead

WHAT I TRIED:
Looked for the network screen the bypass targets. It never appeared.

WHAT ACTUALLY FIXED IT:
Click "Domain join instead." Despite the name, it does NOT join a domain at this stage. It
drops you into the standard local account creation screen ("Who's going to use this device?").
Create the local user there, e.g. labuser, plus three security questions.

WHY THE DIFFERENCE:
OOBE\BYPASSNRO and ms-cxh:localonly are workarounds for Windows 11 HOME and PRO, which hide the
local account option and force a network connection plus Microsoft account. ENTERPRISE exposes
the local path directly under Sign-in options, because enterprise deployments are expected to
join a domain rather than use consumer accounts.

TAKEAWAY: on Windows 11 Enterprise, check Sign-in options FIRST. The command-line bypass is
unnecessary. Josh's video predates all of this, since Windows 10 OOBE offered a local account
without any workaround.

---

### Local account security questions: use throwaway answers, not real ones
**Runbook step 17**

WHAT I WAS DOING:
Creating the local account (labuser) during Windows 11 OOBE on CLIENT-1. After setting the
password, setup required three security questions with free-form answers.

WHAT HAPPENED:
Not an error, but a decision point that is easy to get wrong out of habit. The prompt says
"make sure your answers are unforgettable," which nudges you toward using real personal
answers.

WHAT I DID:
Answered "lab" for all three.

WHY THIS MATTERS:
1. Security question answers ARE credentials. The same questions (first pet, childhood city,
   first school) are used for password recovery on real bank and email accounts. Reusing real
   answers on a throwaway lab VM spreads them somewhere they do not belong.
2. This lab is being documented publicly. Screenshots taken during OOBE could capture those
   answers on screen.
3. This local account gets used for roughly fifteen minutes. Every login after the domain join
   uses a domain account, so the local account and its recovery questions become irrelevant.

RULE OF THUMB: in a lab, use obviously fake and consistent values. Personalizing throwaway
infrastructure adds no value and creates real exposure. Focus on how the system works, not on
making the VM feel like a personal machine.

RELATED: the same principle applies to the lab password throughout this build. It appears in
plaintext in configuration steps and must never be a password used anywhere real.

---

### OOBE offers an update download near the end of Windows 11 setup
**Runbook step 17**

WHAT I WAS DOING:
Finishing Windows 11 OOBE on CLIENT-1 after creating the local account and clearing the
security questions.

WHAT HAPPENED:
Setup went into "Step 1 of 3: Downloading" with a progress bar and an "Update later" link.
Clicking "Update later" produced a second warning: the PC will restart immediately, and the
newest features and security updates will not be available until the PC updates.

Unclear whether this was a required part of setup or something skippable, so I stopped.

WHAT ACTUALLY FIXED IT:
Click "Update later." The restart is a normal part of finishing OOBE, not a penalty. Nothing
in this lab depends on those updates. Windows Update can be run from the desktop afterward if
wanted.

Reason to skip: this downloads through DC-1's NAT over a virtualized link, so it is slow, and
there is no payoff before a domain join.

WORTH NOTING (positive signal, not a problem):
The fact that CLIENT-1 can download ANYTHING at this stage proves the infrastructure is
already working. Its only adapter is on the isolated intnet network. For the download to run,
all four of these had to be functioning simultaneously:
  - DC-1 running
  - DHCP issued CLIENT-1 an address
  - DNS resolved Microsoft's update servers
  - RRAS/NAT routed the traffic out through DC-1's internet adapter
This is effectively the step 19 verification happening early, without running a command.
Worth screenshotting as evidence of the routing chain end to end.

TIME LOST: ~1 min

---

### "Install Guest Additions" is not self-explanatory the second time
**Runbook step 18**

WHAT I WAS DOING:
Reached the CLIENT-1 desktop after OOBE. Runbook said to install Guest Additions, same as was
done on DC-1 earlier in the build.

WHAT HAPPENED:
The instruction "install Guest Additions" was written as a one-liner on the assumption it had
been done once already on DC-1 and would be remembered. It was not. Enough time and steps had
passed that the actual click path was gone, and the step stalled.

WHAT ACTUALLY FIXED IT:
Full click path, written out:
  1. VM menu bar: Devices > Insert Guest Additions CD image
  2. Inside Windows: File Explorer > This PC > CD Drive (D:) VirtualBox Guest Additions
  3. Run VBoxWindowsAdditions-amd64.exe
  4. Next through the prompts, Install
  5. If Windows Security prompts about an Oracle device driver, choose Install
  6. Reboot when offered

Note: the installer defaults to C:\Program Files\Oracle\VirtualBox Guest Additions. That C:\
is the GUEST's virtual drive, not the host's. Same point as the earlier DC-1 entry.

WHAT IT DOES: fixes mouse capture (no more Right Ctrl to release), enables dynamic screen
resolution when resizing the VM window, and enables shared clipboard. Quality of life, not
functionality. The lab works without it, but screenshots and general use are much worse.

TIME LOST: ~1 min

GENERAL PRINCIPLE, and the reason the runbook is written the way it is: instructions that
assume prior familiarity fail for the exact audience this is written for. Someone finding this
via a portfolio is doing it cold, on their first pass, possibly without the video. Anywhere I
had to ask a question during this build is a place the documentation was insufficient. Every
repeated procedure gets the full click path each time it appears.

---

## Reference: evaluation license timelines

Not a problem encountered, but a question that came up mid-build and is worth recording.

Both operating systems in this lab are free Microsoft evaluation editions with time limits.

  Windows Server 2022 (Standard Eval): 180 days
  Windows 11 Enterprise (Eval):         90 days

Not a practical constraint. The full build, extension labs, and documentation take days, not
months.

WHAT HAPPENS AT EXPIRY (neither is destructive):
  Server 2022  - begins shutting down roughly every hour, displays expiry notices
  Windows 11   - nags, disables personalization features
  Data and configuration remain intact in both cases.

WORKAROUNDS, cheapest first:
  1. Rearm. From an elevated prompt: slmgr /rearm
     Server 2022 permits up to 5 rearms (~3 years total). Windows 11 Enterprise permits fewer.
     This is Microsoft's own supported mechanism.
  2. Snapshot before expiry and roll back. The eval clock lives inside the guest, so restoring
     a pre-expiry snapshot restores the remaining days.
  3. Rebuild from a fresh ISO. Free and unlimited.

DOWNGRADING TO SERVER 2019 / WINDOWS 10 IS NOT A FIX. Those evals expire identically, and
Windows 10 reached end of support in October 2025.
