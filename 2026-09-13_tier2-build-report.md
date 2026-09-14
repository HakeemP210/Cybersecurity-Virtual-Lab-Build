# Cybersecurity Home Lab — Tier 2 Build Report

**Author:** Hakeem Phillips
**Date range:** 2026-08-23 through 2026-09-13
**Platform:** Oracle VirtualBox 7.2.6, hosted on Windows 11 Pro
**Status:** Tier 2 (Visibility / SIEM) complete

## 1. Objective

Close the visibility gap identified at the end of Tier 1: pfSense can only see and log traffic that actually crosses between the two network segments, but it is structurally blind to anything happening *within* a segment — Client↔DC traffic, for instance. Tier 2 adds a SIEM (Security Information and Event Management) platform and endpoint agents so that activity the firewall can never see becomes visible, searchable, and (ideally) automatically alerted on.

## 2. Architecture

Full diagram and as-built addressing: see `2026-08-06_homelab-network-diagram.md`.

**Summary:** A Wazuh SIEM (Amazon Linux 2023, `192.168.164.20`) was added to the domain segment. Wazuh agents were installed on the DC and Client for endpoint telemetry, Sysmon was layered on top of both for richer process/network visibility, and pfSense was configured to forward firewall events via syslog.

## 3. Build Timeline

| Date | Milestone |
|---|---|
| 2026-08-23 | Wazuh OVA (Open Virtualization Appliance) imported and booted; static IP configuration and dashboard login troubleshooting |
| 2026-08-24 – 08-26 | Wazuh agents deployed to DC and Client; pfSense syslog forwarding configured; extended clock-drift troubleshooting |
| 2026-09-07 – 09-10 | Sysmon deployed to DC and Client; Wazuh agent config updated to collect the new Sysmon channel |
| 2026-09-10 – 09-13 | End-to-end validation (failed/successful logons, pfSense connectivity); first detection exercise — brute-force simulation from Kali against the DC, confirmed caught by Wazuh's default ruleset |

## 4. Troubleshooting Log

### 4.1 Wrong assumption about the Wazuh appliance's OS
- **Issue:** Assumed the Wazuh OVA was Ubuntu-based (guided toward Netplan for static IP configuration); `/etc/netplan` didn't exist.
- **Root cause:** The appliance runs **Amazon Linux 2023** (confirmed via `/etc/os-release`), a RHEL/Fedora-family distribution, not Debian/Ubuntu. NetworkManager wasn't installed either; the actual mechanism in use was **systemd-networkd**.
- **Fix:** Located and edited `/etc/systemd/network/20-eth0.network`, converting it from `DHCP=ipv4` to static `Address=`/`Gateway=`/`DNS=` entries.
- **Takeaway:** Confirmed via `/etc/os-release` before guessing further — same discipline as Tier 1's "verify, don't assume" lesson, just a second distribution family this time.

### 4.2 No clipboard sharing on the Wazuh VM
- **Issue:** Copy/paste into the Wazuh console didn't work even with VirtualBox's Shared Clipboard set to Bidirectional.
- **Root cause:** Clipboard integration requires VirtualBox Guest Additions, which aren't installed on this minimal appliance image and can't easily be added to it.
- **Fix:** Rewrote multi-line file edits as single-line `printf`/`tee` commands typeable without paste.

### 4.3 `nano` edits silently not saving
- **Issue:** A file edit (`/etc/systemd/network/20-eth0.network`, later `/var/ossec/etc/ossec.conf` on the Wazuh manager) appeared to save in `nano` but the original content was still present on re-check.
- **Fix:** Abandoned interactive editing in favor of `sudo tee`/`sed -i` with exact line numbers (found via `grep -n`) for anything safety-critical.

