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
<table>
<tr>
<th>
Security Rule
</th>
<th>
Network Policy
</th>
<th>
Comments
</th>
</tr>
<tr>

<td>
<pre>
--protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0
</pre>
</td>

<td>
<pre>
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
</pre>
</td>
<td/>
</tr>
<tr>
<td>
Allows SSH traffic from any IP to the instance
<pre>
--protocol tcp --dst-port 22:22 --remote-group SOURCE_GROUP_NAME
</pre>
We assign the security group name as label to the VM and we will use pod selector to allow traffic from the VMs that has the security group name as label
</td>
<td>
<pre>
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
   name: test-network-policy
namespace: default
spec:
   podSelector:
      matchLabels:
        security_group: SECURITY_GROUP_NAME

   policyTypes:
          - Ingress
   ingress:
     - from:
        ports:
           - protocol: TCP
             port: 22
             endPort: 22
</pre>
</td>
<td>  
When a VM is created, it is assigned a label security_group: SECURITY_GROUP_NAME.

This label is later used to allow network traffic only from those specified VMs which can be used to implement remote security group based rules.
</td>
</tr>
<tr>
<td>
<pre>
  --protocol icmp
</pre>
  </td>
  <td>
    <pre>
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
name: allow-ping-in-cluster
spec:
selector: all()
types:

Ingress
ingress:

action: Allow
protocol: ICMP
source:
selector: all()
icmp:
type: 8 # Ping request

action: Allow
protocol: ICMPv6
source:
selector: all()
icmp:
type: 128 # Ping request
</pre></td><td>
Allow ping from any host. Basic k8s network policy does not support ICMP based protocol. Calico supports ICMP based network policy
  </td></tr><tr><td><pre>
  --protocol icmp \
  --remote-group SOURCE_GROUP_NAME
  </pre><td><pre>
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
name: allow-host-unreachable
spec:
selector: security_group=SOURCE_GROUP_NAME
types:

Ingress
ingress:

action: Allow
protocol: ICMP
icmp:
type: 3 # Destination unreachable
code: 1 # Host unreachable
</pre></td><td>
Allow ping from only from the remote group, which includes set of ports that the remote group is assigned to
</td></tr>
  <tr><td><pre>
    --protocol udp 
  --dst-port 53:53
  </pre></td><td>
    <pre>
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
    - protocol: UDP
      port: 53
      endPort: 53
  policyTypes:
  - Ingress
  </pre></td><td/></tr><tr><td><pre>
  --protocol udp \
  --dst-port 53:53 --remote-group SOURCE_GROUP_NAME SECURITY_GROUP
  </pre></td><td><pre>
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
   name: test-network-policy
   namespace: default
spec:
   podSelector:
      matchLabels:
         security_group: SECURITY_GROUP_NAME
policyTypes:
   - Ingress
   ingress:
   - from:
   ports:
       - protocol: UDP
         port: 53
         endPort: 53
</pre></td><td/></tr>
</table>
