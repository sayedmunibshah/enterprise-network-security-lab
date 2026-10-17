# Project 01 — Secure Enterprise Network

**Platform:** Cisco Packet Tracer  
**Status:** Implemented and verified in simulation  
**Devices:** Cisco 2911 router, two Cisco 2960 switches, four PCs, one server

## 1. Overview

This project simulates a small enterprise LAN divided into separate VLANs for ADMIN, IT, STAFF, GUEST, SERVERS, and MANAGEMENT. A Cisco 2911 router provides inter-VLAN routing using router-on-a-stick subinterfaces. The router also provides DHCP for user VLANs and applies an extended ACL to block guest traffic to selected internal networks.

The switches use SSH version 2 for remote management, with a standard ACL that permits management access from the IT subnet. Unused switch access ports are administratively disabled. An internal server provides an HTTP intranet page and DNS resolution for `intranet.company.local`.

This is a **Packet Tracer simulation**, not a production deployment or a test on physical hardware.

## 2. Objectives

- Create departmental VLANs and an IP addressing plan.
- Configure 802.1Q trunks and router-on-a-stick routing.
- Provide DHCP to user endpoints.
- Restrict guest access to selected internal networks.
- Restrict switch SSH management to the IT subnet.
- Disable unused switch ports.
- Configure and verify internal HTTP and DNS services.
- Record repeatable tests and evidence.

## 3. Topology and cabling

| From | To | Purpose |
|---|---|---|
| R1 G0/0 | SW1 G0/1 | Router-to-switch trunk |
| SW1 G0/2 | SW2 G0/1 | Inter-switch trunk |
| SW1 Fa0/2 | PC-ADMIN Fa0 | ADMIN endpoint |
| SW1 Fa0/3 | PC-IT Fa0 | IT endpoint |
| SW1 Fa0/4 | SRV1 Fa0 | Server endpoint |
| SW2 Fa0/2 | PC-GUEST Fa0 | GUEST endpoint |
| SW2 Fa0/3 | PC-STAFF Fa0 | STAFF endpoint |

**Evidence:** `evidence/01-topology.png`

## 4. VLAN and IP plan

| VLAN | Name | Subnet | Gateway |
|---:|---|---|---|
| 10 | ADMIN | `192.168.10.0/24` | `192.168.10.1` |
| 20 | IT | `192.168.20.0/24` | `192.168.20.1` |
| 30 | STAFF | `192.168.30.0/24` | `192.168.30.1` |
| 40 | GUEST | `192.168.40.0/24` | `192.168.40.1` |
| 50 | SERVERS | `192.168.50.0/24` | `192.168.50.1` |
| 99 | MANAGEMENT | `192.168.99.0/24` | `192.168.99.1` |
| 999 | NATIVE_UNUSED | No IP | None |

### Key addresses observed

| Device | Address | Method |
|---|---|---|
| R1 VLAN gateways | `192.168.<VLAN>.1/24` | Static |
| SW1 management SVI | `192.168.99.2/24` | Static |
| SW2 management SVI | `192.168.99.3/24` | Static |
| PC-ADMIN | `192.168.10.21/24` | DHCP |
| PC-IT | `192.168.20.21/24` | DHCP |
| PC-STAFF | `192.168.30.21/24` | DHCP |
| PC-GUEST | `192.168.40.21/24` | DHCP |
| SRV1 | `192.168.50.10/24` | Static |

The first 20 addresses in each user subnet were excluded from DHCP allocation. DHCP pools serve VLANs 10, 20, 30, and 40. SRV1 uses a static address.

## 5. Implemented controls and services

### VLANs and trunks
VLANs 10, 20, 30, 40, 50, 99, and 999 were created on both switches. Required access ports were assigned to the correct VLANs. Trunks use native VLAN 999 and allow `10,20,30,40,50,99,999`.

### Inter-VLAN routing
R1 uses subinterfaces on `G0/0` for VLANs 10, 20, 30, 40, 50, and 99. VLAN 999 is configured as the native VLAN without an IP address.

### DHCP
R1 provides DHCP for ADMIN, IT, STAFF, and GUEST. Each pool supplies a gateway and DNS server `192.168.50.10`.

### Guest ACL
The extended ACL `GUEST-INBOUND` is applied inbound on `G0/0.40`. It denies guest-sourced traffic to:
- `192.168.10.0/24` (ADMIN)
- `192.168.20.0/24` (IT)
- `192.168.30.0/24` (STAFF)
- `192.168.50.0/24` (SERVERS)
- `192.168.99.0/24` (MANAGEMENT)

A final permit rule allows guest traffic to other destinations not matched above. This does not prove Internet connectivity; no Internet/WAN was configured.

### Switch management
SSH version 2 is enabled on SW1 and SW2. VTY lines use local authentication and SSH transport. The standard ACL `IT-MANAGEMENT` permits source addresses in `192.168.20.0/24` and denies other sources. PC-IT successfully SSH-connected to both switches.

### Unused ports
Unused access ports were administratively disabled. The required endpoint and trunk interfaces remain connected.

### Internal HTTP and DNS
SRV1 is `192.168.50.10/24`, gateway `192.168.50.1`, and DNS server `192.168.50.10`. HTTP and DNS are enabled. The A record `intranet.company.local` resolves to `192.168.50.10`.