### 4.4 Wazuh manager's `ossec.conf` broken twice by duplicate `<ossec_config>` tags
- **Issue:** Adding a `<remote>` syslog listener block caused `wazuh-manager` to fail to start (`Configuration error. Exiting`) — twice, from two different malformed edits (extra opening tag, then extra closing tag placed wrong).
- **Fix:** Used `grep -n "ossec_config"` to get exact line numbers, then `sed -i '<N>d;<M>d'` to remove precisely the two bad lines and `tee -a` to append one correct closing tag at the true end of the file.
- **Takeaway:** Same class of error as a Tier 1 pfSense/network mistake — an instructional example (mine) that included illustrative wrapper tags got pasted in literally. Worth being explicit that surrounding context in an example is not meant to be copied.

### 4.5 Dashboard login vs. console login are separate credentials
- **Issue:** Assumed the OVA's printed console credentials (`wazuh-user`/`wazuh`) would also work on the web dashboard.
- **Fix:** Dashboard uses a separate `admin` account; correct password was found via trial rather than the documented `wazuh-passwords.txt` extraction path, which wasn't needed here.

### 4.6 First DC agent enrollment silently never connected
- **Issue:** After installing the Wazuh agent MSI on the DC, it showed as "running" locally but never appeared in the manager's agent list.
- **Root cause:** The deploy wizard's "Server address" field was pre-filled with `192.168.164.11` — the **DC's own IP**, not the SIEM's (`192.168.164.20`). The agent was phoning home to itself.
- **Fix:** Uninstalled (`msiexec /x`), regenerated the deploy command with the correct manager IP, reinstalled.

### 4.7 Client agent install failed with no visible error
- **Issue:** `NET START Wazuh` returned "The service name is invalid" after running the install command on the Client — the service was never created.
- **Root cause:** PowerShell was not running elevated (window title lacked "Administrator:"); the silent `/q` MSI install failed without surfacing an error.
- **Fix:** Re-ran PowerShell as Administrator; install succeeded cleanly.

### 4.8 pfSense syslog: no listener by default, then a clock-drift red herring
- **Issue:** After configuring pfSense to forward firewall events to Wazuh on UDP/514, nothing appeared in the dashboard's Discover view, at any time range.
- **Root cause (part 1):** Wazuh's manager doesn't listen for raw syslog out of the box — required adding a `<remote><connection>syslog</connection>...</remote>` block (see 4.4 above for the config mishaps this caused).
- **Root cause (part 2):** Once the listener was confirmed working via `tcpdump` (packets genuinely arriving), searches still returned nothing. Traced to **pfSense's system clock having drifted stale** since the one-time manual `date` set back in Tier 1 — the isolated lab network has no NTP (Network Time Protocol) server to keep it accurate. Wazuh's own clock was correct; pfSense's was not, which briefly caused an incorrect suspicion in the opposite direction before checking both independently.
- **Fix:** Corrected pfSense's clock via the console shell — also corrected the `date` command syntax itself, since FreeBSD's format (`ccyymmddHHMM.SS`) is read outside-in and differs from the format initially assumed. One clock-fix attempt was also mistakenly run on the **Wazuh** VM instead of pfSense (different `date` command syntax entirely, being Linux vs. FreeBSD) before being caught by checking the shell prompt.
- **Remaining gap:** Even with the clock fixed and connectivity fully proven, pfSense's raw syslog still doesn't render as a proper Wazuh alert — no built-in decoder exists for its log format, and Wazuh's archive-shipping (for un-decoded events) requires a separate Filebeat module not enabled by default. **Deliberately deferred to Tier 5** (write a custom Wazuh decoder/rule set for pfSense).

### 4.9 Sysmon deployment: repeated paste corruption
- **Issue:** Multiple PowerShell paste attempts on the DC and Client dropped or merged command text unpredictably — one attempt silently deleted an entire command from the middle of a block and reattached a stray parameter to the wrong line.
- **Fix:** Iterated through several mitigations: combining commands with semicolons (helped with newline-merging), then falling back to typing commands individually when even that got corrupted. No single root cause was confirmed; treated as an unreliable host/guest clipboard path rather than chased further.

