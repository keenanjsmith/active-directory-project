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

A Windows client resolves a name in a fixed order. Local cache first, then the hosts file, then
the DNS server. The lab starts by proving the name does not exist anywhere.

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
seconds. Hosts file entries show 604800 seconds, which is seven days, because they are read from
disk on every lookup and were never part of the cache at all. The hosts file also generated a
matching reverse entry.

A machine with a bad hosts file entry will not be fixed by flushing DNS. Knowing that is the
difference between a five-minute fix and an hour of guessing.

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

### Coming next

- **File shares and NTFS permissions.** Share-level and NTFS permissions on the same folders, and
  what a normal domain user actually sees when the two disagree.
- **Account lockout, unlock, and password reset.** Group Policy lockout thresholds, reading the
  resulting events on both machines, and the disable and re-enable cycle.

---

## Repo structure

```
active-directory-project/
├── README.md
├── RUNBOOK.md             Step-by-step build instructions for the core domain
├── BUILD-LOG.md           Every problem hit during the build and what fixed it
├── extensions/
│   └── 01-dns-records-and-cache.md
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
