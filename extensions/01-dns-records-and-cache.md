# Extension Lab 1: DNS Records and the Local Resolver Cache

Building an intuition for how a Windows client actually resolves a hostname, and why a stale
cache is one of the most common causes of "it works for everyone but me."

This lab extends the domain built in the main runbook. It requires `DC-1` and `CLIENT-1` to
already exist and be joined. Nothing new is installed.

**Source material:** this lab follows the structure of Josh Madakor's "Building an Intuition for
DNS" exercise, taught in the CourseCareers IT program (module 4.6.2). That version is built on
Azure virtual machines. What follows is the on-premises translation, rebuilt and verified on
VirtualBox against a domain controller with two network adapters, which is where it stops being a
straight port and starts producing problems the cloud version does not have.

> **On the timestamps below.** Each step carries a reference like `Video: 08:32`, pointing at the
> matching moment in the CourseCareers module. **That course is paid, and you do not need it to
> follow this runbook.** Every step here is written to stand on its own: the commands, the
> expected output, the failure modes, and the reasoning are all in this document. The timestamps
> are there for people already enrolled who want to see a step demonstrated, and for my own
> reference when revisiting this later. If you are not enrolled, ignore them and nothing is lost.

**What is mine versus what is his.** The exercise structure and the teaching sequence are his.
The on-premises translation, the environment mapping, the multihomed domain controller findings,
the failure analysis, and every capture in this document are mine, and none of them exist in the
original.

---

## Environment translation

| CourseCareers Azure version | This build |
| --- | --- |
| Azure Portal, start VMs | VirtualBox Manager, start `DC-1` then `CLIENT-1` |
| Remote Desktop to a public IP | VirtualBox console window for each VM |
| `mydomain.com` | `smithlab.local` |
| `mydomain.com\jane_admin` | `SMITHLAB\Keenan_Admin` |
| DC-1 private IP `10.0.0.4` | DC-1 `Internal` adapter, `172.16.0.1` |
| (no equivalent) | DC-1 `Internet` NAT adapter, `10.0.2.15` |
| Client private IP | CLIENT-1, `172.16.0.100` via DHCP |
| Azure VNet | VirtualBox internal network, `intnet` |
| Client is Windows 10 | Client is Windows 11 Enterprise |

The DNS work itself is identical. Once you are inside the OS, nothing in this lab is
cloud-specific. That is worth saying out loud in the README, because it is the honest answer to
"why does an Azure course translate to a local hypervisor without loss."

---

## Before you start

Start `DC-1` first and let it come up fully before starting `CLIENT-1`. The client depends on
`DC-1` for both DHCP and DNS, and starting them together occasionally produces a client that
boots before the DHCP service is listening.

Log into both machines as `SMITHLAB\Keenan_Admin`.

> **Why an admin account on the client for this lab.** Step 9 runs `ipconfig /flushdns`, which
> requires elevation. Josh hits this live in the video and has to reopen PowerShell as
> administrator. Logging in as a privileged account from the start avoids it. Note that the
> **next** lab (file shares) deliberately requires a *normal* domain user on `CLIENT-1`, so do
> not carry this habit forward.

> **Host resource warning.** This machine is a 4-core, 8-thread i5-10300H. Two running guests
> plus video playback on the host is enough to stall a VM. Pause the video while you are working
> inside the guests and scrub back to it between steps. `CLIENT-1` froze once during this build
> for exactly this reason.

---

## Screenshot numbering note

Captures `01` through `03` were taken in build order. Everything from `04` onward is numbered in
**step order**, not capture order, because two Step 2a captures were added retroactively after
the step was already complete. If you are following this runbook fresh, the numbers and the steps
line up exactly.

This supersedes any earlier numbering. The table at the bottom is the single source of truth.

---

## Part 1: The A record

### Step 1. Confirm the failure state first

> Video: `01:49` through `02:20`

On `CLIENT-1`, open PowerShell and run:

```
ping mainframe
nslookup mainframe
```

Both fail. That is the point. `mainframe` is an arbitrary name with no record anywhere, chosen
to be obviously fake.

