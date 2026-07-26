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

**Group Policy is not covered.** A domain without GPOs is a domain that is not actually managing
anything. That is the next extension of this lab.

---

## Repo structure

```
active-directory-project/
├── README.md
├── RUNBOOK.md             Step-by-step build instructions
├── BUILD-LOG.md           Every problem hit during the build and what fixed it
└── docs/
    └── images/            Build and verification screenshots
```

`BUILD-LOG.md` is the part I would read first. The runbook describes the build that worked. The
build log describes the build that actually happened, including the wrong turns, and the entries
are written so someone repeating this does not lose the same time I did.

---

## Credits

Bulk user provisioning uses the `AD_PS` PowerShell script by Josh Madakor.

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
