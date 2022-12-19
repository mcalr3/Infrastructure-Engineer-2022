# 2022 Platform Engineer (Infrastructure) Technical Scenarios

## **Architecture scenario**

Consider following environment entirely deployed inside Google Cloud:

- A Frontend Web application deployed in a public zonal GKE cluster running version 1.13.11 (using public docker images) inside project A
- A Backend application deployed inside a GCE instance with a dedicated external IP address in project B
- A Postgres DB instance (used by the Web Application without SSL encryption) deployed in CloudSQL with public IP in project B
- A Cloud Storage bucket with object ACLs enabled is used as a pull location for occasional releases in project C
- All services are deployed in default VPC networks in a same region.

### I. **What steps would you take to improve overall security of the given environment?** (Please justify your choices.)

### II. **How would you go about implementing a monitoring solution for all these resources and what tools would you use?** (Please justify your choices.)

- REQUIREMENT: Logs/monitoring need to be in a centralised project D.

---

## **Networking scenario**

### I. **How would you implement a publicly facing Web service (with a public DNS and certificate) hosted inside a private Kubernetes cluster to allow for zero downtime?**

Please explain your choices and how would you manage updates to the Web service from a managed CI service (hosted on vendor's infrastructure).


---

## **Answers - RM 19/12/22 **
### Architecture scenario I

**Frontend Security Improvements:**

- Instead of using a Public zonal GKE cluster, a Private zonal cluster should be used. This means that the container running the webapp would not be publically accessible but instead, it would have a privately-routable IP address and in order for the webapp to be publically accessible, it should sit behind a Global External HTTPS CLB (Cloud Load Balancing) load balancer. 
- To further improve ingress security and protect from malicious attacks such as DDOS, a Web Application Firewall (WAF) such as Cloud Armor could also be introduced to sit in front of the load balancer (to drop suspicious packets before they are processed by the CLB and therefore mitigate any ingress traffic cost associated with an attack).
- If the webapp contains lots of static assets, it would also make sense to place a Cloud CDN in front of the load balancer also, in order to reduce load balancer ingress traffic, load on the GKE service/cluster, and provide an additional layer of security.
- Public docker images should be scanned upon docker pull, using an appropriate container-scanning tool such as Snyk or Falcon Crowdstrike to identify man in the middle (MITM) and prevent attacks by using public containers. In addition, a more secure solution (and cheaper as not involving Docker Hub subscription fees), would be to pull the public docker image (including any custom build steps) to a private container repository such as one in GCP. Each of these steps could be carried out in CICD.
- A much later version of GKE should be used as 1.13.11 contains a number of known vulnerabilities. The latest stable release being 1.23.
- SCA (static code analysis) should be carried out on the project using a tool such as Snyk or Sonarcloud to identify any vulnerabilities in the code.

**Backend Security Improvements:**

- The external/public IP address should be removed from the backend instance, and instead a private IP should be used, along with VPC Network Peering in between the VPCs of each project. Route tables and specific firewall rules should be opened between the front & backend apps. The backend would then have access via a seperate firewall rule to the DB instance.

**DB Instance Security Improvements:**

- Again, a DB should not have a public IP. This should be changed to a private network and have appropriate firewall rules to allow DB access from the backend instance.
- SSL encryption to the postgres DB should be enabled. To further improve security this could a custom created SSL certificate controlled by the organisation rather than using a publically/GCP issued cert.

**Cloud Storage Security Improvements:**

- Rather than having an object level ACL in the bucket, IAM should be used where possible to enable easier management of permissions (less likely to leave permissions open) and to restrict the permissions to only resources within the appropriate Project (frontend/backend).
- SCA tool such as Sonarcloud with approprate GCP policies would also allow overly-permissive IAM policies to be identified upon commit of the IAC during CICD.

### Architecture scenario II

**Monitoring the above resources:**
- A GCP Cloud Monitoring dashboard could be used in a seperate Project D by adding the other projects as Monitored Projects. This can be achieved using IAM and the `roles/monitoring.viewer` permission, granted to the assumed role from Project D. 

The dashboard provides a single pane of glass for observability and would include at least the following, some of which would be actively monitored to perform scaling actions or alert an engineer:

- Alerts for HTTP 4XX/5XXs on the frontend load balancer or application endpoint - we would only want a certain threshold of these errors to occur and if there is too high a threshold then this would indicate a bad deployment. These alerts could trigger a manual alert to slack for example but a healthy HTTP 200 should also be used as a healthcheck on the container during deployment of a new container/pod definition. If the endpoint is not healthy then deployment should be rolled back (as would occur with deployment using terraform, for example).
- Monitoring CPU & memory usage on the frontend container pool would NOT usually be used to send manual alerts to slack as it would create too much noise but instead should be used to form a scaling policy for the pod/service definition so that it can dynamically scale based on demand.
- CPU & Memory should be monitored on the backend instance as it is not automatically scaled and therefore if it is running low on resources it should be scaled to the next available instance size to avoid outage.
- Alerts for the Postgres DB covering memory, disk space, disk queue (indicating higher IOPS needed), virtual memory/paging use, network bandwidth.
- Logs ingestion could be used to collect logs from the frontend and backend and log-based alerts can be set up when certain strings are found in the logs, such as `ERROR` lines pertaining to an issue in the application, and thus should trigger engineer action.

### Networking scenario

**Zero downtime deployments:**

Web service deployments can be achieved in a number of ways. It depends on a combination of compute costs vs availability how much downtime an organisation is _potentially_ willing to accomodate.

Above in section 1, I have explained that I would use a public load balancer routed to a private EKS cluster utilising VPC peering to connect parts of the application hosted in different GCP Projects.

One of the best ways of ensuring that there is *zero* downtime during a web service deployment is to use blue/green deployments.
This essentially keeps 2 versions of the web application running in 2 different services in the cluster. Say the current production version serving live traffic is blue, then the new version to be deployed would update the green service and then the load balancer would be updated to point to the DNS of the green service during CICD.

It would work like this:

Prior to new deployment

- LIVE(blue): `web.company.io > gcploadbalancer.projecta.cloud > blue.web.company.io > blue.projecta.cluster.eks`

- LIVE(green/standby): `DNS-BLACKHOLE > green.web.company.io > green.projecta.cluster.eks`

After new deployment

- LIVE(blue): `DNS-BLACKHOLE > blue.web.company.io > blue.projecta.cluster.eks`

- LIVE(green/standby): `web.company.io > gcploadbalancer.projecta.cloud > green.web.company.io > green.projecta.cluster.eks`

Using the above configured alarms and healthchecks, when a service is being deployed, a post-deployment healthcheck would be performed and then if the new/green service doesn't become healthy it would immediately be reverted to the blue service (by updating the target behind the load balancer), and the pipeline will fail and prevent the merge in git.