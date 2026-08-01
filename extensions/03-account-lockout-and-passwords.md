# Extension Lab 3: Account Lockout, Unlocking, and Password Resets

What happens when a user types their password wrong five times, what an administrator does about
it, and the difference between an account that is locked and an account that is disabled. Every
state change is made on `DC-1`, observed on `CLIENT-1`, and then found again in the logs.

This lab extends the domain built in the main runbook. It requires `DC-1` and `CLIENT-1` to already
exist and be joined. Nothing new is installed.

**Source material:** this lab follows the structure of Josh Madakor's "Group Policy and Managing
Accounts" exercise, taught in the CourseCareers IT program (module 4.5.5), along with its companion
lab checklist "Enabling and Unlocking Accounts and Resetting Passwords" and the linked doc "How To
Configure Account Lockout Threshold in Group Policy." That version is built on Azure virtual
machines. What follows is the on-premises translation, rebuilt and verified on VirtualBox.

> **On the timestamps below.** Each part carries a reference like `Video: 07:17`, pointing at the
> matching moment in the CourseCareers module. **That course is paid, and you do not need it to
> follow this runbook.** Every step here is written to stand on its own. The timestamps are for
> people already enrolled who want to see a step demonstrated. If you are not enrolled, ignore
> them and nothing is lost.

**What is mine versus what is his.** The exercise structure and teaching sequence are his. The
on-premises translation, the before-and-after policy evidence in Part 1, the audit verification in
Part 2, the entire 4740 and 4771 analysis in Part 7, the sub-status code table, the logon type
translation for console versus RDP, and every capture in this document are mine.

---

## A note on ordering

Module 4.5.5 comes **before** the DNS module (4.6.2) and the file shares module (4.7.2) in the
CourseCareers sequence. This repository built them in the opposite order, so this is Extension Lab
3 here and the earlier module there. Nothing depends on that. The three labs are independent of
each other and all three depend only on the base domain build.

---

## Where this runbook diverges from the video

The video is a live recording in which the demonstration fails first and gets fixed on camera. That
is good teaching and bad instructions. Five places where this runbook does something different, and
why:

**1. Order.** The video attempts the lockout first, fails because no lockout policy exists by
default, and then configures the policy. This runbook configures the policy first and captures
proof of the before state, which produces the same lesson with better evidence and half the time.

**2. New GPO instead of editing Default Domain Policy.** The video edits Default Domain Policy
because it is already linked at the domain root. That works. This runbook creates a separate GPO
named `Account Lockout Policy` and links it at the root instead, for three reasons: it is reversible
by unlinking rather than by remembering what you changed, it puts link order and precedence on
screen where a reader can see them, and it produces a capture the video's approach cannot.

**3. `gpupdate` on `DC-1`, not on `CLIENT-1`.** See Part 1, Step 8. The video runs it on the client.

**4. Audit verification before generating evidence.** The video has no equivalent to Part 2, and
that omission is why the video's log review comes up empty on the domain controller.

**5. Part 7 is almost entirely new.** The video searches the Event Viewer by username, finds only
logon and logoff events on `DC-1`, states on camera *"I don't actually know where they're going to
show up"* and *"I don't see any like the failed login,"* then moves to `CLIENT-1` and finds event
4625. It never locates the lockout event, never mentions 4740 or 4771, and never filters by event
ID. Part 7 finds all of it and explains why the domain controller looked empty. **This is the most
valuable original content in the lab. Capture it properly.**

---

## Which machine, every time

Every step states its machine in bold before the actions. `DC-1` is the domain controller and is
where every policy change and every administrative action happens. `CLIENT-1` is the workstation
and is where every authentication attempt happens.

The rule of thumb for this lab: **`DC-1` decides, `CLIENT-1` finds out.**

---

## Which shell, and how to tell

**Every command in this runbook is run in PowerShell, elevated.** Some of them are legacy commands
that also work in Command Prompt. Several are not, and will fail outright.

**Windows Terminal is not a shell.** It is a window that hosts one. Which shell you get depends on
its default profile, and if it opens a Command Prompt tab then `Get-ADUser`, `Search-ADAccount`, and
`Unlock-ADAccount` do not exist and you get `is not recognized as an internal or external command`.
That error means you are in the wrong shell, not that anything is broken.

**Read the prompt to know where you are:**

```
PS C:\Users\Keenan_Admin>      PowerShell. Cmdlets work.
C:\Users\Keenan_Admin>         Command Prompt. They do not.
```

**How to open it, which differs by machine:**

| Machine | OS | Right-click **Start** and choose |
| --- | --- | --- |
| `DC-1` | Windows Server 2022 | **Windows PowerShell (Admin)** |
| `CLIENT-1` | Windows 11 Enterprise | **Terminal (Admin)**, then confirm the tab says `PS` |

Server 2022 does not ship Windows Terminal, so that menu offers PowerShell directly. Windows 11
offers Terminal, whose default profile is normally PowerShell but can be changed. If a Terminal tab
opens as Command Prompt, click the **down-arrow** beside the tab and pick **Windows PowerShell**.

If you would rather not think about it, typing `powershell` and pressing **Enter** inside a Command
Prompt switches that window to PowerShell in place.

---

## Environment translation

| CourseCareers Azure version | This build |
| --- | --- |
| Azure Portal, start VMs | VirtualBox Manager, start `DC-1` then `CLIENT-1` |
| Remote Desktop to a public IP | VirtualBox console window for each VM |
| `mydomain.com\jane_admin` | `SMITHLAB\Keenan_Admin` |
| A random user from the `employees` OU, password `Password1` | The same `_USERS` account used in Lab 2 |
| Azure VNet | VirtualBox internal network, `intnet` |
| Client is Windows 10 | Client is Windows 11 Enterprise |
| Failed logons arrive over RDP, **Logon Type 10** | Failed logons arrive at the console, **Logon Type 2** |
| Event 4625 shows the RDP client's source IP | Event 4625 shows a blank or loopback source address |
| "Don't delete your resources in Azure" | Shut down from inside Windows, no billing to worry about |

