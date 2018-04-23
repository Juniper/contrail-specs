# 1. Introduction
Kubernetes (K8s) is an open source container management platform. It provides a portable platform across public and private clouds. K8s supports deployment, scaling and auto-healing of applications. More details can be found at: http://kubernetes.io/docs/whatisk8s/

# 2. Problem statement
1. Service reachablity from outside of the cluster in isolated namespaces and service chaining support.
2. Reachability to cluster-services/public-cloud-services/Internet for pods.
3. Multiple Ingress Controllers.
4 Custom Network support for Ingress.
5. K8s Probes and K8s Service Type Node Port.
6. kubernetes-1.9 network-policy support.

# 3. Proposed solution
## 3.1.1 Service reachablity from outside cluster in isolated namespaces and service chaining support
Even though it is an isolated namespace, service ip is allocated from the cluster-network in contrail version <= 4.1. So, by default service from one network can reach service from another network. security groups in isolated namespace stops the reachability from other networks which also prevents traffic from outside of the cluster. In order to give access to external entity, the security group would be changed to allow all which defeats the isolation purpose. To address this, two networks would be created in the isolated namespaces. One is for pods and one is for services(It is applicable for default namespace as well) and contrail-network-policy would be created between pod-network and service-network for the reachablity between pods and services. Service uses the same service-ipam which will be made as a flat-subnet like pod-ipam. Since each network would have its own vrf, isolation is available in nature by contrail virtual-network design. So security groups does not need any rules to filter the traffic for isolation.
   As services would have its own dedicated network in isolated namespace, it will give the option to create service-chaining between pods from one namespace and services from other namespace.

## 3.1.2 Reachability to Cluster-Services/Public-Cloud-Services/Internet for pods
### 3.1.2.1 contrail ip-fabric-forwarding
In 4.1, pod can reach any service in ip-fabric networks as long as the compute node has the vrouter. When pod reaches to a service in another compute node, it leaves source compute node with encapsulation. If the destination compute node does not have the vrouter, the destination node drops the packets since it does not know how to handle the encapsulated packet. So, services would not be reached by pods. It is same as with public cloud infrastructure and internet where there is no vrouter. If the pods have to reach those services, it has to be part of the underlay network [no encapsulation/de capsulation]. It can be achieved by contrail feature ip-fabric-forwarding. Also pods can be reached within the cluster without gateway.

Contrail feature ip-fabric-forwarding will make overlay networks as part of the underlay(ip-fabric) network. So, there is no need for the encapsulation/decapsulation. ip-fabric-forwarding is only applicable for pod-networks. service-networks is always in overlay. If ip-fabric-forwarding is eanbled, pod-networks would be associated to ip-fabric-ipam instead of pod-ipam which is also flat-subnet.
Ip-fabric-forwarding can be enabled/disabled in global level or namespace level. To enable global level, "ip_fabric_forwarding" has to be set "true" in section "[KUBERNETES]" in /etc/contrail/contrail-kubernetes.conf. By default, it is turned off in global level. To enable/disable in namespace level, "ip_fabric_forwarding" has to be set "true"/"false" in namespace annoation like '"opencontrail.org/ip_fabric_forwarding": "true"/"false"'. Once it is turned on, it can not be modified.

[For more information about contrail ip-fabric-forwarding, please refer gateway-less-forwarding.md]

### 3.1.2.2 contrail ip-fabric-snat
Pods can reach cluster-service/public-cloud-services/Internet using contrail ip-fabric-snat as well. The advantage is pods still can be in overlay. It uses node(compute) ip for source nating to reach the required services. It is only applicable to pod-networks. kube-manager would reserve ports 56000-57023 for TCP and 57024 to 58047 for UDP for source nating during the initialization.

Ip-fabric-snat can be enabled/disabled in global level or namespace level. To enable global level, "ip_fabric_snat" has to be set "true" in section "[KUBERNETES]" in /etc/contrail/contrail-kubernetes.conf. By default, it is turned off in global level. To enable/disable in namespace level, "ip_fabric_snat" has to be set "true"/"false" in namespace annoation like '"opencontrail.org/ip_fabric_snat": "true"/"false"'. It can be turned on/off at any time. To enable/disable in default-pod-network, default namespace should be used. If the ip_fabric_forwarding is enabled, ip_fabric_snat would be ignored

[For more information about contrail ip-fabric-snat, please refer distributed-snat.md]

## 3.1.3 Multiple Ingress Controllers
Multiple Ingress Controllers can co-exist along with contrail Ingress Controller. If k8s ingress resource does not have "kubernetes.io/ingress.class" in annotations or "kubernetes.io/ingress.class"  is "opencontrail" in annotations, contrail ingress controller will create haproxy loadbalancer. Otherwise it will ignore the ingress resource and respective ingress controller will handle the ingress resource. Since Contrail ensures connectivity between pods and services, any ingress controller can reach the endpoints(pods) directly or through services.

## 3.1.4 Custom Network support for Ingress
Contrail supports custom-network in namespace level. It is supported for pods till 4.1. It is extended to contrail ingress controller in 5.0

## 3.1.5 K8s Probes and K8s Service Type Node Port
Contrail network-policy between ip-fabric network and pod-network enables the connectivity between node and pod. It will help kubelet to reach the pod for probes(readiness and liveness). Whenever the pod-network is created the network-policy is attached to pod-network which connects ip-fabric network and pod-network. It is not only the kubelet, any service can reach the pod.

K8s Service Node Port is fully based on the node connectivity to the pod. Since contrail can provide the connectivity between node and pods through contrail network-policy, Node Port can be enabled.

## 3.1.6 kubernetes-1.9 network-policy support
With Kubernetes 1.9, Network Policy has been in `networking.k8s.io/v1` API group. The structure remains unchanged from the beta1 API. The `net.beta.kubernetes.io/network-policy` annotation on Namespaces to opt in to isolation has been removed as well. Contrail-kube-manager needs to support these changes. Contrail has introduced the firewall policy framework. The framework simplified creation of a policy and application of the policy to Virtual-Machines, Containers and Pods. kubernetes network-policy implementation will be moved from Security-Group to Application Policy Set (APS).

# 3.2.1 Implementation

# 4. Implementation

# 5. Performance and scaling impact

## 5.1 API and control plane
None

## 5.2 Forwarding performance

# 6. Upgrade

# 7. Deprecations
None

# 8. Dependencies

# 9. Debugging

# 10. Testing
## 10.1 Unit tests
## 10.2 Dev tests
## 10.3 System tests

# 11. Documentation Impact
