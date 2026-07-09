# Inter-AS-MPLS-VPN-Lab

![Alt Text](Images/Inter-AS-MPLS-VPN_Diagram.png)

To recreate an Inter-AS-MPLS-VPN Type B network, it must contain two different autonomous systems (AS) and the VPN should span these two systems. Option B allows autonomous system boundary routers (ASBR) to exchange VPN routes using the external boundary gate protocol (eBGP).

### Option B 
1. Uses one MP-BGP session between ASBR1 and ASBR2 to carry all VPN routes.
2. The ASBRs do not have vrf configured on them, they simply act as BGP routers and pass VPN labels.
3. Label swapping happens between ASBR 1 and ASBR2. It changes the next hop address to the current router and assigns a new MPLS label.

### Control Plane (Route Exchange)
1. CE sends its routes to PE1
2. PE1 translates these routes using the route distinguishers (RD) and sends them to ASBR1 using internal BGP (iBGP)
3. ASBR1 sends the routes to ASBR2 using MP-BGP.
4. ASBR2 passes the routes to PE2, which advertises the routes to CE3.

### Data Plane (Packet Forwarding)
1. The packet is encapsulated in a two layer stack
2. The outer label is to get the packet to the next hop or ASBR
3. The inner label identifies the VPN

# Software

GNS3 was used to simulate this lab. The routers were emulated with Dynamips and the images used were for the c7200x series.

# Configuration

![Alt Text](Images/Topology.png)

Firstly, the topology was configured as shown in the picture. The routing table shows the ip addresses and interfaces for each link. 

OSPF is configured on all routers apart from the link between ASBR1 and ASBR2 because it uses MP-BGP. MPLS is configured inside AS1 and AS2, and the link between ASes (ASBR1 and ASBR2). iBGP is configured inside the AS (PE1-ASBR1) and eBGP is placed between ASBR1 and ASBR2.

### CE1 CUSTOMER EDGE

The loopback address is set because virtual routing and forwarding (VRF) is virtual. Therefore, OSPF will announce the loopback address to ensure that the BGP paths are stable.
```
interface Loopback0
 ip address 11.11.11.11 255.255.255.255
```

The interface to PE1 is configured:
```
interface GigabitEthernet1/0
 description Link to PE1
 ip address 10.1.11.1 255.255.255.0
 negotiation auto
```

BGP configuration:
- A private AS number is assigned.
- The system logs when a neighbour session goes up or down (useful for logging and debugging).
- The network command advertises the loopback address if it exists in the routing table.
-The command neighbour defines the remote peer (Neighbour IP:Remote AS). In this example, the local AS number (65001) is different from the remote AS number (1), which means this is an eBGP session:
```
router bgp 65001
 bgp log-neighbor-changes
 network 11.11.11.11 mask 255.255.255.255
 neighbor 10.1.11.2 remote-as 1
```

On all routers, ip cef is enabled for high speed packet forwarding:
```
ip cef
```

### PE1 PROVIDER EDGE

VRFs are set up to isolate the routing tables for VPN1 and VPN 2:
- The route distinguisher (RD) is a prepended to an IPV4 prefix. It is a unique identifier for the VPN (AS number:ID).
- The route target is an attribute to control the import and export of routes. Exporting a route advertises that route from the PE. Importing a route places that route into the routing table of the VPN (AS:RD).
```
ip vrf VPN1
 rd 1:1
 route-target export 100:1
 route-target import 100:1

ip vrf VPN2
 rd 1:2
 route-target export 100:2
 route-target import 100:2
```

Interface Configuration:
- ASBR facing links (G1/0) uses MPLS instead of standard routing by using command mpls ip.
- CE facing links (G2/0-G3/0) uses command ip vrf forwarding VPNx to strip standard IP routing and place it into an isolated routing sandbox VPNx.
```
interface GigabitEthernet1/0
 description Core Link to ASBR1
 ip address 10.1.13.1 255.255.255.0
 negotiation auto
 mpls ip

interface GigabitEthernet2/0
 description Connects to CE1 (VPN1)
 ip vrf forwarding VPN1
 ip address 10.1.11.2 255.255.255.0

interface GigabitEthernet3/0
 description Connects to CE2 (VPN2)
 ip vrf forwarding VPN2
 ip address 10.1.12.2 255.255.255.0
 negotiation auto
```

OSPF configuration:
- OSPF runs internally within the AS to advertise its loopback address to ABR1 to allow BGP and the Label Distribution Protocol (LDP) to work. 
```
router ospf 1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.1.13.0 0.0.0.255 area 0
```

