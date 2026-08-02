# Extension Lab 4: Network Traffic Analysis and Windows Firewall

Five protocols watched on the wire, and a firewall rule that stops one of them cold. What each
protocol looks like in a capture, which of them expose their contents to anyone listening, and what
the difference is between traffic that is blocked and traffic that is simply not there.

This lab extends the domain built in the main runbook. It requires `DC-1` and `CLIENT-1` to already
exist and be joined. Two Windows features get installed along the way, no new machines.

**Source material:** this lab follows the structure of Josh Madakor's "Performing Activities on the
Network" lab checklist, taught in the CourseCareers IT program. That version is built on Azure
virtual machines and uses a Network Security Group as its firewall. What follows is the
on-premises translation, rebuilt and verified on VirtualBox.

**What is mine versus what is his.** The exercise structure, the protocol sequence, and the teaching
points about RDP being a continuous stream are his. The on-premises translation, the Windows
Firewall substitution and its two-sided evidence, the full DORA sequence in Part 4, the
encrypted-versus-cleartext comparison across SSH and DNS, and every capture in this document are
mine.

---

## Where this lab diverges from the source, and why

The source checklist assumes two throwaway cloud VMs and an Azure Network Security Group. Neither
exists here. Four substitutions:

**1. No new virtual machines.** The source builds a Windows 10 VM and an Ubuntu VM purely to have
two hosts that can talk to each other. `DC-1` and `CLIENT-1` already do that. The Ubuntu box exists
only as a ping and SSH target, and Windows Server 2022 ships an OpenSSH Server feature that covers
the SSH half in about two minutes.

**2. Windows Defender Firewall replaces the Network Security Group.** An NSG is an Azure construct
with no on-premises equivalent. A host-based firewall is the closest analogue, and the difference
between the two is worth understanding rather than papering over. See Part 2.

**3. RDP has to be created rather than observed.** The source has constant RDP traffic because the
entire lab session runs over Remote Desktop, which is why filtering port 3389 produces nonstop
noise. You are working at the VirtualBox console, so there is no RDP traffic on your network at all
until you make some. Remote Desktop gets enabled on `DC-1` and connected to from `CLIENT-1`.

**4. DHCP is done properly.** The source runs `ipconfig /renew` alone, which produces a two-packet
exchange because a renewal is unicast straight to the server the client already knows. Releasing
first and then renewing produces the full four-packet DORA sequence, which is what the A+ objectives
actually ask about.

---

## Which machine, every time

`CLIENT-1` is the capture machine. Wireshark runs there and every observation happens there.
`DC-1` is the target: it answers the pings, hosts the SSH server, serves DHCP and DNS, accepts the
RDP session, and is where the firewall rule gets created.

The one exception is Part 2, where evidence is read on **both** machines. That is the point of that
part.

---

## Environment translation

| CourseCareers Azure version | This build |
| --- | --- |
| Windows 10 VM created for the lab | `CLIENT-1`, already built |
| Ubuntu VM created for the lab | `DC-1`, with OpenSSH Server added |
| Azure Virtual Network and subnet | VirtualBox internal network `intnet`, `172.16.0.0/24` |
| Remote Desktop into the Windows VM | VirtualBox console window |
| Ubuntu private IP address | `172.16.0.1`, the domain controller |
| `ssh labuser@<private IP>` | `ssh keenan_admin@172.16.0.1` |
| Network Security Group inbound rule | Windows Defender Firewall inbound rule on `DC-1` |
| RDP traffic already present from the lab session | Remote Desktop enabled on `DC-1`, connected from `CLIENT-1` |
| `ipconfig /renew` only | `ipconfig /release` then `/renew`, for the full DORA |
| Delete the resource group when finished | Nothing to delete, but revert the firewall rule |

---

## Before you start

**Start `DC-1` first** and let it reach the logon screen before starting `CLIENT-1`.

**Log into both machines as `SMITHLAB\Keenan_Admin`.** Unlike Lab 2, privilege does not change what
this lab demonstrates, and installing Wireshark plus reading firewall logs both require it.

