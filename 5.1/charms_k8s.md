
Juju charms to deploy Contrail using Kubernetes orchestrator
============================================================

# 1. Introduction

This feature's purpose is to allow deployment of Contrail with Kubernetes using JuJu charms.

# 2. Problem statement

Juju charms for Contrail services currently support the deployment of Contrail with OpenStack.
Additional charms are proposed for the deployment of Contrail with Kubernetes.

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

## 3.4 Nested mode

Contrail supports the provisioning of Kubernetes cluster inside an OpenStack cluster. The contrail-kubernetes-master and the contrail-kubernetes-node charms interface directly with Contrail components that manage the OpenStack cluster. No other Contrail charms are needed in nested mode.

# 4. Implementation

## 4.1. Additional charms

### 4.1.1 Contrail-kubernetes-master

The contrail-kubernetes-master charm installs the contrail-kubernetes-kube-manager docker container.

Proposed definition of charm metadata:

```
requires:
  contrail-controller:
    interface: contrail-controller
  kube-api-endpoint:
    interface: http
    scope: container
provides:
  contrail-kubernetes-config:
    interface: contrail-kubernetes-config
  nrpe-external-master:
    interface: nrpe-external-master
    scope: container
subordinate: true
```

The charm will be defined as a subordinate for deployment on the same node where kubectl is available.

### 4.1.2 Contrail-kubernetes-node

The contrail-kubernetes-node charm installs the contrail-kubernetes-cni-init docker container.

The charm will be defined as a subordinate for deployment on the same nodes as the contrail-agent charms.

Proposed definition of charm metadata:

```
requires:
  cni:
    interface: kubernetes-cni
    scope: container
  contrail-kubernetes-config:
    interface: contrail-kubernetes-config
subordinate: true
```

## 4.2 API schema

### 4.2.1 Contrail-kubernetes-master

- contrail-controller relation will use the existing relation of contrail-controller charm to get the Contrail configuration. In a nested-mode deployment, the Contrail configuration will be provided in the charm config, and the contrail-controller relation is not used
- contrail-kubernetes-config relation will provide Kubernetes pod subnets configuration
- kube-api-endpoint relation will use the existing relation of Kubernetes Master charm to get kubernetes API configuration
- nrpe-external-master relation will provide status monitoring

### 4.2.2 Contrail-kubernetes-node

- contrail-kubernetes-config relation will provide Kubernetes pod subnets configuration
- cni relation will be used for setting up the kubernetes-master and kubernetes-worker charms.

## 4.3 User workflow impact

None

## 4.4 UI Changes

None

# 5. Performance and scaling impact

Since contrail kube-manager token(to communicate to kubernetes-api-server) can not be obtained from any other nodes other than kubernetes-master, kube-manager will be running in kubernetes-master node(s). It might introduce additional latency between contrail kube-manager and contrail config. Investigation has to be done to obtain the token from any node. So that the additional latency can be omitted. It also helps to launch contrail kube-manager anywhere.

## 5.2 Forwarding performance

# 6. Upgrade

None

# 7. Deprecations

None

# 8. Dependencies

None

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact

# 11. References
