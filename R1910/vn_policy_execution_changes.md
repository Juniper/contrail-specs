# 1. Introduction
Changes to virtual-network policy execution semantics.

# 2. Problem statement
We need to add sub-cluster extended community called origin-sub-cluster extended
community to all routes originated within a sub-cluster (similar to origin-vn)
by encoding sub-cluster-id as the value field inside the extended-community.
For this we need to get a type reserved from IANA.

Apply import policy to virtual-network to secondary routes too, but only if self
sub-cluster id does not match with what is associated with the route.

If there is no sub-cluster extended-community associated with the route, then we
don't have to apply the policy for secondary routes.

If route is service-chain route, then do not apply virtual-network policy
(primary or secondary). Instead, apply service-chain policy to only
service-chain primary routes.

# 3. Proposed solution
## Apply virtual-network route policies to secondary route too
For this to achieve, we need to the relax check in
RoutingInstance::ProcessRoutingPolicy so that we don't bail out when the path is
secondary and apply the policy. IsReplicated method defined in BgpPath class
tells whether its a secondary path or not.
Also we need to add and update the test-cases to make sure that relaxing this
check doesn't break any thing else.
Below code needs to be removed from RoutingInstance::ProcessRoutingPolicy().

if (path->IsReplicated())
  return true;

Please check implementation section for more details

## Apply service-chain policy only for primary service chain routes
For this we need to add api called IsServiceChainRoute in class RoutingInstance.
This api will checks whether its a service-chain route. If yes, then we need to
check if the path is a secondary path by calling IsReplicated. If it is a
secondary path then we return and do not apply service-chain policy, else it's
a primary path and we need to apply the policy.

Please check implementation section for more details

## Attach sub-cluster extended communities to all routes originating in that
sub-cluster
Sub-cluster attribte is necessary in order to filter inter cluster routes
correctly using policy.

We need to apply virtual-network route policy to both primary and secondary
routes only if sub-cluster id does not match what is associated with the route.

If no sub-cluster extended community is associated with the route, then do apply
the policy for secondary routes.

## Parse new extended community subcluster```:<as>:<number>```
We need to implement FromString() and ToString() methods. Allow use to configure
this in a route-policy match or action.
Provide support for both 2 byte and 4 byte as formats.

## 3.1 Alternatives considered
None, this is a feature enchancement providing more flexibility.

## 3.2 API schema changes
None.

## 3.3 User workflow impact
None.

## 3.4 UI changes
None.

## 3.5 Notification impact 
None.

# 4. Implementation
To fix first two issues as mentioned in proposed solution section, we will add a
new method IsServiceChainRoute and modify RoutingInstance::ProcessRoutingPolicy
as below:
```
+bool RoutingInstance::IsServiceChainRoute(const BgpRoute *route) const {
+    const BgpTable *table = route->table();
+    if (!table) {
+        return false;
+    }
+    const BgpInstanceConfig *rt_config = this->config();
+    if (!rt_config) {
+        return false;
+    }
+    const ServiceChainConfig *sc_config =
+           rt_config->service_chain_info(table->family());
+    if (!sc_config) {
+        return false;
+    }
+    return true;
+}
+
 bool RoutingInstance::ProcessRoutingPolicy(const BgpRoute *route,
                                            BgpPath *path) const {
-    // Don't apply routing policy on secondary path
-    if (path->IsReplicated())
-        return true;
+    if (IsServiceChainRoute(route) && path->IsReplicated()) {
+        // Don't apply routing policy on secondary path for service chain route
+        return true;
+    }
```
Thus the only scenario where we are not going to apply routing policy is when we
it's a service chain route and its a secondary path.

# 5. Performance and scaling impact
None.

# 6. Upgrade
None

# 7. Deprecations
None.

# 8. Dependencies
None.

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact
# 11. References
