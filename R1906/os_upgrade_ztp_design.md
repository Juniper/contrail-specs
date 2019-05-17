# 1. Introduction
ZTP (Zero Touch Provisioning) process allows zeroized factory default devices to be added to a Fabric. Current ZTP
process discovers the devices that aquire dhcp ip and onboards interfaces as part of Fabric discovery.
Additional functionality that is going to be added as part of this process is to give the user the option to upgrade
all the ZTP'ed devices if he desires.

# 2. Problem statement
User wants to be able to upgrade the device's OS during the greenfield fabric onboarding process.

# 3. Proposed solution
Provide an option for Image Upgrade during Greenfield onboarding.

# 4. Alternatives considered
#### Describe pros and cons of alternatives considered

# 5. API schema changes
None

# 6. UI changes
Image upload page-
1. On the Image Upload page, make the Platform field mandatory.
2. Once the image is uploaded and corresponding vnc object for the image is created, add a reference between the platform(hardware vnc object) and image.
3. Add checks to prevent the user from uploading the same image with the same version for a different hardware platform.
Note: Make sure Image->OS Version->Platform is a 1:1:1 mapping

Greenfield fabric discovery page-
1. Add a checkbox for "Image Upgrade"
2. If the user chooses "Image Upgrade", provide a list of OS versions (data available from the Image Upload page) for him to select.
3. This info will be passed in the fabric onboarding job input

job request -
/execute-job
{
	"job_template_fq_name": ["default-global-system-config", "fabric_onboard_template"],
	"input": {
		"fabric_fq_name": ["default-global-system-config", "fab-lag-mh"],
		"fabric_display_name": "fab-lag-mh",
		"node_profiles": [{
			"node_profile_name": "juniper-mx"
		}, {
			"node_profile_name": "juniper-qfx10k"
		}, {
			"node_profile_name": "juniper-qfx10k-lean"
		}, {
			"node_profile_name": "juniper-qfx5k"
		}, {
			"node_profile_name": "juniper-qfx5k-lean"
		}, {
			"node_profile_name": "juniper-srx"
		}],
		"loopback_subnets": ["14.1.1.0/24"],
		"device_count": 1,
		"overlay_ibgp_asn": 64512,
		"fabric_asn_pool": [{
			"asn_min": 64000,
			"asn_max": 65000
		}],
		"management_subnets": [{
			"cidr": "2.2.2.0/24",
			"gateway": "2.2.2.1"
		}],
		"fabric_subnets": ["20.1.1.0/24"],
		"device_auth": {
			"root_password": "Embe1mpls"
		},
		"os_version": "18.1R3"
	}
}

# 7. Notification impact
Three new device states will be added, "IMAGE_UPGRADING", "IMAGE_UPGRADE_SUCCESSFUL", "IMAGE_UPGRADE_FAILED"

# 8. Provisioning changes
None

