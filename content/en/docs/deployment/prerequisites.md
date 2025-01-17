---
title: "Prerequisites"
linkTitle: "Prerequisites"
weight: 1
---

Axway Open Banking is developed on Kubernetes with a "button-click" style deployment that allows customers to use the Kubernetes solution of their choice.

Preparing a Kubernetes cluster with the appropriate services and settings is required prior to the solution installation.

## General prerequisites

Prior to installation you need to perform the following tasks:

* Read and understand the Architecture Overview guide.
* Make choices that are described in the Architecture Overview guide including:
    * Choose a Kubernetes provider (cloud, on-premise, and so on).
    * Components that will be supported (Demo Applications, mock backend services, and so on).
    * Approach to database deployment (inside Kubernetes versus externalized services).
    * Components that reflect deployment model choice (certificate manager, load balancer/Ingress Controller, and so on).
* Install the following command line tools:
    * Helm
    * Kubectl
* Obtain a token from an Axway team to pull Helm charts and Docker images from with the Axway Registry.
* Create a Kubernetes cluster that conforms to that described in the Architecture Overview guide and reflects the architecture choices described above.

These tasks must be completed for a successful installation.

## Kubernetes setup requirements

A Kubernetes 1.16+ cluster is required to deploy the Axway Open Banking Solution.

### Resources

Each node in the Kubernetes environment requires:

* 23 virtual CPUs
* 70 Gb RAM

Axway also recommends using Node Groups. Node Groups allow operators to group resources by node type based on characteristics such as machine resources, capabilities, or the virtual machine type. Taking this approach can reduce costs, increase performance, and allow specific type of machines to be managed discretely.

The Kubernetes configuration must include three Node Groups:

* *Application*: Hosts all Axway Open Banking components:
    * Some components will have a Horizontal Pod Autoscaler to support the peak load (Axway recommends configuring a node autoscaler).
    * Most components, particularly API Gateway, require low latency.
* *Transversal*: Hosts non-application components such as monitoring tools and infrastructure components such as the Certificate Manager and external DNS. This group can be configured without a node autoscaler.
* *Database*: Hosts all stateful applications.

An affinity node is used on each component to deploy them on the appropriate nodes.

In cases where a customer deploys all their database within the Kubernetes cluster Axway recommends dedicating Cassandra pods to nodes on a one-to-one basis.

{{% alert title="Note" color="primary" %}} The configuration of master nodes is out-of-scope on this page.{{% /alert %}}

### Subnets

A complete architecture requires a minimum of 3 subnets:

* *Bastion*: Required for operators to connect from a Bastion host (although this can be substituted for an alternative solution). Access to pods on all required ports must be allowed. A subnet mask of /32 is considered sufficient.
* *Kubernetes*: The design of this subnet can vary based on a number of conditions:
    * This subnet must have access to both the customer backend services and the database subnet.
    * If CNI is activated, enough IP addresses must be allocated for nodes and the maximum number of pods in the platform. Axway Open Banking generates between 100 and 120 objects that consume an IP address.
    * A subnet mask /24 is therefore recommended to support scaling, upgrade, and others tools for production.
* *Database*: For databases provided inside the Kubernetes cluster a subnet mask of /29 is recommended.

Each subnet must be protected by a firewall implemented at Layer 4 of the OSI model with open routes kept to a bare minium.

### Kubernetes components

The following components are highly recommended.

#### Ingress controller

To control all external traffic, an Ingress controller is required. It is recommended to use the [Nginx Ingress](https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx) controller to use as a reverse proxy and manage the MTLS and TLS termination, and load-balancing when required.

Select the appropriate version that is compatible with your cluster with a minimum of 0.35. The ingress annotations of our Helm chart have been written for the 0.35 version. A more recent version of the `nginx-ingress` may impact these  annotations. Review the nginx official documentation to update them accordingly.

You can use NGINX or another ingress controller with the following requirements:

* Encode certificate in header X-SSL-CERT in web format.
* Return http error 400 if client uses a bad certificate.
* Manage multiple root CAs according to different client certificates.
* Limit cypher spec usage to “DHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384".
* Use request header size compatible with 8K.
* Deny public access to ACP path `/app/default/admin`.

#### Certificate Manager

We recommend that you use [Cert-Manager](https://github.com/jetstack/cert-manager/tree/master/deploy/charts/cert-manager) to easily manage certificates that are used in the Axway Open Banking Solution.

If you have specific certificates you want to use during installation, you can avoid using *cert-manager* but more configuration is required during deployment.

#### DNS

It is also highly recommended to use [External-DNS](https://github.com/bitnami/charts/tree/master/bitnami/external-dns) to synchronize ingress hosts in a DNS zone.

In case `external-dns` is not available in the cluster, you must manually configure the ingress host in your DNS zone. Also remove the cert-manager annotation in all ingress hosts.

Axway uses the Externally Managed Topology (EMT) approach for scaling so instances can be managed by Kubernetes.

Read [our guide](https://docs.axway.com/bundle/axway-open-docs/page/docs/apim_installation/apigw_containers/container_getstarted/index.html) on using EMT for further details.