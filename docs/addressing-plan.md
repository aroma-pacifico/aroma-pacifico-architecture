# Cloud Addressing Plan

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR1 / JR1 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#7 (A07) |
| Depends on | [ADR-001](adr/adr-001-primary-and-second-cloud.md) |
| Feeds | aroma-pacifico/aroma-pacifico-roadmap#10 (A10), aroma-pacifico/aroma-pacifico-roadmap#11 (A11), aroma-pacifico/aroma-pacifico-roadmap#12 (firewall policy table), aroma-pacifico/aroma-pacifico-roadmap#24 (B04, network module) |
| Decision-log entries | PD-009 (closed here), ASM-030 to ASM-036, PD-050 to PD-052 |

## Diagrams

- [Cloud and Site Addressing Plan (SVG)](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/network/addressing-plan.svg) ([PNG](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/network/addressing-plan.png)): Details the top-level RFC 1918 partition across on-premises, AWS primary cloud, and undeployed Azure space, including VPC subnet tiering and scaling from 200 to 1,000 farms without address redesign.

## 1. Purpose and constraints

This plan fixes the private IPv4 addressing of the pilot's cloud networks. Point 3 of the [case statement](case/case-statement.pdf) asks for "Plan de direccionamiento de la nube, sin solapar 10.41.0.0/16, 10.42.0.0/16 ni 10.43.0.0/16. Subredes mínimas: ingreso de telemetría, ingreso de objetos de dron, aplicación, datos, object storage, administración y consulta pública. Desarrollo, pruebas y producción usan CIDR distintos." The technical constraints add that the network and ingestion pattern scales from 200 to 1,000 farms without redesigning the addressing or the ingestion point ("El patrón de red y de ingesta escala de 200 a 1.000 fincas sin rediseñar el direccionamiento ni el punto de ingesta").

[ADR-001](adr/adr-001-primary-and-second-cloud.md) fixes the frame: AWS is the primary cloud in the single Region US East (N. Virginia), `us-east-1`; one AWS Organization holds separate accounts per environment, so each environment owns its own VPC; Azure is a prepared, undeployed export target; and one further block is reserved, not used, for any future Azure network so that preparing Azure never forces a renumbering.

The environments are development, test and production. Their short codes are `dev`, `qa` and `pdn`, the values that the binding IaC rules already require for `var.environment` (`standards/iac-rules/pc-iac-002.md` section 3 and `pc-iac-003.md` section 3.2). This plan uses those codes and introduces no others.

## 2. Reserved collection-center space

The case statement reserves, for each collection center, the /24 networks of its inventory table and repeats the table in the three centers by changing the second octet: 41 Cauca, 42 Huila, 43 Tolima ("Estas direcciones quedan reservadas y la nube no puede solaparse con ellas ... Se repite el mismo esquema en los tres centros, cambiando el segundo octeto"). The table below is the Cauca example as the case statement gives it; Huila and Tolima are the same table with 42 and 43 in the second octet. No zone is added here; the LAN design of the centers belongs to its own task.

| Zone (case statement) | Use | Planned clients | Reserved network (Cauca) |
|---|---|---|---|
| Gerencia (management) | Desktops and phones | 2 desktops, 5 phones | 10.41.1.0/24 |
| Soporte (support) | Desktops and phones | 2 desktops, 5 phones | 10.41.2.0/24 |
| Logística (logistics) | Sack reception and printing | 2 desktops, 20 phones | 10.41.3.0/24 |
| Transporte (transport) | Driver phones in the yard | 20 phones | 10.41.4.0/24 |
| Visitantes (visitors) | Guests | 10 phones | 10.41.5.0/24 |
| Servidores locales (local servers) | Printing and network control | To be designed | 10.41.10.0/24 |
| Enlace a la nube (cloud link) | Edge | Center firewall | 10.41.20.0/24 |

