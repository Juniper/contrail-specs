# 1. Introduction
This blueprint describes adding or deleting flow collector nodes to existing
Contrail cluster at run time.

# 2. Problem statement
Once flow-colletors are added via Contrail UI Setup wizard, we do not have
any way to add new flow-collector node to existing Contrail cluster or delete
existing flow-collector node.
So, user should be able to add or delete flow-collector node using Contrail
UI.

# 3. Proposed solution
Contrail UI will have option to add or delete flow-collector nodes at run time.

# 4. API schema changes
N/A

# 5. Alternatives considered
None

# 6. UI changes / User workflow impact
We will have option to add/delete flow-collector nodes in
Infrastructure -> Cluster -> Assign Flow Collector Nodes

# 7. Notification impact
N/A

# 8. Provisioning changes
When a new flow-collector node is added or existing flow-collector node is
deleted from UI, then ansible playbook will run to first cleanup
the flow-collector services if any existing flow-collector node is deleted,
and then ansible playbook will run to setup the new flow-collector nodes
and to form the cluster.
None

# 9. Implementation
#### Deployer changes
In deployer, we will first derive the installed flow-collector nodes from
database, and then will compare with the new instances, if some flow-collector
node is deleted, then we create inventory file with those deleted nodes and run
the cleanup playbook, then will run the setup playbook to setup the rest of the
cluster with new flow-collector nodes.

# 10. Performance and scaling impact
N/A

# 11. Upgrade
N/A

# 12. Deprecations
N/A

# 13. Dependencies
N/A

# 14. Security Considerations
N/A

# 15. Testing
#### Unit tests
#### Dev tests
#### System tests

# 16. Documentation Impact

# 17. References
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-9294
