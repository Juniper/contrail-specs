**Multiple Interface Networking for Containers**
================================================

Version 0.1

**Objectives**
==============

This document proposes a framework to provide "multiple interfaces
networking" support for Linux containers, where networking is managed by
Contrail SDN. The goal is to layout models and principles that should be
generic enough for all Linux container orchestration platforms.

Multiple networking interfaces support (hereafter referred to as
multi-net) for Linux containers is on full steam in the wild, the
initial and significant charge being led by Kubernetes container
orchestration platform. So, this document relies heavily on the
principles of multi-net support in Kubernetes for reference. But the
design principles are aimed to be generic and independent enough to be
applicable to other orchestration solutions like Openshift Container
Platform, Mesosphere, Docker swam etc. This design also takes into
consideration interworking and compliance with existing Kubernetes CNI
meta plugins like Multus, CNI Genie etc.

**Use cases**
=============

-   Use cases include but not limited to the following:

-   Service Providers tend to keep the management and tenant networks
    independent for isolation, ease of deployment and management,
    programmability etc

-   A container need not necessarily need to be connected to a single
    network. Also, a container need not necessarily only talk to other
    containers in the same networks. Multiple interfaces provide a way
    for containers to be connected to multiple devices in multiple
    networks simultaneously.

-   Provides redundant networking paths for low latency interfaces.

-   Provides redundant networking planes for a container. A container
    may be serviced by multiple SDN's via independent network
    interfaces.

-   Service chaining / VNF scenarios where a single container needs to
    connect to multiple networks.

-   For VPN termination, where one of the interfaces to the container
    acts as a tunnel endpoint, acting as a gateway to/from the
    cluster/network that it is connected to via another interface.

**Glossary**
============

**Container**