> **Expect `Server: UnKnown`.** `nslookup` will report `Server: UnKnown` alongside
> `Address: 172.16.0.1`. That is not a failure. It found the DNS server fine, it just cannot
> resolve that server's *name*, because naming it requires a reverse lookup zone and this build
> has none. You will see `UnKnown` on every `nslookup` in this lab. Same root cause as the PTR
> checkbox you skip in Step 4.

**CAPTURE: `ext-dns-01-ping-fails.png`** with `whoami` visible in the same frame if possible, so
the capture proves both the failure and the account context in one image.

### Step 2. Understand the resolution order

> Video: `02:52` for the order itself, `03:23` for why that order (memory beats disk beats
> network), `03:53` for `displaydns`

Before creating anything, look at what the client checked and in what order:

1. **Local DNS cache** (memory, fastest)
2. **Local hosts file** at `C:\Windows\System32\drivers\etc\hosts` (disk, slower)
3. **DNS server** over the network (slowest)

View the cache:

```
ipconfig /displaydns
```

Long output, and none of it mentions `mainframe`. To read it properly, pipe it to a file:

```
ipconfig /displaydns > $env:USERPROFILE\Desktop\dns-before.txt
```

No capture needed here. The interesting version of this output comes in Step 8.

### Step 2a. Prove the hosts file sits in the middle

> Video: `04:55` opens the hosts file, `05:25` adds the `zebra` line, `05:59` shows it resolving
> without touching DNS

This is in the video but not the checklist. Do it. It takes two minutes and it is the clearest
demonstration in the lab of why a cache flush sometimes fixes nothing, which pays off in Step 9.

Open Notepad **as administrator**, then File > Open, set the file type filter to All Files, and
open `C:\Windows\System32\drivers\etc\hosts`.

Add this line at the bottom:

```
127.0.0.1    zebra
```

Save as plain text. Confirm three things before moving on: the line is on its own line, the file
is still named `hosts` with no `.txt` appended, and line endings are Windows CRLF.

Then run:

```
ping zebra
```

It resolves to the loopback address without ever touching the DNS server. Before you added the
line, `ping zebra` failed exactly the way `ping mainframe` did.

**CAPTURE: `ext-dns-04-hosts-file-zebra.png`** showing the file contents with the `zebra` line
visible.

**CAPTURE: `ext-dns-05-ping-zebra.png`** showing `ping zebra` resolving to 127.0.0.1. This is the
more valuable of the two. File contents show intent, the ping shows resolution actually
happening.

> Leave the `zebra` entry in place. It reappears in Step 9 as the contrast that makes the whole
> lab land.

### Step 3. Find DC-1's IP

> Video: `08:01`

On `DC-1`, open PowerShell:

```
ipconfig
```

You will see **two** IPv4 addresses, because DC-1 is multihomed:

- `Ethernet adapter Internet`, `10.0.2.15`. VirtualBox's NAT adapter, for outbound internet only.
  `CLIENT-1` cannot reach this address. Every NAT-attached VirtualBox guest on earth gets
  `10.0.2.15`.
- `Ethernet adapter Internal`, `172.16.0.1`. The `intnet` lab LAN. This is the DHCP gateway, the
  DNS server address CLIENT-1 already uses, and the address you want.

**Use `172.16.0.1`.** The test that settles it: the A record must hold an address that is
meaningful from CLIENT-1's point of view, not DC-1's. Only one of the two is on a network
CLIENT-1 can see.

Note also that the Internal adapter has no default gateway, and that is correct. DC-1 *is* the
gateway for that subnet. A machine does not route through itself.

No capture needed.

### Step 4. Create the A record

> Video: `06:59` opens DNS Manager, `07:31` walks the forward lookup zone and the auto-created
> records, `08:32` creates the record and hits the PTR problem

On `DC-1`, open **DNS Manager** (Server Manager > Tools > DNS, or run `dnsmgmt.msc`).

Expand `DC-1` > **Forward Lookup Zones** > `smithlab.local`.

You will see existing A records for `DC-1` and `CLIENT-1` created automatically when each machine
joined the domain. Nothing created those by hand. Note them, they matter in Step 5a.

Right-click in the right-hand pane > **New Host (A or AAAA)**.

- Name: `mainframe`
- IP address: `172.16.0.1`
- Leave **Create associated pointer (PTR) record** unchecked

Click **Add Host**.

