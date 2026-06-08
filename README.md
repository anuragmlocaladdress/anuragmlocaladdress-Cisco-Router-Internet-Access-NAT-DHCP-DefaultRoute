# 🌐 Cisco Router — Internet Access Configuration

### Static WAN IP | DHCP WAN | Default Route | Internal DHCP | NAT Overload (PAT)

### Cisco IOS Router — ISP Connectivity with Internal LAN

![Cisco IOS](https://img.shields.io/badge/Cisco_IOS-15.x%2B-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![NAT](https://img.shields.io/badge/NAT-PAT_Overload-FF8300?style=for-the-badge&logo=cisco&logoColor=white)
![DHCP](https://img.shields.io/badge/DHCP-IOS_Server-0F6E56?style=for-the-badge)
![EVE-NG](https://img.shields.io/badge/Lab-EVE--NG-333333?style=for-the-badge)
![Routing](https://img.shields.io/badge/Routing-Default_Route-1BA0D7?style=for-the-badge)

**Author:** Anurag Mishra |  Solution Architect - IT & Security  
📍 Hyderabad, India | [LinkedIn](https://www.linkedin.com/in/anuragmishra6) | [GitHub](https://github.com/anuragmlocaladdress)

---

## 🗺️ Lab Topology
<img width="877" height="272" alt="image" src="https://github.com/user-attachments/assets/81efbefa-b24f-4c20-bda1-ca1756b9ff01" />



---

## 📋 Table of Contents

1. [Lab Overview](#1--lab-overview)
2. [IP Addressing](#2--ip-addressing)
3. [Key Concepts](#3--key-concepts)
4. [Phase 1 — WAN Interface Configuration](#4--phase-1--wan-interface-configuration)
5. [Phase 2 — Default Route to ISP](#5--phase-2--default-route-to-isp)
6. [Phase 3 — LAN Interface Configuration](#6--phase-3--lan-interface-configuration)
7. [Phase 4 — Internal DHCP Server](#7--phase-4--internal-dhcp-server)
8. [Phase 5 — NAT Overload (PAT)](#8--phase-5--nat-overload-pat)
9. [Phase 6 — Verification Commands](#9--phase-6--verification-commands)
10. [How It All Works Together](#10--how-it-all-works-together)
11. [Lessons Learned & Common Issues](#11--lessons-learned--common-issues)
12. [References](#-references)

---

## 1. 🧠 Lab Overview

This runbook walks through configuring a **Cisco IOS router for full internet access** — from WAN interface assignment to NAT PAT, giving internal hosts routed internet connectivity through a single public IP. Built and verified in EVE-NG.

| Feature               | Technology Used                              |
|-----------------------|----------------------------------------------|
| WAN IP Assignment     | Static ISP-assigned IP or DHCP from ISP      |
| Default Route         | Static route pointing to ISP next-hop        |
| Internal Gateway      | IOS physical interface SVI                   |
| Internal DHCP         | Cisco IOS DHCP server (pool on router)       |
| Internet for LAN      | NAT Overload / PAT — single public IP shared |
| Access Control        | Extended ACL to define NAT-eligible traffic  |

---

## 2. 📡 IP Addressing

| Device / Interface   | Role                          | IP Address              |
|----------------------|-------------------------------|-------------------------|
| R1 — E0/1 (WAN)      | ISP-facing / NAT Outside      | 165.13.70.66 /30        |
| ISP Router           | ISP gateway / next-hop        | 165.13.70.65 /30        |
| WAN Subnet           | Point-to-point ISP link       | 165.13.70.64 /30        |
| R1 — E0/0 (LAN)      | Internal gateway / NAT Inside | 172.16.10.1 /24         |
| Internal Devices     | DHCP clients (Win PC)         | 172.16.10.11–254        |
| DNS (public)         | Google DNS / Level3           | 8.8.8.8 / 4.2.2.2       |

---

## 3. 💡 Key Concepts

**WAN Interface — Static vs DHCP**

Some ISPs hand you a fixed public IP (static). Others assign it dynamically via DHCP on the WAN link. In both cases the WAN interface (`E0/1`) is the **NAT Outside** boundary — all traffic toward the internet exits here.

**Default Route (0.0.0.0/0)**

The router has no knowledge of all internet prefixes. A default route tells it: *"if you have no specific match, forward to the ISP."* This is the single most critical line for internet reachability from the router itself.

**NAT Overload / PAT**

Internal devices use RFC 1918 private IPs (`172.16.10.x`) that are not internet-routable. NAT PAT translates all internal source IPs to the **single public WAN IP**, differentiating sessions by port number. Hundreds of hosts share one public IP simultaneously.

**DHCP Server on Router**

Cisco IOS can act as a DHCP server for the LAN. `excluded-address` reserves a range for static assignments (gateway, printers, servers). The pool defines the assignable range, gateway, and DNS pushed to clients.

**WAN Interface Hardening**

`no ip proxy-arp` — prevents router from replying to ARP on behalf of other hosts toward ISP.  
`no ip redirects` — stops router from sending ICMP redirects to ISP.  
`no ip unreachables` — prevents ICMP unreachable messages leaking topology info to outside.

---

## 4. ⚙️ Phase 1 — WAN Interface Configuration

> Configure **Ethernet 0/1** as the ISP-facing interface. Use **Option A** if ISP gave you a fixed IP, **Option B** if ISP assigns dynamically.

### Option A — Static IP (ISP assigned fixed public IP)

```text
Router(config)#interface Ethernet 0/1
Router(config-if)# description INTERNET LINE
Router(config-if)# ip address 165.13.70.66 255.255.255.252
Router(config-if)# no ip proxy-arp
Router(config-if)# no ip redirects
Router(config-if)# no ip unreachables
Router(config-if)# no shutdown
Router(config-if)# exit
```

### Option B — Dynamic IP (ISP uses DHCP on WAN link)

```text
Router(config)#interface Ethernet 0/1
Router(config-if)# description INTERNET LINE
Router(config-if)# ip address dhcp
Router(config-if)# no ip proxy-arp
Router(config-if)# no ip unreachables
Router(config-if)# no ip redirects
Router(config-if)# no shutdown
Router(config-if)# exit
```

> ✅ WAN interface configured — router has a public-facing IP on E0/1

---

## 5. ⚙️ Phase 2 — Default Route to ISP

> Tell the router to forward all traffic with no specific match toward the ISP next-hop.

```text
Router(config)#ip route 0.0.0.0 0.0.0.0 165.13.70.65
Router(config)#end
Router#
```

> ✅ Default route installed — all unmatched traffic forwarded to ISP gateway `165.13.70.65`

---

## 6. ⚙️ Phase 3 — LAN Interface Configuration

> Configure **Ethernet 0/0** as the internal default gateway for `172.16.10.0/24`.

```text
Router(config)#interface Ethernet 0/0
Router(config-if)# description INTERNAL LINE
Router(config-if)# ip address 172.16.10.1 255.255.255.0
Router(config-if)# no shutdown
Router(config-if)# exit
```

> ✅ LAN interface up — internal hosts can use `172.16.10.1` as their default gateway

---

## 7. ⚙️ Phase 4 — Internal DHCP Server

> Configure the router to automatically assign IPs to LAN devices via DHCP.

### Step 4.1 — Exclude Static IPs from the Pool

```text
! Reserve .1 through .10 — router, servers, printers, static devices
Router(config)#ip dhcp excluded-address 172.16.10.1 172.16.10.10
```

### Step 4.2 — Create the DHCP Pool

```text
Router(config)#ip dhcp pool INSIDE
Router(dhcp-config)# network 172.16.10.0 255.255.255.0
Router(dhcp-config)# default-router 172.16.10.1
Router(dhcp-config)# dns-server 8.8.8.8 4.2.2.2
Router(dhcp-config)# exit
Router(config)#exit
Router#
```

> ✅ DHCP server active — clients will receive `172.16.10.11` onward with gateway and DNS

---

## 8. ⚙️ Phase 5 — NAT Overload (PAT)

> Translate all internal private IPs to the single public WAN IP when accessing the internet.

### Step 5.1 — Mark NAT Direction on Each Interface

```text
Router(config)#interface Ethernet 0/0
Router(config-if)# ip nat inside
Router(config-if)# exit

Router(config)#interface Ethernet 0/1
Router(config-if)# ip nat outside
Router(config-if)# exit
```

### Step 5.2 — Define NAT-Eligible Traffic Using an Extended ACL

```text
Router(config)#ip access-list extended NAT-TRAFFIC
Router(config-ext-nacl)# permit ip 172.16.10.0 0.0.0.255 any
Router(config-ext-nacl)# exit
```

### Step 5.3 — Enable NAT Overload Using the WAN Interface IP

```text
Router(config)#ip nat inside source list NAT-TRAFFIC interface Ethernet 0/1 overload
Router(config)#end
Router#
```

> ✅ NAT PAT active — all `172.16.10.x` hosts appear as `165.13.70.66` on the internet

---

## 9. 🔍 Phase 6 — Verification Commands

### Interface Status

```text
Router#show ip interface brief
```

Expected output:

```text
Interface        IP-Address       OK?  Method  Status     Protocol
Ethernet0/0      172.16.10.1      YES  manual  up         up
Ethernet0/1      165.13.70.66     YES  manual  up         up
```

| Check       | Expected        |
|-------------|-----------------|
| E0/0 Status | `up` / `up`     |
| E0/1 Status | `up` / `up`     |
| E0/1 IP     | `165.13.70.66`  |
| E0/0 IP     | `172.16.10.1`   |

---

### Routing Table

```text
Router#show ip route
```

Look for the default route:
```text
S*   0.0.0.0/0 [1/0] via 165.13.70.65
```

---

### Ping ISP Next-Hop (Layer 3 reachability to ISP)

```text
Router#ping 165.13.70.65
```

Expected:
```text
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 165.13.70.65, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 23/187/293 ms
```

---

### Ping Internet (Google DNS — full internet reachability)

```text
Router#ping 8.8.8.8
```

Expected:
```text
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 14/38/94 ms
```

---

### DHCP Pool Status

```text
Router#show ip dhcp pool
Router#show ip dhcp binding
```

Expected pool output:
```text
Pool INSIDE :
 Total addresses      : 254
 Leased addresses     : 0
 Current index        IP address range              Leased addresses
 172.16.10.1          172.16.10.1 - 172.16.10.254   0
```

---

### NAT Statistics

```text
Router#show ip nat statistics
```

Expected:
```text
Outside interfaces:  Ethernet0/1
Inside interfaces:   Ethernet0/0
Hits: 200  Misses: 0
Dynamic mappings:
-- Inside Source
[Id: 1] access-list NAT-TRAFFIC interface Ethernet0/1 refcount 0
```

---

### Active NAT Translations

```text
Router#show ip nat translations
```

Expected:
```text
Pro  Inside global      Inside local       Outside local   Outside global
icmp 165.13.70.66:0    172.16.10.11:0     8.8.8.8:0       8.8.8.8:0
```

---

## 10. 🔄 How It All Works Together

```
  Win PC (172.16.10.11)
        |
        | Sends packet to 8.8.8.8
        | Default gateway → 172.16.10.1 (E0/0)
        |
  Router [E0/0 — NAT Inside]
        |
        | ACL NAT-TRAFFIC matches 172.16.10.0/24 → any
        | NAT PAT: src 172.16.10.11 → 165.13.70.66 (port mapped)
        |
  Router [E0/1 — NAT Outside]
        |
        | Default route → 165.13.70.65 (ISP next-hop)
        |
  ISP → Internet → 8.8.8.8
        |
        | Reply returns to 165.13.70.66
        |
  Router: NAT table match → un-translate to 172.16.10.11
        |
  Win PC receives reply ✅
```

---

## 11. ⚠️ Lessons Learned & Common Issues

| # | Symptom                                   | Root Cause                                    | Fix                                                              |
|---|-------------------------------------------|-----------------------------------------------|------------------------------------------------------------------|
| 1 | Router cannot ping 8.8.8.8                | Default route missing                         | `ip route 0.0.0.0 0.0.0.0 165.13.70.65`                         |
| 2 | Router pings ISP but not internet         | NAT not configured                            | Add `ip nat inside/outside` + ACL + overload statement           |
| 3 | Internal hosts get no IP from DHCP        | Pool not created or excluded range too large  | Verify `show ip dhcp pool` — check network/mask match            |
| 4 | NAT translations table stays empty        | ACL wildcard wrong or `nat inside` missing    | Wildcard for /24 is `0.0.0.255` — verify ACL and interface tags  |
| 5 | NAT overload interface name mismatch      | Wrong interface name in NAT statement         | Must exactly match: `interface Ethernet 0/1` not `e0/1`          |
| 6 | E0/1 stays down after `no shutdown`       | ISP/upstream not connected in EVE-NG          | Check cloud/Net node link in EVE-NG topology                     |
| 7 | DHCP clients get IP but no internet       | NAT not configured or ACL not matching        | Verify `show ip nat statistics` — check Hits counter increments  |
| 8 | `ip address dhcp` shows unassigned        | ISP DHCP server not reachable                 | Confirm upstream cloud node is running and passing DHCP          |

---

## 📚 References

- [Cisco IOS NAT Configuration Guide](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13772-12.html)
- [Cisco IOS DHCP Server Configuration](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_dhcp/configuration/15-sy/dhcp-15-sy-book/config-dhcp-server.html)
- [Cisco IOS Static Routing Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_static/configuration/15-sy/irs-15-sy-book/irs-staticroutes.html)
- [Understanding NAT Overload (PAT)](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13706-9.html)

---

**Anurag Mishra** — Technology Specialist | Network Implementation Engineer  
📍 Hyderabad, India

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/anuragmishra6)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-333?style=for-the-badge&logo=github&logoColor=white)](https://github.com/anuragmlocaladdress)

*All configurations are from a lab environment built in EVE-NG. Sanitize IPs and credentials before production use.*
