# ostokubevirt
Scripts to migrate virtual machines from openstack to kubevirt
## Introduction
Virtualization has been the de facto choice for building business applications for the last two decades. Though most companies are transitioning to microservices-based architectures, virtualization-based workloads continue to exist for the foreseeable future. It has become challenging for companies to manage two management planes, one for virtualization and another for microservices. KubeVirt is a CNCF project that aims to alleviate this problem by letting virtual machines run on Kubernetes so both container-based workloads and virtual machines can be deployed and managed using one platform.

One challenge for businesses is migrating existing virtual machines running on various platforms to KubeVirt. OpenStack is one of the virtualization platforms that businesses want to migrate their workloads to KubeVirt. Fortunately, both OpenStack and KubeVirt are based on libvirt, and the virtualization concepts map relatively straightforward and hence easy to implement the migration path.

This document discusses the migration procedure from OpenStack to KubeVirt. We first map all the possible OpenStack instance attributes to KubeVirt VirtualMachine specifications. We then map OpenStack security group rules to network policies. 

## Mapping OpenStack Concepts to Kubernetes
This section defines the mapping between OpenStack concepts such as instances, flavors, images, security groups, cinder volumes, floating IP addresses, domains, tenants and roles to Kubernetes constructs

### Mapping from OpenStack instances to KubeVirt VM Spec
This section defines the mapping between OpenStack instances flavor and image attributes to KubeVirt VirtualMachine specifications.
| OpenStack Instance Attribute | KubeVirt Spec Path | Comments |
| ---------------------------- | ------------------ | -------- |
| image:--property hw_cpu_sockets | spec.domain.cpu.dedicatedplacement | |
| spec.domain.cpu.numa.guestMappingPassthrough | spec.domain.cpu.sockets |
| image:--property hw_cpu_cores | spec.domain.cpu.dedicatedplacement | |
| spec.domain.cpu.numa.guestMappingPassthrough | spec.domain.cpu.cores |
| image:--property hw_cpu_max_sockets | spec.domain.cpu.dedicatedplacement | |
| spec.domain.cpu.numa.guestMappingPassthrough | spec.domain.cpu.cores | |
| image:--property hw_cpu_max_threads | spec.domain.cpu.dedicatedplacement | |
| spec.domain.cpu.numa.guestMappingPassthrough | spec.domain.cpu.threads | |
| flavor:vcpus | spec.domain.cpu.cores or spec.domain.requests.resources.cpu | |
| flavor:Swap | KubeVirt does not support | |
| flavor:root disk GB | spec.domain.devices.disks.containerdisk | |
| spec.volumes.containerdisk.image | flavor:memory | |
| spec.domain.requests.resources.memory | flavor:Extra Specs | Extra specs determine the scheduling policy where the instance can be scheduled. On k8s, they can translate into node selector in the VM spec |
| flavor:Ephemeral Disk GB | spec.domain.devices.disks.emptydisk | |
| spec.volumes.emptydisk.image | flavor:--property hw:numa_mem.1 |  KubeVirt does not support this option |
| flavor:--property hw:numa_nodes | | KubeVirt does not support this option |
| flavor:--property hw:numa_cpus.0 | | KubeVirt does not support this option |
| flavor:--property hw:cpu_policy | | KubeVirt does not support this option |
| flavor:--property hw:cpu_thread_policy | | KubeVirt does not support this option |
| flavor:--property hw:cpu_sockets | spec.domain.cpu.dedicatedplacement | |
| spec.domain.cpu.numa.guestMappingPassthrough | spec.domain.cpu.sockets | |
| flavor:--property hw:cpu_cores | spec.domain.cpu.dedicatedplacement | |
| spec.domain.cpu.numa.guestMappingPassthrough | spec.domain.cpu.cores | |
| flavor:--property hw:cpu_threads | spec.domain.cpu.dedicatedplacement | |
| spec.domain.cpu.numa.guestMappingPassthrough | spec.domain.cpu.threads | |
| flavor:--property=hw:cpu_max_sockets |  | KubeVirt does not support this option |

### Mapping Instance Security Group Rules to Network Policy
| Security Rule | Network Policy | Comments |
| ------------- | -------------- | -------- |
| --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0 |
```yaml

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/24
    ports:
    - protocol: TCP
      port: 22
      endPort: 22
  policyTypes:
  - Ingress
```
