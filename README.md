# Active Directory Home Lab

A working Windows Server 2022 domain built from scratch in VirtualBox: domain controller, DNS,
DHCP, NAT routing, ~1000 provisioned user accounts, and a domain-joined Windows 11 client that
authenticates against it.

Everything below was built and verified by hand. The final screenshot is a domain user logging
into the client and `whoami` returning the domain identity, which is the end-to-end proof that
every layer underneath it works.

![Domain login verified](docs/images/24-client1-domain-login.png)

---

## Why this lab

Identity is the layer everything else in security sits on. Before you can reason about
privileged access, least privilege, or session control, you have to understand what a directory
actually does: how a machine finds a domain, how it joins one, how a credential gets validated,
and what breaks when any of that is misconfigured.

This lab builds that from an empty hypervisor.

---

## Environment

| | |
|---|---|
| Domain | `smithlab.local` (NetBIOS `SMITHLAB`) |
| Domain controller | `DC-1`, Windows Server 2022 (Desktop Experience), static `172.16.0.1` |
| Client | `CLIENT-1`, Windows 11 Enterprise, DHCP-assigned |
| Internal subnet | `172.16.0.0/24` |
| DHCP scope | `172.16.0.100` to `172.16.0.200` |
| Hypervisor | Oracle VirtualBox |
| Host | Windows 11, i5-10300H, 32 GB RAM |

### Topology

```
        Host / Internet
              |
        [ NAT adapter ]
              |
      +-------------------+
      |      DC-1         |   AD DS  |  DNS  |  DHCP  |  RRAS/NAT
      |   172.16.0.1      |
      +-------------------+
              |
        [ intnet ]  VirtualBox Internal Network, 172.16.0.0/24
              |
      +-------------------+
      |    CLIENT-1       |   Windows 11 Enterprise, DHCP
      +-------------------+
```

`DC-1` carries two NICs. Adapter 1 is NAT (renamed `internet`) and faces the host network.
Adapter 2 is an Internal Network named `intnet` (renamed `internal`) and is the isolated lab
side. `CLIENT-1` has a single adapter on `intnet` and no path to the outside except through
`DC-1`.

That split is the point of the design. `CLIENT-1` cannot reach the internet on its own, so when
it does reach the internet, that traffic provably routed through the domain controller. It also
means the lab domain never touches the home network.

### Roles on DC-1

| Role | What it does here |
|---|---|
| AD DS | The directory itself. Holds users, groups, OUs, and computer accounts. |
| DNS | Installed automatically during promotion. Clients locate the domain via DNS SRV records, so without this nothing joins. `DC-1` points at `127.0.0.1` for its own resolution. |
| DHCP | Hands clients an address, gateway, DNS server, and the domain suffix. |
| RRAS / NAT | Routes `intnet` traffic out through Adapter 1. |

---

## Build walkthrough

### 1. VM creation and network configuration

`DC-1` was created with the VirtualBox Unattended Install wizard explicitly disabled, so the
Server 2022 ISO boots into the normal Windows setup instead of an automated one.

![DC-1 VM settings](docs/images/01-dc1-vm-settings.png)
![Adapter 1, NAT](docs/images/02-dc1-network-adapters-adapter-1.png)
![Adapter 2, Internal Network](docs/images/02-dc1-network-adapters-adapter-2.png)

### 2. Server install, NIC renaming, static IP, hostname

Both adapters arrive named `Ethernet` and `Ethernet 2`, which is unusable once you start
configuring them. Renaming them to `internet` and `internal` before touching anything else
prevents assigning a static IP to the wrong interface.

![Server installed](docs/images/04-dc1-server-installed.png)
![NICs renamed](docs/images/05-dc1-nics-renamed.png)
![Static IP on the internal adapter](docs/images/06-dc1-static-ip.png)
![Server renamed to DC-1](docs/images/07-dc1-renamed.png)

### 3. AD DS install and domain promotion

![AD DS role installed](docs/images/08-adds-role-installed.png)
![Domain promoted](docs/images/09-domain-promoted.png)

### 4. Dedicated domain admin account

An OU for administrative accounts, a user account inside it, and that account added to
`Domain Admins`.

![Admins OU and user](docs/images/10-admins-ou-and-user.png)
![Domain Admins membership](docs/images/11-domain-admins-membership.png)
![Logged in as the domain admin](docs/images/11b-logged-in-as-domain-admin.png)

The third screenshot is worth a note. On a domain controller, ordinary domain users are denied
local logon by default. Reaching a desktop as `SMITHLAB\Keenan_Admin` is itself evidence that
the account carries elevated rights, independent of what the group membership dialog says.

### 5. NAT routing and DHCP

![RRAS NAT configured](docs/images/12-rras-nat-configured.png)
![DHCP scope active](docs/images/13-dhcp-scope-active.png)

The default gateway is set through `003 Router` under **Scope Options**, not Server Options.
The DHCP configuration wizard writes it there, and looking for it in the wrong place is a
common way to lose twenty minutes.

### 6. Bulk user provisioning

Roughly 1000 accounts created in a `_USERS` OU via PowerShell, so the directory contains
realistic volume rather than three test users.

![PowerShell script running](docs/images/14-powershell-script-running.png)
![Users visible in ADUC](docs/images/15-aduc-users-created.png)

### 7. Client build and domain join

`CLIENT-1` is created only after `DC-1` is fully configured. It depends on the DHCP server for
its address, so building it earlier produces a machine sitting on an APIPA address that can do
nothing.

