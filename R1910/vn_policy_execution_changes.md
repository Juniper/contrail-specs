# 1. Introduction
Changes to virtual-network policy execution semantics.

# 2. Problem statement
Routing policies doesn't match with MP-BGP route type. As a consequence, it is
not possible to match routes from the MPLS backbone on any criteria. Hence,
these routes are imported as they are and there is no way to apply policies
on these imported routes.

# 3. Proposed solution
Apply virtual-network routing-policy to secondary(imported) routes too.

If route is service-chain route, then do not apply virtual-network policy
(primary or secondary). Instead, apply service-chain policy to only
service-chain primary routes.

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

## Apply virtual-network route policies to secondary route too
For this to achieve, we need to the relax check
in RoutingInstance::ProcessRoutingPolicy so that we don't bail out when the path
is secondary but still apply the policy. IsReplicated method defined in BgpPath
class tells whether its a secondary path or not.
Also we need to add and update the test-cases to make sure that relaxing this
check doesn't break any thing else.

## Apply service-chain policy only for primary service chain routes
For this we need to add api called IsServiceChainRoute in class RoutingInstance.
This api will check whether its a service-chain route. So if it is a secondary
path and a service-chain route then we return and do not apply service-chain
policy, else it's a primary path and we need to apply the policy.

To fix above two issues, we will add a new method IsServiceChainRoute and modify
RoutingInstance::ProcessRoutingPolicy as below:
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
+    if (path->IsReplicated() && IsServiceChainRoute(route)) {
+        // Don't apply routing policy on secondary path for service chain route
+        return true;
+    }
```
Thus the only scenario where we are not going to apply routing policy is when
it's a secondary path and a service chain route.

# 5. Performance and scaling impact
None.

# 6. Change in existing behaviour
Yes, since we are going to apply policy to secondary path for virtual-network
route and only for primary path for service-chain route.
Earlier this was not supported.

# 7. Upgrade
None

# 8. Deprecations
None.

# 9. Dependencies
None.

# 10. Testing
## 10.1 Unit tests
1. Need to add and update test-cases in routing_policy_test.cc to verfiy the
change in behaviour of how routing policies are applied to secondary path.

## 10.2 Dev tests
## 10.3 System tests
# 11. Documentation Impact
# 12. References