**Snapshot both VMs.** In VirtualBox Manager with **both powered off**: select `DC-1`, click the
hamburger menu beside the machine name, **Snapshots**, **Take**, name it `Pre Ext Lab 4`, **OK**.
Repeat for `CLIENT-1` with the same name.

> **Never roll back `DC-1` alone.** Same rule as every lab in this series. Restore `DC-1` past
> `CLIENT-1`'s last computer account password rotation and the client throws *"The trust
> relationship between this workstation and the primary domain failed."* Roll both back to the same
> point or roll neither back.

**Confirm the two machines can still talk.** On `CLIENT-1`, right-click **Start** > **Terminal**:

```
ping 172.16.0.1
```

Four replies expected. If this fails, stop and fix connectivity before going further, because every
part of this lab depends on it.

---

## Part 0: Prep

Four installs and configuration changes, all of which need doing before any capture starts. Roughly
fifteen minutes.

### Step 1. Install Wireshark on CLIENT-1

**On `CLIENT-1`:**

1. Open **Edge**.
2. Go to `https://www.wireshark.org/download.html`.
3. Download the **Windows x64 Installer**.
4. Run it. Accept the license, keep the default components.
5. When the installer offers **Npcap**, install it. Wireshark cannot capture without it.
6. In the Npcap installer, leave the defaults. Accept **Install Npcap in WinPcap API-compatible
   mode** if offered.
7. Finish and reboot if prompted.

> `CLIENT-1` reaches the internet through `DC-1`'s NAT, which the base build already proved. If the
> download fails, that routing is broken and nothing else in this lab will work either. Check
> `ping 8.8.8.8` from `CLIENT-1` before troubleshooting Edge.

### Step 2. Install OpenSSH Server on DC-1

**On `DC-1`,** right-click **Start** > **Windows PowerShell (Admin)**:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

The tildes are not a typo. That is the literal capability name.

Start it and set it to survive a reboot:

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

Confirm it is listening and that the installer created its firewall rule:

```powershell
Get-Service sshd | Select-Object Name,Status,StartType
Get-NetFirewallRule -Name *OpenSSH* | Select-Object Name,DisplayName,Enabled,Direction,Action
```

Expected: `sshd` **Running** / **Automatic**, and an inbound rule named `OpenSSH-Server-In-TCP`
that is **True** for Enabled and **Allow** for Action.

**CAPTURE: `ext-net-01-openssh-server-installed.png`**
Both commands and their output in one frame.

> **Running an SSH server on a domain controller is not something you would do in production.** It
> is here because it is the cheapest way to generate real SSH traffic without building a Linux VM.
> It goes in the lab-only shortcuts table and it is an honest entry, not a throwaway.

### Step 3. Enable Remote Desktop on DC-1

**On `DC-1`,** in the same elevated PowerShell:

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

Verify:

```powershell
Get-NetFirewallRule -DisplayGroup "Remote Desktop" | Select-Object DisplayName,Enabled,Direction
```

The GUI equivalent, if you would rather see it: **Server Manager** > **Local Server** > click the
**Remote Desktop** value > **Allow remote connections to this computer** > **OK**.

Members of `Domain Admins` can connect to a domain controller by default, so `Keenan_Admin` needs
no additional rights.

### Step 4. Turn on firewall drop logging on DC-1

This is the step that makes Part 2 work, and the source has no equivalent.

**On `DC-1`:**

1. Click **Start**, type `wf.msc`, press **Enter**.
2. In the left pane, right-click **Windows Defender Firewall with Advanced Security on Local
   Computer** (the top node).
3. Click **Properties**.
4. Click the **Domain Profile** tab.
5. In the **Logging** section, click **Customize...**
6. Set **Log dropped packets** to **Yes**.
7. Note the **Name** field. Default is
   `%systemroot%\system32\LogFiles\Firewall\pfirewall.log`.
8. Click **OK**.
9. Repeat steps 4 through 8 for the **Private Profile** and **Public Profile** tabs.
10. Click **OK** to close Properties.

