# BUILD LOG — AD Lab

> **This is a scratch file, not a document.** Dump raw notes here while building. Nobody reads this version. Claude turns it into the polished troubleshooting sections later. Save it as `BUILD-LOG.md` in `D:\` next to your lab folder. Keep it open in a second window.

---

## How to use this (10 seconds per entry)

When something breaks or surprises you:

1. **Screenshot the error** → save as `issue-<step#>-<short-name>.png` in `docs\images\` (e.g. `issue-05-tpm-error.png`)  
2. **Alt-tab here, fill in 2 lines, go back to building.** That's it.

**Do NOT write polished prose.** Fragments are fine. Typos are fine. Swearing is fine — I'll clean it. The only thing that matters is capturing *what happened* and *what fixed it* before you forget.

**Copy the error text verbatim if you can.** Exact error strings are what other people search for, so they're the single most valuable thing in the final write-up.

---

## Entry template — copy this block for each issue

\#\#\# \[STEP \_\] — short title

WHAT I WAS DOING:

WHAT HAPPENED:

(exact error text if you have it)

WHAT I TRIED:

WHAT ACTUALLY FIXED IT:

TIME LOST: \~\_\_ min

SCREENSHOT: issue-\_\_-\_\_\_\_.png

Only **WHAT HAPPENED** and **WHAT ACTUALLY FIXED IT** are required. The rest is bonus.

---

## Also worth logging (not just errors)

- **Decisions you made** — the installer offered options and you picked one. Why?  
- **Things that took way longer than expected** — even if nothing broke. Real time estimates are rare in tutorials and genuinely useful to readers.  
- **Anything that differed from the video** beyond what my runbook already flagged.  
- **Stuff you didn't understand at the time** but figured out later. That's the best teaching material in the whole document.

---

## Session log

Start a new dated block each time you sit down. Helps reconstruct order later.

\=== SESSION 1 — \[DATE\] — started \_\_:\_\_ \===

---

# ENTRIES

### \[STEP \_\] —

WHAT I WAS DOING:

WHAT HAPPENED:

WHAT I TRIED:

WHAT ACTUALLY FIXED IT:

TIME LOST: \~\_\_ min SCREENSHOT:

---

### \[STEP \_\] —

WHAT I WAS DOING:

WHAT HAPPENED:

WHAT I TRIED:

WHAT ACTUALLY FIXED IT:

TIME LOST: \~\_\_ min SCREENSHOT:

---

### \[STEP \_\] —

WHAT I WAS DOING:

WHAT HAPPENED:

WHAT I TRIED:

WHAT ACTUALLY FIXED IT:

TIME LOST: \~\_\_ min SCREENSHOT:

---

\=== SESSION 1 — 7/23/2026 — prep/setup \===

\#\#\# \[PREP\] — VirtualBox default machine folder points at nearly-full C: drive

WHAT I WAS DOING:  
About to create the first VM after installing/opening VirtualBox.

WHAT HAPPENED:  
Default Machine Folder was C:\\Users\\\<user\>\\VirtualBox VMs. C: had only 4.7 GB free.  
Two 60 GB dynamic disks would have filled the system drive mid-install.

WHAT I TRIED:  
Checked free space with dir before creating anything.

WHAT ACTUALLY FIXED IT:  
File \> Preferences \> General \> Default Machine Folder, typed D:\\VirtualBox VMs  
directly into the Folder field instead of using the browser. Clicked OK to commit.  
Verified by reopening Preferences.

NOTE: the folder browser opened inside OneDrive \- Personal by default. Do NOT put VMs  
in a synced OneDrive folder. It will try to sync 60 GB disks, cause sync conflicts on  
running VMs, and risk corruption.

TIME LOST: \~\_\_ min  
SCREENSHOT: \_\_

\---

\#\#\# \[PREP\] — Old VMs showing "Inaccessible" with VERR\_PATH\_NOT\_FOUND

WHAT I WAS DOING:  
Opening VirtualBox for the first time in a while.

WHAT HAPPENED:  
Two old VMs (Kali Linux, Metasploitable2) listed as Inaccessible.  
Error: Runtime error opening '...\\.vbox' for reading VERR\_PATH\_NOT\_FOUND (Path not found).  
Result Code: E\_FAIL (0x80004005)

WHAT I TRIED:  
Ran dir on the VM folder. Came back 0 files. The VM files were already gone.

WHAT ACTUALLY FIXED IT:  
Right-click each dead entry \> Remove \> "Remove only" (NOT "Delete all files").  
Registration entry cleared, list clean.

TIME LOST: \~\_3\_ min  
SCREENSHOT: \_\_

\---

\#\#\# \[STEP 2\] — ISO Image dropdown appears empty when creating a VM

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
ISO Image dropdown \> Other... \> browse to the .iso file.

Once the Server 2022 ISO was attached, Version auto-corrected from "Windows 10 (64-bit)"  
to "Windows 2022 (64-bit)" and the Edition field populated. That auto-detection is a good  
confirmation the ISO is readable.

NOTE: Windows Explorer may list .iso files with Type "WinRAR" if WinRAR is installed.  
Cosmetic only. VirtualBox reads them fine.

TIME LOST: \~\_2\_ min  
SCREENSHOT: \_\_

\---

\#\#\# \[PREP\] — ISO organization

WHAT I WAS DOING:  
ISOs downloaded to the browser default location.

WHAT HAPPENED:  
Not an error, but ISOs scattered in Downloads make VM creation slower and risk being  
deleted by cleanup tools.

