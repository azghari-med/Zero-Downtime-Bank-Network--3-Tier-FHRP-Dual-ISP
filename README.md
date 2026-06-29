<div align="center">

# 🏦 Zero-Downtime Bank Network — 3-Tier · FHRP · Dual-ISP

### Access / Distribution / Core · OSPF · FHRP (HSRP·VRRP·GLBP) · DHCP Relay (Redundant) · Dual-ISP IP SLA Failover

`OSPF` · `HSRP` · `VRRP` · `GLBP` · `IP Helper` · `IP SLA` · `3-Tier` · `STP Root Alignment` · `High Availability`

!\[Cisco](https://img.shields.io/badge/Cisco-1BA0D7?style=for-the-badge\&logo=cisco\&logoColor=white)
!\[HA](https://img.shields.io/badge/High\_Availability-success?style=for-the-badge)
!\[PNetLab](https://img.shields.io/badge/PNetLab-2D9CDB?style=for-the-badge)
!\[Status](https://img.shields.io/badge/Status-Working-success?style=for-the-badge)

*A bank loses money every second it's offline. This is a real enterprise hierarchy built so it never does.*

</div>

\---

## 🎯 The Scenario

**Northbridge Bank** runs a textbook **three-tier enterprise network** — access, distribution, core — with full OSPF routing, gateway redundancy, redundant DHCP, and dual-internet failover. The rule is absolute: **no downtime.** A switch dies, a DHCP server dies, an ISP drops — and a teller never sees a spinning wheel.

```
ACCESS        →  pure L2: VLANs, access ports, trunks, port security.
DISTRIBUTION  →  L3 SVIs, FHRP gateways (HSRP/VRRP/GLBP), DHCP relay,
                 STP root aligned to the active gateway, OSPF to the core.
CORE          →  L3 OSPF backbone, dual-ISP IP SLA failover, redundant
                 DHCP servers (one per core).
SERVICES      →  ONE dual-homed DHCP server, reachable via both cores.
```

> Three VLANs, three FHRPs, on purpose: \*\*HSRP\*\* (Tellers), \*\*VRRP\*\* (Management), \*\*GLBP\*\* (ATMs) — so you can see and explain how each behaves.

\---

## 🗺️ Topology

```
                 ISP1 ───┐                         ┌─── ISP2
               (primary)  │                         │  (backup)
                          │                         │
              ┌───────────┴┐                       ┌┴───────────┐
        ┌──────────── DHCP-SERVER ────────────┐
        │ 10.0.0.10                 10.0.0.20 │
   ─────│   CORE-1   │───────────────────────│   CORE-2   │─────
  10.0.0.10   │ OSPF, IPSLA│       L3 link          │ OSPF, IPSLA│   10.0.0.20
  (CORE-1's   │            │                        │            │   (CORE-2's
   helper     └──┬──────┬──┘                        └──┬──────┬──┘    helper
   target)       │      │                              │      │       target)
                 │      └──────────┐      ┌────────────┘      │
     L3 routed   │                 │      │                   │  L3 routed
     links       │      ┌──────────┼──────┘                   │  links
                 │      │          └───────────┐              │
            ┌────┴──────┴┐                  ┌──┴──────────────┴┐
            │   DIST-1   │                  │      DIST-2      │
            │ HSRP/VRRP  │                  │ HSRP/VRRP backup │
            │ ACTIVE     │                  │ GLBP ACTIVE      │
            │ STP root   │                  │ STP root VLAN 30 │
            │ 10/20      │                  │                  │
            └─────┬──────┘                  └────────┬─────────┘
                  │ trunks (L2)                      │ trunks (L2)
            ┌─────┴──────┐                  ┌────────┴─────────┐
            │   ACC-1    │                  │      ACC-2       │  L2 only
            └─────┬──────┘                  └────────┬─────────┘
           VLAN 10/20/30                        VLAN 10/20/30
           Tellers/Mgmt/ATMs                    Tellers/Mgmt/ATMs

  DIST↔CORE and CORE↔CORE links = SINGLE L3 routed links (no switchport).
  Trunks exist ONLY on ACCESS↔DIST (+ the DIST peer link). ONE dual-homed DHCP server reachable via both cores.
```

|Layer|Devices|Role|
|-|-|-|
|**Access**|ACC-1, ACC-2|Pure L2: access ports, trunks to both dist, port security|
|**Distribution**|DIST-1, DIST-2|L3 SVIs, **FHRP**, **ip helper-address**, **STP root**, **OSPF**|
|**Core**|CORE-1, CORE-2|L3 OSPF backbone, **dual-ISP IP SLA failover**|
|**Services**|One dual-homed DHCP server|Reachable via **both cores** (10.0.0.10 \& 10.0.0.20)|

|VLAN|Dept|FHRP|Active on|STP root|
|-|-|-|-|-|
|10|Tellers|**HSRP**|DIST-1|DIST-1|
|20|Management|**VRRP**|DIST-1|DIST-1|
|30|ATMs|**GLBP**|**DIST-2**|**DIST-2**|

> \*\*The active gateway and the STP root match per VLAN.\*\* VLAN 10/20 → DIST-1. VLAN 30 (GLBP) → DIST-2. This balances load across both distribution switches and avoids traffic hairpinning.

\---

## 🧱 ACCESS LAYER — Pure Layer 2

```cisco
! ===== ACC-1 (ACC-2 identical) =====
vlan 10
 name TELLERS
vlan 20
 name MGMT
vlan 30
 name ATMS

! Host access ports
interface range Ethernet0/1 - 2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict

! Uplink trunks to BOTH distribution switches
interface range Ethernet0/0 , Ethernet1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk allowed vlan 10,20,30
```

\---

## 🛡️ DISTRIBUTION LAYER — Gateways, Relay, STP Root, OSPF

### ⬇️ The trunk DOWN to access (this was missing before — fixed)

```cisco
! ===== DIST-1: trunks DOWN to access switches =====
interface range Ethernet0/2 , Ethernet0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk allowed vlan 10,20,30
```

### DIST-1 — Active for HSRP (10) \& VRRP (20), STP root for 10/20

```cisco
! ===== DIST-1 =====
ip routing

! STP root aligned with the FHRP active
spanning-tree vlan 10,20 root primary
spanning-tree vlan 30 root secondary       ! DIST-2 is primary for VLAN 30

! --- VLAN 10: SVI + HSRP (DIST-1 active) ---
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 110                    ! higher → active
 standby 10 preempt
 standby 10 track 1 decrement 20
 ip helper-address 10.0.0.10                ! relay to DHCP server (via CORE-1)

! --- VLAN 20: SVI + VRRP (DIST-1 master) ---
interface Vlan20
 ip address 192.168.20.2 255.255.255.0
 vrrp 20 ip 192.168.20.1
 vrrp 20 priority 110                        ! higher → master
 vrrp 20 preempt
 ip helper-address 10.0.0.10

! --- VLAN 30: SVI + GLBP (DIST-1 is the BACKUP → LOWER priority) ---
interface Vlan30
 ip address 192.168.30.2 255.255.255.0
 glbp 30 ip 192.168.30.1
 glbp 30 priority 100                        ! LOWER → DIST-2 is AVG
 glbp 30 preempt
 glbp 30 load-balancing round-robin
 ip helper-address 10.0.0.10

! --- L3 routed uplinks to the cores ---
interface Ethernet0/0
 no switchport
 ip address 10.1.1.1 255.255.255.252         ! to CORE-1
interface Ethernet0/1
 no switchport
 ip address 10.1.1.9 255.255.255.252         ! to CORE-2

router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 10.1.1.0 0.0.0.255 area 0
 passive-interface Vlan10
 passive-interface Vlan20
 passive-interface Vlan30
```

### DIST-2 — Backup for HSRP/VRRP, **GLBP ACTIVE (higher priority — the fix)**

```cisco
! ===== DIST-2: trunks DOWN to access switches =====
interface range Ethernet0/2 , Ethernet0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk allowed vlan 10,20,30

! ===== DIST-2 =====
ip routing

spanning-tree vlan 10,20 root secondary
spanning-tree vlan 30 root primary          ! DIST-2 root for VLAN 30 (matches GLBP active)

! --- VLAN 10: HSRP backup (LOWER priority) ---
interface Vlan10
 ip address 192.168.10.3 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 100                     ! lower → standby
 standby 10 preempt
 ip helper-address 10.0.0.20                 ! relay to DHCP server (via CORE-2)

! --- VLAN 20: VRRP backup (LOWER priority) ---
interface Vlan20
 ip address 192.168.20.3 255.255.255.0
 vrrp 20 ip 192.168.20.1
 vrrp 20 priority 100                        ! lower → backup
 ip helper-address 10.0.0.20

! --- VLAN 30: GLBP ACTIVE → HIGHER priority (THE FIX) ---
interface Vlan30
 ip address 192.168.30.3 255.255.255.0
 glbp 30 ip 192.168.30.1
 glbp 30 priority 110                        ! HIGHER → DIST-2 is the AVG ✅
 glbp 30 preempt
 glbp 30 load-balancing round-robin
 ip helper-address 10.0.0.20

interface Ethernet0/0
 no switchport
 ip address 10.1.1.5 255.255.255.252         ! to CORE-1
interface Ethernet0/1
 no switchport
 ip address 10.1.1.13 255.255.255.252        ! to CORE-2

router ospf 1
 router-id 2.2.2.2
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 10.1.1.0 0.0.0.255 area 0
 passive-interface Vlan10
 passive-interface Vlan20
 passive-interface Vlan30
```

> 🔧 \*\*The GLBP fix:\*\* GLBP's AVG (the "active") is the router with the \*\*higher\*\* priority. We want \*\*DIST-2\*\* to be the GLBP active for VLAN 30, so \*\*DIST-2 gets 110 and DIST-1 gets 100\*\* — the opposite of HSRP/VRRP. (In the old version both were 110 on DIST-1, which was wrong.) STP root for VLAN 30 follows to DIST-2 to match.

> 🔧 \*\*Redundant relay paths:\*\* DIST-1 relays to the DHCP server via \*\*CORE-1 (10.0.0.10)\*\*, DIST-2 relays via \*\*CORE-2 (10.0.0.20)\*\* — the same dual-homed server, two paths. If one core path dies, the other distribution switch still reaches the server.

\---

## 🌐 CORE LAYER — Full OSPF Backbone + IP SLA (both cores)

### CORE-1

```cisco
! ===== CORE-1 =====
ip routing

interface Ethernet0/0
 no switchport
 ip address 10.1.1.2 255.255.255.252         ! to DIST-1
interface Ethernet0/1
 no switchport
 ip address 10.1.1.6 255.255.255.252         ! to DIST-2
interface Ethernet0/2
 no switchport
 ip address 10.2.2.1 255.255.255.252         ! core interconnect to CORE-2

! Services segment — link to the DHCP server (CORE-1 side)
interface Ethernet0/3
 no switchport
 ip address 10.0.0.9 255.255.255.248         ! DHCP server = 10.0.0.10 (/29)

! ISP1 (primary)
interface Ethernet1/0
 description ISP1-PRIMARY
 ip address 203.0.113.2 255.255.255.252

router ospf 1
 router-id 10.10.10.10
 network 10.1.1.0 0.0.0.255 area 0
 network 10.2.2.0 0.0.0.3 area 0
 network 10.0.0.8 0.0.0.7 area 0
 default-information originate

! Dual-ISP failover — probe a REAL internet host (8.8.8.8) so the track
! drops only when the actual internet is unreachable.
!
! CRITICAL: pin the probe target with a /32 route via the LOCAL ISP.
! Without it you get two failures:
!   (1) bootstrap deadlock — the probe needs the very default route the
!       track controls; once the track drops the default, the probe can
!       never reach 8.8.8.8 again, so the track can't recover.
!   (2) inter-core coupling — without the pin, 8.8.8.8 may route via the
!       core interconnect (10.2.2.x) to the OTHER core, so one core failing
!       drags the other's track down.
! The /32 keeps 8.8.8.8 reachable out THIS core's own ISP, always.
ip route 8.8.8.8 255.255.255.255 203.0.113.1     ! pin probe target via ISP1

ip sla 1
 icmp-echo 8.8.8.8 source-interface Ethernet1/1
 frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability

ip route 0.0.0.0 0.0.0.0 203.0.113.1 track 1     ! tracked primary (via ISP1)
ip route 0.0.0.0 0.0.0.0 10.2.2.2 20             ! floating backup via CORE-2
```

> 🔧 \*\*Probe 8.8.8.8, not the ISP interface.\*\* Probing the ISP's link IP (e.g. 203.0.113.1) only proves the local link is up — if the ISP's upstream/internet dies, the probe still succeeds and failover never happens. Worse, if the ISP simply doesn't answer ICMP on its interface, the track flaps and silently withdraws your default route. Pointing the probe at a reliable internet host (8.8.8.8) means \*\*track 1 goes down only when the real internet is unreachable\*\*, which is exactly when you want to fail over to ISP2.

### CORE-2 (the full config that was missing)

```cisco
! ===== CORE-2 =====
ip routing

interface Ethernet0/0
 no switchport
 ip address 10.1.1.10 255.255.255.252        ! to DIST-1
interface Ethernet0/1
 no switchport
 ip address 10.1.1.14 255.255.255.252        ! to DIST-2
interface Ethernet0/2
 no switchport
 ip address 10.2.2.2 255.255.255.252         ! core interconnect to CORE-1

! Services segment — link to the DHCP server (CORE-2 side)
interface Ethernet0/3
 no switchport
 ip address 10.0.0.17 255.255.255.248        ! DHCP server = 10.0.0.20 (/29)

! ISP2 (backup)
interface Ethernet1/0
 description ISP2-BACKUP
 ip address 198.51.100.2 255.255.255.252

router ospf 1
 router-id 20.20.20.20
 network 10.1.1.0 0.0.0.255 area 0
 network 10.2.2.0 0.0.0.3 area 0
 network 10.0.0.16 0.0.0.7 area 0
 default-information originate

! Dual-ISP failover (CORE-2 owns ISP2). Use a DIFFERENT target (1.1.1.1)
! pinned via ISP2 — this is what keeps the two cores independent.
!
! Why a different target + local pin: if CORE-2 also probed 8.8.8.8, that
! /32 lives on CORE-1, so CORE-2 would route the probe via the interconnect
! to CORE-1 — coupling the two. Probing 1.1.1.1 pinned to ISP2's next-hop
! sends CORE-2's probe straight out ITS OWN ISP, fully independent of CORE-1.
ip route 1.1.1.1 255.255.255.255 198.51.100.1    ! pin probe target via ISP2

ip sla 2
 icmp-echo 1.1.1.1 source-interface Ethernet1/1
 frequency 5
ip sla schedule 2 life forever start-time now
track 2 ip sla 2 reachability

ip route 0.0.0.0 0.0.0.0 198.51.100.1 track 2    ! tracked primary (via ISP2)
ip route 0.0.0.0 0.0.0.0 10.2.2.1 20             ! floating backup via CORE-1
```

> 🧠 \*\*The IP SLA design lesson (hard-won).\*\* Two non-obvious rules make tracked default routes actually work:
> 1. \*\*Pin the probe target with a `/32` via the local ISP.\*\* Never let the probe depend on the very default route the track controls — that's a deadlock: when the track drops the default, the probe loses its path and can never detect recovery, so the track stays down forever.
> 2. \*\*Use a different probe target per core, each pinned to its own ISP.\*\* Otherwise the second core routes its probe through the inter-core link to the first core, and one core failing drags the other's track down. CORE-1 → 8.8.8.8 via ISP1, CORE-2 → 1.1.1.1 via ISP2 = fully independent failover detection.
> Also: probe a real internet host (not the ISP's own interface) so the track reflects true internet reachability, and source the probe from the actual ISP-facing interface (here `Ethernet1/1`).

> \*\*Note on the DHCP links:\*\* the single DHCP server is dual-homed — `10.0.0.10` on CORE-1's `10.0.0.8/29` and `10.0.0.20` on CORE-2's `10.0.0.16/29`. DIST-1's helper points at `10.0.0.10`, DIST-2's at `10.0.0.20`; both reach the same server, just via different cores.

\---

## 🖥️ SERVICES — One Dual-Homed DHCP Server

A **single DHCP server** is dual-homed to **both cores** — one link to CORE-1 (`10.0.0.10`) and one to CORE-2 (`10.0.0.20`). Either core can reach it, so a single core failure doesn't cut off DHCP. The server has one set of pools and **excludes the gateway range (.1–.10) on every VLAN** so it never hands a client an address that belongs to an FHRP virtual IP or a real SVI.

```cisco
! ===== DHCP SERVER (dual-homed to both cores) =====
interface Ethernet0/0
 ip address 10.0.0.10 255.255.255.248        ! to CORE-1 (10.0.0.8/29)
interface Ethernet0/1
 ip address 10.0.0.20 255.255.255.248        ! to CORE-2 (10.0.0.16/29)

! Reach the VLANs back through the cores (server runs no OSPF)
ip route 0.0.0.0 0.0.0.0 10.0.0.9            ! primary via CORE-1
ip route 0.0.0.0 0.0.0.0 10.0.0.17 10        ! floating backup via CORE-2

! Exclude the gateway range on EVERY VLAN (.1 = FHRP virtual,
! .2/.3 = real SVIs, .4–.10 reserved for static infrastructure)
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp excluded-address 192.168.30.1 192.168.30.10

ip dhcp pool TELLERS
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1                ! HSRP virtual IP
 dns-server 208.67.222.222
ip dhcp pool MGMT
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1                ! VRRP virtual IP
 dns-server 208.67.222.222
ip dhcp pool ATMS
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1                ! GLBP virtual IP
 dns-server 208.67.222.222
```

> 🔑 \*\*Why exclude .1–.10 on every VLAN:\*\* the pool is the whole `/24`, so without exclusions DHCP could hand a client `.1` (the FHRP virtual gateway) or `.2/.3` (the real SVIs) → an IP conflict that breaks the VLAN. Reserving `.1–.10` on \*\*all three\*\* VLANs keeps the gateways and static infrastructure safe; clients get `.11–.254`.

> 🔑 \*\*Why dual-homed instead of two servers:\*\* one server on two links means either core can deliver DHCP, but there's only one lease database — no split-scope to manage and no risk of two servers handing out the same address. The redundancy is in the \*\*paths to the server\*\* (via CORE-1 or CORE-2), which is what the topology needs.

\---

## 🌍 NAT — On the ISP Router (a real troubleshooting lesson)

> 🧠 \*\*Why NAT lives on the ISP router, not the core.\*\* In this lab the "core" devices are \*\*Layer 2 switches\*\* (IOSvL2). NAT requires a routed (Layer 3) data path — `ip nat inside/outside` only translates on a true router. Configuring NAT on an L2 core silently does nothing (translations never fire, `Hits: 0`). So NAT is placed on the \*\*ISP routers\*\*, which are full Layer 3 routers and the actual boundary to the internet. This matches reality: the edge/ISP router is where private addresses get translated to public.

The VLANs use private addresses (192.168.10/20/30). The ISP router **PATs (overload)** that traffic onto its internet-facing interface. The ACL permits **each VLAN explicitly** — it deliberately does **not** use `192.168.0.0/16`, because the DHCP/cloud uplink address also falls in that range and must not be translated.

```cisco
! ===== ISP1 router: NAT to the internet =====

! Inside interface (facing CORE-1 / the LAN side)
interface Ethernet0/0
 ip address 203.0.113.1 255.255.255.252
 ip nat inside

! Outside interface (facing the cloud / internet, IP via DHCP)
interface Ethernet0/1
 ip address dhcp
 ip nat outside

! Per-VLAN NAT ACL — list each internal VLAN explicitly.
! NOTE: do NOT use 192.168.0.0/16 here — the DHCP cloud uplink
! address is also in 192.168.x and must not be translated.
ip access-list extended NAT-INTERNET
 permit ip 192.168.10.0 0.0.0.255 any
 permit ip 192.168.20.0 0.0.0.255 any
 permit ip 192.168.30.0 0.0.0.255 any

! PAT (overload) onto the cloud-facing interface
ip nat inside source list NAT-INTERNET interface Ethernet0/1 overload

! Static routes: reach the internal VLANs via CORE-1, default to cloud
ip route 192.168.10.0 255.255.255.0 203.0.113.2
ip route 192.168.20.0 255.255.255.0 203.0.113.2
ip route 192.168.30.0 255.255.255.0 203.0.113.2
ip route 0.0.0.0 0.0.0.0 dhcp
```

```cisco
! ===== CORE-1 edge toward ISP1 (static, L3 link) =====
! The core points its default at the ISP router; the ISP router
! points static routes back at the core's edge IP.
interface Ethernet1/1
 ip address 203.0.113.2 255.255.255.252
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

> 🔑 \*\*Why per-VLAN, not 192.168.0.0/16:\*\* the ISP's cloud-facing interface gets its IP via DHCP, and that pool also lives in 192.168.x. A broad `192.168.0.0 0.0.255.255` would try to translate (or mis-handle) that uplink traffic. Listing VLAN 10/20/30 explicitly translates \*\*only\*\* real client traffic and leaves the cloud uplink alone.

```
Client (192.168.10.x) → Internet:
  → CORE (L2) switches it up → CORE-1 routes to ISP1 (static)
  → ISP1 matches NAT ACL → PAT to its cloud IP → out to internet ✅
  → reply returns to ISP1 → un-NAT → static route back to CORE-1 → client ✅
```

> ⚠️ \*\*The lesson:\*\* NAT on a Layer 2 "core" never fires (`Hits: 0`) because there's no routed boundary there. Always place NAT on a Layer 3 router at the actual inside/outside edge — here, the ISP router. Static routes (ISP ↔ core) tie the two ends together without leaking the ISP link into OSPF.

\---

## 🔗 The DHCP Chain (now redundant)

```
Client in VLAN 10 boots:
1 → broadcasts DISCOVER
2 → DIST SVI (ip helper) → unicast to its DHCP server
      DIST-1 → DHCP server via CORE-1 (10.0.0.10)
      DIST-2 → DHCP server via CORE-2 (10.0.0.20)
3 → OSPF routes it to the server ✅
4 → server replies: IP + default-router = FHRP virtual gateway
5 → reply routes back → client online ✅

If a core/server path fails → the other distribution
switch + the other DHCP server keep serving ✅
```

\---

## 🧪 The Failover Drills

|#|Break this|Expected result|Proof|
|-|-|-|-|
|1|Shut DIST-1|DIST-2 takes HSRP/VRRP gateway|`show standby brief` 📸|
|2|Pull ISP1 (block 8.8.8.8 reachability)|IP SLA 1 down → traffic fails to ISP2/CORE-2|`show track 1`, `show ip route` 📸|
|3|GLBP: shut DIST-2|ATMs keep working via DIST-1|`show glbp brief` 📸|
|4|Drop CORE-1's link to the DHCP server|Clients still get IPs via CORE-2 path (10.0.0.20)|`show ip dhcp binding` 📸|
|5|Cut a DIST↔CORE link|OSPF reroutes via other core|`show ip ospf neighbor` 📸|
|6|Continuous ping throughout|1–2 drops, then recovers|ping output 📸|

\---

## 🧠 What This Lab Demonstrates

```
✅ Real 3-tier hierarchy with correct role separation
✅ OSPF backbone gluing the tiers (end-to-end routing)
✅ L3 routed links dist↔core and core↔core (no switchport)
✅ Trunks correctly configured on BOTH ends of access↔dist
✅ All three FHRPs with CORRECT priorities:
   HSRP/VRRP active on DIST-1, GLBP active on DIST-2 (higher priority)
✅ STP root aligned to the active gateway per VLAN
✅ DHCP relay (ip helper-address) to one dual-homed server, reachable via both cores
✅ ip helper-address relay pointed at each side's server
✅ Dual-ISP automatic failover (IP SLA + tracking + floating routes)
✅ Pinned /32 probe targets (per-core, per-ISP) — no deadlock, no inter-core coupling
✅ NAT/PAT on the ISP router (L3 edge) with per-VLAN ACL (not 192.168.0.0/16)
✅ No single point of failure at any tier
```



<div align="center">

## 🏦 Why This Lab Matters

No single point of failure — at *any* tier.
Switch, gateway, DHCP server, or ISP: lose one, the bank keeps running.

|Tier|Redundancy|
|-|-|
|**Access**|Dual uplinks, loop-free L2, port security|
|**Distribution**|FHRP gateways (correct priorities) + STP alignment + redundant relay|
|**Core**|OSPF backbone + dual-ISP IP SLA failover + core interconnect|
|**Services**|ONE dual-homed DHCP server, reachable via both cores|



</div>

