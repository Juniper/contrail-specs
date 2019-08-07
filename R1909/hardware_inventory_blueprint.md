# 1. Introduction
Junos devices display a list of all Flexible PIC Concentrators (FPCs) and 
PICs installed in the router or switch chassis, including the hardware version
level and serial number. This story involves providing this information for the 
Physical Routers in Contrail Command UI.

# 2. Problem statement
Currently user cannot view the hardware inventory information in the UI.
User should be able to view the details of the 'show chassis hardware' command
in the UI and also be able to pull these latest hardware details from the device
on demand.

# 3. Proposed solution
During the on-boarding of the fabric, hardware inventory for the device will
be fetched and saved in the DB. The User will be able to view this detail for 
the Physical router in the UI. Also User will be able to click a button to 
pull this information from the device in real time.

# 4. API schema changes

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
The User will be able to view the hardware/chassis details in a tabular format
in a newly added tab in the page where the Physical interface and Logical
interfaces are currently displayed. The page will also have a action button 
to update the hardware data in realtime

![Overlay features list](images/hardware_inventory.png)

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
The major implementation modules include:
1) Playbook to read the hardware details from the device using the 
'show chassis hardware' command. This playbook will fetch the data from the 
device in json format and process it to meet the defined schema and 
save the JSON in DB.
2) Upon request from user to fetch the hardware information in real time,
the same playbook will be used to update the chassis data in DB


# 10. Performance and scaling impact
N/A

# 11. Upgrade
N/A

# 12. Deprecations
N/A

# 13. Dependencies
N/A

# 14. Testing
#### Unit tests
#### Dev tests
#### System tests

# 15. Documentation Impact
The feature will have to be documented

# 16. References
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-37
JUNOS documentation: https://www.juniper.net/documentation/en_US/junos/topics/reference/command-summary/show-chassis-hardware.html