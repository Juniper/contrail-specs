Configuration of appformix flows on contrail-vrouter-agent and rsyslog
===
# 1.      Introduction
This blueprint describes configuration parameters needed for appformix flows on contrail
vrouter-agent and rsyslog configuration file.

# 2.      Problem Statement
appformix flow collector is responsible for receiving and processing flow
packets from the network devices. The flow collector receives sflow, ipfix,
netflow and contrail-flow packets. To send contrail-flows to flow collector
and to consume contrail-flows by collector, user has to do below configurations:

1. Configure flow_export_rate in global-vrouter-config
2. Add `syslog` as sample_destination in contrail vrouter-agent configuration file
3. Add flow collector IP in rsyslog configuration file

Once appformix-flow node is provisioned, user should not do all the above
configurations manually in contrail vrouter-agent or rsyslog to start
functionality of appformix flows.

# 3.      Proposed Solution
Currently as part of contrail vrouter-agent provisioning, rsyslog is installed
on host and  additional rsyslog configuration file `contrail-lbaas-haproxy.conf`
is added in `/etc/rsyslog.d` directory.

As part of this blueprint, rsyslog will not be installed on host while
contrail vrouter-agent installation. A new container for rsyslog will be
provisioned in the nodes along with contrail vrouter-agent and during
provisioning stage, all the above configurations will be set.


# 3.1    Alternatives considered
No

# 3.2    API changes
No

# 3.3      User workflow impact
User does not have to configure flow-export-rate, sample_collection manually,
as part of the contrail provisioning, those configurations will be taken care.

## 3.4      UI Changes
When adding appformix-flow collector node, we need to add `appformix_flows` role
for that node.

## 4.1      Work items
### 4.1.1 Changes in Service Monitor
The path for ```haproxy.log.sock``` needs to be changed from /var/log/contrail/lbass to
/var/run/contrail/loadbalaner

### 4.1.2 Changes in contrail-ansible-deployer
`appformix_flows` role defines where we need to provison appformix-flow collector module.
If there are multiple `appformix_flows` role defined in instances.yml file, then we must have keepalived
If `appformix_flows` role can be added for any node in instances.yml file, then we must have
`keepalived_shared_ip` specified under `xflow_configuration` in instanes.yml file.

We need to set an environment variable `XFLOW_NODE_IP` derived as below:
1. If `xflow_configuration.keepalived_shared_ip` is defined, then assign this value
2. If `xflow_configuration.keepalived_shared_ip` is not defined, then assign the IP
   where we have `appformix_flows` role defined

### 4.1.3 Changes in contrail-container-builder
A new rsyslog ontainer will be created. Whatever configurations were being done on
host rsyslog config file, we need to move them to rsyslog container.

The below changes need to be done.
1. If environment variable XFLOW_NODE_IP is set, then we need to set
   `FLOW_EXPORT_RATE` to be 100, if not specified, then 0
2. In contrail vrouter-agent config file, `syslog` should be added for
   `SAMPLE_DESTINATION`
3. A new configuration file for appformix-flow will be added in /etc/rsyslog.d directory,
   and we need to add below:
   ```:msg, contains, "SessionData" @$XFLOW_NODE_IP:$RSYSLOGD_XFLOW_LISTEN_PORT```
   We need to define default RSYSLOGD_XFLOW_LISTEN_PORT to be 9898
   So the syslog messages containing `SessionData` string should be sent to XFLOW_NODE_IP address.

### 4.1.4 Changes in Other Deployers
Other deployers like contrail-helm-deployer, openshift-ansible, tripleo, also need to have similar changes.

# 5 Performance and Scaling Impact
None

## 5.1     API and control plane Performance Impact
None

## 5.2     Forwarding Plane Performance
None

# 6 Upgrade
None

# 7       Deprecations
None

# 8       Dependencies
None

# 9       Testing
## 9.1    Dev Tests
1. From Contrail Command UI, add one appformix-flow node, install contrail, appformix
   and appformix-flow. Verify that in /etc/contrail/common.env file, we have
   XFLOW_NODE_IP is set as the IP address where appformix_flows role is provisioned.
2. From Contrail Command UI, add two appformix-flow nodes, install contrail,
   appformix and appformix-flow. Verify that in /etc/contrail/common.env file,
   we have XFLOW_NODE_IP is set as IP address which was used as keepalived_shared_ip.

# 10      Documentation Impact

# 11      References

