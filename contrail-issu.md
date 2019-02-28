# 1. Introduction
This is to provide in service upgrade of contrail controller.

# 2. Problem statement
Defining upgrade path is very crucial for contrail services. The following points may need to be addressed in the upgrade design.

1) Upgrade could be because of a bug fix or because of a new features or both.
2) Upgrade is supported one version up. However, design should be such that it can accommodate upgrade from any version to any higher version.
3) Depending in the cluster size and the configuration, upgrade duration may last from few minutes to few months.
4) Upgrade should support rollback in the event of failure.
5) Upgrade and rollback procedure should be under the control of orchestrator.
6) Data path should have minimal impact during the upgrade.
7) A consolidated web and analytics services may be needed during the upgrade.

All the above points may not be currently addressed, but they have to be kept in mind during the design.

# 3. Proposed solution
Contrail is a distributed system. Upgrade procedure may involve one or more nodes to be upgraded. Currently we consider all upgrades to go through ISSU and we will keep minor upgrades as out of scope of this document.

## 3.1 Upgrade Procedure
Controller is side by side upgraded and computes are in place upgraded. The proposed ISSU approach may need appropriate changes at ISSU infrastructure, application and orchestrator layers.

A high level ISSU procedure is defined below. More details are provided against each of the orchestration layer.

Config services upgrade:
1) Orchestrator to spawn a new controller cluster in parallel to existing controller cluster.
2) Orchestrator need to generate the configuration file needed for the ISSU application, on the inventory, login, application details including old and new clusters.
3) Orchestrator to start the ISSU procedure. ISSU shell script does the task of pre and post changes before ISSU application is started. They may be referred and absorbed as needed by orchestrator.
4) ISSU application would start syncing the config data, by translating it to the objects that newer version understands and posts it to newer version Cassandra and notifies newer applications through a rmq message.
5) Newer api-server would update its south bound just as the way it is doing today.
6) The iBGP mesh established among the old and new control nodes help exchange of routes.
7) Step 5 and 6 continues till all computes are upgraded. Now system is ready for compute update.
8) Orchestrator upgrades compute nodes with the newer version of software and and makes it point to newer controller.
9) Orchestrator downgrades compute nodes if needed with the older version of software and makes it point to older controller.
10) Orchestrator should have options to upgrade or rollback one or more computes at a time.
11) Once all the computes are upgraded, orchestrator would stop ISSU run sync process shuts down the old controller services and start the final sync and zk sync processes which does a final snapshot of rest of the keyspaces and zk data.
12) Orchestrator changes the north bound to point to newer controller for SDN controller service.
13) Till step 11, northbound was interacting with the old controller api-server.
14) After step 11, where finalize is done, northbound should start interacting with the new controller api-server.
15) Note at no point of time api-server requests should not be posted to new api-server till finalize, as those objects could be lost during the upgrade.

Analytic services upgrade:
1) All applications in the contrail generate lot of analytics data that show the current and history of the activity in the system.
2) There are applications that are written/could be written to generate alarms as well for billing purposes.
3) Currently we do not support migration of this data across versions, as it is not only expensive but sometime may not be truly useful.
4) Since upgrade may last for days to months depending on the cluster size and customer preference, computes may be distribuetd among old and new controllers resulting in them publishing their state to respective analytic services on newer and older controllers.
5) So, customers may need to monitor both the analytics engines till the upgrade of all computes.
6) If for any reason north bound cannot interact with dual analytic engines, they may update the analytics configuration in the respective conf files and issue a sig-hup to the applications to reread and connect to the configured analytics services.
7) Analytics claimed to be backward compatible with certain functionality but not completely backward compatible.
8) Analytics need to address full backward compatibility.

WebUI services upgrade:
1) Similar to analytics application, there will be a new webui services that are installed.
2) State of newer controller services and migrated computes are available only on the newer webui, while the older services and computes that are not yet upgraded are still on the older webui.
3) There is no consolidated view supported today. Discussions happened but no conclusion.

Compute service upgrade:
1) Currently this results in reinstallation of software. So, there is a traffic hit for a brief period of time.
2) Compute node upgrade could be hitless and was discussed.
3) Compute node team to take action on it.

### 3.1.1 ISSU infrastructure changes:
ISSU infrastructure in total has four processes running depending on various stages of upgrade.
1) issu-contrail-pre-sync does the initial sync of the system, started as part of step 4.
2) issu-contrail-run-sync does the run time sync while computes are upgraded, started as part of step-4 after pre-sync is done, and lasts till step 11.
3) issu-contrail-post-sync does the final sync of the Cassandra database, started
 as part of step 11.
4) issu-contrail-zk-sync does the sync of zk database, started as part of step 11.

Applications have to register the key-spaces and column families that are need to be upgraded and respective translation functions for schema translations.  Based on the above information, above issu applications does the sync of the objects after subjecting them to the registered translation functions, and also notify the applications in the newer version.

issu-contrail-run-sync is under supervisor/systemctl control for software high availability, as that is the only process that lasts for the whole duration of all computes.

For hardware high availability, the procedure can be initiated on any other system, but it is recommended that the failed system is recovered before continuing with the ISSU procedure.

The whole ISSU process set can be containerized and can be spawned during the upgrade procedure by the orchestrator. However, this is not currently supported. As of current status processes are expected to run in the container of the api-server.

### 3.1.2 Application layer changes:
Applications are required to register with the ISSU infrastructure with the new set of key-spaces that need to be replicated.
Applications need to take care of registering the translated functions as appropriate if translation data is needed. But default ISSU applications assumes NULL translations.

### 3.1.3 Orchestrator changes.
Orchestrator is the key component that drives the ISSU procedure. The following approaches were discussed. Orchestrator may pick the preferred approach.