![CLIENT-1 VM settings](docs/images/16-client1-vm-settings.png)
![CLIENT-1 network adapter](docs/images/17-client1-network-adapter.png)
![CLIENT-1 installed](docs/images/18-client1-installed.png)

### 8. Verification

![ipconfig on CLIENT-1](docs/images/19-client1-ipconfig.png)
![Ping out to the internet](docs/images/20-client1-ping-google.png)
![Domain join](docs/images/21-client1-domain-join.png)
![DHCP lease visible on the server](docs/images/22-dhcp-lease.png)
![Computer account in ADUC](docs/images/23-aduc-computers.png)

Four independent confirmations, each proving a different layer:

| Check | What it proves |
|---|---|
| `ipconfig` shows a `172.16.0.1xx` address, correct gateway, correct DNS, correct domain suffix | DHCP is serving the right options, not just an address |
| Ping to an external host succeeds | RRAS/NAT is routing `intnet` traffic out through `DC-1` |
| Domain join succeeds | DNS SRV resolution works and credentials validate against AD DS |
| Computer account appears in ADUC, lease appears in the DHCP console | The server side agrees with the client side |

And the final one, at the top of this README: a domain user logging into `CLIENT-1` and
`whoami` returning the domain identity.

---

## Design decisions

**A separate admin account instead of the built-in Administrator.** The built-in account is the
first thing an attacker tries, and its actions are hard to attribute. A named administrative
account gives an audit trail: logs show who did the thing, not just that Administrator did it.

**Two NICs on the DC.** Covered above. The isolation is deliberate, and it forces the routing
to be real rather than incidental.

**A full DHCP scope with options rather than static client addressing.** Static addresses would
have been faster and would have taught nothing. The scope is where the domain suffix and DNS
server get delivered, and those are the settings that make the domain join work.

---

## Lab-only shortcuts

These are here because leaving them out would be dishonest, and because knowing why they are
wrong is more useful than pretending they were not done.

| Shortcut | Why it is fine here | Why it would be unacceptable in production |
|---|---|---|
| IE Enhanced Security Configuration disabled | Isolated lab, needed to download the provisioning script | Removes a hardening control on the most privileged server in the environment |
| `Set-ExecutionPolicy Unrestricted` | Needed to run the unsigned provisioning script | Allows any script to run unsigned. Production uses `RemoteSigned` at minimum, with signed scripts |
| Every generated user has the same hardcoded password | Bulk-generated filler accounts with no real access | Single shared credential across 1000 accounts, no forced rotation, no complexity variance |
| Single domain controller | Nothing to replicate with | No redundancy. One failure takes down authentication for the entire domain |

No real credentials appear anywhere in this repo. Password placeholders are written as
`<LabPassword>`.

---

## What I would do differently

**`.local` was the wrong choice for the domain suffix.** It is a long-standing lab convention,
which is why it is used here, but Microsoft now discourages it for new deployments because
`.local` collides with mDNS/Bonjour and can produce resolution failures on networks with Apple
devices. A real build should use a subdomain of a domain you actually own, such as
`ad.example.com`.

**Snapshots of a domain controller are a trap outside this lab.** Reverting a DC in production
causes USN rollback, where the restored DC reuses update sequence numbers its replication
partners have already seen and quietly stops replicating. It is safe here only because this is
a single DC with no partners.

There is a second version of that trap that does apply here. Once `CLIENT-1` is domain-joined,
rolling `DC-1` back alone breaks the computer account password, and the client throws:

> The trust relationship between this workstation and the primary domain failed.

The fix is to snapshot both machines together, or to rejoin the client after a rollback.

**Group Policy is not covered here.** A domain without GPOs is a domain that is not actually
managing anything. That is picked up in the extension labs below.

---

## Extension Labs

The core build produced a working domain. These labs run on that same domain and go after the
things that actually generate help desk tickets.

Each one lives in `extensions/` as its own runbook, written for this environment rather than
copied from a cloud tutorial. The source material targets Azure. Translating it to VirtualBox
surfaced problems the cloud version hides, and those are documented rather than smoothed over.

---

### 1. DNS Records and the Local Resolver Cache

**Runbook:** [`extensions/01-dns-records-and-cache.md`](extensions/01-dns-records-and-cache.md)

Name resolution end to end: creating records, watching a client answer from cache instead of from
the server, and understanding why flushing the cache sometimes fixes nothing at all.

#### Part 1: The A record

A Windows client checks its local resolver cache first, then queries the DNS server. The hosts
file is not a separate step at lookup time: the DNS Client service preloads hosts entries into
that same cache, so a hosts entry is answered from cache along with everything else. The lab
starts by proving the name does not exist anywhere.

![ping and nslookup for mainframe both fail before any record exists](docs/images/ext-dns-01-ping-fails.png)

`nslookup` reports `Server: UnKnown` while still finding the server at `172.16.0.1`. It resolved
the address fine and could not resolve that server's own name, because naming it requires a
reverse lookup zone this build does not have.

An A record is created on the domain controller pointing `mainframe` at the internal adapter,
not the NAT adapter. That distinction matters: the record has to hold an address that means
something from the client's point of view, and only one of the two interfaces is on a network the
client can reach.

![The mainframe A record created in DNS Manager](docs/images/ext-dns-02-a-record-created.png)

![ping mainframe resolving to 172.16.0.1 from the client](docs/images/ext-dns-03-ping-succeeds.png)

Note the client appending the domain suffix on its own, resolving `mainframe.smithlab.local`.

To make the resolution order visible, a hosts file entry is added for a name that has no DNS
record at all.

![The hosts file with a zebra entry pointing at loopback](docs/images/ext-dns-04-hosts-file-zebra.png)

