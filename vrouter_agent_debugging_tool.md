# 1. Introduction
This blue print describes the implementation of a tool needed for debugging
vrouter, agent and controller.

# 2. Problem statement
We need a separate tools which will do below task:
1. Collect logs, libraries, gcore(optional), introspect logs, sandesh traces and
docker logs from agent.
2. Collect output of vrouter specific commandss mentioned below.
```
        commands = ['nh --list', 'vrouter --info', 'dropstats',
                    'dropstats -l 0', 'vif --list', 'mpls --dump',
                    'vxlan --dump', 'vrfstats --dump', 'vrmemstats',
                    'qosmap --dump-fc',
                    'qosmap --dump-qos', 'mirror --dump', 'virsh list',
                    'ip ad', 'ip r', 'flow -s']
```
3. Collect logs from control node.

# 3. Proposed solution
We need to split this into 4 sub-tasks:

1. Write a python script which has the logic of collecting all the required logs
This script should be part of contrail-utils as contrail-utils is present in the
base container and we create all containers from base container.

2. Create a separate tools container from base container. Include all the
dependent python packages required in this container.

3. Write a helping playbook for ansible so that it starts tools container and
collect logs. This should take care of running the container in openstack and
kubernetes deployements.

4. Write a helping bash script for RHOSP(tripleo) deployment which will run this
container from undercloud and collect logs.

5. For openshift, we need to check on how to run the tools container.

User can also run this docker manually and then execute the script which would
take a yaml file as input and perform tasks mentioned in problem statement.
The name of the script file should be vrouter_agent_debug_tool.py

```
Synopsis:
    Collect logs, gcore(optional), introspect logs, sandesh traces and
    docker logs from vrouter node.
    Collect logs specific to control node.
Usage:
    vrouter-agent-debug-tool -i <input_file>
Options:
    -h, --help          Display this help information.
    -i                  Input file which has vrouter and control node details.
                        This file should be a yaml file.
                        Sepcify deployment_method from one of these: 'rhosp_director'/'ansible'/'kubernetes'
                        Specify vim from one of these: 'rhosp'/'openstack'/'kubernetes'
                        Template and a sample input_file is shown below.
                        You need to mention ip, ssh_user and ssh_pwd/ssh_key_file
                        required to login to vrouter/control node.
                        You can add multiple vrouter or control node details.
                        Specify gcore_needed as true if you want to collect gcore.
Template for input file:
------------------------
provider_config:
  deployment_method: 'rhosp_director'/'ansible'/'kubernetes'
  vim: 'rhosp'/'openstack'/'kubernetes'
  vrouter:
    node1
      ip: <ip-address>
      ssh_user: <username>
      ssh_pwd: <password>
      ssh_key_file: <ssh key file>
    node2:
    .
    .
    .
    noden:
  control:
    node1
      ip: <ip-address>
      ssh_user: <username>
      ssh_pwd: <password>
      ssh_key_file: <ssh key file>
    node2:
    .
    .
    .
    noden:
  gcore_needed: <true/false>

A sample input.yaml is shown below.

provider_config:
  deployment_method: 'rhosp_director'
  vim_type: 'rhosp'
  vrouter:
    node1:
      ip: 192.168.24.7
      ssh_user: heat-admin
      ssh_key_file: '/home/stack/.ssh/id_rsa'
    node2:
      ip: 192.168.24.8
      ssh_user: heat-admin
      ssh_key_file: '/home/stack/.ssh/id_rsa'
  control:
    node1:
      ip: 192.168.24.6
      ssh_user: heat-admin
      ssh_key_file: '/home/stack/.ssh/id_rsa'
    node2:
      ip: 192.168.24.23
      ssh_user: heat-admin
      ssh_key_file: '/home/stack/.ssh/id_rsa'
  gcore_needed: true
```

The script reads vrouter and control node details from the yaml file and creates
ssh connection with them based on ssh password/keys.
Once the connection is made, it executes commands on the remote host and
collects logs.

The user needs to specify deployment_method and vim(virtualized infrastructure
manager) in the input file based on which we will decide vrouter_agent container
name and log path.

For now we will support below mentioned vim.
1. rhosp deployed using rhosp director
2. openstack deployed using ansible
3. kubernetes deployed using ansible

Support for pure kubernetes will be taken up later.

## 3.1 Alternatives considered
None

## 3.2 API schema changes
Not Applicable

## 3.3 User workflow impact
None

## 3.4 UI changes
None

## 3.5 Notification impact
None

# 4. Implementation
## 4.1 Work items
1. Write a python script which would have the logic to create ssh connections
with remote host, execute command on remote host and collect outputs of commands
executed. This script should be part of contrail-utils.
2. Create a separate tools container created from base container. Since base
container contains contrail-utils, the script will be automatically available.
Include all the dependent python packages in the Dockerfile. For this we need to
create a new tools folder in contrail-container-builder.
3. Create a helping playbook for ansible depoyment that runs the docker and
collect logs. For e.g.
```
ansible-playbook -i inventory -e gcore=enable collect_sos_data.yaml
```
4. Create a helping bash script for RHSOP deployment which will run the docker
container in undercloud, copy logs on a monted volume and then exit.

# 5. Performance and scaling impact
None

## 5.1 API and control plane
None

## 5.2 Forwarding performance
None

# 6. Upgrade
None

# 7. Deprecations
None

# 8. Dependencies
None

# 9. Testing
## 9.1 Unit tests
Need to test this script in below deployements:
1. 'rhosp' deployed using 'rhosp director'
2. 'openstack' deployed using 'ansible'
3. 'kubernetes' deployed using 'ansible'

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References
