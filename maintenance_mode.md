# 1. Introduction
Network devices need occasional maintenance windows to perform image upgrades,
swap line cards, etc. The existing hitless image upgrade workflow provides
a process in which devices are first placed into a maintenance mode, the
device image is upgraded, and then the device is taken out of maintenance mode.
We want to have the additional flexibility to take a device in and out of
maintenance mode without upgrading the image.


# 2. Problem statement
There is currently no way to place a device into maintenance mode without
also upgrading the image.

# 3. Proposed solution
We will provide two new workflows, one to enter maintenance mode and one to
exit maintenance mode. At this time, we are only supporting one device per
workflow. However, it is possible to execute multiple workflows in sequence.

1) Activate-Maintenance-Mode Workflow
- Input: fabric_uuid, device_uuid, mode (test, activate), advanced_parameters.
- Perform health checks on the device. Test-only phase.
- If any health checks fail, edit advanced_parameters as needed, or fix issues in the network,
 until health checks pass.
- Activate maintenance mode. This sends configuration to the device to divert
traffic away.
- Device status changes to "Maintenance Mode"

2) Deactivate-Maintenance-Mode Workflow
- Remove maintenance-mode configuration from device, allowing traffic to once again flow through.
- Run another health check
- Device status changes back to "Active"

# 4. User workflow impact
The steps involved in the UI are similar to the hitless image upgrade workflow.
Therefore, it is desirable to emulate that same workflow here as much as possible.
This will give the UI a similar look and feel between operations and allow users
of the hitless image upgrade feature to easily navigate maintenance mode.

Hitless image upgrade uses a wizard to guide the user through the health check phase.
This is important because the user may need to go back and forth between the 
advanced_parameters page and the test run itself. Since activating
maintenance mode also runs these same health checks, it is important to
provide this same capability, at least for activate maintenance mode.
Deactivating maintenance mode may not need a wizard, but it would be useful
to display the job logs in progress.

1) Activate-Maintenance-Mode

Unlike hitless image upgrade, maintenance mode activation does not require the
user to select an image and a list of devices for upgrade.
In this case, we only need to select a device. Here are the sequence of steps.

- Select device and initiate "activate maintenance mode" action
- Screen1: Parameters page. Show advanced_parameters just like with hitless image upgrade
and allow the user to modify these parameters if desired. "Next" button at the bottom
will trigger UI to call activate-maintenance-mode workflow with mode=test_run.

#### - Screen1: Parameters Page

<img src="images/maintenance_mode_1.png" alt="drawing" height=700 width="1000"/>

- Screen2: Job progress page. Show job progress of test run just as with hitless image upgrade.
We have a choice here of whether or not to show the device state in one window along
with the job status in another window. While displaying the device state is optional, it is a
nice summary for the user to view. So it is recommended to display device state in addition
to job logs.
"Previous" button allows the user to go back to Screen1 in order to modify parameters.
"Next" button will trigger UI to call activate-maintenance-mode workflow with mode=activate.
A warning dialog should display warning the user, similar to hitless image upgrade, when clicking the Next button.
- If the test-run fails, the user should be given the option of going to the previous page and modifying
the advanced_parameters and running the test again.
- If the test-run succeeds, the user should additionally be given the option of going to the next
page, which starts the activate workflow.
- Note: The sample image below was created from the existing hitless image upgrade workflow,
which only shows job logs during the test run. So this image is an example of what Screen2
could look like without showing device status.

#### - Screen2: Job Progress Page - Test-Run

<img src="images/maintenance_mode_2.png" alt="drawing" height=700 width="1000"/>

- Screen3: Job progress page. Show job progress of activate workflow, same as with
hitless image upgrade. "Finish" button will quit wizard. As with Screen2 above, it is optional
but recommended to display device status along with the job logs. The sample image below is an example which shows
device status.
- If maintenance mode activation fails, the user can simply click the Finish button and start over again (should rarely fail).
- If maintenance mode activation succeeds, the user simply clicks Finish button to exit.

#### - Screen3: Job Progress Page - Activate

<img src="images/maintenance_mode_3.png" alt="drawing" height=700 width="1000"/>

- Once the device is in maintenance mode, the device status shown on the main fabric devices summary page
should change from "Active" to "Maintenance Mode" (or a short version like "MM").
 

2) Deactivate-Maintenance-Mode

- Select device and initiate "deactivate maintenance mode" action.
- Screen4: Job progress page. Show job progress of deactivate workflow.
This page is similar to Screen3 of the activate workflow.
This will show the results of the last health check. In this case, however,
the user cannot modify advanced_parameters, so there is no need to display
the parameters page. "Finish" button will quit wizard. As with the activate
workflow, it is optional but recommended to display device status along with the job logs.

#### - Screen4: Job Progress Page - Deactivate

<img src="images/maintenance_mode_4.png" alt="drawing" height=700 width="1000"/>

# 5. Alternatives considered
(None)

# 6. API schema changes

