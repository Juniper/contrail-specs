# 1. Introduction
RMA stands for Return Material Authorization. If a hardware failure is
determined by the JTAC engineer and repair/replacement is needed, an RMA will
be created. This RMA provides the customer with the ability to return the
specified defective unit for replacement in line with the purchased
SLA (Service Level Agreement).

# 2. Problem statement
As of today there is no support for RMA via Contrail Command. There is a need
for putting the RMA workflow in place to support the repair/replacement of
Juniper devices. The user should be able to just enter the serial number of the
new device and be able to replace it in place of the old device seamlessly.

#### Usecase 1:
Customer should be able to RMA a device that has been on-boarded
using the brownfield workflow. In this scenario the underlay configuration
is not managed by Contrail.

#### Usecase 2:
Customer should be able to RMA a device that has been on-boarded
using the greenfield workflow. In this scenario both the underlay and the
overlay configuration is managed by Contrail.

# 3. Proposed solution
The RMA support in Contrail will include the following changes/additions:

1) To provide support for RMA, the device underlay configuration is backed up
before any configuration is pushed onto it from Contrail during device
on-boarding step. This is required only in the brownfield workflow.
2) The user will be able to put the device into 'RMA' state by selecting the
'RMA device' action in the Fabric Devices page.
3) Once the replacement device is in place and wired into the network,
the user can select the action 'Reactivate'. The user will then be prompted
to enter in the serial number of the new device if he has not entered it yet.
The user can enter the serial number either by editing the device directly or
after selecting the device to be reactivated. Once this is done, Contrail
will internally push the backed up configurations (in case of brownfield
usecase) and also automatically apply the same roles as the earlier device.

#### Notes:
1) When the device is being replaced with the new one the management IP address,
underlay configuration, ASN number, loopback addresses must not be changed. The
old values should be retained.
2) The new device should be of the same model and os version as the old one.
3) The new device should be wired exactly the same as the old one into the
network.
4) The out of band configuration changes (the configuration changes done
manually) on the device after the configuration has been backed up during the
onboarding step will not be backed up. Backup/restore of out of band changes
is not supported.


# 4. User workflow impact
The following screenshots capture the use visible changes

#### - Add "RMA device" to Fabric devices -> Action

<img src="images/RMA_device_blueprint_1.png">

#### - Pop up warning dialog for above action

<img src="images/RMA_device_blueprint_2.png">

#### - Add "RMA" state to Fabric Device edit screen

<img src="images/RMA_device_blueprint_3.png">

#### - Add "reactivate" to Fabric devices -> Action

<img src="images/RMA_device_blueprint_4.png">

#### - Pop up reactivate dialog box with serial number input

<img src="images/RMA_device_blueprint_5.png">

# 5. Alternatives considered
#### Describe pros and cons of alternatives considered

# 6. API schema changes

#### Reactivation Workflow
There will be a new job template called "reactivate_device_template" with the following json input schema:

```
  "input_schema": {
    "title": "reactivate device",
    "$schema": "http://json-schema.org/draft-06/schema#",
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "serial_number": {
        "type": "string"
      }
    }
  }
```

The input for the execute-job for device reactivation will look like this:

```
{
  "job_template_fq_name": ["default-global-system-config", "reactivate_device_template"],
  "input": {
    "serial_number": <serial_number_str>
  }
  "device_list": [<device_uuid>]
}
```


#### Physical Router Changes
The physical_router object will have the following schema additions:

- new values for physical_router.state field: "RMA", "reactivate"
(TBD) There is no "state" variable on physical_router object. Is this the UVE state?

# 7. UI changes
#### - Add "RMA device" to Fabric devices -> Action

#### - Pop up warning dialog for above action

#### - Add "RMA" state to Fabric Device edit screen

#### - Add "reactivate" to Fabric devices -> Action

#### - Pop up reactivate dialog box with serial number input


# 8. Notification impact
- (Physical router UVE state changes, "RMA" and "Reactivate")

# 9. Provisioning changes
(None)

# 10. Implementation

## 10.1 Data model changes
- new values for physical_router.state field: "RMA", "reactivate"
(TBD) There is no "state" variable on physical_router object. Is this the UVE state?

## 10.2 Save brownfield configuration during device import

- Brownfield only: During device import, underlay configuration is always read 
from the device and stored in the VNC database,
either on the physical_router object or new object (TBD)
- Need to ensure this brownfield_config is not too big. If it is too big, what to do (TBD)

## 10.3 Put device into RMA state
1. UI changes state of physical_router object to "RMA"
2. Device Manager senses this state change and sets an internal do-not-push flag indicating that 
the configuration for this device should not be pushed

## 10.4 Configure new device serial number
1. If desired, enter the serial number of the new device on the device information page in the UI.
If the serial number is not set here, it can optionally be set in the next step.

## 10.5 Put device into Reactivate state
1. UI calls reactivate device workflow via execute-job API with device UUID and optional serial_number input
2. Device Manager starts reactivate device job and reactivate Ansible playbook invoked:
   - Read physical_router object from database
   - Update serial_number if provided with input
   - If brownfield device, fetch brownfield_config and push to device
   - If greenfield device, invoke IP assignment task
   - Change physical_router state to "reactivate"
   - Device Manager senses this state change, clears the do-not-push flag, and pushes the overlay config to the device

### 10.5.1 Workflow error handling
- If missing serial_number, fail
- If missing brownfield_config for brownfield device, fail
- If reactivation fails, ensure that the device is in RMA state itself

# 11. Performance and scaling impact

The RMA workflow itself is only expected to be executed one at a time, 
so there is minimal scaling impact anticipated.

Saving brownfield configuration could be impactful if this configuration is large. 
Performance could be impacted when saving the configuration during device import.
Performance could also be impacted during reactivation if the device.

# 12. Upgrade
(None)

# 13. Deprecations
(None)

# 14. Dependencies
(None)

# 15. Testing
#### Unit tests
#### Dev tests
#### System tests

# 16. Documentation Impact
Requires new section on RMA

# 17. References