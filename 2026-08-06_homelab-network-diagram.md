# Cybersecurity Home Lab — Network Diagram

**Last updated:** 2026-09-13
**Platform:** Oracle VirtualBox 7.2.6
**Status:** Tier 1 (Network Foundations) and Tier 2 (Visibility/SIEM) complete — as-built

## As-Built State — Tier 2 Complete

This diagram reflects the lab exactly as built and confirmed working, including final IP addressing. See the companion checklist docs (`2026-08-06_tier1-network-foundations-checklist.md`, `2026-08-23_tier2-visibility-siem-checklist.md`) for the step-by-step builds, and the build reports (`2026-08-23_tier1-build-report.md`, `2026-09-13_tier2-build-report.md`) for the full narratives and troubleshooting logs.

```mermaid
flowchart LR
    subgraph OUTSIDE["Outside / Attack Segment<br/>Host-only Network (192.168.56.0/24)"]
        KALI["Kali Linux VM<br/>Attacker Machine<br/>192.168.56.21"]
    end

    subgraph FW["pfSense CE VM — 'pfSense-FW'<br/>Firewall + Router"]
        direction LR
        WAN["WAN NIC<br/>192.168.56.10/24"]
        LAN["LAN NIC<br/>192.168.164.10/24"]
        WAN --- LAN
    end

    subgraph INSIDE["Inside / Target Segment<br/>Host-only Network (192.168.164.0/24)"]
        DC["Windows Server 2022 VM<br/>Domain Controller (DC)<br/>192.168.164.11 — testlab.com"]
        CLIENT["Windows 10 VM<br/>Domain Client — Paul Maudib<br/>192.168.164.12"]
        SIEM["Wazuh-SIEM VM<br/>192.168.164.20"]
        DC <--> CLIENT
        DC -. Sysmon + Windows logs .-> SIEM
        CLIENT -. Sysmon + Windows logs .-> SIEM
    end

    KALI <--> WAN
    LAN <--> DC
    LAN <--> CLIENT
    LAN <--> SIEM
    LAN -. syslog UDP/514 .-> SIEM
```

## As-Built Addressing

| Device | Interface | IP Address | Gateway | Notes |
|---|---|---|---|---|
| pfSense-FW | WAN | 192.168.56.10/24 | — | Faces Kali; no upstream gateway (isolated segment) |
| pfSense-FW | LAN | 192.168.164.10/24 | — | Gateway for the Inside segment; forwards firewall events via syslog to Wazuh |
| Kali Linux | eth0 | 192.168.56.21/24 | 192.168.56.10 | Attacker VM |
| Windows Server 2022 (DC) | Ethernet | 192.168.164.11/24 | 192.168.164.10 | Domain Controller (DC), Active Directory (AD) domain `testlab.com`, DNS points to self (127.0.0.1); Wazuh agent + Sysmon installed |
| Windows 10 (Client) | Ethernet | 192.168.164.12/24 | 192.168.164.10 | User: Paul Maudib; DNS points to DC (192.168.164.11); Wazuh agent + Sysmon installed |
| Wazuh-SIEM (Amazon Linux 2023) | eth0 | 192.168.164.20/24 | 192.168.164.10 | Security Information and Event Management (SIEM) — collects DC/Client agent telemetry and pfSense syslog |

## Legend

| Component | Role |
|---|---|
| Kali Linux VM | Attack simulation machine — sits on its own segment, outside the "protected" network |
| pfSense CE VM | Virtual firewall and router (FW) — controls and logs traffic between the two segments |
| Windows Server 2022 VM | Domain Controller (DC) — runs Active Directory (AD), the service that manages users, computers, and permissions for the network |
| Windows 10 VM | Domain Client — a regular user workstation joined to the AD domain |
| Host-only Networks | VirtualBox virtual switches — each isolates traffic to only the VMs attached to it; neither has internet access |

## Firewall Rules (As-Built)

**LAN rules** (Client → DC, Active Directory logon ports — for firewall-syntax practice; not actually enforced since Client and DC share a subnet and this traffic never routes through pfSense — see the build report for why):

| Source | Destination | Protocol | Port | Purpose |
|---|---|---|---|---|
| 192.168.164.12 (Client) | 192.168.164.11 (DC) | TCP/UDP | 53 (DNS) | Domain Name System — name resolution |
| 192.168.164.12 (Client) | 192.168.164.11 (DC) | TCP/UDP | 88 (Kerberos) | Authentication |
| 192.168.164.12 (Client) | 192.168.164.11 (DC) | TCP/UDP | 389 (LDAP) | Directory query/lookup |

**WAN rules** (Kali → DC, controlled attack path — genuinely enforced and logged):

| Source | Destination | Protocol | Port | Purpose |
|---|---|---|---|---|
| 192.168.56.21 (Kali) | 192.168.164.11 (DC) | ICMP | — | Connectivity testing |
| 192.168.56.21 (Kali) | 192.168.164.11 (DC) | TCP/UDP | 3389 (RDP) | Remote Desktop Protocol attack surface |
| 192.168.56.21 (Kali) | 192.168.164.11 (DC) | TCP | 445 (SMB) | Server Message Block — file sharing / attack surface |

All three WAN rules block-by-default otherwise (pfSense's WAN interface has no other rules — implicit deny-all), and RFC1918 (private network) blocking was deliberately disabled on WAN since the WAN segment itself is a private range.

## Current State (Before Tier 1)

For reference, this is the flat network from the original setup — all three VMs on one Host-only Network with no firewall between them:

```mermaid
flowchart LR
    subgraph FLAT["Host-only Network (vboxnet0) — no segmentation"]
        KALI2["Kali Linux VM"]
        DC2["Windows Server 2022 VM (DC)"]
        CLIENT2["Windows 10 VM (Client)"]
    end
    KALI2 <--> DC2
    KALI2 <--> CLIENT2
    DC2 <--> CLIENT2
```

## Tier 2 Additions (Complete)

- **Wazuh-SIEM VM** (Amazon Linux 2023, `192.168.164.20`): collects Wazuh agent telemetry (Windows Event Logs + Sysmon) from the DC and Client, and receives pfSense firewall events via syslog (UDP/514).
- **Temporary second NAT adapters**: used transiently on the DC, Client, and pfSense during installs (Wazuh agent MSI, Sysmon, pfSense packages) to reach the internet from an otherwise fully isolated segment, then removed afterward — the lab has no permanent internet access on either Host-only network.

## Known Limitation

pfSense's raw syslog reaches Wazuh at the network level (confirmed via `tcpdump`) but isn't decoded into a proper Wazuh alert — no built-in decoder exists for pfSense's log format. Deliberately deferred to Tier 5 (write a custom Wazuh decoder/rule). See the Tier 2 build report for details.

## Planned Future Additions (Tier 3+)

- **AD hardening baseline**: Group Policy, audit policy tuning, vulnerability scan.
- **Attack simulation** (Tier 4) against the confirmed SMB/RDP attack surface, validated against Wazuh detections.
- **Custom Wazuh decoder for pfSense** (Tier 5), closing the known limitation above.
