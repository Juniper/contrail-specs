**Kubernetes and Openshift Support**
=====================================

Version 0.1


**Objectives**
==============

This documents captures the Kubernetes and Openshift features that will be
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
  TLS bootstrapping streamlines Kubernetes’ ability to add and remove nodes to the cluster.
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

On the networking and Networking policy aspects, following are the additions to networking
in Kubernetes 1.12.

- **Egress support for Network Policy**

  PR: https://github.com/kubernetes/features/issues/366

  Egress specification in Kubernetes network policy spec has been upgraded to GA.
  The egress spec may include podSelector, namespaceSelector, pod&namespaceSelector and egress CIDR.

  Contrail has support for the following as of 5.0:

       * podSelector

       * namespaceSelector

       * egress CIDR

  Support needs to be added in 5.1 for:

       * podSelector & namespaceSelector


**CIDR support in Network Policy**
-----------------------------------

  PR: https://github.com/kubernetes/features/issues/367

  CIDR selector support for egress and ingress network policies has been upgraded to GA.

  Contrail has support for egress and ingress CIDR as of 5.0.

**SCTP support for Services, Pod, Endpoint, and NetworkPolicy**
----------------------------------------------------------------

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


