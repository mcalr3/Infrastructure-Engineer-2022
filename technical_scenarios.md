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
### 1. Architecture scenario

**Frontend Security Improvements:**

- Instead of using a Public zonal GKE cluster, a Private zonal cluster should be used. This means that the container running the webapp would not be publically accessible but instead, it would have a privately-routable IP address and in order for the webapp to be publically accessible, it should sit behind a Global External HTTPS CLB (Cloud Load Balancing) load balancer. 
- To further improve ingress security and protect from malicious attacks such as DDOS, a Web Application Firewall (WAF) such as Cloud Armor could also be introduced to sit in front of the load balancer (to drop suspicious packets before they are processed by the CLB and therefore mitigate any ingress traffic cost associated with an attack).
- If the webapp contains lots of static assets, it would also make sense to place a Cloud CDN in front of the load balancer also, in order to reduce load balancer ingress traffic, load on the GKE service/cluster, and provide an additional layer of security.
- Public docker images should be scanned upon docker pull, using an appropriate container-scanning tool such as Snyk or Falcon Crowdstrike to identify man in the middle (MITM) and prevent attacks by using public containers. A more secure solution (and cheaper as not involving Docker Hub subscription fees), would be to pull the public docker image (including any custom build steps) to a private container repository such as one in GCP. Each of these steps could be carried out in CICD.
- A much later version of GKE should be used as 1.13.11 contains a number of known vulnerabilities. 
- SCA (static code analysis) should be carried out on the project using a tool such as Snyk or Sonarcloud to identify any vulnerabilities in the code.

**Backend Security Improvements:**

- The external/public IP address should be removed from the backend instance, and instead a private IP should be used, along with VPC Network Peering in between the VPCs of each project. Route tables and specific firewall rules should be opened between the front & backend apps. The backend would then have access via a seperate firewall rule to the DB instance.

**DB Instance Security Improvements:**

- Again, a DB should not have a public IP. This should be changed to a private network and have appropriate firewall rules to allow DB access from the backend instance.
- SSL encryption to the postgres DB should be enabled. To further improve security this could a custom created SSL certificate controlled by the organisation rather than using a publically/GCP issued cert.

**Cloud Storage Security Improvements:**

- Rather than having an object level ACL in the bucket, IAM should be used where possible to enable easier management of permissions (less likely to leave permissions open) and to restrict the permissions to only resources within the appropriate Project (frontend/backend).
- SCA tool such as Sonarcloud with approprate GCP policies would also allow overly-permissive IAM policies to be identified upon commit of the IAC during CICD.
