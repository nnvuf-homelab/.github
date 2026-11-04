# nnvuf-homelab

This homelab runs **MARINA**, a private document library and retrieval system.
Its services collect documents under operator control, index them, and return
source passages through a browser UI or read-only MCP tools. The
[marina-docs repository](https://github.com/nnvuf-homelab/marina-docs) is the
canonical guide; check its operations runbooks for current deployment status.

## How the repositories fit together

Application code, cluster configuration, deployment admission, and documentation
have separate homes:

```mermaid
flowchart LR
    Docs["marina-docs<br/>Architecture, guides, runbooks"]
    Infra["Infrastructure<br/>Cluster, Flux bootstrap, shared services"]
    Shipyard["Shipyard<br/>Build and image promotion"]
    Fleet["Fleet<br/>Application admission and Helm releases"]
    Apps["Application repositories<br/>Lighthouse · Chartroom · Trawler<br/>Harbormaster"]

    Apps -->|"Repository CI builds images"| Shipyard
    Shipyard -->|"Promotion PRs pin image digests"| Fleet
    Infra -->|"Flux reconciles admitted releases"| Fleet
    Docs -. "Documents the system" .-> Infra
    Docs -.-> Fleet
    Docs -.-> Apps
```

Infrastructure owns cluster services and runtime configuration. Fleet defines
which applications Flux may deploy; Shipyard promotes application images.
Application repositories own source code, while `marina-docs` owns the
cross-project documentation. See [Fleet's ownership and release guide](https://github.com/nnvuf-homelab/marina-docs/blob/main/fleet/README.md).

## MARINA at a glance

```mermaid
flowchart LR
    Operator["Operator"] --> UI["Chartroom<br/>Document library"]
    Operator --> Fetch["Trawler<br/>Manual public-URL acquisition"]
    UI -->|"Authenticated /v1 API"| API["Lighthouse<br/>Collections, documents, search, questions"]
    Fetch -->|"Existing document upload API"| API
    API --> DB[("PostgreSQL<br/>Metadata and jobs")]
    API --> Originals[("Private originals storage")]
    API --> Vectors[("Qdrant<br/>Vectors")]
    Worker["Lighthouse worker<br/>Ingestion and deletion"] --> DB
    Worker --> Originals
    Worker --> Vectors
    API --> Models["Ollama<br/>Embedding and generation models"]
    Worker --> Models
    MCP["Harbormaster<br/>Read-only MCP gateway"] -->|"List, search, get source"| API
    RAG["Agentic RAG role<br/>Question route"] --> DB
    RAG --> Vectors
    RAG --> Models
```

Chartroom manages collections, uploads, indexing status, and source-backed
questions. Lighthouse is the API and indexing backend. Trawler is an optional,
operator-controlled acquisition path; it does not crawl in the background.
Harbormaster provides a read-only MCP interface. Lighthouse also defines an
`agentic-rag` role; the `knowledge-learning` role is a scaffold, not an active
learning service. See the [product guides](https://github.com/nnvuf-homelab/marina-docs#products).

### Document and answer flows

```mermaid
flowchart TD
    Upload["Upload in Chartroom"] --> Ingest["Lighthouse queues document"]
    Acquire["Operator requests one URL from Trawler"] --> Normalize["Validate, fetch and normalize public text"]
    Normalize --> Handoff["Operator hands off through Lighthouse upload API"]
    Handoff --> Ingest
    Ingest --> Worker["Lighthouse worker"]
    Worker --> Store["Store original and extracted chunks"]
    Store --> Index["Create embeddings and update Qdrant"]
    Store --> Metadata[("PostgreSQL metadata and job status")]
    Index --> Ready["Document becomes ready"]
    Ready --> Ask["Question or search via Chartroom"]
    Ask --> Retrieve["Lighthouse retrieves matching source passages"]
    Retrieve --> Answer["Return cited passages to the user"]
```

Only ready documents participate in answers. Answers are source-first: inspect
the cited passages and their context rather than treating a quotation as a
guarantee of truth or completeness. Trawler preserves provenance and reuses
Lighthouse's existing upload/indexing path; see the [Trawler guide](https://github.com/nnvuf-homelab/marina-docs/blob/main/trawler/README.md).

## Access boundaries

```mermaid
flowchart LR
    Browser["User browser"] -->|"HTTPS"| Edge["Public ingress"]
    Edge -->|"Forward-auth check"| Authentik["Authentik"]
    Authentik -->|"Allow or challenge"| Edge
    Edge -->|"Authorized request"| Chartroom["Chartroom"]
    Chartroom -->|"Same-origin API proxy<br/>session cookie and Origin"| API["Lighthouse API"]
    API -->|"Validates session with trusted outpost"| Authentik

    Client["MCP client"] -->|"Per-user delegated identity"| MCP["Harbormaster"]
    MCP -->|"Allow-listed read-only API calls"| API
    MCP -. "No direct access" .-> Stores["Database · vectors · originals"]
    Trawler["Private operator-run Trawler"] -->|"No public-origin credentials"| Internet["Public HTTPS sources"]
```

The browser session is Authentik-managed; Lighthouse independently validates
it rather than trusting identity headers. Harbormaster is limited to listing
collections, searching, and retrieving exact source chunks. Trawler's service
should remain private; only the one-shot handoff uses an operator's Lighthouse
session. See the [Chartroom guide](https://github.com/nnvuf-homelab/marina-docs/blob/main/chartroom/README.md),
[Harbormaster guide](https://github.com/nnvuf-homelab/marina-docs/blob/main/harbormaster/README.md),
and [MCP OAuth runbook](https://github.com/nnvuf-homelab/marina-docs/blob/main/Infrastructure/docs/runbooks/marina-mcp-oauth.md).

## Cluster lifecycle and operations

```mermaid
flowchart TD
    Source["Application source change"] --> CI["Build and publish image"]
    CI --> Promote["Shipyard promotion PR<br/>immutable image digest"]
    Promote --> Admit["Fleet release and admission allow-list"]
    Bootstrap["Infrastructure Flux bootstrap"] --> Reconcile["Flux reconciles Fleet"]
    Admit --> Reconcile
    Reconcile --> Init["Lighthouse pre-install / pre-upgrade initialization"]
    Init --> Workloads["Helm-managed application workloads"]
    Init -->|"Failure blocks rollout"| Hold["Release remains unrolled"]
```

```mermaid
flowchart LR
    Fleet["Fleet and Shipyard<br/>Release and image lifecycle"]
    Infra["Infrastructure<br/>Ingress, secrets, storage, monitoring"]
    Lighthouse["Lighthouse<br/>Metrics, metadata, originals and restore semantics"]
    Trawler["Trawler<br/>Acquisition journal and its recovery"]
    Docs["marina-docs<br/>Runbooks and acceptance records"]

    Infra -->|"Prometheus discovery, alerts, dashboards"| Lighthouse
    Infra -->|"Persistent volume and private workload"| Trawler
    Fleet -->|"Images, releases, initialization"| Lighthouse
    Fleet -->|"Images and releases"| Trawler
    Docs -. "Operational procedures" .-> Infra
    Docs -.-> Fleet
    Docs -.-> Lighthouse
    Docs -.-> Trawler
```

Runtime credentials, ingress, persistent storage, and monitoring are provisioned
by Infrastructure and consumed through Fleet releases. Backups and recovery are
service-specific; a Lighthouse archive does not include Trawler's acquisition
journal. Production release gates and operational acceptance can change, so
consult [MARINA operations](https://github.com/nnvuf-homelab/marina-docs/blob/main/Infrastructure/docs/runbooks/marina-operations.md)
and the [Lighthouse production runbook](https://github.com/nnvuf-homelab/marina-docs/blob/main/lighthouse/PRODUCTION-RUNBOOK.md)
before making operational decisions.

## Further reading

- [Canonical documentation index](https://github.com/nnvuf-homelab/marina-docs)
- [Infrastructure and operations](https://github.com/nnvuf-homelab/marina-docs/tree/main/Infrastructure/docs)
- [Lighthouse architecture and development](https://github.com/nnvuf-homelab/marina-docs/blob/main/lighthouse/README.md)
- [Fleet deployment ownership](https://github.com/nnvuf-homelab/marina-docs/blob/main/fleet/README.md)