### 4.10 Event Viewer "Access is denied" on the Client
- **Issue:** Opening Event Viewer to check the new Sysmon log threw "Access is denied (5)," despite PowerShell running elevated in another window.
- **Root cause:** Elevation doesn't carry over between separately-launched processes — Event Viewer opened via Start-menu search runs at normal privilege regardless of what else is elevated.
- **Fix:** Relaunched Event Viewer specifically via "Run as administrator."

### 4.11 Hydra reported success while actually failing (RDP)
- **Issue:** A Hydra RDP brute-force attempt exited cleanly (`1 of 1 target completed, 0 valid password found`) with no obvious failure — but the actual per-attempt output showed `[ERROR] freerdp: The connection failed to establish` for every try.
- **Root cause:** Hydra's RDP module (built on an older FreeRDP negotiation flow) could not complete the handshake against a target requiring NLA (Network Level Authentication) — confirmed afterward by `netexec`'s own recon output showing `nla:True`. No real login attempt ever reached the DC, so this would never have produced genuine failed-logon telemetry despite looking like a completed run.
- **Fix:** Substituted `netexec` (the maintained successor to `crackmapexec`, which failed to install), which completed real authentication attempts against both RDP and SMB.
- **Takeaway:** A clean summary line from a tool is not proof it did what was intended — the per-attempt detail told a different story than the final line.

### 4.12 Hydra's SMB module failed for a different, expected reason
- **Issue:** Hydra's SMB module could not authenticate at all.
- **Root cause:** It targets the legacy SMB1 protocol, which Windows Server 2022 disables by default — the same category of "secure default looks like a bug" lesson from Tier 1's RFC1918 and WAN-deny-by-default findings.
- **Fix:** Same `netexec` substitution as above.

## 5. Detection Exercise Results (Part 6)

Brute-force attempts against the DC (`192.168.164.11`) via both RDP and SMB, sourced from Kali (`192.168.56.21`), using `netexec` with an 8-entry wordlist against the `administrator` account:

- **24 failed-logon events** (Windows Event ID 4625) captured across both protocols.
- **Rule 60122** ("Logon Failure - Unknown user or bad password") fired per individual event, level 5 — but mapped to MITRE ATT&CK **T1531 (Account Access Removal)**, an inaccurate technique for a failed login attempt.
- **Rule 60204** ("Multiple Windows Logon Failures") additionally fired twice, level **10**, `rule.frequency: 8` — matching the exact size of the wordlist — correctly mapped to **T1110 (Brute Force)** under the **Credential Access** tactic.
- **Conclusion:** Wazuh's default ruleset detects this attack pattern out of the box, without any custom rule-writing, and the escalated correlation rule carries more accurate threat classification than the base per-event rule.

**Bonus finding (not part of the original plan):** `netexec`'s SMB recon banner reported `Null Auth: True` on the DC — anonymous/null SMB sessions are accepted. This is a real hardening gap, flagged for remediation in Tier 3.

## 6. Key Lessons Learned

- **Confirm the actual OS/tool stack before following generic instructions** — assumptions about Ubuntu/Netplan and about Hydra's RDP module both cost real time before being caught by direct verification.
- **A tool's summary output can be misleading** — "0 valid passwords found, target completed" read as a clean failure-only run; the real per-line output told a very different story.
- **Detection engineering has two distinct layers worth checking separately**: are individual events logged at all, and does anything correlate a *pattern* across them into a higher-confidence alert? Tier 2 confirmed both layers work for brute-force login attempts.
- **Isolated lab networks lose time sync silently** — this is now the second clock-drift incident (pfSense, again) since there's no NTP source; worth treating as an expected recurring maintenance item rather than a one-off fix.

## 7. Next Steps (Tier 3+)

- AD hardening baseline: Group Policy, audit policy tuning, vulnerability scan — including remediating the SMB Null Auth finding above.
- Tier 4 attack simulation against the now-confirmed SMB/RDP surface, validated against Wazuh detections.
- Tier 5: write a custom Wazuh decoder/rule set for pfSense syslog, closing the Section 4.8 gap; also revisit the T1531 mismatch on rule 60122 as a candidate for a custom rule override.
