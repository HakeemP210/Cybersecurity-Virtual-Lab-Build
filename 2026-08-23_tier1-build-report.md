# Cybersecurity Home Lab — Tier 1 Build Report

**Author:** Hakeem Phillips
**Date range:** 2026-08-04 through 2026-08-23
**Platform:** Oracle VirtualBox 7.2.6, hosted on Windows 11 Pro
**Status:** Tier 1 (Network Foundations) complete

## 1. Objective

Build a segmented home lab to practice hands-on skills developed while studying for the CompTIA Security+ certification, simulating a small corporate network: a Windows domain (Active Directory), a firewall/router boundary, and an isolated attacker machine for controlled offensive-security exercises. The goal of Tier 1 specifically was to take an initially flat network (all VMs on one virtual switch) and introduce a firewall to create a real "inside vs. outside" boundary, with logged, rule-based access between them.

## 2. Architecture

Full diagram and as-built IP addressing: see `2026-08-06_homelab-network-diagram.md`.

**Summary:**
- **Windows Server 2022** — Domain Controller (DC) for Active Directory (AD) domain `testlab.com`, also running DNS (Domain Name System).
- **Windows 10** — domain-joined client workstation, user "Paul Maudib."
- **Kali Linux** — isolated attacker VM, sits on a separate network segment from the domain.
- **pfSense CE** (Community Edition) — firewall/router VM added during Tier 1, with one NIC (Network Interface Card) facing each segment.
- Two isolated VirtualBox Host-only Networks: `192.168.56.0/24` (Outside/attack segment) and `192.168.164.0/24` (Inside/domain segment) — neither has internet access.

## 3. Build Timeline

| Date | Milestone |
|---|---|
| 2026-08-04 | Windows 10 client VM created; Active Directory installed on Windows Server; first AD user (Paul Maudib) created; DNS server role added |
| 2026-08-08 | Two isolated Host-only networks created; pfSense installed |
| 2026-08-11 | Firewall interface IPs assigned (WAN 192.168.56.10, LAN 192.168.164.10); pfSense install/config troubleshooting (see Section 4) |
| 2026-08-16 | pfSense web GUI (Graphical User Interface) setup wizard completed; first LAN firewall rules (DNS/Kerberos/LDAP) built |
| 2026-08-18 | WAN firewall rules built (ICMP, RDP); Kali given a static IP; cross-segment connectivity troubleshooting |
| 2026-08-23 | SMB rule added and tested; firewall logging and clock sync fixed; all four VMs snapshotted as "Segmented Network"; Tier 1 closed out |

## 4. Troubleshooting Log

This is the substantive part of the build — nearly every stage surfaced a real, diagnosable problem rather than working on the first attempt. Each entry follows: **Issue → Root Cause → Fix.**

### 4.1 pfSense VM wouldn't boot — "CPU doesn't support long mode"
- **Root cause:** The VM's guest OS type was set to a 32-bit variant, which hides 64-bit CPU features from the guest even though the host CPU supports them. pfSense CE requires 64-bit ("long mode").
- **Fix:** Changed Settings → General → Basic → Version to a 64-bit type; confirmed VT-x/AMD-V (Intel/AMD hardware virtualization) and PAE/NX were enabled under Settings → System → Acceleration/Processor.

### 4.2 FreeBSD kernel panic — "running without device atpic requires a local APIC"
- **Root cause:** VirtualBox's I/O APIC (Advanced Programmable Interrupt Controller — manages hardware interrupts) is disabled by default on new VMs; recent FreeBSD (which pfSense is built on) requires it.
- **Fix:** Enabled "Enable I/O APIC" under Settings → System → Motherboard.

### 4.3 Netgate Installer — "Cannot connect to installer daemon"
- **Root cause:** The VM's emulated NIC type (PCnet-FAST III) isn't supported by pfSense's installer kernel, so no network interface could initialize, which the live installer's daemon process depends on.
- **Fix:** Changed both NICs' Adapter Type from PCnet-FAST III to Intel PRO/1000 MT Desktop.