The cloud plan keeps the whole of 10.41.0.0/16, 10.42.0.0/16 and 10.43.0.0/16 out of the cloud, not only the seven /24 networks per center. The case statement says the table is the space for wiring the site and that no equipment exists yet ("Todavía no hay equipos: la tabla es el espacio al cablear el sitio"), so the centers will grow inside their /16; any cloud subnet placed in an unused /24 of 10.41.0.0/16 would block that growth and would also break a single summary route per center. Each center is therefore one /16 route on the cloud side.

A future fourth department follows the same second-octet rule and receives 10.44.0.0/16 (ASM-030). To leave room for it and for further departments, the cloud plan keeps the whole of 10.40.0.0/13, that is 10.40.0.0 to 10.47.255.255, out of the cloud. This lets the cloud side reach every present and future center with one summary route, 10.40.0.0/13, so adding a department adds a site, not a route redesign.

## 3. Top-level allocation

The cloud uses its own /13 above the collection-center space, with one /16 per environment so that the second octet names the environment in the same way the case statement uses it to name the department. A /16 is also the largest primary CIDR block AWS accepts for a VPC, so the environment never has to add secondary blocks to grow.

| Block | First address | Last address | Purpose | Status |
|---|---|---|---|---|
| 10.40.0.0/13 | 10.40.0.0 | 10.47.255.255 | Collection centers: 10.41 Cauca, 10.42 Huila, 10.43 Tolima (case statement), 10.44 future fourth department (ASM-030), the rest kept for center growth. | Never used by the cloud |
| 10.80.0.0/13 | 10.80.0.0 | 10.87.255.255 | AWS supernet of the pilot, `us-east-1`. | Partly allocated |
| 10.80.0.0/16 | 10.80.0.0 | 10.80.255.255 | VPC of development (`dev`), development account. | Allocated |
| 10.81.0.0/16 | 10.81.0.0 | 10.81.255.255 | VPC of test (`qa`), test account. | Allocated |
| 10.82.0.0/16 | 10.82.0.0 | 10.82.255.255 | VPC of production (`pdn`), production account. | Allocated |
| 10.83.0.0/16 | 10.83.0.0 | 10.83.255.255 | AWS reserve; first candidate for any VPC the network or security account later needs (ASM-034). | Reserved |
| 10.84.0.0/14 | 10.84.0.0 | 10.87.255.255 | AWS reserve for further environments or accounts (ASM-034). | Reserved |
| 10.88.0.0/13 | 10.88.0.0 | 10.95.255.255 | Future Azure network, if the prepared export target ever needs one; mirrors the AWS layout as 10.88.0.0/16 `dev`, 10.89.0.0/16 `qa`, 10.90.0.0/16 `pdn` (ASM-035). | Reserved, not deployed |

The three environment blocks are distinct and do not overlap each other: each is a different /16 with a different second octet. The Azure reservation is a separate /13 that touches neither the AWS supernet nor the collection centers, so a later Azure network, a VPN to it, or an export path through private addressing can be added without renumbering any AWS or collection-center network. ADR-001 designs no cross-cloud link; this block only keeps the option open.

## 4. Subnet layout inside each environment VPC

Every environment VPC uses the same layout, so the network module of aroma-pacifico/aroma-pacifico-roadmap#24 is one pattern with three variable files. Each tier receives a /20 block, divided into four /22 Availability Zone slots. Each environment spans three Availability Zones, `az1`, `az2` and `az3`, and the fourth slot, `az4`, is reserved (ASM-031). The Availability Zone IDs behind `az1` to `az3` are chosen by the network module, because AWS maps zone names to physical zones per account.

A /22 holds 1,024 addresses; AWS reserves the first four and the last address of every subnet, so 1,019 are usable. Beyond the seven minimum tiers of the case statement, the plan adds one small `transit` tier for the attachment of the environment VPC to the network account's hub, which carries traffic to and from the collection centers (ASM-033). The case statement asks for minimum subnets, so an extra tier is allowed; its /28 slots hold 16 addresses each, 11 usable.

