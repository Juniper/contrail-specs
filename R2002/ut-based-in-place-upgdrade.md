


# 1. Introduction

In-place upgrade schema based validation provides us information about compatibility of Control/Agent services of version N-2..N-x with N of Config services. This validation is to understand where Schema compatibility breaks and always be treated as a supplementary to SysTest validation of N-1 -> N upgrade. 

# 2. Problem statement

SysTest performs N-1 to N upgrade validation on every contrail release. However N-2 to N or N-n to N can not be repeated for every release. 

In order to allow for jumping across a number of releases, there must be some level of validation and, as mentioned above, the current method of validation is too costly with the increasing number of combinations to be validated. 

With the change to monthly releases customers must be able to perform more frequent upgrades of the Contrail Cluster with the possibly lowest impact. Equally important, customers will have to be able to jump across a number of releases.


# 3. Proposed solution


Instead of bringing up a complete environment and performing the upgrade, each component in the Contrail system is mocked. 

In the next step, each component is replaced by an N version of the mock, starting at the top with the Config mock. With each replaced component the validation is performed again. If the validation succeeds the procedure is repeated with N-x -> N.

To simulate the above behavior, we will generate a config db JSON dump, using a baseline Db dump and changed objects . This generated JSON db dump will be used as an input to contrail-node and vrouter UT.  Control-node and vrouter will run UT’s for N-X to N-2 versions against this generated  JSON db dump . CAT (Control-node Agent Testing) framework will
be used to run UT.






# 4. API schema changes

N/A


# 5. Alternatives considered

N/A


# 6. UI changes / User workflow impact


## 6.1   For Integration testing, CI will do the following to validate in-place upgrade.

1.   Users need to provide test scripts to create/update impacted objects if there is a schema change.

2)   Generate JSON db dump with modified resources as explained in (9.1.1).   

3)  Run CAT testing for controller and Agent with (2) as the input for Releases N-1 to   N-X.
    i)  CAT testing takes controller UT and Agent UT executables and JSON db dump as the inputs.
    ii) Control node UT and Agent UT images  for previous releases will be stored in a repo
       and  will be retrieved to invoke the CAT test for previous versions. 

4)  If there is a failure, adjust the schema changes to fix the issue.


# 7. Notification impact

N/A

# 8. Provisioning changes

N/A


# 9. Implementation

## 9.1 Config changes
   Modify config mock to support loading db dump in JSON format.
   Identify modified resources from  1907 to the current release .
   Develop unit test cases which will create/update modified resources .   
   Modify config mock to support save db dump in JSON format.

###  9.1.1 Modified JSON db dump generation

-   Load the baseline db dump in JSON format using cassandra mock.
-   If there are resources in the schema that are being updated, developers  need to provide scripts to create/update the object and objects need to be updated. For the initial version we will be developing test cases.  
-   After creating/updating objects in mocked db, export db in JSON format.

## 9.2 Control Node/Vrouter/CI changes 
Develop a test case to take db dump  in JSON format  and test against X previous
versions of control-node and Agent.
Mechanism to store X previous versions of control-node and Agent images.
Enhancements in CAT framework infra to  work with JSON db dump in standard config
Db backup format as input. 
Add extra control-node  CAT test cases to for coverage.
Add extra Agent  CAT test cases to for coverage.


# 10. Device specifics


# 11. Performance and scaling impact


N/A


# 12. Upgrade

N/A


# 13. Deprecations

N/A


# 14. Dependencies


# 15. Testing


# 16. Documentation Impact


# 17. References

*   Story

*   Architecture 
    

