
# 1. Introduction

In contrail, IBGP routers do not support route-reflector configuration because
of which full mesh connection is required among IBGP
neighbours. As part of this feature, route reflector support will be added to
control by supporting RFC 4456.
   

# 2. Problem statement

Need for route-reflector support for IBGP is well known. Typically, all BGP
speakers within a single AS must be fully meshed and any external routing
information must be re-distributed to all other routers within that AS. For n
BGP speakers within an AS that requires to maintain n\*(n-1)/2 unique Internal
BGP (IBGP) sessions. This "full mesh" requirement clearly does not scale when
there are a large number of IBGP speakers each exchanging a large volume of
routing information, as is common in many of today's networks.
Route-Reflection is one well known way to alleviate that problem.


# 3. Proposed solution

Use Route Reflectors so that full mesh connectivity is not needed among IBGP
peers in contrail.

## 3.1 Alternatives considered

None

## 3.2 API schema changes

A Bgp router that has cluster id assigned to it, will work as route reflector.
To achieve that, Bgp Router Parameters will add one more parameter:

```
diff --git a/schema/bgp_schema.xsd b/schema/bgp_schema.xsd
index 78c5b3b..8ae0203 100644
--- a/schema/bgp_schema.xsd
+++ b/schema/bgp_schema.xsd
@@ -128,6 +128,8 @@
          description='Administratively up or down.'/>
     <xsd:element name='vendor' type='xsd:string' required='optional'
operations='CRUD'
          description='Vendor name for this BGP router, contrail, juniper or
cisco etc.'/>
+    <xsd:element name='cluster-id' type='xsd:integer' required='optional'
     operations='CRUD'
+         description='Cluster Id for this BGP router.'/>
     <xsd:element name='autonomous-system' type='xsd:integer' required='true'
operations='CUR'
          description='Autonomous System number for this BGP router. Currently
only 16 bit AS number is supported. For contrail control nodes this has to be
equal to global AS number.'/>
     <xsd:element name='identifier' type='smi:IpAddress' required='true'
operations='CUR'
```


## 3.3 User workflow impact

User can assign any number of route reflectors in the network. Each RR should
have unique cluster id so that they can recognise each other.  Once RRs are
used, there is no need to have full mesh connectivity and a user can have any
form of connectivity.

## 3.4 UI changes
#### Describe any UI changes

## 3.5 Notification impact
#### Describe any log, UVE, alarm changes

# 4. Implementation

BgpProtocolConfig is modified to have cluster id for a router. This is used by
a router for its own configuration. In addition, CheckSplitHorizon function is
modified to support route reflectors. In existing code, it returns true if the
peer is IBGP neighbour since we don’t forward routes learnt from IBGP
neighbours. However, with route reflector support, following changes are made:

```
+bool BgpPeer::CheckSplitHorizon(uint32_t server_cluster_id, uint32_t
ribout_cid) const {
+    if (PeerType() != BgpProto::IBGP)
+        return false;
+    // check if router is a route reflector
+    if (!server_cluster_id) return true;
+    // check if received from client or non-client by comparing the clusterId
+    // of server with that of peer from which this route is received
+    if (server_cluster_id != peer_cluster_id_) {
+        // If received from non-client, reflect to all the clients only
+        if (ribout_cid && ribout_cid != peer_cluster_id) {
+            return true;
+        }
+    }
+    return false;
+}
+
```

Now, function checks if the router is acting as route reflector. If it is not
a route reflector then old behaviour is retained. However, if it is a RR, it
checks if the peer from which route is received is a client or non-client. If
it is a client then the route is advertised to all the clients and non-clients.
On the other hand, if it is a non-client, then the route is forwarded only to
the clients (and not to other non-clients).

In addition route reflector adds couple of attributes:
1. When a RR reflects a route, it must prepend the local CLUSTER_ID to the
   CLUSTER_LIST.  If the CLUSTER_LIST is empty, it MUST create a new
   one. Using this attribute an RR can identify if the routing information has
   looped back to the same cluster due to misconfiguration. If the local
   CLUSTER_ID is found in the CLUSTER_LIST, the advertisement received SHOULD
   be ignored.
2. A route reflector adds its own bop identifier in the ORIGINATOR_ID
   attribute, if it is not already present. However, if this attribute is
   already present then the route is ignored if received with its BGP Identifier
   as the ORIGINATOR_ID.

# 5. Performance and scaling impact

N/A

# 6. Upgrade

N/A

# 7. Deprecations

N/A

# 8. Dependencies

N/A

# 9. Testing

Unite test cases will be added to check for different scenarios when a router
acts as a route reflector.

# 10. Documentation Impact

# 11. References

