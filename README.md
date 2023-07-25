# GKE Peering with Service Discovery Solution

## What
Peering multiple gke clusters from different projects with service discovery

## Why
Better breakdown of cost, since clusters are separated into different projects, giving us more information of the operating costs of the infrastructure.

# Internal Load Balancer with External DNS
Services can be exposed within the peered clusters using a kubernetes `LoadBalancer` service type, we can set this up together with [external-dns](https://github.com/kubernetes-sigs/external-dns) with a private dns zone configured across the peered networks for service discovery.

## Prerequisites
- target networks should be peered together
- firewall should be configured to allow connection between each vpc

## Private DNS Zone
DNS zone should have been configured and make sure that the peered networks are added into it. DNS zones can be managed on [cloud dns](https://console.cloud.google.com/net-services/dns/zones/new/create).

## Install External DNS
After configuring a dns zone, external-dns should be installed on the clusters that we want services to be discovered. External DNS will automatically sync dns records on the target zone so other services can reach the target service using a hostname assigned to it. 

Follow this [helm chart](https://github.com/kubernetes-sigs/external-dns/blob/master/charts/external-dns/README.md) for installation.

## Exposing a Service
To expose a service using this approach, create an internal load balancer for the service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-name
  annotations:
    # allocate a private ip instead of a public ip
    networking.gke.io/load-balancer-type: "Internal"
    # sync this dns record on the configured dns zone
    external-dns.alpha.kubernetes.io/hostname: <service-sub-domain>.<dns-zone-name>
spec:
  type: LoadBalancer
```

## Terraform
Implementation 

# GKE Multi-cluster Services (MCS)
Multi-cluster Services (MCS) is a gcp solution for managing service discovery across a fleet. Services are exposed to other clusters by using a `ServiceExport` crd, then the operator should automatically export the services across the fleet on the same namespaces. The service can be reached through `<SERVICE_EXPORT_NAME>.<NAMESPACE>.svc.clusterset.local`

## Prerequisites
- target networks should be peered together
- each gke cluster are registered on the same fleet

## Enabling MCS
Refer to the documentation below on how to enable mcs on a fleet

- [Configuring multi-cluster Services](https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-services)
- [Multiple Projects Setup (Cross Project Communication)](https://cloud.google.com/kubernetes-engine/docs/how-to/msc-setup-with-shared-vpc-networks#vpc_setup_fleet_host)

## Terraform
for more information [MCS Terrafrom](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/gke_hub_membership)

```terraform

```

## Using MCS
Assume a namespace named `example`, and service `nginx` on that namespace. To export that service across the fleet, create a `ServiceExport` resource on that namespace with the name of the service.

```yaml
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
 # namespace where the service is located
 namespace: example
 # name of the service
 name: nginx
```

After applying this resource, the operator will automatically create a `ServiceImport` resource on the same namespace across the fleet.

`Note: namespace should be existing on the other clusters, the operator as of now does not automatically create the namespace for the service`

Once the service is successfully imported, it should be resolved by `nginx.example.svc.clusterset.local`. To verify, run a debug pod on the cluster and test.

```bash
#!/bin/bash
# run a debug pod
kubectl run mcs-test --image=ubuntu -it --rm -- bash

# install curl, dnsutil
apt-get update
apt-get install -y curl dnsutil

dig +short nginx.example.svc.clusterset.local
curl http://example.svc.clusterset.local
```

For more debugging information, check the operator pod logs on `gke-mcs` namespace.

# External DNS vs GKE MCS
- [+] Pros
- [-] Cons
- [=] neither

| Criteria | With External DNS | GKE MCS |
| :------: | ----------------: | ------: |
| **Operator Installation** | [-] requires external-dns installation on each cluster | [+] installation is automatically done after enabling mcs on the fleet |
| **Firewall** | [-] manual firewall rules | [+] firewall is automatically managed |
| **DNS Configuration** | [-] manual dns zone config | [+] no manual configuration of dns is required |
| **Usage** | [=] hostnames are managed through metadata annotations | [=] services are exposed through crds `ServiceExport/ServiceImport` |
| **IAM Bindings** | [+] less iam-bindings (just need to allow external-dns KSA to manage the dns zone) | [-] more iam-bindings, since the operator manages more things |
| **Public DNS Management** | [+] can be an alternative to managing public dns records, instead of terraform, as it is purposedly built for this function | [=] not possible to manage public dns records |
| **Fleet Management** | [+] its not required to configure a fleet | [=] requires that each cluster is registered on the same fleet |
| **Namespace Management** | [+] no need to sync namespaces on each cluster | [-] as it is fleet managed, `namespace sameness` is enforced, the operator does not automatically create the namespace if it is not yet present |

# Conclusion

Overall, mcs makes more sense for managing cluster-to-cluster communication on multiple gke clusters as gcloud designed it for this purpose. On the other hand, external-dns is a tool that is built for managing dns records from cluster resource's annotations (ingress/service). It is also possible to enable them both. Use mcs for cluster-to-cluster networking, and external-dns for managing dns records.