> **The logon type difference is the one that will confuse you if nobody warns you.** The video
> reads a source IP address out of event 4625 and treats it as the useful field, because Josh is
> connecting to the client over Remote Desktop from somewhere else. You are typing at the VirtualBox
> console window, which Windows records as a local interactive logon. Your 4625 events will show
> **Logon Type 2** and a **Source Network Address** of `-` or `127.0.0.1`. That is correct, not
> missing data. The equivalent useful field in your build is **Caller Computer Name** on event 4740,
> covered in Part 7.

---

## Before you start

**Start `DC-1` first** and let it reach the logon screen before starting `CLIENT-1`.

**Log into `DC-1` as `SMITHLAB\Keenan_Admin`.** Every action in Parts 1, 2, 4, 5, 6, and the `DC-1`
half of Part 7 runs as this account.

**The test account is `bhastings`.** The video says to pick any random user and that it does not
matter which. It does here. `bhastings` is the `_USERS` account that carried Extension Lab 2, and
reusing him keeps one continuous story across three labs instead of three disconnected
demonstrations. He is the same user who could read the share he was supposed to be locked out of.
Now he is the user who cannot log in at all.

Confirm its starting state. **On `DC-1`**, right-click **Start** > **Windows PowerShell (Admin)**:

```powershell
Get-ADUser bhastings -Properties Enabled,LockedOut,BadLogonCount,PasswordLastSet |
  Format-List Name,SamAccountName,Enabled,LockedOut,BadLogonCount,PasswordLastSet
```

Expected before you begin: `Enabled : True`, `LockedOut : False`, `BadLogonCount : 0`.

**Clear any saved credentials for `bhastings` on `CLIENT-1`.** Lab 2 reached the shares by typing
UNC paths rather than by mapping drives, so there is probably nothing to clean up. Check anyway,
because a saved credential pointed at `dc-1` will retry in the background against the same bad
password counter you are about to test, and the lockout will fire earlier than you expect.

