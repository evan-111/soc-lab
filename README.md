# SOC Lab

A multi-node Security Operations Center (SOC) lab simulating enterprise-grade security infrastructure using virtualization. Built on Proxmox VE with segmented networks, SIEM, SOAR, and Active Directory for blue team training and detection validation.

---

## Hardware

| Component | Specs |
|---|---|
| **Dell OptiPlex 7070 (x2 nodes)** | Intel i5-9500 (6C/6T), 64 GB RAM, 2 TB storage each |
| **TP-Link SG108E** | 8-port managed switch (VLAN support) |
| **Attacker machine** | Kali Linux (laptop) |

---

## Architecture

```
                            ISP Router
                                |
                           WAN (VLAN 10)
                                |
                         TP-Link SG108E
                      (Managed Switch)
                    Port 1: ISP Router
               Port 2: Node 1 (trunk, VLANs 10,20,30,40,50,60)
               Port 3: Node 2 (trunk, VLANs 10,20,30,40,50,60)
                          /              \
                         /                \
                        /                  \
                   Node 1                  Node 2
              (Management Stack)     (Network & Endpoints)
              ┌─────────────────┐   ┌──────────────────────────┐
              │ OPNSense        │   │ VYOS                     │
              │ Wazuh SIEM      │   │ Windows Server AD        │
              │ Shuffle SOAR    │   │ Windows 11 Pro           │
              └─────────────────┘   │ Ubuntu Server            │
                                    │ Honeypot (DMZ)           │
                                    │ DVWA / Juice Shop (DMZ)  │
                                    └──────────────────────────┘
```

### Data Flow

```
Internet -> OPNSense (firewall/NAT) -> VYOS (internal router)
                                              |
                ┌─────────────────────────────┼─────────────────────────────┐
                │                             │                             │
           VLAN 30 DMZ                  VLAN 40 INT-LAN               VLAN 50 SOC
       (Honeypot, DVWA)              (Ubuntu, Win 11, Kali)       (Wazuh, Shuffle)
                                              │
                                         VLAN 60 AD
                                       (Windows Server)
```

---

## Network Segmentation

| VLAN | ID | Subnet | Purpose |
|---|---|---|---|
| WAN | 10 | DHCP (ISP) | Internet-facing, OPNSense WAN |
| MGMT | 20 | 10.0.20.0/24 | Proxmox host management |
| DMZ | 30 | 10.0.30.0/24 | Honeypot, DVWA, Juice Shop |
| INT-LAN | 40 | 10.0.40.0/24 | Ubuntu Server, Windows 11, Kali |
| SOC | 50 | 10.0.50.0/24 | Wazuh, Shuffle SOAR |
| AD | 60 | 10.0.60.0/24 | Windows Server (Active Directory) |

---

## VM Distribution

### Node 1 - Management Stack

| VM | vCPU | RAM | Disk | Purpose |
|---|---|---|---|---|
| OPNSense | 2 | 4 GB | 20 GB | Edge firewall, NAT, DHCP |
| Wazuh | 6 | 16 GB | 200 GB | SIEM (indexer + server + dashboard) |
| Shuffle SOAR | 4 | 8 GB | 80 GB | Automated incident response |

### Node 2 - Network & Endpoints

| VM | vCPU | RAM | Disk | Purpose |
|---|---|---|---|---|
| VYOS | 1 | 2 GB | 10 GB | Internal router, inter-VLAN routing |
| Windows Server AD | 2 | 8 GB | 80 GB | Domain controller, DNS, DHCP |
| Windows 11 Pro | 2 | 4 GB | 60 GB | Domain-joined endpoint |
| Ubuntu Server | 1 | 4 GB | 40 GB | General purpose / services |
| Honeypot | 1 | 2 GB | 30 GB | Attack surface / threat intel |
| DVWA / Juice Shop | 1 | 4 GB | 20 GB | Web app vulnerability targets |

---

## Tech Stack

| Category | Tools |
|---|---|
| **Hypervisor** | Proxmox VE (clustered, 2 nodes) |
| **Firewall** | OPNSense |
| **Routing** | VYOS |
| **SIEM** | Wazuh (indexer + server + dashboard) |
| **SOAR** | Shuffle |
| **Directory Services** | Windows Server Active Directory |
| **Endpoint** | Windows 11, Ubuntu |
| **Attack Platform** | Kali Linux |
| **Web Targets** | DVWA, Juice Shop |
| **Honeypot** | T-Pot / Cowrie (Docker) |
| **IDS/IPS** | Suricata |

---

## Setup Overview

1. Install Proxmox VE on both Dell nodes
2. Configure VLANs on TP-Link SG108E (trunk ports)
3. Create Linux bridges on Proxmox for each VLAN
4. Deploy OPNSense - WAN on VLAN 10, LAN on native MGMT VLAN
5. Deploy VYOS - upstream to OPNSense, routes all internal VLANs
6. Deploy Wazuh (Ubuntu VM) - all-in-one install
7. Deploy Shuffle SOAR (Ubuntu VM) - Docker compose
8. Deploy Windows Server AD - domain controller
9. Deploy endpoint VMs - Windows 11, Ubuntu, DMZ targets
10. Install Wazuh agents on all endpoints
11. Configure syslog forwarding from OPNSense and VYOS to Wazuh
12. Build SOAR playbooks - auto-block malicious IPs via OPNSense API

---

## Attack Scenarios

| Scenario | Source | Target | Detection | Response |
|---|---|---|---|---|
| Port scan | Kali (VLAN 40) | DMZ hosts | Wazuh alert | Shuffle blocks IP on OPNSense |
| Web app attack | Kali | DVWA / Juice Shop | Wazuh + Suricata | Alert, IP block |
| Brute force | Kali | Windows AD | Wazuh (Windows event logs) | Account lockout alert |
| Lateral movement | Compromised endpoint | Internal servers | Wazuh (Sysmon) | Host isolation playbook |

---

## Future Plans

- Integrate Jetson Orin Nano as inline AI-based IDS/IPS
- Deploy Security Onion for full packet capture and network monitoring
- Add Proxmox Backup Server for VM backups
- Implement VPN access for external attack simulations
