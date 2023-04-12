# System Availability

## Applications
We have autoscaling set up at the node pool level and within the Kubernetes deployments. Based on configurable memory and CPU limits the entire system can scale up or down to accommodate current traffic levels. Each Kubernetes deployment scales according to the limits we set within the deployment, and the node pool spins Digital Ocean servers up or down according to each node's limits. These limits are set to 75% memory or 75% CPU usage, but these can be configured according to each application's needs. Typically we set the minimum number of nodes and deployments to 3, to allow for rolling updates and zero-downtime deployments.

## Database
Our standard databsae is PostgreSQL, which we have a standby server in case of failure, and is kept in-sync with the primary server at all times. This is managed by Digital Ocean. In the case of failure, as detected by Digital Ocean, all traffic is directed to the standby database, which becomes the new primary, and a new standby server is automatically spawned. This process happens without any intervention from us, and requires no changes to the application or its environment. Other database software options are available, such as MySQL or MongoDB, with similar functionality. Database snapshots are taken daily and saved for one week. If more frequent backups or backups with a longer duration are required (e.g. 60 days), we can accommodate this.