> **Do all three profiles even though `DC-1` is on the domain profile.** Firewall settings are
> per-profile, and confirming which profile is active is an extra step you do not need mid-lab. If
> you want to check anyway: `Get-NetConnectionProfile` on `DC-1`.

**CAPTURE: `ext-net-02-firewall-logging-enabled.png`**
The Customize Logging dialog with Log dropped packets set to Yes and the log path visible.

---

## Part 1: ICMP

> Source checklist: steps 9 through 12.

### Step 1. Start a capture

**On `CLIENT-1`:**

1. Open **Wireshark**.
2. On the welcome screen, find the interface with a live traffic sparkline beside it. `CLIENT-1`
   has one network adapter, usually listed as **Ethernet**.
3. Double-click it to start capturing.
4. In the **display filter** bar at the top, type:

```
icmp
```

5. Press **Enter**.

The filter bar turns green when the syntax is valid. The list clears, because nothing is pinging
anything yet.

> **Display filter, not capture filter.** The bar at the top of the packet list filters what you
> *see* and leaves everything captured underneath. A capture filter, set before you start, discards
> non-matching traffic permanently. Use display filters for this lab. Being able to change your mind
> about what you are looking at without restarting the capture is the whole point.

### Step 2. Ping the domain controller

**On `CLIENT-1`,** right-click **Start** > **Terminal**:

```
ping 172.16.0.1
```

Four replies. Switch back to Wireshark.

You should see eight packets: four **Echo (ping) request** from `CLIENT-1` to `172.16.0.1`, each
followed by an **Echo (ping) reply** coming back.

**CAPTURE: `ext-net-03-icmp-request-reply.png`**
The four request and reply pairs, with the Source and Destination columns legible.

Click any request and expand **Internet Control Message Protocol** in the detail pane below. Two
fields are worth reading:

- **Type: 8 (Echo (ping) request)** on the outbound, **Type: 0 (Echo (ping) reply)** on the return.
- **Identifier** and **Sequence Number**, which are how a reply gets matched to its request.

That request-and-reply pairing is the thing to hold onto. Part 2 breaks it deliberately.

### Step 3. Ping something on the internet

**On `CLIENT-1`,** in the same terminal:

```
ping www.google.com
```

Back in Wireshark, more ICMP appears, and the destination is a public address rather than
`172.16.0.1`.

**CAPTURE: `ext-net-04-icmp-internet.png`**
The public-destination ICMP, with the resolved address visible.

Worth noticing: those packets leave `CLIENT-1` addressed to a public host even though `CLIENT-1`
has no route to the internet of its own. They go to `DC-1`, which is the default gateway, and RRAS
translates them on the way out. From the client's point of view that is invisible. The capture shows
the client's version of the story, not the whole path.

---

## Part 2: The firewall block

> Source checklist: step 13. The source disables inbound ICMP on the Ubuntu VM's Network Security
> Group. This does it with Windows Defender Firewall on `DC-1`, and adds evidence the cloud version
> cannot produce.

### Step 1. Start a continuous ping

**On `CLIENT-1`,** in the terminal:

```
ping -t 172.16.0.1
```

`-t` runs until you stop it with Ctrl+C. Leave it running and leave the window visible. You want to
watch the exact moment the behavior changes.

Wireshark should be running with the `icmp` filter active.

### Step 2. Create the block rule on DC-1

**On `DC-1`,** in **Windows Defender Firewall with Advanced Security** (`wf.msc`):

1. In the left pane, click **Inbound Rules**.
2. In the right-hand **Actions** pane, click **New Rule...**
3. Rule Type: select **Custom**. Click **Next**.
4. Program: leave **All programs**. Click **Next**.
5. Protocol and Ports: in the **Protocol type** dropdown, select **ICMPv4**.
6. Click the **Customize...** button beside Internet Control Message Protocol settings.
7. Select **Specific ICMP types**, tick **Echo Request**, click **OK**.
8. Click **Next**.
9. Scope: leave both as **Any IP address**. Click **Next**.
10. Action: select **Block the connection**. Click **Next**.
11. Profile: leave all three ticked. Click **Next**.
12. Name: `Lab Block ICMPv4 Echo Request`. Click **Finish**.

