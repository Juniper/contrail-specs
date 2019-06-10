# 1. Introduction
Tungsten Fabric Manager (TFM) supports discovery of network devices. With
Tungsten vCenter Fabric Manager (TVFM), TFM will be able to manage network devices
connected to ESXi Hosts. TVFM needs to know the ESXi-Host/Switch connection
details and the Distrituted Virtual Switch(DVS) details for ports/vmnics.

# 2. Problem Statement
To configure Network devices, TVFM needs to know the ESXi-Hosts, its port
details, network connectivity for ports, and DVS name associated to the ports.
As of now, these details can be imported via YAML/JSON import. but it becomes
cumbersome/error-prone for admin to generate yaml/json file and then import
that file.


# 3. Proposed Solution
In order to address the above problem, we propose the following solution for
discovering 

Develop the vCenter import tool will:
* Take vCenter credentials, host, data-center name, and other job-manager related
  values as input
* Read ESXi-Host, its ports details(name,mac-address, LLDP info and DVS-name
  if any)
* Read and Compare ESXi details with Contrail-Command nodes based on
  hostname/ports. 
* Post the updated details to the Contrail-Command or Create new nodes if
  doesn't exist.

## 3.1 Alternatives considered
* Adding vCenter import binary into the contrail-command. This was rejected
  as:
  1. this will cause multiple UI changes for vCenter job.
  2. any changes in contrail-command agents framework would have required
     changes/updates to this as well.
* Develop the ansible-playbook for vCenter import job. This would require to
  make use to ansible even though ansible plays no role into actual work as
  vCenter import job interacts only to vCenter and Contrail-Command


## 3.2 API schema changes
Update job-template schema to API Server. This is needed for the details of
executable that will be called by the job-manager. Currently
job-template-playbooks is there, overloading this for executable template
won't be good idea. so adding a new field/dict for executable.

1. node: This resource will contain only hostname and will be referenced by
ports.

Job-Template-Executables{
- *executable-path*: string
- *executable-args* : string
- *job-completion-weightage*: integer
}

2. JobTemplateType: Add one more type for executable

## 3.3 User workflow impact
No Impact on current user-workflows

## 3.4 UI Changes
UI need to allow user-input for following on Import Page (or Import vCenter
page), following details might have been already available with cluster, admin
might have provided during setting up cluster:
1. vCenter hostname/Ip-Address
2. vCenter Credentials
3. Datacenter to Import

# 4. Implementation
For implementing the above requirements, it is proposed to make use of the
enhance job template infra and develop a executable.

Changes to Job-Manager:
Current job-manager doesn't support calling executable binaries direclty
instead it forces to use ansible-playbooks. Job-Manager framework needs to be
enhanced to allow calling executables from job-manager.

1. All user-input will passed to job_ctx.job_input.
2. Executable would write logs/status a file and that will be read by
   job-manager to update UVE
3. Executable will make use of govmomi library by vmware to read vCenter
   database.

# 5. Performance and scaling impact
No Performance and Scaling impact will be there as we are mostly reading from
thirdparty software.

# 6. Upgrade
Uprade from previously should not have any impact as there will be new addition to
schema (vnc schema and Tungsten Fabric schema).

# 7. Deprecations
N/A
# 8. Dependencies

# 9. Testing
## 9.1 Unit tests

## 9.2 Dev tests
    1. Provide valid valid vcenter host and credential details, executable
       should be able to import esxi-hosts from datacenter to Contrail-Command
    2. On enabling LLDP-Listen on DVS, switch connection details should also
       be imported to contrail-command.

# 10. Documentation Impact
    1. vCenter import needs to be documentsed.

# 11. References

# 12. Not Supported features for current release