MULTI-PROTOCOL BGP configuration:
- The neighbour command forms a BGP relationship with the router at 3.3.3.3
- Update-source Loopback0 uses its own loopback address as the source IP. This ensures network redundancy as the loopback address is always available, and the iBGP session will stay active.
```
router bgp 1
 bgp log-neighbor-changes
 neighbor 3.3.3.3 remote-as 1
 neighbor 3.3.3.3 update-source Loopback0
```
- Address family vpnv4 is a BGP configuration mode for handling VPN4 routing information
- The activate command enables the exchange of VPN4 information with the router.
- The send-community both command attaches both Standard and Extended BGP communities. Route Targets are classified as extended, if this command isnt configured the routers in AS2 wont know which vrf route the packets belong to. 
```
 address-family vpnv4
  neighbor 3.3.3.3 activate
  neighbor 3.3.3.3 send-community both
 exit-address-family
```

```
 address-family ipv4 vrf VPN1
  neighbor 10.1.11.1 remote-as 65001
  neighbor 10.1.11.1 activate
 exit-address-family
```

# Network Topology & Routing Table Documentation

This repository maintains the running configurations and state documentation for the MPLS Inter-AS network architecture. Below is the organized reference tracking for all device interfaces, VRF routing instances, and BGP tables.

---

## 1. Global Interface & Address Mapping

The following table comprehensively indexes all physical and logical interfaces across the current infrastructure topology.

| Device | Interface | IP Address | Subnet Mask | Description / Role | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **CE1** | Loopback0 | [cite_start]`11.11.11.11` [cite: 8] | [cite_start]`255.255.255.255` [cite: 8] | Router ID / Loopback | Up |
| | FastEthernet0/0 | [cite_start]*None* [cite: 9] | [cite_start]*N/A* [cite: 9] | Unused | [cite_start]Shutdown [cite: 9] |
| | GigabitEthernet1/0 | [cite_start]`10.1.11.1` [cite: 9] | [cite_start]`255.255.255.0` [cite: 9] | Link to PE1 | Up |
| **CE2** | Loopback0 | [cite_start]`22.22.22.22` [cite: 19] | [cite_start]`255.255.255.255` [cite: 19] | Router ID / Loopback | Up |
| | FastEthernet0/0 | [cite_start]*None* [cite: 20] | [cite_start]*N/A* [cite: 20] | Unused | [cite_start]Shutdown [cite: 20] |
| | GigabitEthernet1/0 | [cite_start]`10.1.12.1` [cite: 20] | [cite_start]`255.255.255.0` [cite: 20] | Link to PE1 (VPN2) | Up |
| **PE1** | Loopback0 | [cite_start]`1.1.1.1` [cite: 30] | [cite_start]`255.255.255.255` [cite: 30] | Core Router ID / Loopback | Up |
| | FastEthernet0/0 | [cite_start]*None* [cite: 31] | [cite_start]*N/A* [cite: 31] | Unused | [cite_start]Shutdown [cite: 31] |
| | GigabitEthernet1/0 | [cite_start]`10.1.13.1` [cite: 31] | [cite_start]`255.255.255.0` [cite: 31] | Core Link to ASBR1 (MPLS Enabled) | [cite_start]Up [cite: 31] |
| | GigabitEthernet2/0 | [cite_start]`10.1.11.2` [cite: 32] | [cite_start]`255.255.255.0` [cite: 32] | CE1 Connection (VRF VPN1) | [cite_start]Up [cite: 32] |
| | GigabitEthernet3/0 | [cite_start]`10.1.12.2` [cite: 33] | [cite_start]`255.255.255.0` [cite: 33] | CE2 Connection (VRF VPN2) | [cite_start]Up [cite: 33] |
| | GigabitEthernet4/0 | [cite_start]*None* [cite: 34] | [cite_start]*N/A* [cite: 34] | Unused | [cite_start]Shutdown [cite: 34] |
| **ASBR1**| Loopback0 | [cite_start]`3.3.3.3` [cite: 51] | [cite_start]`255.255.255.255` [cite: 51] | Core Router ID / Loopback | Up |
| | FastEthernet0/0 | [cite_start]*None* [cite: 52] | [cite_start]*N/A* [cite: 52] | Unused | [cite_start]Shutdown [cite: 52] |
| | GigabitEthernet1/0 | [cite_start]`10.1.13.3` [cite: 52] | [cite_start]`255.255.255.0` [cite: 52] | Link to PE1 (MPLS Enabled) | [cite_start]Up [cite: 52] |
| | GigabitEthernet2/0 | [cite_start]`10.12.0.3` [cite: 53] | [cite_start]`255.255.255.0` [cite: 53] | Inter-AS Link to ASBR2 (MPLS BGP) | [cite_start]Up [cite: 53] |

