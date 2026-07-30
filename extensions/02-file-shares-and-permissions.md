# Extension Lab 2: Network File Shares and Permissions

What a normal user can actually do with a share, why they sometimes see a folder they cannot
open, and what happens between share permissions and NTFS permissions when the two disagree.

This lab extends the domain built in the main runbook. It requires `DC-1` and `CLIENT-1` to
already exist and be joined. Nothing new is installed.

**Source material:** this lab follows the structure of Josh Madakor's "Network File Shares and
Permissions" exercise, taught in the CourseCareers IT program (module 4.7.2). That version is
built on Azure virtual machines. What follows is the on-premises translation, rebuilt and verified
on VirtualBox.

> **On the timestamps below.** Each step carries a reference like `Video: 05:35`, pointing at the
> matching moment in the CourseCareers module. **That course is paid, and you do not need it to
> follow this runbook.** Every step here is written to stand on its own. The timestamps are for
> people already enrolled who want to see a step demonstrated. If you are not enrolled, ignore
> them and nothing is lost.

**What is mine versus what is his.** The exercise structure and teaching sequence are his. The
on-premises translation, the share versus NTFS analysis in Part 5, the Kerberos token explanation,
and every capture in this document are mine.

---

## Which machine, every time

Every step states its machine in bold before the actions. `DC-1` is the server and holds the
folders, the shares, and Active Directory. `CLIENT-1` is the workstation and is where every access
test happens.

The rule of thumb: creating or permissioning anything happens on `DC-1`. Trying to open something
happens on `CLIENT-1`.

---

## Environment translation

| CourseCareers Azure version | This build |
| --- | --- |
| Azure Portal, start VMs | VirtualBox Manager, start `DC-1` then `CLIENT-1` |
| Remote Desktop to a public IP | VirtualBox console window for each VM |
| `mydomain.com\jane_admin` | `SMITHLAB\Keenan_Admin` |
| `mydomain.com\<someuser>`, password `Password1` | Any account from the `_USERS` OU, password `Password1` |
| Azure VNet | VirtualBox internal network, `intnet` |
| Client is Windows 10 | Client is Windows 11 Enterprise |

---

## Before you start

**Start `DC-1` first** and let it come up fully before starting `CLIENT-1`.

**Log into `DC-1` as `SMITHLAB\Keenan_Admin`.**

**Log into `CLIENT-1` as a normal domain user.** This is the opposite of the DNS lab and it is the
single most important setup detail here. The entire lab is about what a user *without* privilege
can and cannot do. Logging in as an admin makes every test pass and teaches you nothing.

Pick a user from the `_USERS` OU in Active Directory Users and Computers on `DC-1`. All 1000
generated accounts share the password `Password1`. Write down whichever one you pick, because you
will log in and out as that account several times.

> On the sign-in screen, click **Other user** and enter `SMITHLAB\username`. First login for a
> new profile takes a minute or two while Windows builds it. That is normal, not a hang.

> **Why `\\dc-1` will work now.** In the DNS lab, `DC-1` was registering both of its network
> adapters in DNS, including the NAT adapter no client can reach. Had that not been fixed, this
> lab is exactly where it would have surfaced: `\\dc-1` would connect roughly half the time and
> hang the rest. If you are following this runbook without having done Lab 1, check that
> `dc-1` resolves to only one address before starting.

---

## Part 1: Create the folders and share three of them

> Video: `03:14` creates the folders, `05:00` shows the permission table, `05:35` through `06:55`
> shares all three

