# Business Continuity

## System Availability

### Applications
Autoscaling is configured both for each Kubernetes deployment and for the cluster's node pool, so that the system scales up and down with traffic. Deployments scale according to configurable CPU and memory thresholds, which default to 75% and are tuned to each application's needs. The node pool adds or removes DigitalOcean servers as needed. We typically run at least three replicas of each production service, which tolerates the failure of an individual node and allows deployments with no downtime.

Frontend services run on Cloudflare Workers, which serves requests from Cloudflare's global network.

### Databases
**Current platform:** PostgreSQL runs in the customer's namespace, and is managed by an open-source Kubernetes operator. By default, a standby replica is kept in sync with the primary. If the primary fails, the operator automatically promotes the standby to become the new primary, without any change to the application.

**Legacy platform:** some projects that began before January 1, 2026 use DigitalOcean-managed PostgreSQL. On these clusters, a standby node is kept in sync with the primary. If DigitalOcean detects a failure, traffic is automatically redirected to the standby, which becomes the new primary, and a replacement standby is created. This happens without intervention from us, and without any change to the application.

## Backups
- **Current platform:** databases are backed up daily, and backups are retained for two weeks. Point-in-time recovery is available, so a database can be restored to any moment within the last two weeks, not only to the time of the most recent daily backup. Backups are stored in a private Cloudflare R2 bucket in the customer's dedicated Cloudflare account. Because R2 is a separate provider from DigitalOcean, where the database runs, a DigitalOcean failure does not affect the backups.
- **Legacy platform:** DigitalOcean takes a daily backup of each managed database, and retains it for seven days.
- **On request:** we can deliver backups to additional destinations, such as storage in a customer's own cloud account, and we can accommodate longer retention periods. We can also keep a second copy of user-uploaded files with another provider. By default, uploaded files are stored only in Cloudflare R2, which stores data redundantly.
- **Restore testing:** we test restoring from backup at least once a year.

## Recovery Objectives
Our standard recovery objectives are listed below.

- The **Recovery Time Objective (RTO)** is the maximum time we target for restoring service.
- The **Recovery Point Objective (RPO)** is the maximum period of data loss we target, measured back from the time of the incident.

| Scenario | RTO | RPO |
| --- | --- | --- |
| Failure of a node, pod, or database primary | 15 minutes | Near zero |
| Data corruption or accidental deletion | 8 hours | 15 minutes |
| Loss of a DigitalOcean region | 48 hours | 15 minutes |
| Broad Cloudflare outage | Dependent on Cloudflare's restoration of service | No data loss |

These objectives apply to the current platform. On the legacy platform, restoring from a backup may lose up to 24 hours of data, because backups are taken daily.

## Disaster Scenarios

### Loss of a node or service instance
Kubernetes automatically reschedules affected workloads onto healthy nodes, and the node pool replaces failed servers. No intervention is required.

### Data corruption or accidental deletion
On the current platform, we use point-in-time recovery to restore the affected database to the moment just before the problem occurred, provided this was within the last two weeks. On the legacy platform, we restore from the most recent daily backup taken before the problem occurred. See [Backups](#backups).

### DigitalOcean outage
Our infrastructure is defined in Terraform, and our applications run in containers on Kubernetes, which is an open standard. If a DigitalOcean region becomes unavailable for a long period, we can recreate the environment in another region and restore data from the backups held in Cloudflare R2. Moving to another Kubernetes provider, such as AWS, Google Cloud, or Azure, is also possible, but would require adapting parts of our Terraform configuration and would take longer. In either case, container images would be rebuilt from source code in GitHub.

### Cloudflare outage
Cloudflare provides DNS, the tunnels to our backend services, frontend hosting, and object storage. Cloudflare's network is globally distributed, and localized problems are usually routed around automatically. During a broad Cloudflare outage, applications would be unreachable until service is restored.

Objects stored in Cloudflare R2, including database backups and files uploaded by users, would be temporarily unavailable during such an outage, but would not be lost. R2 stores data redundantly. Databases run in DigitalOcean and would be unaffected.

### GitHub outage
Running applications are unaffected by a GitHub outage, but deployments would be paused until it is resolved. Every developer has a complete copy of the Git history of the projects they work on, and recent container images remain in our container registry, so no source code would be lost.

### Security incident
Security incidents are handled under our [Incident Response](./incident_response.md) procedure, which uses these recovery capabilities where needed.
