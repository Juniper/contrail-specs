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
When enabled, contrail will support 4 byte ASN.

## 3.4 UI changes
There will be a knob added under global system config to enable/disable 4 byte
asn support. By default, this feature will be disabled. If this knob is
enabled, UI shall allow 32 bit ASN values to a BGP router.

## 3.5 Notification impact

# 4. Implementation
Currently, there is an attribute AS_PATH which has a sequence of 2 byte asn.
This would need to be changed since some BGP neighbors support 2 byte ASN and
some others may support 4 byte asn. Because of this requirement there are 2
new data structures added:
1. AS_PATH with 4 byte asn
2. AS4_PATH with 4 byte asn

Newly added AS4_PATH attribute is transitive and optional:
```
struct As4PathSpec : public BgpAttribute {
    static const uint8_t kFlags = Transitive|Optional;
    std::vector<as_t> path_segment;
}
```

## 4.1 Interaction between NEW BGP Speakers
A BGP speaker that supports four-octet AS numbers will advertise
this to its peers using BGP Capabilities Advertisements.  The AS
number of the BGP speaker MUST be carried in the Capability Value
field of the "AS4Support capability".

When a new BGP speaker processes an OPEN message from another new BGP
speaker, it must use the AS number encoded in the Capability Value
field of the "AS4Support capability" in lieu of the "My Autonomous
System" field of the OPEN message.

A BGP speaker that advertises such a capability to a particular peer,
and receives from that peer the advertisement of such a capability,
must encode AS numbers as four-octet entities in both the AS_PATH
attribute in the updates it sends to the peer and must assume that this
attribute in the updates received from the peer encode AS numbers as
four-octet entities.

The new attribute, AS4_PATH, must not be carried in an UPDATE message
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
When communicating with an OLD BGP speaker, a NEW BGP speaker sends
the AS path information in the AS_PATH attribute encoded with two-octet
AS numbers. The NEW BGP speaker also sends the AS path information
in the AS4_PATH attribute (encoded with four-octet AS numbers), except
for the case where all of the AS path information is composed of
mappable four-octet AS numbers only.  In this case, the NEW BGP speaker
does not send the AS4_PATH attribute.

In the AS_PATH attribute encoded with two-octet AS numbers, non-mappable
four-octet AS numbers are represented by the well-known two-octet AS
number, AS_TRANS. This will preserve the path length property of the AS
path information and also help in updating the AS path information
received on a NEW BGP speaker from an OLD BGP speaker.

The AS4_PATH attribute, being optional transitive, will be carried
across a series of OLD BGP speakers without modification and will help
preserve the non-mappable four-octet AS numbers in the AS path
information.

### 4.2.2 Processing Received Updates
When a NEW BGP speaker receives an update from an OLD BGP speaker, it
may also receive the AS4_PATH attribute along with the
existing AS_PATH attribute. If the AS4_PATH attribute is also received,
both of the attributes will be used to construct the exact AS path
information, and therefore the information carried by both of the
attributes will be considered for AS path loop detection.

While reconstructing AS_PATH information, If the number of AS numbers in the
AS_PATH attribute is less than the number of AS numbers in the AS4_PATH
attribute, then the AS4_PATH attribute is ignored, and the AS_PATH attribute
is taken as the AS path information. If the number of AS numbers in the
AS_PATH attribute is larger than or equal to the number of AS numbers in the
AS4_PATH attribute, then the AS path information is constructed by taking as
many AS numbers and path segments as necessary from the leading part of the
AS_PATH attribute, and then prepending them to the AS4_PATH attribute so that
the AS path information has an identical number of AS numbers as the AS_PATH
attribute.

### 4.3 Handling AS-OVERRIDE
When as-override feature is enabled, bgp router is supposed to delete latest
asn in as_path and replace it with local asn. Similar to prepending of asn in
as path, as override will depend on whether the peer supports 4 byte asn or
not. If the peer does not support 4 byte asn and local asn fits in 2 btyes
then peer asn in simply replaced with local asn (same as existing code).
However if local asn does not fit in 2 bytes then we need to replace peer asn
with as_trans and add local asn in as4_path attribute.

## 4.4 Handling BGP Communities
When the high-order two octets of the community attribute is neither
0x0000 nor 0xffff, these two octets encode the AS number. This would not
work for a NEW BGP speaker with a non-mappable four-octet AS number.
Such BGP speakers should use four-octet AS specific extended communities
instead.

## 4.5 Fabric Support for 4 Byte ASN
CFM Ansible plugins support configuration of 4 byte AS number to Juniper
devices. To configure a “target” extended community, which includes a 4-byte
AS number in the plain-number format, the plugins append the letter “L” to the
end of the number. Example: Consider a target community with the 4-byte AS
number 334324 and an assigned number of 132. In this case, CFM pushes the Route
target as “target:334324L:132” to the Juniper device.