**This has to be done while logged in as `bhastings`, not as `Keenan_Admin`.** Saved credentials and
drive mappings are per-user and per-session. Lab 2's build log already records this trap: `net use *
/delete /y` reported no entries because it was run from an elevated session under a different
account. Signing in as an admin to check `bhastings`' credentials shows you a clean slate that is
not his.

**On `CLIENT-1`,** signed in as `SMITHLAB\bhastings`:

1. Open **File Explorer** > **This PC**. If any mapped network drive is listed, right-click it and
   click **Disconnect**.
2. Click **Start**, type `Credential Manager`, press **Enter**.
3. Click **Windows Credentials**.
4. Look for any entry naming `dc-1` or `smithlab`. If one exists, click it, then click **Remove**.
5. Open a **non-elevated** PowerShell. Right-click **Start** > **Terminal**, not Terminal (Admin).
   Elevation launches a different logon session with its own separate connection table, which is
   exactly the mistake documented in Lab 2. Run:

```
net use
```

Expected on a clean client: `There are no entries in the list.` If entries appear, run `net use *
/delete /y` from that same non-elevated window.

6. Sign out.

**Snapshot both VMs before touching anything.** In VirtualBox Manager with **both VMs powered
off**: select `DC-1`, click the hamburger menu beside the machine name, click **Snapshots**, click
**Take**, name it `Pre Ext Lab 3`, click **OK**. Repeat for `CLIENT-1` with the same name.

> **Never roll back `DC-1` alone.** Already documented in the main README, and it matters more here
> because this lab deliberately changes a domain password. Restore `DC-1` past `CLIENT-1`'s last
> computer account password rotation and you get *"The trust relationship between this workstation
> and the primary domain failed."* Roll both back to the same point or roll neither back.
>
> Lab 1's finding applies too: a snapshot restore silently reverses directory changes with no
> warning. Roll back mid-lab and the GPO, the password, and the account state all quietly revert.

---

## Part 1: Configure the account lockout policy

> Video: `06:37` asks ChatGPT for the procedure, `07:17` opens GPMC, `08:32` navigates to Account
> Lockout Policy, `09:50` through `11:15` sets the three values, `11:34` reviews them on the
> Settings tab, `12:01` runs `gpupdate`.

### Step 1. Prove there is no lockout policy yet

The video spends its first six minutes failing eleven logons in a row and getting logged in anyway,
because **Windows Server does not configure account lockout by default.** That is a real and
important fact. This step captures it in ten seconds instead of six minutes.

**On `DC-1`,** right-click **Start** > **Windows PowerShell (Admin)**, then:

```
net accounts /domain
```

Read this line:

```
Lockout threshold:                                    Never
```

**CAPTURE: `ext-lockout-01-net-accounts-before.png`**
The full output showing the threshold at `Never`.

Keep this window open. You will run the same command again in Step 9 and the pair is the evidence.

> **Out of the box, a domain account can be guessed at forever.** No threshold, no lockout, no
> limit. Every bit of protection you are about to configure is opt-in, and a domain nobody has
> configured has none of it. That is the single most useful thing in this lab and it is worth
> stating plainly.

### Step 2. Open Group Policy Management on DC-1

**On `DC-1`:** click **Start**, type `gpmc.msc`, press **Enter**.

If it does not launch, the feature is missing. **Server Manager** > **Manage** > **Add Roles and
Features** > **Next** through to **Features** > tick **Group Policy Management** > **Next** >
**Install**. On a promoted domain controller it is normally already there.

### Step 3. Create the GPO at the domain root, not on an OU

**On `DC-1`,** in Group Policy Management:

1. In the left pane, expand **Forest: smithlab.local**.
2. Expand **Domains**.
3. Expand **smithlab.local**.
4. Right-click the **smithlab.local** node itself. Not an OU beneath it.
5. Click **Create a GPO in this domain, and Link it here...**
6. Name: `Account Lockout Policy`
7. Leave **Source Starter GPO** as `(none)`.
8. Click **OK**.

> **The written guidance is wrong here, and it fails silently. The video is not.** Josh's companion
> doc says to link the GPO to "an Organizational Unit (OU) or domain." Linking an account lockout
> GPO to an OU does not lock out domain accounts. On camera he edits Default Domain Policy instead,
> which is already linked at the domain root, so the video sidesteps the problem without ever
> naming it. Anyone following the written doc and picking the OU option will not be so lucky.
>
> Account Policies, meaning Password Policy, Account Lockout Policy, and Kerberos Policy, behave
> differently from every other Group Policy setting. For **domain user accounts** the policy is
> only honored when it comes from a GPO **linked at the root of the domain**, because the domain
> controllers read it and enforce it themselves at authentication time. It is a property of the
> domain, not of any container.
>
> Link the same GPO to an OU and it does not disappear. It applies to the **local SAM database** of
> any computer objects in that OU. Local accounts on those machines get a lockout threshold. Domain
> accounts get nothing. Nothing errors and nothing warns you. If you want to see it for yourself,
> do the OU version first, watch it not work, then move the link.

### Step 4. Confirm the link order

**On `DC-1`,** click the **smithlab.local** node in the left pane, then click the **Linked Group
Policy Objects** tab on the right.

`Account Lockout Policy` should sit at **Link Order 1** with `Default Domain Policy` beneath it. A
newly linked GPO takes link order 1, the highest precedence at that level, so your settings win
over anything conflicting in Default Domain Policy.

**CAPTURE: `ext-lockout-02-gpmc-gpo-linked-domain-root.png`**
The left pane showing the GPO under the domain
node, with the Linked Group Policy Objects tab and link order visible on the right. This single
capture is the proof the link is at the root and not on an OU, and it is the capture the video's
edit-the-default approach cannot produce.

### Step 5. Open the editor and navigate to Account Lockout Policy

**On `DC-1`,** in Group Policy Management:

1. In the left pane, expand **Group Policy Objects**.
2. Right-click **Account Lockout Policy** and click **Edit**.

The **Group Policy Management Editor** opens in a new window. In its left pane, expand in this
exact order:

3. **Computer Configuration**
4. **Policies**
5. **Windows Settings**
6. **Security Settings**
7. **Account Policies**
8. Click **Account Lockout Policy**

The right pane shows the settings, all reading **Not Defined**.

> **You may see three settings or four.** The classic three are Account lockout duration, Account
> lockout threshold, and Reset account lockout counter after. Recent builds add a fourth, **Allow
> Administrator account lockout**, which the video mentions in passing at `11:00`. Historically the
> built-in Administrator account was exempt from lockout entirely, which meant it was the one
> account an attacker could hammer indefinitely. That setting removes the exemption. Leave it
> undefined for this lab. If it is present and you want to set it, `Enabled` is the more defensible
> choice, with the obvious caveat that you can now lock yourself out of your own domain.

> This lives under **Computer Configuration**, not User Configuration, even though it governs user
> accounts. That is not a navigation mistake. Account policy is enforced by the machine doing the
> authenticating, which for domain accounts is always the domain controller.

**CAPTURE: `ext-lockout-03-lockout-policy-undefined.png`**
The right pane with everything Not Defined, left
pane fully expanded so the path is legible.

### Step 6. Set the threshold first

Order matters here. Setting the threshold before the other two triggers a dialog worth capturing.

**On `DC-1`,** in the Group Policy Management Editor:

1. Double-click **Account lockout threshold**.
2. Tick **Define this policy setting**.
3. Set the spinner to `5`.
4. Click **Apply**.

A **Suggested Value Changes** dialog appears. Windows recognizes that a threshold with no duration
and no observation window is incomplete, and proposes both.

**CAPTURE: `ext-lockout-04-suggested-value-changes.png`**
The dialog, before you dismiss it.

5. Click **OK** to accept.
6. Click **OK** to close the properties window.

> **A threshold of `0` means never lock out.** The spinner accepts it, and it is the disabled
> state, not "lock immediately." `1` is the strictest legal value and a bad idea in production,
> because one typo locks a user out. This is also what `Never` meant in the Step 1 output.

> **The video arrives at these values from the other direction and it looks confusing on camera.**
> Josh sets the duration first, watches Windows auto-populate the threshold at 5, unchecks it to
> see what happens, then sets the threshold and gets the auto-population again. He ends with the
> reset counter at 10 minutes rather than 30. Either value works and the lab does not change. Take
> the suggested values and move on.

### Step 7. Verify the three values

**On `DC-1`,** confirm the right pane reads:

| Setting | Value |
| --- | --- |
| Account lockout duration | 30 minutes |
| Account lockout threshold | 5 invalid logon attempts |
| Reset account lockout counter after | 30 minutes |

These three are constantly confused and at least one shows up regularly on A+ Core 2 Security
questions:

- **Threshold** is how many failures it takes.
- **Duration** is how long it stays locked before clearing itself. Set to `0`, it stays locked until
  an administrator intervenes.
- **Reset counter after** is the observation window, meaning how long a failure is remembered. With
  30 minutes, four bad passwords followed by 31 minutes of silence puts the counter back to zero.

**CAPTURE: `ext-lockout-05-lockout-policy-configured.png`**
All three defined.

### Step 8. Close the editor and refresh policy on the domain controller

**On `DC-1`:** close the Group Policy Management Editor window, then in the elevated PowerShell window
from Step 1:

```
gpupdate /force
```

Wait for both the computer and user policy success lines.

> **The video runs `gpupdate /force` on `CLIENT-1`. Run it on `DC-1` instead, and this is worth
> understanding rather than copying.**
>
> `CLIENT-1` never evaluates whether `bhastings` has crossed the threshold. It packages the
> credential and hands it to `DC-1`, and `DC-1` decides. The machine that has to be holding the
> policy is the domain controller.
>
> The video's client-side `gpupdate` is harmless but it is not what made the lockout start working.
> Domain controllers refresh Group Policy on a five minute cycle by default, rather than the 90
> minutes plus random offset that member computers use, so `DC-1` picks the policy up on its own
> within about five minutes whether you touch it or not. Josh spent roughly three and a half
> minutes between editing the GPO and retesting, and the retest worked. Running `gpupdate` on the
> DC removes that timing from the equation entirely.
>
> If you want to reproduce the video's verification step, `gpresult /r` from an elevated prompt
> lists the GPOs applied to the machine you run it on and when policy last refreshed. It is a
> genuinely useful command. It is just answering a question about the client that does not bear on
> whether domain accounts lock out.

### Step 9. Prove the policy is live

**On `DC-1`,** in the same elevated PowerShell window, run the same command from Step 1:

```
net accounts /domain
```

The three lines have changed:

```
Lockout threshold:                                    5
Lockout duration (minutes):                           30
Lockout observation window (minutes):                 30
```

**CAPTURE: `ext-lockout-06-net-accounts-after.png`**
The full output.

Captures 01 and 06 are a matched before-and-after pair and they should be presented that way in the
README. This is the fastest possible check on a real network and worth committing to memory. It
reads the effective domain account policy without opening a console.

> **`net accounts` without `/domain` reads the local SAM.** On `DC-1` the two agree, because a
> domain controller has no local SAM. On `CLIENT-1` they do not. Run it bare on the client and you
> see the local machine's untouched policy. That is not a failed deployment, it is the
> OU-versus-domain-root distinction showing up from another angle.

---

## Part 2: Verify auditing before you generate evidence

> Video: no equivalent. This part is the reason the video's log review fails.

Do this **before** triggering the lockout. Security events that were not audited when they happened
cannot be recovered afterward. If auditing is off, the only fix is to turn it on and run Part 3
again.

### Step 1. Check the audit subcategories on DC-1

**On `DC-1`,** in the elevated PowerShell window:

```
auditpol /get /category:*
```

That prints the whole audit policy with no quoting at all, which matters because PowerShell parses
quotes and commas before a legacy command ever sees them. The filtered form you will find online,
`auditpol /get /category:"Account Management","Account Logon"`, is written for Command Prompt and
misbehaves in PowerShell. The full dump is about sixty lines, it is easy to read, and it makes a
better capture anyway.

Find these three rows in the output:

| Subcategory | Needed for | Required setting |
| --- | --- | --- |
| User Account Management | 4740, 4767, 4724, 4725, 4722, 4738 | Success |
| Kerberos Authentication Service | 4771 | Success and Failure |
| Logon | 4625 | Success and Failure |

On a Server 2022 domain controller, User Account Management normally audits Success out of the box,
so the account management events will be there. **Kerberos Authentication Service is the one to
actually check**, because it records the bad password attempts and depending on how the domain was
stood up it may be recording Success only.

If it does not show Failure:

```
auditpol --% /set /subcategory:"Kerberos Authentication Service" /failure:enable
```

**The `--%` is required and is not a typo.** It is PowerShell's stop-parsing token. Everything after
it is handed to `auditpol` verbatim, quotes intact. Without it PowerShell strips the quotes and
`auditpol` sees `Kerberos`, `Authentication`, and `Service` as three separate arguments and fails.
Any legacy command taking a quoted argument with spaces needs this, and it is worth remembering well
past this lab.

Re-run `auditpol /get /category:*` and confirm the row now reads `Success and Failure`.

**CAPTURE: `ext-lockout-07-auditpol-dc1.png`**
The output with all three subcategories visible and correct.

> **`auditpol` changes are local and Group Policy will overwrite them.** If any GPO applying to
> this machine defines the same subcategory under **Computer Configuration > Policies > Windows
> Settings > Security Settings > Advanced Audit Policy Configuration**, the next refresh silently
> reverts your change. Nothing defines it in this domain, so the local setting persists here. In a
> managed environment the correct move is a GPO, not `auditpol`.

### Step 2. Check the audit subcategory on CLIENT-1

**On `CLIENT-1`,** log in as `SMITHLAB\Keenan_Admin`, right-click **Start** > **Terminal (Admin)**,
click **Yes** at the UAC prompt, confirm the tab prompt begins with `PS`, then:

```
auditpol /get /subcategory:"Logon"
```

Confirm `Success and Failure`. If it reads Success only:

```
auditpol --% /set /subcategory:"Logon" /failure:enable
```

**Sign out of `CLIENT-1` when done.** Part 3 needs a clean logon screen.

---

## Part 3: Trigger the lockout

> Video: `04:08` fails eleven logons with no policy in place and gets in anyway, `15:03` retries
> after the policy is configured and locks out on the sixth attempt.

### Step 1. Establish the baseline

**On `DC-1`,** in elevated PowerShell:

```powershell
Get-ADUser bhastings -Properties BadLogonCount,LockedOut,lockoutTime |
  Format-List Name,BadLogonCount,LockedOut,lockoutTime