[Operating-system-level virtualization](https://en.wikipedia.org/wiki/Operating-system-level_virtualization)
method for running multiple isolated systems (containers) on a control host using a single Linux kernel. Source:
[Wikipedia](https://en.wikipedia.org/wiki/LXC))

**Network**

IP Layer3 network.

**Network Namespace**

Refers to a network stack in the Linux kernel. Linux kernel supports
creation of multiple,independent networking stack that are referred to
as network namespaces. One or more linux containers can share a network
namespace.

**Network Interface/s of a Container**

One or more network interfaces (physical/virtual) attached to the
container's Linuxnetwork namespace.

**Network Attachment**

Synonymous to network interface in a container. There can be multiple
network attachments on a container, connected to the same or disparate
networks.

**Network Plugin or CNI Plugin**

SDN plugin that creates and configures network interfaces in a Linux
network namespace. The plugin interfaces with the control plane to
receive the configuration for a network namespace, creates required
network attachments and configures thenetwork attachments.

CNI stands for Container Network Interface

**Container Orchestrator**

Platform that manages the lifecycle of workloads. The focus of this spec
is containerizedworkloads. Kubernetes, Openshift Container Platform,
Docker Swarm etc are some prominent implementations.

**Cluster**

A deployed and operational instance of a Container Orchestrator.

**Non-Goals**
=============

Service Chaining design and implementation in Contrail SDN is not described/covered by this spec.

**Kubernetes CNI Multi-Net Spec explained**
===========================================

Contrail SDN's multi-net design and implementation leans heavily on
principles and modeling of [Kubernetes multi-net specification](https://docs.google.com/document/d/1Ny03h6IDVy_e_vmElOqR7UdTPAG_RNydhVE1Kx54kFQ/edit).
To its credit, though this specification is aimed at a kubernetes
specific design and consideration, core constructs of this spec are
fairly generic and easily extensible to non-kubernetes models. Hence it
is an attractive template for Contrail's multi-net design as well.

While we recommend readers to consume [Kubernetes multi-net
spec](https://docs.google.com/document/d/1Ny03h6IDVy_e_vmElOqR7UdTPAG_RNydhVE1Kx54kFQ/edit)
directly and in entirety, here we will lay out some of its core
constructs:

*Changes to the Kubernetes API*

The specification explicitly avoids changes to the existing Kubernetes
API. Those changes require a much longer upstream process, but the hope
is this spec can serve as a basis for those changes in the future by
proving the concepts and providing one or more real-world
implementations of multiple pod network attachments.

*Interaction with the Kubernetes API*

The interaction of additional pod network attachments with the
Kubernetes API and its objects, such as Services, Endpoints, proxies,
etc is not specified. SIG Network discussed this topic at length in
July/August 2017 and decided it would require changes to the Kubernetes
API, which is an explicit non-goal of this specification.

*Changes to the Kubernetes CNI Driver*

To ensure easy use of this specification and implementations of it, no
changes will be required of the Kubernetes CNI driver (eg
pkg/kubelet/dockershim/network/cni/\*). Changes to the driver to support
multiple pod network attachments have already been proposed and multiple
proof-of-concepts written, but there is as yet no upstream consensus on
how multiple attachments should work and thus what changes would be
required in the CNI driver.

*Replacement of the Kubernetes Cluster-Wide Default Network*

To preserve compatibility and expectations with existing Kubernetes
deployments, this specification does not attempt to change any of the
existing Kubernetes cluster-wide network behavior. All pods must still
be attached to the cluster-wide default network as they are today.

**Contrail Multi-Net Support**
==============================

As is, Contrail SDN does have networking support for containers.
However, the networking support is limited to single network interface
per container. The idea behind current container networking has been
that containers need networking. Networking needs an interface. So,
every container at the time of creation is allocated one interface for
its networking. Simple enough.

This semantics implies a notion that containers just care about
networking and do not really care about the networks they are connected
to. The notion of networks was thus offshored to the SDN implementation
of the cluster and cluster itself remained network agnostic.

This simplified notion falls apart in a multi interfaces world, as the
very essence of the need for multiple interfaces is to be able to
connect containers to multiple networks. Containers thus need to specify
what networks they would like to be connected to. This makes "network" a
first-class object in the cluster, meaning, the cluster is now "network
aware".

**Object Model**
================

**Network Object Model**
------------------------

The object model of the container orchestration platform should have the
semantics to represent a network and ability to associate the network/s
to the containers. If the data model cannot natively support network
objects, Contrail will depend on custom extensions either in the data
model or outside the data model to represent network.

### **Clusters' Network Object Model**

In this case the cluster data model has a network template that Contrail
can read and understand. Contrail will go ahead and watch for existence
of the object model and understand the template.

**Kubernetes Illustration**

Kubernetes supports a custom extension to represent networks in its
object model, through its CustomResourceDefinition(CRD) feature. This
extension adds support for a new kind of object called
NetworkAttachmentDefinition, which represents a network in Kubernetes
data model.
```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: network-attachment-definitions.k8s.cni.cncf.io
spec:
  group: k8s.cni.cncf.io
  version: v1
  scope: Namespaced
  names:
 	plural: network-attachment-definitions
 	singular: network-attachment-definition
 	kind: NetworkAttachmentDefinition
 	shortNames:
 	- net-attach-def
  validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            config:
             type: string
```
####

### **Contrail Extensions to Cluster Data Model**

If the cluster does not natively support Network representation, then
Contrail will create such a template in the cluster data model at
startup. This spec does not dictate the location where this data model
is created and referenced, as it is highly cluster and deployment
specific. But the general guidance and best practice is to use cluster
infrastructure/data store to create, maintain and reference this data
model.

**Kubernetes Illustration**

In the case of Kubernetes, at the time of startup, Contrail Kube Manager
will query kubernetes API server for the presence of
network-attachment-definitions.k8s.cni.cncf.io custom resource
definition. In the absence of this CRD, Kube Manager will create this
CRD in Kubernetes object model.

### **Contrail Native Network Data Model**

Contrail SDN already has a representation of network, natively. Network
objects created in Contrail can by themselves be used to represent
networks, independent of the cluster data model. The cluster
orchestrator should then provide the provision to pass Contrail network
representation, as a part of container create notification, so Contrail
can identify network config.
```
Contrail Network Identifier:

{
   	domain: <domain-name>,
   	project: <project-name>,
   	name:<network-name>
}

```

**Network Configuration**
=========================

A network object creates a network config in the cluster. The network
object should at a minimum specify the following attributes for a
network:

1.  Network Name

2.  Network Scope

3.  Network CIDR

4.  Optionally other relevant network attributes like IP version, default GW etc.

Networks can be created in the cluster by:

1.  Creating a network natively in Contrail.

2.  Creating a network object in cluster.

### **Creating a network natively in Contrail.**

Networks can be created in Contrail through its API server and/or GUI.
Contrail network semantics and support are well understood. So, there is
no need to elaborate.

**Creating a network in the cluster**

Networks can be created in the cluster through native cluster objects,
by interfacing with the cluster api server.

**Kubernetes illustration**

```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: <network-name>
  namespace: <namespace-name>
  annotations:
    "opencontrail.org/cidr" : [<ip-subnet>]
    "opencontrail.org/ip_fabric_snat" : <True/False>
    "opencontrail.org/ip_fabric_forwarding" : <True/False>
spec:
  config: '{
    “cniVersion”: “0.3.0”,
    "type": "contrail-k8s-cni"
}'

name: name of the network
namespace: namespace that the network object belongs to.
opencontrail.org/cidr: CIDR of the network
opencontrail.org/ip_fabric_snat: Enable/Disable fabric SNAT
opencontrail.org/ip_fabric_forwarding: Flag for ip fabric forwarding

```

**Container Creation**
======================

Cluster orchestrator exposes API endpoints to create/update/delete
containers. Container object definition should be enhanced to specify
the list of networks the container should be attached to.

**Kubernetes Illustration**

Kubernetes manages containers in collections called Pods. A Pod is a
collection of one or more containers that share a network namespace. Pod
API is enhanced to accept an annotation to specify a list of networks,
representing interfaces to be created in the Pod's network namespace.
```
kind: Pod
metadata:
  name: my-pod
  namespace: my-namespace
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
      { "name": "net-a" },
      { "name": "net-b" },
      { "name": "other-ns/net-c" }
    ]'
spec:
  containers:

```

Kubernetes provides the flexibility where pods in a namespace can refer
to the networks created on other namespaces using their fully scoped
name. In the above illustration, net-a and net-b belong to the same
namespace are pod (i.e my-namespace). net-c belongs to the namespace
"other-ns" and hence its name needs to be fully qualified.

**Workflow / Illustration**
===========================

### **1. Create Network Object Model (if needed)**

If the cluster does not have native support for network object model,
create the model.

**Kubernetes Illustration:**

Create NetworkAttachmentDefinition CRD object

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: network-attachment-definitions.k8s.cni.cncf.io
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: k8s.cni.cncf.io
  # version name to use for REST API: /apis/<group>/<version>
  version: v1
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: network-attachment-definitions
    # singular name to be used as an alias on the CLI and for display
    singular: network-attachment-definition
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: NetworkAttachmentDefinition
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - net-attach-def
  validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            config:
              type: string


```

### **2. Create Networks**

Create networks in the cluster.

**Kubernetes Illustration: Creating Network in Cluster**
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: network-a
  annotations:
    "opencontrail.org/cidr" : ['10.1.1.0/24']
    "opencontrail.org/ip_fabric_snat" : False
    "opencontrail.org/ip_fabric_forwarding" : True
spec:
  config: '{
    “cniVersion”: “0.3.0”,
    "type": "contrail-k8s-cni"
}'

```

### **OR**

**Kubernetes Illustration: Reference to existing Contrail Network**
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: extns-network
  annotations:
    "opencontrail.org/network" : '{"domain":"default-domain", "project": "k8s-extns", "name":"k8s-extns-pod-network"}'
spec:
  config: '{
    “cniVersion”: “0.3.1”,
    "type": "contrail-k8s-cni"
}'

```

### **3. Create Workloads**

Create workloads that reference the networks.

The workloads will be created with one interface for each referenced
network.

**Kubernetes Illustration:**

Create Pod referencing the networks.
```
apiVersion: v1
kind: Pod
metadata:
  name: multiNetworkPod
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
      { "name": "network-a" },
      { "name": "network-b" }
    ]'
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
    stdin: true
    tty: true
  restartPolicy: Always

```

**Contrail Implementation for Kubernetes**
==========================================

Contrail SDN solution for Kubernetes consists of the following
agents/plugins:

Contrail Kube Manager

A config component that interface with Kubernetes API server. This
component converts the config objects of interest to us from
Kubernetes and creates corresponding Contrail config objects through
the Contrail API server.

Contrail CNI

A control plane plugin that is responsible to create and configure
network interfaces for a Pod on the host, where the Pod is scheduled
to run. If there are multiple interfaces needed for a Pod, this plugin
will be responsible to create and configure all interfaces.

Contrail Vrouter Agent

A config/data plane component that holds network and interface
configuration required for a Pod.

### **Contrail Kube Manager (KM)**

Contrail Kube Manager (hereafter referred to as KM) will be enhanced to
add multiple Virtual Machine Interface objects, one for each network
referenced on the Pod. It will exhibit the following behavior w.r.t
multi-interface support:

1.  Be default, it will always create one VMI for a Pod in the cluster
    pod subnet of kubernetes. KM should create this VMI regardless of
    the presence/absence of network references in the Pod spec.

2.  In order to add multiple network support, the network object CRD
    should be created in Kubernetes API server. The CRD defines the
    template/blueprint for a network object. If it's not created, then
    Kubernetes API server will not understand the network objects. KM
    on bootup, will validate if network CRD is found in the Kubernetes
    API server. If it is not found, then KM will create a network CRD
    itself.

3.  Once network CRD is created, the kubernetes apiserver is ready to
    accept CRUD of network objects. KM will issue a watch request for
    network objects, so it can be notified of all network object CRUD.
    If KM restarts, it should ensure that network objects are
    synchronized first before pods are synchronized.

4.  Network association with workloads:\
    Pod creation request can contain one or more references to
    networks to indicate the number of interfaces required for the
    Pod. KM will create one VMI for each such network reference.

The following heuristics will be applied while determining the networks for a pod:

a.  If there are no explicit network references on the Pod, no new VMI's
    will be created, other than the one for cluster pod network.

b.  If there are networks referenced in the Pod spec, one VMI will be
    created for each referenced network.

c.  There can be multiple references to the same network, in which case,
    as many VMI's will be created.

d.  The VMI's objects in config represent interfaces on container. When
    it comes to interfaces on a container, ordering of the interfaces
    is critical as this translates to interface name etc. KM will
    interpret the order of the network references on a Pod to signify
    the interface order. To represent this ordering in config, KM will
    attach an annotation in the VMI objects of a pod to records its
    position in the sequence i.e ordering. Other modules consuming
    these VMI's are expected to rely on this annotation to infer
    interface order.

    If the references a Pod are: net-a, net-b annotations:

```
k8s.v1.cni.cncf.io/networks: net-a,net-b
```

There will be 3 VMI's on the Pod.

```
First VMI : VMI at index '0' for kubernetes pod network association.

Second VMI : VMI at index '1' for net-a

Third VMI : VMI at index '2' for net-b
```

e.  KM should add network name as an annotation on the VMI. This is
    helpful for correlation purpose in the case of presence of
    meta-plugin. More on meta-plugin later in this spec.

f.  When multiple interfaces are created on the Pod, then default route
    should/could point to any of the interfaces. In the absence of
    explicit config, the default route in the Pod should point to the
    first interface. Potentially we could expose and support an
    annotation on the Pod that specifies interface for default route.

<!-- -->

5.  KM will not support any runtime add/mod/delete of individual
    networks on the Pod, post Pod creation. If any change in Pod
    network membership is desired, the Pod has to be deleted and
    recreated.

6.  When a Pod is deleted, KM should delete all VMI config objects
    allocated for the Pod. In addition, all IP's allocated for the Pod
    should be deallocated as well.

### **Contrail VRouter Agent**

VRouter agent is the control/data component of the data plane. There is
one agent instance running on each compute node of the cluster. It peers
with Contrail controller running on the control nodes to download config
relevant to the compute node. When a Pod is created, the corresponding
VM and VMI objects for the Pod are pushed to the VRouter agent. If the
Pod has multiple interfaces, the VRouter agent will have the required
config to reflect the association.

The VRrouter agent exposes an API endpoint, to query configuration for a
Pod. As a part this query it returns the VMI associated with the Pod.
For multi interface support, this API should be extended to support the
following:

1.  Return a list of all VMI's associated with a Pod.

2.  Return all annotations that are attached on each VMI.

### **Contrail CNI**

Kubernetes kubelet service invokes Contrail CNI when a Pod is added or
deleted.

When invoked for Pod add, the CNI is responsible for creating and
configuring the Pod's network namespace. The CNI is responsible to
remove the interfaces from a Pod's network namespace when it is invoked
for Pod delete.

Contrail CNI is a light-weight Golang based plugin, that is expected to
function in a plug-n-play system. It can operate in two modes:

**Meta-plugin / Thick Plugin / Standalone Plugin**

In this mode, Contrail CNI is responsible for setting up networking for
all interfaces of the Pod. The kubelet invokes the Contrail CNI
directly.

Kubelet is not aware of the presence of multiple interfaces on a Pod.
Hence the ADD/DEL to the Contrail CNI will be issued for addition of an
interface for cluster wide default network.

Contrail CNI has the following responsibilities in this scenario:

1.  Determine the number of interfaces to be attached on the Pod. This
    is determined by querying the VRouter agent. VRouter agent will
    return a list of VMI's that are associated with the Pod. The
    number of interfaces in the list determines the number of
    interfaces to be attached on the Pod.

2.  CNI will depend on the VMI configuration read from VRouter agent to
    determine interface configuration.

3.  CNI will sort the list of VMI's using the "index" key stored in the
    annotations of the VMI. This index determines the order of the
    interfaces inside in the Pod.

4.  The name to be used for the interfaces should be constructed by CNI.
    The naming could be in the form of "ethx" where x is the index.
    However this is left to the discretion of the implementation.

5.  CNI should look for "default-gw" annotation on all VMI's to
    determine the interface/network that the default route should
    point to. If no such annotation is found, CNI should assume that
    first interface/network for default route.

6.  The interfaces can be IPv4 or IPv6. CNI should support configuration
    of both.

7.  If creation, configuration of any of the interfaces fails, CNI
    should return failure to kubelet.

8.  The result of successful create/failure should only include the
    IP/config related to the first interface on the Pod. Kubelet is
    only aware of the first interface and so only expects reply for
    the first interface.

9.  When CNI receives a DEL request for a Pod, it should delete all
    interfaces for the Pod. Will CNI depend on VRouter agent or local
    file system to determine all interfaces for the Pod ?

**Plugin mode**

The plugin, when invoked, is responsible to setup networking for a
single interface of a Pod. If the pod has multiple interfaces, then we
need an external plugin (officially referred to as CNI Delegating
Plugin) that will iterate through all interfaces of the Pod and invoke
the CNI plugin once for each interface. Multus, CNI Genie are examples
of such CNI Delegating Plugins. In this model, it is possible that CNI
Delegating plugin works with Contrail and non-Contrail plugins at the
same time. This opens up an interesting case where Contrail plugin sets
up networking for subset of interfaces of a Pod and non-contrail plugins
setup networking for others.

Contrail CNI when invoked will be given the following information to
identify the interface of a Pod for which it is being invoked for Add or
Delete.

**\[container ID, network name, CNI\_IFNAME\]**

Container ID:

Id of the pod instances on the host it is running.

Network Name:

The name of the network object that the Pod interface is being added
to. This name is the name of the network's NetworkAttachmentDefinition
object.

CNI_IFNAME:

Generated by the CNI Delegating Plugin for a given attachment and must
be unique across all interfaces of the Pod.

Contrail CNI has the following responsibilities in this scenario:

1.  It will be invoked for one interface or multiple interfaces. In the
    case of multiple interfaces, CNI will be invoked once for each of
    those interfaces. Each invocation should result in
    creation/mutation of the interface for with it was invoked.

2.  Contrail CNI should use the CNI\_IFNAME as the name of the
    configuring interface.

3.  CNI should use the network name passed by the meta-plugin to
    identity the VMI associated with the interface.

4.  If there are multiple interfaces for the same network, CNI will be
    invoked twice with the same network name. In this case, CNI should
    be able to determine the interface for which it is being invoked.
    This can be based on comparisons against its local state and
    VRouter agent state.

5.  Result of the ADD/DEL request should reflect the status of ADD/DEL
    for the interface for which CNI was invoked. ADD/DELETE action of
    one interface should not affect the state of any other interface.

**References**

[Kubernetes Multiple Network Standard](https://docs.google.com/document/d/1Ny03h6IDVy_e_vmElOqR7UdTPAG_RNydhVE1Kx54kFQ/edit)

[Multus Meta Plugin Spec](https://github.com/intel/multus-cni)

[Kubernetes CNI Spec](https://github.com/containernetworking/cni/blob/master/SPEC.md)
