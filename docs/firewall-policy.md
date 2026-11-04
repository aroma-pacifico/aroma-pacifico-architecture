# Perimeter and Cloud Firewall Policy

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR2 / JR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#12 (A12) |
| Builds on | [`docs/collection-center-network.md`](collection-center-network.md) (A11, section 6), [`docs/connectivity.md`](connectivity.md) (A10, sections 6, 7 and 9), [`docs/addressing-plan.md`](addressing-plan.md) (A07, section 4), [`docs/landing-zone.md`](landing-zone.md) (A08), [`docs/security-and-identity.md`](security-and-identity.md) (A09), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) |
| Feeds | B04 network module (aroma-pacifico/aroma-pacifico-roadmap#24), B05 collection-center association (aroma-pacifico/aroma-pacifico-roadmap#25), B06 security module (aroma-pacifico/aroma-pacifico-roadmap#26), aroma-pacifico/aroma-pacifico-roadmap#27, A19 complete diagram (aroma-pacifico/aroma-pacifico-roadmap#19), A21 observability (aroma-pacifico/aroma-pacifico-roadmap#34), A23 cost (aroma-pacifico/aroma-pacifico-roadmap#36) |
| Decision-log entries | PD-011, PD-052, PD-083, PD-091 and PD-093 (closed here), ASM-120 to ASM-129, PD-120 to PD-125 |

## 1. Purpose and scope

Point 3 of the [case statement](case/case-statement.pdf) asks for "Firewall de perímetro en cada centro y firewall de nube. Tabla de políticas: origen, destino, puerto, acción y si la regla queda en el piloto o es temporal", and repeats that "La VLAN de visitantes no enruta hacia logística ni gerencia". Epic 6 then codes, in the security module, the "reglas de firewall marcadas como permanentes en el diagrama". This document is that policy table: one table for the perimeter firewall of a collection center, written once with `10.4x` for the three centers, and one table for the cloud, enforced by managed AWS mechanisms. Quotations are in the original Spanish, which prevails over the English rendering (ASM-001).

The tables derive from earlier work and do not redesign it. The zones, VLANs, gateways and the inter-zone intent matrix are A11 (section 6 of the collection-center network); the IPsec tunnels to the hub, the backup link, the `transit` tier and the public endpoints are A10 (sections 6, 7 and 9 of the connectivity design); the CIDRs are A07; the identities, TLS policies and the upload grants of the drone are A09. An "Allow" cell of the A11 matrix is treated as an upper bound: a rule opens only what a named flow needs, so some rules below are narrower than their cell (section 7 lists each narrowing). No value here is a measurement.

## 2. How to read the tables

| Column | Meaning |
|---|---|
| ID | `CC-` for the center perimeter firewall, `CL-` for the cloud, then the group code and a number. IDs are stable; a removed rule keeps its ID. |
| Group | One of the groups of section 3. |
| Source, Destination | Addresses from A07 and A11. `10.4x` is 10.41 Cauca, 10.42 Huila, 10.43 Tolima. Cloud tiers are the `pdn` tier blocks of A07, for example `application` 10.82.48.0/20. |
| Port / protocol | Real ports. "IP 50" is the ESP protocol number. |
| Action | Allow, Deny (always logged), Route, Blackhole, or Redirect. |
| Mechanism (cloud only) | The managed AWS control that enforces the rule. |
| Validity | **Permanent** stays for the whole pilot. **Temporary** names the condition that removes it. |
| Before production | **Audit**: checked against the deployed configuration and its logs by a person other than its author before `pdn` go-live, and recorded in the infrastructure release notes (PD-125). **Review**: confirmed in the plan review of the module that codes it. Every temporary rule is Audit, and its removal is audited too. |

The center firewall evaluates rules top-down and the first match wins, so the center table is written in evaluation order. Traffic between two hosts of the same zone is switched, not routed, and never reaches the firewall; the `cc-transport` and `cc-guest` SSIDs isolate clients from each other (A11 section 3.2). Every rule is stateful on the firewall and in security groups: the reply to an allowed flow is allowed. Network ACLs are stateless, so their rows state the return ports.

## 3. Rule groups

| Group code | Group | What it covers |
|---|---|---|
| GST | Visitor and transport isolation | The explicit denies that keep Visitantes and Transporte away from logistics and management, and their internet exit. |
| BAS | Zone services | DHCP, DNS, time, 802.1X and RADIUS, device logs to the firewall. |
| LNK | Cloud-to-collection-center links | IPsec tunnels, primary and backup, endpoint name resolution, the hub route filtering and the temporary test path. |
| ADM | Administration | Management of the center devices from the cloud, device log export, firmware windows, commissioning. |
| DRN | Drone objects | The ground station at 10.4x.2.10 toward mission download, upload grants and Amazon S3. |
| LOG | Logistics | Reception posts and phones toward the application, code-sheet printing. |
| STF | Staff | Gerencia and Soporte toward the application, printing and the internet. |
| TEL | Telemetry | Farm telemetry to AWS IoT Core and transporter positions to the public API; neither crosses the tunnel. |
| PUB | Public lookup | The unauthenticated sack lookup and its front door. |
| APP, DAT, EGR | Cloud application, data and egress | Cloud-side controls on the private application path, the data tier and outbound traffic. |
| DEF | Default | The final explicit deny of each table. |

## 4. Perimeter firewall of each collection center

One table serves the three centers. Address objects used in it are fixed by A11 (sections 3.1, 3.4 and 4.1).

| Object | Value |
|---|---|
| Management addresses | Firewall 10.4x.10.1, switches 10.4x.10.2 to .9, access points 10.4x.10.20 to .49 |
| Printers | 10.4x.10.50 to .59 |
| Ground station | 10.4x.2.10/32 |
| Firewall edge | 10.4x.20.1, the source of the firewall's own traffic toward the hub |
| Private ranges | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 100.64.0.0/10 |
| Tunnel peers | The two outside addresses that AWS assigns to the center's Site-to-Site VPN connection |

| ID | Group | Source | Destination | Port / protocol | Action | Justification | Validity | Before production |
|---|---|---|---|---|---|---|---|---|
| CC-GST-01 | GST | Visitantes 10.4x.5.0/24 | Logística 10.4x.3.0/24 and Gerencia 10.4x.1.0/24 | Any | Deny | "La VLAN de visitantes no enruta hacia logística ni gerencia". Second layer behind the internet-only routing context, which has no route to any 10.4x network (A11 section 4.3). | Permanent | Audit |
| CC-GST-02 | GST | Transporte 10.4x.4.0/24 | Logística 10.4x.3.0/24 and Gerencia 10.4x.1.0/24 | Any | Deny | Transporters are not staff; the matrix denies them every user and management zone (A11 section 6). | Permanent | Audit |
| CC-GST-03 | GST | Visitantes, Transporte | Private ranges, 10.40.0.0/13 included, and every firewall interface except the zone's own gateway | Any | Deny | Covers Soporte, Servidores locales, the tunnel, other centers and the cloud. | Permanent | Audit |
| CC-BAS-01 | BAS | Each client zone | Its own gateway 10.4x.Z.1 | UDP 67 (DHCP), UDP and TCP 53 (DNS), UDP 123 (NTP) | Allow | Gateway services of A11 section 3.3; the firewall serves time to every zone (ASM-123). | Permanent | Review |
| CC-BAS-02 | BAS | Switches and access points (management addresses except .1) | Firewall 10.4x.10.1 | UDP 1812 and 1813 (RADIUS) | Allow | 802.1X EAP-TLS of Gerencia, Soporte and Logística, validated by RADIUS on the firewall (A11 section 3.2). | Permanent | Audit |
| CC-BAS-03 | BAS | Switches and access points | Firewall 10.4x.10.1 | UDP 514 (syslog), UDP 162 (SNMP traps), UDP 123 (NTP) | Allow | Device logs and time stay local; the firewall exports them (CC-ADM-02). | Permanent | Review |
| CC-BAS-04 | BAS | Firewall 10.4x.10.1 | Switches and access points | TCP 22, TCP 443, UDP 161 (SNMPv3) | Allow | "Firewall to switches and access points allowed" (A11 section 6). | Permanent | Review |
| CC-LNK-01 | LNK | Firewall primary WAN address | Tunnel peers, and the reverse | UDP 500 (IKEv2), UDP 4500 (NAT traversal, ESP in UDP) | Allow | The two IPsec tunnels to the hub (ASM-078, ASM-120). The edge sits behind the internet service equipment at 10.4x.20.2 (A11 section 3.4), so NAT traversal is the normal case. | Permanent | Audit |
| CC-LNK-02 | LNK | Firewall primary WAN address | Tunnel peers, and the reverse | IP 50 (ESP) | Allow | Needed only if the provider hands the firewall a public address without NAT; disabled otherwise. | Permanent | Audit |
| CC-LNK-03 | LNK | Firewall cellular backup WAN address | Tunnel peers, and the reverse | UDP 500, UDP 4500 | Allow | The backup link re-establishes the same tunnels (ASM-078); cellular access is behind carrier NAT, so only NAT traversal is used. | Permanent | Audit |
| CC-LNK-04 | LNK | Firewall 10.4x.20.1 | Route 53 Resolver inbound endpoint addresses in `pdn` `administration` 10.82.96.0/20 | UDP and TCP 53 | Allow | Conditional forwarding of the exact host names of the platform endpoints so they resolve to private addresses (ASM-122, closes PD-093). | Permanent | Review |
| CC-LNK-05 | LNK | Firewall WAN addresses | Internet NTP servers and the provider's DNS resolvers | UDP 123; UDP and TCP 53 | Allow | The firewall's own time and the resolution of internet names for the zones that break out (A11 section 6). | Permanent | Review |
| CC-LNK-06 | LNK | Logística 10.4x.3.0/24 | `qa` `application` 10.81.48.0/20 | TCP 443 | Allow | Acceptance tests of the logistics app from a real center against `qa` (ASM-121). | Temporary: removed when the center's acceptance is signed off, or at the latest before `pdn` go-live | Audit |
| CC-LNK-07 | LNK | Any address arriving through the tunnel | Any center address not matched by CC-ADM-01 | Any | Deny | The cloud initiates only management (A11 section 6). | Permanent | Audit |
| CC-LNK-08 | LNK | Internet, on either WAN interface | Any center address, the firewall included | Any other than CC-LNK-01 to 03 | Deny | Nothing on the internet initiates a flow to the center; no management on the WAN. | Permanent | Audit |
| CC-LNK-09 | LNK | Any center zone | Other centers' /16 and the rest of 10.40.0.0/13 | Any | Deny | No center-to-center path; the hub has no such route either (CL-LNK-02). | Permanent | Review |
| CC-ADM-01 | ADM | `pdn` `administration` 10.82.96.0/20, through the tunnel | Management addresses | TCP 22 (SSH), TCP 443 (HTTPS management API) | Allow | Session-based administration from the cloud (A07 section 4.2, A11 section 4.2). It also pulls configuration backups and pushes the certificate revocation list to the RADIUS service (ASM-123). | Permanent | Audit |
| CC-ADM-02 | ADM | Firewall 10.4x.20.1 | Interface endpoints for Amazon CloudWatch Logs, Amazon CloudWatch and IAM Roles Anywhere in `pdn` `administration` 10.82.96.0/20 | TCP 443 | Allow | Logs and metrics of the three WAN links and the devices for A21, with short-lived credentials and no static key (ASM-123, PD-123). | Permanent | Audit |
| CC-ADM-03 | ADM | Every client zone | Management addresses, and management services (TCP 22, 443) on every firewall interface | Any | Deny | "No user zone, including Gerencia and Soporte, can reach a management address" (A11 section 4.2). | Permanent | Audit |
| CC-ADM-04 | ADM | Printers | Any | Any | Deny | A printer never initiates traffic (A11 section 5). | Permanent | Review |
| CC-ADM-05 | ADM | Management addresses | Vendor update service host names of the procured device class | TCP 443 | Allow | Firmware and signature updates through the local breakout, the only internet exit of Servidores locales (A11 section 6). | Temporary: opened per approved maintenance window outside the harvest freeze of point 4, closed at its end | Audit |
| CC-ADM-06 | ADM | Commissioning laptop 10.4x.10.60/32 on one wired VLAN 110 port | Management addresses | TCP 22, TCP 443 | Allow | Initial configuration before the tunnel exists (ASM-128). | Temporary: removed, and the port disabled, when CC-ADM-01 works through the tunnel and the site is accepted | Audit |
| CC-ADM-07 | ADM | Servidores locales 10.4x.10.0/24 | Internet | Any | Deny | No general internet for devices (ASM-089). | Permanent | Review |
| CC-DRN-01 | DRN | Ground station 10.4x.2.10/32 | `pdn` `application` 10.82.48.0/20 | TCP 443 | Allow | Mission package download and upload-grant request under the pilot identity (A10 section 6, A09 section 6). | Permanent | Audit |
| CC-DRN-02 | DRN | Ground station 10.4x.2.10/32 | `pdn` `drone_object_ingress` 10.82.32.0/20 | TCP 443 | Allow | Interface endpoints of the drone object ingress (A10 section 6; endpoints named by A15, PD-121). | Permanent | Audit |
| CC-DRN-03 | DRN | Ground station 10.4x.2.10/32 | `pdn` `object_storage` 10.82.80.0/20 (Amazon S3 interface endpoint) | TCP 443 | Allow | Multipart upload of flight records and objects with 15-minute pre-signed URLs (ASM-093). | Permanent | Audit |
| CC-DRN-04 | DRN | Ground station 10.4x.2.10/32 | AWS IoT Core data endpoint of the `pdn` account, public, through the local breakout | TCP 8883 (MQTT over TLS, mutual X.509) | Allow | The kit's grant topic (ASM-093) on the public endpoint, as from a municipal seat (ASM-126). | Permanent | Audit |
| CC-DRN-05 | DRN | Ground station 10.4x.2.10/32 | Any other destination | Any | Deny | The ground station has its /32 rules only (A11 section 4.1). | Permanent | Audit |
| CC-DRN-06 | DRN | Soporte 10.4x.2.0/24 except .10 | `drone_object_ingress` 10.82.32.0/20 and `object_storage` 10.82.80.0/20 | Any | Deny | Only the ground station has the drone path. | Permanent | Review |
| CC-LOG-01 | LOG | Logística 10.4x.3.0/24 | `pdn` `application` 10.82.48.0/20 | TCP 443 | Allow | Reception accepted or rejected, sack reads, code-sheet requests and staff sign-in, through the application API and Amazon Cognito interface endpoints (A10 section 9, A11 section 5). | Permanent | Audit |
| CC-LOG-02 | LOG | Logística 10.4x.3.0/24 | Printers | TCP 631 (IPP, with TLS where the printer offers it) | Allow | Code sheets for sacks without a code ("Impresión de hojas de códigos"); never crosses the WAN (ASM-124). | Permanent | Review |
| CC-LOG-03 | LOG | Logística 10.4x.3.0/24 | Allow-list of host names: operating system updates, app store and device management of the managed device platform | TCP 443 | Allow | The named update services the A11 matrix leaves to this task (ASM-125). | Permanent | Audit |
| CC-LOG-04 | LOG | Logística 10.4x.3.0/24 | Internet, other | Any | Deny | No general internet for reception posts (ASM-089). | Permanent | Review |
| CC-STF-01 | STF | Gerencia 10.4x.1.0/24, Soporte 10.4x.2.0/24 | `pdn` `application` 10.82.48.0/20 | TCP 443 | Allow | Reports web and center application with staff sign-in (A11 section 6). | Permanent | Audit |
| CC-STF-02 | STF | Gerencia, Soporte | Printers | TCP 631 | Allow | "Gerencia and Soporte may also print" (A11 section 5). | Permanent | Review |
| CC-STF-03 | STF | Gerencia, Soporte | Each other, Logística, Transporte, Visitantes | Any | Deny | Matrix of A11 section 6. | Permanent | Review |
| CC-STF-04 | STF | Gerencia, Soporte | Internet (public addresses) | TCP 80, TCP 443, UDP 443 | Allow | Local breakout in the staff class (ASM-089, ASM-125). | Permanent | Review |
| CC-STF-05 | STF | Gerencia, Soporte | Internet, other ports | Any | Deny | Only web ports are needed by staff. | Permanent | Review |
| CC-TEL-01 | TEL | Any center zone | `pdn` `telemetry_ingress` 10.82.16.0/20 | Any | Deny | Farm telemetry never crosses a center; the tier is not routed from the centers (A10 section 4.2, CL-LNK-02). | Permanent | Review |
| CC-TEL-02 | TEL | Transporte 10.4x.4.0/24 | Public application API front door host name | TCP 443 | Allow | Transporter positions and custody events from the yard, transport application class (A10 section 7, A11 section 8). | Permanent | Review |
| CC-PUB-01 | PUB | Gerencia, Soporte, Transporte, Visitantes | Public lookup front door host name | TCP 443 | Allow | The lookup is public and is reached only through the local breakout, never the tunnel. Logística reads sacks through the application (CC-LOG-01). | Permanent | Review |
| CC-GST-04 | GST | Transporte, Visitantes | Internet (public addresses) | TCP 25 (SMTP) | Deny | Guest phones must not relay mail from the center's address. | Permanent | Review |
| CC-GST-05 | GST | Transporte, Visitantes | Internet (public addresses) | Any other | Allow | Non-operational class, capped per zone and per client at 2 / 0.5 Mbps (A11 section 8, ASM-125). | Permanent | Review |
| CC-DEF-01 | DEF | Any | Any | Any | Deny | Explicit default deny, logged; denied flows feed A21 (PD-094). | Permanent | Audit |

## 5. Cloud policy

### 5.1 Enforcement mechanisms

The cloud has no firewall appliance. Each rule is enforced by the managed control that sits where the traffic enters, and the security groups, prefix lists, network ACLs and endpoint policies belong to the security and network stacks of ADR-003 (PC-IAC-022). Center addresses are grouped in customer-managed prefix lists, owned by the network account and shared through AWS Resource Access Manager, so a security group names a list and not 9 CIDRs (ASM-120). Network ACLs cannot reference prefix lists, so their rows list CIDRs.

| Prefix list | Entries |
|---|---|
| `cc-firewalls` | 10.41.20.1/32, 10.42.20.1/32, 10.43.20.1/32 |
| `cc-staff` | 10.4x.1.0/24 and 10.4x.2.0/24 of the three centers (6 entries) |
| `cc-logistics` | 10.41.3.0/24, 10.42.3.0/24, 10.43.3.0/24 |
| `cc-ground-stations` | 10.41.2.10/32, 10.42.2.10/32, 10.43.2.10/32 |
| `cc-device-mgmt` | 10.4x.10.0/27, 10.4x.10.32/28 and 10.4x.10.48/31 of the three centers, that is .1 to .49 without the printers (9 entries) |

### 5.2 Policy table

| ID | Group | Source | Destination | Port / protocol | Action | Mechanism | Justification | Validity | Before production |
|---|---|---|---|---|---|---|---|---|---|
| CL-LNK-01 | LNK | VPN attachments of the three centers | `pdn` `application` 10.82.48.0/20, `drone_object_ingress` 10.82.32.0/20, `object_storage` 10.82.80.0/20, `administration` 10.82.96.0/20 | Routing | Route to the `pdn` VPC attachment | Transit gateway route table `centers`, static routes | Centers reach only these four tiers of production (ASM-121). | Permanent | Audit |
| CL-LNK-02 | LNK | VPN attachments of the three centers | 10.80.0.0/13 except the four routes of CL-LNK-01, and 10.40.0.0/13 | Routing | Blackhole | Route table `centers`, blackhole routes; no VPN route is propagated into it | No path to `public_query`, `telemetry_ingress`, `data`, `transit`, `dev`, `qa` or another center. | Permanent | Audit |
| CL-LNK-03 | LNK | `pdn` VPC attachment | 10.41.0.0/16, 10.42.0.0/16, 10.43.0.0/16 to their VPN attachment; rest of 10.40.0.0/13 blackhole | Routing | Route, Blackhole | Route table `pdn`, static routes | The cloud reaches each center by its own /16 (A07 section 2) and a fourth department only when added. | Permanent | Audit |
| CL-LNK-04 | LNK | `dev` and `qa` VPC attachments | 10.40.0.0/13 | Routing | Blackhole | Route tables `dev` and `qa` | Non-production never reaches a center. | Permanent | Review |
| CL-LNK-05 | LNK | `qa` VPC attachment and VPN attachments | `qa` `application` 10.81.48.0/20 and 10.4x.3.0/24 | Routing | Route | Static routes added to `centers` and `qa`, mirrored by a `qa` security group rule from `cc-logistics` on TCP 443 | Pairs with CC-LNK-06. | Temporary: removed with CC-LNK-06 | Audit |
| CL-LNK-06 | LNK | Center firewall WAN | Hub | UDP 500, UDP 4500, IP 50 | Allow | One AWS Site-to-Site VPN connection per center on the transit gateway: IKEv2 only, static routing with the center /16 as its only route, two tunnels, certificate-based authentication of the customer gateway (ASM-120, PD-120) | A static route cannot be widened by a misconfigured or compromised center firewall, unlike a learned BGP route. | Permanent | Audit |
| CL-LNK-07 | LNK | Private tiers of `pdn` | 10.40.0.0/13 | Routing | Route to the transit gateway | VPC route tables: only `application`, `drone_object_ingress`, `object_storage` and `administration` carry the route | The other tiers cannot even answer a center. | Permanent | Review |
| CL-LNK-08 | LNK | 10.40.0.0/13 and 10.82.0.0/16 | `transit` /28 subnets | All | Allow; any other source Deny | Network ACL of `transit` | Attachment subnets pass only center and VPC traffic. | Permanent | Review |
| CL-APP-01 | APP | `cc-staff`, `cc-logistics`, `cc-ground-stations`; application tier | Application API and Amazon Cognito interface endpoints in `application` | TCP 443 | Allow | Security group of the endpoints | Pairs with CC-LOG-01, CC-STF-01, CC-DRN-01. | Permanent | Audit |
| CL-APP-02 | APP | Callers through the endpoint | The platform's private APIs | HTTPS | Allow only the platform's API ARNs | VPC endpoint policy on `execute-api`; API resource policy that allows only `aws:SourceVpce` of that endpoint | Center traffic cannot reach any other API, and the private API cannot be called from anywhere else. | Permanent | Audit |
| CL-APP-03 | APP | 10.40.0.0/13 and 10.82.0.0/16 | `application` subnets | Inbound TCP 443 from those ranges and TCP 1024 to 65535 (returns) from any; outbound TCP 1024 to 65535 to those ranges and TCP 443 to any | Allow; rest Deny | Network ACL of `application` | Stateless second layer; outbound TCP 443 stays open because gateway endpoint destinations are AWS public ranges, which an ACL cannot name, and security groups narrow it. | Permanent | Review |
| CL-DRN-01 | DRN | `cc-ground-stations` | Amazon S3 interface endpoint in `object_storage` and interface endpoints in `drone_object_ingress` | TCP 443 | Allow | Security groups of the endpoints | Pairs with CC-DRN-02 and CC-DRN-03. | Permanent | Audit |
| CL-DRN-02 | DRN | Any principal through the S3 interface endpoint | Drone object bucket | `s3:PutObject`, `s3:AbortMultipartUpload` | Allow only on that bucket | VPC endpoint policy | The endpoint cannot be used to read or to write any other bucket. | Permanent | Audit |
| CL-DRN-03 | DRN | Ground stations at a center or at a municipal seat (public S3 endpoint) | Drone object bucket | HTTPS | Deny unless TLS 1.2 or later and signed by the upload-grant role | S3 bucket policy: `aws:SecureTransport`, `s3:TlsVersion`, `aws:PrincipalArn`; Block Public Access | "Objetos del dron no se publican en claro" (point 2); the public path has no network rule, so the bucket policy is its firewall (A09 section 7.1). | Permanent | Audit |
| CL-DRN-04 | DRN | 10.41.2.10/32, 10.42.2.10/32, 10.43.2.10/32 and 10.82.0.0/16 | `drone_object_ingress` and `object_storage` subnets | Inbound TCP 443 from those sources and TCP 1024 to 65535 (returns) from any; outbound TCP 1024 to 65535 to those sources and TCP 443 to any, for the reception functions | Allow; rest Deny | Network ACLs of both tiers | Stateless second layer. | Permanent | Review |
| CL-TEL-01 | TEL | Internet: farm nodes, drone kits | AWS IoT Core data endpoint | TCP 8883 (MQTT over TLS) | Allow with mutual X.509 | IoT Core domain configuration with a TLS 1.2 minimum policy; per-thing IoT policy (A09 section 6) | Telemetry ingestion (ADR-003, A10 section 4.2). WAF does not apply to IoT Core. | Permanent | Audit |
| CL-TEL-02 | TEL | Internet: farm nodes | AWS IoT Core data endpoint | TCP 443, ALPN `x-amzn-mqtt-ca` | Allow with mutual X.509 | Same | Alternative where 8883 is blocked (A10 section 4.2). | Permanent | Review |
| CL-TEL-03 | TEL | Internet: farmer and transporter apps | Public application API front door | TCP 443 | Allow | AWS WAF web ACL (ASM-129); Cognito token validation | Positions, custody events and sack registration (A10 sections 5 and 7). | Permanent | Audit |
| CL-TEL-04 | TEL | Any | `telemetry_ingress` subnets | Any | Deny | Network ACL; no interface endpoint is created in the pilot | No private telemetry path is needed (A10 section 4.2); the reservation stays. | Permanent | Review |
| CL-ADM-01 | ADM | `cc-firewalls` | Route 53 Resolver inbound endpoint, two addresses in `administration` | UDP and TCP 53 | Allow | Security group of the resolver endpoint | Pairs with CC-LNK-04 (ASM-122). | Permanent | Audit |
| CL-ADM-02 | ADM | `cc-firewalls`; 10.82.0.0/16 | CloudWatch Logs, CloudWatch and IAM Roles Anywhere interface endpoints in `administration` | TCP 443 | Allow | Security group of the endpoints; endpoint policies limited to the platform's log groups and trust anchor | Pairs with CC-ADM-02. | Permanent | Audit |
| CL-ADM-03 | ADM | Management session host in `administration` | `cc-device-mgmt` | Outbound TCP 22, TCP 443; no inbound rule | Allow | Security group of the session host (PD-123) | Pairs with CC-ADM-01. | Permanent | Audit |
| CL-ADM-04 | ADM | Firewall /32s, 10.40.0.0/13 replies, 10.82.0.0/16 | `administration` subnets | Inbound UDP and TCP 53 and TCP 443 from the firewall /32s, TCP 1024 to 65535 from 10.40.0.0/13, TCP 443 from the VPC; outbound UDP and TCP 1024 to 65535 to the firewall /32s, TCP 22 and 443 to 10.40.0.0/13, TCP 1024 to 65535 to the VPC | Allow; rest Deny | Network ACL of `administration` | Stateless second layer. | Permanent | Review |
| CL-ADM-05 | ADM | 0.0.0.0/0 | Any security group | TCP 22, TCP 3389 | Deny (detected) | AWS Config managed rules `restricted-ssh`, `vpc-sg-open-only-to-authorized-ports`, `vpc-default-security-group-closed` | People operate AWS through IAM Identity Center with MFA (A08 section 4), never through an open port. | Permanent | Audit |
| CL-PUB-01 | PUB | Internet | Public lookup front door (A16, PD-051) | TCP 443, TLS 1.2 minimum | Allow GET and HEAD on the lookup path; default action Block | AWS WAF web ACL: AWS managed rule groups `AWSManagedRulesCommonRuleSet`, `AWSManagedRulesKnownBadInputsRuleSet`, `AWSManagedRulesAmazonIpReputationList`, and a rate-based rule per source address (ASM-129, PD-102) | "Consulta pública limitada" (criterion 1); peaks of 30 reads per minute; guessing of codes (A09 section 8.1). | Permanent | Audit |
| CL-PUB-02 | PUB | Internet | Public lookup front door | TCP 80 | Redirect to HTTPS | Viewer protocol policy of the front door | HTTP carries no data (A09 section 7.1). | Permanent | Review |
| CL-PUB-03 | PUB | CloudFront origin-facing addresses | Load balancer in `public_query`, only if PD-051 chooses one | TCP 443 | Allow; any other source Deny | Security group with the AWS managed prefix list `com.amazonaws.global.cloudfront.origin-facing` | The origin cannot be reached around the WAF. | Permanent if a load balancer is chosen | Audit |
| CL-PUB-04 | PUB | 0.0.0.0/0 | `public_query` subnets | Inbound TCP 443; outbound TCP 1024 to 65535 | Allow; rest Deny | Network ACL of `public_query` | The only internet-facing tier (A07 section 4). | Permanent | Review |
| CL-DAT-01 | DAT | 10.40.0.0/13 | `data` subnets | Any | Deny | Network ACL of `data`, lowest rule number | No center ever reaches a store. | Permanent | Audit |
| CL-DAT-02 | DAT | `application` 10.82.48.0/20 | `data` subnets | The port of the store A13 selects | Allow; rest Deny | Network ACL and security group of the store | Only the application tier reads and writes data. | Permanent | Review |
| CL-DAT-03 | DAT | Private tiers | Amazon S3 and Amazon DynamoDB | HTTPS | Allow only resources of the organization | Gateway endpoint policies with `aws:ResourceOrgID` | AWS services through VPC endpoints (PC-IAC-020). | Permanent | Review |
| CL-EGR-01 | EGR | Private tiers | Internet | Any | Deny | No NAT gateway and no default route to the internet in private route tables; security group egress limited to the destinations above (ASM-127) | No rule in this table needs internet egress from a private tier. | Permanent | Audit |
| CL-DEF-01 | DEF | Any | Any | Any | Deny | Implicit deny of security groups; final `*` deny of every network ACL; no transit gateway route; WAF default Block | Explicit default deny of the cloud. | Permanent | Audit |
| CL-DEF-02 | DEF | Rejected flows | Log | Any | Log | VPC Flow Logs of the `pdn` VPC, rejected traffic, to CloudWatch Logs with the `logs` key (ASM-127) | Denied attempts are observable (point 6). | Permanent | Review |

### 5.3 AWS Network Firewall is not proposed

AWS Network Firewall would add stateful inspection inside the VPC, but it is charged per firewall endpoint per hour in every Availability Zone where it runs, plus per gigabyte processed, and an endpoint would be needed in each of the three Availability Zones of `pdn` (ASM-031) for the whole pilot. Against the low-investment constraint ("El caficultor invierte poco") it buys little here: center traffic is already inspected by the center firewall before it enters the tunnel, the transit gateway route tables admit only four tiers, every destination of a center flow is an interface endpoint guarded by a security group and an endpoint policy, internet entry is only through IoT Core with mutual TLS and front doors with AWS WAF, and private tiers have no internet egress to inspect (CL-EGR-01). It is therefore not deployed (ASM-127). A23 can price it if a later requirement, such as domain filtering of private egress, appears (PD-124).

## 6. Rules to audit before production, and temporary rules

Every rule marked Audit is checked before `pdn` go-live (PD-125). They are the rules that open a path across a trust boundary or that are the only control on it:

| Boundary | Rules |
|---|---|
| Visitors and transporters away from logistics and management | CC-GST-01, CC-GST-02, CC-GST-03, CC-ADM-03 |
| Internet to the center and the tunnel | CC-LNK-01, CC-LNK-02, CC-LNK-03, CC-LNK-07, CC-LNK-08, CL-LNK-06 |
| Hub route filtering | CL-LNK-01, CL-LNK-02, CL-LNK-03 |
| Device management and log export | CC-BAS-02, CC-ADM-01, CC-ADM-02, CL-ADM-01, CL-ADM-02, CL-ADM-03, CL-ADM-05 |
| Drone objects | CC-DRN-01 to CC-DRN-05, CL-DRN-01, CL-DRN-02, CL-DRN-03 |
| Application path | CC-LOG-01, CC-LOG-03, CC-STF-01, CL-APP-01, CL-APP-02 |
| Internet entry to the platform | CL-TEL-01, CL-TEL-03, CL-PUB-01, CL-PUB-03 |
| Data, egress and default | CL-DAT-01, CL-EGR-01, CC-DEF-01, CL-DEF-01 |

The temporary rules are few and each has its removal condition. Epic 6 codes only the cloud rules marked Permanent (PD-122); a temporary cloud rule is added and removed through the pipeline as a production change with its own release note.

| Rule | Opens | Removed when |
|---|---|---|
| CC-LNK-06 and CL-LNK-05 | Logística to `qa` `application` | The center's acceptance tests are signed off, at the latest before `pdn` go-live. |
| CC-ADM-05 | Device firmware downloads | The approved maintenance window ends; it is reopened per window. |
| CC-ADM-06 | Commissioning laptop at 10.4x.10.60 | Management through the tunnel works and the site is accepted; the switch port is disabled. |

## 7. Traceability to the A11 intent matrix

| A11 cell (source to destination) | Rules | Narrowing |
|---|---|---|
| Gerencia, Soporte, Logística to Servidores locales: printing only | CC-LOG-02, CC-STF-02 | IPP on TCP 631 only (ASM-124). |
| Gerencia, Soporte, Logística to cloud `application` | CC-LOG-01, CC-STF-01, CL-LNK-01, CL-APP-01 to 03 | TCP 443 to `pdn` only. |
| Ground station to `drone_object_ingress` and `object_storage` | CC-DRN-01 to 03, CL-DRN-01 to 04 | None. |
| Gerencia, Soporte to internet, staff class | CC-STF-04, CC-STF-05, CC-PUB-01 | Web ports only; the ground station only to IoT Core on TCP 8883 (ASM-125, ASM-126). |
| Logística to internet: named time and update services | CC-BAS-01, CC-LOG-03, CC-LOG-04 | Time comes from the gateway, not the internet. |
| Transporte to internet, with the public API | CC-TEL-02, CC-PUB-01, CC-GST-04, CC-GST-05 | SMTP denied. |
| Visitantes to internet | CC-PUB-01, CC-GST-04, CC-GST-05 | SMTP denied. |
| Servidores locales: firewall to switches and access points | CC-BAS-02 to 04 | None. |
| Servidores locales to `administration`: logs, metrics, configuration backup and time | CC-ADM-02, CC-ADM-01 | Logs and metrics only are device-initiated; backups are pulled by the cloud session, and time is served locally (ASM-123). |
| Servidores locales to internet: named update services, temporary | CC-ADM-05 | None. |
| Cloud `administration` to Servidores locales: device management | CC-ADM-01, CL-ADM-03 | TCP 22 and 443 to the management addresses only. |
| Every Deny cell | CC-GST-01 to 03, CC-LNK-07 to 09, CC-ADM-03, CC-ADM-04, CC-ADM-07, CC-DRN-05, CC-DRN-06, CC-LOG-04, CC-STF-03, CC-TEL-01, CC-DEF-01 | Explicit and logged. |

## 8. Compliance with the case statement and the issue

| Requirement | Where it is met |
|---|---|
| "Firewall de perímetro en cada centro y firewall de nube. Tabla de políticas: origen, destino, puerto, acción y si la regla queda en el piloto o es temporal." Issue criterion: source, destination, port and protocol, justification, permanent or temporary. | Section 4 (center) and section 5.2 (cloud), columns of section 2. |
| "La VLAN de visitantes no enruta hacia logística ni gerencia." Issue criterion: guests blocked from logistics and management. | CC-GST-01 to 03 and CC-ADM-03, ahead of every allow; the routing context of A11 section 4.3 is the first layer. |
| Issue criterion: rules differentiated for telemetry, drone objects, administration, public lookup and cloud-to-center links. | Groups TEL, DRN, ADM, PUB and LNK (section 3) in both tables. |
| Issue criterion: rules to audit or review before production. | Column "Before production" and section 6. |
| Epic 6: "reglas de firewall marcadas como permanentes". | Validity column; PD-122. |
| Explicit default deny at the end of each table. | CC-DEF-01 and CL-DEF-01. |

## 9. Decision-log entries

| ID | Summary |
|---|---|
| PD-011 | Each rule is classified as permanent or temporary; approval is the lead's merge of this table. Closed here. |
| PD-052 | The tables use the A07 ranges. Closed here. |
| PD-083 | Center tunnels, backup link, drone path and logistics path are classified. Closed here. |
| PD-091 | Port-level rules derived from the A11 matrix and classified. Closed here. |
| PD-093 | Route 53 Resolver inbound endpoint in `administration`. Closed here. |
| ASM-120 | Hub is an AWS Transit Gateway in the network account with one static, IKEv2, certificate-authenticated VPN connection per center; prefix lists shared through AWS RAM. |
| ASM-121 | Centers reach only four `pdn` tiers; `qa` only temporarily for acceptance; `dev` never. |
| ASM-122 | Route 53 Resolver inbound endpoint and exact-host-name conditional forwarding. |
| ASM-123 | Firewall serves time locally; device logs and metrics exported over HTTPS with IAM Roles Anywhere; backups and revocation list moved by the cloud session. |
| ASM-124 | Printing on IPP, TCP 631. |
| ASM-125 | Internet service sets per zone. |
| ASM-126 | Ground station grant channel to the public IoT Core endpoint through the breakout. |
| ASM-127 | No NAT gateway, no AWS Network Firewall; VPC Flow Logs of rejected traffic. |
| ASM-128 | Commissioning laptop address and temporary-rule lifecycle. |
| ASM-129 | AWS WAF configuration of the public front doors. |
| PD-120 | Re-check the AWS features named here against live documentation. |
| PD-121 | A15 names the interface endpoints of `drone_object_ingress`. |
| PD-122 | Epic 6 codes the permanent cloud rules. |
| PD-123 | The management session host and the log export path are fixed. |
| PD-124 | A23 prices the cloud network controls. |
| PD-125 | Pre-production audit of the rules marked Audit. |

## 10. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Point 3 (policy table, visitor VLAN), point 2 (TLS), point 4 (harvest freeze), Epic 6 (permanent rules), criterion 1 (limited public lookup). |
| [`docs/collection-center-network.md`](collection-center-network.md) | Zones, addresses, routing context, matrix of section 6, traffic classes. |
| [`docs/connectivity.md`](connectivity.md) | Tunnels, backup link, drone and transporter paths, IoT Core ports. |
| [`docs/addressing-plan.md`](addressing-plan.md) | Tier blocks, 10.40.0.0/13, `transit`. |
| [`docs/security-and-identity.md`](security-and-identity.md) | TLS policies, upload grants, buyer view, Identity Center access. |
| [`docs/landing-zone.md`](landing-zone.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | Network account, domain separation, PC-IAC-020. |
| [Transit gateway route tables](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html), [Site-to-Site VPN tunnel options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNTunnels.html) | Static and blackhole routes, IKEv2, certificate authentication. |
| [Managed prefix lists](https://docs.aws.amazon.com/vpc/latest/userguide/managed-prefix-lists.html), [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html) | Prefix lists in security groups and routes; stateless ACLs. |
| [Route 53 Resolver inbound endpoints](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-inbound-queries.html) | Name resolution from the centers. |
| [AWS WAF managed rule groups](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-list.html) | Rule groups of CL-PUB-01 and CL-TEL-03. |
| [AWS Network Firewall pricing](https://aws.amazon.com/network-firewall/pricing/) | Pricing dimensions of section 5.3; no figure is quoted. |
