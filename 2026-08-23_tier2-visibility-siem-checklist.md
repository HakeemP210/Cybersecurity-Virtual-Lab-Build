# Tier 2 Checklist — Visibility (SIEM)

**Goal:** Add a SIEM (Security Information and Event Management — a system that collects logs from other machines and lets you search/alert on them in one place) to the domain segment, so you can actually *see* what's happening on the network instead of just filtering it. This directly addresses the gap from Tier 1: pfSense can only see traffic that crosses between segments — it's structurally blind to Client↔DC traffic on the same subnet. A SIEM with agents installed on those machines closes that gap.

**Estimated time:** 3–5 hours (spread across a few sessions is fine)
**Reference:** `2026-08-23_tier1-build-report.md`, `2026-08-06_homelab-network-diagram.md`

---

## Terms used in this doc

| Acronym | Full name | What it means |
|---|---|---|
| SIEM | Security Information and Event Management | A system that collects logs from many machines, stores them centrally, and lets you search/alert on them |
| Agent | — | A small program installed on a monitored machine that forwards its logs to the SIEM |
| Syslog | System Logging Protocol | A standard way for devices (like firewalls) to send log messages to a remote server |
| Sysmon | System Monitor | A free Microsoft tool that logs much more detailed Windows activity (process creation, network connections) than the default Windows logs |
| OVA | Open Virtualization Appliance | A pre-built virtual machine file you can import directly, instead of installing an OS from scratch |
| RAM | Random Access Memory | Your computer's working memory — how much you have limits how many VMs you can run at once |
| GUI | Graphical User Interface | The visual, browser-based dashboard for the SIEM |
| EPS | Events Per Second | A rough measure of how much log volume a SIEM is processing — not something you'll need to worry about at lab scale |

---

## Part 0 — Before you start: resource planning

Your host has **~16 GB RAM** total. A SIEM is the heaviest thing you'll run in this lab — even a lightweight one typically wants 4 GB minimum, comfortably 6–8 GB. Running DC + Client + Kali + pfSense + SIEM **all at once** will be tight.

- [ ] **0.1** Plan to keep Kali powered off except when actually running an attack/detection exercise (Part 7). You don't need it running while setting up log ingestion.
- [ ] **0.2** We're using **Wazuh** for this build — it's a free, open-source SIEM that's noticeably lighter to run as a single all-in-one VM than something like Security Onion, which is built for larger/multi-sensor deployments. Good fit for this lab's resources.
- [ ] **0.3** Take a snapshot of all four existing VMs before starting, same habit as Tier 1.

## Part 1 — Deploy the SIEM VM

