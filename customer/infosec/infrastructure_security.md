# Infrastructure Security

This document describes Goji Labs-managed hosting. Customers who host in their own cloud account have their infrastructure governed by that account's configuration, which we agree with them during the project.

## Architecture Overview
- **Backend services** (APIs, workers, databases, caches, and queues) run in a Kubernetes cluster managed by DigitalOcean (DOKS).
- **Frontend services** run on Cloudflare Workers, in a Cloudflare account dedicated to each customer.
- **DNS, TLS, and edge security** are provided by Cloudflare.
- **Files and database backups** are stored in Cloudflare R2.

## Tenant Isolation
Each customer has a dedicated Kubernetes namespace for each environment, such as `staging-customer` and `production-customer`. The Kubernetes cluster is shared, but all of a customer's backend services, databases, caches, and queues run inside that customer's own namespaces. Production and staging run in separate namespaces, and do not share databases or other data stores. Kubernetes network policies restrict traffic between namespaces, so one customer's workloads cannot connect to another customer's workloads.

Each customer also has a dedicated Cloudflare account, which holds their DNS, frontend services, and object storage.

Where a customer requires stronger isolation, we can provide a dedicated Kubernetes cluster or a dedicated managed database instance, at additional hosting cost.

## Network Security
- **No public origin.** Every public-facing web service is exposed through a Cloudflare Tunnel. The tunnel connects outward from the cluster to Cloudflare. Our web services therefore have no public IP address, no open inbound ports, and no public load balancer. The only way to reach them is through Cloudflare over TLS.
- **DDoS protection.** All traffic passes through Cloudflare's network, which provides DDoS mitigation.
- **Web Application Firewall.** Cloudflare's WAF is configured for projects that need it, based on the project's risk profile and the customer's requirements.
- **Data stores are private.** On our current platform, databases, caches, and queues are reachable only from inside the customer's namespace, and are never exposed to the internet.

## Encryption
- **In transit:** TLS 1.2 or higher is required between Cloudflare and our origin services. At the edge, our default minimum is TLS 1.2. If a customer explicitly requires support for legacy clients, we can lower the edge minimum to TLS 1.1 or 1.0 for that customer, at their request.
- **At rest:** databases, Kubernetes volumes, and Cloudflare R2 object storage are all encrypted at rest.

## Data Stores

### Current platform (projects started on or after January 1, 2026)
PostgreSQL, Redis, and Memcached run inside each customer's namespace, and are managed by open-source Kubernetes operators. They are not shared with other customers, and are not accessible from outside the namespace. Database backups are copied to a private R2 bucket in the customer's Cloudflare account. See [Business Continuity](./business_continuity.md).

### Legacy platform (some projects started before January 1, 2026)
Some older projects use shared, DigitalOcean-managed PostgreSQL and Redis clusters, with a separate database for each project. Applications connect to them over a private VPC network. Their public endpoints are protected by an IP allowlist, which is empty except when an administrator is temporarily added to perform a long-running operation, such as a large migration. The administrator's address is removed as soon as the operation is complete.

### Administrative access to data
Routine database maintenance is performed from a purpose-built maintenance container running inside the cluster, accessed through `kubectl`. This avoids exposing databases outside the cluster. See [Access Management](./access_management.md) for who holds this access.

## Container Security
- Applications are built on distroless base images, which contain no shell, package manager, or other tools beyond what the application needs to run. This minimizes both the attack surface and the number of components that need patching.
- Containers contain no SSH or remote desktop services.
- Images are built by our CI pipeline and stored in a private DigitalOcean Container Registry. See [Secure Software Development](./secure_sdlc.md).

## Secrets Management
Application secrets, such as database credentials and API keys, are stored as Kubernetes Secrets in the customer's namespace, and are injected into containers at runtime. Secrets are never committed to source code. GitHub secret scanning detects accidental commits.

## Logging and Monitoring
We operate a self-hosted Grafana stack for observability:

- **Prometheus** for metrics
- **Loki** for logs
- **OpenTelemetry** for traces and application performance monitoring (APM)

Infrastructure and hardware alerting is in place. **Planned (Q1 2027):** application-specific alerting.

Our providers also keep audit logs of administrative activity, including GitHub's organization audit log and the activity logs for Google Workspace, DigitalOcean, and Cloudflare.

Application logs are retained for at least 14 days. Longer retention is available on request.

## Infrastructure as Code
Environments are defined in Terraform. Our Terraform code is stored in private GitHub repositories, and its state is stored in a dedicated PostgreSQL database. Because environments are defined in code, they are consistent and reproducible, which also supports disaster recovery.
