# Incident Response

This procedure describes how Goji Labs responds to security incidents that affect our systems or our customers' systems. It follows the incident response lifecycle in NIST SP 800-61: preparation, detection and analysis, containment, eradication and recovery, and post-incident activity.

## Definitions
- **Security event:** any observable occurrence that may have security implications, such as a failed login, a Dependabot alert, or an unusual spike in traffic.
- **Security incident:** an event that actually or potentially compromises the confidentiality, integrity, or availability of customer data or systems. Examples include unauthorized access to an account, exposure of credentials, data exfiltration, malicious code, or a successful attack against an application.

## Roles
- **Incident lead:** the CTO. The incident lead coordinates the response, makes containment decisions, and owns communication with the customer.
- **COO:** supports the response, and coordinates contractual and customer communication.
- **Response team:** engineers assembled by the incident lead, chosen according to the systems involved.

Our engineering team spans time zones from US Pacific (UTC-7) to Armenia (UTC+4), so someone is almost always available to respond during the week. On weekends, the CTO or engineering management responds.

## Severity Levels
| Severity | Description | Examples |
| --- | --- | --- |
| **Critical** | Confirmed compromise of customer data or production systems, or an active attack in progress | Exfiltration of a database, attacker access to production, leaked production credentials that have been used |
| **High** | Likely compromise, or exposure of sensitive data or credentials with no confirmed misuse | Credentials committed to a repository, a compromised team member account, an exploitable vulnerability in production |
| **Low** | A security event with limited impact that has been contained | A blocked phishing attempt, a vulnerability with no practical exploit path |

Severity is set by the incident lead, and may be changed as more is learned.

## Reporting an Incident
- **Team members** report suspected incidents to the CTO immediately through Slack. If the CTO is unavailable, they report to engineering management. Team members should report anything suspicious, even if they are unsure. A false alarm costs us very little.
- **Customers** can report suspected incidents to [security@gojilabs.com](mailto:security@gojilabs.com), or through their usual project communication channel.

## Response Procedure
We begin mitigation as soon as we become aware of an incident. Steps within each phase are carried out in parallel wherever possible.

### 1. Detection and analysis
1. Confirm whether the event is an incident, and assign a severity.
2. Identify the affected customers, systems, accounts, and data.
3. **Preserve evidence before making destructive changes.** Export the relevant logs, including:
   - application and infrastructure logs from Loki
   - the GitHub organization audit log
   - the activity logs for Google Workspace, DigitalOcean, and Cloudflare

   Snapshot any compromised workloads, where practical.
4. Open a private incident channel, and keep a timeline of actions and findings.

### 2. Containment
Depending on the scope of the incident:

1. **Identity:** suspend affected Google Workspace accounts, or reset their credentials, and end all of their active sessions. Because DigitalOcean, GitHub, Cloudflare, Slack, and LastPass all require Google single sign-on, this cuts off access to all of them.
2. **Credentials:** revoke and reissue any affected credentials. These include:
   - DigitalOcean API tokens
   - Cloudflare API tokens
   - R2 access keys
   - GitHub personal access tokens, deploy keys, and SSH keys
   - GitHub Actions secrets
   - third-party API keys
3. **Application secrets:** rotate database passwords and other application secrets, update the Kubernetes Secrets, and restart the affected workloads.
4. **Accounts:** review the member lists of GitHub, DigitalOcean, Cloudflare, and Google Workspace, and remove any unknown accounts.
5. **Workloads:** remove any unrecognized workloads from the cluster.
6. **Network:** if necessary, take an affected service offline by disabling its Cloudflare Tunnel route, or block malicious traffic with Cloudflare WAF rules. On the legacy platform, clear all managed database IP allowlists.
7. **Providers:** where appropriate, contact the support or security teams of DigitalOcean, Cloudflare, and GitHub.

### 3. Eradication and recovery
1. Determine the root cause and how the attacker got in.
2. Review recent changes in the source code and in the infrastructure (Terraform) for unauthorized alterations.
3. Remove malicious code or artifacts, and fix the vulnerability that was exploited.
4. Update any vulnerable third-party libraries and base images.
5. Rebuild affected services from trusted source code through our normal CI/CD pipeline. If data has been altered or destroyed, restore it from a backup taken before the compromise. See [Business Continuity](./business_continuity.md).
6. Monitor closely to confirm that the threat has been eliminated.

## Customer Notification
- We notify affected customers within **48 hours** of becoming aware of an incident that affects them, and sooner if the customer's input would help the response.
- Where a contract or Business Associate Agreement specifies a shorter notification period, or specific content for the notice, those terms take precedence.
- Our notice describes what happened, the data and systems affected, the actions we have taken, and any actions we recommend the customer take. We provide updates as the investigation progresses.
- Our customers decide whether, and how, to notify their own users, partners, and regulators. We assist by providing the technical details of the cause and the remediation.

## Post-Incident Activity
- We provide the customer with a written incident report. It includes a timeline, the root cause, the impact, the remediation, and the steps taken to prevent the incident from happening again.
- Follow-up actions are tracked until they are complete.
- Lessons learned are fed back into these policies and our development practices.

## Testing
We will run a tabletop exercise of this procedure annually. See the [Roadmap](../infosec.md#roadmap).