![ping zebra resolving to 127.0.0.1 without ever reaching the DNS server](docs/images/ext-dns-05-ping-zebra.png)

#### The multihomed domain controller problem

Reviewing the zone at this point surfaced something the tutorial version cannot produce. `DC-1`
has two adapters, NAT for outbound internet and internal for the lab network, and both were
registered in DNS. A lookup for `dc-1` returned two addresses in round-robin order, and roughly
half of them pointed at a network the client has no route to.

Nothing was broken yet. It would have surfaced later as intermittent timeouts on `\\dc-1` during
the file shares lab, which is far harder to diagnose than a consistent failure.

![The zone after removing the NAT address records](docs/images/ext-dns-06-multihomed-dns-cleanup.png)

Suppressing it needed three separate actions rather than one, and the first attempt did not hold.
Unchecking "Register this connection's addresses in DNS" on the adapter governs the DNS *client*
service only. The DNS *server* service registers zone records for every interface it listens on,
independently of that checkbox, and had to be restricted to the internal address.

![Restricting the DNS server to listen only on the internal address, verified with ipconfig /registerdns](docs/images/ext-dns-14-dns-interfaces-restricted.png)

A third record was marked static, created at role install time, and had to be deleted by hand.
Static records never age out and no registration setting touches them. Running
`ipconfig /registerdns` afterward confirmed the fix held.

There was a fourth wrinkle: rolling `DC-1` back to an earlier snapshot restored the zone along
with everything else, reintroducing records that had already been removed. The lesson is
ordering. Take the snapshot after the cleanup, not before.

#### Part 2: The stale cache

This is the part that is actually about help desk work. The record is changed on the server while
the client still holds the previous answer in memory.

![The mainframe record changed to a different address on the server](docs/images/ext-dns-07b-record-changed.png)

![The client answering from cache, then the flush, then the corrected answer](docs/images/ext-dns-07-stale-cache-ping.png)

Server and client disagree. The client never asked, because it found an answer in its own cache
first. That single frame contains the whole diagnostic sequence: the wrong answer, the flush, and
the right answer.