| Tier key | Case statement subnet | Tier block (E = environment octet) | Routing | Index |
|---|---|---|---|---|
| `public_query` | consulta pública | 10.E.0.0/20 | Public: the only tier routed to the internet gateway | 0 |
| `telemetry_ingress` | ingreso de telemetría | 10.E.16.0/20 | Private | 1 |
| `drone_object_ingress` | ingreso de objetos de dron | 10.E.32.0/20 | Private | 2 |
| `application` | aplicación | 10.E.48.0/20 | Private | 3 |
| `data` | datos | 10.E.64.0/20 | Private, no route to the internet | 4 |
| `object_storage` | object storage | 10.E.80.0/20 | Private | 5 |
| `administration` | administración | 10.E.96.0/20 | Private | 6 |
| (reserved tiers) | none | 10.E.112.0 to 10.E.239.255 | Not created | 7 to 14 |
| `transit` | none (added by ASM-033) | 10.E.240.0/20, used as /28 slots | Private, routes 10.40.0.0/13 to the hub | 15 |

The eight reserved tier blocks, 10.E.112.0 to 10.E.239.255, hold 32,768 addresses per environment (128 values of the third octet times 256), so a new tier can be added later at a free index without moving any existing subnet. The remainder of the transit block, 10.E.240.64 to 10.E.255.255, holds a further 4,032 addresses (16 times 256, minus the 64 used by the four /28 slots).

### 4.1 Subnet CIDRs per environment

| Tier key | AZ slot | `dev` (10.80.0.0/16) | `qa` (10.81.0.0/16) | `pdn` (10.82.0.0/16) |
|---|---|---|---|---|
| `public_query` | `az1` | 10.80.0.0/22 | 10.81.0.0/22 | 10.82.0.0/22 |
| `public_query` | `az2` | 10.80.4.0/22 | 10.81.4.0/22 | 10.82.4.0/22 |
| `public_query` | `az3` | 10.80.8.0/22 | 10.81.8.0/22 | 10.82.8.0/22 |
| `public_query` | `az4` (reserved) | 10.80.12.0/22 | 10.81.12.0/22 | 10.82.12.0/22 |
| `telemetry_ingress` | `az1` | 10.80.16.0/22 | 10.81.16.0/22 | 10.82.16.0/22 |
| `telemetry_ingress` | `az2` | 10.80.20.0/22 | 10.81.20.0/22 | 10.82.20.0/22 |
| `telemetry_ingress` | `az3` | 10.80.24.0/22 | 10.81.24.0/22 | 10.82.24.0/22 |
| `telemetry_ingress` | `az4` (reserved) | 10.80.28.0/22 | 10.81.28.0/22 | 10.82.28.0/22 |
| `drone_object_ingress` | `az1` | 10.80.32.0/22 | 10.81.32.0/22 | 10.82.32.0/22 |
| `drone_object_ingress` | `az2` | 10.80.36.0/22 | 10.81.36.0/22 | 10.82.36.0/22 |
| `drone_object_ingress` | `az3` | 10.80.40.0/22 | 10.81.40.0/22 | 10.82.40.0/22 |
| `drone_object_ingress` | `az4` (reserved) | 10.80.44.0/22 | 10.81.44.0/22 | 10.82.44.0/22 |
| `application` | `az1` | 10.80.48.0/22 | 10.81.48.0/22 | 10.82.48.0/22 |
| `application` | `az2` | 10.80.52.0/22 | 10.81.52.0/22 | 10.82.52.0/22 |
| `application` | `az3` | 10.80.56.0/22 | 10.81.56.0/22 | 10.82.56.0/22 |
| `application` | `az4` (reserved) | 10.80.60.0/22 | 10.81.60.0/22 | 10.82.60.0/22 |
| `data` | `az1` | 10.80.64.0/22 | 10.81.64.0/22 | 10.82.64.0/22 |
| `data` | `az2` | 10.80.68.0/22 | 10.81.68.0/22 | 10.82.68.0/22 |
| `data` | `az3` | 10.80.72.0/22 | 10.81.72.0/22 | 10.82.72.0/22 |
| `data` | `az4` (reserved) | 10.80.76.0/22 | 10.81.76.0/22 | 10.82.76.0/22 |
| `object_storage` | `az1` | 10.80.80.0/22 | 10.81.80.0/22 | 10.82.80.0/22 |
| `object_storage` | `az2` | 10.80.84.0/22 | 10.81.84.0/22 | 10.82.84.0/22 |
| `object_storage` | `az3` | 10.80.88.0/22 | 10.81.88.0/22 | 10.82.88.0/22 |
| `object_storage` | `az4` (reserved) | 10.80.92.0/22 | 10.81.92.0/22 | 10.82.92.0/22 |
| `administration` | `az1` | 10.80.96.0/22 | 10.81.96.0/22 | 10.82.96.0/22 |
| `administration` | `az2` | 10.80.100.0/22 | 10.81.100.0/22 | 10.82.100.0/22 |
| `administration` | `az3` | 10.80.104.0/22 | 10.81.104.0/22 | 10.82.104.0/22 |
| `administration` | `az4` (reserved) | 10.80.108.0/22 | 10.81.108.0/22 | 10.82.108.0/22 |
| `transit` | `az1` | 10.80.240.0/28 | 10.81.240.0/28 | 10.82.240.0/28 |
| `transit` | `az2` | 10.80.240.16/28 | 10.81.240.16/28 | 10.82.240.16/28 |
| `transit` | `az3` | 10.80.240.32/28 | 10.81.240.32/28 | 10.82.240.32/28 |
| `transit` | `az4` (reserved) | 10.80.240.48/28 | 10.81.240.48/28 | 10.82.240.48/28 |