The rule takes effect immediately. No reboot, no `gpupdate`.

**CAPTURE: `ext-net-05-firewall-block-rule.png`**
The new rule in the Inbound Rules list, with the Action column showing Block.

**PowerShell equivalent**, which is what you would use at scale:

```powershell
New-NetFirewallRule -DisplayName "Lab Block ICMPv4 Echo Request" -Direction Inbound -Protocol ICMPv4 -IcmpType 8 -Action Block
```

> **A Block rule was used instead of disabling the existing Allow rule, and the choice matters.**
> Ping works today because an inbound rule named `File and Printer Sharing (Echo Request -
> ICMPv4-In)` permits it. Disabling that rule would also stop the pings and would be closer to what
> the source does.
>
> Creating an explicit Block rule instead demonstrates something disabling cannot: **Windows
> Firewall evaluates block rules before allow rules.** Both rules now exist, both match the same
> traffic, and the block wins. That precedence is the answer to "there is an allow rule for this,
> why is it still being denied," which is a real troubleshooting question.

### Step 3. Watch it stop

**On `CLIENT-1`,** look at the still-running ping window. Replies stop and `Request timed out.`
begins repeating.

**CAPTURE: `ext-net-06-ping-timeout.png`**
Frame the Wireshark window and the terminal together in one shot. The terminal shows successful
replies above and timeouts below, and the packet list above it shows echo requests still going out
at one per second with nothing coming back. The transition is the evidence in both panes, so do not
clear the screen first.

Look at the **Info** column in Wireshark. Every request now reads `(no response found!)` where the
same column in Part 1 read `(reply in 184)`. Wireshark pairs requests with replies by identifier and
sequence number, so when the pairing fails it says so explicitly. Same view, one column different.

**This is the observation the lab is built around.** The requests are still leaving `CLIENT-1`.
Nothing on the client changed. From the client's side, "blocked" and "the host is switched off" and
"the cable is unplugged" produce an identical capture: outbound requests, silence. A capture on the
sender cannot tell you which of those is happening.

### Step 4. Prove the packets arrived and were dropped

Here is where the on-premises version does something the Azure version cannot.

**On `DC-1`,** open an elevated PowerShell:

```powershell
Get-Content C:\Windows\System32\LogFiles\Firewall\pfirewall.log -Tail 20
```

You should see lines ending in `DROP` with the protocol `ICMP`, the source address of `CLIENT-1`,
and the destination `172.16.0.1`.

**CAPTURE: `ext-net-07-dc1-firewall-drop-log.png`**
The DROP entries with source and destination legible.

> **If the log is empty or the file does not exist**, give it thirty seconds and retry. The firewall
> service buffers writes rather than flushing every packet. If it is still empty after a minute, go
> back to Part 0 Step 4 and confirm logging is enabled on the profile `DC-1` is actually using.
> `Get-NetConnectionProfile` tells you which one that is.

**Why this matters, and why it is the honest version of the NSG substitution.** An Azure Network
Security Group filters at the virtual network layer, before traffic reaches the VM's interface. The
VM never sees the packet, so there is nothing on the VM to log. A host-based firewall makes its
decision *after* the packet has arrived on the interface, which is why `DC-1` can state that it
received the request and discarded it deliberately.

Put the two captures side by side and the distinction is complete. The client can prove it sent
something and heard nothing back. The server can prove it received that something and refused it.
Neither capture alone answers the question. Together they do, and this is the same two-sided
evidence standard the DNS lab established.

### Step 5. Remove the rule and confirm recovery

**On `DC-1`,** in `wf.msc` > **Inbound Rules**, find `Lab Block ICMPv4 Echo Request`, right-click,
click **Delete**, confirm **Yes**.

**PowerShell equivalent:**

```powershell
Remove-NetFirewallRule -DisplayName "Lab Block ICMPv4 Echo Request"
```

