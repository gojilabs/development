# Devops Process

## Tools

### Docker
Every project has distinct containers and a separate `Dockerfile` for each service. For example, in a full-stack web application, the backend service will have a `Dockerfile`, and the frontend service will have its own `Dockerfile`.

### Github
We commit all code to our (private Github repositories. We leverage Github Actions to build our docker containers and to push these built containers to our container registry.

### DigitalOcean
DigitalOcean is a cloud server provider. They're analagous to AWS, with many API-compatible services, but at a lower cost and with a lot less complexity. We use the following managed services from them:
- [Spaces](https://docs.digitalocean.com/products/spaces/): An S3-compatible object-storage system. Every project has a private space, used to store historical Terraform plans and database backups. Most projects also use a space for assets, containing user uploaded content, and sits behind Cloudflare.
- [Kubernetes](https://docs.digitalocean.com/products/kubernetes/): A managed kubernetes service which we use to orchestrate all containers. We host projects in a cost-effective way by having production clusters with multiple projects running in them, and one staging cluster where staging environments across all projects run
- [Databases](https://docs.digitalocean.com/products/databases/postgresql/): Managed PostgreSQL, where each project gets its own database slice of a shared production database. The staging database is shared for all projects.

### Kubernetes
Kubernetes allows us to orchestrate the various services that make up our projects. It allows us to define:
1. Deployments
  - Which process needs to run
  - What port to expose to the `Ingress`
  - CPU / Memory requests and limits
  - How many copies (called `Pods`) of a deployment to run (scaling)
  - Bindings to environment variables, called `ConfigMaps` and `Secrets`
  - Rollout strategy on deploy
2. Ingress
  - Manages incoming network traffic from the internet
  - Connected to and manages the DigitalOcean Load Balancer. Any changes we want made to the DOLB must be made in the `Ingress` for them to persist.
  - TLS termination via Cloudflare Strict SSL, we present Cloudflare's Origin TLS certificate
  - Routing traffic meant for a hostname to a given `Pod` based on hosts and ports 
3. Services
  - A `Deployment` with internal-only networking, these are not connected to the `Ingress`.
  - We use this for pubsub Redis (for WebSocket/ActionCable and for Sidekiq workers) and caching Redis
4. Cron Jobs

#### Links
(Deployment)[https://kubernetes.io/docs/concepts/workloads/controllers/deployment/]
(Ingress)[https://kubernetes.io/docs/concepts/services-networking/ingress/]

### Rollbar
Error tracking, backtraces, and error metrics

### Terraform
Terraform helps us manage the growing number of projects that we host and maintain. TF allows us to define each project's devops configuration in code by allowing us to define the way the various services should connect. According to the TF definitions we set, the `terrform` CLI will generate and apply a `plan` so that the final configuration matches the desired configuration.

## Project Creation
TODO