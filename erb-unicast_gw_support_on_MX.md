# 1. Introduction
Contrail Fabric Manager (CFM) supports assigning physical roles and routing bridging (overlay) roles to a device. These roles together define the roles and responsibilities of the device in the DC.
The Edge-Routed Bridging (ERB) for QFX series switches feature configures the inter-VN unicast traffic routing to occur at the leaf (ToR) switches in an IP CLOS with underlay connectivity topology.
The ERB feature introduces the ERB-UCAST-Gateway and CRB-MCAST-Gateway roles in Contrail Networking Release 5.1.

ERB is supported on the following devices running Junos OS Release 18.1R3 and later:

QFX5110-48S,QFX5110-32Q,QFX10002-36Q,QFX10002-72Q,QFX10008,QFX10016

# 2. Problem statement
As stated in the introduction, we currently donot support ERB-UCAST-Gateway on MX.

When a Virtual Network is configured on a port of a device provision the following (only for UNICAST traffic)
    The Interface/IFL to VXLAN VNI mapping
    The Bridge Domain
When a Logical Router across 2 or more Virtual Network is created, provision on all Devices where these VN exist
    The Anycast Gateway (irb) units and addresses
    The L3 VRF with the irb per Virtual Network which part of the Logical Router.
# 3. Proposed solution
Add support for ERB-UCAST-Gateway below MX platforms-
- MX240
- MX480

# 4. API schema changes
N/A
# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
N/A

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
    1. CFM uses the node profile object to determine the roles applicable for a type of device. We will add the ERB-UCAST-Gateway role for MX as leaf.

    2. CFM uess the jinja templates to configure the devices with required configurations
        a. Add the underlay configuration related templates (eBGP related configurations and BMS Access Infra) for the MX devices under the ERB-UCAST-Gateway.
        b. Add Link-Aggregation-Groups (LAG) and Multi Homing (MH) related templates.
        c. Add a template to route the inter virtual network traffic.

# 10. Device specifics

#### Supported Junos Software Release
MX devices running with 18.4R2-S3 or above are supported for this functionality.

# 11. Performance and scaling impact
N/A

# 12. Upgrade
N/A

# 13. Deprecations
N/A

# 14. Dependencies

# 15. Testing
Below is the link to the test plan,
https://docs.google.com/spreadsheets/d/1vtUmWLmUH929AdJgvDiopYxJMHySHVomezVZdsxbaqU/edit?usp=sharing

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 17. References
JIRA Story: https://contrail-jws.atlassian.net/browse/CEM-5407

# 18. Caveats or Known issues

On MX devices, DC-GW and ERB-Ucast-GW supported at leaf level. These roles can't co-exist together for this release as we have not qualified, some more code changes required, but individually they are supported.
