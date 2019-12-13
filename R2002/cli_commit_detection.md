# 1. Introduction
Contrail will now be able to detect CLI-user generated config in fabric
devices and provide the user an option to either accept and incorporate them as 
part of Contrail config in a seperate apply-group or reject and delete the config from the device. 
This spec covers the design and implementation of providing the above capability to a Contrail user. 

# 2. Problem statement
Currently for any device managed by Contrail, configuration that is injected via CLI
is ignored and Contrail has no knowledge of the changes made to the device.
The user needs to be aware of these changes to manage the device correctly. 

#### Use cases
The main use case is when a Contrail user wants to check the CLI injected changes done by
a same/different user and have the ability to either accept or reject chose changes. 

# 3. Proposed solution
Contrail fabric will now detect user generated CLI configurations 
made before pushing any other new config down to a particular device. 
Information regarding this change such as the user who made the change, time, 
and the config diff is dispalyed to the Contrail user so that he can make the decision 
to either accept or reject the change. 
Depending on the option selected, the following changes take place. 

Options and Results:
- Approve - The committed config will be put into a seperate group back on the device.
The config is also stored in the database for futher use case like when the device is RMA'd.

- Reject - The CLI committed config will be deleted from the device. The diff
will not be saved in the database.

# 4. User workflow impact

1) At any time before contrail tries to push new config down to the device, 
a check to see if there are any CLI-user committed config is first made. 
2) If a change is detected, it is stored in the database and the device is
flagged to be out of sync. UI will notify the user about this in the fabric page
containing a list of devices. 
3) User can then use a workflow to either approve or reject the changes, per device. 

# 5. API schema changes
We introduce a new VNC db object called 'cli-configs' to store the CLI-injected changes on the device. The API schema changes are below. 

<xsd:simpleType name="CliCommitStatus">
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="diffs_pending"/>
         <xsd:enumeration value="commit_failure"/>
         <xsd:enumeration value="commit_success"/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="CliDiffInfoType">
    <xsd:all>
        <xsd:element name="username" type="xsd:string"/>
        <xsd:element name="time" type="xsd:string"/>
        <xsd:element name="config_changes" type="xsd:string"/>
        <xsd:element name="status" type="xsd:CliCommitStatus"/>
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="CliDiffListType">
    <xsd:all>
        <xsd:element name="commit-diff-info" type="CliDiffInfoType" maxOccurs="unbounded"/>
    </xsd:all>
</xsd:complexType>

<xsd:element name="cli-config" type="ifmap:IdentityType"/>

<xsd:element name="physical-router-cli-config"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('physical-router-cli-config',
          'physical-router', 'cli-config', ['has'], 'optional', 'CRUD',
          'CLI commits done on a physical router.') -->
<xsd:element name="accepted-cli-config" type="xsd:string"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('accepted-cli-config', 'cli-config', 'optional', 'CRUD',
              'Aggregated cli accepted configs. This config will be pushed when the device undergoes RMA along with Contrail configuration') -->
<xsd:element name="commit-diff-list" type="CliDiffListType"/>
<!--#IFMAP-SEMANTICS-IDL
          ListProperty('commit-diff-list', 'cli-config', 'optional', 'CRUD',
              'CLI diff object containing details about the commit such as username, time and the configuration diff') -->


The physical-router object has a new cli commit status attribute to indentify if the device is in sync or out of sync. The changes are below.
<xsd:simpleType name="CommitStateType">
     <xsd:restriction base='xsd:string'>
         <xsd:enumeration value='in_sync'/>
         <xsd:enumeration value='out_of_sync'/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:element name="physical-router-cli-commit-state" type='CommitStateType' default='in_sync'/>
<!--#IFMAP-SEMANTICS-IDL
          Property('physical-router-cli-commit-state', 'physical-router', 'optional', 'CRUD',
             'CLI commit state for the physical router. Used to check if device is in sync with Contrail managed configs') -->


# 6. Implementation

1. Detect a user generated CLI config on the device before config push by contrail 
- When contrail tries to push new config down to the device as a result
of a job being triggered such a role assignment or creation of overlay objects such as
a VPG, a check to see if there are any CLI injected configs on the device is done. 
- Changes are determined by comparing commits. Any commits made via cli is compared with 
the most recent config push by Contrail (via netconf)  using rollback mechanism. 
- If no changes are detected then the triggered backend workflow continues without any futher processing. 
- If there are CLI changes detected, the changes are first stored in the cli-commit object in the
database and the device is flagged to be out of sync. 

2. Workflow to accept/reject changes 
- The UI will then display all the devices that are out of sync in the fabric devices page to 
prompt the user to start the "CLI sync" workflow. ( Not sure if it's a workflow or something. Will update when the mockups are out)
- Once the user selects the workflow all the devices which have been out of sync will be displayed. 
- The user will be able to view the CLI changes per device and will also have an option to choose 
what changes he/she wants to accept or reject for that particular device. 

## 2.1 Accept Case 
- If the user accepts a change, then the particular config is first removed from the device, put into a 
seperate  group "__contrail__cli__group" and re-commited on the device. 
- The config accepted is also copied on to the "cli-commit" db object for that device
 under a different attribute called "accepted_configs".
- Any consecutive accepted changes wil be appended to this attribute so that this can be pushed as one big
config back to the device during RMA. 
- At the end of the workflow, the 'cli-commit' object will be updated with a status flag depnding on 
whether the sync was successful, still pending, or failed. 
- If the commit goes through successfully, the "commit_diff_list" dict object is deleted from the list since we 
no longer need it.

## 2.2 Reject Case 
- If the user rejects the change, the committed set/delete commands are appropriately reversed  
and are commited onto the device. 
- The particular cli_change dict object is also deleted from the cli-object. 


# 7. UI Changes - ( will update this once the screens are ready)

- If there are any CLI changes detected in any one of the devices in the fabric, the UI needs to prompt the user 
to execute the "CLI-sync" workflow. 
- Upon clicking the workflow a new screen displaying all the devices out of sync is shown on the left side and the 
right window will show all the CLI changes for that device along with info such as who made what change and at what time. 
- The user will have an option to individually/bulk select each of these changes to be either approved or rejected. 
- Only after the user has finished selecting the desired changes from all the devices, and upon clicking the "Done" 
button, the job will to save/delete config will get triggered. 
- The second screen will display per device job logs to check the progress of the workflow. 


# 8. Testing

# 9. Documentation Impact