> **PITFALL.** If you tick the PTR checkbox it will fail, because this build has no reverse
> lookup zone. Josh hits exactly this in the video and moves on. It is not an error in your
> configuration and it does not affect anything in this lab.

**CAPTURE: `ext-dns-02-a-record-created.png`** showing the `mainframe` record in DNS Manager with
the zone tree visible on the left.

### Step 5. Verify

> Video: `09:03` for the successful ping, `09:35` for the resolution-order recap

Back on `CLIENT-1`:

```
ping mainframe
```

It resolves to `172.16.0.1` and replies. Note that the output shows
`Pinging mainframe.smithlab.local`, which confirms the client is appending the domain suffix
automatically.

The client checked its cache (empty), checked the hosts file (no `mainframe` entry), and reached
the DNS server, which answered.

**CAPTURE: `ext-dns-03-ping-succeeds.png`**

### Step 5a. Clean up the multihomed DC registration

> Not in the video or the checklist. This is a finding specific to this build.

Look back at the DNS Manager record list from Step 4. `dc-1` has **two** A records, one for
`172.16.0.1` and one for `10.0.2.15`. The `(same as parent folder)` entries are duplicated the
same way. DC-1 registered both NICs in DNS, including the NAT adapter no client can reach.

This is harmless right now and will break the next lab. When you type `\\dc-1` from CLIENT-1, DNS
returns both addresses in round-robin order. Roughly half the time it gets `172.16.0.1` and
works. The other half it gets `10.0.2.15` and hangs until it times out. Intermittent failures
like that are miserable to diagnose because the obvious test passes.

On `DC-1`:

1. Open Network Connections (`ncpa.cpl`), right-click the **Internet** adapter > **Properties**.
2. Select **Internet Protocol Version 4 (TCP/IPv4)** > **Properties** > **Advanced** > **DNS**
   tab.
3. Uncheck **Register this connection's addresses in DNS**. OK out of all three dialogs.
4. In DNS Manager, delete both A records pointing at `10.0.2.15`: the `dc-1` one and the
   `(same as parent folder)` one.
5. Press F5 to refresh and confirm only `172.16.0.1` entries remain.

This is the correct production answer for a multihomed domain controller, not a lab shortcut. A
DC should only register the interface clients actually use.

**CAPTURE: `ext-dns-06-multihomed-dns-cleanup.png`** showing the cleaned record list. If you can
get the unchecked "Register this connection's addresses in DNS" box in a second frame, that is
better evidence, but one capture is enough.

> This one is worth a build log entry as a **finding**, not a failure. "Found and fixed a
> multihomed DC registering an unreachable NIC in DNS" is a stronger line than anything else in
> this lab.

---

## Part 2: The stale cache

This is the part of the lab that is actually about help desk work.

### Step 6. Change the record

> Video: `10:05`

> **Order matters, and so does the clock.** Steps 5 through 7 must be done consecutively, in one
> sitting, with no reboot in between. The demonstration depends on `CLIENT-1` holding a cached
> answer, and two things destroy that cache: the record's TTL expiring, and the machine
> restarting. Records in an AD-integrated zone default to a one-hour TTL, and the DNS client
> cache is memory-resident so it does not survive a reboot. If you take a break here, the client
> will have nothing cached, will query DC-1 fresh, and will correctly return the new address.
> The lab then appears broken when it is actually working. This happened during this build and is
> logged as an entry in `BUILD-LOG.md`.

> If you have stepped away, reset cleanly: change the record back to `172.16.0.1`, run
> `ipconfig /flushdns` on CLIENT-1, `ping mainframe` to repopulate the cache, then immediately
> change the record to `8.8.8.8` and continue.

On `DC-1`, in DNS Manager, double-click the `mainframe` record and change the address to
`8.8.8.8`. Click OK.

That address belongs to Google's public DNS resolver. It is used here only because it is a
recognizable address that is obviously not DC-1.

No capture needed. The next step is the evidence.

### Step 7. Observe the stale answer

> Video: `10:37` for the ping, `11:11` for the explanation and the real-world help desk framing

On `CLIENT-1`:

```
ping mainframe
```

The address it returns will not match what the server currently holds. The client never asked
the DNS server, because it found an answer in its own cache first.

