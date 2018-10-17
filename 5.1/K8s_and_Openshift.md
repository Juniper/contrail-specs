**Kubernetes and Openshift Support**
=====================================

Version 0.1


**Objectives**
==============

This documents captures the additional Kubernetes and Openshift features that will be
supported in Contrail Release 5.1.

The general objective is to be able to support and be compliant with:

  **Kubernetes Release 1.12**

  **Openshift Release 3.11**

**Kubernetes 1.12**
====================
Kubernetes 1.12 brings in incremental feature enhancements and bug fixes,
with emphasis on trust and stability.

Some major features in this release are:

- TLS Bootstrapping of K8s Nodes:

  TLS bootstrapping streamlines Kubernetes ability to add and remove nodes to the cluster.
  This feature officially graduates to general availability in 1.12.

- Priority Based Multitenancy

  This release adds the ability to support priority on the various resource quotas via the
  new ResourceQuotaScopeSelector feature.

- Improved Autoscaling

  Improved support for pod auto scaling by creating/deleting pods based on load.

- Improved Network Policy specification.

**Contrail support for Kubernetes 1.12**
----------------------------------------

Contrail implements networking for Kubernetes and is required to be in compliance with the
networking and network policy requirements of this release.

As of Contrail 5.0.2, Contrail supports Kubernetes 1.9.2.

In Contrail 5.1, the intent is to update support to Kubernetes 1.11.

***Support Operating System***
----------------------------

**Centos 7.5**

***Networking***
----------------
On the networking and Networking policy aspects, following are the additions to networking
in Kubernetes 1.12.

- ***Egress support for Network Policy***
  ---------------------------------------

  PR: https://github.com/kubernetes/features/issues/366

  Egress specification in Kubernetes network policy spec has been upgraded to GA.
  The egress spec may include podSelector, namespaceSelector, pod&namespaceSelector and egress CIDR.
  
  Contrail has support for the following as of 5.0:

       * podSelector

       * namespaceSelector

       * egress CIDR

  Support needs to be added in 5.1 for:

       * podSelector & namespaceSelector


- ***CIDR support in Network Policy***
  ------------------------------------

  PR: https://github.com/kubernetes/features/issues/367

  CIDR selector support for egress and ingress network policies has been upgraded to GA.

  Contrail has support for egress and ingress CIDR as of 5.0.

- ***Provisioning***
  ------------------

  - Contrail-ansible-deployer

    Contrail ansible deployer will be updated to support Kubernetes 1.11.
    Corresponding docker version updates(if any) will also need to be added to ansible deployer.

  - Single Yaml Install

    Changes will be required to add NotReady tolerations on all pods, so Contrail pods can be run even when node
    is tainted as NotReady. This will need to be done for standalone, nested and non-nested yaml file.

- ***SCTP support for Services, Pod, Endpoint, and NetworkPolicy (Not evaluated yet)***
  -------------------------------------------------------------------------------------

  PR: https://github.com/kubernetes/kubernetes/pull/64973

  Kubernetes 1.12 add support for SCTP as additional protocol alongside TCP and UDP in Pod,
  Service, Endpoint, and NetworkPolicy. This is released as a Alpha feature in this 1.12.

  This feature depends on support for SCTP protocol in Contrail as a whole, which does not exist.
  Hence this is not targeted for Contrail Release 5.1.


**References**
---------------

[Kubernets 1.12 Feature List](https://docs.google.com/spreadsheets/d/177LIKnO3yUmE0ryIg9OBek54Y-abw8OE8pq-9QgnGM4/edit#gid=0)

[Egress Support PR](https://github.com/kubernetes/features/issues/366)

[CIDR Support in Network Policy PR](https://github.com/kubernetes/features/issues/367)

[SCTP support PR](https://github.com/kubernetes/kubernetes/pull/64973)

**OpenShift Container Platform 3.11**
=====================================

Red Hat OpenShift Container Platform 3.11 is based on Kubernetes 1.11.
It is supported on Red Hat Enterprise Linux 7.4 and 7.5.

This will be the last release in 3.x stream by RedHat.

Some major features in this release are:

- HA NAMESPACE-WIDE EGRESS IP

  Adding basic active/backup HA for project/namespace egress IPs now allows a namespace to
  have multiple egress IPs hosted on different cluster nodes.

- POD PRIORITY AND PREEMPTION

  Pod priority indicates the importance of a pod relative to other pods and queues the pods
  based on that priority. Pod preemption allows the cluster to evict, or preempt, lower-priority
  pods so that higher-priority pods can be scheduled if there is no available space on a suitable node.

- CLUSTER ADMINISTRATOR CONSOLE

  OpenShift Container Platform 3.11 introduces a cluster administrator console tailored toward application
  development and cluster administrator personas.

- CONTROL SHARING THE PID NAMESPACE BETWEEN CONTAINERS (TECHNOLOGY PREVIEW)

**Contrail support for OCP 3.11**
----------------------------------------

As of Contrail 5.0.2, Contrail supports OCP 3.9. In Contrail 5.1, the intent is to upgrade to OCP 3.11.

i.e Contrail has to catch up with OCP 3.10 and OCP 3.11.

***Support Operating System***
----------------------------

**RHEL 7.5**

***Networking***
----------------
On the networking and Networking policy aspects, following features will need to be added to Contrail 5.1:

- ***Expanding Service Network***
  -----------------------------

  https://docs.openshift.com/container-platform/3.10/install_config/configuring_sdn.html#expanding-the-service-network

  This features allows for growing the service network address range to a larger address space.

  This does not cover migration to a different range, just the increase of an existing range.

- ***Namespace-wide Egress IP (Automatic and HA) (Not evaluated yet)***
  ------------------------------------------------------------------

  [Automatic Namespace Egress IP](https://docs.openshift.com/container-platform/3.11/release_notes/ocp_3_11_release_notes.html#ocp-311-fully-automatic-namespace-wide-egress-ip)

  [HA Namespace Egress IP](https://docs.openshift.com/container-platform/3.11/release_notes/ocp_3_11_release_notes.html#ocp-311-ha-namespace-wide-egress-ip)

  Projects/namespaces are automatically allocated a single egress IP on a node in the cluster,
  and that IP is automatically migrated from a failed node to a healthy node.

  This will potentially need involved changes in vroute agent, kube-manager, CNI etc.
  Have not been evaluated/vetted/discussd enough.

- ***Provisioning***
  ------------------

  - Ansible version should be upgraded to 2.6

    https://docs.openshift.com/container-platform/3.11/release_notes/ocp_3_11_release_notes.html#ocp-311-support-for-ansible-2-6

  - Registry auth credentials are not required.

    https://docs.openshift.com/container-platform/3.11/release_notes/ocp_3_11_release_notes.html#ocp-311-registry-auth-credentials-required

  - Ansible logs are enabled by default

    https://docs.openshift.com/container-platform/3.11/release_notes/ocp_3_11_release_notes.html#ocp-311-customer-installations-are-logged

  - Increased cluster scaling limits

    https://docs.openshift.com/container-platform/3.11/scaling_performance/cluster_limits.html#scaling-performance-cluster-limits

**References**
--------------

[OCP 3.10 release notes](https://docs.openshift.com/container-platform/3.10/release_notes/ocp_3_10_release_notes.html)

[OCP 3.11 release notes](https://docs.openshift.com/container-platform/3.11/release_notes/ocp_3_11_release_notes.html)


