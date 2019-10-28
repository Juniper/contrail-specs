# 1. Introduction
Contrail Fabric manager supports the fabric onboard both green field and brownfield. As of 1911 MX devices are supported only from brownfield for spine role with DC-Gateway overlay role.
Users ask is to support the ZTP, which enables the them to use the MX devices as part of the "New fabric onboard" workflow.

# 2. Problem statement

Support of ZTP for MX in EVPN VXLAN fabric. More details can be found under https://contrail-jws.atlassian.net/browse/CEM-5406


# 3. Proposed solution
    As of now (prior to 1912) MX is supported for physical role as spine and DC-Gateway overlay role for the brownfield scenario. 

    From 1912 onwards we will add the support for the MX (MX240, MX480 which we will qualify as part of our testing) for the ZTP.
    
    Supported Roles:
        Physical Role: Spine
        Overlay Role: DC-Gateway

    Hence the users should able to discover the MX devices as part of the "New Fabric Onbard" workflow.
    As part of this support in similar lines to QFX
        1. It will take the devices to ZTP from the user as input.
        2. Creates the DHCP and TFTP configurations and restarts the DHCP service.
        3. Responds to the DHCP request from the MX devices and allocates the IP's.
        4. Configures the required to manage the box - enables ssh, configures lldp, makes the device to allow ssh for root user etc.
        5. Controller will connect to the Device and delete's the DHCP on fxp0 interface and assigns the static IP.
        6. Reads the Hardware inventory, physical and logical interfaces and creates them in the CFM database.
        7. We wil not support the image upgrade as part ZTP in this release (1912), it will be supported in future releases of the CFM.
        8. Configures the underlay IP clos, i.e creates the loopbacks and assigns the ip's from fabric managmenet, provided by the user input.

# 4. User workflow impact

(None)

# 5. API schema changes

(None)

# 6. Implementation

1. DevicestoZTP .yaml format - belwo format is same for both MX and QFX devices.
    supplemental_day_0_cfg:
    - name: 'cfg1'
        cfg: |
        set system ntp server 167.99.20.98
    device_to_ztp:
    - serial_number: 'Juniper-mx240-JN11CCA4FAFC'
        supplemental_day_0_cfg: 'cfg1'

2. Below is the log can be seen in the device during the ZTP. All the MX devices comes by default with the DHCP enabled on the fxp0 interface. 
For the first time when the device is booted the fxp0 interface will broadcasts the DHCP requests as we configure the Contrail Controller as 
DHCP server it will responds to the requests and provides the basic configuration to boot.

    you can see in the below logs - Contrail Controller with IP 10.204.219.250 responded to the fxp0 interface broadcast messages.

    Auto Image Upgrade: DHCP Options for client interface fxp0.0 ConfigFile:       
    test_juniper.sh Gateway: 10.204.219.254 DHCP Server: 10.204.219.250           
    File Server: 10.204.219.250 Options state:                                   
    Partial Options::Config File set,Image File not set,File Server set                                                                               
                                                                                
    Auto Image Upgrade: DHCP Client Bound interfaces: fxp0.0                                                                                
                                                                                
    Auto Image Upgrade: DHCP Client Unbound interfaces:                                                                                
                                                                                
    Auto Image Upgrade: To stop, on CLI apply                                      
    "delete chassis auto-image-upgrade"  and commit                                                                               
                                                                                
    Auto Image Upgrade: Active on client interface: fxp0.0                                                                               
                                                                                
    Auto Image Upgrade: Interface::   "fxp0"                                       
    
    Auto Image Upgrade: Server::      "10.204.219.250"                             
    
    Auto Image Upgrade: Image File::  "NOT SPECIFIED"                              
    
    Auto Image Upgrade: Config File:: "test_juniper.sh"                            
    
    Auto Image Upgrade: Gateway::     "10.204.219.254"                             
    
    Auto Image Upgrade: Protocol::    "tftp"                                       
                                                                                
                                                                                
    Auto Image Upgrade: Start fetching test_juniper.sh file from server 10.204.219.
    250 through fxp0 using tftp                                                    
                                                                                
                                                                                
    Auto Image Upgrade: File test_juniper.sh fetched from server 10.204.219.250 thr
    ough fxp0                                                                      
                                                                                
                                                                                
    Auto Image Upgrade: Executing script test_juniper.sh                           
                                                                                
                                                                                
    Auto Image Upgrade: test_juniper.sh executed successfully. See /var/log/script_
    output                                                                         
                                                                                
                                                                                
    Broadcast Message from root@JN11CCA4FAFC                                       
            (no tty) at 15:18 UTC...                                               
                                                                                
    Auto image Upgrade: Stopped                                    
 
# 7. UI Changes

(None)

# 8. Testing

# 9. Documentation Impact

# 10. References

Below are the references for Junos ZTP.
    1. https://contrail-jws.atlassian.net/browse/CEM-5406
    2. https://www.juniper.net/documentation/en_US/junos/topics/topic-map/zero-touch-provision.html
    3. https://apps.juniper.net/feature-explorer/parent-feature-info.html?pFName=Zero%20Touch%20Provisioning%20(EZ%20Touchless%20Provisioning%20using%20DHCP)
