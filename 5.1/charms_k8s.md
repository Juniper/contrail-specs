
Juju charms to deploy Contrail using Kubernetes orchestrator
============================================================

= 1. Introduction

A user would like to deploy a Canonical based Kubernetes environment using JuJu charms.
The user will use a charm bundle to deploy Kubernetes, ETCD, and other cluster specific components.
As part of this charm bundle the operator will use a Contrail Charm to deploy the servers/services required for Contrail,
and integrate the Contrail CNI executable into the nodes.

Once the charm bundle has completed the resulting Kubernetes deployment should have all nodes (except the Master)
in a ready state, ready to run workloads

# 2. Problem statement

Juju charms for Contrail services currently support the deployment of Contrail based on OpenStack orchestrator.
Additional charms are proposed for the deployment of Contrail using the Kubernetes orchestrator.

# 3. Proposed solution

## 3.1 Deploying Contrail with Kubernetes will be based on existing Contrail charms:

- contrail-agent
- contrail-analytics
- contrail-analyticsdb
- contrail-controller

## 3.2 Additional charms must be implemented:

- contrail-kubernetes-master
- contrail-kubernetes-node

## 3.3 The Charmed Distribution Of Kubernetes will be used:
https://jujucharms.com/canonical-kubernetes/

= 4. Implementation

== 4.1. Additional charms

=== 4.1.1 Contrail-kubernetes-master

The contrail-kubernetes-master charm installs the contrail-kubernetes-kube-manager docker container.

Proposed definition of charm metadata:

```
requires:
  contrail-controller:
    interface: contrail-controller
provides:
  nrpe-external-master:
    interface: nrpe-external-master
    scope: container
```

The charm must be deployed on one of the nodes with any available kubectl.

=== 4.1.2 Contrail-kubernetes-node

The contrail-kubernetes-node charm installs the contrail-kubernetes-cni-init docker container.

The charm will be defined as a subordinate for deployment on the same nodes as the contrail-agent charms.

Proposed definition of charm metadata:

```
requires:
  cni:
    interface: kubernetes-cni
    scope: container
subordinate: true
```

== 4.2 API schema

=== 4.2.1 Contrail-kubernetes-master

- contrail-controller relation will use the existing relation of contrail-controller charm to get the Contrail configuration
- nrpe-external-master relation will provide status monitoring

=== 4.2.2 Contrail-kubernetes-node

- cni relation will provide a relation for setting up the kubernetes-master and kubernetes-worker charms.

== 4.3 User workflow impact

None

== 4.4 UI Changes

None

# 5. Performance and scaling impact
## None
#### None

## 5.2 Forwarding performance
#### None

# 6. Upgrade
#### None
#### None

# 7. Deprecations
#### None

# 8. Dependencies
#### None

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact

# 11. References