**On `CLIENT-1`,** the still-running ping resumes replying within a second or two.

**CAPTURE: `ext-net-08-ping-resumes.png`**
Timeouts above, replies below, same window.

Press **Ctrl+C** to stop the ping.

---

## Part 3: SSH

> Source checklist: steps 14 through 18.

### Step 1. Filter for SSH

**On `CLIENT-1`,** in Wireshark's display filter bar, replace the filter with:

```
ssh
```

Press **Enter**. The list clears.

### Step 2. Connect

**On `CLIENT-1`,** in the terminal:

```
ssh keenan_admin@172.16.0.1
```

First connection prompts about the host's authenticity and fingerprint. Type `yes` and press
**Enter**. Then enter the password.

> **If the login is rejected**, the server may want the domain-qualified form. PowerShell treats a
> bare backslash as an escape, so it needs quoting:
>
> ```
> ssh "smithlab\keenan_admin@172.16.0.1"
> ```
>
> The alternate UPN-style form also works on most builds:
>
> ```
> ssh keenan_admin@smithlab.local@172.16.0.1
> ```

Once connected, the prompt changes to `keenan_admin@DC-1 C:\Users\keenan_admin>`. You are in a
shell on the domain controller. Run a few things so there is traffic to look at:

```
whoami
hostname
dir
```

**CAPTURE: `ext-net-09-ssh-session.png`**
Frame the Wireshark window and the SSH session together in one shot. The terminal shows `whoami`
returning `smithlab\keenan_admin` and `hostname` returning `DC-1`. The packet list above it shows
the traffic those commands generated, every row reading `Encrypted packet`.

Type `exit` and press **Enter** to disconnect.

### Step 3. Read the capture

Every keystroke and every line of output generated packets, and not one of them gives up its
contents. The Info column reads `Encrypted packet (len=...)` on every row. Not the command you
typed, not the output that came back.

**Two things are visible anyway, and they are worth naming.** First, scroll to the top of the capture and the
opening exchange is in the clear: the version banner names the SSH implementation and its version
number on both sides, which is how a scanner fingerprints a server before it tries anything. The
encryption starts after the key exchange, not before it.

Second, everything after that point is opaque but the **metadata is not**. Read the timestamps in
the packet list. Client packet, server packet, microseconds apart, over and over, one pair per
keystroke, because SSH echoes each character back from the remote side as you type. An observer sees
which hosts are talking, on which port, for how long, in which direction, and roughly how much. That
is enough to infer how much was typed and where the pauses were, without decrypting a byte. Traffic
analysis is built on exactly this.

Hold that thought for Part 5.

---

## Part 4: DHCP

> Source checklist: steps 19 and 20. The source runs `ipconfig /renew` alone, which produces two
> packets. This produces all four.

### Step 1. Filter for DHCP

**On `CLIENT-1`,** in the Wireshark filter bar:

```
dhcp
```

> On Wireshark 2.x and older the filter is `bootp`, because DHCP is an extension of the older BOOTP
> protocol and the dissector carried that name for years. Current versions accept `dhcp`. If the
> bar turns red, you are on an old build.

### Step 2. Release and renew

**On `CLIENT-1`,** right-click **Start** > **Terminal (Admin)**:

```
ipconfig /release
```

The adapter loses its address. Then:

```
ipconfig /renew
```

The address comes back.

> **Releasing is safe here.** `CLIENT-1` is being driven from the VirtualBox console, not over the
> network, so dropping its IP does not disconnect you from anything. Doing this over RDP would cut
> your own session.

### Step 3. Read the sequence

**CAPTURE: `ext-net-10-dhcp-dora.png`**
The DHCP packet list showing the full exchange.

You should see, in order:

| Packet | Direction | What it is |
| --- | --- | --- |
| DHCP Release | client to server | Giving up the current lease |
| DHCP Discover | client broadcast | Is there a DHCP server out there |
| DHCP Offer | server to client | Here is an address you can have |
| DHCP Request | client broadcast | I would like that address |
| DHCP ACK | server to client | It is yours, here are your options |

