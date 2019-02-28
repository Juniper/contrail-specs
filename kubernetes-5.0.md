## 1. Introduction
Kubernetes (K8s) is an open source container management platform. It provides a portable platform across public and private clouds. K8s supports deployment, scaling and auto-healing of applications. More details can be found at: http://kubernetes.io/docs/whatisk8s/

## 2. Problem statement
1. Service isolation via virtual-networks.
2. Ip-fabric-forwarding
3. Ip-fabric-snat.
3. Third-Party Ingress Controllers.
4. Custom Network support for Ingress.
5. K8s Probes and K8s Node-Port Service.
6. kubernetes-1.9 network-policy support.

## 3. Proposed solution
### 3.1.1 Service isolation via virtual-netorks
Till 4.1, service ip is allocated from cluster-network even for isolated namespaces. So, service from one isolated namespaces can reach service from another isolated namespace. Security groups in isolated namespace prevents reachability from other namespaces which also prevents reachablity from outside of the cluster. In order to provide reachablity to external entity, the security group would be changed to allow all which defeats the isolation. To address this, two virtual-networks would be created in the isolated namespaces. One is for pods(pod-network) and another one is for services(service-network). Contrail network-policy would be created between pod-network and service-network for the reachablity between pods and services. Service uses the same service-ipam which will be a flat-subnet like pod-ipam. It is applicable for default namespace as well. Since virtual-networks are isolated by default in contrail, services from one isolated namespace can not reach service from another isolated namespace.

### 3.1.2 Ip-fabric-forwarding
Since pods are in overlay, it can not be reached directly from underlay(ip-fabric network). Either gateway or vrouter is needed to access any pods from underlay. Virtual-networks can be created as part of underlay with contrail ip-fabric-forwarding feature. It does not need any encapsulation/decapsulation. Ip-fabric-forwarding is only applicable for pod-networks. service-networks are always in overlay. If ip-fabric-forwarding is eanbled, pod-networks would be associated to ip-fabric-ipam instead of pod-ipam which is also a flat-subnet.

Ip-fabric-forwarding can be enabled/disabled in global level or namespace level. To enable it in global level, "ip_fabric_forwarding" has to be set "true" in section "[KUBERNETES]" in /etc/contrail/contrail-kubernetes.conf. By default, it is turned off in global level. To enable/disable in namespace level, "ip_fabric_forwarding" has to be set "true"/"false" in namespace annoation like '"opencontrail.org/ip_fabric_forwarding": "true"/"false"'. Once it is turned on, it can not be modified.

Please refer https://github.com/Juniper/contrail-specs/blob/master/gateway-less-forwarding.md for more information.

### 3.1.3 ip-fabric-snat
With contrail ip-fabric-snat, Pods(which are in the overlay) can reach the internet without floating-ips or logical-router. It uses node(compute) ip for source nating to reach the required services. It is only applicable to pod-networks. kube-manager would reserve ports 56000-57023 for TCP and 57024 to 58047 for UDP for source nating in global-config during the initialization.

Ip-fabric-snat can be enabled/disabled in global level or namespace level. To enable global level, "ip_fabric_snat" has to be set "true" in section "[KUBERNETES]" in /etc/contrail/contrail-kubernetes.conf. By default, it is turned off in global level. To enable/disable in namespace level, "ip_fabric_snat" has to be set "true"/"false" in namespace annoation like '"opencontrail.org/ip_fabric_snat": "true"/"false"'. It can be turned on/off at any time. To enable/disable in default-pod-network, default namespace should be used. If the ip_fabric_forwarding is enabled, ip_fabric_snat would be ignored.

Please refer https://github.com/Juniper/contrail-specs/blob/master/distributed-snat.md for more information.

### 3.1.4 Third-Party Ingress Controllers
Multiple Ingress Controllers can co-exist in contrail. If k8s ingress resource does not have "kubernetes.io/ingress.class" in annotations or "kubernetes.io/ingress.class"  is "opencontrail" in annotations, kube-manager will create haproxy loadbalancer. Otherwise it will be ignored and respective ingress controller will handle the ingress resource. Since Contrail does ensure the reachablity between pods and services, any ingress controller can reach the endpoints(pods) directly or through services.

### 3.1.5 Custom Network support for Ingress
Contrail supports custom-network in namespace level. It is supported for pods till 4.1. It is extended to ingress resources as well in 5.0.

### 3.1.6 K8s Probes and K8s Service Node-Port
Kubelet needs reachablity to pods for liveness and readiness probe. Contrail network policy will be created between ip-fabric network and pod-network to provide the reachablity between node and pods. Whenever the pod-network is created the network-policy is attached to pod-network which provides reachablity between node and pods. So any process in the node can reach the pods.

K8s Service Node-Port is fully based on the node reachablity to pods. Since contrail can provide the connectivity between node and pods through contrail network-policy, Node Port can be supported.

### 3.1.7 kubernetes-1.9 network-policy support
With Kubernetes 1.9, Network Policy has been in `networking.k8s.io/v1` API group. The structure remains unchanged from the beta1 API. The `net.beta.kubernetes.io/network-policy` annotation on Namespaces to opt in to isolation has been removed as well. Contrail-kube-manager needs to support these changes. Contrail has introduced the firewall policy framework. The framework simplified creation of a policy and application of the policy to Virtual-Machines, Containers and Pods. kubernetes network-policy implementation will be moved from Security-Group to Application Policy Set (APS).

Please refer https://github.com/Juniper/contrail-controller/wiki/Kubernetes:-Implementing-Network-Policy-with-Contrail-FW-Policy for more information.

## 4. Implementation

## 5. Performance and scaling impact

### 5.1 API and control plane
None

### 5.2 Forwarding performance

## 6. Upgrade

## 7. Deprecations
None

## 8. Dependencies

## 9. Debugging

## 10. Testing
### 10.1 Unit tests
### 10.2 Dev tests
### 10.3 System tests

## 11. Documentation Impact
