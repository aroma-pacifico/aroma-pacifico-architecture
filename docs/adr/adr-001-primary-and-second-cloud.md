# ADR-001: Primary cloud and prepared second cloud

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Owner | SR1 / SR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#2 |
| Supersedes | none |
| Superseded by | none |

## Context

The case statement leaves the choice of the primary cloud to the presenter, who must justify it, and asks for a second cloud that is prepared without duplicating the platform ("Elige la nube principal y deja preparada una segunda, sin duplicar la plataforma"). The owner took the decision on 2026-11-04: AWS is the primary cloud and Azure is the prepared second cloud. This record justifies it, fixes the AWS Region, and defines in testable terms what "prepared" means.

Four forces drive the decision. The first is the workload shape of Part E: inference is event-driven, the function ends when the result is persisted, there is no persistent compute waiting for objects, and the design must state the region of the managed inference service and which objects leave the country. The second is data: personal data of farmers and transporters, readings, photos, flights and sack history are retained for 5 years, and sack events have an RPO of 15 minutes while the public query has an RTO of 4 hours. The third is the delivery boundary: Epic 6 is the only coded epic and must fit 100 hours and 2 weeks, so the second cloud cannot add a second platform to code. The fourth is the network: development, test and production use distinct CIDRs that never overlap 10.41.0.0/16, 10.42.0.0/16 or 10.43.0.0/16.