## 6. Verification commands

Run these in the **CLI tab of the relevant Packet Tracer device**. They are Cisco IOS commands, not Ubuntu shell commands.

### On R1
```text
enable
show ip interface brief
show ip route
show ip dhcp binding
show access-lists GUEST-INBOUND
show running-config
```

### On SW1 and SW2
```text
enable
show vlan brief
show interfaces trunk
show interfaces status
show ip interface brief
show ip ssh
show access-lists IT-MANAGEMENT
show running-config
```

### From PC-ADMIN (Desktop > Command Prompt)
```text
ipconfig
ping 192.168.10.1
ping 192.168.20.21
ping 192.168.50.10
```

Then open PC-ADMIN > Desktop > Web Browser and test:
```text
http://192.168.50.10
http://intranet.company.local
```

### From PC-GUEST (Desktop > Command Prompt)
```text
ping 192.168.40.1
ping 192.168.10.21
ping 192.168.50.10
```

Expected: the guest gateway responds; the ADMIN PC and server tests are blocked by the guest ACL.

## 7. Test results

| Test | Expected | Observed | Result |
|---|---|---|---|
| ADMIN DHCP | Address in VLAN 10 | `192.168.10.21/24`, gateway `192.168.10.1` | PASS |
| ADMIN to gateway | Success | 4/4 replies | PASS |
| ADMIN to IT PC | Success | 4/4 replies | PASS |
| ADMIN to server | Success | 3/4 replies; first packet timed out | PASS with initial packet loss |
| HTTP by IP | Page loads | Successful | PASS |
| HTTP by DNS name | Page loads | Successful | PASS |
| GUEST to gateway | Success | 4/4 replies | PASS |
| GUEST to ADMIN PC | Blocked | `Destination host unreachable` from `192.168.40.1` | PASS |
| GUEST to server | Blocked | `Destination host unreachable` from `192.168.40.1` | PASS |
| Switch SSH status | SSH v2 enabled | Confirmed on both switches | PASS |
| Management ACL | IT subnet permitted; others denied | ACL shown and IT match counters observed | PASS |
| Required switch links | Connected | Access links and trunks connected | PASS |
| Unused access ports | Disabled | Confirmed in interface status | PASS |

The first ping to the server timed out, while the following three succeeded. This is consistent with initial ARP resolution in a simulator and was not persistent packet loss in this test.

## 8. Evidence screenshots

Capture the following screenshots from Packet Tracer. Keep screenshots readable and crop out unrelated desktop content.

1. `01-topology.png` — entire topology and device labels.
2. `02-vlan-and-trunks.png` — `show vlan brief` and `show interfaces trunk`.
3. `03-routing-and-dhcp.png` — R1 `show ip route`, `show ip interface brief`, and `show ip dhcp binding`.
4. `04-guest-acl-blocking.png` — guest gateway success and blocked pings to ADMIN/server.
5. `05-ssh-management-acl.png` — `show ip ssh` and `show access-lists IT-MANAGEMENT` on switches.
6. `06-intranet-http-dns.png` — browser showing intranet by DNS name; optionally also capture IP URL.
7. `07-disabled-ports.png` — `show interfaces status` showing unused ports disabled.

Do not include passwords, private keys, or other secrets in screenshots. If a configuration view contains credentials, crop or redact that part.

## 9. Limitations and possible next steps

- The network is simulated in Packet Tracer; it is not a physical or production deployment.
- No WAN or Internet connectivity is configured.
- The guest ACL blocks specified internal subnets but permits other destinations. A stricter future policy could explicitly permit required guest services and deny other traffic.
- No dedicated firewall, IDS/IPS, centralized logging, high availability, or redundant routing is implemented.
- Tests cover the scenarios listed above, not every possible traffic flow or a full security assessment.
- Further versions could add monitoring, redundancy, more explicit guest policies, and automated configuration checks.

## 10. Repository structure

```text
01-secure-enterprise-network/
├── README.md
├── 01-Secure-Enterprise-Network.pkt
├── evidence/
│   ├── 01-topology.png
│   ├── 02-vlan-and-trunks.png
│   ├── 03-routing-and-dhcp.png
│   ├── 04-guest-acl-blocking.png
│   ├── 05-ssh-management-acl.png
│   ├── 06-intranet-http-dns.png
│   └── 07-disabled-ports.png
└── Project-01-Secure-Enterprise-Network-Documentation.pdf
```

The `.pkt`, README, and screenshots are the essential first version. Configuration exports are optional; review and sanitize them before publishing.

## 11. Why this project is useful

This project demonstrates foundational enterprise networking tasks: segmentation, routing, address allocation, access control, secure device administration, service configuration, and verification. It gives a reviewer evidence that the design was tested, not merely drawn.

It is a suitable foundation for future projects involving enterprise WAN routing, branch connectivity, network hardening, monitoring, and automation.

## 12. Conclusion

The simulated enterprise network successfully separates user groups into VLANs, routes between VLANs, assigns user addresses through DHCP, restricts guest access to selected internal networks, limits switch SSH management to the IT subnet, disables unused ports, and serves an internal webpage through HTTP and DNS.

**Artifact:** `01-Secure-Enterprise-Network.pkt`  
**Environment:** Cisco Packet Tracer  
**Status:** Selected connectivity, service, and security-policy tests completed.