```

`BadLogonCount` should be `0`. If it is not, either wait out the 30 minute observation window or log
in once successfully as that user to reset it.

### Step 2. Feed it bad passwords

**On `CLIENT-1`,** at the logon screen:

1. Click **Other user** in the bottom-left if a different account is shown.
2. Username: `SMITHLAB\bhastings`
3. Password: a deliberately wrong one.
4. Press **Enter**.
5. Observe: *"The user name or password is incorrect. Try again."*
6. Click **OK**.

**CAPTURE: `ext-lockout-08-client1-bad-password-error.png`**
Capture this on the **first** failure, before
the account is locked. You need it to contrast against the next capture. The two messages are
different and that difference is the point of the lab.

7. Repeat. The account locks **on** the fifth failure, but the fifth attempt still returns the
   ordinary bad-password message, because the lock is applied as a result of that attempt rather
   than before it. The message you are looking for appears on the **sixth** attempt:

> *"The referenced account is currently locked out and may not be logged on to."*

**CAPTURE: `ext-lockout-09-client1-locked-out-error.png`**

The video confirms this. Josh counts out loud through six failures and the lockout message appears
on the sixth, with the threshold set to five. If you stop at five because the threshold said five,
you will conclude the policy is broken when it is working correctly.

> **Use a genuinely novel wrong password every time.** Since Server 2003, domain controllers do not
> increment `badPwdCount` when the submitted password matches one of the account's **two most
> recent previous passwords**. That exists so a phone or a mapped drive holding a stale credential
> does not lock people out repeatedly. Here it means that if your "wrong" password happens to be
> one this account previously held, that attempt is free and the fifth failure never arrives. Mash
> the keyboard differently each time.

> **The account can lock earlier than you expect.** Windows credential providers sometimes submit
> more than once per press, and any background process still holding a stale credential contributes
> to the same counter. If you skipped the credential cleanup in Before You Start, that is the
> likely cause. The policy is not broken.

### Step 3. Confirm it from the server side

**On `DC-1`,** in elevated PowerShell:

```powershell
Search-ADAccount -LockedOut | Select-Object Name,SamAccountName,LockedOut
```

`bhastings` should be listed. This is the command you would actually use on a real network, where
you know someone is locked out but not who.

Client-side and server-side evidence together is the standard Lab 1 set. The message on `CLIENT-1`
says the account is locked. This proves `DC-1` agrees.

---

## Part 4: Unlock the account

> Video: `15:52` finds the account with right-click Find, `16:13` opens the Account tab and unlocks,
> `16:33` logs in successfully, `17:21` confirms with `whoami`.

### Step 1. Find the account without browsing for it

You have roughly a thousand users in `_USERS`. Scrolling is not the answer.

**On `DC-1`:**

1. Click **Start**, type `dsa.msc`, press **Enter**.
2. In the left pane, right-click **smithlab.local**.
3. Click **Find...**
4. In the **Name** field, type `bhastings`.
5. Click **Find Now**.
6. Double-click the result in the pane below.

This is the video's technique and it is the right one. Learn it now, because every real environment
is too large to browse.

### Step 2. Unlock

**On `DC-1`,** in the user's properties window:

1. Click the **Account** tab.
2. Below the **User logon name** fields, find the checkbox reading **Unlock account. This account
   is currently locked out on this Active Directory Domain Controller.**

**CAPTURE: `ext-lockout-10-dc1-aduc-unlock-checkbox.png`**
The Account tab with the checkbox visible and
enabled, before you tick it.

3. Tick it, click **Apply**, click **OK**.

If the checkbox is greyed out or absent, the account is not currently locked.

**PowerShell equivalent**, which is what you would use at scale:

```powershell
Unlock-ADAccount -Identity bhastings
```

### Step 3. Verify end to end

**On `CLIENT-1`,** at the logon screen, log in as `SMITHLAB\bhastings` with the **correct**
password. It works. The password never changed. Only the lock state did.

Once logged in, right-click **Start** > **Terminal** and run (`whoami` works in either shell):

```
whoami
```

Expected: `smithlab\bhastings`

---

## Part 5: Reset the password

> Video: `18:22` right-click Reset Password, sets it to `Password2`, mentions the unlock checkbox
> in the same dialog. The video stops there. Step 3 and the pitfall below are not in it.

### Step 1. Sign the user out first

**On `CLIENT-1`,** sign out of `bhastings`. Resetting a password while that account holds a live
session leaves the session running on the old credential and muddies the demonstration.

### Step 2. Reset from ADUC

**On `DC-1`,** in Active Directory Users and Computers:

1. Find `bhastings` again using right-click **smithlab.local** > **Find...**
2. Right-click the account in the results.
3. Click **Reset Password...**
4. Enter a new password twice. Use one this account has never held.
5. Tick **User must change password at next logon**.

> **If that checkbox is greyed out, `Password never expires` is set on the account.** The two are
> mutually exclusive. Every account created by the bulk provisioning script during the base build
> carries `Password never expires`, so this applies to all 1000 of them and not just your test user.
>
> Fix it on the **Account** tab of the user's properties: untick **Password never expires**, click
> **Apply**, then return to Reset Password. It is worth noticing rather than routing around, because
> a credential that never expires and never has to be rotated is the pattern that turns one leaked
> password into permanent access. It belongs in the README's lab-only shortcuts table next to the
> shared hardcoded password.
6. Leave **Unlock the user's account** ticked if offered. It does no harm.

**CAPTURE: `ext-lockout-11-dc1-reset-password-dialog.png`**
The dialog with the must-change box ticked.

7. Click **OK**, then **OK** on the confirmation.

> **"The password does not meet the length, complexity or history requirements of the domain."**
> Three separate policies throw this one message and the dialog will not tell you which. Default
> domain policy: minimum length 7, complexity enabled, last 24 passwords remembered. **History is
> the one that catches people here**, because the obvious move is to reset back to `Password1` and
> that password is still in history for this account.

### Step 3. Observe the forced change on the client

The video ticks nothing and stops at the reset. This step is where the interesting behavior is.

**On `CLIENT-1`,** at the logon screen, log in as `SMITHLAB\bhastings` with the new password.

Windows does not log you in. It shows *"The user's password must be changed before signing in"* and
takes you straight to a change screen.

**CAPTURE: `ext-lockout-12-client1-forced-password-change.png`**

Enter a new password twice and press **Enter**. The login completes.

> **Minimum password age is one day, and it applies to the user but not to the admin.** Ticking
> "User must change password at next logon" sets `pwdLastSet` to `0`, which forces the prompt and
> exempts that one change from minimum age. So the forced change works.
>
> A **second** change the same day does not. Press Ctrl+Alt+Del, choose **Change a password**, and
> you get the same generic length-complexity-history error while the real cause is the one-day
> minimum age. An admin reset in ADUC bypasses minimum age. A user-initiated change does not. Worth
> knowing before you spend twenty minutes chasing a complexity rule that was never the problem.

---

## Part 6: Disable and re-enable the account

> Video: `18:57` through `21:03`. This part matches the video almost exactly.

Locked and disabled are not the same thing, and this part exists to prove it. Locked is a temporary
state produced by failed authentication that clears itself. Disabled is a deliberate administrative
action that clears only when an administrator reverses it. They produce different messages,
different event IDs, and different failure sub-status codes.

### Step 1. Disable

**On `CLIENT-1`,** sign out of `bhastings`.

**On `DC-1`,** in Active Directory Users and Computers, find `bhastings`, right-click it, click
**Disable Account**, click **OK**.

Search for the account again. Its icon now carries a small downward-pointing arrow. The video points
this out and notes it is hard to see. It is.

**CAPTURE: `ext-lockout-13-dc1-aduc-account-disabled.png`**
The results pane with the disabled icon visible.
Zoom the VM display or crop tight if the arrow is illegible at capture resolution.

**PowerShell equivalent:** `Disable-ADAccount -Identity bhastings`

### Step 2. Observe the disabled error

**On `CLIENT-1`,** at the logon screen, log in as `SMITHLAB\bhastings` with the **correct**
password.

> *"Your account has been disabled. Please see your system administrator."*

**CAPTURE: `ext-lockout-14-client1-account-disabled-error.png`**

**Optional, and it produces no capture.** While `bhastings` is disabled, sign in on this same machine
as a different domain user. `rjennette` is the account Extension Lab 2 used to demonstrate nested
group membership, so she is already established here as the second known-good user. She logs in
normally.

That proves two things the rest of the lab does not. **An account state follows the account, not the
workstation**, which is worth being able to say precisely, because users report this constantly as
"the computer locked me out." And **the policy is domain-wide while the counter is per-account**: one
GPO at the domain root governs every user, and Bill burning five attempts costs Ramona nothing.

Skip it if you are moving quickly. Nothing downstream references it and the screenshot manifest does
not include it. Sign her out before continuing.

This capture belongs next to `ext-lockout-09` in the README. Same user, same machine, two completely
different messages, because the underlying account states are different. That side-by-side is the
clearest single artifact this lab produces.

### Step 3. Re-enable and verify

**On `DC-1`,** find `bhastings`, right-click, click **Enable Account**, click **OK**. Search again
and confirm the down arrow is gone.

**On `CLIENT-1`,** log in as `SMITHLAB\bhastings`. It works immediately.

> The video makes a point worth repeating: do not reflexively re-enable a disabled account. Accounts
> get disabled for reasons, usually offboarding or a compromise, and the person asking you to turn
> it back on is not always the person authorized to make that call. Re-enabling here is for the sake
> of the lab.

---

## Part 7: Read the logs

> Video: `21:08` through `25:59`. The video searches by username, finds nothing useful on `DC-1`,
> says on camera *"I don't actually know where they're going to show up,"* moves to `CLIENT-1`,
> hits a permissions problem, and eventually finds event 4625. It never locates the lockout event.
> Everything below is the version that finds all of it.

Everything above generated evidence. This part collects it. Do `DC-1` first.

### Step 1. Open the Security log on DC-1

**On `DC-1`:** click **Start**, type `eventvwr.msc`, press **Enter**. In the left pane expand
**Windows Logs** and click **Security**.

> **Do not search this log by username the way the video does.** The Security log on a domain
> controller holds tens of thousands of events and Find walks them one at a time, in order, matching
> a text string. It is why the video surfaces a run of logon and logoff events and then gives up
> without ever reaching the lockout. Filter by event ID instead. It is a different operation and it
> returns the answer in about a second.

### Step 2. Find the lockout

**On `DC-1`,** in the right-hand **Actions** pane, click **Filter Current Log...**, type `4740` in
the **Includes/Excludes Event IDs** field, click **OK**. Click the resulting event and read the
**General** tab.

The body names the account that was locked and, in **Caller Computer Name**, the machine the bad
passwords came from. On a real network that field is the entire investigation. It tells you whether
this is a user fat-fingering at their desk or an attacker spraying credentials from somewhere that
should not have them.

**CAPTURE: `ext-lockout-15-dc1-event-4740.png`**
General tab showing subject account and caller computer
name.

**This event exists on `DC-1` and only on `DC-1`.** The client never knows the account was locked.
It only knows its authentication attempt was refused. That is the architecture the whole lab has
been demonstrating, recorded as a single log entry.

### Step 3. Find the failed pre-authentications

**On `DC-1`,** click **Filter Current Log...** again, clear the field, type `4771`, click **OK**.
Open one and find **Failure Code**. It should read `0x18`.

You should see roughly one 4771 per bad password attempt.

**CAPTURE: `ext-lockout-16-dc1-event-4771.png`**

> **This is why the video came up empty on the domain controller.** Josh goes looking for failed
> logins on `DC-1`, does not find them, and concludes they must only be on the client. They were
> there the whole time under an event ID he was not looking for.
>
> Most people expect 4625, "an account failed to log on." On a domain controller authenticating a
> domain-joined Windows client, the failure surfaces as **4771**, because the client is using
> Kerberos and the failure happens at pre-authentication. Failure code `0x18` means the
> pre-authentication data was bad, which for a password logon means the password was wrong.
>
> **4625 does appear on the DC** when the authentication came in over NTLM instead, which happens
> when a client connects by IP rather than name, from a non-domain-joined machine, or from an older
> client. You may see both. If you see only 4771, that is correct, not a gap in your evidence.
>
> If 4771 is missing entirely, Kerberos Failure auditing was off when you ran Part 3. Turn it on
> and re-run Part 3. Events cannot be recovered retroactively.

### Step 4. Find the administrative actions

**On `DC-1`,** click **Filter Current Log...**, clear the field, and enter:

```
4724,4725,4722,4767,4738
```

Click **OK** and read the list top to bottom. It is the complete administrative narrative of this
lab in reverse order: unlock, reset, disable, enable, each accompanied by a 4738 recording that the
account object changed.

**CAPTURE: `ext-lockout-17-dc1-account-management-events.png`**
Capture the filtered **list**, not a single
event detail. The sequence is the story.

> Every one of these records **Subject: Account Name** as `Keenan_Admin`, not `Administrator`. That
> is the audit trail the main README argued for when it justified creating a separate named admin
> account instead of working from the built-in one. This is the lab where that argument stops being
> theoretical.

### Step 5. Read the client-side log

The video hits a genuine problem here and solves it in a way worth stealing. Josh is logged into the
client as a standard user and cannot open the Security log at all, because reading it requires
administrative rights. Rather than signing out, he launches Event Viewer as a different user.

**On `CLIENT-1`,** logged in as `bhastings`:

1. Click **Start** and type `eventvwr.msc`.
2. In the results, right-click **Event Viewer** and click **Run as administrator**.
3. At the credential prompt, enter `SMITHLAB\Keenan_Admin` and the password.

If **Run as administrator** does not offer a credential prompt on your build, hold **Shift**,
right-click the result, and click **Run as different user** instead.

That technique is worth keeping. It is the everyday help desk move for doing privileged work on a
user's machine without making them log out of their session.

4. Expand **Windows Logs**, click **Security**.
5. In the **Actions** pane, click **Filter Current Log...**, event ID `4625`, click **OK**.

The log takes a while to populate. The video waits too.

Open several events and read these fields:

- **Failure Information > Status**
- **Failure Information > Sub Status**
- **Logon Information > Logon Type**

The sub-status is the useful one. `0xC000006A` means the username was valid and the password was
wrong. `0xC0000064` means the username itself does not exist. Same event ID, and on a real network
the difference between the two tells you whether someone is guessing passwords against a known
account or guessing at account names.

**CAPTURE: `ext-lockout-18-client1-event-4625.png`**
Capture one showing **Sub Status `0xC000006A`** (the
username is valid, the password is wrong) and **Logon Type 2**.

> **Do not go looking for a client-side 4625 marked locked out or disabled. There is not one.** In a
> Kerberos domain the client asks the domain controller for a ticket before any logon session is
> attempted locally. If the account is locked or disabled, the DC refuses at that point and the
> workstation never gets far enough to log an interesting failure. Every 4625 on `CLIENT-1` will be
> an ordinary bad password or an unknown username.
>
> The evidence that the account was locked lives on `DC-1`, as event 4740, which you already
> captured in Step 2. That is the architecture the whole lab has been demonstrating, showing up one
> more time: the client does not know and does not decide.

> **Your Logon Type will be `2` where the video's is `10`, and your source address will be empty.**
> Josh reads an IP address out of these events because he is connecting to the client over Remote
> Desktop. You are typing at the VirtualBox console, which is a local interactive logon. Logon Type
> 2 with a blank source address is correct for your build and is not missing data. Note it in the
> README rather than treating it as a defect, because knowing which logon type corresponds to which
> access method is itself the useful skill.

The timestamps are worth reading as a sequence the way the video does. The failures sit two or three
seconds apart, then there is a gap of roughly a minute where you were in ADUC unlocking the account,
then a success. That shape is what an incident looks like in a log, and reading shape rather than
individual entries is the actual skill.

### Event ID reference

| ID | Meaning | Logged on | Triggered by |
| --- | --- | --- | --- |
| 4625 | An account failed to log on | `CLIENT-1`, and `DC-1` if NTLM | Every failure in Parts 3 and 6 |
| 4740 | A user account was locked out | `DC-1` | Part 3, fifth failure |
| 4767 | A user account was unlocked | `DC-1` | Part 4 |
| 4771 | Kerberos pre-authentication failed | `DC-1` | Every failure in Part 3 |
| 4776 | The DC attempted to validate credentials (NTLM) | `DC-1` | Appears if any NTLM auth occurred |
| 4724 | An attempt was made to **reset** an account's password | `DC-1` | Part 5, admin reset |
| 4723 | An attempt was made to **change** an account's password | `DC-1` | Part 5, user-initiated change |
| 4725 | A user account was disabled | `DC-1` | Part 6 |
| 4722 | A user account was enabled | `DC-1` | Part 6 |
| 4738 | A user account was changed | `DC-1` | Accompanies most of the above |
| 4634 / 4647 | Logoff | Both | The events the video's username search surfaced instead |

The 4723 versus 4724 distinction is worth carrying. **4723 is a user changing their own password.
4724 is an administrator resetting someone else's.** In an investigation those are not close to the
same event.

### Failure sub-status codes for event 4625

| Sub Status | Meaning |
| --- | --- |
| `0xC0000064` | The username does not exist |
| `0xC000006A` | The username is correct, the password is wrong |
| `0xC0000234` | The account is currently locked out |
| `0xC0000072` | The account is disabled |
| `0xC0000070` | Logon from this workstation is not permitted |
| `0xC0000193` | The account has expired |
| `0xC0000071` | The password has expired |

**The locked-out and disabled codes are listed for reference and you will not see them in this lab.**
They appear where authentication runs over NTLM or against a local account. On a domain-joined
Windows client using Kerberos, those two states are refused by the domain controller before the
workstation attempts a logon session, so they are recorded on `DC-1` as event 4740 rather than on
the client.

`0xC0000064` versus `0xC000006A` is the pair to remember. A flood of `0xC0000064` is someone
guessing usernames. A flood of `0xC000006A` against a small set of valid usernames is a password
spray against accounts an attacker has already confirmed exist. Same event ID, very different
situations.

### Logon types seen in event 4625

| Type | Meaning | Where it comes from |
| --- | --- | --- |
| 2 | Interactive | Typing at the console, which is your build |
| 3 | Network | Accessing a share, which is what Lab 2 generated |
| 10 | RemoteInteractive | RDP, which is the video's build |

---

## Cleanup

Leave the GPO in place. Leave `bhastings` enabled and unlocked.

If you want the account back at its original state, reset the password once more without ticking
"User must change password at next logon."

The video ends by telling you to shut down your Azure resources but not delete them, because the
domain controller is needed for the DNS and file permissions labs. You have already built both of
those. The next lab on these machines is Windows Firewall and Wireshark.

Shut both VMs down from **inside Windows** using Start > Shut down, not VirtualBox's Power Off.
`DC-1` is a domain controller and the directory database needs to close cleanly.

---

## Complete screenshot list

Eighteen captures. This is the full set.

| # | File | Part | What it shows |
| --- | --- | --- | --- |
| 01 | `ext-lockout-01-net-accounts-before.png` | 1 | Lockout threshold reading `Never` by default |
| 02 | `ext-lockout-02-gpmc-gpo-linked-domain-root.png` | 1 | GPO linked at the domain root, link order 1 |
| 03 | `ext-lockout-03-lockout-policy-undefined.png` | 1 | Settings Not Defined, full nav path |
| 04 | `ext-lockout-04-suggested-value-changes.png` | 1 | The Suggested Value Changes dialog |
| 05 | `ext-lockout-05-lockout-policy-configured.png` | 1 | All three settings defined |
| 06 | `ext-lockout-06-net-accounts-after.png` | 1 | Same command, policy now live |
| 07 | `ext-lockout-07-auditpol-dc1.png` | 2 | Audit subcategories verified before evidence |
| 08 | `ext-lockout-08-client1-bad-password-error.png` | 3 | Incorrect password message, first failure |
| 09 | `ext-lockout-09-client1-locked-out-error.png` | 3 | Locked out message, sixth attempt |
| 10 | `ext-lockout-10-dc1-aduc-unlock-checkbox.png` | 4 | Account tab, unlock checkbox enabled |
| 11 | `ext-lockout-11-dc1-reset-password-dialog.png` | 5 | Reset Password with must-change ticked |
| 12 | `ext-lockout-12-client1-forced-password-change.png` | 5 | Forced change at logon |
| 13 | `ext-lockout-13-dc1-aduc-account-disabled.png` | 6 | Disabled account icon in ADUC |
| 14 | `ext-lockout-14-client1-account-disabled-error.png` | 6 | Account disabled message |
| 15 | `ext-lockout-15-dc1-event-4740.png` | 7 | Lockout event with caller computer name |
| 16 | `ext-lockout-16-dc1-event-4771.png` | 7 | Kerberos pre-auth failure, code `0x18` |
| 17 | `ext-lockout-17-dc1-account-management-events.png` | 7 | Filtered list: 4724, 4725, 4722, 4767, 4738 |
| 18 | `ext-lockout-18-client1-event-4625.png` | 7 | 4625 with a meaningful Sub Status |

All of these go in `D:\ad-lab\docs\images\` alongside the existing captures. The `ext-lockout-`
prefix keeps them sorted separately from the core build and from Labs 1 and 2.

**These captures are displayed in the [README](../README.md), not here.** This runbook is
instructions for building the lab. The README is the visual walkthrough of what the lab produced.

Both VMs are dark themed as of this lab, so these will not match the base project or the first two
extensions visually. That is intentional and reads as the project improving over time.

If you capture something useful that is not on this list, save it with the next free number and say
so. Extra evidence is easy to work in. Missing evidence means booting the VMs again.

---

## Build log

Snags continue in the existing [`BUILD-LOG.md`](../BUILD-LOG.md) at the repo root under a new
session heading. The log does not restart for extension labs.

```
[STEP x] Short title
WHAT I WAS DOING:
WHAT HAPPENED:
WHAT I TRIED:
WHAT ACTUALLY FIXED IT:
TIME LOST:
SCREENSHOT:
```

---

## Things that are likely to go sideways

**`Get-ADUser is not recognized as an internal or external command`.** You are in Command Prompt,
not PowerShell. The prompt will read `C:\Users\...>` instead of `PS C:\Users\...>`. Type
`powershell` and press Enter to switch that window in place, or reopen with right-click **Start** >
**Windows PowerShell (Admin)** on `DC-1`.

**`auditpol` rejects a subcategory name.** PowerShell stripped the quotes before `auditpol` saw
them. Put `--%` immediately after `auditpol` and before the switches. See Part 2.

**The account will not lock no matter how many times you try.** Check the GPO link. If it is on an
OU instead of the domain root, domain accounts are unaffected. See Part 1, Step 3. Confirm with
`net accounts /domain`, which should read `5` and not `Never`.

**You stopped at five attempts and nothing happened.** The account locks on the fifth failure but
the locked-out message appears on the sixth attempt. Try once more.

**The account locks after three attempts instead of five.** A background process is holding a stale
credential for `bhastings` and contributing to the same counter. A saved Credential Manager entry or
a leftover `net use` connection from Lab 2 is the usual culprit. Clear it while signed in as
`bhastings`, from a non-elevated prompt. See Before You Start.

**The counter never moves.** Your "wrong" password matches one of the account's two most recent
previous passwords, which the DC does not count. Use something obviously novel.

**No 4771 events on `DC-1`.** Kerberos Failure auditing was off when you ran Part 3. Turn it on and
re-run Part 3. Events cannot be recovered retroactively.

**No 4625 events on `CLIENT-1`.** Logon Failure auditing was off. Same fix, same limitation.

**Event Viewer on `CLIENT-1` shows nothing under Security.** You are running it as a standard user.
Relaunch it as `SMITHLAB\Keenan_Admin` using Run as administrator or Shift + right-click > Run as
different user. See Part 7, Step 5.

**No 4625 on `CLIENT-1` shows locked out or disabled.** Expected. Those states are refused by the
domain controller before the client attempts a logon session. The evidence is event 4740 on `DC-1`.

**Your 4625 events have no source IP address.** Expected. You are logging on at the console, Logon
Type 2. The video's IP address comes from its RDP connection.

**"User must change password at next logon" is greyed out.** The account has `Password never
expires` set, which the bulk provisioning script applies to every user it creates. Untick it on the
Account tab first. See Part 5.

**The password reset is rejected as not meeting requirements.** Almost always password history, not
complexity. You cannot reuse the last 24 passwords on that account, including `Password1`.

**A second password change on the same day is rejected.** Minimum password age is one day. The
error text blames complexity and is misleading.

**The unlock checkbox is missing in ADUC.** The account is not currently locked. Confirm with
`Search-ADAccount -LockedOut`.

**You cannot find the account in a thousand users.** Right-click `smithlab.local` > **Find...**

---

## A+ Core 2 objective coverage

- **2.1** Security concepts: authentication, account policies, lockout
- **2.5** Workstation security best practices: password policy, account management, disabling
  accounts, failed attempt lockout, timeout
- **2.6** Securing a workstation: account states and least privilege
- **1.2** Microsoft command-line and administrative tools: `gpupdate`, `gpresult`, `net accounts`,
  `auditpol`, Event Viewer, ADUC, Group Policy Management
- **4.x** Change and access management: documented administrative action and audit trail

Account lockout settings in particular are a recurring CompTIA question, usually framed as a
scenario where you have to pick the right one of threshold, duration, and observation window from a
description of the desired behavior. Part 1, Step 7 is written specifically to make that
distinction stick.
