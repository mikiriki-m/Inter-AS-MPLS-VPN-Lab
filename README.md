# Inter-AS-MPLS-VPN-Lab

![Alt Text](Images/Inter-AS-MPLS-VPN_Diagram.png)

To recreate an Inter-AS-MPLS-VPN Type B network, it must contain two different autonomous systems (AS) and the VPN should span these two systems. Option B allows autonomous system boundary routers (ASBR) to exchange VPN routes using the external boundary gate protocol (eBGP).

Option B 
1. Uses one MP-BGP session between ASBR1 and ASBR2 to carry all VPN routes.
2. The ASBRs do not have vrf configured on them, they simply act as BGP routers and pass VPN labels.
3. Label swapping happens between ASBR 1 and ASBR2. It changes the next hop address to the current router and assigns a new MPLS label.

Control Plane (Route Exchange)
1. CE sends its routes to PE1
2. PE1 translates these routes using the route distinguishers (RD) and sends them to ASBR1 using internal BGP (iBGP)
3. ASBR1 sends the routes to ASBR2 using MP-BGP.
4. ASBR2 passes the routes to PE2, which advertises the routes to CE3.

Data Plane (Packet Forwarding)
1. The packet is encapsulated in a two layer stack
2. The outer label is to get the packet to the next hop or ASBR
3. The inner label identifies the VPN

# Software

GNS3 was used to simulate this lab. The routers were emulated with Dynamips and the images used were for the c7200x series.

# Configuration

Firstly, the topology was 