### 4.4 Installer — "no valid storage devices detected"
- **Root cause:** The virtual hard disk was only 2 GB (well under pfSense's minimum) and attached to a legacy IDE controller rather than SATA.
- **Fix:** Removed the undersized disk; created a new 20 GB disk on a SATA (AHCI) controller.

### 4.5 Installer kept relaunching after every reboot
- **Root cause:** The installer ISO remained mounted and ordered ahead of the hard disk in the boot sequence, so the VM kept booting the installer instead of the newly installed OS.
- **Fix:** Ejected the ISO from the virtual optical drive; reordered Settings → System → Motherboard → Boot Device Order so Hard Disk is prioritized.

### 4.6 IP addressing conflict with the host machine
- **Root cause:** VirtualBox's default Host-only networks assign `.1` on each subnet to the physical host machine's own virtual adapter (confirmed via `Get-NetIPAddress` on the Windows host). Assigning pfSense's interfaces to `.1` on either subnet would have collided with the host itself.
- **Fix:** Assigned pfSense's WAN and LAN interfaces to `.10` on their respective subnets instead, reserving `.1` for the host.

### 4.7 RFC1918 (private network) blocking on WAN
- **Root cause:** pfSense's setup wizard defaults to blocking RFC1918 private address space on WAN — correct for a real internet-facing firewall, but this lab's WAN segment (`192.168.56.0/24`) is itself a private range, and Kali's source address is too. Left checked, this setting would silently drop 100% of Kali's traffic regardless of any firewall rule.
- **Fix:** Unchecked "Block RFC1918 Private Networks" during the setup wizard, per the wizard's own guidance for exactly this scenario.

### 4.8 LAN rules (DNS/Kerberos/LDAP) never generated log hits
- **Root cause:** Not a misconfiguration — the Client and DC share the same subnet (`192.168.164.0/24`) and same virtual switch, so traffic between them is switched directly at Layer 2 (via ARP) and never routed through pfSense at all. A firewall can only filter/log traffic that actually crosses between the interfaces it sits between.
- **Takeaway:** Confirmed by testing `nslookup` from the Client, which succeeded even before any LAN rule existed. This distinguishes "the rule is wrong" from "this traffic structurally never reaches the firewall" — an important networking fundamental, not a lab failure.

### 4.9 Kali → DC connectivity failure (multi-layer)
Diagnosed via systematic fault-domain isolation rather than guesswork:
- **Layer 1 — missing default gateway:** Kali initially had no route to the Inside segment at all (`Destination Host Unreachable`, generated locally). Fixed by setting a static default route to pfSense's WAN IP.
- **Layer 2 — stale pfSense WAN interface state:** After the gateway was fixed, Kali still couldn't resolve ARP (Address Resolution Protocol) for anything on the segment, including the gateway itself. Isolated by testing pfSense → DC (via pfSense's own Diagnostics → Ping tool, sourced from LAN) — this succeeded, proving the LAN side and the firewall's LAN interface were healthy. This narrowed the fault specifically to the WAN side, most likely a leftover artifact from earlier switching pfSense's WAN adapter between NAT and Host-only during install troubleshooting (Section 4.3). A full VM reboot of pfSense resolved it.
- **Layer 3 — expected non-response confirmed as expected, not a bug:** Pinging pfSense's own WAN address (`192.168.56.10`) directly continued to fail — correctly, since no rule permits traffic addressed to the firewall itself, only traffic passing through it. This is standard, secure-by-default firewall behavior, not a defect.
- **Result:** `ping 192.168.164.11` from Kali succeeded end to end once the above were resolved — the first fully verified, firewall-enforced, cross-segment traffic flow in the lab.

### 4.10 Windows Firewall silently blocking ICMP on the DC
- **Root cause:** pfSense correctly passed and logged the ICMP traffic, but the DC still didn't reply — Windows Server blocks inbound ICMPv4 Echo Requests by default at the host firewall level, independent of the network firewall.
- **Fix:** Enabled the built-in "File and Printer Sharing (Echo Request - ICMPv4-In)" rule in Windows Defender Firewall on the DC.
- **Takeaway:** A clean demonstration of defense-in-depth — network-layer (pfSense) and host-layer (Windows Firewall) filtering are independent controls; passing one doesn't guarantee passing the other.

### 4.11 RDP (Remote Desktop Protocol) connection failing outright
- **Root cause:** Unlike the ICMP case (which failed silently after being received), this failed at the TCP socket level (`ERRCONNECT_CONNECT_FAILED`) — nothing was listening on port 3389 at all, because Remote Desktop is disabled by default on Windows Server 2022.
- **Fix:** Enabled Remote Desktop under System Properties, plus the corresponding Windows Firewall rule group.

### 4.12 SMB test result misread at first glance
- **Observation:** `smbclient` returned `NT_STATUS_LOGON_FAILURE`, which looks like a failure but actually confirms success at the network layer — the TCP connection and SMB (Server Message Block) protocol negotiation both completed; only the (intentionally invalid) login credentials were rejected. Correctly distinguishing an authentication rejection from a connectivity failure is a recurring, useful skill covered again here.

### 4.13 Firewall logs empty despite working rules
- **Root cause:** The "Log packets that are handled by this rule" checkbox hadn't been enabled on the newer WAN rules.
- **Fix:** Enabled logging per rule.

### 4.14 Firewall log timestamps inaccurate
- **Root cause:** pfSense normally keeps its clock accurate via NTP (Network Time Protocol), but neither lab segment has internet access, so the clock drifted freely with no correction source.
- **Fix:** Set the correct timezone (America/Chicago) under System → General Setup, and manually corrected the system clock via the pfSense console shell.
- **Disposition:** Date is accurate; minor time-of-day drift remains and was accepted as a non-issue for a fully isolated lab environment.

## 5. Firewall Rule Set (Final)

See `2026-08-06_homelab-network-diagram.md` for the full table. Summary:
- **LAN (Client → DC):** DNS (53), Kerberos (88), LDAP (389) — the three ports representing the sequential steps of an AD logon (locate the DC, authenticate, query the directory). Written for rule-syntax practice; not functionally exercised due to the same-subnet behavior described in 4.8.
- **WAN (Kali → DC):** ICMP, RDP (3389), SMB (445) — a deliberately narrow, logged "controlled attack path," standing in contrast to leaving WAN fully open or fully closed. All three confirmed working and logging correctly.

## 6. Key Lessons Learned

- **A working configuration and a functionally tested configuration are not the same thing** — several rules (Section 4.8) were syntactically correct from the start but never actually exercised until the underlying network topology was understood.
- **Systematic fault-domain isolation** (client → gateway → firewall → destination, testing each hop independently) resolved multi-layer failures far faster than trial-and-error changes.
- **Defense-in-depth is visible, not just theoretical**, once you have independent network- and host-level firewalls in the same lab (Section 4.10).
- **Secure defaults sometimes look like bugs** — RFC1918 blocking, WAN's implicit deny-all, and pfSense not answering pings to itself all initially looked like problems before being understood as intentional.

## 7. Next Steps (Tier 2+)

- Deploy a SIEM (Security Information and Event Management) or lightweight sensor (e.g., Wazuh, Security Onion) on the Inside segment to gain visibility into Client↔DC traffic that pfSense structurally cannot see (Section 4.8).
- Forward pfSense and Windows Event Logs into the SIEM.
- Begin Tier 4-style attack simulation from Kali against the now-confirmed SMB/RDP attack surface, with detection validated against the SIEM.