# 9. Implementation
GREENFIELD-
Schema-
Fabric onboard input schema to accomodate OS version input.
{
  "input_schema": {
    "title": "fabric info",
    "$schema": "http://json-schema.org/draft-06/schema#",
    "type": "object",
    "additionalProperties": false,
    "required": [
      "fabric_fq_name",
      "management_subnets",
      "loopback_subnets",
      "fabric_subnets",
      "fabric_asn_pool",
      "device_auth",
      "node_profiles"
    ],
    "properties": {
      "fabric_fq_name": {
        "type": "array",
        "items": {
          "type": "string"
        }
      },
      "fabric_display_name": {
        "type": "string"
      },
      "management_subnets": {
        "type": "array",
        "items": {
          "type": "object",
          "description": "List of the management network subnets for the fabric",
          "additionalProperties": false,
          "required": [
            "cidr",
            "gateway"
          ],
          "properties": {
            "cidr": {
              "type": "string",
              "pattern": "^([0-9]{1,3}\\.){3}[0-9]{1,3}(\/([0-9]|[1-2][0-9]|3[0-2]))?$"
            },
            "gateway": {
              "type": "string",
              "format": "ipv4"
            }
          }
        }
      },
      "loopback_subnets": {
        "type": "array",
        "items": {
          "type": "string",
          "description": "List of the subnet prefixes that fabric device loopback ips can be allocated.",
          "pattern": "^([0-9]{1,3}\\.){3}[0-9]{1,3}(\/([0-9]|[1-2][0-9]|3[0-2]))?$"
        }
      },
      "fabric_subnets": {
        "type": "array",
        "items": {
          "type": "string",
          "description": "List of the subnet prefixes that could be carved out for the p2p networks between fabric devices.",
          "pattern": "^([0-9]{1,3}\\.){3}[0-9]{1,3}(\/([0-9]|[1-2][0-9]|3[0-2]))?$"
        }
      },
      "fabric_asn_pool": {
        "type": "array",
        "items": {
          "title": "eBGP ASN Pool for fabric underlay network",
          "type": "object",
          "description": "List of the ASN pools that could be used to configure the eBGP peers for the IP fabric",
          "properties": {
            "asn_min": {
              "type": "integer"
            },
            "asn_max": {
              "type": "integer"
            }
          },
          "required": [
            "asn_min",
            "asn_max"
          ]
        }
      },
      "overlay_ibgp_asn": {
        "type": "integer",
        "title": "iBGP ASN for Contrail overlay network",
        "default": 64512
      },
      "device_auth": {
        "title": "Device Auth",
        "type": "object",
        "additionalProperties": false,
        "required": [
          "root_password"
        ],
        "properties": {
          "root_password": {
            "type": "string"
          }
        }
      },
      "node_profiles": {
        "type": "array",
        "items": {
          "title": "Node profile",
          "type": "object",
          "description": "List of node profiles to be used by the fabric.",
          "additionalProperties": false,
          "properties": {
            "node_profile_name": {
              "type": "string"
            },
            "serial_nums": {
              "type": "array",
              "description": "Optional list of serial numbers of fabric devices that we want to associate with this node profile.",
              "items": {
                "type": "string"
              }
            }
          },
          "required": [
            "node_profile_name"
          ]
        }
      },
      "interface_filters": {
        "type": "array",
        "items": {
          "type": "object",
          "maxProperties": 2,
          "additionalProperties": false,
          "properties": {
            "op": {
              "enum": [
                "regex"
              ]
            },
            "expr": {
              "type": "string"
            }
          },
          "title": "filter object",
          "description": "filter object having op and expr fields",
          "default": {},
          "examples": [
            {
              "op": "regex",
              "expr": "^ge-"
            },
            {
              "op": "regex",
              "expr": "^xe"
            }
          ]
        }
      },
      "import_configured": {
        "type": "boolean",
        "default": false,
        "description": "Not importing configured interfaces by default. Set this option to true if configured interfaces need to be imported as part of onboarding."
      },
      "device_count": {
        "title": "Number of fabric devices",
        "type": "integer",
        "description": "Total number of devices in the fabric that needs to be zero-touch provisioned"
      },
      "supplemental_day_0_cfg": {
        "title": "List of day 0 config that can be used to supplement the ZTP config",
        "type": "array",
        "items": {
          "type": "object",
          "required": [
            "name",
            "cfg"
          ],
          "properties": {
            "name": {
              "title": "name for the config",
              "type": "string"
            },
            "cfg": {
              "title": "config blob that contains the supplemental day-0 config",
              "type": "string"
            }
          }
        }
      },
      "device_to_ztp": {
        "title": "List of device serial numbers for the devices to be ZTPed",
        "type": "array",
        "items": {
          "type": "object",
          "required": [
            "serial_number"
          ],
          "properties": {
            "serial_number": {
              "title": "Device serial number",
              "type": "string"
            },
            "supplemental_day_0_cfg": {
              "title": "Name of the supplemental day-0 config that can be optionally specified for this device",
              "type": "string"
            }
          }
        }
      },
      "os_version": {
        "type": "string"
      }
    }
  }

Fabric onboarding Workflow-
Current tasks:
- DHCP discovery
- Device discovery
- Static IP assignment
- Role discovery
- Interface import
- Topology discovery

Image Upgrade task will occur after Device discovery. It will be a device level job which makes the image upgrade process run parallelly on all the devices.
Device discovery spits out a list of devices that have been created, with all the relevant device info. [vendor, family, platform, serial number, hostname]
Image upgrade task will find the image associated to the device s platform, and if the input OS version and the image OS version match, it will upgrade the device.

Per device states are updated appropriately:
"IMAGE_UPGRADING" - When the task is in progress
"IMAGE_UPGRADE_SUCCESSFUL" - If image upgrade is successful
"IMAGE_UPGRADE_FAILED" - If image upgrade failed

On failure, the same is indicated in the job logs and UVEs. Mark the device state and continue with the workflow.

BROWNFIELD-
No change.

# 10. Performance and scaling impact
None

# 11. Upgrade
None

# 12. Deprecations
None

# 13. Dependencies
None

# 14. Testing
#### Unit tests
#### Dev tests
#### System tests

# 15. Documentation Impact

# 16. References