WHAT ACTUALLY FIXED IT:  
Created D:\\ISOs\\ and moved both files there with clear names:  
  WindowsServer2022\_Eval.iso  
  Windows11\_Enterprise\_Eval.iso  
ISOs are read-only install media, they can live anywhere. Keeping them out of the  
VirtualBox machine folder avoids mixing them with VM disk files.

TIME LOST: \~\_1\_ min  
SCREENSHOT: \_\_

📋 BUILD-LOG ENTRY

\#\#\# \[STEP 4\] — Guest Additions installer defaults to C:\\ and it looks like it is filling the host system drive

WHAT I WAS DOING:  
Installing VirtualBox Guest Additions inside DC-1 after the Server 2022 install finished.

WHAT HAPPENED:  
The installer's Destination Folder defaulted to:  
  C:\\Program Files\\Oracle\\VirtualBox Guest Additions

My host C: drive is nearly full (about 4.7 GB free) and all lab files were deliberately  
moved to D:. It looked like the install was about to write to the wrong drive and undo  
the earlier storage planning.

WHAT I TRIED:  
Stopped before clicking Next to check whether the path needed changing.

WHAT ACTUALLY FIXED IT:  
Nothing needed changing. That C:\\ is the GUEST's virtual C: drive, not the host's.  
It lives inside DC-1's 60 GB virtual disk file at D:\\VirtualBox VMs\\DC-1\\.  
Every Windows guest labels its own system drive C: no matter what the host uses.

Confirmation: the installer reported "Space available: 49.7 GB". The host C: only had  
4.7 GB free, so the number could only be coming from the VM's own virtual disk.

RULE OF THUMB: once you are clicking inside the VM window, all drive letters are the  
guest's. Host paths are only relevant in the VirtualBox Manager and settings dialogs.

TIME LOST: \~\_2\_ min  
SCREENSHOT: it chose c.png

📋 BUILD-LOG ENTRY

\#\#\# \[STEP 7\] — DHCP rejects scope: "address range too large for a single scope"

WHAT I WAS DOING:  
Creating the DHCP scope on DC-1 for the internal network (intended range  
172.16.0.100 \- 172.16.0.200).

WHAT HAPPENED:  
The New Scope Wizard threw:  
"The address range you have specified is too large for a single scope."  
It then offered to create a superscope containing 3 scopes with 256 addresses each,  
Yes or No.

WHAT I TRIED:  
Answered No, went Back, and retyped all four fields.

WHAT ACTUALLY FIXED IT:  
A digit had landed in the wrong octet of the End IP address, so DHCP was reading a range  
spanning multiple /24 networks (roughly 768 addresses) instead of 101\. The IP entry boxes  
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

TIME LOST: \~\_1\_ min  
SCREENSHOT: \_N/A\_

📋 BUILD-LOG ENTRY

\#\#\# \[STEP 7\] — Router option (003) is under Scope Options, not Server Options

WHAT I WAS DOING:  
Verifying after creating the DHCP scope that the default gateway option had actually saved,  
since this is a known failure point that breaks client networking later.

WHAT HAPPENED:  
Clicked IPv4 \> Server Options and the right pane showed only descriptive help text with no  
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
right-click Scope Options \> Configure Options \> check 003 Router \> enter 172.16.0.1 \> Add.

ALSO VERIFY HERE:  
  \- Scope icon green (up arrow), not red. Red \= created but not activated.  
  \- Server node icon green. Red \= not authorized. Right-click server \> Authorize \> Refresh.

TIME LOST: \~\_1\_ min  
SCREENSHOT: \_n/a\_

📋 BUILD-LOG ENTRY

\#\#\# \[SUPPLEMENTAL\] — VirtualBox throws VBOX\_E\_INVALID\_VM\_STATE when restoring a snapshot with the safety checkbox enabled

WHAT I WAS DOING:  
Verifying that a snapshot of DC-1 would restore correctly, before relying on it as a rollback  
point for the rest of the build.

WHAT HAPPENED:  
Restoring the snapshot with "Create a snapshot of the current machine state" ticked threw:  
  Cannot delete the current state of the running machine (machine state: Snapshotting).  
  Result Code: VBOX\_E\_INVALID\_VM\_STATE (0x80BB0002)  
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

TIME LOST: \~\_\_ min  
SCREENSHOT: \_\_

Go **Back** and tick **"Skip Unattended Installation"** on the first screen. That checkbox is why this wizard appeared. You checked it for DC-1, and my runbook failed to repeat the instruction for CLIENT-1. My omission.

Once it's ticked, Next takes you straight to Hardware (memory and CPUs) like Josh's video shows.

**Why not just use the unattended install:** it would create a user called `vboxuser` with password `changeme`, apply `myguest.virtualbox.org` as a DNS suffix, and run the whole Windows setup without you. You'd lose the manual OOBE steps entirely, which is where the local-account bypass lives, and that's one of the more useful things you're documenting. It also wouldn't help with TPM, which is a VM hardware setting, not an installer setting.

After you're through the wizard, before you boot: **Settings → System → enable EFI and set TPM v2.0.** Windows 11 refuses to install without both.

📋 BUILD-LOG ENTRY

\#\#\# \[STEP 10\] — VirtualBox hijacks CLIENT-1 creation with the Unattended Guest OS Install wizard

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

TIME LOST: \~\_1\_ min  
SCREENSHOT: \_n/a\_

📋 BUILD-LOG ENTRY

\#\#\# \[REFERENCE\] — Evaluation license timelines and what happens at expiry

