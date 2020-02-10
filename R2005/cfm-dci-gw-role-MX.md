# 1. Introduction
Contrail Fabric Manager (CFM) should be able to automate the interconnect of two different data centers. Multiple Tenants (Shared VRF) which are connected to a Logical Router using single Shared VRF in given DC should be able to exchange routes with other shared VRF of another DC. It should also be possible to have dedicated tenant should be able to exchange routes with another dedicated VRF on another DC.

### Supported Deployment model for MX:
Supported Roles:
- DCI-Gateway@spine

#### Supported Device Models
We are qualifying DCI-GW overlay role on MX family for MX 240, MX 480, MX 960, MX 10003 models.

# 2. Problem statement
CFM needs to support DCI gateway role for MX devices. 

#### Software Release versions
MX devices running with 18.4R2-S3 or above are supported for this functionality.

#### Deployment model:
Supported Roles:
- DCI-Gateway@spine

# 3. Proposed solution
The proposed solution is to use the existing CFM architecture for role based config push to the DC devices and add the role required for MX. CFM already supports the DCI gateway role for QFX devices, we will leverage the existing configuration templates and make respective changes required for MX support and add any templates as needed.

# 4. API schema changes
N/A

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
The existing user workflow to assign roles remains the same. The user will see additional choice (`DCI-Gateway`) for the applicable devices.

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
## 9.1 Add roles to node profile
CFM uses the node profile object to determine the roles applicable for a type of device. We will add the `DCI-Gateway` role for MX  as `spine`.

# 10. Device specifics
MX devices running with 18.4R2-S3 or above are supported for this functionality.

# 11. Performance and scaling impact
N/A

# 12. Upgrade
#### Backward compatible changes

# 13. Deprecations
N/A

# 14. Dependencies
N/A

# 15. Testing
Below is the link to the test plan:
https://docs.google.com/spreadsheets/d/1eVCGS-SVw8VRWtCgEHa2ZC-aRBKJxaSyXR8KkXv6pZw/edit#gid=0

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 17. References
JIRA Story: https://contrail-jws.atlassian.net/browse/CEM-11256
