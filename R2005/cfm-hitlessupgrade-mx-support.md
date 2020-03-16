# 1. Introduction
Image/Software upgrade on the device is a time consuming task. Whenever a device is re-imaged it goes through a series of steps that also includes a reboot. Depending on the size of the image, it roughly takes about 20 mins for the device to come up to start functioning again. During this procedure, if live traffic is still going through the device a number of packets are lost. This has an adverse effect on the performance of the fabric especially when multiple devices are being upgraded simultaneously. Hence there is a requirement to divert the traffic flowing through this device to ensure zero packet loss.

Use cases
The main use case is when the user wants to perform a controller assisted maintenance procedure (Software upgrade here) on a DC Fabric managed by Contrail with zero packet loss.

# 2. Problem statement
MX in EVPN VXLAN fabric needs to be supported for the Hitless upgrade.

# 3. Proposed solution
Will used the existing infrastructure of the Hitless upgrade as it is to support the MX devices. Except the new checks as part of the upgrade and the health checks everything will remain common. Even though the MX boxes supports ISSU/NSSU (In Service Software Upgrade/Non Stop Software Upgrade) it's not recomended from the field to consider as Hitless with MX as single spine, So will not use this ISSU/NSSU approach. 

# 4. API schema changes
New property (device-image-name) will be added to store the actual image file name as part of the upload.


<xsd:element name="device-image-name" type="xsd:string"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('device-image-name', 'device-image', 'required', 'CRUD',
              'Name of the device image. As of now, for Juniper devices, it will be used by device manager during image upgrade to send additional flag for vmhost based REs.') -->

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
There will not be any change in the workflow however new filed device-image-name added to store the actual name of the image.

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
The existing framwork of the hitless upgrade will be used as it is. L2 (Layer 2/MAC/Switching) related checks are different from the QFX so additional temples with requied rpcs will be added to handle the same.

# 10. Device specifics
# 10.1 
MX devices comes with dual RE, we will have additional steps during the upgrade as follows,

    Step 1: Disable NSR (Non Stop Routing), Graceful Switchover

    Step 2: Upgrade Backup
    Step 3: Switch over
    Step 4: Upgrade Master

    Step 5: Enable NSR (Non Stop Rouring) and Graceful Switchover

# 10.2 
MX boxes supports the VM based Routing Engines so the commands are different,
    request vmhost software add   /var/tmp/junos-vmhost-install-ptx-x86-64-15.1F5-S2.8.tgz
However as we are using the ansible-modules (Juniper.junos) we will be passing vmhost: true flag to differentiate with VM Host based REs.

#### Supported Junos Software Release 
MX devices running with 18.4R2-S3 or above are supported for this functionality.

#### Supported Junos MX models
We are going test with the MX240/480/960, MX100003 models available in our lab.

# 11. Performance and scaling impact
Compared to the QFX boxes it will take more time to complete the upgrade as we need to perform one RE at a time. Looking for the options to see if we can upgrade both the RE's same time.

# 12. Upgrade
N/A

# 13. Deprecations
N/A

# 14. Dependencies
NA

# 15. Testing
Below is the link to the test plan,
https://docs.google.com/spreadsheets/d/1q2P20Uwt0g48UHYqfqaA0Q9qrx3PtRzXDBAjV6ThcNM/edit?usp=sharing

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 17. References
Hitless Upgrade Spec for the QFX: https://github.com/Juniper/contrail-specs/blob/master/5.1/hitless_image_upgrade.md
Jira User Story: https://contrail-jws.atlassian.net/browse/CEM-6362
Image Upgrade of VM Host based RE: https://www.juniper.net/documentation/en_US/junos/topics/concept/installation_upgrade.html

# 18. Caveats or Known issues
NA