CONTEXT:  
Both OSes in this lab are free Microsoft evaluation editions with time limits, which raises  
the question of whether the lab has to be finished inside that window.

TIMELINES:  
  Windows Server 2022 (Standard Eval): 180 days  
  Windows 11 Enterprise (Eval):         90 days

ANSWER: not a practical constraint. The full build, extension labs, and documentation take  
days, not months.

WHAT HAPPENS AT EXPIRY (neither is destructive):  
  Server 2022  \- begins shutting down roughly every hour, displays expiry notices  
  Windows 11   \- nags, disables personalization features  
  Data and configuration remain intact in both cases.

WORKAROUNDS, cheapest first:  
  1\. Rearm. From an elevated prompt: slmgr /rearm  
     Server 2022 permits up to 5 rearms (\~3 years total). Windows 11 Enterprise permits fewer.  
     This is Microsoft's own supported mechanism.  
  2\. Snapshot before expiry and roll back. The eval clock lives inside the guest, so restoring  
     a pre-expiry snapshot restores the remaining days.  
  3\. Rebuild from a fresh ISO. Free and unlimited.

DOWNGRADING TO SERVER 2019 / WINDOWS 10 IS NOT A FIX. Those evals expire identically, and  
Windows 10 reached end of support in October 2025\.

TIME LOST: n/a  
SCREENSHOT: n/a

📋 BUILD-LOG ENTRY

\#\#\# \[STEP 11\] — Windows 11 Enterprise: use "Domain join instead" for a local account, not OOBE\\BYPASSNRO

WHAT I WAS DOING:  
Reaching the account setup screen during Windows 11 Enterprise OOBE on CLIENT-1, trying to  
create a local account instead of signing in with a Microsoft account.

WHAT HAPPENED:  
The documented workaround (Shift+F10, then OOBE\\BYPASSNRO) did not apply. Setup never presented  
a "Let's connect you to a network" screen to bypass. Instead it went straight to a sign-in  
prompt. Clicking "Sign-in options" offered only two choices:  
  \- Face / fingerprint / PIN / security key  
  \- Domain join instead

WHAT I TRIED:  
Looked for the network screen the bypass targets. It never appeared.

WHAT ACTUALLY FIXED IT:  
Click "Domain join instead." Despite the name, it does NOT join a domain at this stage. It  
drops you into the standard local account creation screen ("Who's going to use this device?").  
Create the local user there, e.g. labuser, plus three security questions.

WHY THE DIFFERENCE:  
OOBE\\BYPASSNRO and ms-cxh:localonly are workarounds for Windows 11 HOME and PRO, which hide the  
local account option and force a network connection plus Microsoft account. ENTERPRISE exposes  
the local path directly under Sign-in options, because enterprise deployments are expected to  
join a domain rather than use consumer accounts.

TAKEAWAY: on Windows 11 Enterprise, check Sign-in options FIRST. The command-line bypass is  
unnecessary. Josh's video predates all of this, since Windows 10 OOBE offered a local account  
without any workaround.

TIME LOST: \~\_\_ min  
SCREENSHOT: \_\_

📋 BUILD-LOG ENTRY

\#\#\# \[STEP 11\] — Local account security questions: use throwaway answers, not real ones

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
1\. Security question answers ARE credentials. The same questions (first pet, childhood city,  
   first school) are used for password recovery on real bank and email accounts. Reusing real  
   answers on a throwaway lab VM spreads them somewhere they do not belong.  
2\. This lab is being documented publicly. Screenshots taken during OOBE could capture those  
   answers on screen.  
3\. This local account gets used for roughly fifteen minutes. Every login after the domain join  
   uses a domain account, so the local account and its recovery questions become irrelevant.

RULE OF THUMB: in a lab, use obviously fake and consistent values. Personalizing throwaway  
infrastructure adds no value and creates real exposure. Focus on how the system works, not on  
making the VM feel like a personal machine.

RELATED: the same principle applies to the lab password throughout this build. It appears in  
plaintext in configuration steps and must never be a password used anywhere real.

TIME LOST: n/a  
SCREENSHOT: n/a

📋 BUILD-LOG ENTRY

\#\#\# \[STEP 11\] — OOBE offers an update download near the end of Windows 11 setup

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
  \- DC-1 running  
  \- DHCP issued CLIENT-1 an address  
  \- DNS resolved Microsoft's update servers  
  \- RRAS/NAT routed the traffic out through DC-1's Internet adapter  
This is effectively the step 12 verification happening early, without running a command.  
Worth screenshotting as evidence of the routing chain end to end.

RUNBOOK ACTION: add a note at step 11 covering this screen, the second warning, and the  
recommendation to skip. Also note the positive-signal interpretation so readers understand  
what the download is demonstrating instead of just waiting on it.

TIME LOST: \~ 1\_ min  
SCREENSHOT: \_n/a\_

📋 BUILD-LOG ENTRY

\#\#\# \[STEP 11\] — "Install Guest Additions" is not self-explanatory the second time

WHAT I WAS DOING:  
Reached the CLIENT-1 desktop after OOBE. Runbook said to install Guest Additions, same as was  
done on DC-1 earlier in the build.

WHAT HAPPENED:  
The instruction "install Guest Additions" was written as a one-liner on the assumption it had  
been done once already on DC-1 and would be remembered. It was not. Enough time and steps had  
passed that the actual click path was gone, and the step stalled.

