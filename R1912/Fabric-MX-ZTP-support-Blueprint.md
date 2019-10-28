# 1. Introduction
In CEM we onboard Fabric in one of the below mentioned ways.
a. Brownfield (underlay configuration of the fabric devices are managed by admin. CEM only handles the overlay)
b. Greenfield or Zero Touch Deployment (whole fabric management is done by CEM - underlay and overlay)

Both the above Fabric management schemes need be supported on QFX and MX devices.
Currently we : 

                                    QFX          MX
Onboarding scheme

Brownfield(overlay)                 yes          yes
Greenfield(underlay and overlay)    yes          NO

#### Supported Deployment model for MX:
Supported Roles:
- DC-Gateway@spine

#### Supported Device Models for MX:
MX-80, MX80-48t, MX204, MX240, MX480, MX960, MX2008, MX2010, MX2020, MX10003

# 2. Problem statement
As stated in the introduction, we currently donot support Zero Touch Provisioning on MX.

# 3. Proposed solution
Add support for Greenfield fabric onboarding for the below MX platforms-
- MX240
- MX480
Note: Image upgrade during ZTP will NOT be supported in this release.

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
For MX to work in ZTP, user needs to make sure that the boxes are plugged in the ZTP subnet and are zeroized.
Refer to 'Device specifics' section for more details.

From the workflow point of view, there is no change in the onboarding process between QFX and MX.

Fabric onboarding workflow recap-
1. User provides the following input
- Device Info
    supplemental_day_0_cfg:
    - name: 'cfg1'
        cfg: |
        set system ntp server 167.99.20.98
    device_to_ztp:
    - serial_number: 'Juniper-mx240-JN11CCA4FAFC'
      hostname: 'mx-dc1'
      supplemental_day_0_cfg: 'cfg1'
- ZTP subnet/ management subnet (dnsmasq/dhcp server serves IPs from this subnet)
- Fabric subnet (to configure the links between leaves and spines) 
- Loopback subnet (to configure loopback ip)

2. Device manager (DM) generates the customized DHCP config and restarts dnsmasq. 
It also generates the Day 0 config to be committed on the device and pushes it to the TFTP server.

#### Customized DHCP config
dhcp-vendorclass=set:jn,Juniper
dhcp-range=10.1.1.249,10.1.1.253,255.255.255.248,10.1.1.255,12h
dhcp-option=option:router,10.204.219.254
dhcp-option=66,10.1.1.250
dhcp-option=150,10.1.1.250
dhcp-option=tag:jn,encap:43,1,"test_mx_juniper.sh"
dhcp-option=tag:jn,encap:43,3,"tftp"
#### Day 0 config (once the device acquires an IP, it pulls the script from the TFTP server)
#!/bin/sh
serial_number=$(echo "show system information | display xml" | cli | grep serial-number | awk -F">" '{print $2}' | awk -F"<" '{print $1}')
cat <<EOF > /tmp/juniper.config
system {
    host-name "$serial_number"
    root-authentication {
        encrypted-password "\$6\$yfVsxz4Z\$7SUvHhu/geDG9nOUzDmkr7nNpAPEHTT5xID8pyg303UL5WMNvjmoMVDGXtNnSI05fOjm8OcJOSLbbZlwFLOm70";
    }
    services {
        ssh {
            root-login allow;
        }
        telnet;
        netconf {
            ssh;
        }
    }
}
protocols {
    lldp {
        advertisement-interval 5;
        interface all;
    }
}
EOF
cli <<EOF
configure
load merge /tmp/juniper.config relative
commit and-quit
EOF


3. Devices are then discovered, node-profiles are assigned, interfaces are onboarded, hardware inventory is read and lldp info is processed.
4. Roles (physical and overlay role) are assigned, fabric links are configured, loopback IP is configured and the relevant underlay config (IP-Clos) is pushed.

# 10. Device specifics
Pre-requisite for ZTP to work is to make sure the device is zeroized. This is to ensure the dhcp client on the device is up and requesting for an IP.
All MX devices, by default come with DHCP enabled on the fxp0 interface.

Command to zeroize:
request system zeroize

#### Supported Junos Software Release
MX devices running with 18.1R3 or above are supported for this functionality.

# 11. Performance and scaling impact
N/A

# 12. Upgrade
N/A

# 13. Deprecations
N/A

# 14. Dependencies

# 15. Testing
Below is the link to the test plan,
https://docs.google.com/spreadsheets/d/1AYYG48nd5OPiesJmVgQ4w756OfHlUb8YFNBpRwuPcxw/edit?usp=sharing

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 17. References
JIRA Story: https://contrail-jws.atlassian.net/browse/CEM-5406
Zero Touch Provisioning: https://www.juniper.net/documentation/en_US/junos/topics/topic-map/zero-touch-provision.html
Zero Touch Provisioning support by Juniper devices with OS version: https://apps.juniper.net/feature-explorer/parent-feature-info.html?pFName=Zero%20Touch%20Provisioning%20(EZ%20Touchless%20Provisioning%20using%20DHCP)
