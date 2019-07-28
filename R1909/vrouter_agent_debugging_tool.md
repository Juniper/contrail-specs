# 1. Introduction
This blue print describes the implementation of a tool needed for debugging vrouter, agent and controller.

# 2. Problem statement
We need a separate tools container which will do below task:
1. Collect logs, libraries, gcore(optional), introspect logs, sandesh traces and docker logs from agent.
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
1. Create a separate tools container with script installed in it. It will be used as a runtime container. 
User starts container with parameters and results are stored on a mounted volume.
2. We need to write a script in python which would take a yaml file as input and perform tasks mentioned in problem statement.
This yaml file should be in below format.

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

The script reads vrouter and control node details from the yaml file and creates ssh connection with them based on ssh password/keys.
Once the connection is made, it executes commands on the remote host and collects logs.

The user needs to specify deployment_method and vim(virtualized infrastructure manager) in the input file based on which we
will decide vrouter_agent container name and log path.

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
1. Write a python script which would have the logic to create ssh connections with remote host, execute command on remote host and collect outputs of commands executed.
2. Create a separate tools container with above script installed. For this we need to create a new tools folder in contrail-container-builder.
Inside this tools folder we need to copy the script, Dockerfile, entrypoint.sh
3. Create a bash script which will run the docker container, copy logs on a monted volume and then exit. This script needs to be in /usr/bin/ of the host where this docker image will be available.

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
Have tested this script in below deployements:
1. 'rhosp' deployed using 'rhosp director'
2. 'openstack' deployed using 'ansible'
3. 'kubernetes' deployed using 'ansible'

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References
