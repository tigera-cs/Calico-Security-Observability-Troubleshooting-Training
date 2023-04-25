# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/tigera-cs/Calico-Security-Observability-Troubleshooting-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#overview)
* [Cluster applications and connectivity](https://github.com/tigera-cs/Calico-Security-Observability-Troubleshooting-Training/blob/main/2.%20Secure%20Pod%20Traffic%20Using%20Calico%20Security%20Policy/README.md#cluster-applications-and-connectivity)
* [Deploy Sample Microservices]()
* [Install Calico Enterprise command line utility "calicoctl"](https://github.com/tigera-cs/Calico-Security-Observability-Troubleshooting-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#install-calico-enterprise-command-line-utility-calicoctl)
* [Deploy a three-tier sample application called "yaobank" (Yet Another Online Bank)](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#deploy-a-three-tier-sample-application-called-yaobank-yet-another-online-bank)
* [Access CE Manager UI using Ingress](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#access-ce-manager-ui-using-ingress)


### Overview

Kubernetes' network model is both flexible and dynamic, enabling applications to be deployed anywhere in the cluster without being tied to the underlying network infrastructure. However, this can present challenges for legacy network firewalls that struggle to keep up with Kubernetes' constantly changing workloads. Calico provides a robust security policy framework that defines and enforces policies, ensuring that only authorized traffic flows between pods and services. Calico's powerful policy engine enhances the security of microservices architectures without compromising the flexibility and agility of Kubernetes. In this lab, we'll demonstrate how to use Calico's policy engine to secure some example microservices that blong to different tenants.

### Cluster applications and connectivity

This lab uses two applications that run across 4 namespaces and belong to two tenants, Tenant-1 and Tenant-2. Star app pods run in three namespaces (management-ui, client, stars) and belong to Tenant-1. Yaobank pods runs in a single and belong to Tenant-2. The ingress-nginx namespace hosts an ingress controller, which will be used to connect to the applications through the respective frontend microservice for each application. The ingress-nginx and kube-system namespaces are owned by the platform and are managed independent of the tenant namespaces.

#### `Tenant-1`
  - Star app
    -  management-ui namesapce
    -  client namesapce
    -  stars namesapce
  
#### `Tenant-2`
  - Yaobank app
    -  yaobank namespace



<h1 align="center">Cluster Topology</h1>


<p align="center">
<img src="img/3.tenants.png">
</p>


### Deploy Sample Microservices
