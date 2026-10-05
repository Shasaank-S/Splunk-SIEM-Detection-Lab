# Lab Setup

How the lab was built. Networking and results are covered in the [README](README.md); this page focuses on the virtual machines and how each component was installed and configured, at a high level.

## Virtual machines

| VM | Role | Key details |
|---|---|---|
| **pfSense** | Firewall, router, IDS host | 4 adapters: NAT (WAN), RED, BLUE, Host-Only (MGMT) |
| **Kali Linux** | Attacker | 1 adapter on RED; extra adapters disabled so it can't bypass pfSense |
| **Ubuntu** | Target + Splunk server | 2 adapters (BLUE, MGMT); 6 vCPU / 6 GB RAM |
| **Windows 11** (physical host) | VirtualBox host + monitored endpoint | Sysmon + Splunk Universal Forwarder |

All VMs use static IPs on VirtualBox **Internal Networks** (RED, BLUE) and a **Host-Only** network (MGMT), so the segments stay isolated and attacker traffic is forced through pfSense.

## pfSense (firewall + router)

1. Install pfSense as a VM and assign the four interfaces: WAN (NAT), LAN/RED (`10.10.10.1`), OPT1/BLUE (`10.20.20.1`), OPT2/MGMT (`192.168.56.254`).
2. Give each interface its static address and reach the WebGUI from the Windows host at `https://192.168.56.254`.
3. Add firewall rules: RED can reach the Ubuntu target, BLUE can reach the Internet, and MGMT can reach the WebGUI.
4. Enable remote syslog to forward firewall and system logs to Ubuntu.

![pfSense dashboard and interfaces](screenshots/03-pfsense-interfaces-dashboard.png)

The firewall rules per interface: BLUE (OPT1) is allowed outbound, and MGMT (OPT2) can only reach the firewall's WebGUI over HTTPS and ping it.

<p align="center">
  <img src="screenshots/04-pfsense-blue-firewall-rule.png" width="49%" alt="pfSense OPT1 (BLUE) rule: allow BLUE outbound">
  <img src="screenshots/05-pfsense-management-rules.png" width="49%" alt="pfSense OPT2 (MGMT) rules: HTTPS and ICMP to the firewall only">
</p>

## Splunk Enterprise (Ubuntu)

1. Install Splunk under `/opt/splunk` and run it as a dedicated `splunk` user instead of root.
2. Enable the receiving port `9997` and Splunk Web on `8000`.
3. Create four indexes: `windows`, `linux`, `suricata`, `firewall`.
4. Add the `splunk` user to the `adm` group so it can read `/var/log` (auth, syslog, Apache).

## Splunk Universal Forwarder (Windows 11)

1. Install Sysmon and the Splunk Universal Forwarder on the Windows host.
2. Point the forwarder at the Ubuntu receiver, `192.168.56.20:9997`.
3. Forward Security, System, Microsoft Defender, and Sysmon event logs into `index=windows`.

![Forwarder and Sysmon running](screenshots/18-windows-forwarder-services.png)

## Suricata IDS (pfSense)

1. Install the Suricata package from the pfSense package manager and enable the ET Open rules.
2. Disable hardware offloading so the engine inspects clean packets.
3. Create a Suricata instance on the RED-facing interface (`em1`) in IDS mode.
4. Set EVE JSON output to **syslog** so alerts flow to Ubuntu and into `index=suricata`.

![Suricata installed on pfSense](screenshots/06-pfsense-suricata-package.png)

Once all four pieces are running, generate traffic from Kali and confirm it appears in Splunk. The [README](README.md) walks through the Nmap, Gobuster, and Hydra simulations and the evidence each one produced.
