# Collection Center Network Pattern

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | JR1 / SR1 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#11 (A11) |
| Builds on | [`docs/addressing-plan.md`](addressing-plan.md) (A07), [`docs/connectivity.md`](connectivity.md) (A10, section 9), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [`docs/landing-zone.md`](landing-zone.md) (A08, section 4) |
| Feeds | A09 identities (aroma-pacifico/aroma-pacifico-roadmap#9), A12 firewall policy table (aroma-pacifico/aroma-pacifico-roadmap#12), A16 sack and trip (aroma-pacifico/aroma-pacifico-roadmap#16), A19 complete diagram (aroma-pacifico/aroma-pacifico-roadmap#19), A21 observability (aroma-pacifico/aroma-pacifico-roadmap#34), A23 cost (aroma-pacifico/aroma-pacifico-roadmap#36) |
| Decision-log entries | ASM-080 to ASM-089, PD-090 to PD-096 |

## 1. Purpose and scope

Point 3 of the [case statement](case/case-statement.pdf) says that every collection center is born without a LAN and that one must be designed, with segmentation and a WAN exit to the cloud ("Cada centro de acopio nace sin LAN. Hay que diseñarla, con segmentación y salida WAN hacia la nube"). It asks for a "WLAN segmentada por zona, con SSID propio. La VLAN de visitantes no enruta hacia logística ni gerencia", for a reference capacity of "2 Mbps de bajada y 0,5 Mbps de subida" per active client, and for control of non-operational traffic ("Hay control de tráfico no operativo"). Part D adds "Wifi y firewall del inventario", "Puestos de logística que registran la recepción", "Impresión de hojas de códigos para los sacos que lleguen sin código" and "El mismo patrón, tres veces, sin redibujar tres redes distintas si el patrón ya está".

This document defines that pattern once. It keeps the addressing plan unchanged: each center owns its /16 (10.41.0.0/16 Cauca, 10.42.0.0/16 Huila, 10.43.0.0/16 Tolima) inside 10.40.0.0/13, which the cloud reaches with one summary route through the `transit` tier. It keeps the WAN design of the connectivity document unchanged: the center firewall in the cloud link zone, two IPsec tunnels to the hub of the network account over the 100 Mbps internet service, a cellular backup link, local internet breakout for visitors, and an outbox on every logistics post (ASM-078, ASM-079). It also keeps ADR-002: missions are preloaded at the municipal seat or at the collection center, so the drone ground station needs a place on the center LAN.

The firewall rules with ports, and whether each is permanent or temporary, belong to A12 (aroma-pacifico/aroma-pacifico-roadmap#12). This document gives the zone-level intent that A12 derives them from (section 6). Device models, prices and the number of access points are out of scope: the first two belong to A23, and the third is answered by the site survey (PD-092). No value in this document is a measurement; capacities, guarantees and caps are design values.

## 2. One pattern, three centers

The case statement repeats the inventory table in the three centers by changing the second octet ("Se repite el mismo esquema en los tres centros, cambiando el segundo octeto: 41 Cauca, 42 Huila, 43 Tolima"). The pattern therefore has exactly two parameters, and every other value in this document (VLAN IDs, SSID names, authentication, gateway position, DHCP pools, policy intent, traffic classes, guarantees and caps) is identical in the three centers.

| Parameter | Cauca | Huila | Tolima | Used for |
|---|---|---|---|---|
| `center_key` | `cauca` | `huila` | `tolima` | Device names, the `CollectionCenter` tag of the landing zone (PD-072), observability of the three WAN links. |
| `center_octet` (written `4x` below) | 41 | 42 | 43 | Second octet of every address in the center. |

Every address below is written as `10.4x.Z.H`, where `Z` is the third octet of the zone network in the case statement and `H` the host. The derived networks are the case statement's own table, repeated:

| Zone (case statement) | Network rule | Cauca | Huila | Tolima |
|---|---|---|---|---|
| Gerencia (management) | 10.4x.1.0/24 | 10.41.1.0/24 | 10.42.1.0/24 | 10.43.1.0/24 |
| Soporte (support) | 10.4x.2.0/24 | 10.41.2.0/24 | 10.42.2.0/24 | 10.43.2.0/24 |
| Logística (logistics) | 10.4x.3.0/24 | 10.41.3.0/24 | 10.42.3.0/24 | 10.43.3.0/24 |
| Transporte (transport) | 10.4x.4.0/24 | 10.41.4.0/24 | 10.42.4.0/24 | 10.43.4.0/24 |
| Visitantes (visitors) | 10.4x.5.0/24 | 10.41.5.0/24 | 10.42.5.0/24 | 10.43.5.0/24 |
| Servidores locales (local servers) | 10.4x.10.0/24 | 10.41.10.0/24 | 10.42.10.0/24 | 10.43.10.0/24 |
| Enlace a la nube (cloud link) | 10.4x.20.0/24 | 10.41.20.0/24 | 10.42.20.0/24 | 10.43.20.0/24 |

No zone is added. Every design element of this task has a place in one of the seven zones (section 4), so the rest of each /16 stays free for the growth the addressing plan reserves it for.

## 3. Zones, VLANs and SSIDs

### 3.1 Zone table

The VLAN ID of each zone is 100 plus the third octet of its network, so the ID can be read from the address and is the same in every center; VLAN 1 is not used for any zone (ASM-080). Each zone's gateway is the `.1` address on a firewall subinterface for that VLAN, so all routing between zones happens on the firewall and every inter-zone flow passes its policy (section 6). The switch and the access points only carry VLAN tags; they do not route.

| Zone | Network | VLAN | SSID (same in all centers) | Authentication | Gateway | Planned clients (case statement) | DHCP pool | Lease |
|---|---|---|---|---|---|---|---|---|
| Gerencia | 10.4x.1.0/24 | 101 | `cc-management` | WPA3-Enterprise, 802.1X EAP-TLS, center-managed devices | 10.4x.1.1 | 2 desktops, 5 phones (7) | 10.4x.1.50 to .69 (20) | 12 h |
| Soporte | 10.4x.2.0/24 | 102 | `cc-support` | WPA3-Enterprise, 802.1X EAP-TLS, center-managed devices and the enrolled drone ground station | 10.4x.2.1 | 2 desktops, 5 phones (7); plus the ground station at 10.4x.2.10 (section 4.1) | 10.4x.2.50 to .69 (20) | 12 h |
| Logística | 10.4x.3.0/24 | 103 | `cc-logistics` | WPA3-Enterprise, 802.1X EAP-TLS, center-managed devices | 10.4x.3.1 | 2 desktops, 20 phones (22) | 10.4x.3.50 to .99 (50) | 12 h |
| Transporte | 10.4x.4.0/24 | 104 | `cc-transport` | WPA3-Personal (SAE), passphrase given to registered transporters, client isolation | 10.4x.4.1 | 20 phones (20) | 10.4x.4.50 to .89 (40) | 4 h |
| Visitantes | 10.4x.5.0/24 | 105 | `cc-guest` | WPA3-Personal (SAE), passphrase shown at reception, client isolation | 10.4x.5.1 | 10 phones (10) | 10.4x.5.50 to .69 (20) | 1 h |
| Servidores locales | 10.4x.10.0/24 | 110 | Wired only | Fixed ports; network devices by DHCP reservation, printers static | 10.4x.10.1 | Designed here: network devices and printers (section 3.4) | Reservations only, no dynamic pool | Not applicable |
| Enlace a la nube | 10.4x.20.0/24 | 120 | Wired only, firewall edge ports | Not a client zone | Not applicable; the firewall is the edge | Center firewall (case statement) | None, static | Not applicable |

The five client zones each have their own SSID, as the case statement requires; the two infrastructure zones have none, so no wireless client can ever join them. Five SSIDs per radio is the most this pattern broadcasts; a new SSID is a change to the pattern, not to one center. Staff desktops and the logistics reception posts connect by cable to access ports of their zone VLAN, with the same 802.1X EAP-TLS admission as on Wi-Fi.

### 3.2 Authentication and identity source

Staff zones (Gerencia, Soporte, Logística) use WPA3-Enterprise with 802.1X and EAP-TLS. Network admission authenticates the device, and the person is authenticated by the application: the case statement requires MFA for the collection-center operator ("MFA para operario de acopio"), and that login, with MFA, happens in the platform applications against the application identity directory that A09 chooses (landing zone section 4). The network does not create a second set of passwords.

The identity source for staff network admission is therefore the A09 directory, through device certificates: a center-managed device receives a client certificate from the platform's private certificate authority only when it is enrolled to a staff identity of that directory, and the certificate carries the zone of that identity (management, support or logistics), which the RADIUS policy maps to VLAN 101, 102 or 103 (ASM-081). The RADIUS service runs on the center firewall and validates certificates locally against the CA chain and a cached revocation list, so a device can join, and reception can continue, while the WAN is down, which the connectivity design requires (connectivity section 9, "Without WAN"). Revoking a device certificate removes the device from the network at its next authentication. Which directory, which CA and how devices are enrolled is decided by A09 (PD-090).

The two non-staff zones use WPA3-Personal (SAE), because transporters and visitors are not center staff and their phones are not center-managed. Their passphrases differ, are rotated at least at the start of every harvest for `cc-transport` and at least weekly for `cc-guest`, and are distributed by the logistics staff (transport) or shown at reception (visitors) (ASM-081). On these two SSIDs WPA2/WPA3 transition mode is permitted if older phones require it; staff SSIDs never fall back to WPA2. Client isolation is on for both, so two phones on the same SSID cannot reach each other.

### 3.3 Addressing inside a client zone

Every client zone follows the same host layout (ASM-083): `.1` is the gateway on the firewall, `.2` to `.49` are for static addresses and DHCP reservations, and the dynamic pool starts at `.50`. The pool size is twice the planned clients, rounded up to a multiple of 10, which absorbs device replacement and lease turnover without sizing for clients the case statement does not plan: Gerencia and Soporte 7 x 2 = 14, rounded to 20; Logística 22 x 2 = 44, rounded to 50; Transporte 20 x 2 = 40; Visitantes 10 x 2 = 20. Everything above the pool stays free inside the /24, so even the largest pool uses 50 of the 253 usable host addresses of a /24 that are not the gateway. Lease times are shortest where clients change most: 1 hour for visitors, 4 hours for transport and 12 hours for staff. The firewall serves DHCP and is the DNS resolver of every zone at its `.1` address. For the staff and logistics zones it forwards the names of the platform's AWS interface endpoints to a resolver reachable through the tunnel, so that those names resolve to the private endpoint addresses of the environment VPC and not to the public ones; where that resolver lives in the cloud is decided with the firewall table and the network module (PD-093).

### 3.4 Infrastructure zones

The case statement leaves the clients of Servidores locales to the design ("Por diseñar") and describes the zone as "Impresión y control de la red". The pattern places there the management interfaces of every network device and the code-sheet printers (ASM-087):

| Address range | Holds | Assignment |
|---|---|---|
| 10.4x.10.1 | Firewall: gateway of the zone and management address of the firewall | Static |
| 10.4x.10.2 to .9 | Management addresses of the PoE switch or switches | Static |
| 10.4x.10.10 to .19 | Reserved for a future local service | Not assigned |
| 10.4x.10.20 to .49 | Management addresses of the Wi-Fi access points; their number comes from the site survey (PD-092) | DHCP reservation by MAC address |
| 10.4x.10.50 to .59 | Code-sheet printers, one per reception post to start, that is two (section 5) | Static |
| 10.4x.10.60 to .254 | Free | Not assigned |

The zone has no dynamic pool, so a device plugged into a VLAN 110 port that is not reserved receives no address. Enlace a la nube, VLAN 120, exists only on the firewall's edge ports, never on the access switch trunk, so no client port can be bridged into it. It holds static addresses only: 10.4x.20.1 is the firewall's edge interface and the source address of the firewall's own traffic toward the hub, 10.4x.20.2 is the LAN side of the internet service equipment and 10.4x.20.3 the LAN side of the cellular backup router if that router is a separate device (ASM-088). The public addresses of the internet service belong to the provider and are not part of this plan.

## 4. Where each design element connects

### 4.1 Drone ground station

A10 left the zone of the drone ground station to this task (connectivity section 6). The ground station belongs to the contracted drone operator (ADR-002, ASM-023), so it is not a logistics post and must not sit next to the sack reception posts; it needs the private path through the tunnel to `application` (mission download), `drone_object_ingress` and the S3 interface endpoint in `object_storage` (connectivity section 6), so it cannot sit in Visitantes or Transporte, whose traffic never enters the tunnel; and it is not printing or network control, so it does not belong in Servidores locales.

It connects in **Soporte (VLAN 102)**, on `cc-support` or on a wired Soporte port, with a fixed DHCP reservation at **10.4x.2.10** (ASM-082). The pattern reads Soporte as the zone of the center's support functions, which include support to the field operation; it is not the network administration zone, which is Servidores locales. The ground station is enrolled as a device of the drone operator company and receives a network certificate whose RADIUS policy places it in VLAN 102 (PD-090). Its fixed address lets A12 write its rules to its /32 only, so the rest of Soporte gains no path to the drone tiers, and lets the traffic control of section 8 guarantee the drone uploads by source and destination. Adding a separate drone zone was considered and rejected: one visiting device does not justify a new VLAN, SSID and set of rules in every center, and the /32 gives the same isolation.

### 4.2 Network device management

The network devices are managed from **Servidores locales (10.4x.10.0/24)**, the zone of "control de la red". Every management interface (firewall, switches, access points) has its address in that zone, and management sessions to those addresses are accepted only from the `administration` tier of the environment VPC through the tunnel (addressing plan section 4.2: session-based administration without bastion hosts), by platform operators who enter AWS through IAM Identity Center with MFA (landing zone section 4). No user zone, including Gerencia and Soporte, can reach a management address (section 6). The devices send their logs, metrics and configuration backups to the cloud through the same tunnel, which gives A21 the data on the three WAN links that point 6 requires. If the tunnel and the backup link are both down, the firewall's local console port is the emergency path; it requires physical presence and is used only during an incident.

### 4.3 Center firewall

The firewall is the only device that routes. Its edge sits in Enlace a la nube, as A10 fixes; its LAN side is an 802.1Q trunk to the PoE switch carrying VLANs 101 to 105 and 110, with one subinterface and gateway per zone; it places VLANs 104 and 105 in an internet-only routing context, a virtual router or the firewall's equivalent, whose only route is the default route to the local internet breakout (section 6).

## 5. Logistics workstations and code-sheet printing

Logistics is the zone of "Recepción de sacos e impresiones". Its two desktops are the receiving posts: they connect by cable to Logística ports and run the collection-center application, through which the collection-center operator, signed in with MFA, records the arrival, acceptance or rejection of each sack and its trip (Part C, "en acopio (aceptado o rechazado)"). The 20 logistics phones join `cc-logistics` and read sack codes in the yard with the same application. Both reach the `application` tier through the tunnel, and both keep the encrypted outbox of the connectivity design, so a reception recorded during a WAN outage drains when the link returns, with its original time and responsible (ASM-079). At harvest each center receives up to 400 sacks a day; those events are small API calls inside the logistics traffic class of section 8.

The code-sheet printers sit in Servidores locales, at 10.4x.10.50 to .59, wired to the PoE switch. A receiving post asks the platform for the codes of sacks that arrived without one ("Impresión de hojas de códigos para los sacos que lleguen sin código"), receives the sheet through the tunnel and sends it to a printer. The print job goes from Logística (10.4x.3.0/24) to the printer addresses, routed by the firewall between VLAN 103 and VLAN 110 and allowed only on the printing protocols that A12 lists; it never crosses the WAN. Gerencia and Soporte may also print to the same printers. A printer never initiates traffic to a user zone, to the cloud or to the internet. Printing a sheet whose codes are already issued works without WAN; how codes are issued for a sack that arrives while the WAN is down is part of the sack and code design of A16 (PD-096).

## 6. Inter-zone policy intent

The matrix gives, for each source zone (rows) and destination (columns), whether a flow may be initiated. It is stateful: replies to an allowed flow are allowed. Every zone may reach its own gateway for DHCP, DNS and, in staff zones, 802.1X; that is not repeated in the cells. Anything not allowed is denied and logged. Gerencia and Soporte break out to the internet locally, while Logística and Servidores locales have no general internet access, which keeps the reception posts and the network devices off the open internet (ASM-089). A12 derives one or more rules with ports from each "Allow" cell and classifies each as permanent or temporary (PD-091, PD-083).

| Source \ Destination | Gerencia | Soporte | Logística | Transporte | Visitantes | Servidores locales | Cloud via tunnel | Internet, local breakout |
|---|---|---|---|---|---|---|---|---|
| Gerencia | Allow | Deny | Deny | Deny | Deny | Allow printing only | Allow `application` (reports web and center application) | Allow, staff class |
| Soporte | Deny | Allow | Deny | Deny | Deny | Allow printing only | Allow `application`; ground station 10.4x.2.10 only: also `drone_object_ingress` and `object_storage` | Allow, staff class |
| Logística | Deny | Deny | Allow | Deny | Deny | Allow printing only | Allow `application` | Deny, except the named time and update services A12 lists |
| Transporte | Deny | Deny | Deny | Deny (client isolation) | Deny | Deny | Deny: the transporter app uses the public API (connectivity section 7) | Allow; platform public API in the transport application class, the rest non-operational |
| Visitantes | Deny | Deny | Deny | Deny | Deny (client isolation) | Deny | Deny | Allow, non-operational class |
| Servidores locales (devices, printers) | Deny | Deny | Deny | Deny | Deny | Deny between devices; firewall to switches and access points allowed | Allow logs, metrics, configuration backup and time to `administration` | Deny, except named update services A12 lists as temporary |
| Cloud `administration` tier | Deny | Deny | Deny | Deny | Deny | Allow device management | Not applicable | Not applicable |
| Other cloud tiers | Deny | Deny | Deny | Deny | Deny | Deny | Not applicable | Not applicable |
| Internet | Deny | Deny | Deny | Deny | Deny | Deny | Not applicable | Not applicable |

The case statement's rule that "La VLAN de visitantes no enruta hacia logística ni gerencia" is met twice. Visitantes, and Transporte with it, live in the internet-only routing context of section 4.3, which has no route to any 10.4x network, to 10.40.0.0/13 or to the tunnel, so their packets toward logistics or management cannot be routed at all; and the policy denies the same flows as a second layer. The firewall itself, in Enlace a la nube, initiates only the IPsec tunnels to the hub over the primary or backup link, DNS and time toward the provider or the internet, and the management of switches and access points; nothing on the internet can initiate a flow to the center, and the cloud initiates only management from `administration`. The environment the tunnel reaches for each tier is set by A12 under the hub design (ASM-033, ASM-078).

## 7. Capacity

### 7.1 Reference demand per zone

The case statement gives the reference per active client, "2 Mbps de bajada y 0,5 Mbps de subida", and the WAN, "100 Mbps simétricos, compartidos con la operación". The table applies the reference to the planned clients of the inventory table, at 100 % active and at the assumed concurrency. The assumed concurrency is that **50 % of the planned clients of each zone are active at once at the harvest peak, rounded up to a whole client per zone** (ASM-084).

| Zone | Planned clients | Down at 100 % (x 2 Mbps) | Up at 100 % (x 0.5 Mbps) | Active at 50 %, rounded up | Down at 50 % | Up at 50 % |
|---|---|---|---|---|---|---|
| Gerencia | 7 | 14 Mbps | 3.5 Mbps | 4 | 8 Mbps | 2 Mbps |
| Soporte | 7 | 14 Mbps | 3.5 Mbps | 4 | 8 Mbps | 2 Mbps |
| Logística | 22 | 44 Mbps | 11 Mbps | 11 | 22 Mbps | 5.5 Mbps |
| Transporte | 20 | 40 Mbps | 10 Mbps | 10 | 20 Mbps | 5 Mbps |
| Visitantes | 10 | 20 Mbps | 5 Mbps | 5 | 10 Mbps | 2.5 Mbps |
| Total clients | 66 | 132 Mbps | 33 Mbps | 34 | 68 Mbps | 17 Mbps |

Servidores locales has no reference clients; its traffic is the network control class of section 8. The drone ground station is not a client of the inventory table and is sized by its own class.

### 7.2 Comparison with the WAN

The WAN is 100 Mbps each way, shared with the operation. The pattern plans with 90 Mbps each way and keeps 10 Mbps for IPsec encapsulation overhead, tunnel keepalives and routing control (ASM-085). Demand below includes the client reference of section 7.1, the 2 Mbps network control guarantee and the drone guarantee of 5 Mbps down and 20 Mbps up of section 8.

| Case | Down demand | Down against 90 Mbps | Up demand | Up against 90 Mbps |
|---|---|---|---|---|
| Clients only, 50 % active | 68 Mbps | 75.6 % | 17 Mbps | 18.9 % |
| Clients only, 100 % active | 132 Mbps | 146.7 % (exceeds by 42 Mbps; by 32 Mbps against the full 100 Mbps) | 33 Mbps | 36.7 % |
| All classes, 50 % active | 68 + 2 + 5 = 75 Mbps | 83.3 % | 17 + 2 + 20 = 39 Mbps | 43.3 % |
| All classes, 100 % active | 132 + 2 + 5 = 139 Mbps | 154.4 % (exceeds by 49 Mbps) | 33 + 2 + 20 = 55 Mbps | 61.1 % |

At the assumed concurrency the WAN carries all reference demand in both directions. The uplink is never the limit, even at 100 % active, which matters because drone uploads are uplink traffic. The downlink is the limit only if every planned client is active at once, and then by 49 Mbps; the traffic control of section 8 decides who absorbs that shortfall, and it is the non-operational traffic, not logistics or the drones.

## 8. Non-operational traffic control

The firewall classifies every flow by source zone, and for the ground station and transport also by destination, and applies a guaranteed minimum and a cap per class, in priority order. Upload classes are shaped on the WAN egress; download classes are shaped on the LAN-side egress of each zone, which TCP flows follow. Every value is a design value (ASM-086); none is a measurement.

| Class | Members | Priority | Guaranteed down / up | Cap down / up |
|---|---|---|---|---|
| Network control | Firewall tunnels and management, device logs and metrics from Servidores locales, RADIUS, DNS, time | 1 | 2 / 2 Mbps | None |
| Logistics | All Logística traffic (reception, code sheets, sack reads) | 2 | 44 / 11 Mbps, the full reference of its 22 clients | None |
| Drone | Ground station 10.4x.2.10 to `application`, `drone_object_ingress` and `object_storage` | 3 | 5 / 20 Mbps | None |
| Staff | All Gerencia and Soporte traffic except the ground station | 4 | 28 / 7 Mbps, the full reference of their 14 clients | None |
| Transport application | Transporte to the platform's public API endpoints | 4 | 2 / 1 Mbps | None |
| Non-operational | Transporte except the platform API, and all Visitantes | 5, lowest | None | Transporte 20 / 5 Mbps, Visitantes 10 / 2.5 Mbps; 2 / 0.5 Mbps per client |

The guarantees add up to 2 + 44 + 5 + 28 + 2 = 81 Mbps down and 2 + 11 + 20 + 7 + 1 = 41 Mbps up, both within the 90 Mbps plan, so every guarantee can be honored at the same time. Logistics and the staff zones are guaranteed their full reference at 100 % active, not only at 50 %, so reception never depends on the concurrency assumption. Unused guaranteed capacity is lent to lower classes and returned when its owner needs it.

Non-operational traffic is rate-limited at three levels: per client at the case statement's reference, so one phone cannot take more than 2 Mbps down and 0.5 Mbps up; per zone at the caps above, which equal the 50 % reference of each zone (10 transport phones x 2 = 20 Mbps, 5 visitor phones x 2 = 10 Mbps); and by priority, because it is served last. In the worst case of section 7.2, with every planned client active and every operational class at its guarantee, the operational classes take 81 Mbps and non-operational traffic receives the remaining 90 - 81 = 9 Mbps down, below its 30 Mbps combined cap; on the uplink it keeps its full 7.5 Mbps cap. The drone class may burst above its guarantee into unused uplink: with every other class at its guarantee or cap, 90 - 2 - 11 - 7 - 1 - 7.5 = 61.5 Mbps up remain for it. As pure arithmetic, not an object size, each gigabyte (8,000 megabits) takes 8,000 / 20 = 400 seconds at the drone guarantee and 8,000 / 61.5 = about 130 seconds at that burst.

When the primary service fails and the cellular backup carries the tunnels, the same classes and priorities apply to whatever capacity the backup offers; no backup capacity is assumed. Non-operational traffic is then suspended, so the backup carries only operational classes (ASM-086).

## 9. Hardware device classes

The pattern specifies device classes only. Models and prices belong to A23 (PD-095), and the number of access points and switch ports to the site survey (PD-092).

| Device class | Required capabilities | Zone |
|---|---|---|
| Perimeter firewall, one per center (ASM-088) | Stateful zone policy between VLAN subinterfaces; IPsec site-to-site VPN with two tunnels; a second WAN interface for the cellular backup, integrated or through a separate cellular router; an internet-only routing context; DHCP server and DNS resolver; RADIUS server for EAP-TLS with local revocation checking; traffic classification by zone, address and destination with guarantees and caps; log and metric export. | Enlace a la nube (edge), Servidores locales (management) |
| Cellular backup router, if not integrated in the firewall | LTE or a later generation (connectivity section 11); handoff to the firewall's second WAN interface. | Enlace a la nube |
| Managed PoE switch | 802.1Q VLAN trunks and access ports; 802.1X on access ports; MAC-based port security for printers; PoE for the access points; management address in VLAN 110. | Servidores locales (management) |
| Wi-Fi access points | Five SSIDs per radio, each mapped to its VLAN tag; WPA3-Enterprise and WPA3-Personal; client isolation per SSID; per-client rate limits; management address in VLAN 110. | Servidores locales (management) |
| Network printers for code sheets | Wired Ethernet; static address. | Servidores locales |

## 10. Behavior when the WAN fails

If the primary internet service fails, the firewall re-establishes the same IPsec tunnels over the cellular backup (ASM-078) and the traffic classes keep operational traffic first. If both links fail, the LAN keeps working: staff devices still join, because RADIUS is local; the receiving posts and logistics phones keep recording receptions in their outbox (ASM-079); printing of already issued codes continues, because it never crosses the WAN; the ground station keeps flight records and objects in its buffer until the tunnel returns (ADR-002); and visitors and transport lose internet access. A21 observes each of the three WAN links and their traffic classes (PD-094).

## 11. Compliance with the case statement and the issue

| Requirement | Where it is met |
|---|---|
| Issue criterion: one repeatable pattern for Cauca, Huila and Tolima, not three networks; "El mismo patrón, tres veces" | Section 2: two parameters, every other value shared. |
| Issue criterion: VLANs or SSIDs per zone, guests isolated from logistics and management; "WLAN segmentada por zona, con SSID propio. La VLAN de visitantes no enruta hacia logística ni gerencia." | Section 3.1 (VLAN, SSID, authentication, gateway, pool per zone) and section 6 (matrix and internet-only routing context). |
| Issue criterion: logistics workstations for receiving and for printing code sheets; "Puestos de logística que registran la recepción", "Impresión de hojas de códigos" | Section 5, with the printers in section 3.4. |
| Issue criterion: capacity with 100 Mbps symmetric WAN and 2 / 0.5 Mbps per active client | Section 7, per zone and in total, at 50 % (ASM-084) and at 100 %. |
| "Hay control de tráfico no operativo" | Section 8. |
| "Wifi y firewall del inventario" | Sections 3, 4.3 and 9. |
| Lead note: zone of the drone ground station | Section 4.1. |
| Lead note: where network devices are managed from | Section 4.2. |
| Lead note: device classes only, no access point count | Section 9 and PD-092. |

## 12. Decision-log entries

| ID | Summary |
|---|---|
| ASM-080 | VLAN ID is 100 plus the third octet of the zone network (101 to 105, 110, 120), the same in all three centers; VLAN 1 is unused. |
| ASM-081 | Five SSIDs with English names shared by the three centers; staff SSIDs use WPA3-Enterprise EAP-TLS with device certificates tied to A09 identities and validated by RADIUS on the firewall; transport and visitor SSIDs use WPA3-Personal with rotated passphrases and client isolation. |
| ASM-082 | Soporte is the zone of the center's support functions, not network administration; the drone ground station connects in Soporte at 10.4x.2.10. |
| ASM-083 | Host layout of a client zone: `.1` gateway, `.2` to `.49` static and reserved, pool from `.50` sized at twice the planned clients rounded up to 10; leases of 12 h, 4 h and 1 h. |
| ASM-084 | 50 % of the planned clients of each zone are active at once at the harvest peak, rounded up per zone. |
| ASM-085 | 90 Mbps of the 100 Mbps WAN are planned each way; 10 Mbps are kept for IPsec overhead and control. |
| ASM-086 | Traffic classes, priorities, guarantees and caps of section 8 are design values; non-operational traffic is suspended on the backup link. |
| ASM-087 | Servidores locales holds the management interfaces of all network devices and the code-sheet printers, one per reception post to start. |
| ASM-088 | One firewall per center, with the cellular backup integrated or on a separate router in Enlace a la nube, and the edge addresses of section 3.4. |
| ASM-089 | Gerencia and Soporte have local internet breakout; Logística and Servidores locales have no general internet access. |
| PD-090 | A09 fixes the directory, the private CA and the enrollment of center devices and the ground station. |
| PD-091 | A12 derives port-level rules from the matrix of section 6 and classifies them. |
| PD-092 | The site survey of each center sets the number and placement of access points and switch ports. |
| PD-093 | Name resolution of the cloud interface endpoints from the centers is designed. |
| PD-094 | A21 observes the three WAN links per traffic class. |
| PD-095 | A23 prices the device classes of section 9. |
| PD-096 | A16 defines how codes are issued for a sack that arrives while the WAN is down. |

## 13. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Point 3 (inventory table, WLAN, capacity, traffic control, WAN), Part C (sack states), Part D (collection centers), point 2 (MFA). |
| [`docs/addressing-plan.md`](addressing-plan.md) | Center /16s, 10.40.0.0/13 summary, `transit`, `administration`, `application`, `drone_object_ingress` and `object_storage` tiers. |
| [`docs/connectivity.md`](connectivity.md) | Sections 6, 7 and 9: ground station path, transporter app path, center WAN, backup link, outbox. |
| [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) | Mission preload at the collection center; contracted drone operator. |
| [`docs/landing-zone.md`](landing-zone.md) | Section 4: identity entry points; PD-072 `CollectionCenter` tag. |
| [`docs/decision-log.md`](decision-log.md) | ASM-023, ASM-033, ASM-078, ASM-079 and the entries of section 12. |