> **Which direction the mismatch runs does not matter.** Whether the client returns the old
> address while the server holds the new one, or the reverse, the lesson is identical: the client
> answered from cache instead of from the authoritative source. What proves the point is the
> mismatch itself, plus what happens after a flush.

> `CLIENT-1` reaches the internet through DC-1's RRAS NAT, so pings to `8.8.8.8` may actually
> succeed. Ignore whether the ping replies. Watch the **address on the first line of the ping
> output**, which is what the name resolved to.

**CAPTURE: `ext-dns-07-stale-cache-ping.png`**

Capture the whole sequence in one frame rather than a single ping: the mismatched answer, the
`ipconfig /flushdns` that follows, and the corrected answer after it. Three commands in one
image is stronger documentation than three separate captures, because a reader can see cause and
effect without cross-referencing filenames.

**CAPTURE: `ext-dns-07b-record-changed.png`**

A DNS Manager shot of the `mainframe` record showing what the server actually holds. This exists
because the client-side capture alone cannot prove what the server said at that moment, and the
mismatch is the entire point. Server-side and client-side evidence together, or the sequence is
unverifiable to anyone reading the repo.

### Step 8. Look at the cache directly

> Video: `12:14`

```
ipconfig /displaydns > $env:USERPROFILE\Desktop\dns-cached.txt
notepad $env:USERPROFILE\Desktop\dns-cached.txt
```

Find the `mainframe` entry holding `172.16.0.1`. The `zebra` entry from Step 2a is in here too.

**CAPTURE: `ext-dns-08-displaydns-stale-entry.png`** with the `mainframe` record and its stale
address visible in the text.

### Step 9. Flush and re-check

> Video: `12:45` is where he runs it without elevation and gets the error, `13:17` is the retry as
> administrator, `13:48` is where `zebra` survives the flush

```
ipconfig /flushdns
ipconfig /displaydns
```

The cache is now essentially empty. **But `zebra` is still there.**

That is not a bug and it is the most important observation in the lab. `zebra` comes from the
hosts file, which is read from disk on every lookup and is not part of the cache. Flushing DNS
will not fix a machine with a bad hosts file entry, and that is a real troubleshooting scenario
that costs people hours.

**CAPTURE: `ext-dns-09-flush-zebra-survives.png`** showing the post-flush `displaydns` output
with `zebra` present and `mainframe` gone. This is the single best image in the lab and it was
missing from the first version of this runbook.

### Step 10. Re-resolve

> Video: `14:18`

```
ping mainframe
```

Now it resolves to `8.8.8.8`. Cache was empty, hosts file had no entry, so the client went to
the DNS server and got the current answer.

**CAPTURE: `ext-dns-10-flushed-and-resolved.png`**

**The help desk translation:** one user cannot reach a resource that everyone around them can
reach. The resource's IP changed, DNS was updated correctly, but that one machine is still
holding the old answer in cache, or has a stale hosts file entry from something someone did
months ago. `ipconfig /flushdns` fixes the first. Only reading the hosts file fixes the second.

---

## Part 3: The CNAME record

### Step 11. Create the alias

> Video: `14:51` introduces CNAME, `15:23` creates it

On `DC-1`, in DNS Manager, right-click inside `smithlab.local` > **New Alias (CNAME)**.

- Alias name: `search`
- Fully qualified domain name for target host: `www.google.com`

Click OK.

A CNAME maps one name to another name, where an A record maps a name to an address.

**CAPTURE: `ext-dns-11-cname-created.png`**

### Step 12. Verify

> Video: `15:54` for the ping, `16:29` for `nslookup`

On `CLIENT-1`:

```
ping search
nslookup search
```

`nslookup` shows the alias resolving through to `www.google.com` along with its IPv4 and IPv6
addresses.

**CAPTURE: `ext-dns-12-cname-nslookup.png`**

> **PITFALL specific to this build.** This requires DC-1's DNS server to resolve external names,
> which it does through the forwarders configured automatically during the DNS role install. If
> `search` fails to resolve, open DNS Manager, right-click `DC-1` > **Properties** >
> **Forwarders** and confirm entries exist. Josh's Azure lab never hits this because Azure
> supplies DNS resolution to the VNet by default. This is one of the few places where the
> on-prem translation actually differs.

### Step 13. Try it in a browser

