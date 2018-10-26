
# 1. Introduction

Today, upstream kolla-ansible repo provides a destroy role which is used to
remove all Openstack Kolla components on that file. This includes images,
containers and config YAML files. We wish to integrate the destroy role and
invoke it specifically on those nodes where Openstack components are running.

# 2. Problem statement

The contrail-ansible-deployer repo is currently integrated with our custom fork
of the upstream repo:
[Contrail-Kolla-Ansible](https://github.com/Juniper/contrail-kolla-ansible/tree/contrail/queens).
This fork keeps in sync with upstream kolla-ansible repo and thus already has
the code for the destroy role but it is currently not being used.

Integrating this Ansible role into contrail-ansible-deployer will allow the user
to delete and remove Openstack nodes from a cluster as part of a well defined
workflow.

Since we already plan to support deletion of Contrail Nodes via
contrail-ansible-deployer, we could also similarly specifically call Openstack
destroy role for the nodes we are deleting. This integration is crucial to
removal of compute nodes which get deployed with vrouter role and
openstack-compute role.

# 3. Proposed solution

There is an existing install_openstack.yml playbook that is used by
contrail-ansible-deployer to deploy Openstack nodes. This playbook will continue
to be used to deploy a cluster with Openstack components. The deleted nodes will
be first caluclated in a similar fashion as it is for Contrail Roles today. With
this list of deleted Openstack nodes we can call the destroy role specifically
for them and this will remove all Openstack related files, images and containers
from that node.

In the case of deleting unreachable nodes or removing nodes from a cluster, we
can implement a separate "de-register" step to remove those nodes. This will
mark their services as unavailable and remove them from any groups like
allocation-pools, etc.

## 3.1 Use cases

### Deleting a Compute Node

It is a common scenario to remove an Openstack Compute node if it goes down. In
this scenario, the Nova services for that node have to be disabled and it has to
be removed from all allocation-pools and availability zones. It also has to be
cleaned up if still reachable.

### Replacing a Compute Node

This scenario is a super set of the previous case, where the node will first be
deleted from the cluster before a new node is added to it.  Both actions have a
common workflow so only one set of provision tasks need to be run. Within this
single run, the ordering of tasks will be such that the removal of deleted node
will happen first.

### Replacing an Opentack Controller:

There is a chance that one node that is part of an Openstack Controller Cluster
in an HA environment goes down. In this scenario, the administrator might want
to replace the Controller with a different node. To do this cleanly, we must
make sure we remove the dead Controller in an orderly fashion, first logically
removing all its registered services. After this we can spawn Openstack roles on
the new node exactly as done today. The services from the new node will get
registered with the running Opentack nodes and this will complete the
replacement.  From user-interface perspective, the user just has to replace the
deleted node section in instances file with the section for the new node.

## 3.2 Alternatives considered

N/A

## 3.3 Provision Architecture and Workflow Changes

The architecture of ansible playbooks for deployment largely remains unchanged.
The workflow for delete will be the same as the workflow for add. The
install_openstack.yml playbook will take care of the removal logic.

# 4. Implementation

The logic for calculation of nodes to remove will be done by the existing
install_openstack.yml playbook in a similar fashion to how it is done in
install_contrail.yml for Contrail nodes. This calculated list will be passed to
contrail-kolla-ansible playbook which will run the destroy role for those nodes.

# 5. Performance and scaling impact

N/A

# 6. Upgrade

N/A

# 7. Deprecations

N/A

# 8. Dependencies

N/A

# 9. Testing

The removal of Openstack Computes and Controllers has to be tested via funtional
tests. It will also be a part of sanity.  As part of this, both single and
multi-interface cases have to be tested. These tests will not be added to CI.

# 10. Documentation Impact

New documentation has to be added for removal of Openstack nodes.

# 11. References

