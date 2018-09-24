
# 1. Introduction
Existing routing policies in contrail neither allows a user to do match on
extended communities nor to add an action to add/set/remove extended
communities. There is a requirement from some customers to be able to filter
routes based on extended communities.

# 2. Problem statement
The enhancements introduced address the requirement of being able to match and
apply action on extended communities.

# 3. Proposed solution

Routing policies will be further enhanced to:
 * Allow Import Routing Policies terms to be able to match on extended policies.
 * Allow Import Routing Policies action to be able to add/set/remove extended
   communities.

    Examples: 
```
from extcommunity-list target:2500:1 interface then action reject
from protocol interface then add extcommunity target:2500:1 action accept 
```

## 3.1 Alternatives considered
None, this is a feature enchancement providing more flexibility.

## 3.2 API schema changes

```
--- a/schema/routing_policy.xsd
+++ b/schema/routing_policy.xsd
@@ -41,10 +41,11 @@
 </xsd:complexType>

 <xsd:complexType name="ActionUpdateType">
-    <xsd:element name="as-path"     type="ActionAsPathType"/>
-    <xsd:element name="community"   type="ActionCommunityType"/>
-    <xsd:element name="local-pref"  type="xsd:integer"/>
-    <xsd:element name="med"         type="xsd:integer"/>
+    <xsd:element name="as-path"         type="ActionAsPathType"/>
+    <xsd:element name="community"       type="ActionCommunityType"/>
+    <xsd:element name="extcommunity"    type="ActionCommunityType"/>
+    <xsd:element name="local-pref"      type="xsd:integer"/>
+    <xsd:element name="med"             type="xsd:integer"/>
 </xsd:complexType>

 <xsd:complexType name='TermActionListType'>
@@ -85,6 +86,8 @@
     <xsd:element name="community" type="xsd:string"/> <!-- DEPRECATED -->
     <xsd:element name="community-list" type="xsd:string"
maxOccurs="unbounded"/>
     <xsd:element name="community-match-all" type="xsd:boolean"/>
+    <xsd:element name="extcommunity-list" type="xsd:string"
     maxOccurs="unbounded"/>
+    <xsd:element name="extcommunity-match-all" type="xsd:boolean"/>
 </xsd:complexType>

 <xsd:complexType name='PolicyTermType'>
```

## 3.3 User workflow impact

Users will be able to configure new Term Match and Action attributes as stated
earlier.

## 3.4 UI changes

 * UI to add additional extcommunity Term Match under Configure > Networking >
   Routing > Routing Policies
 * UI to add ability to update the extcommunity list attribute as an action.

## 3.5 Notification impact None.

# 4. Implementation

Control-Node will add support to match on extended communities similar to
communities support. Match input can be string representation og extended
communities or hexadecimal value for the same.
    E.g: ".*200170000000b", "target:23:11"

Similarly, support to provide an action for modifying extended community will
also be added. Extended communities can be added, modified or deleted similar to
communities.

# 5. Performance and scaling impact None.

## 5.2 Forwarding performance There should be no forwarding performance impact.

# 6. Upgrade None.

# 7. Deprecations None.

# 8. Dependencies None.

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact

# 11. References
