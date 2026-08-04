# ScaleOps

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

ScaleOps is an autonomous Kubernetes cost optimization and resource management platform that continuously manages cloud infrastructure resources in real-time based on actual workload behavior. ScaleOps eliminates the need for manual resource configuration by automatically right-sizing CPU, memory, and replica counts for containers and clusters. The platform integrates natively with AWS, GCP, and Azure cost management tools and is deployed via a single Helm command into Kubernetes clusters.

**URL:** [Visit APIs.json URL](https://raw.githubusercontent.com/api-evangelist/scaleops/refs/heads/main/apis.yml)

## Scope

- **Type:** Index
- **Position:** Consuming
- **Access:** 3rd-Party

## Tags

AWS, Azure, Cost Optimization, FinOps, GCP, Helm, Kubernetes, Resource Management

## Timestamps

- **Created:** 2026-05-02
- **Modified:** 2026-05-02

## APIs

### ScaleOps Platform API
The ScaleOps Platform API provides programmatic access to Kubernetes cost optimization features including workload resource recommendations, real-time optimization controls, cost monitoring, and cluster configuration.

**Human URL:** [https://scaleops.com/](https://scaleops.com/)

#### Tags
Automation, Cost Monitoring, FinOps, Kubernetes, Resource Optimization, Workloads

#### Properties
- [Documentation](https://docs.scaleops.com/)
- [Getting Started](https://docs.scaleops.com/)
- [Helm Chart](https://catalog.redhat.com/en/software/container-stacks/detail/66eaefc1fa4fb0aa5f835b90)

## Schemas

- [json-schema/scaleops-workload-schema.json](json-schema/scaleops-workload-schema.json) — Kubernetes workload optimization schema

## JSON Structures

- [json-structure/scaleops-workload-structure.json](json-structure/scaleops-workload-structure.json) — Workload structure documentation

## JSON-LD

- [json-ld/scaleops-context.jsonld](json-ld/scaleops-context.jsonld) — JSON-LD context mapping ScaleOps vocabulary to schema.org and Kubernetes vocabulary

## Vocabulary

- [vocabulary/scaleops-vocabulary.yml](vocabulary/scaleops-vocabulary.yml) — ScaleOps terminology and domain vocabulary (11 terms)

## Common Properties

- [Website](https://scaleops.com/)
- [Documentation](https://docs.scaleops.com/)
- [GitHub](https://github.com/scaleops-sh)
- [Blog](https://scaleops.com/blog/)
- [Cost Monitoring](https://scaleops.com/product/kubernetes-cost-monitoring/)

## Maintainers

**FN:** Kin Lane

**Email:** kin@apievangelist.com