**On `DC-1`**, open File Explorer (Windows key + E), go to `C:\`, and create four folders:

```
read-access
write-access
no-access
accounting
```

> The names describe what each folder is *for*, not what it *has*. A folder named `read-access`
> has no permissions at all until you set them. The names are a convenience so the lab is easy to
> follow, and they could just as easily be `zebra`, `armadillo`, and `horse`.

**CAPTURE: `ext-shares-01-four-folders-created.png`**

Now share three of them. Right-click each folder, **Properties**, **Sharing** tab, **Share**
button, then type the group name, click **Add**, set the permission level, and click **Share**.

| Folder | Group | Permission |
| --- | --- | --- |
| `read-access` | `Domain Users` | Read |
| `write-access` | `Domain Users` | Read/Write |
| `no-access` | `Domain Admins` | Read/Write |
| `accounting` | nothing yet | leave it alone |

**CAPTURE: `ext-shares-02-read-access-shared.png`**

**CAPTURE: `ext-shares-03-write-access-shared.png`**

**CAPTURE: `ext-shares-04-no-access-shared.png`**

`accounting` is deliberately left unshared. It comes back in Part 3, and its absence in Part 2 is
itself a result worth seeing.

---

## Part 2: Test access as a normal user

> Video: `07:32` moves to the client, `08:09` browses the UNC path, `08:45` recaps group
> membership, `09:20` through `09:56` runs all three tests

**On `CLIENT-1`**, logged in as your normal user, open File Explorer and type into the address
bar:

```
\\dc-1
```

You will see the three shares you created plus `NETLOGON` and `SYSVOL`, which are default shares
every domain controller has. Ignore those two.

**`accounting` does not appear.** It exists on `DC-1` and it is not shared, so from the network it
may as well not exist.

**CAPTURE: `ext-shares-05-dc1-share-list.png`**

Before testing, be clear about who you are. Your user is a member of **Domain Users**. It is
**not** a member of Domain Admins. Every result below follows from that one fact.

**Test 1, `read-access`.** Open it. You can see inside. Now right-click and try to create a new
text file. It fails with a permission error, and it will keep failing no matter how many times you
click Try Again.

**CAPTURE: `ext-shares-06-read-access-denied-write.png`** with the permission error visible.

**Test 2, `write-access`.** Open it, create a file, open the file, type something, save it. All of
it works. Domain Users has Read/Write here.

**CAPTURE: `ext-shares-07-write-access-file-created.png`**

**Test 3, `no-access`.** You cannot even open it. Not a matter of writing, you have no read
permission at all, because only Domain Admins was granted anything.

**CAPTURE: `ext-shares-08-no-access-denied.png`** with the access denied dialog visible.

> **The help desk translation.** "I can see the folder but I can't open it" and "I can open it but
> I can't save" are two different tickets with two different causes, and users describe both as
> "I don't have access." Knowing which one you are looking at tells you whether the fix is a group
> membership or a permission change.

---

## Part 3: The ACCOUNTANTS security group

> Video: `11:06` creates the OU and group, `12:19` shares the folder to it, `12:56` shows the
> failure, `13:31` through `14:07` fixes membership, `15:18` shows it working

This is how access is granted in a real environment. You do not permission a folder to a person.
You permission it to a group and put people in the group.

**On `DC-1`** for all of Part 3 up to the client test. Open Active Directory Users and Computers.

Create an **Organizational Unit** named `_GROUPS` at the domain root, alongside your existing
`_USERS` and `ADMINS` OUs. Right-click `smithlab.local` in the **left** pane, **New** >
**Organizational Unit**. It is not strictly required, and it keeps groups from scattering into the
default containers.

> **An OU is not a group.** If the new object shows "Security Group" in the Type column instead of
> "Organizational Unit," or does not appear in the left-hand tree, you created a group by mistake.
> Delete it and create it again as an OU. Only OUs appear in the left pane. An OU is a container
> for organizing objects and targeting Group Policy. A security group is a membership list you
> attach permissions to. You cannot put objects inside a security group, and you cannot permission
> a folder to an OU.

Right-click `_GROUPS`, **New** > **Group**. Name it `ACCOUNTANTS`, leave the defaults of **Global**
and **Security**, click OK.

**CAPTURE: `ext-shares-09-accountants-group-created.png`**

**Still on `DC-1`**, share the `accounting` folder to the group you just created. The folder lives
at `C:\accounting` on this machine, so open File Explorer here, not on the client.

Right-click `accounting`, **Properties**, **Sharing** tab, **Share** button, type `ACCOUNTANTS`,
click **Add**, set **Read/Write**, click **Share**.

> Watch the principal you type. In Part 2 this dialog is where a mistyped name granted a standard
> user Full control of `no-access`. Confirm it reads `ACCOUNTANTS` and nothing else before you
> click Share, and check the **Security** tab afterward to see what the wizard actually wrote.

**CAPTURE: `ext-shares-10-accounting-shared-to-group.png`**

**On `CLIENT-1`**, refresh `\\dc-1`. The `accounting` folder now appears, because it is shared.
Double-click it. **It fails.**

**CAPTURE: `ext-shares-11-accounting-visible-but-denied.png`**

That is the distinction worth understanding. Being shared makes a folder *visible* on the network.
Being permissioned makes it *openable*. Your user is not in `ACCOUNTANTS` yet, so it can see the
door and not walk through it.

**Sign out of `CLIENT-1` completely.** Not lock, not disconnect. Sign out.

**On `DC-1`**, open the `ACCOUNTANTS` group, **Members** tab, **Add**, type your user's name,
**Check Names**, OK, **Apply**, OK.

**CAPTURE: `ext-shares-12-user-added-to-group.png`**

**Sign back into `CLIENT-1`** as the same user. Browse to `\\dc-1\accounting`. It opens. Create a
file, edit it, save it. All of it works.

**CAPTURE: `ext-shares-13-accounting-access-works.png`**

> **Why the sign-out was mandatory.** Windows builds your Kerberos access token at logon and it
> lists every group you belong to at that moment. It is not rechecked while you stay signed in.
> Adding someone to a group changes Active Directory instantly and changes nothing about the
> session already running on their machine. This is why "log off and back on" is the correct first
> instruction after any group change, and why it is not the brush-off it sounds like. On a real
> help desk this single fact resolves a large share of "I was added to the group and still can't
> get in" tickets.

---

## Part 4: Groups inside groups

> Video: `16:33` explains the concept, `17:08` adds Domain Users to ACCOUNTANTS, `18:19` logs in as
> a different user to prove it

A group can be a member of another group, and membership flows through. This is how access scales
past a handful of people.

**On `DC-1`**, open `ACCOUNTANTS`, **Members** tab, **Add**, and this time add the group
**Domain Users** rather than a person. Apply, OK.

Every one of your 1000 generated accounts is a member of Domain Users, so every one of them is now
effectively a member of ACCOUNTANTS, without touching a single individual account.

**On `CLIENT-1`**, sign out and sign back in as a **completely different** user from `_USERS`, one
you have never added to anything. Same password, `Password1`. Browse to `\\dc-1\accounting`. It
opens, and you can write to it.

**CAPTURE: `ext-shares-14-nested-group-different-user.png`** with the username visible. Run
`whoami` in PowerShell in the same frame if you can, so the capture proves which account it is.

> **Why this matters more than it looks.** Nested groups are how real access sprawl happens.
> Someone grants a broad group membership in a narrow group as a convenience, nobody documents it,
> and two years later a thousand people have access to a finance share that was supposed to serve
> eight. Access reviews exist to catch exactly this. It is worth noticing that you just created
> that situation in about fifteen seconds.

**Undo it before moving on.** Remove Domain Users from the `ACCOUNTANTS` members list, so the lab
is left in a sane state for the next one.

---

## Part 5: Share permissions versus NTFS permissions

> Not in the video or the checklist. This is the part of file sharing that actually generates
> tickets, and the lab skips it.

Every shared folder in Windows is protected by **two independent permission sets**, and a user
receives whichever of the two is more restrictive.

- **Share permissions** apply only to access coming over the network.
- **NTFS permissions** apply always, whether the user is across the network or sitting at the
  machine.

If share says Read/Write and NTFS says Read, the user gets Read. Access is granted only where both
sets agree. Checking one and not the other proves nothing.

### Step 1. Open the folder's Properties on DC-1

**On `DC-1`:**

1. Press **Windows key + E** to open File Explorer.
2. In the left pane, click **This PC**.
3. Double-click **Local Disk (C:)**.
4. Right-click the **`write-access`** folder.
5. Click **Properties**.

A window titled `write-access Properties` opens with tabs across the top: General, Sharing,
Security, Previous Versions, Customize.

### Step 2. Read the NTFS permissions

1. Click the **Security** tab.
2. Look at the **Group or user names** box at the top.
3. Click each entry in turn and read the **Permissions for** box underneath it.

This is the NTFS list. Note which principals are here and what each one is allowed.

**Leave this window open.** You are going to compare it against the share list.

### Step 3. Read the share permissions

In the same `write-access Properties` window:

1. Click the **Sharing** tab.
2. Click the **Advanced Sharing** button. A window titled `Advanced Sharing` opens.
3. Click the **Permissions** button. A window titled `Permissions for write-access` opens.
4. Look at the **Group or user names** box, and click each entry to read its permissions below.

This is the share list. It is a completely separate set of entries from the Security tab.

> **Two buttons on the Sharing tab, and they do different things.** The **Share** button is the
> simple wizard. The **Advanced Sharing** button is where share permissions are actually edited.
> There is also a read-only view of this same list at **Security** tab > **Advanced** > **Share**
> tab, where Add and Remove are greyed out. That view is for looking, not editing.

### Step 4. Capture the comparison

Arrange the `Permissions for write-access` window so the Security tab list is also visible behind
it, then capture both in one frame.

**CAPTURE: `ext-shares-15-share-vs-ntfs-permissions.png`**

If you cannot get both in one shot, take two captures and name the second
`ext-shares-15b-share-permissions.png`.

### Step 5. Close everything without changing anything

1. Click **Cancel** on `Permissions for write-access`.
2. Click **Cancel** on `Advanced Sharing`.
3. Click **Cancel** on `write-access Properties`.

Cancel rather than OK, so nothing is modified. This step is observation only.

### What you should find, and why it matters

The simple **Share** button you used in Part 1 wrote to **both** lists at once. It put the named
group into the NTFS ACL, and it put **`Everyone` with Full Control** into the share permissions.
That is by design: the wizard opens the share wide and expects NTFS to do the real gatekeeping.

Two consequences, and you hit both of them earlier in this lab:

**One mistake lands in two places.** On `no-access`, a mistyped principal in that wizard wrote an
explicit NTFS entry granting a standard user Full control. The Sharing tab still looked correct.

**A wide-open share defeats group membership.** On `accounting`, `Everyone` had Full Control at the
share layer, so removing a user from `ACCOUNTANTS` changed nothing. The NTFS list looked correct in
isolation and was correct. It was simply not the gate that mattered.

The common production configuration is the opposite of what the wizard produces: set the share
permission to Full Control for Authenticated Users, then control everything through NTFS, so there
is exactly one place to look. Worth knowing before inheriting somebody else's file server.

## Cleanup

Leave everything in place. The next lab (account lockout and password resets) uses the same domain
and the same user accounts.

The video ends by saying you can delete the VMs. **Do not.** That instruction is for people
finishing the course. You have two more labs on these machines.

Shut both VMs down from **inside Windows** using Start > Shut down, not VirtualBox's Power Off.
`DC-1` is a domain controller and the directory database needs to close cleanly.

---

## Complete screenshot list

Fifteen captures. This is the full set.

| # | File | Part | What it shows |
| --- | --- | --- | --- |
| 01 | `ext-shares-01-four-folders-created.png` | 1 | The four folders on `C:\` |
| 02 | `ext-shares-02-read-access-shared.png` | 1 | `read-access` shared to Domain Users, Read |
| 03 | `ext-shares-03-write-access-shared.png` | 1 | `write-access` shared to Domain Users, Read/Write |
| 04 | `ext-shares-04-no-access-shared.png` | 1 | `no-access` shared to Domain Admins, Read/Write |
| 05 | `ext-shares-05-dc1-share-list.png` | 2 | `\\dc-1` from the client, `accounting` absent |
| 06 | `ext-shares-06-read-access-denied-write.png` | 2 | Can open, cannot create a file |
| 07 | `ext-shares-07-write-access-file-created.png` | 2 | File created and saved successfully |
| 08 | `ext-shares-08-no-access-denied.png` | 2 | Cannot open at all |
| 09 | `ext-shares-09-accountants-group-created.png` | 3 | The ACCOUNTANTS group in `_GROUPS` |
| 10 | `ext-shares-10-accounting-shared-to-group.png` | 3 | `accounting` shared to ACCOUNTANTS |
| 11 | `ext-shares-11-accounting-visible-but-denied.png` | 3 | Visible on the network, access denied |
| 12 | `ext-shares-12-user-added-to-group.png` | 3 | The user in the group's Members tab |
| 13 | `ext-shares-13-accounting-access-works.png` | 3 | Access working after re-login |
| 14 | `ext-shares-14-nested-group-different-user.png` | 4 | A different user getting in through Domain Users |
| 15 | `ext-shares-15-share-vs-ntfs-permissions.png` | 5 | Security tab beside share permissions |

All of these go in `D:\ad-lab\docs\images\` alongside the existing captures. The `ext-shares-`
prefix keeps them sorted separately from the core build and from Lab 1.

**These captures are displayed in the [README](../README.md), not here.** This runbook is
instructions for building the lab. The README is the visual walkthrough of what the lab produced.

If you capture something useful that is not on this list, save it with the next free number and
say so. Extra evidence is easy to work in. Missing evidence means booting the VMs again.

---

## Build log

Snags continue in the existing [`BUILD-LOG.md`](../BUILD-LOG.md) at the repo root. The log does not
restart for extension labs.

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

**A share does not appear after you create it.** Press F5 in Explorer on `CLIENT-1`. The share list
is cached briefly.

**Everything works when it should not.** You are logged into `CLIENT-1` as an admin. Sign out and
sign back in as a normal user from `_USERS`.

**`\\dc-1` hangs instead of failing.** DNS is handing back an address the client cannot reach. See
Lab 1, Part 1, on the multihomed domain controller.

**Group membership change does not take effect.** Sign out fully and back in. See the Kerberos
token note in Part 3. Locking the screen is not enough.

**The user cannot be found when adding to the group.** Click **Locations** in the Select Users
dialog and make sure it is set to the domain rather than the local machine.

---

## A+ Core 2 objective coverage

- **2.1** Physical and logical security concepts: permissions, principle of least privilege
- **2.5** Workstation security: user and group permissions, share versus NTFS
- **1.2** Command-line and administrative tools: ADUC, share management
- **4.x** Change and access management concepts

Share versus NTFS in particular is a recurring CompTIA question, usually framed as a scenario
where a user has one permission over the network and a different one locally.