The rule behind the table is mechanical, so a reviewer can rebuild any cell: the third octet of a /22 is `16 x tier index + 4 x (AZ slot number - 1)`, and the second octet is 80, 81 or 82 for `dev`, `qa` or `pdn`. For example, `pdn` `data` `az3` is tier index 4 and slot 3, so the third octet is 16 x 4 + 4 x 2 = 72, giving 10.82.72.0/22, as in the table.

### 4.2 Managed services that need no subnet

Several tiers of the case statement map to AWS managed services that live outside any VPC. The plan still reserves each tier, but it states what the subnet holds instead of inventing a subnet for the service itself. The service choices named here are the ones ADR-001 already makes (Lambda, Amazon S3) or the assumed ingestion endpoint of ASM-032; the ingestion, drone and lookup tasks may change them, and the reservation holds either way.

| Tier key | Managed service of the tier | Needs a subnet for the service? | What the tier subnet hosts |
|---|---|---|---|
| `telemetry_ingress` | Regional managed MQTT over TLS endpoint, assumed AWS IoT Core (ASM-032). Farm nodes publish over cellular to the public regional endpoint. | No. The endpoint is regional and public; devices hold an identity, not a cloud IP. | The interface VPC endpoint of the ingestion data plane, so the collection centers can publish over their WAN link without the internet, and the network interfaces of any VPC-attached ingestion function. Pilot decision: no interface endpoint is created and the tier stays empty and denied ([firewall policy](firewall-policy.md#52-policy-table), CL-TEL-04). |
| `drone_object_ingress` | Flight objects land in Amazon S3 through authorized HTTPS uploads. | No for uploads over a public cellular link. | The interface VPC endpoints used when flight logs and objects are uploaded from a collection center where the drone is prepositioned, and the network interfaces of the VPC-attached reception functions. Pilot decision: the ground station uses the endpoints in `application` and `object_storage`, the reception function is not VPC-attached, and the tier holds no endpoint ([drone service](drone-service.md#9-ingress-endpoints-of-the-ground-station-pd-121) section 9, ASM-140, PD-140). |
| `object_storage` | Amazon S3 holds photos, maps, polygons and flight logs (Part E). | No. Buckets are regional. The S3 gateway endpoint attaches to route tables and consumes no addresses. | The S3 interface VPC endpoint, which, unlike the gateway endpoint, can be reached from the collection centers through the hub. |
| `public_query` | Public sack lookup. Whether its front door is an internet-facing load balancer or a managed regional or edge service is decided by the sack lookup design, at the latest in the complete diagram of aroma-pacifico/aroma-pacifico-roadmap#19 (PD-051). | Only if a load balancer is chosen. | The internet-facing load balancer if chosen, and the NAT gateways of the private tiers, because this is the only public tier. Pilot decision: the front door is Amazon CloudFront with AWS WAF, no load balancer and no NAT gateway, so the tier is created and stays empty ([sack and trip](sack-and-trip.md#72-front-door-cloudfront-no-load-balancer-closes-pd-051) section 7.2, ASM-127, PD-051 closed). |
| `application` | Lambda functions and application services attached to the VPC. | Yes. | Network interfaces of the application functions and services. |
| `data` | Transactional and time-series stores (Part E). | Yes. A managed database needs a subnet group across Availability Zones. | Database instances and their subnet group. No route to the internet. |
| `administration` | Session-based administration without bastion hosts, logs, keys and parameters. | Yes, for the interface endpoints. | Interface VPC endpoints for administration, logging, key and parameter services, and any build or plan runner the pipeline places in the VPC. |
| `transit` | Hub in the network account that terminates the collection-center WAN links (ASM-033). | Yes, for the attachment. | The attachment network interfaces of the hub, one per active Availability Zone. |

### 4.3 Names reusable as Terraform variables

The tier keys and Availability Zone keys above are `snake_case` map keys, as `standards/iac-rules/pc-iac-002.md` section 4.1 requires for `for_each` collections (`map(object)` with stable keys). The network module of aroma-pacifico/aroma-pacifico-roadmap#24 receives them as values, never as defaults (`standards/terraform-iac-best-practices.md`, no environment-specific defaults). The block below illustrates the production values only; it is documentation, not a `.tf` file.

```hcl
environment    = "pdn"
vpc_cidr_block = "10.82.0.0/16"

subnets = {
  public_query         = { az1 = "10.82.0.0/22",    az2 = "10.82.4.0/22",     az3 = "10.82.8.0/22" }
  telemetry_ingress    = { az1 = "10.82.16.0/22",   az2 = "10.82.20.0/22",    az3 = "10.82.24.0/22" }
  drone_object_ingress = { az1 = "10.82.32.0/22",   az2 = "10.82.36.0/22",    az3 = "10.82.40.0/22" }
  application          = { az1 = "10.82.48.0/22",   az2 = "10.82.52.0/22",    az3 = "10.82.56.0/22" }
  data                 = { az1 = "10.82.64.0/22",   az2 = "10.82.68.0/22",    az3 = "10.82.72.0/22" }
  object_storage       = { az1 = "10.82.80.0/22",   az2 = "10.82.84.0/22",    az3 = "10.82.88.0/22" }
  administration       = { az1 = "10.82.96.0/22",   az2 = "10.82.100.0/22",   az3 = "10.82.104.0/22" }
  transit              = { az1 = "10.82.240.0/28",  az2 = "10.82.240.16/28",  az3 = "10.82.240.32/28" }
}

collection_center_cidr_blocks = {
  cauca  = "10.41.0.0/16"
  huila  = "10.42.0.0/16"
  tolima = "10.43.0.0/16"
}
```

Physical names follow `standards/iac-rules/pc-iac-003.md`, `{client}-{project}-{environment}-{type}-{key}`, with hyphens only. For subnets the module builds `{key}` from a short tier code and the slot number, so the physical name stays within the 28-character limit: `pubq`, `tlmi`, `droi`, `app`, `data`, `objs`, `adm` and `tgw`, giving for example `{client}-{project}-pdn-snet-tlmi1`. The values of `client` and `project` are set by the IaC repository, not by this plan.

## 5. Scaling from 200 to 1,000 farms

Farms do not receive cloud IP addresses. A farm node publishes over cellular data, or through store-and-forward when it has no signal, to one regional ingestion endpoint per environment, and the endpoint authenticates the device by its identity, not by its address (ASM-032). Growing from 200 to 1,000 farms therefore adds 800 device identities to the same endpoint and adds no subnet, no route, no endpoint and no CIDR. The transporter phones and the drone uploads over a public link behave the same way.

The case statement gives the ingestion load at 200 farms: cultivation 200 farms x 6 readings per day = 1,200 readings per day, and active drying in harvest about 100 farms x 24 readings per day = 2,400 readings per day, a total of 3,600 readings per day. At 1,000 farms, keeping the same readings per farm and the same half of the farms in drying (ASM-036), cultivation is 1,000 x 6 = 6,000 and drying is 500 x 24 = 12,000, a total of 18,000 readings per day, five times the pilot load, or 18,000 / 86,400 = about 0.21 readings per second on average. That load reaches the same endpoint and does not touch the address plan; the capacity of the endpoint is a matter for the ingestion design, not for the addressing.

The subnet that would grow if any address consumption tracked the farms is `telemetry_ingress`. Its normal consumption is a constant: one address per interface endpoint per Availability Zone and the shared network interfaces of the VPC-attached functions, independent of the number of farms. To show headroom without depending on that, the table below uses a deliberately pessimistic bound in which every farm consumed one private address in the tier.

| Quantity | 200 farms | 1,000 farms |
|---|---|---|
| Usable addresses per `telemetry_ingress` /22 slot | 1,024 - 5 = 1,019 | 1,019 |
| Usable addresses in the three active slots | 3 x 1,019 = 3,057 | 3,057 |
| Pessimistic consumption, one address per farm | 200 | 1,000 |
| Share of the three active slots used | 200 / 3,057 = 6.5 % | 1,000 / 3,057 = 32.7 % |
| Free addresses in the three active slots | 3,057 - 200 = 2,857 | 3,057 - 1,000 = 2,057 |
| Worst Availability Zone if spread evenly | 67 of 1,019 (200 / 3, rounded up) | 334 of 1,019 (1,000 / 3, rounded up) |
| Everything in one Availability Zone | 200 of 1,019 | 1,000 of 1,019, still fits |
| Further reserve without readdressing | `az4` slot: 1,019; reserved tier blocks: 32,768 per environment | Same |

Even under that bound, 1,000 farms fit in a single `telemetry_ingress` slot, and the tier has about three times the needed addresses across its active slots. The collection-center side does not change either: the three centers keep their /16, a fourth department receives 10.44.0.0/16 inside 10.40.0.0/13, and the cloud already routes the whole /13 through the `transit` tier. Neither the ingestion point nor the addressing is redesigned.

## 6. Verification

Two ranges do not overlap when the last address of one is lower than the first address of the other. Every cloud block starts at 10.80.0.0 or higher, and every reserved collection-center range ends at 10.43.255.255 or lower, so each row below passes by comparing the second octet alone. Every subnet in section 4.1 lies inside its environment /16, so the environment rows also cover every subnet.

| Cloud block | First | Last | Reserved range | First | Last | Test | Result |
|---|---|---|---|---|---|---|---|
| 10.80.0.0/16 (`dev`) | 10.80.0.0 | 10.80.255.255 | 10.41.0.0/16 | 10.41.0.0 | 10.41.255.255 | 10.41.255.255 < 10.80.0.0 | No overlap |
| 10.80.0.0/16 (`dev`) | 10.80.0.0 | 10.80.255.255 | 10.42.0.0/16 | 10.42.0.0 | 10.42.255.255 | 10.42.255.255 < 10.80.0.0 | No overlap |
| 10.80.0.0/16 (`dev`) | 10.80.0.0 | 10.80.255.255 | 10.43.0.0/16 | 10.43.0.0 | 10.43.255.255 | 10.43.255.255 < 10.80.0.0 | No overlap |
| 10.81.0.0/16 (`qa`) | 10.81.0.0 | 10.81.255.255 | 10.41.0.0/16 | 10.41.0.0 | 10.41.255.255 | 10.41.255.255 < 10.81.0.0 | No overlap |
| 10.81.0.0/16 (`qa`) | 10.81.0.0 | 10.81.255.255 | 10.42.0.0/16 | 10.42.0.0 | 10.42.255.255 | 10.42.255.255 < 10.81.0.0 | No overlap |
| 10.81.0.0/16 (`qa`) | 10.81.0.0 | 10.81.255.255 | 10.43.0.0/16 | 10.43.0.0 | 10.43.255.255 | 10.43.255.255 < 10.81.0.0 | No overlap |
| 10.82.0.0/16 (`pdn`) | 10.82.0.0 | 10.82.255.255 | 10.41.0.0/16 | 10.41.0.0 | 10.41.255.255 | 10.41.255.255 < 10.82.0.0 | No overlap |
| 10.82.0.0/16 (`pdn`) | 10.82.0.0 | 10.82.255.255 | 10.42.0.0/16 | 10.42.0.0 | 10.42.255.255 | 10.42.255.255 < 10.82.0.0 | No overlap |
| 10.82.0.0/16 (`pdn`) | 10.82.0.0 | 10.82.255.255 | 10.43.0.0/16 | 10.43.0.0 | 10.43.255.255 | 10.43.255.255 < 10.82.0.0 | No overlap |
| 10.83.0.0/16 (AWS reserve) | 10.83.0.0 | 10.83.255.255 | 10.41.0.0/16 | 10.41.0.0 | 10.41.255.255 | 10.41.255.255 < 10.83.0.0 | No overlap |
| 10.83.0.0/16 (AWS reserve) | 10.83.0.0 | 10.83.255.255 | 10.42.0.0/16 | 10.42.0.0 | 10.42.255.255 | 10.42.255.255 < 10.83.0.0 | No overlap |
| 10.83.0.0/16 (AWS reserve) | 10.83.0.0 | 10.83.255.255 | 10.43.0.0/16 | 10.43.0.0 | 10.43.255.255 | 10.43.255.255 < 10.83.0.0 | No overlap |
| 10.84.0.0/14 (AWS reserve) | 10.84.0.0 | 10.87.255.255 | 10.41.0.0/16 | 10.41.0.0 | 10.41.255.255 | 10.41.255.255 < 10.84.0.0 | No overlap |
| 10.84.0.0/14 (AWS reserve) | 10.84.0.0 | 10.87.255.255 | 10.42.0.0/16 | 10.42.0.0 | 10.42.255.255 | 10.42.255.255 < 10.84.0.0 | No overlap |
| 10.84.0.0/14 (AWS reserve) | 10.84.0.0 | 10.87.255.255 | 10.43.0.0/16 | 10.43.0.0 | 10.43.255.255 | 10.43.255.255 < 10.84.0.0 | No overlap |
| 10.88.0.0/13 (Azure reserve) | 10.88.0.0 | 10.95.255.255 | 10.41.0.0/16 | 10.41.0.0 | 10.41.255.255 | 10.41.255.255 < 10.88.0.0 | No overlap |
| 10.88.0.0/13 (Azure reserve) | 10.88.0.0 | 10.95.255.255 | 10.42.0.0/16 | 10.42.0.0 | 10.42.255.255 | 10.42.255.255 < 10.88.0.0 | No overlap |
| 10.88.0.0/13 (Azure reserve) | 10.88.0.0 | 10.95.255.255 | 10.43.0.0/16 | 10.43.0.0 | 10.43.255.255 | 10.43.255.255 < 10.88.0.0 | No overlap |

The wider reservations pass by the same test: the lowest cloud address, 10.80.0.0, is above 10.47.255.255, the last address of 10.40.0.0/13, so no cloud block overlaps the future fourth department at 10.44.0.0/16 or any other center space.

The cloud blocks also do not overlap one another, which is what "Desarrollo, pruebas y producción usan CIDR distintos" requires.

| Pair | Last address of the lower block | First address of the higher block | Result |
|---|---|---|---|
| `dev` 10.80.0.0/16 and `qa` 10.81.0.0/16 | 10.80.255.255 | 10.81.0.0 | No overlap |
| `qa` 10.81.0.0/16 and `pdn` 10.82.0.0/16 | 10.81.255.255 | 10.82.0.0 | No overlap |
| `pdn` 10.82.0.0/16 and AWS reserve 10.83.0.0/16 | 10.82.255.255 | 10.83.0.0 | No overlap |
| AWS reserve 10.83.0.0/16 and AWS reserve 10.84.0.0/14 | 10.83.255.255 | 10.84.0.0 | No overlap |
| AWS supernet 10.80.0.0/13 and Azure reserve 10.88.0.0/13 | 10.87.255.255 | 10.88.0.0 | No overlap |

The prefixes are valid network boundaries: a /13 must start at a second octet divisible by 8 (40, 80 and 88 are), a /14 at one divisible by 4 (84 is), a /22 at a third octet divisible by 4 (every value in section 4.1 is), and a /28 at a fourth octet divisible by 16 (0, 16, 32 and 48 are). The arithmetic was checked twice: by hand with the bounds above, and with the Python `ipaddress` module, whose output is attached to the pull request of aroma-pacifico/aroma-pacifico-roadmap#7.

## 7. Decision-log entries

| ID | Summary |
|---|---|
| PD-009 | The cloud CIDRs for `dev`, `qa` and `pdn` are chosen without overlapping the collection centers. Closed by this plan. |
| ASM-030 | A fourth department follows the case statement's second-octet rule as 10.44.0.0/16; 10.40.0.0/13 stays out of the cloud. |
| ASM-031 | Each environment spans three Availability Zones; a fourth slot per tier is reserved. |
| ASM-032 | Field devices, transporter phones and public drone uploads reach regional managed endpoints over TLS and hold no cloud private address; the telemetry endpoint is assumed to be AWS IoT Core. |
| ASM-033 | A `transit` tier attaches each environment VPC to a hub in the network account that terminates the collection-center WAN links. |
| ASM-034 | 10.83.0.0/16 and 10.84.0.0/14 are held as AWS reserve inside 10.80.0.0/13. |
| ASM-035 | 10.88.0.0/13 is reserved for a future Azure network and is not deployed. |
| ASM-036 | At 1,000 farms the readings per farm and the share of farms in drying stay as in the case statement. |
| PD-050 | The network module codes these CIDRs and keys as variable values. |
| PD-051 | The sack lookup design decides whether `public_query` hosts a load balancer. |
| PD-052 | The firewall policy table uses these ranges as sources and destinations. |

## 8. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Point 3 (collection-center inventory, cloud addressing plan), technical constraints (200 to 1,000 farms), Part A ingestion figures. |
| [`docs/decision-log.md`](decision-log.md) | PD-009, ASM-030 to ASM-036, PD-050 to PD-052. |
| `docs/adr/adr-001-primary-and-second-cloud.md` (aroma-pacifico/aroma-pacifico-architecture#8) | AWS primary in `us-east-1`, one account and one VPC per environment, Azure reservation (PD-018 of that pull request). |
| `standards/iac-rules/pc-iac-002.md`, `pc-iac-003.md` | Environment codes `dev`, `qa`, `pdn`; `snake_case` keys; physical naming pattern. |
| [Amazon VPC CIDR blocks](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html) | VPC block sizes from /16 to /28. |
| [Amazon VPC subnet CIDR blocks](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html) | Five reserved addresses per subnet. |
| [Gateway endpoints for Amazon S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html) | Gateway endpoints are not reachable from on-premises networks; interface endpoints are. |
