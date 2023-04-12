# Security Breach Response

In the event of a security breach, our first priority is to prevent further damage. The CTO will assemble a team and begin mitigation efforts immediately.

## Immediate Response
Most of these actions can be taken in parallel by the response team.

1. Notify the customer.
2. Reach out to Cloudflare, Digital Ocean, and Github to notify them of a breach, asking for their support.
3. Reset all employee passwords to the Google Workspace, Github, and Digital Ocean accounts, and end all active sessions in these services along with our password manager.
4. Rotate all SSH keys.
5. Review the Github, Digital Ocean, and Cloudflare team member lists, and remove any unknown accounts.
6. Expire and regenerate all existing Digital Ocean tokens.
7. Clear all IP address whitelists.
8. Remove any foreign servers running within the VPC.
9. Regenerate all database passwords, install the new database URIs in the application environment, and restart the application.

## Further Response
1. Review the source code using "git blame" to find any suspicious code alterations.
2. Update third-party libraries.
3. Reset credentials/rotate API keys for all third-party services.
4. Review the Github and Digital Ocean event logs, to better understand the chain of events.
5. Provide a report to the customer. If the customer chooses to notify their partners and users, we can assist in messaging the technical cause and solution.