The case statement does not require that data stay in Colombia, does not give a latency figure other than the alert targets, and does not define "prepared". Those gaps are filled by the assumptions [ASM-002](../decision-log.md#assumptions) to [ASM-005](../decision-log.md#assumptions). The region of the inference service and the list of objects that leave the country remain open in PD-013 for aroma-pacifico/aroma-pacifico-roadmap#17; this ADR only fixes the Region that the answer must use.

### Region availability, checked on 2026-11-04

The table below records what current AWS documentation states. It was checked on 2026-11-04 through the official pages listed in [References](#references); direct downloads of `docs.aws.amazon.com` are blocked by the network policy of the agent session that wrote this record, so the check was made through indexed copies of those pages and must be repeated by aroma-pacifico/aroma-pacifico-roadmap#17 before the inference service is fixed (PD-018). No price was consulted or used.

| Location | Status in AWS documentation | Consequence for the pilot |
|---|---|---|
| Colombia | No AWS Region exists. | No Region choice keeps data inside Colombia; data residency is handled by ASM-002. |
| Bogotá Local Zone (`us-east-1-bog-1a`, parent US East (N. Virginia)) | Announced, not generally available, on the AWS Local Zones locations page. | Not used. Even when available, a Local Zone hosts a subset of services and its parent Region still hosts the managed services. |
| South America (São Paulo), `sa-east-1` | Lambda and Amazon S3 available. Amazon Bedrock runtime available, but the Claude vision models are listed with global cross-Region inference only, no in-Region or geographic inference. Amazon Rekognition has an endpoint; the General Reference notes that some Regions support only some operations, and the default DetectLabels quota is lower than in US East (N. Virginia). | A Bedrock vision call would leave the Region to an unspecified Region, so the design could not state which country an object goes to. |
| US East (N. Virginia), `us-east-1` | Lambda and Amazon S3 available. Amazon Bedrock runtime available with in-Region inference for vision-capable models, for example Amazon Nova Pro. Amazon Rekognition available with the highest default quotas. Parent Region of the announced Bogotá Local Zone. | Storage, functions and managed inference can sit in one Region, so the objects that leave Colombia go to one named country and one named Region. |

## Decision

AWS is the primary cloud, and every environment (development, test and production) runs in the single Region US East (N. Virginia), `us-east-1`. Azure is the prepared second cloud: it receives no deployment during the pilot, and "prepared" means only the three properties defined below.

### What "prepared without duplicating the platform" means

The second cloud is prepared for portability and exit, not for failover (ASM-004). The recovery targets of the case statement, an RPO of 15 minutes for sack events and an RTO of 4 hours for the public query, are met inside `us-east-1` across Availability Zones, as designed by the later Part C tasks.

| Property | What is delivered | Where it lives | Fits the 100 hours because |
|---|---|---|---|
| Provider-neutral module interfaces | The network, security and ingestion modules expose inputs and outputs named after the case statement, not after AWS: `environment`, `cidr_block`, the seven subnet roles (telemetry ingress, drone object ingress, application, data, object storage, administration, public query), `tags`, and the identifiers of the stores that hold durable objects and sack history. AWS-specific identifiers stay inside the module. | Module `variables.tf` and `outputs.tf` in the IaC repository (ADR-003). | It is a naming rule on interfaces that are written anyway; no second implementation is written. |
| Open-format durable data | Durable objects (photos, maps, polygons, flight logs) already live in object storage, as Part E requires. Sack history is also exported as append-only, open-format files (for example JSON Lines) to a dedicated export prefix in the same storage, so the full 5-year record can leave AWS without a database-specific tool. | Architecture document and data design tasks; the storage resource is part of the primary platform. | The export prefix is one more prefix in a bucket the pilot already needs. |
| Declared, not deployed, export target | An Azure Blob Storage container with an immutability policy is the named destination for the durable objects and the sack history export. It is declared as a documented contract (container naming, prefix layout, encryption, retention of 5 years) and as a disabled, typed input in the IaC interface. Copying from Amazon S3 to Azure Blob is a documented AzCopy capability, so no custom tool is designed. | This ADR, the architecture document and the IaC interface. | No `azurerm` provider is added to any root stack, so no Azure plan, credential or state is needed. |

The following are explicitly not duplicated: no Azure subscription, landing zone or management group is created; no Azure virtual network, Azure Functions, Azure inference service, database or identity directory is designed or coded; no running replication or scheduled copy job exists; no second CI/CD pipeline, second Terraform state backend or second set of platform identities exists; no active-active or DNS failover between clouds exists; and no second AWS Region is used. The 200 farms, the three collection centers and the field devices connect only to AWS.

### Latency to Cauca, Huila and Tolima

The only latency figures in the case statement are alert targets: under 5 minutes from an out-of-range reading when a delivery path exists, and within 24 hours when the buffer drains. Sack lookup peaks at 30 reads per minute. Neither figure is sensitive to the difference between `us-east-1` and `sa-east-1`, because a network round trip to either Region is expected to take well under one second, far below 5 minutes; that is an expectation, not a measurement. Field links in the pilot are cellular and store-and-forward, so the slowest leg is the field link, not the Region. No round trip was measured; ASM-003 records the assumption and the alert-delivery SLO of point 6 is the future test that validates it.

### Account and environment model

The case statement requires separate accounts for security, network, production and test, with policies, identities, tags and budgets applied from the root ("cuentas, suscripciones o proyectos separados para seguridad, red, producción y pruebas"). On AWS this maps to one AWS Organization with separate accounts for security, network, production and test, and a separate development account (ASM-005). Each environment owns its own VPC in its own account, so each one can receive a distinct CIDR block with no shared address space between environments.

The constraint passed to aroma-pacifico/aroma-pacifico-roadmap#7 is the following: development, test and production each receive a distinct, non-overlapping CIDR block in `us-east-1`; no block overlaps 10.41.0.0/16, 10.42.0.0/16 or 10.43.0.0/16; each block holds the seven subnet roles of point 3 and leaves room to grow from 200 to 1,000 farms without readdressing; and one further block is reserved, not used, for any future Azure network, so that preparing Azure never forces a renumbering (PD-018). The values are chosen in PD-009, not here.

### How the decision avoids depending on 2020 prices and services

The case statement says that 2020 prices, devices and cloud services are not a valid quote ("Precios, dispositivos y servicios de nube de 2020 no son cotización válida"). This decision uses no price at all: it rests on service availability in current AWS documentation dated in 2026, checked on 2026-11-04 and to be checked again before the inference service is fixed (PD-018). The case statement does not name the provider or services of the 2020 design, and this ADR inherits none of them; the 2020 requirements that remain valid and the components that are retired are recorded in the [2020 legacy table](../legacy-2020.md), written by aroma-pacifico/aroma-pacifico-roadmap#5. Costs are priced at current prices for `us-east-1` by aroma-pacifico/aroma-pacifico-roadmap#36, under PD-017.

## Alternatives considered

| Alternative | Advantages | Disadvantages | Reason rejected |
|---|---|---|---|
| AWS primary, Azure prepared, Region `us-east-1` (chosen) | Functions, object storage and in-Region managed inference in one Region; the repository IaC rules are written for AWS; the Bogotá Local Zone, when available, has `us-east-1` as parent. | Data leaves Colombia to the United States; latency is not measured (ASM-003). | Chosen. |
| AWS primary, Region `sa-east-1` | A Region in South America. | The Claude vision models on Bedrock are global cross-Region only, so objects would go to an unspecified Region; Rekognition operation coverage and default quotas are lower. Data still leaves Colombia. | It cannot answer "qué objetos salen del país" with a single named destination. |
| AWS primary, GCP prepared | Same primary benefits; Cloud Storage also offers retention locks and a transfer service from S3. | For an undeployed export target the difference with Azure is small; the case statement names Azure Functions and Cloud Functions equally. | The owner chose Azure on 2026-11-04; GCP gives no advantage that justifies reversing that decision. |
| Azure primary, AWS prepared | Azure Functions and managed vision services exist. | `standards/iac-rules/` is written for AWS: PC-IAC-008 allows only the S3 backend, PC-IAC-017 uses SSM Parameter Store, and PC-IAC-003 names AWS resources. Changing the primary would require rewriting binding standards inside the 100 hours. | Cost in standards rework and risk to the 100-hour budget. |
| GCP primary | Cloud Functions and managed vision services exist. | Same standards rework as Azure primary, plus no team rule written for GCP. | Same reason as Azure primary. |
| Full second platform (active-active or warm standby on Azure) | Cloud-level failover. | Duplicates the platform, which the case statement forbids, and doubles the IaC, identities, pipeline and cost. | Contradicts "sin duplicar la plataforma" and the 100-hour limit. |

## Consequences

The platform becomes simpler to code and to defend: one provider, one Region, one Terraform provider family, one state backend and one set of identities, aligned with the binding IaC rules. What becomes harder is that every object and every personal record leaves Colombia for the United States, which must be stated in the defense and validated in the clarification session (ASM-002, PD-005), and that leaving AWS later depends on keeping the interfaces neutral and the durable data in open formats, which reviewers must enforce.

| Dimension | Impact |
|---|---|
| Network | One VPC per environment account in `us-east-1`, each with a distinct CIDR outside 10.41.0.0/16 to 10.43.0.0/16, plus one reserved block for a future Azure network (PD-009, PD-018). The collection centers connect their WAN edge (10.4x.20.0/24) to the `us-east-1` network account only. No cross-cloud link, VPN or peering to Azure is designed. |
| Security | Identities, MFA, encryption keys and audit live in AWS only; no Azure identity exists. Encryption at rest and TLS in transit apply to the export prefix like any other data. The future export target must keep encryption, the 5-year retention and the buyer-view restrictions; no personal data leaves AWS during the pilot. International transfer of personal data to the United States is an assumption to validate (ASM-002). |
| IaC | Terraform with the AWS provider only, in `us-east-1`, following `standards/iac-rules/` without changes. Module interfaces use provider-neutral names; the Azure export target is a disabled, typed input and a documented contract. No `azurerm` provider, Azure state or Azure credential enters any root stack, so the 100-hour plan contains no Azure task. |
| Costs | No Azure cost during the pilot, because nothing is deployed there. The export prefix adds only the storage of the sack history files in the primary bucket. A future export would add AWS data transfer out and Azure storage, to be priced at current prices if it is ever activated. All pilot costs are priced for `us-east-1` at current prices by aroma-pacifico/aroma-pacifico-roadmap#36; this ADR quotes no price. |

## Compliance with the case statement

| Case statement passage | How this decision complies |
|---|---|
| "La nube principal la elige quien presenta y la justifica." (The presenter chooses and justifies the primary cloud.) | AWS is chosen and justified in [Context](#context), [Decision](#decision) and [Alternatives considered](#alternatives-considered). |
| "Elige la nube principal y deja preparada una segunda, sin duplicar la plataforma." (Choose the primary cloud and leave a second one prepared, without duplicating the platform.) | Azure is prepared through three defined properties, and the list of what is not duplicated is explicit ([What "prepared" means](#what-prepared-without-duplicating-the-platform-means)). |
| "Desarrollo, pruebas y producción usan CIDR distintos." (Development, test and production use distinct CIDRs.) | One account and one VPC per environment allow distinct blocks; the constraint is passed to aroma-pacifico/aroma-pacifico-roadmap#7 ([Account and environment model](#account-and-environment-model)). |
| "Precios, dispositivos y servicios de nube de 2020 no son cotización válida." (2020 prices, devices and cloud services are not a valid quote.) | No price is used and the availability check is dated 2026-11-04 ([How the decision avoids depending on 2020 prices and services](#how-the-decision-avoids-depending-on-2020-prices-and-services)). |
| "Según la nube elegida: Lambda, Azure Functions o Cloud Functions. No hay cómputo persistente en espera de objetos." (Depending on the chosen cloud: Lambda, Azure Functions or Cloud Functions. No persistent compute waits for objects.) | The inference functions are AWS Lambda in `us-east-1`. |
| "El diseño indica la región del servicio, qué objetos salen del país." (The design states the service Region and which objects leave the country.) | The managed inference service runs in `us-east-1`, in-Region, so objects leave Colombia only to that Region; the object list is closed in PD-013. |
| "Datos personales de caficultores y transportadores. Retención: 5 años." (Personal data of farmers and transporters. Retention: 5 years.) | Personal data stay in `us-east-1` for 5 years (ASM-002); the declared Azure target keeps the same retention. |
| "La implementación de infraestructura como código cabe en 2 semanas y en 100 horas." (The IaC implementation fits in 2 weeks and 100 hours.) | No Azure code is written; the only cost of preparation is interface naming and one disabled input. |

## References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](../case/case-statement.pdf) | Only source of requirements. |
| [`docs/decision-log.md`](../decision-log.md) | ADR-001 index row; ASM-002 to ASM-005; PD-001, PD-005, PD-009, PD-013, PD-017, PD-018. |
| [`docs/legacy-2020.md`](../legacy-2020.md) | 2020 requirements kept and components retired (aroma-pacifico/aroma-pacifico-roadmap#5). |
| `standards/iac-rules/pc-iac-003.md`, `pc-iac-008.md`, `pc-iac-017.md` | Binding IaC rules written for AWS. |
| aroma-pacifico/aroma-pacifico-roadmap#2, #5, #7, #17, #36 | This task, the legacy table, the addressing plan, the inference functions and the cost document. |
| [AWS Regions](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions.html) | Region list; no Region in Colombia. |
| [AWS Local Zones locations](https://aws.amazon.com/about-aws/global-infrastructure/localzones/locations/) | Bogotá Local Zone `us-east-1-bog-1a`, announced, parent `us-east-1`. |
| [Amazon Bedrock regional availability by endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/endpoints-region-availability.html) and [by models](https://docs.aws.amazon.com/bedrock/latest/userguide/models-region-compatibility.html) | Bedrock runtime in `sa-east-1` and `us-east-1`; per-model inference type. |
| [Claude Sonnet 4.6 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html) and [Amazon Nova Pro model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-amazon-nova-pro.html) | Global-only inference in `sa-east-1`; in-Region image input in `us-east-1`. |
| [Amazon Rekognition endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/rekognition.html) | Endpoints, per-Region operation notes and default quotas. |
| [Copy data from Amazon S3 to Azure Storage with AzCopy](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-s3) | Basis for the declared, not deployed, export path. |
