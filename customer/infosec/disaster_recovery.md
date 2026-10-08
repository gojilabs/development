# Disaster Recovery

## Overview
In case of a disaster, our choice to spread different parts of the system across different cloud services makes disaster recovery simpler. All services (Kubernetes, database(s)) have failover strategies built-in and are triggered automatically when automated health checks fail. According to the customer's backup frequency and storage duration requirements, we can restore the database to the most recent snapshot within hours.

## Github Outage
In the unlikely event that Github is down, several of the most recent versions of each application are kept in Digital Ocean Container Registry, and depending on the language used, it may be possible to copy the source code from the container. Our first choice in this situation would be to use the code from a developer's laptop that matches what was last deployed.

## Digital Ocean Outage
In the unlikely event that Digital Ocean is down, much of the internet, including some communications and some of the world economy will halt. Because we use Kubernetes, which is an open standard, migrating to another host which supports Kubernetes, like Google Cloud, AWS, or Azure, will require less effort than if we hosted the application directly on virtual machines. We manage initial spin-up of all application environments in Terraform, we would need time to adapt some of our Terraform toolkit to these other cloud providers.

## Cloudflare Outage
If Cloudflare were to go down, a larger portion of the internet would become inaccessible.  In this scenario, we would quickly switch DNS providers to Digital Ocean, and then generate new TLS certificates.