---

## 2. Active VRF Routing Table (PE1 - VPN1)

The following entries reflect the active local routing state extracted from PE1's `show ip route vrf VPN1` instance:

| Protocol Type | Destination Prefix | Next Hop | Outbound Interface / BGP Details |
| :---: | :--- | :--- | :--- |
| **C** (Connected) | [cite_start]`10.1.11.0/24` [cite: 46] | [cite_start]Directly Connected [cite: 46] | [cite_start]GigabitEthernet2/0 [cite: 46] |
| **L** (Local) | [cite_start]`10.1.11.2/32` [cite: 46] | [cite_start]Directly Connected [cite: 46] | [cite_start]GigabitEthernet2/0 [cite: 46] |
| **B** (BGP) | [cite_start]`11.11.11.11/32` [cite: 46, 47] | [cite_start]`10.1.11.1` [cite: 47] | [cite_start]External BGP via CE1 [cite: 47] |
| **B** (BGP) | [cite_start]`33.33.33.33/32` [cite: 47] | [cite_start]`3.3.3.3` [cite: 47] | [cite_start]Internal iBGP via ASBR1 loopback [cite: 47] |

---

## 3. ASBR1 BGP VPNv4 Routing Table

The exchange table mapping how multi-protocol labels cross autonomous boundaries, from the perspective of `ASBR1`:

| Route Distinguisher (RD) | Target Network Prefix | Next Hop Address | Metric / LocPrf | AS-Path / Origin Type |
| :--- | :--- | :--- | :---: | :--- |
| **1:1** (VPN1 via PE1) | [cite_start]`11.11.11.11/32` [cite: 64] | [cite_start]`1.1.1.1` [cite: 64] | [cite_start]0 / 100 [cite: 64] | [cite_start]`65001 i` [cite: 64] |
| **1:2** (VPN2 via PE1) | [cite_start]`22.22.22.22/32` [cite: 64, 65] | [cite_start]`1.1.1.1` [cite: 64] | [cite_start]0 / 100 [cite: 65] | [cite_start]`65002 i` [cite: 65] |
| **3:1** (Remote ASBR2) | [cite_start]`33.33.33.33/32` [cite: 65] | [cite_start]`10.12.0.4` [cite: 65] | *N/A* | [cite_start]`2 65003 i` [cite: 65] |
| **4:1** (Remote ASBR2) | [cite_start]`44.44.44.44/32` [cite: 65] | [cite_start]`10.12.0.4` [cite: 65] | *N/A* | [cite_start]`2 65004 i` [cite: 65] |

---

## 4. MPLS Architectural & Path Analysis

### Inter-AS Option B Implementation
* [cite_start]**Design Pattern**: ASBR1 explicitly suppresses traditional target filtering via `no bgp default route-target filter`[cite: 55].
* [cite_start]**Label Distribution**: Label assignment relies on standard MPLS over BGP interactions (`mpls bgp forwarding`) across the Inter-AS boundary link[cite: 53].

### Operational Path Verification (CE1 Data)
[cite_start]A trace execution targeting loopback address `33.33.33.33` yields the following explicit label stack layout[cite: 16]:

* [cite_start]**Hop 1 (`10.1.11.2`)**: PE1 Ingress VRF Node[cite: 16].
* [cite_start]**Hop 2 (`10.1.13.3`)**: ASBR1 Transiting Label Core Node -> **[MPLS: Label 20, Exp 0]**[cite: 16].
* [cite_start]**Hop 3 (`10.12.0.4`)**: ASBR2 Border Interface Node -> **[MPLS: Label 18, Exp 0]**[cite: 16].
* [cite_start]**Hop 4 (`10.2.21.2`)**: Remote PE Egress Node Area -> **[MPLS: Label 17, Exp 0]**[cite: 16].
* [cite_start]**Hop 5 (`10.2.21.1`)**: Remote Customer Edge Target Node[cite: 16].

```
 address-family ipv4 vrf VPN2
  neighbor 10.1.12.1 remote-as 65002
  neighbor 10.1.12.1 activate
 exit-address-family
```
