# 1. Introduction
Implement a user configurable global parameter for T1 route-target.

# 2. Problem statement
In existing implementation, contrail uses hard coded route target in
the form of <asn>:7999999 for 2 byte asn and <asn>:7999 for 4 byte asn.
This approach is not flexible because customer can configure an arbitrary
route-target for this purpose. Customer should be able to configure a
global route target which should be used for all VNs.

# 3. Proposed solution
Here are various steps of proposal:
1. Add a new property to global-systems-config which would indicate number
in the route target. It's default value would be 7999999. Whenever
customer configures this value, it should be verified by both UI and Api
server to make sure that it fits in the range based on existing global
asn value.
2. UI should be modified to provide configurability for this field.
3. Control node should use this newly added field as ESI rtarget index
whenever defined and 7999999 otherwise. In addition, control node should
reset all the existing bgp connections and change all corresponding
route targets whenever this global index is changed.
3. Fabric should also be able to ingest this field and use it to
configure route targets on connected devices. In addition, fabric
should be able to handle change in this field similar to control node.

## 3.1 Alternatives considered
None

## 3.2 User workflow impact
None

# 4. Implementation
Changes are required in many modules. Here is the breakup of work for
different modules.

## 4.1 Schema Changes
A new property evpn-type1-rtarget-number will be added to
global-systems-config in schema. This field can be upto 4 bytes wide.
Following is the proposed change in schema:

'''
diff --git a/schema/vnc_cfg.xsd b/schema/vnc_cfg.xsd
index a336c52..209eef1 100644
--- a/schema/vnc_cfg.xsd
+++ b/schema/vnc_cfg.xsd
@@ -1018,6 +1018,10 @@
targetNamespace="http://www.contrailsystems.com/2012/VNC-CONFIG/0">
 <!--#IFMAP-SEMANTICS-IDL
      Property('enable-4byte-as', 'global-system-config', 'optional', 'CRUD',
               'Knob to enable 4 byte Autonomous System number support.') -->
+<xsd:element name="evpn-type1-rtarget-number" type="xsd:integer" required='optional' default='7999999'/>
+<!--#IFMAP-SEMANTICS-IDL
+     Property('evpn-type1-rtarget-number', 'global-system-config', 'optional', 'CRUD',
+              'Index in Global Route Target for EVPN Type1 routes.') -->
 <xsd:element name="config-version" type="xsd:string"/>
 <!--#IFMAP-SEMANTICS-IDL
      Property('config-version', 'global-system-config', 'system-only', 'R',
'''

## 4.2 Controller Changes
Controller uses default route target of asn:7999999 for Type-1 EVPN routes.
Instead, it will use 'evpn-type1-rtarget-number' from global-system-config,
whenever configured. If the value is not configred, default value of 7999999
will be used. In addition, if configured value is modified then all
existing bgp connections will be reset and esi route targets will be
modified with new value.

## 4.3 Fabric Changes
Similar to Controller, Fabric also uses default route target of asn:7999999
for Type-1 EVPN routes. Changes similar to control node would be required
in fabric also to use route target index from global-system-config if
configured, otherwise keep using 7999999 as default. Fabric should also
be able to handle cases of configured route target value being changed.

## 4.4 UI Changes
UI should show this new configurable field. Similar to other route target
fields, UI should verify that new value is feasible with existing global
autonomous system number. It should be noted that if global ASN is 4
bytes then this index should be less than 2 bytes and vice versa. UI
already does this check at many other places.

## 4.5 Provisioning Changes
While provisionig control node, it should be possible to configure
'evpn-type1-rtarget-number'. This will require changes in provision_control.py
to add a new parameter for that effect. In addition, changes will be
required in contrail-container-builder also to be able to use this
new parameter while calling provision_control.py.

# 5. Performance and scaling impact
N/A

# 6. Upgrade
During the upgrade, this new field will not be configured so default
value will be used. As a result, there will not be any impact.

# 7. Deprecations
N/A

# 8. Dependencies

## 8.1

# 9. Testing
## 9.1 Unit tests
Unit tests will be added by control node to test usage of new field.
Unit tests will also be added by Fabric team for the same.

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References