- [ ] **1.1** Download the Wazuh OVA (a pre-built virtual appliance — the easiest way to get Wazuh running without a manual Linux install) from Wazuh's official documentation/downloads page.
- [ ] **1.2** In VirtualBox, go to **File → Import Appliance** and import the OVA.
- [ ] **1.3** Give it a clear name, e.g. `Wazuh-SIEM`. Allocate at least **4 GB RAM** (6–8 GB if your host can spare it) and 2 CPUs.
- [ ] **1.4** **Settings → Network → Adapter 1** — attach it to the **same Host-only network as the DC and Client** (`192.168.164.0/24`), matching the plan in the network diagram. The SIEM needs to be reachable by both machines it's monitoring.
- [ ] **1.5** Boot the VM. Log in at the console (default Wazuh OVA credentials are in Wazuh's docs — change the password immediately once logged in).
- [ ] **1.6** Assign it a static IP on the domain segment, e.g. **`192.168.164.20`** (avoiding `.10` pfSense, `.11` DC, `.12` Client), gateway `192.168.164.10`, DNS `192.168.164.11` (the DC) — same pattern as your other Inside VMs.
- [ ] **1.7** From the Client or DC, open a browser to `https://192.168.164.20` and confirm the Wazuh dashboard loads (you'll get a self-signed certificate warning — expected, same as pfSense earlier).

## Part 2 — Forward pfSense logs to the SIEM

- [ ] **2.1** In pfSense, go to **Status → System Logs → Settings**.
- [ ] **2.2** Under **Remote Logging Options**, check **"Send log messages to remote syslog server."**
- [ ] **2.3** Set the **Remote log servers** field to `192.168.164.20:514` (Wazuh's IP, standard syslog port).
- [ ] **2.4** Check the categories you want forwarded — at minimum, **Firewall Events**.
- [ ] **2.5** Save. On the Wazuh side, you'll need to confirm (or configure) that it's listening for syslog input — check Wazuh's documentation for enabling a syslog listener on the manager if it isn't already active by default in the OVA.
- [ ] **2.6** Generate a test event (e.g., ping Kali → DC again) and confirm it shows up in Wazuh's dashboard under events/discover.

## Part 3 — Install Wazuh agents on the DC and Client

- [ ] **3.1** In the Wazuh dashboard, go to **Agents → Add agent**. It'll generate a download link and a registration command specific to your Wazuh manager's IP.
- [ ] **3.2** On the **DC**, download and run the Wazuh agent MSI installer. During setup, enter the Wazuh manager IP (`192.168.164.20`) when prompted.
- [ ] **3.3** Start the Wazuh agent service (it should start automatically, but confirm via `services.msc` → "Wazuh Agent" → Running).
- [ ] **3.4** Repeat steps 3.2–3.3 on the **Client**.
- [ ] **3.5** Back in the Wazuh dashboard, go to **Agents** and confirm both show status **Active**.

## Part 4 — Add Sysmon for richer Windows visibility (recommended)

Default Windows logs are pretty thin. Sysmon gives you much more useful detail — process creation with full command lines, network connections, and more — which matters a lot once you start running actual attacks against the DC in later tiers.

- [ ] **4.1** Download **Sysmon** from Microsoft's Sysinternals site.
- [ ] **4.2** Download a community Sysmon config (the **SwiftOnSecurity** config is the standard beginner-friendly starting point — well-commented and widely used).
- [ ] **4.3** Install Sysmon on the DC: `sysmon64.exe -i sysmonconfig.xml` (run as Administrator).
- [ ] **4.4** Repeat on the Client.
- [ ] **4.5** Confirm Sysmon is logging: open **Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational** and confirm events are appearing.
- [ ] **4.6** Confirm the Wazuh agent is picking up Sysmon events too — this may require adding the Sysmon event log channel to the agent's config (`ossec.conf` on the Windows agent) if it isn't collected by default.

## Part 5 — Validate end to end

- [ ] **5.1** On the Client, deliberately fail a login (wrong password) — confirm the event shows up in Wazuh within a minute or two.
- [ ] **5.2** On the DC, log in successfully as an admin — confirm that shows up too.
- [ ] **5.3** Check that pfSense firewall events from Part 2 are still flowing.
- [ ] **5.4** At this point you should have three log sources visible in one place: pfSense (network), DC (endpoint + Sysmon), Client (endpoint + Sysmon) — this is the actual payoff of Tier 2.

## Part 6 — First detection exercise

- [ ] **6.1** Power Kali back on.
- [ ] **6.2** From Kali, generate a failed-login storm against the DC — e.g., using `hydra` against RDP or SMB with a small wordlist and an intentionally wrong password set, through the WAN rule you already built in Tier 1.
- [ ] **6.3** In Wazuh, search/filter for the resulting failed-logon events on the DC (Windows Event ID 4625 is the standard "failed logon" event to look for).
- [ ] **6.4** Note whether Wazuh's default rule set generates an actual **alert** for the repeated failures (many SIEMs ship a brute-force-detection rule out of the box) or whether you only see the raw events with no alert — either result is useful information, and the latter sets up a good Tier 5 exercise (writing a custom detection rule).

### Troubleshooting note: `hydra` failed on both protocols, `netexec` substituted for both

Initially looked like Hydra's RDP module worked and only SMB failed — on closer read of the actual output, **both failed**, just differently:

- **RDP:** Hydra reported `1 of 1 target completed, 0 valid password found` (which reads like a clean run), but the actual log lines showed `[ERROR] freerdp: The connection failed to establish` repeated for every attempt. The connection never completed the RDP handshake at all, so no real login attempt ever reached the DC — meaning it never would have generated genuine `4625` events, even though the tool exited without an obvious failure message. Root cause: Hydra's RDP module is built on an older FreeRDP negotiation flow that frequently can't complete the handshake against a target requiring **NLA (Network Level Authentication)** — confirmed by netexec's own recon banner later showing `nla:True` on this DC. Hydra's own output even flags the module as "experimental."
- **SMB:** Hydra's SMB module targets the legacy SMB1 protocol, which Windows Server 2022 disables by default (see Tier 1 lessons on secure defaults looking like bugs).
- `crackmapexec` also failed to install as an alternative.

**`netexec` (the actively maintained successor to crackmapexec) worked cleanly for both protocols:**

```bash
netexec smb 192.168.164.11 -u administrator -p wordlist.txt
netexec rdp 192.168.164.11 -u administrator -p wordlist.txt
```

Both produced genuine `STATUS_LOGON_FAILURE` responses for all 8 passwords — real authentication attempts reaching the DC, exactly what this exercise needed.

Bonus: netexec's recon banners revealed real findings before any credentials were tried — SMB signing enabled (good), SMBv1 disabled (good), RDP requiring NLA (good), and **Null Auth: True** on SMB (anonymous/null sessions accepted — a real hardening gap worth remediating in Tier 3).

**Lesson for the portfolio:** a tool reporting "0 valid passwords found" and exiting cleanly isn't automatically proof it actually tested anything — worth reading the actual per-attempt output, not just the summary line.

## Part 7 — Wrap up

- [ ] **7.1** Snapshot all five VMs now (DC, Client, Kali, pfSense, Wazuh) — label something like "Tier 2 Complete."
- [ ] **7.2** Update the network diagram to add the Wazuh SIEM node and its IP.
- [ ] **7.3** Write a short build-report addendum for the portfolio, same format as the Tier 1 report (issue → root cause → fix), covering whatever actually went wrong during this tier.

---

## Known gap, deferred to Tier 5

**pfSense's raw syslog is confirmed reaching Wazuh at the network level** (verified twice via `tcpdump` on port 514), but it doesn't appear as a proper Wazuh **alert** — Wazuh has no built-in decoder for pfSense's log format, so the messages arrive but never match a rule. Getting them to show as real, parsed alerts requires two follow-on pieces of work, deliberately deferred rather than rushed:

- Enabling Filebeat's archive-shipping module on the Wazuh manager (so raw/unmatched events actually reach the searchable index, not just the local `logall` file).
- Writing a **custom Wazuh decoder and rule set** for pfSense's log format.

This is explicitly the kind of task Tier 5 (Detection Engineering & Response) already calls for — "write custom Sigma/Wazuh detection rules for attacks that weren't caught out of the box." Treat pfSense's raw firewall log as the first real candidate for that exercise, rather than a Tier 2 blocker.

---

**Next up:** Tier 3 — AD hardening baseline (Group Policy, audit policy, vulnerability scan) using the visibility you just built to confirm changes actually take effect.