Discover, Offer, Request, Acknowledge. **DORA**, and it comes up on the A+ objectives regularly.

Click the **DHCP ACK** and expand **Dynamic Host Configuration Protocol** in the detail pane, then
scroll to the **Option** fields. This is where the base build's DHCP scope configuration shows up on
the wire: **Router** (the gateway), **Domain Name Server**, and **Domain Name**. Those are the exact
scope options set during the core lab, arriving at the client.

> Running `ipconfig /renew` on its own skips Discover and Offer entirely. A renewal is a unicast
> conversation with a server the client already knows, so it is Request and ACK only. That is why
> the source's two-packet result looks nothing like the four-packet sequence every study guide
> shows, and it is worth knowing which situation produces which.

---

## Part 5: DNS

> Source checklist: steps 21 and 22.

### Step 1. Filter for DNS

**On `CLIENT-1`,** in the Wireshark filter bar:

```
dns
```

### Step 2. Look up two names

**On `CLIENT-1`,** in the terminal:

```
nslookup google.com
nslookup disney.com
```

**CAPTURE: `ext-net-11-dns-cleartext.png`**
The DNS query and response pairs, with the queried names readable in the Info column.

### Step 3. Read what is exposed

Click a query packet and expand **Domain Name System (query)** > **Queries**. The name you looked up
is sitting there in plain text. Do the same on the response and the returned addresses are equally
plain.

**Put this next to Part 3.** The SSH session was encrypted so thoroughly that not even the commands
were recoverable. The DNS lookup that found the server in the first place was sent in the clear.
Anyone positioned on the path sees every name you resolve, even when every connection that follows
is encrypted. The content of your traffic and the record of where it went are two different
exposures, and encrypting one does nothing for the other.

That gap is the entire reason DNS over HTTPS and DNS over TLS exist.

Worth noticing in the capture: the queries go to `172.16.0.1`, not to a public resolver. `CLIENT-1`
asks `DC-1`, and `DC-1` forwards on its behalf. This capture shows only the first leg. The forwarded
query out to the internet happens on `DC-1`'s other adapter and is invisible from here.

---

## Part 6: RDP

> Source checklist: steps 23 and 24. The source already has RDP traffic because the whole session
> runs over it. Here it has to be generated.

### Step 1. Filter for RDP

**On `CLIENT-1`,** in the Wireshark filter bar:

```
tcp.port == 3389
```

> Filtering on `rdp` alone usually returns nothing useful. Modern RDP is wrapped in TLS, so
> Wireshark dissects it as TLS and the RDP dissector never engages. Filtering by port is the
> reliable approach, which is why the source specifies it that way.

### Step 2. Connect to DC-1

**On `CLIENT-1`,** in the terminal:

```
mstsc /v:172.16.0.1
```

Log in as `SMITHLAB\Keenan_Admin`. A certificate warning about the remote computer's identity is
expected in a lab with no PKI. Click **Yes**.

You now have a Remote Desktop session into the domain controller, running inside a VirtualBox window
on your client, which is itself running on your laptop. That is a lot of nesting. Keep track of
which window is which before typing anything.

**CAPTURE: `ext-net-12-rdp-session.png`**
The Remote Desktop window showing `DC-1`'s desktop.

### Step 3. Watch it never stop

Leave the RDP session open, do nothing in it, and switch back to Wireshark on `CLIENT-1`.

**CAPTURE: `ext-net-13-rdp-continuous.png`**
The packet list filling continuously with port 3389 traffic while nothing is being done in the
session.

Every other protocol in this lab produced traffic in response to an action. Ping when you pinged.
DNS when you looked something up. DHCP when you renewed. RDP does not stop, because RDP is a live
video stream of one machine's screen sent to another. A blinking cursor is a screen change. A clock
updating is a screen change. The connection transmits continuously whether anyone is touching it or
not.

**The practical version:** an idle RDP session is not free. It consumes bandwidth for as long as it
is open, which is why "just leave it connected" is a real cost on a constrained link, and why an
abandoned RDP session is visible in traffic monitoring long after the person walked away.

