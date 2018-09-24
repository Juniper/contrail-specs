# 1. Introduction
Provide BGP support for 4 byte autonomous systems

# 2. Problem statement
In current contrail implementation, the Autonomous System (AS)
number is encoded as a two-octet entity. To prepare for the anticipated
exhaustion of the two-octet AS numbers, there is a need to support rfc
6793 which describes usage of 4 byte ASN in Bgp.

# 3. Proposed solution
RFC is implemented as it is.

## 3.1 Alternatives considered
None

## 3.2 API schema changes
Current schema restricts configuration of ASN to 2 byte number, it will
be changed to be able to configure 4 byte asn.

```
diff --git a/schema/vnc_cfg.xsd b/schema/vnc_cfg.xsd
index d040fc2..1d7fe79 100644
--- a/schema/vnc_cfg.xsd
+++ b/schema/vnc_cfg.xsd
@@ -682,7 +682,7 @@
targetNamespace="http://www.contrailsystems.com/2012/VNC-CONFIG/0">
 <xsd:simpleType name="AutonomousSystemType">
      <xsd:restriction base="xsd:integer">
          <xsd:minInclusive value="1"/>
-         <xsd:maxInclusive value="65534"/>
+         <xsd:maxInclusive value="4294967294"/>
      </xsd:restriction>
 </xsd:simpleType>
```

## 3.3 User workflow impact
Contrail will by default support 4 byte ASN.

## 3.4 UI changes
UI shall allow 32 bit ASN values to a BGP router.

## 3.5 Notification impact

# 4. Implementation

## 4.1 Interaction between NEW BGP Speakers
A BGP speaker that supports four-octet AS numbers SHALL advertise
this to its peers using BGP Capabilities Advertisements.  The AS
number of the BGP speaker MUST be carried in the Capability Value
field of the "AS4Support capability".

When a NEW BGP speaker processes an OPEN message from another NEW BGP
speaker, it MUST use the AS number encoded in the Capability Value
field of the "AS4Support capability" in lieu of the "My Autonomous
System" field of the OPEN message.

A BGP speaker that advertises such a capability to a particular peer,
and receives from that peer the advertisement of such a capability,
MUST encode AS numbers as four-octet entities in both the AS_PATH
attribute in the updates it sends to the peer and MUST assume that this
attribute in the updates received from the peer encode AS numbers as
four-octet entities.

The new attribute, AS4_PATH, MUST NOT be carried in an UPDATE message
between NEW BGP speakers. A NEW BGP speaker that receives the AS4_PATH
attribute in an UPDATE message from another NEW BGP speaker MUST discard the
path attribute and continue processing the UPDATE message.

## 4.2 Interaction between NEW and OLD BGP Speakers
### 4.2.1 BGP Peering
Peering between a NEW BGP speaker and an OLD BGP speaker is possible
only if the NEW BGP speaker has a two-octet AS number. AS_TRANS MUST be
used when the NEW BGP speaker does not have a two-octet AS number (even
if multiple Autonomous Systems would use it).

### 4.2.2 Generating Updates
When communicating with an OLD BGP speaker, a NEW BGP speaker MUST send
the AS path information in the AS_PATH attribute encoded with two-octet
AS numbers. The NEW BGP speaker MUST also send the AS path information
in the AS4_PATH attribute (encoded with four-octet AS numbers), except
for the case where all of the AS path information is composed of
mappable four-octet AS numbers only.  In this case, the NEW BGP speaker
MUST NOT send the AS4_PATH attribute.

In the AS_PATH attribute encoded with two-octet AS numbers, non-mappable
four-octet AS numbers are represented by the well-known two-octet AS
number, AS_TRANS. This will preserve the path length property of the AS
path information and also help in updating the AS path information
received on a NEW BGP speaker from an OLD BGP speaker.

The NEW BGP speaker constructs the AS4_PATH attribute from the AS path
information. Whenever the AS path information contains the
AS_CONFED_SEQUENCE or AS_CONFED_SET path segment, the NEW BGP speaker
MUST exclude such path segments from the AS4_PATH attribute being
constructed.

The AS4_PATH attribute, being optional transitive, will be carried
across a series of OLD BGP speakers without modification and will help
preserve the non-mappable four-octet AS numbers in the AS path
information.

### 4.2.2 Processing Received Updates
When a NEW BGP speaker receives an update from an OLD BGP speaker, it
MUST be prepared to receive the AS4_PATH attribute along with the
existing AS_PATH attribute. If the AS4_PATH attribute is also received,
both of the attributes will be used to construct the exact AS path
information, and therefore the information carried by both of the
attributes will be considered for AS path loop detection.

## 4.3 Handling BGP Communities
When the high-order two octets of the community attribute is neither
0x0000 nor 0xffff, these two octets encode the AS number. This would not
work for a NEW BGP speaker with a non-mappable four-octet AS number.
Such BGP speakers should use four-octet AS specific extended communities
instead.

## 4.4 Transition
When an Autonomous System is using a two-octet AS number, then the BGP
speakers within that Autonomous System MAY be upgraded to support the
four-octet AS number extensions on a piecemeal basis. There is no
requirement for a coordinated upgrade of the four-octet AS number
capability in this case. An Autonomous System can use a four-octet AS
number only after all the BGP speakers within that Autonomous System
have been upgraded to support four-octet AS numbers.

# 5. Performance and scaling impact

##5.2 Forwarding performance
TBD

# 6. Upgrade
During the upgrade, ASN can not be changed so there should be no impact
on upgrade.

# 7. Deprecations
N/A

# 8. Dependencies

## 8.1

# 9. Testing
## 9.1 Unit tests

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References
[1]: [BGP Support for Four-Octet Autonomous System (AS) Number Space](https://tools.ietf.org/html/rfc6793)
