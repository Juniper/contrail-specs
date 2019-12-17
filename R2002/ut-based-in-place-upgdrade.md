

# 1. Introduction

Unit test based In-Place validation ensures that after a schema change, config remains compatible with control-node and agent. 


# 2. Problem statement

Current upgrade validation of Contrail is N to N+1. The validation is done by installing a full cluster, create some workloads on the cluster, upgrade the cluster and validate the consistency of the created workloads. This process is very expensive in terms of time and as such only allows for validation between two adjacent releases.

With the change to monthly releases customers must be able to perform more frequent upgrades of the Contrail Cluster with the possibly lowest impact. Equally important, customers will have to be able to jump across a number of releases. If a customer installs release N1and runs with it for a while and detects a bug which is fixed in N5 cannot be asked to go N1->N2->N3->N4->N5, it must be possible to smoothly go from N1->N5.

In order to allow for jumping across a number of releases, there must be some level of validation and, as mentioned above, the current method of validation is too costly with the increasing number of combinations to be validated. 


# 3. Proposed solution

Instead of bringing up a complete environment and perform the upgrade, each component in the Contrail system is mocked. Initially all mocks are using release N-x code and based on that a dataset is applied simulating workloads using N-x schema. At each layer of the system the result is validated and compared against the reference set.

In the next step, each component is replaced by a N version of the mock, starting at the top with the Config mock. With each replaced component the validation is performed again. If the validation succeeds the procedure is repeated with N-x -> N.

To simulate the above behavior, we will generate a config db json dump using a baselineDb dump and changed objects in schema as the input . This generated json db dump will be used as an input to contrail-node and vrouter UT .  Control-node and vrouter will run UT’s for N-X to N-1 versions against this generated  json db dump .


# 4. API schema changes

N/A


# 5. Alternatives considered

N/A


# 6. UI changes / User workflow impact


## 6.1   For Integration testing, CI will do the following to validate in-place upgrade.


1.   User need to provide test scripts to create/update impacted objects if there is a schema change.

2)	Generate json db dump with modified resources as explained in (9.1.1).   

3)	Run control plane UT testing for controller (Similar to bulk update) with (2) as the input for Releases N-1 to   N-X.

4)	If there is a failure, adjust the schema changes to fix the issue.

5) 	There should be a way to override this CI test case, in case we want to go in with non backward compatible schema change.


## 6.2  Validate a release for compatibility with X previous releases . 

1) Using the test scripts for X previous releases and a baseline db dump create an updated json db dump. 
2) Invoke controller/vrouter UT with updated json db dump as input.


# 7. Notification impact

N/A


# 8. Provisioning changes

N/A


# 9. Implementation


## 9.1 Config changes

   Develop a Tool to generate valid json db dump based on changed objects. 

###  9.1.1 Modified json db dump generation


-   Load the baseline db dump in json format using cassandra mock.
-   If there are resources in the schema being updated, developer need to provide scripts to create/update the object and objects need to updated.
-   After creating/updating objects in mocked db , export db into json format.


## 9.2 Control Node changes 


-   Develop a UT  test which can take json db jump as input, and run it against last X releases.


## 9.3 Vrouter changes 


-   Develop a UT  test which can take json db jump as input, and run it against last X releases.


## 9.4 CI changes.

-   Invoke config tool to find the schema difference and create valid db dump in json format.
-   Provide option to override the test.
-   Invoke control/vrouter UT test with json file as input.

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

    [https://contrail-jws.atlassian.net/browse/CEM-8951](https://contrail-jws.atlassian.net/browse/CEM-8951)

*   Architechure 
    [https://docs.google.com/document/d/1XzgoZ0QYMljIiK8is9glahVPbXveA1FR3s4SRij9Ssk/edit](https://docs.google.com/document/d/1XzgoZ0QYMljIiK8is9glahVPbXveA1FR3s4SRij9Ssk/edit)

