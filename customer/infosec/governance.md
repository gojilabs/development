# Security Governance

## Policy Ownership
Goji Labs' information security documents serve as our security policies. They are owned, reviewed, and approved by our CTO and COO. These policies are reviewed every six months. The documents are kept under version control, so every change is recorded with its author and date.

## Roles and Responsibilities
- **CTO:** leads the security program, approves access to production infrastructure, and leads incident response.
- **COO:** co-owns security policy, and participates in incident response and customer communication.
- **Engineering management:** provides escalation coverage, including on weekends.
- **Project architects:** act as the technical authority for their projects, and are responsible for overall code quality and for reviewing other developers' code.
- **Everyone:** follows these policies, protects company and customer credentials, and promptly reports anything suspicious.

## Personnel Security
Our team is made up of US-based full-time employees and long-term contractors located in time zones from the US Pacific coast to Armenia. Our long-term contractors work with us full-time, and are held to the same security policies and practices as our employees.

- Everyone signs a confidentiality agreement and acknowledges our policies when they join.
- Everyone receives a Goji Labs Google Workspace account, which is their sole identity for company systems. Personal email accounts are never used with company or customer services.
- Joining and departing team members are processed against a written onboarding and offboarding checklist. A departing team member's access is revoked within one hour of their departure. See [Access Management](./access_management.md).

## Security Awareness
- Every quarter we share real examples of phishing attempts with the whole team, so everyone knows what current attacks look like.
- We maintain a culture of openness. Team members are encouraged to raise anything that looks suspicious, including their own mistakes, without fear of blame. Early reporting is the most effective defense we have.
- Internal communication happens in Slack, not email. Slack requires Google single sign-on with a gojilabs.com account, so an email that claims to come from a colleague or from management is treated as suspicious by default and reported.

## Data Handling
- Customer data is used only to provide services to that customer.
- Non-production environments use synthetic data. Where synthetic data is not suitable, we use anonymized data. Staging environments are separated from production environments (see [Infrastructure Security](./infrastructure_security.md)).
- Data is encrypted in transit and at rest. See [Infrastructure Security](./infrastructure_security.md).
- When a project handles regulated data, such as protected health information (PHI), we identify the requirements at the start of the project and design for them. We sign BAAs where required, and may recommend hosting in the customer's own cloud account.

## AI Tools
- Our developers use Claude Code (Anthropic) for AI-assisted software development. Under our commercial terms, Anthropic does not train its models on our data.
- Production data, protected health information (PHI), and credentials are never provided to AI tools.
- AI-generated code goes through exactly the same review, automated checks, and QA as any other code. See [Secure Software Development](./secure_sdlc.md).
- Our design team uses Figma and Paper.

## Third-Party Service Providers
The following providers support our standard development and hosting practices. Individual projects may use additional providers, such as email, SMS, or payment services, which are agreed with the customer during the project.

| Provider | Purpose | Data involved |
| --- | --- | --- |
| DigitalOcean | Kubernetes hosting, container registry, managed databases (legacy projects) | Application data and backend services |
| Cloudflare | DNS, CDN, DDoS protection, WAF, Cloudflare Tunnel, Workers (frontend hosting), R2 object storage (files and database backups) | Application data and files, web traffic |
| GitHub | Source code hosting, code review, CI/CD | Source code, deployment credentials |
| Google Workspace | Identity provider, email, documents | Business communications |
| Slack | Internal communication | Project discussions |
| LastPass | Credential storage and sharing | Credentials |
| Anthropic (Claude Code) | AI-assisted software development | Source code |
| Figma, Paper | Product design | Designs |

Customers who choose hosting in their own cloud account do not use DigitalOcean or Goji Labs' Cloudflare-based hosting.