![The stale mainframe entry visible in the client's DNS cache](docs/images/ext-dns-08-displaydns-stale-entry.png)

Then the sharpest moment in the lab. After `ipconfig /flushdns`, the cached DNS answer is gone
and the hosts file entry still resolves.

![After the flush the hosts file entry survives while the cached DNS record is gone](docs/images/ext-dns-09-flush-zebra-survives.png)

The TTL values make it visible. Cached DNS answers carry the record's real TTL, observed at 4268
seconds. The hosts file entry shows 604800 seconds, seven days, which is the fixed TTL Windows
assigns to preloaded hosts entries rather than anything the name's owner published. The hosts file
also generated a matching reverse entry.

**Why the hosts entry survives the flush, corrected.** An earlier version of this write-up said
hosts entries are read from disk on every lookup and were never in the cache. That is wrong, and a
reader on TikTok caught it. Microsoft's documentation for `ipconfig /displaydns` states that the
resolver cache includes entries preloaded from the local Hosts file alongside records obtained from
queries. The flush genuinely empties the cache. The DNS Client service then immediately reloads the
hosts file back into it, which is why the entry is visible again a moment later. It reappears
because it was put back, not because it bypassed the cache.

The practical conclusion is unchanged, and the corrected mechanism explains it better: a machine
with a bad hosts file entry will not be fixed by flushing DNS, because the flush reloads the bad
entry from disk every single time. Knowing that is the difference between a five-minute fix and an
hour of guessing.

![The client resolving to the current address after the cache is cleared](docs/images/ext-dns-10-flushed-and-resolved.png)

**The help desk translation:** one user cannot reach a resource everyone around them can reach.
The resource's address changed, DNS was updated correctly, and that one machine is still holding
the old answer in cache, or has a stale hosts file entry from something someone did months ago.
A flush fixes the first. Only reading the hosts file fixes the second.

#### Part 3: CNAME aliases

A CNAME maps one name to another name, where an A record maps a name to an address.

![The search CNAME alias created in DNS Manager](docs/images/ext-dns-11-cname-created.png)

![nslookup resolving the alias through to its target, with the alias shown in the response](docs/images/ext-dns-12-cname-nslookup.png)

The response carries `Aliases: search.smithlab.local`, which is the CNAME chain made explicit.
This step also exercised the DNS server's forwarders, since resolving the target requires external
resolution. The Azure version never tests this, because Azure supplies resolution to the virtual
network automatically.

The lab closes on a failure worth understanding rather than working around.

![The browser rejecting the alias on a certificate name mismatch](docs/images/ext-dns-13-cname-browser-cert-error.png)

The name resolved correctly and the connection still failed, because the certificate is issued for
the target's real name and not the alias. The browser then upgraded to HTTPS on its own and HSTS
removed the option to proceed anyway. DNS resolution, TLS validation, and HSTS are three separate
layers, and satisfying the first does not satisfy the others.

#### What this lab produced beyond the tutorial

Four findings, none of which exist in the source material, because they only appear in a local,
multihomed, snapshot-driven environment:

- A domain controller advertising an address no client could reach, and the three separate
  settings required to stop it
- A snapshot rollback silently undoing that fix
- The stale cache demonstration failing twice, once from TTL expiry across a break and once from a
  reboot clearing a memory-resident cache, both of which made a working lab look broken
- A single-sided screenshot that could not prove a claim about two machines disagreeing, which
  changed how evidence is captured for the rest of the series

Full detail on all four is in [`BUILD-LOG.md`](BUILD-LOG.md).

---

### 2. Network File Shares and Permissions

**Runbook:** [`extensions/02-file-shares-and-permissions.md`](extensions/02-file-shares-and-permissions.md)

What a normal user can actually do with a share, why they sometimes see a folder they cannot open,
and what happens when share permissions and NTFS permissions disagree.

Every test in this lab was run from `CLIENT-1` signed in as a standard domain account with no
elevated rights, because the entire point is what someone without privilege can and cannot reach.

#### Setting up the shares

Four folders on the domain controller, three of them shared to different groups at different
permission levels, and one deliberately left unshared.

![Four folders created on the domain controller's C drive](docs/images/ext-shares-01-four-folders-created.png)

![read-access shared to Domain Users with Read permission](docs/images/ext-shares-02-read-access-shared.png)

![write-access shared to Domain Users with Read and Write permission](docs/images/ext-shares-03-write-access-shared.png)

![no-access shared to Domain Admins only](docs/images/ext-shares-04-no-access-shared.png)

#### What a standard user sees

Browsing the domain controller by UNC path from the client returns the three shares plus `NETLOGON`
and `SYSVOL`, which every domain controller has by default. The unshared `accounting` folder does
not appear at all. It exists on disk, and from the network it may as well not.

![The share list on the domain controller as seen from the client](docs/images/ext-shares-05-dc1-share-list.png)

Three tests, three different outcomes, all following from one fact: this account is a member of
Domain Users and not Domain Admins.

![Read access granted but file creation refused](docs/images/ext-shares-06-read-access-denied-write.png)

![Write access granted, a file created and saved successfully](docs/images/ext-shares-07-write-access-file-created.png)

![No access at all, the folder cannot be opened](docs/images/ext-shares-08-no-access-denied.png)

**The help desk translation:** "I can see the folder but I can't open it" and "I can open it but I
can't save" are two different tickets with two different causes, and users describe both as "I
don't have access." Knowing which one you are looking at tells you whether the fix is a group
membership or a permission change.

#### Permissioning by group

Access in a real environment is not granted to a person. It is granted to a group, and people are
added to the group.

![The ACCOUNTANTS security group created in a dedicated OU](docs/images/ext-shares-09-accountants-group-created.png)

![The accounting folder shared to the ACCOUNTANTS group](docs/images/ext-shares-10-accounting-shared-to-group.png)

Once shared, the folder becomes visible on the network immediately. It still cannot be opened,
because the user is not in the group yet.

![The accounting folder now visible on the network but refusing access](docs/images/ext-shares-11-accounting-visible-but-denied.png)

That distinction is worth sitting with. Sharing a folder makes it *visible*. Permissioning it makes
it *openable*. The user can see the door and cannot walk through it.

![The user added to the group's membership list](docs/images/ext-shares-12-user-added-to-group.png)

![Access working after signing out and back in](docs/images/ext-shares-13-accounting-access-works.png)

**Why the sign-out was mandatory.** Windows builds a Kerberos access token at logon that enumerates
every group the account belongs to at that moment, and it is not re-evaluated for the life of the
session. Adding someone to a group changes Active Directory instantly and changes nothing about the
session already running on their machine. This is why "log off and back on" is the correct first
instruction after any group change, and it is not the brush-off it sounds like.

It also works in reverse, which is the half people miss. **Removing** a membership does not revoke
access until that user's next sign-in, so an offboarding task is not actually complete while the
person is still logged in.

#### Groups inside groups

A group can be a member of another group, and membership flows through. Adding `Domain Users` to
`ACCOUNTANTS` gave all 1000 generated accounts access to the finance share without touching a
single individual account. Verified by signing in as an account that had never been added to
anything.

![A different user accessing the share purely through nested group membership](docs/images/ext-shares-14-nested-group-different-user.png)

This is how real access sprawl happens. Someone grants a broad group membership inside a narrow
group as a convenience, nobody documents it, and two years later a thousand people can read a
finance share that was meant for eight. Access reviews exist to catch exactly this. It took about
fifteen seconds to create.

#### Share permissions versus NTFS permissions

Not in the source material, and the thing that actually generates tickets. Every shared folder is
guarded by two independent permission sets, and a user receives whichever is more restrictive.
Share permissions apply only over the network. NTFS permissions apply always. Checking one and not
the other proves nothing.

![NTFS permissions on the Security tab](docs/images/ext-shares-15-share-vs-ntfs-permissions.png)

![Share permissions, a completely separate list, reached through Advanced Sharing](docs/images/ext-shares-15b-share-permissions.png)

The Windows Share wizard writes to both lists at once. It puts the named principal into the NTFS
ACL and puts **`Everyone` with Full Control** into the share permissions, on the assumption that
NTFS will do the real gatekeeping. Two failures in this lab came directly out of that behavior.

**A mistyped principal silently granted a standard user Full control.** On `no-access`, the wizard
received an individual account rather than `Domain Admins`, broke inheritance, and wrote an explicit
NTFS entry granting that account complete control of the folder. The Sharing tab still looked
correctly configured. The failure was invisible until access was tested as the user who was supposed
to be excluded.

**A wide-open share defeated group membership entirely.** On `accounting`, `Everyone` held Full
Control at the share layer, so removing a user from `ACCOUNTANTS` changed nothing. The NTFS list
looked correct and was correct. It simply was not the gate that mattered. Diagnosing this meant
checking the permission set that had not been inspected yet, after the obvious one had already been
cleared.

The common production configuration is the opposite of what the wizard produces: set the share
permission broadly and control everything through NTFS, so there is exactly one place to look.

Full detail on both is in [`BUILD-LOG.md`](BUILD-LOG.md).

---

### 3. Account Lockout, Unlocking, and Password Resets

**Runbook:** [`extensions/03-account-lockout-and-passwords.md`](extensions/03-account-lockout-and-passwords.md)

Account lockouts are the single most common ticket a help desk sees. This lab configures the policy
that produces them, triggers one deliberately, works through the full administrative response, and
then finds every step of it again in the security logs.

The theme running through all of it: **the domain controller decides, the workstation finds out.**

#### Part 1: There is no lockout policy until you make one

The source video opens by failing eleven logons in a row and getting signed in anyway. That is not
a mistake in the demonstration. It is the default state of a Windows domain.

![net accounts /domain showing lockout threshold set to Never](docs/images/ext-lockout-01-net-accounts-before.png)

`Lockout threshold: Never`. Out of the box, a domain account can be guessed at indefinitely. Every
protection in this lab is opt-in, and a domain nobody has configured has none of it.

The policy goes in a new GPO linked at the **root of the domain**, not on an OU.

![The Account Lockout Policy GPO linked at the smithlab.local node at link order 1](docs/images/ext-lockout-02-gpmc-gpo-linked-domain-root.png)

That placement is not a preference. Account Policies behave differently from every other Group
Policy setting: for domain user accounts they are only honored from a GPO linked at the domain root,
because the domain controllers read and enforce them at authentication time. Link the same GPO to an
OU and it does not error, it does not warn, and it does not work. It quietly applies to the local SAM
database of the computers in that OU instead, so local accounts get a lockout threshold and domain
accounts get nothing.

![Account lockout duration, threshold, and reset counter all reading Not Defined](docs/images/ext-lockout-03-lockout-policy-undefined.png)

Setting the threshold first triggers a dialog worth pausing on. Windows recognizes that a threshold
with no duration and no observation window is an incomplete policy and proposes the other two.

![The Suggested Value Changes dialog proposing 30 minutes for duration and reset counter](docs/images/ext-lockout-04-suggested-value-changes.png)

![All three settings defined: 30 minute duration, 5 attempt threshold, 30 minute observation window](docs/images/ext-lockout-05-lockout-policy-configured.png)

Three settings that get confused constantly. **Threshold** is how many failures it takes.
**Duration** is how long the lock lasts before clearing itself. **Reset counter after** is the
observation window, meaning how long a single failure is remembered.

![The same net accounts command now reporting threshold 5, duration 30, observation window 30](docs/images/ext-lockout-06-net-accounts-after.png)

`gpupdate /force` runs on the **domain controller**, not the client. The client never evaluates
whether an account has crossed the threshold. It packages the credential, hands it to `DC-1`, and
`DC-1` decides. The policy has to reach the domain controller. It does not have to reach the client.

#### Part 2: The domain controller audits the response, not the incident

This part has no equivalent in the source material, and it is the reason the source material's log
review comes up empty.

![auditpol output with Kerberos Authentication Service, Logon, and Account Lockout all set to Success and Failure](docs/images/ext-lockout-07-auditpol-dc1.png)

A Server 2022 domain controller ships with `User Account Management` auditing **Success**, so it
faithfully records an administrator unlocking an account. It ships with `Kerberos Authentication
Service` failure auditing **off**, so it records nothing about the bad passwords that locked it.

The administrative response is logged. The attack that caused it is not.

`Account Lockout` set to Success only is the same problem in miniature, since every event that
subcategory produces is a failure by definition.

**Auditing is not retroactive.** Events that were not being audited when they occurred cannot be
recovered afterward. Verifying the audit configuration before generating evidence is the only
ordering that works, and getting it backwards costs the entire lab.

#### Part 3: Triggering the lockout

Two messages, same account, same machine, seconds apart.

![The password is incorrect. Try again.](docs/images/ext-lockout-08-client1-bad-password-error.png)

![The referenced account is currently locked out and may not be logged on to.](docs/images/ext-lockout-09-client1-locked-out-error.png)

With the threshold at 5, the account locks **on** the fifth failure, but the fifth attempt still
returns the ordinary bad-password message, because the lock is applied as a result of that attempt
rather than before it. The locked-out message appears on the sixth. Stopping at five and concluding
the policy is broken is an easy mistake to make.

Two behaviors worth knowing before testing this. Since Server 2003, domain controllers do **not**
increment the bad password counter when the submitted password matches one of the account's two most
recent previous passwords, so a "wrong" password that the account previously held is a free attempt
and the counter never moves. And any background process still holding a stale credential contributes
to the same counter, which can fire the lockout earlier than expected.

#### Part 4: The unlock

![The unlock checkbox on the Account tab, reading This account is currently locked out on this Active Directory Domain Controller](docs/images/ext-lockout-10-dc1-aduc-unlock-checkbox.png)

That extended wording only renders while the account is actually locked. Once unlocked, the label
shortens to `Unlock account` and the checkbox proves nothing, which makes this a capture with a
window that closes.

The wording is worth reading closely for a second reason. It says **this** domain controller. The
bad password counter is maintained per DC and does not replicate, which is invisible in a single-DC
lab and is exactly why real lockout investigations start at the PDC Emulator.

Finding the account matters as much as fixing it. With roughly a thousand users in `_USERS`,
right-click the domain node and use **Find** rather than browsing.

#### Part 5: Password reset

![Reset Password dialog](docs/images/ext-lockout-11-dc1-reset-password-dialog.png)

![The user's password must be changed before signing in](docs/images/ext-lockout-12-client1-forced-password-change.png)

The rejection message on a failed reset, *"the password does not meet the length, complexity or
history requirements of the domain"*, covers three separate policies and does not say which one
fired. In this domain it is almost always history: the last 24 passwords are remembered, so resetting
a provisioned account back to its original password is refused.

There is a subtler one underneath it. Minimum password age is one day, and an administrator reset in
ADUC bypasses it while a user-initiated change does not. So the forced change at logon works, and a
second change the same day fails with the same misleading message about complexity.

#### Part 6: Disabled is not locked

![The disabled account in ADUC](docs/images/ext-lockout-13-dc1-aduc-account-disabled.png)

![Your account has been disabled. Please see your system administrator.](docs/images/ext-lockout-14-client1-account-disabled-error.png)

Same user, same machine, correct password, and a completely different message from the lockout in
Part 3. Locked is a transient state produced by failed authentication that clears itself after the
duration expires. Disabled is a deliberate administrative action that clears only when an
administrator reverses it. They produce different messages, different event IDs, and different
failure codes.

#### Part 7: Finding all of it in the logs

The source video searches the Security log by username, surfaces a run of logon and logoff events,
says on camera *"I don't actually know where they're going to show up"*, and never finds the lockout
at all. Filtering by event ID instead of searching by text returns it immediately.

![Event 4740, a user account was locked out, with Caller Computer Name CLIENT-1](docs/images/ext-lockout-15-dc1-event-4740.png)

Event **4740** exists on `DC-1` and only on `DC-1`. **Number of events: 1**, so the policy fired
exactly once. The Subject is `SYSTEM` / `DC-1$` rather than an administrator, because the domain
controller locked the account on its own. And **Caller Computer Name** names the machine the bad
passwords came from, which on a real network is the entire investigation: it separates a user
fat-fingering their password at their desk from credentials being sprayed from somewhere that should
not have them.

![Event 4771, Kerberos pre-authentication failed, failure code 0x18](docs/images/ext-lockout-16-dc1-event-4771.png)

This is why the video came up empty. Most people go looking for 4625, "an account failed to log on."
On a domain controller authenticating a domain-joined Windows client, the failure surfaces as
**4771**, because the client uses Kerberos and the failure happens at pre-authentication. Failure
code `0x18` means the pre-authentication data was bad, which for a password logon means the password
was wrong. The events were there the whole time under an event ID nobody was looking for.

![Filtered list of 4724, 4725, 4722, 4767, and 4738 events](docs/images/ext-lockout-17-dc1-account-management-events.png)

Read top to bottom, this is the complete administrative narrative of the lab in reverse: unlock,
reset, disable, enable, each accompanied by a 4738 recording that the account object changed.

Every one of these records **Subject: Account Name** as `Keenan_Admin` rather than `Administrator`.
That is the audit trail the core build's separate named admin account was created for, and this is
where that argument stops being theoretical. Note the contrast with 4740 above, where the subject
was `SYSTEM`: the domain controller locked the account, a named human unlocked it.

![Event 4625 on CLIENT-1 with Sub Status 0xC000006A and Logon Type 2](docs/images/ext-lockout-18-client1-event-4625.png)

On the client side, **do not go looking for a 4625 marked locked out or disabled. There is not one.**
In a Kerberos domain the client requests a ticket from the domain controller before attempting any
local logon session. A locked or disabled account is refused at that point, so the workstation never
reaches a stage where it would log an interesting failure. Every 4625 on the client is an ordinary
bad password or an unknown username.

The sub-status is what carries the meaning. `0xC000006A` means the username was valid and the
password was wrong. `0xC0000064` means the username itself does not exist. On a real network that
difference separates password guessing against a known account from guessing at account names.

**Logon Type 2** is worth noting against the source material, which shows Type 10 and reads a source
IP address out of these events. That version connects to the client over RDP. This one is a local
console logon, so the source address is loopback. Knowing which logon type corresponds to which
access method is the transferable part: 2 is interactive at the console, 3 is network, 10 is
RemoteInteractive over RDP.

#### Lab-only shortcuts in this lab

| Shortcut | Why it is fine here | Why it is wrong in production |
|---|---|---|
| One domain-wide lockout policy | Single test account, one client | Real environments need Fine-Grained Password Policies so service and privileged accounts get different thresholds than staff |
| Threshold of 5 with a 30 minute duration | Fast to trigger, self-clearing for a demo | Aggressive lockout is itself a denial of service vector an attacker can weaponize |
| No monitoring or alerting on 4740 | Manual log review is the exercise | A lockout event nobody sees is not a control. Production forwards these to a SIEM with alerting on volume and on caller computer name anomalies |
| Audit subcategory set with `auditpol` | Nothing in this domain defines it in GPO | Audit policy belongs in a GPO under Advanced Audit Policy Configuration, or the next policy refresh silently reverts it |
| Password reset typed into ADUC by hand | One account | Self-service reset, or a ticketed process with identity verification. Help desk resets over an unverified phone call are a well-worn social engineering path |
| Provisioned accounts carry `Password never expires` | Filler accounts from the bulk script | A credential that never expires and never rotates turns one leaked password into permanent access |
| Single domain controller | Nothing to replicate with | The bad password counter is per-DC and does not replicate, which is why real investigations start at the PDC Emulator |

---

### 4. Network Traffic Analysis and Windows Firewall

**Runbook:** [`extensions/04-network-traffic-and-firewall.md`](extensions/04-network-traffic-and-firewall.md)

Five protocols watched live on the wire, and a firewall rule that stops one of them cold. What each
protocol exposes to anyone listening, and what the difference is between traffic that is blocked and
traffic that was never sent.

The source exercise runs on Azure and uses a Network Security Group as its firewall. There is no
on-premises equivalent, so this uses Windows Defender Firewall instead, and that substitution turned
out to produce better evidence than the original.

#### Setup

Two Windows features and a logging change, no new machines. The source builds a throwaway Ubuntu VM
to have something to ping and SSH into. `DC-1` already answers pings, and Windows Server 2022 ships
an OpenSSH Server feature.

![OpenSSH Server installed and running on DC-1 with its inbound firewall rule](docs/images/ext-net-01-openssh-server-installed.png)

![Windows Defender Firewall logging enabled for dropped packets](docs/images/ext-net-02-firewall-logging-enabled.png)

That second one has no equivalent in the source material and it is what makes the firewall section
work. Windows Defender Firewall can write every dropped packet to a log file, and it is off by
default.

#### ICMP, and what a working request looks like

![Four ICMP echo request and reply pairs between CLIENT-1 and DC-1](docs/images/ext-net-03-icmp-request-reply.png)

Four requests out, four replies back. The detail worth holding onto is in the Info column:
Wireshark annotates each request with `(reply in 184)` and each reply with `(request in 183)`,
because it matches them by identifier and sequence number. When that pairing works, it says so.

![ICMP to a public address, routed through the NAT gateway](docs/images/ext-net-04-icmp-internet.png)

Pinging a public host produces packets addressed to the internet from a client that has no route to
the internet of its own. They go to `DC-1`, which is the default gateway, and RRAS translates them on
the way out. The capture shows the client's version of the story, not the whole path.

#### The block, and the thing the cloud version cannot show

![The Block rule for ICMPv4 Echo Request in DC-1's inbound rules](docs/images/ext-net-05-firewall-block-rule.png)

A **Block** rule rather than disabling the existing Allow rule, deliberately. Both rules now exist
and both match the same traffic, and the block wins, because Windows Firewall evaluates block rules
first. That precedence is the answer to "there is an allow rule for this, why is it still being
denied," which is a real troubleshooting question.

![Ping showing replies then Request timed out, with Wireshark showing requests and no replies](docs/images/ext-net-06-ping-timeout.png)

Same view as before, one column different. Every request now reads `(no response found!)` where it
previously read `(reply in 184)`.

**And this is the limit of a one-sided capture.** The requests are still leaving `CLIENT-1`. Nothing
on the client changed. From here, "blocked by a firewall," "the host is powered off," and "the cable
is unplugged" produce an identical capture: outbound requests, silence. The sender cannot tell you
which one is happening.

![DC-1's pfirewall.log showing DROP entries for ICMP from 172.16.0.100](docs/images/ext-net-07-dc1-firewall-drop-log.png)

`DC-1` can. The drop log names the protocol, the source, and the decision.

This is where the substitution beats the original. An Azure Network Security Group filters at the
virtual network layer, **before** traffic reaches the VM's interface, so the VM never sees the packet
and has nothing to log. A host-based firewall decides **after** the packet has arrived, which means
the destination can state that it received the request and discarded it on purpose.

Put the two together and the question is answered. The client proves it sent something and heard
nothing. The server proves it received that something and refused it. Neither capture alone is
sufficient, which is the same two-sided evidence standard the DNS lab established in Extension Lab 1.

![Ping resuming after the block rule is removed](docs/images/ext-net-08-ping-resumes.png)

#### SSH: the content is opaque, the metadata is not

![SSH session into DC-1 with the encrypted packets that carried it](docs/images/ext-net-09-ssh-session.png)

`whoami` returns `smithlab\keenan_admin` and `hostname` returns `DC-1`. That is a shell on the domain
controller, authenticated as a domain account. Every packet that carried it reads `Encrypted packet`.
Not the commands, not the output.

Two things leak anyway.

The **opening exchange is in the clear.** Scroll to the top of the capture and the version banner
names the SSH implementation and version on both sides. Encryption starts after the key exchange,
not before it, which is how a scanner fingerprints a server before it tries anything.

The **timing is in the clear.** Read the timestamps: client packet, server packet, microseconds
apart, over and over, one pair per keystroke, because SSH echoes each character back from the remote
side as you type. An observer learns which hosts are talking, for how long, in which direction, and
roughly how much was typed and where the pauses fell. Traffic analysis is built on exactly this, and
it never decrypts a byte.

#### DHCP: the full DORA

![DHCP Release, Discover, Offer, Request, and ACK captured in sequence](docs/images/ext-net-10-dhcp-dora.png)

The source runs `ipconfig /renew` alone, which produces a two-packet exchange, because a renewal is
a unicast conversation with a server the client already knows. Releasing first forces the whole
thing: **D**iscover, **O**ffer, **R**equest, **A**cknowledge.

Expand the ACK and the scope options are sitting there on the wire: Router, Domain Name Server, and
Domain Name. Those are the exact values configured on the DHCP scope during the core build, arriving
at the client.

#### DNS: every name you look up, in plain text

![DNS queries and responses with disney.com and its address readable](docs/images/ext-net-11-dns-cleartext.png)

Put this next to the SSH capture. That session was encrypted so thoroughly the commands were
unrecoverable. The DNS lookup that found the server in the first place went out in the clear. Anyone
on the path sees every name a client resolves, no matter how well the connections that follow are
protected. Content and destination are two separate exposures, and encrypting one does nothing for
the other. That gap is the reason DNS over HTTPS and DNS over TLS exist.

Two smaller things visible in the same capture.

The queries for `google.com.smithlab.local` returning `No such name` are not an error. Windows
appends the machine's DNS suffix and tries the qualified name first, fails, then falls back to the
bare name. Four wasted round trips per lookup, and a real contributor to "DNS feels slow" on domain
networks.

`nslookup` reporting `Server: UnKnown` is Extension Lab 1's finding surfacing from another angle.
nslookup tries a reverse lookup on `172.16.0.1` to name the server, and the `PTR` query for
`1.0.16.172.in-addr.arpa` is visible in the capture, failing. There is no reverse lookup zone in this
domain. That is listed in the lab-only shortcuts above, and this is what the consequence looks like.

#### RDP: a protocol that never stops talking

![Remote Desktop session into DC-1 from CLIENT-1](docs/images/ext-net-12-rdp-session.png)

The Server 2022 watermark inside a `CLIENT-1` window is the proof of what is connected to what.

![Port 3389 traffic continuing with the RDP session idle](docs/images/ext-net-13-rdp-continuous.png)

Read that capture from the top and it is a complete connection lifecycle: the old session being torn
down with `[RST, ACK]`, a fresh TCP handshake, two packets Wireshark dissects as **RDP**, a TLS 1.2
handshake, and then `Application Data` forever.

**Only those two packets are dissected as RDP.** Everything after the TLS handshake is opaque
application data and the RDP dissector never engages, which is why the filter has to be
`tcp.port == 3389` rather than `rdp`. Filtering on the protocol name returns two packets and looks
like a failure.

One of those two carries `Cookie: mstshash=keenan_ad`. **That is the username, truncated, sent
before the TLS handshake begins.** Same pattern as the SSH version banner: the negotiation that sets
up the encryption necessarily happens before the encryption exists, and it gives away who is
connecting.

Then the point of the section: that traffic does not stop. Every other protocol here produced packets
in response to an action. RDP is a live video stream of one machine's screen, so a blinking cursor
is a screen change and a clock updating is a screen change. An idle session consumes bandwidth for as
long as it stays open, which is why an abandoned RDP connection stays visible in traffic monitoring
long after the person walked away.

#### Lab-only shortcuts in this lab

| Shortcut | Why it is fine here | Why it is wrong in production |
|---|---|---|
| OpenSSH Server installed on a domain controller | Cheapest way to generate real SSH traffic without a Linux VM | A DC should run the minimum services required to be a DC. Every additional listener is attack surface on the most privileged host in the environment |
| Remote Desktop enabled on a domain controller | Needed to generate RDP traffic to capture | DC administration belongs on a privileged access workstation behind a jump host, not direct RDP from a general-purpose client |
| Domain admin credentials used for SSH and RDP | One admin account exists in this lab | Interactive admin logons to a DC expose those credentials to anything already resident on the connecting workstation |
| Firewall rule applied to all three profiles | Simpler than determining the active profile mid-lab | Rules should be scoped to the profile and source addresses they are meant for |
| Wireshark installed on a domain-joined client | Isolated lab | Capture tooling on general-purpose endpoints is itself a risk, and capture files routinely contain credentials and session tokens |
| Firewall drops logged to a local text file | Answers the question during the exercise | `pfirewall.log` rotates at 4 MB and is deleted by anyone with local admin. It is a troubleshooting aid, not an audit trail. Forward it |
| RDP certificate warning clicked through | No PKI here, nothing is real | Clicking through certificate warnings is the exact habit that makes machine-in-the-middle attacks work |

---

### Coming next

- **osTicket.** A ticketing system build in its own repository, moving from infrastructure into the
  service desk workflows that sit on top of it.

---

## Repo structure

```
active-directory-project/
├── README.md
├── RUNBOOK.md             Step-by-step build instructions for the core domain
├── BUILD-LOG.md           Every problem hit during the build and what fixed it
├── extensions/
│   ├── 01-dns-records-and-cache.md
│   ├── 02-file-shares-and-permissions.md
│   ├── 03-account-lockout-and-passwords.md
│   └── 04-network-traffic-and-firewall.md
└── docs/
    └── images/            Build and verification screenshots
```

`BUILD-LOG.md` is the part I would read first. The runbook describes the build that worked. The
build log describes the build that actually happened, including the wrong turns, and the entries
are written so someone repeating this does not lose the same time I did.

---

## Credits

This lab is built on Josh Madakor's Active Directory tutorial:
https://youtu.be/MHsI8hJmggI

The architecture, the topology, and the build order are his. So is the `AD_PS`
PowerShell script used to provision the bulk user accounts.

What I added is a rebuild on current software (Server 2022, Windows 11 Enterprise,
and VirtualBox 7 in place of Server 2019, Windows 10, and an older VirtualBox),
documentation of the places where that shift changes the actual steps, and a build
log of everything that broke along the way.

He put this lab out for people to take somewhere. This is where I am taking it: the
on-prem identity foundation for a series that ends in privileged access management.

---

## AI authorship disclosure

I used Claude as a working partner on this project. Being specific about where:

| Component | Who did it |
|---|---|
| Lab architecture and design decisions | Me, with Claude as a sounding board |
| VM builds, OS installs, all hands-on configuration | Me |
| Troubleshooting during the build | Me, working through it with Claude |
| Runbook drafting | Claude, from my direction, corrected by me where it was wrong |
| README and documentation drafting | Claude, from my notes and screenshots |
| Verification that any of it actually works | Me |

The runbook was rewritten mid-build because its original phase ordering was wrong: it had the
client being built before the DHCP server existed. I caught that by comparing it against the
source material and stopped rather than proceeding. That correction is in `BUILD-LOG.md`.

I am disclosing this because I think it is the honest thing to do, and because how someone uses
these tools is more informative than whether they used them.