#### Physical Router "Maintenance-Mode" managed-state
We will add a new state for the physical_router.physical_router_managed_state attribute.
The new state will be called "maintenance". The physical_router.physical_router_managed_state
should be treated as a read-only attribute by the UI in the maintenance-mode workflow.
Only the backend will modifiy this attribute.

#### Maintenance Mode Activate Workflow

There will be a new job template called "maintenance_mode_activate_template" with the following json input schema:

```
   "input_schema":{
      "title": "Activate maintenance mode",
    "$schema": "http://json-schema.org/draft-06/schema#",
    "type": "object",
    "additionalProperties": false,
    "required": ["fabric_uuid", "device_uuid"],
    "properties":{
        "fabric_uuid":{
            "format":"uuid",
            "type":"string",
            "title":"Fabric UUID",
            "description":"UUID of the device fabric"
        },
        "device_uuid":{
            "format":"uuid",
            "type":"string",
            "title":"Device UUID",
            "description":"UUID of the device to enter maintenance mode"
        },
        "mode": {
          "enum": ["test_run", "activate"],
          "description": "Mode in which to run workflow"
        },
        "advanced_parameters": {
          "title": "Advanced parameters",
          "description": "Optional parameters used to override defaults",
          "type": "object",
          "additionalProperties": false,
          "default": {},
          "properties": {
            "bulk_device_upgrade_count": {
              "type": "integer",
              "description": "Maximum number of devices to upgrade simultaneously",
              "default": 4
            },
            "health_check_abort": {
              "type": "boolean",
              "description": "Enable/disable abort upon health check failures",
              "default": true
            },
            "Juniper": {
              "type": "object",
              "additionalProperties": false,
              "default": {},
              "properties": {
                "bgp": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "bgp_flap_count": {
                      "type": "integer",
                      "description": "Number of flaps allowed for BGP neighbors",
                      "default": 4
                    },
                    "bgp_flap_count_check": {
                      "type": "boolean",
                      "description": "Enable/disable bgp_flap_count check",
                      "default": true
                    },
                    "bgp_down_peer_count": {
                      "type": "integer",
                      "description": "Number of down peers allowed",
                      "default": 0
                    },
                    "bgp_down_peer_count_check": {
                      "type": "boolean",
                      "description": "Enable/disable bgp_down_peer_count check",
                      "default": true
                    },
                    "bgp_peer_state_check": {
                      "type": "boolean",
                      "description": "Enable/disable bgp peer state check",
                      "default": true
                    }
                  }
                },
                "alarm": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "system_alarm_check": {
                      "type": "boolean",
                      "description": "Enable/disable system alarm check",
                      "default": true
                    },
                    "chassis_alarm_check": {
                      "type": "boolean",
                      "description": "Enable/disable chassis alarm check",
                      "default": true
                    }
                  }
                },
                "interface": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "interface_error_check": {
                      "type": "boolean",
                      "description": "Enable/disable interface error check",
                      "default": true
                    },
                    "interface_drop_count_check": {
                      "type": "boolean",
                      "description": "Enable/disable interface drop check",
                      "default": true
                    },
                    "interface_carrier_transition_count_check": {
                      "type": "boolean",
                      "description": "Enable/disable interface carrier transition check",
                      "default": true
                    }
                  }
                },
                "routing_engine": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "routing_engine_cpu_idle": {
                      "type": "integer",
                      "description": "Routing engine CPU idle time",
                      "default": 60
                    },
                    "routing_engine_cpu_idle_check": {
                      "type": "boolean",
                      "description": "Enable/disable routing engine CLU idle time check",
                      "default": true
                    }
                  }
                },
                "fpc": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "fpc_cpu_5min_avg": {
                      "type": "integer",
                      "description": "FPC CPU 5 minute average utilization",
                      "default": 50
                    },
                    "fpc_cpu_5min_avg_check": {
                      "type": "boolean",
                      "description": "Enable/disable FPC CPU 5 minute average utilizationcheck",
                      "default": true
                    },
                    "fpc_memory_heap_util": {
                      "type": "integer",
                      "description": "FPC memory heap utilization",
                      "default": 45
                    },
                    "fpc_memory_heap_util_check": {
                      "type": "boolean",
                      "description": "Enable/disable FPC memory heap utilization check",
                      "default": true
                    }
                  }
                },
                "lacp": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "lacp_down_local_check": {
                      "type": "boolean",
                      "description": "Enable lacp interface status check on target device",
                      "default": true
                    },
                    "lacp_down_peer_check": {
                      "type": "boolean",
                      "description": "Enable lacp interface status check on peer device",
                      "default": true
                    }
                  }
                },
                "active_route_count_check": {
                  "type": "boolean",
                  "description": "Enable/disable active route count check",
                  "default": true
                },
                "l2_total_mac_count_check": {
                  "type": "boolean",
                  "description": "Enable/disable l2 total mac count check",
                  "default": true
                },
                "storm_control_flag_check": {
                  "type": "boolean",
                  "description": "Enable/disable storm control flag check",
                  "default": true
                }
              }
            }
          }
        }
      }
   }
```