## 4.6 Transition
When an Autonomous System is using a two-octet AS number, then the BGP
speakers within that Autonomous System MAY be upgraded to support the
four-octet AS number extensions on a piecemeal basis. There is no
requirement for a coordinated upgrade of the four-octet AS number
capability in this case. An Autonomous System can use a four-octet AS
number only after all the BGP speakers within that Autonomous System
have been upgraded to support four-octet AS numbers.

### 4.6.1 Route Target Update
Contrail Schema automatically generates a route target for every VN whenever
it is created. These route targets have 4 byte index field, whereby, assuming
that AS number field is 2 bytes. In case of transition from 2 byte ASN to 4
byte ASN, these route targets need to be updated by changing the index field
so that its value fits in 2 bytes. In addition, if ASN value is bigger than 2
bytes, then all new route targets generated should have index field that
should be 2 byte only.

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
Unit tests have been added for testing new BGP attribute. AS_PATH is tested
for the cases when peer does or does not support 4 byte asn. Following test
cases are added:
1. EBgpAsPrepend1RiboutAs2PeerAs2:
    Peer is old speaker, ribout is old speaker and AS_PATH is empty. Local asn
should be prepended to as_path
    Inbound AS_PATH: [100], Outbound AS_PATH: [local_as, 100]
2. EBgpAsPrepend1RiboutAs2PeerAs2WithAs4Path
    Peer is old speaker, ribout is old speaker and AS_PATH has 2 asns along
with AS4_PATH having 2 values. Local asn should be prepended to both as_path
and as4_path
    Inbound AS_PATH: [23456, 100], AS4_PATH: [65000, 100]
    Outbound AS_PATH: [local_as, 23456, 100] AS4_PATH: [local_as, 65000, 100]
3. EBgpAsPrepend1RiboutAs2PeerAs2WithAs4Path2
    If as4_path length is more than as_path length then as4_path should be
ignored
    Inbound AS_PATH: [100], AS4_PATH: [75000, 65000]
    Outbound AS_PATH: [local_as, 100]
4. EBgpAsPrepend1RiboutAs2PeerAs4
    Peer is old speaker, ribout is new speaker and AS_PATH4 has a small asn.
    Local asn should be prepended to as_path
    Inbound AS_PATH4: [100]
    Outbound AS_PATH: [local_as, 100]
5. EBgpAsPrepend1RiboutAs2PeerAs4WithAs4Path
    Peer is new speaker, ribout is old speaker and as_path4 has mixed asns.
    as_path and as4_path should be sent out.
    Inbound AS_PATH4: [80000, 70000, 100]
    Outbound AS_PATH: [local_as, 23456, 23456, 100]
            AS4_PATH: [local_as, 80000, 70000, 100]
6. EBgpAsPrepend1RiboutAs2PeerAs4WithAs4PathSet
7. EBgpAsPrepend2RiboutAs2PeerAs4WithAs4PathSet
    Same as previous one but with AS_SET instead of AS_SEQUENCE
8. EBgpAsPrepend1RiboutAs4PeerAs2
    Peer is old speaker, ribout is new speaker and non empty AS_PATH. Local asn
    should be prepended to as_path
    Inbound AS_PATH: [600, 500, 100]
    Outbound AS_PATH4: [local_as, 600, 500, 100]
9. EBgpAsPrepend1RiboutAs4PeerAs2WithAs4Path
    Peer is old speaker, ribout is new speaker and both as_path and as4_path
    are present. as_path4 should be created by combining as_path and as4_path
    and local asn should be prepended by taking 4 asns from as4_path and 1
    from as_path
    Inbound AS_PATH: [650, 23456, 400, 23456, 100]
            AS4_PATH:[80000, 400, 70000, 100]
    Outbound AS_PATH4: [local_as, 650, 80000, 400, 70000, 100]
10. EBgpAsPrepend1RiboutAs4PeerAs4
    Peer is new speaker, ribout is new speaker and non empty as_path4. Local asn
    should be prepended to as_path4
11. EBgpAsPrepend1RiboutAs4PeerAs4Type2
    Same as previous one but with 4 byte values in as_path4.
12. RemovePrivateAllAs4
    Add private 4 byte asn in as_path4 and make sure it gets removed
13. RemovePrivateAllReplace1
    Add private 4 byte asn in as_path4 and make sure it gets replaced with
    local as. Peer is 2 byte so if local asn is > 0xFFFF then private asn
    should be replaced with AS_TRANS and actual as added to as4_path
14. RemovePrivateAllAs4Replace1
    Same as previous one but with 4 byte private asn
15. RemovePrivateAllReplace2
    Remove private all (w/ replace) replaces all private ASes. All
    private ASes up to the first public AS are replaced with the local
    as. Remaining private ASes are replaced with the nearest public AS.
16. RemovePrivateAllAs4Replace2
    Same as previous one but with 4 byte private asn
17. All these test cases are run with DUT being 4 byte capable as well as not
    capable.

In addition, some other tests have been added to test connectivity of bgp
speakers in hetrogeneous environment where some peers are 4 byte capable and
others are 2 byte capable only.

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References
[1]: [BGP Support for Four-Octet Autonomous System (AS) Number Space](https://tools.ietf.org/html/rfc6793)
