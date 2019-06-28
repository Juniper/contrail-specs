# 1. Introduction
When provisioning the fabric in the greenfield scenario the user must be able
to provide a selective set of devices and only these devices should be ZTPed.
The device list can be provided by the user via the UI using a JSON or Yaml 
file

This is related to the jira user story:
https://contrail-jws.atlassian.net/browse/CEM-9

# 2. Problem statement
Currently selective ZTP is already enabled via exposing an option for the user
to upload a yaml file in the fabric on-boarding UI. In this file the list of 
devices and any supplementing config snippets required to be pushed to the
device during ZTP can be provided. 

The problem with the existing implementation is that the ZTP process uses the
device serial number as the hostname. There is no option for the user to 
specify the hostnames.

# 3. Proposed solution
Will enable user to specify the hostname in the yaml file used to specify the
device list for ZTP. This hostname will be an optional field and when not
specified, the serial number will be used instead.

The hostname provided by the user should comply to the valid hostname rules.
Each element of the hostname must be from 1 to 63 characters long and the 
entire hostname, including the dots, can be at most 253 characters long.
Valid characters for hostnames are ASCII(7) letters from a to z, the digits
from 0 to 9, and the hyphen (-).  A hostname may not start with a hyphen.

The YAML file snippet is provided below:

```yaml
supplemental_day_0_cfg:
  - name: 'cfg1'
    cfg: |
      set system ntp server 167.99.20.98
device_to_ztp:
  - serial_number: 'DK588'
    supplemental_day_0_cfg: 'cfg1'
    hostname: '5a12-qfx5'
  - serial_number: 'VF3717350117'
    hostname: '5a12-qfx9'
```

# 4. User workflow impact
The user workflow is not impacted apart from the fact that the user can now
provide the hostname too while providing the devices to be ZTPed.

# 5. References