WHAT ACTUALLY FIXED IT:  
Full click path, written out:  
  1\. VM menu bar: Devices \> Insert Guest Additions CD image  
  2\. Inside Windows: File Explorer \> This PC \> CD Drive (D:) VirtualBox Guest Additions  
  3\. Run VBoxWindowsAdditions-amd64.exe  
  4\. Next through the prompts, Install  
  5\. If Windows Security prompts about an Oracle device driver, choose Install  
  6\. Reboot when offered

Note: the installer defaults to C:\\Program Files\\Oracle\\VirtualBox Guest Additions. That C:\\  
is the GUEST's virtual drive, not the host's. Same point as the earlier DC-1 entry.

WHAT IT DOES: fixes mouse capture (no more Right Ctrl to release), enables dynamic screen  
resolution when resizing the VM window, and enables shared clipboard. Quality of life, not  
functionality. The lab works without it, but screenshots and general use are much worse.

RUNBOOK ACTION: do not abbreviate repeated steps. A reader following this doc for the first  
time has never done ANY of these steps, so "same as before" carries no meaning for them. Every  
repeated procedure gets the full click path each time it appears, or an explicit link back to  
the section where it is written out in full.

GENERAL PRINCIPLE FOR THE FINAL DOC: instructions that assume prior familiarity fail for the  
exact audience this is written for. Someone finding this via a portfolio is doing it cold, on  
their first pass, possibly without the video. Anywhere I had to ask a question during this  
build is a place the doc was insufficient, and it needs to be resolved in the final version.

TIME LOST: \~\_1\_ min  
SCREENSHOT: \_n/a\_

---

BUILD-LOG ENTRY

\[STEP 2a\] CLIENT-1 froze mid-lab with two guests and video playback running  
WHAT I WAS DOING: Working through the hosts file exercise on CLIENT-1 while watching the  
CourseCareers 4.6.2 video on the host machine, with both DC-1 and CLIENT-1 running.  
WHAT HAPPENED: CLIENT-1 stopped responding entirely. No keyboard or mouse input registered in  
the guest window. DC-1 was unaffected.  
WHAT I TRIED: Waited to see if it was thrashing rather than dead. Checked the VirtualBox status  
bar for disk and CPU activity. Sent a real Ctrl-Alt-Del through Input \> Keyboard \> Insert  
Ctrl-Alt-Del rather than the host keyboard shortcut, which does not reach the guest.  
WHAT ACTUALLY FIXED IT: Machine \> Reset from the VirtualBox menu. ACPI Shutdown did not get a  
response. No data was lost because the only on-disk change at that point was the hosts file  
edit, which had already been saved.  
ROOT CAUSE: The host is a 4-core, 8-thread i5-10300H. Two running guests plus browser video  
playback oversubscribed the CPU. Pausing video playback while working inside the guests  
prevented any recurrence. This matters more for the osTicket lab, which asks for a VM with 4  
vCPUs on its own.  
TIME LOST: 2 min   
SCREENSHOT: N/A

BUILD-LOG ENTRY

\[STEP 5a\] Multihomed DC registered its unreachable NAT adapter in DNS  
WHAT I WAS DOING: Reviewing the forward lookup zone in DNS Manager after creating the mainframe  
A record.  
WHAT HAPPENED: Noticed dc-1 had two Host (A) records, one for 172.16.0.1 (the intnet adapter  
clients use) and one for 10.0.2.15 (the VirtualBox NAT adapter). The (same as parent folder)  
entries were duplicated the same way. Nothing was broken yet, and that is the problem: DNS would  
have returned both addresses in round-robin order, so roughly half of all lookups for dc-1 would  
hand a client an address on a network it cannot reach. The failure would have surfaced as  
intermittent hangs and timeouts on \\\\dc-1 during the file shares lab, which is far harder to  
diagnose than a consistent failure.  
WHAT I TRIED: Confirmed via ipconfig on DC-1 that the two addresses belonged to the Internet  
(NAT) and Internal (intnet) adapters, and confirmed CLIENT-1 has no route to 10.0.2.15.  
WHAT ACTUALLY FIXED IT: On DC-1, opened ncpa.cpl \> Internet adapter \> Properties \> IPv4 \>  
Advanced \> DNS tab and unchecked "Register this connection's addresses in DNS." Then deleted  
both 10.0.2.15 Host (A) records in DNS Manager and refreshed. Only 172.16.0.1 entries remain.  
This is the correct production configuration for a multihomed domain controller, not a lab  
workaround. A DC should only register the interface its clients actually use.  
TIME LOST: 3 min  
SCREENSHOT: N/A

BUILD-LOG ENTRY

\[STEP 6\] Stale-cache demonstration failed because the cached entry had already expired  
WHAT I WAS DOING: Changing the mainframe A record from 172.16.0.1 to 8.8.8.8 on DC-1, then  
pinging mainframe from CLIENT-1 to observe the client returning the old, cached address.  
WHAT HAPPENED: CLIENT-1 immediately returned 8.8.8.8 instead of the expected stale 172.16.0.1.  
The demonstration produced the opposite of the intended result.  
WHAT I TRIED: First theory was that clicking Apply rather than OK in DNS Manager had changed the  
behavior. That was wrong. Apply and OK perform the same server-side write, and neither has any  
effect on the DNS client cache, which lives in memory on CLIENT-1.  
WHAT ACTUALLY FIXED IT: Nothing was broken. The Step 5 lookup that populated the cache happened  
roughly 21 hours earlier, and CLIENT-1 had also been hard-reset after an unrelated freeze. The  
default TTL on records in an AD-integrated zone is one hour, and the DNS client cache is  
memory-resident and does not survive a reboot. By the time the record was changed, the client had  
nothing cached, so it queried DC-1 and correctly received the current answer.  
Redoing the sequence in one sitting (flush, ping to populate, change the record, ping again)  
produced the intended stale result.  
LESSON: The runbook now notes that Steps 5 through 7 must be performed consecutively within the  
TTL window with no reboot in between. Cache behavior is time-dependent, and a lab written without  
that constraint stated will silently fail for anyone who takes a break in the middle.  
TIME LOST: 5-7 minutes  
SCREENSHOT: N/A

