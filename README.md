# ostokubevirt
Scripts to migrate virtual machines from openstack to kubevirt
## Introduction
Virtualization has been the de facto choice for building business applications for the last two decades. Though most companies are transitioning to microservices-based architectures, virtualization-based workloads continue to exist for the foreseeable future. It has become challenging for companies to manage two management planes, one for virtualization and another for microservices. KubeVirt is a CNCF project that aims to alleviate this problem by letting virtual machines run on Kubernetes so both container-based workloads and virtual machines can be deployed and managed using one platform.

One challenge for businesses is migrating existing virtual machines running on various platforms to KubeVirt. OpenStack is one of the virtualization platforms that businesses want to migrate their workloads to KubeVirt. Fortunately, both OpenStack and KubeVirt are based on libvirt, and the virtualization concepts map relatively straightforward and hence easy to implement the migration path.

This document discusses the migration procedure from OpenStack to KubeVirt. We first map all the possible OpenStack instance attributes to KubeVirt VirtualMachine specifications. We then map OpenStack security group rules to network policies. 
