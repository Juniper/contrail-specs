Configuration of flow collector on contrail-vrouter-agent and rsyslogd
===
# 1.      Introduction
This blueprint describes configuration parameters needed for flow
collector on contrail-vrouter-agent and rsyslogd configuration file.

# 2.      Problem Statement
flow collector is responsible for receiving and processing flow packets from
the network devices. The flow collector receives sflow, ipfix, netflow and
contrail-flow packets. This blueprint describes the configurations needed to send
contrail-flows to flow collector. contrail-flow will be sent over syslog
messages to flow collector using rsyslogd.
User has to do below configurations for this:

1. Configure flow_export_rate in global-vrouter-config
2. Add `syslog` as sample_destination in contrail vrouter-agent configuration file
3. Add flow collector IP in rsyslogd configuration file

Once flow collector node is provisioned, user should not do all the above
configurations manually in contrail vrouter-agent or rsyslogd to start
functionality of flow collector.

# 3.      Proposed Solution
Currently as part of contrail vrouter-agent provisioning, rsyslogd is installed
on host and additional rsyslogd configuration file `xflow.conf` is added in
`/etc/rsyslog.d` directory.

As part of this blueprint, rsyslog will not be installed on host while
contrail vrouter-agent installation. A new container for rsyslogd will be
provisioned in the nodes along with contrail vrouter-agent and during
provisioning stage, all the above configurations will be set.

The new rsyslogd container can be used for sending flow messages to any syslog
enabled collector. flow-collector is one usage of this new rsyslogd container.

# 3.1    Alternatives considered
No

# 3.2    API changes
No

# 3.3      User workflow impact
User does not have to configure flow-export-rate, sample_collection manually,
as part of the contrail provisioning, those configurations will be taken care.

## 3.4      UI Changes

## 4.1      Work items
### 4.1.1 Changes in Service Monitor
The path for ```haproxy.log.sock``` needs to be changed from /var/log/contrail/lbass to
/var/run/contrail/loadbalaner

### 4.1.2 Changes in contrail-ansible-deployer
We need to set an environment variable `XFLOW_NODE_IP` derived as below:
1. If `xflow_configuration.keepalived_shared_ip` is defined, then assign this value
2. If `xflow_configuration.keepalived_shared_ip` is not defined, then assign the
   management IP of flow collector node.

### 4.1.3 Changes in contrail-container-builder
A new rsyslogd container will be created. Whatever configurations were being done on
host rsyslogd config file, we need to move them in rsyslogd container.

The below changes need to be done.
1. If environment variable XFLOW_NODE_IP is set, then we need to set
   `FLOW_EXPORT_RATE` to be 100, if not specified, then 0
2. In contrail vrouter-agent config file, `syslog` should be added for
   `SAMPLE_DESTINATION`
3. A new configuration file for flow collector will be added in /etc/rsyslog.d directory,
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
1. From Contrail Command UI, add one flow collector node and provision.
   Verify that in /etc/contrail/common.env file, we have XFLOW_NODE_IP is
   set as the IP address which is used for flow collector loadbalancer ip .
2. From Contrail Command UI, add one flow collector node and provision.
   Verify that in /etc/contrail/common.env file, we have XFLOW_NODE_IP is
   set as the IP address which is used for flow collector loadbalancer ip .
3. In test case 1 and 2, verify below:
  3.1 flow_export_rate in global-vrouter-config has been set to 100
  3.2 Verify that in new rsyslogd container in /etc/rsyslog.d, we have new
      configuration file for flow collector `xflow.conf` and we have entry
      for flow collector ip address.

# 10      Documentation Impact

# 11      References