BUILD-LOG ENTRY

\[STEP 7\] Step 7 evidence required a server-side capture that the runbook did not specify  
WHAT I WAS DOING: Capturing evidence of the client answering from cache instead of from the  
authoritative DNS server.  
WHAT HAPPENED: The client-side capture showed ping mainframe returning an address, followed by  
ipconfig /flushdns, followed by ping mainframe returning a different address. Read in isolation  
the screenshot is ambiguous, because it cannot show what the mainframe record held on DC-1 at  
the moment of each ping. Without that, a reader cannot tell whether the first answer was stale or  
simply correct at the time.  
WHAT I TRIED: Reviewed the capture against the runbook, which specified only a single client-side  
screenshot for this step.  
WHAT ACTUALLY FIXED IT: Added a second capture, ext-dns-07b-record-changed.png, showing the  
mainframe record in DNS Manager. The two together establish the mismatch: server holding one  
address, client returning another, flush reconciling them.  
LESSON: Any lab step whose lesson is a disagreement between two machines needs evidence from both  
machines. A single-sided capture documents an observation but cannot prove the claim built on it.  
The runbook has been updated to require both captures at this step.  
TIME LOST:  
SCREENSHOT:

BUILD-LOG ENTRY

\[STEP 8\] Restarting CLIENT-1 wiped the DNS cache and emptied the evidence for Step 8  
WHAT I WAS DOING: Running ipconfig /displaydns on CLIENT-1 to capture the stale mainframe entry  
sitting in cache while DC-1 held a different address.  
WHAT HAPPENED: The mainframe entry was absent from the output entirely. The zebra entry from the  
hosts file was present, which made it look like the command had worked but the DNS portion had  
failed.  
WHAT I TRIED: Searched the generated text file for mainframe directly. Confirmed the file had  
been created and was populated with several hundred other cached records, so the command itself  
had run correctly.  
WHAT ACTUALLY FIXED IT: Nothing was broken. CLIENT-1 had been restarted immediately before this  
step because of a recurring freeze. The Windows DNS client cache is memory-resident, so a  
restart clears it completely regardless of remaining TTL. The zebra entry survived because it  
comes from the hosts file on disk, which is read fresh on every lookup and is not part of the  
cache. The TTL values confirmed it: zebra showed 604800 seconds, a hosts file value, while genuine  
cached DNS answers carry the record's actual TTL, observed at 4268 seconds for mainframe once  
repopulated.  
Repopulating the cache (set the record to 172.16.0.1, ping to cache it, change the record to  
8.8.8.8, then displaydns) produced the intended result on the first attempt.  
LESSON: Any lab step that depends on cache state is invalidated by a reboot, not just by elapsed  
time. The runbook's warning about consecutive execution was written around TTL expiry only and  
did not account for a restart mid-sequence.  
TIME LOST: 7 minutes  
SCREENSHOT: N/A

BUILD-LOG ENTRY

