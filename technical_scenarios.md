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
