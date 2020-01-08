# 1. Introduction
This blueprint describes discovery of bare metal servers (BMS) by
Contrail Telemetry module.

# 2. Problem statement
Once BMSs are added in Contrail Command, user should be able to see
those devices and connections in Contrail Command UI in Fabrics -> Topology
View section, and monitor traffic.

# 3. Proposed solution
Contrail telemetry module will poll Contrail Config API Server to see if any BMS
has been added or deleted. If so, it notifies network device adapter plugin of
telemetry module to update the database and a new server entry is created using
the BMS name. When the device connections are derived using LLDP, then this BMS
connection will be added.

# 4. API schema changes
Each network device will have connection details with BMSs. The schema is
as below:

```
class NetworkDeviceBMSConnectionsConfig:
    resource_fields = {
        'NetworkDeviceId': fields.String,
        'ConnectionInfo': fields.List(
            fields.Nested(NetworkDeviceConnectFields.resource_fields)),
    }
```

# 5. Alternatives considered
None

# 6. UI changes / User workflow impact
None

# 7. Notification impact
N/A

# 8. Provisioning changes
None

# 9. Implementation
Contrail telemetry module polls Contrail Config API Server using below REST API
`/virtual-machine-interfaces?fields=virtual_machine_interface_bindings`
to check if any new BMS is added or existing BMS got deleted. If there
is any change, it notifies network device adapter plugin of telemetry
module to update the database.
If a new BMS is added, a new server entry is created with BMS name in Contrail
telemetry module database.
If a BMS is deleted, then the server entry from Contrail telemetry module
database gets deleted.

In Contrail Command UI, a BMS can be added using two workflows:
### 9.1 Workloads -> Instances
We get remote connetion details with the BMS name and port id, using that
information, Contrail telemetry module builds the BMS connection data for this
network device, and creates server entry of the BMS.
### 9.2 Overlay -> Virtual Port Group (VPG)
We do not get remote connection details. We get only the local interface details
of the network device. In this case, we will create a new BMS server using name as
`bms_vpg_<vpg_name>` and remote interface name as `interface`

# 10. Performance and scaling impact
N/A

# 11. Upgrade
N/A

# 12. Deprecations
N/A

# 13. Dependencies
N/A

# 14. Security Considerations
N/A

# 15. Testing
#### Unit tests
#### Dev tests
#### System tests

# 16. Documentation Impact

# 17. References
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-9278