\[STEP 5a\] Disabling adapter DNS registration did not stop a DC from advertising its NAT address  
WHAT I WAS DOING: Re-checking the Step 5a multihomed cleanup after noticing that 10.0.2.15  
records had reappeared in the smithlab.local zone following work on the CNAME steps.  
WHAT HAPPENED: "Register this connection's addresses in DNS" was already unchecked on the  
Internet (NAT) adapter, yet the zone again contained a dc-1 Host (A) record at 10.0.2.15 marked  
static, plus a (same as parent folder) Host (A) at 10.0.2.15 timestamped the same day.  
WHAT I TRIED: Verified the checkbox state on the Internet adapter's IPv4 Advanced DNS tab.  
Compared the returned records against the earlier cleanup capture to confirm both had previously  
been removed.  
WHAT ACTUALLY FIXED IT: The adapter checkbox governs the DNS client service only. On a domain  
controller the DNS server service separately registers zone records for every interface it is  
configured to listen on, and that setting is not affected by the client-side checkbox. Restricting  
the DNS server to listen only on 172.16.0.1 (DNS Manager, DC-1 Properties, Interfaces tab, "Only  
the following IP addresses") stopped the dynamic re-registration. The static dc-1 record required  
manual deletion, since static records are created at role install time, never age out, and are  
unaffected by any registration setting. A snapshot taken before the original cleanup had also  
restored the deleted records on rollback.  
LESSON: Suppressing DNS registration on a multihomed domain controller requires three separate  
actions, not one: uncheck adapter registration (DNS client), restrict listening interfaces (DNS  
server), and manually delete any static records. A snapshot must be taken after the cleanup, or a  
rollback silently reintroduces the problem.  
TIME LOST: 4 minutes  
SCREENSHOT: N/A

BUILD-LOG ENTRY

\[HOST\] VirtualBox running under Hyper-V with nested paging disabled, causing repeated guest freezes  
WHAT I WAS DOING: Investigating why CLIENT-1 froze twice during the DNS extension lab while DC-1  
was unaffected.  
WHAT HAPPENED: The VirtualBox status bar showed a green turtle icon. The execution tooltip  
reported Execution engine: native API, Nested Paging: Inactive, Unrestricted Execution: Inactive,  
Paravirtualization Interface: Hyper-V. VirtualBox was not running on the CPU directly. It was  
running through the Windows Hypervisor Platform API as a nested guest, with hardware memory  
translation unavailable, so every guest memory access went through software-emulated page table  
walks. Only CLIENT-1 froze because it runs a full desktop with a browser and heavy memory churn,  
while DC-1 sits in DNS Manager with a stable working set.  
WHAT I TRIED: Disabled Memory Integrity in Windows Security Core isolation and rebooted. No  
change. Unchecked Virtual Machine Platform, Windows Hypervisor Platform, and Windows Subsystem  
for Linux in Windows Features and rebooted. No change. Hyper-V itself is not listed on Windows 11  
Home. Ran bcdedit /set hypervisorlaunchtype off and rebooted. No change.  
Diagnosed properly with bcdedit /enum {current}, systeminfo, and msinfo32. The boot setting had  
stored correctly as Off and Memory Integrity was genuinely disabled, but msinfo32 reported  
Virtualization-based security: Running. VBS launches the hypervisor independently of  
hypervisorlaunchtype, which is why the boot setting had no effect.  
WHAT ACTUALLY FIXED IT: Three registry values under HKLM\\SYSTEM\\CurrentControlSet\\Control,  
setting DeviceGuard\\EnableVirtualizationBasedSecurity to 0, DeviceGuard\\Scenarios\\  
HypervisorEnforcedCodeIntegrity\\Enabled to 0, and Lsa\\LsaCfgFlags to 0, followed by a reboot.  
Turtle gone. Execution engine now VT-x/AMD-V with Nested Paging and Unrestricted Execution both  
Active.  
LESSON: On Windows 11 Home, hypervisorlaunchtype off is not sufficient to release the hardware  
virtualization extensions. Virtualization-based security is a separate mechanism that overrides  
it, and msinfo32 is the tool that reveals which layer is actually holding the hypervisor. Fix  
ladders are worth working top to bottom, but diagnosing before climbing further would have saved  
two reboots here.  
TRADEOFF: This disables Memory Integrity, LSA protection, and VBS on the host. Acceptable for a  
machine used primarily for lab work, and worth revisiting if this machine's role changes. Smart  
App Control was deliberately left enabled, since disabling it is irreversible without reinstalling  
Windows.  
TIME LOST:  
SCREENSHOT:

BUILD-LOG ENTRY

\[STEP: Part 2, Test 3\] A normal user could open the no-access share despite having no share permission  
WHAT I WAS DOING: Testing the no-access folder from CLIENT-1 as bhastings, a standard domain user  
with no elevated rights. The folder had been shared to Domain Admins only, so the expected result  
was an access denied error.  
WHAT HAPPENED: The folder opened normally. Explorer showed the folder as empty rather than  
refusing access.  
WHAT I TRIED: Confirmed the account was not a Domain Admin via Settings \> Accounts. Confirmed the  
share permission had been set to Domain Admins. Opened the folder's Security tab on DC-1 and found  
inherited NTFS entries for Users and Authenticated Users that had never been added by hand.  
WHAT ACTUALLY FIXED IT: Access to a Windows share is gated by two independent permission sets and  
a user receives the more restrictive of the two. Only the share permission had been configured.  
The NTFS permissions on C:\\no-access were inherited from the root of C:\\, which grants read access  
to Users and Authenticated Users by default, and every domain user is a member of both. The simple  
Share button compounds this: it does not restrict the share to the named group, it opens the share  
and relies on NTFS for actual access control. With NTFS never configured, nothing was stopping the  
user.  
Fixed by opening Security \> Advanced on the folder, disabling inheritance and converting inherited  
permissions to explicit, then removing Users and Authenticated Users and leaving Domain Admins,  
SYSTEM, and Administrators. Access from CLIENT-1 then failed as intended.  
LESSON: Setting a share permission is not the same as securing a folder, and a lab that only ever  
sets share permissions will appear to work while teaching the wrong model. Any folder created  
under C:\\ inherits permissive read access from the root, so restricting access requires breaking  
inheritance rather than adding a permission. This is the share versus NTFS distinction that  
CompTIA tests as a scenario question, encountered here as an actual failure rather than a  
definition.  
TIME LOST: 2 minutes  
SCREENSHOT: N/A

BUILD-LOG ENTRY

\[STEP: Part 3\] Removing a user from a security group did not revoke access in the live session  
WHAT I WAS DOING: Attempting to capture the pre-membership state for Part 3, where a user can see  
the accounting share on the network but cannot open it. I had already added bhastings to the  
ACCOUNTANTS group and captured the working state, so I removed him from the group to reproduce the  
denied state.  
WHAT HAPPENED: After removing bhastings from ACCOUNTANTS on DC-1, closing the accounting folder on  
CLIENT-1, and reopening it, he still had full access to the share.  
WHAT I TRIED: Confirmed on DC-1 that the Members tab no longer listed bhastings. Closed and  
reopened both the folder and Explorer entirely.  
WHAT ACTUALLY FIXED IT: Signing out of CLIENT-1 completely and signing back in as the same user.  
The Kerberos access token is built at logon and enumerates group membership at that moment, and it  
is not re-evaluated for the life of the session. Windows also caches the authenticated SMB session  
to the server, so a new Explorer window reuses the existing connection rather than reauthenticating.  
Neither closing the folder nor reopening Explorer triggers a new token.  
LESSON: Group membership changes require a re-login to take effect, and this applies to removal  
exactly as much as it applies to addition. The addition case is widely known. The removal case is  
the one with real consequences: revoking a group membership does not revoke access until that user  
signs out, so an offboarding or access-reduction task is not actually complete while the person is  
still logged in. Disabling the account or forcing a session termination is what makes it immediate.  
TIME LOST: 8 minutes   
SCREENSHOT: N/A

BUILD-LOG ENTRY

\[STEP: Part 3\] Removing a user from the permissioned group did not deny access, because the share granted Everyone  
WHAT I WAS DOING: Trying to reproduce the denied state for the accounting share so I could capture  
a user seeing the folder on the network but being unable to open it. bhastings had been removed  
from the ACCOUNTANTS group.  
WHAT HAPPENED: bhastings retained full access to \\\\dc-1\\accounting after being removed from the  
group, after a full sign-out, and after a host restart.  
WHAT I TRIED: Confirmed the ACCOUNTANTS Members tab was empty on DC-1. Confirmed the folder's NTFS  
ACL granted ACCOUNTANTS and did not name bhastings. Ran net use \* /delete /y, which reported no  
entries, because it had been run from an elevated session under a different account and only  
affects the session that runs it. Checked the Sharing tab, which is the permission set I had not  
inspected.  
WHAT ACTUALLY FIXED IT: The share permissions granted Everyone Full Control. The Windows Share  
Wizard writes Everyone into the share ACL by design and relies on NTFS for actual access control,  
so group membership was never the gate. Removed Everyone from the share permissions and added  
ACCOUNTANTS with Change and Read. Access then denied correctly after a re-login.  
LESSON: Two independent permission sets guard every share, and checking one of them proves nothing.  
The NTFS ACL looked correct in isolation and was correct, which made it easy to stop looking.  
Verifying a share means opening both the Security tab and Advanced Sharing \> Permissions, every  
time. Also worth noting that read-access and write-access behaved correctly in Part 2 while  
carrying the same wide-open share ACL, meaning those tests produced the right answers for the wrong  
reason.  
TIME LOST: 12 minutes  
SCREENSHOT: N/A

BUILD-LOG ENTRY

\#\#\# \[STEP: Before you start\] Runbook specified Terminal on a Server 2022 machine, and Terminal opened Command Prompt  
\*\*WHAT I WAS DOING:\*\* Running the pre-flight Get-ADUser check on DC-1 to confirm bhastings was  
enabled and unlocked before starting the lab.  
\*\*WHAT HAPPENED:\*\* Get-ADUser returned "is not recognized as an internal or external command,  
operable program or batch file."  
\*\*WHAT I TRIED:\*\* Re-typed the command assuming a typo. Confirmed the syntax matched the runbook.  
\*\*WHAT ACTUALLY FIXED IT:\*\* The command was running in Command Prompt, not PowerShell. Get-ADUser  
is a PowerShell cmdlet and does not exist in cmd. Two separate causes stacked: the runbook said  
"Terminal (Admin)," which is a Windows 11 menu entry, while DC-1 runs Server 2022 where that menu  
reads "Windows PowerShell (Admin)"; and Windows Terminal is a host application rather than a shell,  
so even where it exists it can open a Command Prompt tab depending on its default profile. Opening  
Windows PowerShell (Admin) directly resolved it.  
\*\*LESSON:\*\* The prompt tells you which shell you are in before you run anything. "PS C:\\Users\\...\>"  
is PowerShell and cmdlets work. "C:\\Users\\...\>" is Command Prompt and they do not. Related trap  
found in the same pass: auditpol commands copied from Command Prompt documentation fail in  
PowerShell because PowerShell strips the quotes off arguments like "Kerberos Authentication  
Service" before the legacy command receives them. Either use a form with no quoting, or use the  
\--% stop-parsing token.  
\*\*TIME LOST:\*\*  1 min  
\*\*SCREENSHOT:\*\* N/A

BUILD-LOG ENTRY

\#\#\# \[STEP: Part 2\] Domain controller audited the administrative response but not the attack that caused it  
\*\*WHAT I WAS DOING:\*\* Running the Part 2 pre-flight audit check on DC-1 before generating any lockout  
evidence, to confirm the subcategories that record failed authentication were actually enabled.  
\*\*WHAT HAPPENED:\*\* Kerberos Authentication Service read "Success" only. Credential Validation read  
"Success" only. Account Lockout read "Success" only. Every subcategory that records an administrator  
action was already enabled out of the box, and every subcategory that records a failed authentication  
was not.  
\*\*WHAT I TRIED:\*\* Nothing to troubleshoot. The check was written specifically to catch this state  
before it caused a silent failure downstream.  
\*\*WHAT ACTUALLY FIXED IT:\*\* Enabled failure auditing on all three with auditpol, using the \--%  
stop-parsing token so PowerShell passed the quoted subcategory names through intact. Re-ran  
auditpol /get /category:\* and confirmed all three read "Success and Failure."  
\*\*LESSON:\*\* A Server 2022 domain controller ships auditing the response and not the incident. User  
Account Management logs the administrator unlocking an account. Kerberos Authentication Service  
failure auditing, which logs the bad password attempts that locked it, is off. Had this lab run in  
the source material's order, every 4771 generated during the lockout would have been discarded at  
the moment it occurred and Part 7 would have found nothing, with no way to recover the events after  
the fact. Auditing is not retroactive. Verifying it before generating evidence is the only order  
that works. Note also that "Account Lockout" set to Success is close to meaningless, since the  
events that subcategory generates are failures by definition.  
\*\*TIME LOST:\*\*  3 min  
\*\*SCREENSHOT:\*\* ext-lockout-07-auditpol-dc1.png

BUILD-LOG ENTRY

\#\#\# \[STEP: Part 2\] Domain controller audited the administrative response but not the attack that caused it  
\*\*WHAT I WAS DOING:\*\* Running the Part 2 pre-flight audit check on DC-1 before generating any lockout  
evidence, to confirm the subcategories that record failed authentication were actually enabled.  
\*\*WHAT HAPPENED:\*\* Kerberos Authentication Service read "Success" only. Credential Validation read  
"Success" only. Account Lockout read "Success" only. Every subcategory that records an administrator  
action was already enabled out of the box, and every subcategory that records a failed authentication  
was not.  
\*\*WHAT I TRIED:\*\* Nothing to troubleshoot. The check was written specifically to catch this state  
before it caused a silent failure downstream.  
\*\*WHAT ACTUALLY FIXED IT:\*\* Enabled failure auditing on all three with auditpol, using the \--%  
stop-parsing token so PowerShell passed the quoted subcategory names through intact. Re-ran  
auditpol /get /category:\* and confirmed all three read "Success and Failure."  
\*\*LESSON:\*\* A Server 2022 domain controller ships auditing the response and not the incident. User  
Account Management logs the administrator unlocking an account. Kerberos Authentication Service  
failure auditing, which logs the bad password attempts that locked it, is off. Had this lab run in  
the source material's order, every 4771 generated during the lockout would have been discarded at  
the moment it occurred and Part 7 would have found nothing, with no way to recover the events after  
the fact. Auditing is not retroactive. Verifying it before generating evidence is the only order  
that works. Note also that "Account Lockout" set to Success is close to meaningless, since the  
events that subcategory generates are failures by definition.  
\*\*TIME LOST:\*\*  
\*\*SCREENSHOT:\*\* ext-lockout-07-auditpol-dc1.png. The full audit policy is roughly 60 rows and the  
font had to be reduced to fit it in one frame, so Credential Validation sits one line below the  
bottom edge. It was set to Success and Failure in the same pass as the other two and its post-change  
state is recorded in the terminal output above. Highlighted in the capture: Logon, Account Lockout,  
User Account Management, and Kerberos Authentication Service.

BUILD-LOG ENTRY

\#\#\# \[STEP: Part 5\] Bulk-provisioned accounts carry Password never expires, which blocks the forced password change  
\*\*WHAT I WAS DOING:\*\* Working through the account lockout lab on bhastings, an account created by the  
bulk PowerShell provisioning script during the base domain build.  
\*\*WHAT HAPPENED:\*\* The Account tab showed Password never expires already checked. That setting is  
mutually exclusive with User must change password at next logon, which Part 5 of the runbook requires  
ticking during the password reset. The checkbox is greyed out and cannot be selected while Password  
never expires is set.  
\*\*WHAT I TRIED:\*\* Noticed it while capturing the unlock checkbox rather than after hitting the wall  
in Part 5\.  
\*\*WHAT ACTUALLY FIXED IT:\*\* Unchecked Password never expires on the Account tab, clicked Apply. User  
must change password at next logon then became selectable.  
\*\*LESSON:\*\* Every one of the 1000 accounts from the provisioning script carries PasswordNeverExpires  
set to true, so this applies to any of them, not just this one. It is a lab convenience that quietly  
disables a control the lab is trying to demonstrate. In production it is worse than an inconvenience:  
an account whose password never expires and never has to be changed is a credential with no rotation,  
which is the exact pattern that turns one leaked password into permanent access. Belongs in the  
README's lab-only shortcuts table alongside the shared hardcoded password.  
\*\*TIME LOST:\*\*  
\*\*SCREENSHOT:\*\*

BUILD-LOG ENTRY

\#\#\# \[STEP: Part 7\] Runbook asked for a client-side 4625 marked locked out or disabled, which cannot exist on a Kerberos domain client  
\*\*WHAT I WAS DOING:\*\* Taking the final capture of the lab, a 4625 on CLIENT-1 showing a meaningful  
failure sub-status.  
\*\*WHAT HAPPENED:\*\* No event in the client Security log carried 0xC0000234 (locked out) or 0xC0000072  
(disabled). Every 4625 was either 0xC000006A, a bad password, or 0xC0000064, an unknown username.  
\*\*WHAT I TRIED:\*\* Scanned the full filtered 4625 list looking for the two codes the runbook named.  
\*\*WHAT ACTUALLY FIXED IT:\*\* Nothing to fix. The runbook was wrong. On a domain-joined Windows client  
authenticating over Kerberos, the client requests a ticket from the domain controller before any  
local logon session is attempted. A locked or disabled account is refused at that point, so the  
workstation never reaches a stage where it would log an interesting failure. Captured 0xC000006A  
with Logon Type 2 instead.  
\*\*LESSON:\*\* Those two sub-status codes are real, but they surface where authentication runs over NTLM  
or against a local SAM account, not on a Kerberos domain client. The evidence that the account was  
locked is event 4740 on DC-1, which is the same architectural point the whole lab demonstrates:  
the client does not know and does not decide. Also worth noting the sub-status pair that does appear  
here: 0xC000006A means the username was valid and the password wrong, 0xC0000064 means the username  
does not exist. On a real network that difference separates password guessing against a known  
account from guessing at account names.  
\*\*TIME LOST:\*\*12 minutes  
\*\*SCREENSHOT:\*\* ext-lockout-18-client1-event-4625.png  