### Step 4. Disconnect

Close the Remote Desktop window. Traffic on 3389 stops.

**On `CLIENT-1`,** stop the Wireshark capture with the red square in the toolbar.

---

## Cleanup

**Confirm the block rule is gone.** On `DC-1`:

```powershell
Get-NetFirewallRule -DisplayName "Lab Block ICMPv4 Echo Request" -ErrorAction SilentlyContinue
```

No output means it is removed. If it returns a rule, delete it. Leaving it in place breaks ping to
the domain controller for every future lab.

**Leave firewall logging on.** It costs nothing and it is genuinely useful.

**Decide about OpenSSH and RDP.** Both were enabled for this lab and both are lab-only shortcuts on
a domain controller. Leaving them is fine in an isolated lab, and there is an argument for leaving
RDP specifically, since it is useful for future work. If you want them off:

```powershell
Stop-Service sshd
Set-Service -Name sshd -StartupType Disabled
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 1
```

**Do not delete the VMs.** The source checklist ends by deleting the Azure resource group to stop
billing. Nothing here is billing you.

Shut both VMs down from **inside Windows** using Start > Shut down, not VirtualBox's Power Off.
`DC-1` is a domain controller and the directory database needs to close cleanly.

---

## Complete screenshot list

Thirteen captures. This is the full set.

| # | File | Part | What it shows |
| --- | --- | --- | --- |
| 01 | `ext-net-01-openssh-server-installed.png` | 0 | sshd running on `DC-1`, firewall rule present |
| 02 | `ext-net-02-firewall-logging-enabled.png` | 0 | Log dropped packets set to Yes, log path visible |
| 03 | `ext-net-03-icmp-request-reply.png` | 1 | Four request and reply pairs to `DC-1` |
| 04 | `ext-net-04-icmp-internet.png` | 1 | ICMP to a public address through the NAT gateway |
| 05 | `ext-net-05-firewall-block-rule.png` | 2 | The Block rule in Inbound Rules on `DC-1` |
| 06 | `ext-net-06-ping-timeout.png` | 2 | Replies then timeouts, with requests unanswered in the capture |
| 07 | `ext-net-07-dc1-firewall-drop-log.png` | 2 | `pfirewall.log` DROP entries naming the source |
| 08 | `ext-net-08-ping-resumes.png` | 2 | Timeouts above, replies below, after rule removal |
| 09 | `ext-net-09-ssh-session.png` | 3 | SSH into `DC-1`, with the encrypted packets that carried it |
| 10 | `ext-net-10-dhcp-dora.png` | 4 | Release, Discover, Offer, Request, ACK |
| 11 | `ext-net-11-dns-cleartext.png` | 5 | Queried names readable in plain text |
| 12 | `ext-net-12-rdp-session.png` | 6 | Remote Desktop into `DC-1` from `CLIENT-1` |
| 13 | `ext-net-13-rdp-continuous.png` | 6 | Port 3389 traffic with the session idle |

