# Access Management

## Principles
- Every person has a unique, named identity. Shared logins are used only where a provider supports nothing else, and they are stored in our password manager.
- Access is centralized through a single identity provider protected by multi-factor authentication.
- Access is granted according to role and project, and removed promptly when it is no longer needed.

## Identity and Single Sign-On
Google Workspace is our identity provider. Every team member has a gojilabs.com account, and the following services require single sign-on through Google Workspace:

- DigitalOcean
- GitHub
- Cloudflare
- Slack
- LastPass

Because access to these services is tied to the Google Workspace account, suspending a single account removes a person's access to all of them.

## Multi-Factor Authentication
Multi-factor authentication (MFA) is enforced for every Google Workspace account at the organization level. SMS is not permitted as a second factor. Because every service listed above requires Google single sign-on, MFA also applies to every sign-in to our hosting, source code, DNS, communication, and password management services.

## Password Management
We use LastPass to store and share credentials when sharing is necessary. LastPass is itself accessed through Google single sign-on. Credentials are never shared over Slack, email, or other channels. Accounts that do not support single sign-on, such as the dedicated accounts used to sign and publish mobile apps, are stored in LastPass.

## Source Code Access
All source code is hosted in private GitHub repositories. Access is granted through GitHub teams, so that only the developers actively working on a project have access to its repositories. SSH keys are used only to push code to GitHub.

## Production Infrastructure Access
Access to DigitalOcean, and through it to our Kubernetes clusters, is currently held by most backend developers and by our CTO, COO, and CEO. Most frontend developers do not have access. Access to the Kubernetes API requires authentication with DigitalOcean, which requires Google single sign-on and MFA.

We do not use SSH or remote desktop to access servers, and our containers contain no SSH or remote desktop tools. Administrative work is performed with `kubectl`. For database maintenance, we run a purpose-built maintenance container inside the customer's namespace, so that databases are never exposed outside the cluster. See [Infrastructure Security](./infrastructure_security.md).

**Planned (Q1 2027):** role-based access control scoped to Kubernetes namespaces, so that each developer can only access the namespaces of the projects they work on.

## Service Accounts and Secrets
- DigitalOcean API tokens are created with the minimum permissions needed for their purpose.
- CI/CD credentials are stored as encrypted GitHub Actions secrets, scoped to the repository and environment that need them.
- Application secrets are stored as Kubernetes Secrets in the customer's namespace, and are never committed to source code. GitHub secret scanning is enabled to detect accidental commits.

## Customer Access
Each customer has a dedicated Cloudflare account and dedicated Kubernetes namespaces. On request, we can grant the customer's own staff access to these, as well as to their GitHub repositories.

## Access Lifecycle
- **Onboarding:** new team members are provisioned against a written checklist, and receive only the access their role and projects require.
- **Offboarding:** a departing team member's access is revoked within one hour of their departure, following a written checklist.
- **Access reviews:** we review access every quarter, and remove any access that is unknown or no longer needed.
