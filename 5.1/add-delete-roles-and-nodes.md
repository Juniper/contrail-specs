
# 1. Introduction

Any deployed Contrail Cluster will require to be modified as time goes on,
whether it is to scale the cluster by adding additional Controllers/Compute
Nodes or to maintain the existing Compute/Storage nodes. There will be a
particular need to pull unresponsive compute nodes off the cluster while
simultaneously adding healthy servers to replace them. There is also a common
case of user misconfiguration of the initial deployment which might require a
removal and cleanup before that node can be re-added to the cluster.

# 2. Problem statement

In an Ansible-provisioned Contrail Cluster, we do not currently have the
ability to modify the cluster in a simplified way using Ansible. There is only
a specialized work flow for adding additional Compute Nodes to a cluster. There
are currently no playbooks or workflows to delete any node or role. In previous
releases of Contrail, we have documented manual workflows such as the below:

[ Adding a Compute Node
](https://www.juniper.net/documentation/en_US/contrail4.0/topics/task/installation/add-new-compute-node-vnc.html)

[ Adding a Controller Node
](https://www.juniper.net/documentation/en_US/contrail4.1/topics/concept/add-node-existing-container.html)

Neither of these provided a seamless Ansible-backed workflow which allowed the
user to intuitively manage their Contrail Cluster. Since we have moved all our
provisioning logic to Ansible we need to be able to invoke the same set of
playbooks regardless of the provisioning operation we intend to deploy. We
should not expect the user to manually log in to any node on the cluster in
order to expand or shrink its size. All provisioning or deployment based
actions should have a simple way of being invoked from an ansible playbook.
Ansible needs to be able to handle all these previously manual tasks
internally.

# 3. Proposed solution

Our proposed solution has two broad aims: 1. Simplify and standardize the
workflow needed to achieve any desired cluster deployment 2. Maintain the
ability to have fine-grained control over deployment of roles

Goal number 1 is straightforward: we wish to be able to deploy a cluster based
on a specified instances file, regardless of the action needed to reach that
state (fresh deployment/delete node/add role). If a user specifies what they
wish their cluster to look like in the instances file, the Ansible playbook has
to be able to figure out the set of actions required to have the cluster
deployed exactly as intended.

Goal number 2 relates to the granularity at which we perform these actions.
Since we currently allow a user to deploy a single role on a single node, we
need to maintain the ability to do this minimal action as a playbook.

Further, we want our solution to be generic such that a similar workflow can be
followed regardless of the cloud orchestrator being deployed. The complexity of
each action in any orchestrator should be handled by the playbooks for that
particular orchestrator. If absent, these actions can be written as plays in
custom task per orchestrator role that we support. For example, today Openstack
kolla ansible playbooks provide a destroy role that we can invoke directly to
cleanup an Openstack node.

Our solution is to implement a wrapper playbook "provision.yml" which acts as
the common playbook called for any provisioning action to be deployed on the
cluster. This playbook consumes the supplied instances.yml file and first
constructs a lists of actions to take for each node in the cluster and then
calls the appropriate action playbooks for those nodes. The construction of
this list of actions depends on both the supplied instances config as well as
getting information from the existing cluster (if any).

As the first step before deployment, the wrapper playbook will map the state of
the existing cluster (getting the deployment details from Config API server)
and compare this to the intended state specified in the instances file.
Comparing the two it will come up with a list of roles per node that have to be
provisioned or removed. Let us look at the different use cases in a little mode
detail.

## 3.1 Use cases

In each use case we will use the below to illustrate how our action list is
calculated:

Instances(A) - List of roles provided for Node A in instances file Cluster(A) -
List of roles currently successfully deployed for Node A

In each of the following cases, the user provides the nodes, associated roles
and contrail configuration in the instances file as done today.

### 3.1.1 Fresh Provision of cluster

Since the wrapper playbook will not be able to contact any API server or
orchestrator node, it will be able to calculate that there is no existing
deployment.  i.e. : Cluster(A) = [] For every node A in cluster

Thus our final list of roles to provision will be Instances(A) for every node A
in cluster. We can call the appropriate install_role playbook for each of these
roles for each node.

### 3.1.2 Adding a role to a node in the cluster

The wrapper playbook will be able to contact API server and the orchestrator of
the existing deployment and calculate the list of provisioned roles for all
nodes.  For one of these nodes there will be an extra role in Instances(A)
Thus, the set of roles to be provisioned for each node will be Instances(A) -
Cluster(A) We can call the appropriate install_role playbook for these roles.

### 3.1.3 Deleting a role from a node in the cluster

The wrapper playbook will be able to contact API server and the orchestrator of
the existing deployment and calculate the list of provisioned roles for all
nodes.  For one of these nodes there will be an extra role in Cluster(A) Thus,
the set of roles to be deleted for each node will be Cluster(A) - Instances(A)
We can call the appropriate delete_role playbook for these roles.

### 3.1.4 Removing a node from the cluster

The instances.yml file will not have any entry for this node but we will still
get the node role list from API and orchestrator.  In this scenario,
Instances(A) will be empty while Cluster(A) will have currently provisioned
roles on that node.  Thus, the set of roles to be deleted for each node will be
Cluster(A).  As in case 3.1.3, we can call the appropriate delete_role playbook
for these roles.  In the case where the node to remove is not reachable, we can
ignore running tasks on it and de-register it from the Contrail
API/Orchestrator.  There can be an additional option to cleanup the node, which
is dependent on rechability.

## 3.2 Alternatives considered

### Specifically calling an add or delete playbook

The general idea here was to have the operator manually call the appropriate
add or delete playbook based on their requirements.

PROS: This solution does not interfere with the provision code for a fresh
provision.  The user can craft a very specific configuration to add to cluster
or delete from cluster.

CONS: There are a lot of unsupported edge cases and exposing the delete and add
playbooks directly to the end user increases the chances of failed deployments.
In addition there are issues with the input files that operator would use to
delete roles. These are specified below.

#### Using a separate instances file just to delete

Today, we use an instances file to provision a cluster, with the understanding
that the instances file reflects the expected end state of the cluster after
provision. By adding an additional file that lists only the nodes we want to
delete, we lose a complete picture of what the cluster looks like after any
such operation as ansible does not maintain state.  In addition we change the
meaning of the input instances file which as of today is a declarative
representation of the cluster we want.

#### Passing list of nodes to delete as an additional cluster parameter

This approach is flawed in a similar way to the one above as we lose the
picture of the cluster post a delete operation. It depends on user manually
maintaining the state in some YAML file.  We also go out of line with the
inherent declarative state of instances file.

## 3.3 Provision Architecture and Workflow Changes

The architecture of ansible playbooks for deployment largely remains unchanged
with the exception of a wrapper playbook running on top of all existing
playbooks. Today we have action specific playbooks like install_contrail.yml,
configure_instances.yml, install_<orchestrator>.yml and destroy_contrail.yml.
These are expected to be called explicitly by user. Going forward, the user
will only call a single playbook: provision.yml which will call all the
appropriate playbooks for the action we are trying to provision.

As a result, all roles that we support in instances.yaml have to have an
appropriate install_<role>.yml and delete_<role>.yml because that is the
granularity that we have to support for add/delete of role.

If an operator is trying to do simultaneous add and delete of roles, for
example, trying to change a compute node to a controller, we need to delete the
role first, initiate a cleanup of node and only then do the provision of the
added role. Even though this is not a workflow change from something operator
would manually do, it is a change as far as Ansible Deployer is concerned as we
do not run many different playbooks on same node today.

# 4. Implementation

The wrapper playbook will be an ansible playbook that calls roles to calculate
the various role lists described above for each node. It will be implemented in
a similar fashion to "create_openstack_host_group" role which calculates the
list of nodes for each openstack role. This involves the use of filter plugins
written in python which read the instances file and construct the node_role
dictionaries.

Additional tasks will communicate with the config API and Orchestrator of an
existing cluster. This will run first to see what the existing deployment looks
like. The cluster node_role dictionary will also be constructed as described
above.

# 5. Performance and scaling impact

N/A

# 6. Upgrade

N/A

# 7. Deprecations

Some workflows will be deprecated and wikis have to be updated reflecting use
of the wrapper playbook.

# 8. Dependencies

N/A

# 9. Testing

The wrapper playbook needs to be tested in all supported scenarios described
above. In addition, each individual role playbook has to be tested for the
action it is expected to achieve (add/delete).

# 10. Documentation Impact

Documentation has to be changed reflecting usage of wrapper playbook. New
documentation has to be added for add and delete of roles.

# 11. References
