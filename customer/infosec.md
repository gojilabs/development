# Information Security

Goji Labs designs, builds, hosts, and maintains software for our customers. This section describes how we protect our customers' source code, data, and systems. These documents are Goji Labs' information security policies. They are owned by our CTO and COO and apply to everyone who works on Goji Labs projects.

The practices described here are in effect for all projects as of January 1, 2026, unless noted otherwise. Planned improvements are clearly labeled and listed in the [Roadmap](#roadmap) below; anything not labeled as planned is current practice.

## Policies

- [Security Governance](./infosec/governance.md): policy ownership, personnel security, security awareness, data handling, and third-party service providers
- [Access Management](./infosec/access_management.md): identity, single sign-on, multi-factor authentication, and the access lifecycle
- [Infrastructure Security](./infosec/infrastructure_security.md): hosting, network isolation, encryption, secrets, and monitoring
- [Vulnerability Management](./infosec/vulnerability_management.md): dependency and secret scanning, patching, and penetration testing
- [Incident Response](./infosec/incident_response.md): how we detect, contain, recover from, and report security incidents
- [Secure Software Development](./infosec/secure_sdlc.md): our SDLC, code review, CI/CD, and AI-assisted development
- [Business Continuity](./infosec/business_continuity.md): availability, backups, and disaster recovery

## Hosting Models

We support two hosting models. Unless otherwise stated, these documents describe the first.

1. **Goji Labs-managed hosting (default).** Backend services run on Kubernetes in DigitalOcean, and frontend services run on Cloudflare Workers. Every customer has a dedicated Cloudflare account and dedicated Kubernetes namespaces for each environment.
2. **Customer-owned cloud.** We build and deploy into the customer's own cloud account (e.g. AWS, Google Cloud, or Azure). This is common when a customer has HIPAA or other compliance requirements. In this model, infrastructure controls are governed by the customer's cloud account and policies, while our access management, secure development, and incident response practices continue to apply to our team and our work.

### Platform generations

Projects started on or after January 1, 2026 run on our current platform, in which each customer's databases, caches, and queues run inside that customer's own isolated Kubernetes namespace. Some projects started before that date still use shared, DigitalOcean-managed data stores. Where the two differ, the relevant document describes both.

## Compliance

We have Business Associate Agreements (BAAs) in place for several HIPAA-compliant applications we have built. We design to the regulatory requirements of each project, such as HIPAA, GDPR, and CCPA. Compliance requirements are easiest and least expensive to meet when they are identified at the very beginning of a project, because they often shape architectural and system design decisions. We ask every customer about them during discovery.

Goji Labs does not currently hold a SOC 2 or ISO 27001 certification.

## Roadmap

The following improvements are planned and are **not yet in effect**:

| Improvement | Target |
| --- | --- |
| Namespace-scoped role-based access control, limiting each developer's production access to the projects they work on | Q1 2027 |
| Application-specific alerting (in addition to the infrastructure alerting already in place) | Q1 2027 |
| First third-party penetration test | Q1 2027 |
| Incident response tabletop exercise, repeated annually thereafter | December 2026 |
