# 1. Introduction
Implement a user configurable global parameter for T1 route-target

# 2. Problem statement
In existing implementation, contrail uses hard coded route target in
the form of <asn>:7999999 for 2 byte asn and <asn>:7999 for 4 byte asn.
This approach is not flexible because customer can configure an arbitrary
route-target for this purpose. Customer should be able to configure a
global route target which should be used for all VNs.

# 3. Proposed solution
Here are various steps of proposal:
1. Add a new property to global-systems-config which would indicate number
in the route target. It's default value would be 7999999.  Whenever
customer configures this value, it should be verified by both UI and Api
server that it fits in the range based on asn value.
2. UI should be modified to provide configurability for this field.
3. Control node should use this newly added field as ESI rtarget index
whenever defined and 7999999 otherwise. In addition, control node should
reset all the existing bgp connections and change all associated route
targets whenever this global index is changed.
3. Fabric should also be able to ingest this field and use it to
configure route targets on connected devices. In addition, fabric
should be able to handle change in this value accordingly.

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

## 4.2 Controller Changes
Controller uses default route target of 79999999 for Type-1 EVPN routes.
Instead, it will get corresponding value from global-system-config, if
configured. If the value is not configred, default value od 79999999
will be used. In addition, if configured value is modified then all
existing bgp connections will be reset.

## 4.3 Fabric Changes
Similar to Controller, Fabric also uses default route target of 79999999
for Type-1 EVPN routes. Similar changes would be required in fabric also
to use route target from global-system-config if configured, otherwise
keep using 79999999 as default. Fabric should also be able to handle
cases of configured route target value being changed in config.

## 4.4 UI Changes
UI should show this new field and it should be configurable by user.
Similar to other route target fields, UI should verify that new value
is feasible with existing global autonomous system number. It should
be noted that if global ASN is 4 bytes then this index should be less
than 2 bytes and vice versa. UI already does this check at many other
places.

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
