# 1. Introduction
Add a new extended community called origin sub-cluster in order to filter inter cluster routes correctly using policy.

# 2. Problem statement
Since Contrail 5.0, Routing policy scope has been extended to support several route types and can be applied at network level.
This improves significantly the import routing toolbox.
But this routing policy doesn’t match with MP-BGP route type. As a consequence, it is not possible to match routes from the MPLS backbone on any criteria.
Thus, these routes are imported as they are.

# 3. Proposed solution
Inside Contrail POP, and per customer, we must be able to match routes with some community tag in order to assign a local preference.
Other routes not matching the community tag must be rejected.
Hence we need to add an extended community called origin-sub-cluster to all routes originated within a sub-cluster (similar to origin-vn) by encoding sub-cluster-id as the value field inside the extended-community.
For a 2 byte ASN format, the type will be Experimental(0x80) and sub-type will be Origin Sub-Cluster(0x85). Hence fisrt 2 bytes will store type and sub-type, next 2 bytes will store ASN and last 4 bytes will store sub-cluster-id.
For a 4 byte ASN format, the type will be Experimental4ByteAs(0x82) and sub-type will be Origin Sub-Cluster(0x85). Hence fisrt 2 bytes will store type and sub-type, next 4 bytes will store ANS and last 2 bytes will store sub-cluster-id. 

Also we need to take care of below scenarios while applying policies:
1. Apply virtual-network policy to secondary routes too, but only if self sub-cluster id does not match with that associated with the route.
2. If there is no sub-cluster extended-community associated with the route, then we have to apply the policy for secondary routes.

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

### Get sub-cluster-id from coming from the schema
We can get sub_cluster_id from BgpIfmapConfigManager and add set it to the BgpProtocolConfig data_ in BgpIfmapProtocolConfig::Update method.
Later when sub-cluster-id is required, we can get it from BgpProtocolConfig data_.
```
         IFMapNode *adj = static_cast<IFMapNode *>(iter.operator->());
         if (strcmp(adj->table()->Typename(), "sub-cluster") == 0) {
             data_.set_subcluster_name(adj->name());
+            const autogen::SubCluster *sc =
+                static_cast<autogen::SubCluster *>(adj->GetObject());
+            if (sc) {
+                data_.set_subcluster_id(sc->id());
+            }
             break;
         }
     }
```

### Attach sub-cluster extended communities to all routes originating in that sub-cluster
Sub-cluster attribute is necessary in order to filter inter cluster routes correctly using policy.

BgpAttrPtr BgpTable::AddExtCommunitySubCluster will have the code logic for setting sub-cluster id in extended community
```
+BgpAttrPtr BgpTable::AddExtCommunitySubCluster(BgpAttr *attr,
+        uint16_t subcluster_id) {
+    ExtCommunityPtr ext_community = attr->ext_community();
+    if (!ext_community) {
+        return NULL;
+    }
+    SubCluster sc(server()->autonomous_system(), subcluster_id);
+    ext_community = server()->extcomm_db()->
+        ReplaceSubClusterAndLocate(ext_community.get(), sc.GetExtCommunity());
+    return server()->attr_db()->
+        ReplaceExtCommunityAndLocate(attr, ext_community);
+}
```
Now when we insert a new path to a route and if the path is a primary path then we need to add origin sub-cluster extended community in the bgp attribute list of the path.
We need to add below code to BgpRoute::InsertPath()
```
        if (table->IsRoutingPolicySupported()) {
+           if (!path->IsReplicated()) {
+               // Add sub-cluster extended community to all routes
+               // originated within a sub-cluster
+               BgpAttr *attr = new BgpAttr(*(path->GetAttr()));
+               uint16_t subcluster_id = SubClusterId();
+               if (subcluster_id) {
+                   BgpAttrPtr modified_attr = table->AddExtCommunitySubCluster(
+                           attr, subcluster_id);
+                   // Update the path with new set of attributes
+                   if (modified_attr) {
+                       path->SetAttr(modified_attr, path->GetOriginalAttr());
+                   }
+               }
+           }
```
### Apply virtual-network route policies to secondary route only if self sub-cluster id does not match with what is associated with the route.
For this to achieve, we need to the add below check in RoutingInstance::ProcessRoutingPolicy
```
 bool RoutingInstance::ProcessRoutingPolicy(const BgpRoute *route,
                                            BgpPath *path) const {
+    ExtCommunityPtr ext_community = path->GetAttr()->ext_community();
+    if (path->IsReplicated() &&
+            ext_community && ext_community->GetSubClusterId()) {
+        //If self sub-cluster id macthes with what is associated with the route
+        //then no need to apply policy.
+        if (route->SubClusterId() == ext_community->GetSubClusterId()) {
+            return true;
+        }
+    }
```

### Parse new extended community subcluster```:<as>:<number>```
Need to implement FromString() method which would take string in format 'subcluster:<asn>:<id>' and return a subcluster object.
Aslo this method should validate whether the string is valid sub-cluster extended community string.
It should check if the asn and id are in valid range.

Need to implement ToString() method which will return a string in format 'subcluster:<asn>:<id>' from a subcluster object.

# 5. Performance and scaling impact
None.

# 6. Change in existing behaviour
Yes, as we have changed the way we are going to apply the routing-policy.
1. Apply routing-policy to secondary routes coming from different sub-cluster.
2. Apply routing-policy to secondary routes which are not associated to any sub-cluster.

# 7. Upgrade
None.

# 8. Deprecations
None.

# 9. Dependencies
None.

# 10. Testing
## 10.1 Unit tests
1. Need to add and update test-cases in routing_policy_test.cc to verfiy the change in behaviour of how routing policies are applied to secondary path.

2. Need to add sub_cluster_test.cc to test whether sub-cluster extended communities is applied to all the routes originating in a sub-cluster.
## 10.2 Dev tests
1. Need to write a detailed test plan to test this feature.
2. Need to add UT automations.
## 10.3 System tests
# 11. Documentation Impact
# 12. References