> Video: `16:59` attempts it, `17:30` explains the certificate name mismatch

Browse to `http://search`. It fails with a certificate name mismatch. The address resolved
correctly, but the certificate Google presents is issued for `google.com`, not `search`, so the
browser refuses.

That failure is worth understanding rather than working around. DNS resolution and certificate
validation are separate layers, and satisfying one does not satisfy the other.

**CAPTURE: `ext-dns-13-cname-browser-cert-error.png`** showing the certificate warning.

---

## Cleanup

Leave the `mainframe` and `search` records in place. Leave both VMs built. The next lab uses the
same domain.

If you are stopping here, shut both VMs down from **inside Windows** using Start > Shut down, not
VirtualBox's Power Off. DC-1 is a domain controller and the directory database needs to close
cleanly.

---

## Complete screenshot list

Fourteen captures. This is the full set. Nothing else is expected of you.

Capture `07b` was added mid-build. It is not padding: the Step 7 mismatch cannot be verified
from a client-side screenshot alone, and the original runbook had no server-side evidence for it.

| # | File | Step | What it shows | Status |
| --- | --- | --- | --- | --- |
| 01 | `ext-dns-01-ping-fails.png` | 1 | `ping` and `nslookup` for `mainframe` both failing | done |
| 02 | `ext-dns-02-a-record-created.png` | 4 | The `mainframe` A record in DNS Manager | done |
| 03 | `ext-dns-03-ping-succeeds.png` | 5 | `ping mainframe` resolving to 172.16.0.1 | done |
| 04 | `ext-dns-04-hosts-file-zebra.png` | 2a | The hosts file with the `zebra` line | done |
| 05 | `ext-dns-05-ping-zebra.png` | 2a | `ping zebra` resolving to 127.0.0.1 | done |
| 06 | `ext-dns-06-multihomed-dns-cleanup.png` | 5a | DNS records after removing the 10.0.2.15 entries | done |
| 07 | `ext-dns-07-stale-cache-ping.png` | 7 | Client answering from cache, the flush, and the corrected answer | done |
| 07b | `ext-dns-07b-record-changed.png` | 7 | Server-side proof: the `mainframe` record at 8.8.8.8 | done |
| 08 | `ext-dns-08-displaydns-stale-entry.png` | 8 | The stale `mainframe` entry inside `displaydns` | to do |
| 09 | `ext-dns-09-flush-zebra-survives.png` | 9 | Post-flush cache: `zebra` present, `mainframe` gone | to do |
| 10 | `ext-dns-10-flushed-and-resolved.png` | 10 | Ping resolving to 8.8.8.8 after the flush | to do |
| 11 | `ext-dns-11-cname-created.png` | 11 | The `search` CNAME in DNS Manager | to do |
| 12 | `ext-dns-12-cname-nslookup.png` | 12 | `nslookup search` resolving through to Google | to do |
| 13 | `ext-dns-13-cname-browser-cert-error.png` | 13 | Certificate name mismatch in the browser | to do |

All of these go in `D:\ad-lab\docs\images\` alongside the existing 25 from the core build. The
`ext-` prefix keeps them clear of the `01` through `24` numbering already referenced by filename
in `RUNBOOK.md` and `README.md`.

If you capture something useful that is not on this list, save it with the next free number and
tell me. Extra evidence is easy to work in. Missing evidence means booting the VMs again.

---

## Build log

Snags from this lab continue in the existing `BUILD-LOG.md` in this repo, starting at **entry
16**. Same VMs, same domain, same record. The log does not restart for extension labs.

```
[STEP x] Short title
WHAT I WAS DOING:
WHAT HAPPENED:
WHAT I TRIED:
WHAT ACTUALLY FIXED IT:
TIME LOST:
SCREENSHOT:
```

Two entries are already earned from this lab: the `CLIENT-1` freeze during Step 2a, and the
multihomed DC registration found in Step 5a.

---

## A+ Core 2 objective coverage

- **1.2** Command-line tools: `ipconfig /displaydns`, `ipconfig /flushdns`, `ping`, `nslookup`
- **1.6** Windows networking features and name resolution
- **2.x** Name resolution as a troubleshooting surface

The stale-cache exercise in particular is the kind of scenario CompTIA wraps into a PBQ or a
situational question rather than asking about directly.
