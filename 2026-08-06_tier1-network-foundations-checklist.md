# Tier 1 Checklist — Network Foundations (Firewall/Router Deployment)

**Goal:** Split the flat lab network into two segments (an "Outside" attack segment and an "Inside" target segment) separated by a virtual firewall/router, so traffic between Kali Linux and the domain has to pass through — and be logged by — a firewall.

**Estimated time:** 1–2 hours
**Reference diagram:** `2026-08-06_homelab-network-diagram.md`

---

## Terms used in this doc

| Acronym | Full name | What it means |
|---|---|---|
| VM | Virtual Machine | A simulated computer running inside VirtualBox |
| NIC | Network Interface Card | A virtual (or physical) network adapter |
| ISO | International Organization for Standardization (disk image file) | A single file containing an install disc's contents |
| WAN | Wide Area Network | The "outside"-facing network connection on a firewall |
| LAN | Local Area Network | The "inside"-facing, protected network connection on a firewall |
| IP (address) | Internet Protocol address | The numeric address a device uses on a network (e.g., 192.168.56.10) |
| GUI | Graphical User Interface | A visual, point-and-click interface (as opposed to a text-only console) |
| DHCP | Dynamic Host Configuration Protocol | A service that automatically hands out IP addresses to devices |
| DC | Domain Controller | The server that runs Active Directory |
| AD | Active Directory | Microsoft's service for managing users, computers, and permissions on a network |
| DNS | Domain Name System | The service that translates names (like `dc01.lab.local`) into IP addresses |

---

## Part 1 — Prep

- [ ] **1.1** Take a snapshot of all three existing VMs (Windows Server 2022, Windows 10, Kali Linux) before making any changes, so you can roll back if something breaks.
- [ ] **1.2** Download the pfSense CE installer ISO (or OPNsense, if you prefer) from the official site to your host machine.
- [ ] **1.3** Confirm your host machine has enough free RAM/CPU to run 4 VMs at once (DC, Client, Kali, and the new firewall VM).

## Part 2 — Create a second isolated network

- [ ] **2.1** Open **VirtualBox → File → Host Network Manager**.
- [ ] **2.2** Click **Create** to add a second Host-only Network adapter (it will likely be named `vboxnet1`). Your existing network is probably `vboxnet0`.
- [ ] **2.3** Give `vboxnet1` its own IP range that does **not** overlap with `vboxnet0` (e.g., if `vboxnet0` is `192.168.56.0/24`, set `vboxnet1` to `192.168.57.0/24`).
- [ ] **2.4** Leave DHCP disabled on both host-only networks for now — you'll use static IP addresses for clarity in the lab.

## Part 3 — Build the firewall VM

- [ ] **3.1** In VirtualBox, click **New** to create a VM for the firewall (e.g., name it `pfSense-FW`). Allocate at least 1 GB RAM and 1 CPU core (2 GB RAM is safer if your host allows it).
- [ ] **3.2** Attach the pfSense/OPNsense ISO as the optical drive so it boots into the installer.
- [ ] **3.3** Go to the VM's **Settings → Network**:
  - **Adapter 1**: Attached to **Host-only Adapter** → `vboxnet0` (this becomes the WAN side, facing Kali).
  - **Adapter 2**: Enable it, set Attached to **Host-only Adapter** → `vboxnet1` (this becomes the LAN side, facing the DC and Client).
- [ ] **3.4** Start the VM and run through the pfSense/OPNsense installer using default options. Reboot when prompted (remove the ISO from the optical drive first).
- [ ] **3.5** At the console menu after install, confirm/assign interfaces:
  - WAN = the NIC on `vboxnet0`
  - LAN = the NIC on `vboxnet1`
- [ ] **3.6** From the console menu, set the **LAN interface's IP address** (e.g., `192.168.57.1/24`). This will be the "gateway" IP that the DC and Client will point to later.

## Part 4 — Move the existing VMs onto the right segments

- [ ] **4.1** Shut down the Kali Linux VM. In **Settings → Network → Adapter 1**, confirm it's attached to `vboxnet0` (the Outside/WAN segment) — it likely already is, since that's your original network.
- [ ] **4.2** Shut down the Windows Server 2022 (DC) VM. In **Settings → Network → Adapter 1**, change it from `vboxnet0` to `vboxnet1` (the Inside/LAN segment).
- [ ] **4.3** Shut down the Windows 10 (Client) VM. Change its **Adapter 1** the same way, from `vboxnet0` to `vboxnet1`.
- [ ] **4.4** Start the DC and Client VMs. Update their static IP addresses and default gateway to match the new `vboxnet1` range, using the firewall's LAN IP (e.g., `192.168.57.1`) as the gateway.
- [ ] **4.5** Update the DNS settings on the DC and Client if needed so they still point to the DC for name resolution (DNS should keep pointing at the DC's new IP, not the firewall).

## Part 5 — Access the firewall's web interface

- [ ] **5.1** From the Windows 10 Client VM, open a browser and go to the firewall's LAN IP (e.g., `https://192.168.57.1`).
- [ ] **5.2** Log in with the default credentials (pfSense default: `admin` / `pfsense`) and immediately change the password.
- [ ] **5.3** Run through the setup wizard (hostname, time zone, interface confirmation).

## Part 6 — Create basic firewall rules

By default, pfSense/OPNsense blocks everything on WAN and allows everything outbound on LAN. You want to make this more intentional:

- [ ] **6.1** Go to **Firewall → Rules → LAN**. Review the existing "allow all" rule.
- [ ] **6.2** Add a rule allowing the Client to reach the DC on the ports AD/DNS needs (DNS: port 53, Kerberos: port 88, LDAP: port 389) — or for a first pass, just allow all LAN-to-LAN traffic and tighten later.
- [ ] **6.3** Go to **Firewall → Rules → WAN**. Add a rule that allows Kali to reach only specific IP/ports on the LAN side (e.g., just RDP or just the DC's IP), instead of leaving WAN fully open or fully closed. This gives you a controlled "attack path" to test detection later.
- [ ] **6.4** Enable logging on each rule you create (there's a "Log packets that are handled by this rule" checkbox) so you can review hits later.

## Part 7 — Test the segmentation

- [ ] **7.1** From Kali, try to ping the DC's new IP address. Confirm it behaves as expected based on your WAN rule (blocked, or allowed only on the port you opened).
- [ ] **7.2** From the Client, confirm it can still reach the DC (browse a shared folder, run `nslookup` against the DC, or log in to the domain) — this proves LAN-side rules aren't breaking normal AD function.
- [ ] **7.3** In the firewall GUI, go to **Status → System Logs → Firewall** and confirm you can see the test traffic you just generated.

## Part 8 — Wrap up

- [ ] **8.1** Take a fresh snapshot of all four VMs (DC, Client, Kali, Firewall) now that the segmented network is working.
- [ ] **8.2** Update your network diagram/documentation if you changed any IP addresses from the plan.
- [ ] **8.3** Write a short note (for your portfolio repo) describing what you built and why — this becomes the write-up for Tier 1.

---

**Next step:** Tier 2 — deploy a SIEM (Security Information and Event Management system) and forward these firewall logs into it.
