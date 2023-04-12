# Access Control

## Overview
We host all applications in Digital Ocean and DNS is managed in Cloudflare. Applications run in a serverless, containerized environment and are orchestrated by Kubernetes.

## Public Web
Web requests enter the cluster via one HTTPS port through a Digital Ocean Load Balancer. No other ports are open. Cloudflare proxies all request to the load balancer, so the load balancer's IP address is not publicly known, and this provides us with a basic level of DDoS protection. We enforce full, strict HTTPS through Cloudflare, which requires valid TLS origin certificates on the Kubernetes ingress. 

## Kubernetes
Console access to the Kubernetes cluster is available through the kubectl command, which requires either a private key for the Kubernetes cluster or a Digital Ocean access token, which requires access to our Digital Ocean team. Only specific individuals in our organization are granted access, and two-factor authentication is required for all authentication with Digital Ocean. The Kubernetes cluster has no public IP address, and it exists in a VPC. SSH and RDP tools are not installed in any of our containers.

## Database
For any databases required by the application (e.g. PostgreSQL, Redis) we rely on Digital Ocean managed databases with private VPC connections. These have both private and public IP addresses, but public access is protected by an IP whitelist. Except for when long-running manual database operations are underway (e.g. manual backup, large migrations), this IP whitelist is empty. If customer policies allow, we can add an administrator's IP address to the whitelist temporarily in order to execute a long-running operation, but then we manually clear the IP whitelist once the operation is complete. For shorter-running tasks, we are able to use kubectl to gain console access to a running container, or a specialized container for this purpose, and execute command from within the VPC. This method is preferred over direct database access but is not always practical. Our applications connect to the database(s) from within the same VPC.

## Source Code
All source code is hosted in private Github repositories. Access is controlled by teams in Github, where only the developers who are actively working on a project have access to said project. Our authentication rules in Github enforce either two-factor authentication for each user or SSO via our Google Workspace.

## Internal Workflow
All of our employees have their own Goji Labs email accounts provided by our Google Workspace. Two-factor authentication is enforced at the organization level for all employees. Personal email accounts are not allowed to be used with any service. All of our internal communication happens over Slack, and employees are trained to report any emails claiming to be internal or from management. We use a popular password management service to share credentials when necessary, which uses SSO with our Google Workspace for authentication.