#### Maintenance Mode Deactivate Workflow

There will be a new job template called "maintenance_mode_deactivate_template" with the following json input schema:

```
   "input_schema":{
      "title": "Deactivate maintenance mode",
    "$schema": "http://json-schema.org/draft-06/schema#",
    "type": "object",
    "additionalProperties": false,
    "required": ["fabric_uuid", "device_uuid"],
    "properties":{
        "fabric_uuid":{
            "format":"uuid",
            "type":"string",
            "title":"Fabric UUID",
            "description":"UUID of the device fabric"
        },
        "device_uuid":{
            "format":"uuid",
            "type":"string",
            "title":"Device UUID",
            "description":"UUID of the device to exit maintenance mode"
        },
        "advanced_parameters": {
          "title": "Advanced parameters",
          "description": "Optional parameters used to override defaults",
          "type": "object",
          "additionalProperties": false,
          "default": {},
          "properties": {
            "bulk_device_upgrade_count": {
              "type": "integer",
              "description": "Maximum number of devices to upgrade simultaneously",
              "default": 4
            },
            "health_check_abort": {
              "type": "boolean",
              "description": "Enable/disable abort upon health check failures",
              "default": true
            },
            "Juniper": {
              "type": "object",
              "additionalProperties": false,
              "default": {},
              "properties": {
                "bgp": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "bgp_flap_count": {
                      "type": "integer",
                      "description": "Number of flaps allowed for BGP neighbors",
                      "default": 4
                    },
                    "bgp_flap_count_check": {
                      "type": "boolean",
                      "description": "Enable/disable bgp_flap_count check",
                      "default": true
                    },
                    "bgp_down_peer_count": {
                      "type": "integer",
                      "description": "Number of down peers allowed",
                      "default": 0
                    },
                    "bgp_down_peer_count_check": {
                      "type": "boolean",
                      "description": "Enable/disable bgp_down_peer_count check",
                      "default": true
                    },
                    "bgp_peer_state_check": {
                      "type": "boolean",
                      "description": "Enable/disable bgp peer state check",
                      "default": true
                    }
                  }
                },
                "alarm": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "system_alarm_check": {
                      "type": "boolean",
                      "description": "Enable/disable system alarm check",
                      "default": true
                    },
                    "chassis_alarm_check": {
                      "type": "boolean",
                      "description": "Enable/disable chassis alarm check",
                      "default": true
                    }
                  }
                },
                "interface": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "interface_error_check": {
                      "type": "boolean",
                      "description": "Enable/disable interface error check",
                      "default": true
                    },
                    "interface_drop_count_check": {
                      "type": "boolean",
                      "description": "Enable/disable interface drop check",
                      "default": true
                    },
                    "interface_carrier_transition_count_check": {
                      "type": "boolean",
                      "description": "Enable/disable interface carrier transition check",
                      "default": true
                    }
                  }
                },
                "routing_engine": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "routing_engine_cpu_idle": {
                      "type": "integer",
                      "description": "Routing engine CPU idle time",
                      "default": 60
                    },
                    "routing_engine_cpu_idle_check": {
                      "type": "boolean",
                      "description": "Enable/disable routing engine CLU idle time check",
                      "default": true
                    }
                  }
                },
                "fpc": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "fpc_cpu_5min_avg": {
                      "type": "integer",
                      "description": "FPC CPU 5 minute average utilization",
                      "default": 50
                    },
                    "fpc_cpu_5min_avg_check": {
                      "type": "boolean",
                      "description": "Enable/disable FPC CPU 5 minute average utilizationcheck",
                      "default": true
                    },
                    "fpc_memory_heap_util": {
                      "type": "integer",
                      "description": "FPC memory heap utilization",
                      "default": 45
                    },
                    "fpc_memory_heap_util_check": {
                      "type": "boolean",
                      "description": "Enable/disable FPC memory heap utilization check",
                      "default": true
                    }
                  }
                },
                "lacp": {
                  "type": "object",
                  "additionalProperties": false,
                  "default": {},
                  "properties": {
                    "lacp_down_local_check": {
                      "type": "boolean",
                      "description": "Enable lacp interface status check on target device",
                      "default": true
                    },
                    "lacp_down_peer_check": {
                      "type": "boolean",
                      "description": "Enable lacp interface status check on peer device",
                      "default": true
                    }
                  }
                },
                "active_route_count_check": {
                  "type": "boolean",
                  "description": "Enable/disable active route count check",
                  "default": true
                },
                "l2_total_mac_count_check": {
                  "type": "boolean",
                  "description": "Enable/disable l2 total mac count check",
                  "default": true
                },
                "storm_control_flag_check": {
                  "type": "boolean",
                  "description": "Enable/disable storm control flag check",
                  "default": true
                }
              }
            }
          }
        }
      }
   }
```

# 7. UI changes
See section 4 "User Flow Impact".

# 8. Notification impact
(None)

# 9. Provisioning changes
(None)

# 10. Implementation