All of these go in `D:\ad-lab\docs\images\` alongside the existing captures. The `ext-net-` prefix
keeps them sorted separately from the core build and from Labs 1, 2, and 3.

**These captures are displayed in the [README](../README.md), not here.** This runbook is
instructions for building the lab. The README is the visual walkthrough of what the lab produced.

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

**Wireshark shows no interfaces, or the capture button is greyed out.** Npcap did not install or is
not running. Re-run the Wireshark installer and make sure Npcap is selected. A reboot is sometimes
required.

**Wireshark opens but no traffic appears at all.** You are on the wrong interface, or the display
filter is excluding everything. Clear the filter bar and confirm packets are arriving before
filtering.

**The filter bar turns red.** Syntax error. `tcp.port == 3389` needs the double equals. `dhcp` is
`bootp` on old Wireshark versions.

**`Add-WindowsCapability` fails on `DC-1`.** It needs internet, which `DC-1` has through its NAT
adapter. If it fails with a download error, confirm `DC-1` itself can reach the internet, not just
`CLIENT-1`.

**SSH connects but rejects the password.** Try the quoted domain form,
`ssh "smithlab\keenan_admin@172.16.0.1"`, or the UPN form,
`ssh keenan_admin@smithlab.local@172.16.0.1`. PowerShell treats a bare backslash as an escape
character.

**The ping never stops after creating the block rule.** The rule was created as Outbound instead of
Inbound, or the protocol was left as Any instead of ICMPv4, or Echo Request was not ticked under
Customize. Check the rule's properties rather than rebuilding it.

**`pfirewall.log` does not exist or is empty.** Logging is off on the profile `DC-1` is actually
using. Run `Get-NetConnectionProfile` to see which profile is active and enable logging there.
Writes are also buffered, so wait thirty seconds before concluding it is broken.

**RDP will not connect.** Confirm `fDenyTSConnections` is `0` and the Remote Desktop firewall group
is enabled on `DC-1`. Both are in Part 0 Step 3.

**Ping to `DC-1` fails permanently after this lab.** The block rule is still there. See Cleanup.

---

## Lab-only shortcuts

| Shortcut | Why it is fine here | Why it is wrong in production |
| --- | --- | --- |
| OpenSSH Server installed on a domain controller | Cheapest way to generate real SSH traffic without a Linux VM | A DC should run the minimum services required to be a DC. Every additional listener is additional attack surface on the most privileged host in the environment |
| Remote Desktop enabled on a domain controller | Needed to generate RDP traffic to capture | DC administration belongs on a privileged access workstation with a jump host, not direct RDP from a general-purpose client |
| Domain admin credentials used for SSH and RDP | One admin account exists in this lab | Interactive admin logons to a DC from a workstation expose those credentials to anything already resident on that workstation |
| Firewall rule applied to all three profiles | Simpler than determining the active profile mid-lab | Rules should be scoped to the profile and the source addresses they are meant for |
| Wireshark installed on a domain-joined production-style client | Isolated lab | Packet capture tooling on general-purpose endpoints is itself a risk, and capture files routinely contain credentials and session tokens |
| No PKI, RDP certificate warning clicked through | Nothing here is real | Clicking through certificate warnings is the exact habit that makes machine-in-the-middle attacks work |

---

## What I would do differently in production

**Capture on both ends, always.** Part 2 is the argument. A capture from the sender cannot
distinguish a firewall drop from a dead host from a cable fault. Adding the destination's own record
of the decision turns an ambiguous observation into a definite one.

**Do not troubleshoot from ping alone.** ICMP is frequently blocked as a matter of policy on
internet-facing hosts, which means "ping fails" carries much less information than people assume.
The host may be perfectly healthy and answering on every port that matters.

**Encrypt DNS.** Part 5 is the argument. Every name resolved by every client is visible to anyone on
the path, regardless of how well the resulting connections are protected.

**Log firewall drops centrally.** `pfirewall.log` is a local text file that rotates at 4 MB and is
trivially deleted by anyone with local admin. It answers a question during a troubleshooting session
and is not an audit trail. Forward it, the same argument as event 4740 in Lab 3.

---

## A+ Core 2 and Core 1 objective coverage

- **Core 1 2.1** Ports and protocols: SSH 22, DNS 53, DHCP 67/68, RDP 3389, all observed live
- **Core 1 2.6** Network services: DNS and DHCP behavior on the wire, DORA
- **Core 1 5.7** Troubleshooting network issues: `ping`, `nslookup`, `ipconfig`, packet capture
- **Core 2 1.2** Command-line tools: `ping -t`, `ipconfig /release` and `/renew`, `nslookup`,
  `mstsc`, `ssh`
- **Core 2 2.1** Security concepts: encrypted versus cleartext protocols, host-based firewalls
- **Core 2 2.5** Workstation security: firewall configuration and rule precedence

The DORA sequence and the port numbers in particular show up constantly, usually as a scenario
asking which step of the process failed or which port a described service uses. Watching them
happen once is worth more than a dozen flashcard repetitions.
