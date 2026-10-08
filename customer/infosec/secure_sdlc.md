# Secure Software Development

## Standards
Our developers follow the OWASP Top 10 and the security guidance of the frameworks and languages each project uses. Project architects are responsible for overall code quality, and for making sure these practices are followed on their projects.

## Planning and Design
- **Compliance requirements first.** During discovery, we ask every customer about regulatory and compliance requirements, such as HIPAA, GDPR, and CCPA. These requirements often drive architectural decisions, such as where data is hosted, what is stored, and how it is encrypted, so we identify them before design begins.
- **Threat modeling.** We perform threat modeling when a customer requests it or the risk profile of a project calls for it.

## Source Control
- All code is version-controlled with Git, in private GitHub repositories.
- Access is limited to the developers working on each project. See [Access Management](./access_management.md).
- Secrets are never committed to source code. GitHub secret scanning detects accidental commits.

## Code Review
Every change is submitted as a pull request. The default branch is protected, and a pull request cannot be merged until it has:

1. at least one approving review from a developer, who evaluates the change for correctness, simplicity, and clarity
2. passed an automated lint and style check
3. passed an automated, in-depth technical review that analyzes the change for defects
4. passed automated security scans

Force pushes are disabled on protected branches, so the history of reviewed code cannot be rewritten.

## AI-Assisted Development
Our developers use Claude Code, under commercial terms in which Anthropic does not train its models on our data. Code written with AI assistance is held to exactly the same standards as any other code. It goes through the same pull request review, automated checks, QA, and staging process, and the developer who submits it is accountable for it.

## Dependencies and Licensing
- GitHub Dependabot monitors dependencies for known vulnerabilities. See [Vulnerability Management](./vulnerability_management.md).
- We check the license of every open-source dependency. If the only available option for a need is under a copyleft license such as the GPL, which may not be compatible with commercial use, we discuss the tradeoffs with the customer before adopting it.

## Environments and Release Process
Each project has separate staging and production environments, and each environment runs in its own Kubernetes namespace. Staging uses synthetic data or, where synthetic data is not suitable, anonymized data.

1. A change is merged after review, and is deployed to staging.
2. The change is tested and approved in staging through QA.
3. The approved change is promoted to production.

**Hotfixes:** an urgent fix may go directly to production only if it is both very low risk and highly urgent. Hotfixes still require a pull request and the same reviews and automated checks. Any hotfix that does not meet both criteria follows the standard process.

## Continuous Integration and Deployment
Deployments are automated with GitHub Actions.

1. When a pull request is merged, a GitHub Actions workflow builds the backend container images from the reviewed source code.
2. Images are tagged with the Git commit they were built from, so every running container can be traced back to an exact, reviewed version of the code.
3. Images are pushed to a private DigitalOcean Container Registry, and the workflow updates the image used by the Kubernetes deployment.
4. Kubernetes performs a rolling update. New pods must pass health checks before they receive traffic, and only then are the old pods terminated. This means deployments cause no downtime, and a failed deployment never replaces a healthy one.
5. Frontend services are deployed to Cloudflare Workers in the customer's dedicated Cloudflare account.

Because previous images remain in the registry, we can roll back quickly to a known-good version.

## Infrastructure as Code
Infrastructure is defined in Terraform, which is stored in private GitHub repositories, so infrastructure changes are version-controlled like application code.

## Mobile Applications
iOS and Android apps are signed and published through dedicated developer accounts, whose credentials are stored in LastPass. Mobile builds go through the same pull request and review process as all other code.