Approach 1:
This basically starts with 3 node cluster, expands to 5 node cluster during ISSU, and rollback to 3 node cluster after ISSU.
1) To start with we have three v1 controllers.
2) We bring up two new controllers with v2 version, and re-clusters db as 5 node cluster, with the new replication factor as 5 instead of three.
3) So basically we have three v1 controllers and two v2 controllers with 5 db nodes. Note, v1 and v2 don’t step on each other in db, as they are logically partitioned.
4) Let the database be synced using ISSU procedure.
5) Now migrate all computes from v1 to v2.
6) Once all computes are migrated, shut down v1 contrail-services except db service to fail over northbound connectivity from v1 to v2.
7) Upgrade one node of v1 controller to v2 controller. May result in another Cassandra db re clustering.
8) Delete other two v1 nodes’ db from Cassandra cluster, and decommission them.
9) Now we are back to three v2 controllers.
11) Note, adding and deleting nodes from Cassandra is possible as per
http://docs.datastax.com/en/cassandra/2.1/cassandra/operations/opsAddingRemovingNodeTOC.html

Approach 2:
This basically starts with 5 node cluster and remains at 5 node cluster during
 and after ISSU.

1) Start with 5 node cluster.
2) Upgrade two controller nodes in the cluster to v2. Let the Cassandra
re clustering happen, by deleting db nodes and re adding back as part of v2.
3) V1 and v2 don’t step on each other in DB as they are logically partitioned.
4) Let the database be synced using ISSU procedure.
5) Complete compute migration from v1 to v2.
6) Once all computes are migrated, shut down v1 contrail-services except db service to fail over northbound connectivity from v1 to v2.
7) Now upgrade rest of the nodes, one node at a time or two nodes and then one node. You can’t upgrade rest of all nodes at the same time, as DB nodes fall to minority, and when they come back you would lose data.
8) After the upgrade and re clustering we are back with 5 v2 node controller cluster.

Approach 3:
1) Start with three node cluster.
2) Upgrade first node to v2. V1 and v2 controllers don’t step on each other in db as they are logically partitioned.
3) Let DB sync continue with ISSU procedure.
4) Upgrade computes.
5) Upgrade next controller.
6) Shutdown last v1 controller to failover northbound connectivity.
7) Upgrade the final v1 controller.
8) Since DB is separated, no Cassandra re clustering needed.
9) While upgrading second node, we are at single point of failure from contrail services perspective. To overcome that, we can take up Approach1 or Approach2 without worrying about Cassandra re clustering as DB is separate container, which we may call Approach 4.

Orchestrator should support upgrade of the following versions.
1) Upgrade from 3.2.x to 5.0
2) Upgrade from 5.0 to 5.x
3) Upgrade from 4.x to 5.0

## 3.2 API schema changes
None needed for application.

## 3.3 User workflow impact
Impacted.

## 3.4 UI changes
UI to review and provide consolidated view of the upgrade process.

# 4. Implementation

# 5. Performance and scaling impact
No performance impact, unless it is a compute node that is being upgraded.

## 5.2 Forwarding performance
Impacted during compute upgrade unless workloads are migrated before upgrade or infrastructure is built to support hitless upgrade.

# 6. Upgrade
This is an upgrade story.

# 7. Deprecations
None

# 8. Dependencies
Remote cluster installation and lifetime management.

# 9. Testing
## 9.1 Unit tests
Tests were written to validate the ISSU infrastructure.
## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact
None
# 11. Notes for discussion
1) ISSU does not enable by default new features in the newer application. Applications have to define a procedure to enable these.
2) Minor upgrades.
3) Version control.
4) Single view webui and analytics.
5) Service instances of service chain spread across old and new clusters.
6) Contrail compute upgrades for zero downtime and kernel upgrades.

# 12 Upgrade procedure contrail with contrail-k8s-manager

Contrail-k8s-manger in k8s is equivalent of contrail-neutron-plugin in the neutron server in open stack world. It doesn’t maintain any state. It interacts only with contrail-api server in contrail side of communication. So similar to contrail upgrade in open stack, where neutron doesn’t play any role in the upgrade, contrail-k8s-manager doesn’t play any role in the upgrade. It would be shut down in the newer version of contrail cluster, during the upgrade. However the communication between k8-api-server and old version of contrail-k8s-manager will continue, till all the computes are upgraded. As part of final step of upgrade, contrail-k8s-manager will be brought up along with the other services like schema-transformer, device-manager etc… and k8s-api-server communication will switch to newer contrail-k8s-manager.

Note, during the compute upgrade, if the compute is brought out of service, and work loads are redistributed, more down time might be expected, as the computes are shut down and relaunched in the newer systems.

# 13 Upgrade procedure for the contrail-vcenter as a plugin
Contrail-vcenter-plugin in vcenter environment is similar to contrail-neutron-plugin in the open stack world. It doesn’t maintain any state. However it interacts, with api-server and vrouter-agent for port operations. Upgrade is similar to that of contrail with contrail-k8s-manager upgrade, that is to spawn a parallel controller cluster for v2 and corresponding contrail-vcenter-plugin, which is down. After the initial sync, computes are ready for the upgrade. After the compute upgrade, old contrail-vcenter-plugin would continue to connect to the vrouter-agent, and vrouter-agent would connect to new controller. Once all the computes are upgraded, contrail-vcenter-plugin and vrouter-agent channel is moved newer contrail-vcenter-plugin by shutting down older contrail-vcenter-plugin and bringing up newer contrail-vcenter-plugin. And rest of the upgrade procedure remains the same.

Before ESXI upgrade, it can be marked for maintenance so that workloads can be redistributed for minimal data path down time.

# 14. References
http://www.opencontrail.org/opencontrail-in-service-software-upgrade/
