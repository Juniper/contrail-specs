# 1. Introduction
Mesos is built using the same principles as the Linux kernel, only at a different level of abstraction.
The Mesos kernel runs on every machine and provides applications (e.g., Hadoop, Spark, Kafka,
Elasticsearch) with APIs for resource management and scheduling across entire datacenter and cloud
 environments. Ref: http://mesos.apache.org/

# 2. Problem statement
Mesos containerized infrastructure has lowest elements as Application (group of tasks) and Pods.
There is a need to address networking segment (which includes network assignment, load balancing
and security) for seamless connectivity among pods and apps.

# 3. Proposed solution
Currently, Mesos Marathon framework leverages custom network provider for different network providers.
We use this to insert Contrail as networking components in the traditional Mesos infrastructure.
Contrail will bring in its features like virtual IPs, load balancer, network isolation, security and
policy based routing.

### Contrail as Network and IPAM provider
Contrail will assign IP to task & pod. Contrail will also attach them into its networking components
and would help route packets according to policies and route rules.

### 3.1 Task/Pod network assignment
As per the task request Task or Pod will be assigned to the designated network and will attach
provided IP or IP from the subnet range.

```yaml
  "ipAddress": {
    "networkName": "contrail-cni-plugin",
    "labels": {
      "networks": "blue-network",
      "project-name": "default-project",
      "domain-name": "default-domain",
    }
  }
```

### 3.2 Public/floating IP
As per the request, a task or pod can be allocated a public IP from the public network you can also
mention which IP to allocate from the public IP network subnet range.
```yaml
  "ipAddress": {
    "networkName": "contrail-cni-plugin",
    "labels": {
      "networks": "blue-network",
      "project-name": "default-project",
      "domain-name": "default-domain",
      "floating-ips": "default-domain:default-project:__public__:__fip_pool_public__(10.66.77.123),default-domain:default-project:__public__:__fip_pool_public2__(10.33.44.11)",
    }
  }
```

### 3.3 Assigning security group on an interface
An interface can be assigned with a requested security group.
```yaml
  "ipAddress": {
    "networkName": "contrail-cni-plugin",
    "labels": {
      "networks": "blue-network",
      "project-name": "default-project",
      "domain-name": "default-domain",
      "security-groups": "default-domain:default-project:security_groups_mesos"
    }
  }
```

### 3.4 Load balancer and DNS support
Extending Contrail IP-fabric features Contrail will be able to support Mesos-DNS and Marathon-lb &/ dcos-layer4-lb.

### 3.5 A sample marathon input file:
You can also mention the network.
```yaml
{
  "id": "/app-no-1",
  "acceptedResourceRoles": [
    "slave_public"
  ],
  "backoffFactor": 1.15,
  "backoffSeconds": 1,
  "container": {
    "type": "MESOS",
    "volumes": [],
    "docker": {
      "image": "ubuntu-upstart",
      "forcePullImage": true,
      "parameters": []
    }
  },
  "cpus": 0.1,
  "disk": 0,
  "instances": 1,
  "maxLaunchDelaySeconds": 3600,
  "mem": 128,
  "gpus": 0,
  "networks": [
    {
      "name": "contrail-cni-plugin",
      "mode": "container"
    }
  ],
  "requirePorts": false,
  "upgradeStrategy": {
    "maximumOverCapacity": 1,
    "minimumHealthCapacity": 1
  },
  "killSelection": "YOUNGEST_FIRST",
  "unreachableStrategy": {
    "inactiveAfterSeconds": 0,
    "expungeAfterSeconds": 0
  },
  "healthChecks": [],
  "fetch": [],
  "constraints": []
}
```
# 4. Implementation


## Architecture Diagram

```
+---------------------------------------------------+                +---------------------------------------------------+
|                                                   |                |                                                   |
|                                                   |                |                                                   |
|                                                   |                |                                                   |
|                                                   |                |                                                   |
|  +---------------+            +---------------+   |                |                          +-------------------+    |
|  |               | Updates    |               |   |                |                          |                   |    |
|  |               | via VNCAPIs|               |   |                |                          |  Mesos Executor   |    |
|  | Mesos Manager |            |    Contrail   |   |                |                          |                   |    |
|  |               |            |   Controller  +------------------------+                      |                   |    |
|  |               +----------> |               |   |                |   | Updates task IP      +--------+----------+    |
|  |               |            |               |   |                |   | info to agent                 | Invokes CNI   |
|  |               |            |               |   |                |   |                               | when task is  |
|  |               |            |               |   |                |   |                               v spawned       |
|  +------+--------+            +---------------+   |                |   |                                               |
|         ^                                         |                |   |                      +-------------------+    |
|         | Get task/pod                            |                |   |          Polls Agent |                   |    |
|         | info from API/Events                    |                |   |          for task info       CNI         |    |
|         |                                         |                |   |         +------------+                   |    |
|  +------+--------+                                |                |   v         v            |                   |    |
|  |               |                                |                |                          +-------------------+    |
|  |               |                                |                |  +--------------+                                 |
|  |               |                                |                |  |              |                                 |
|  |   Marathon    |                                |                |  |Contrail Agent|       DCOS/Mesos slave          |
|  |               |         DCOS/Mesos master      |                |  |              |       components are running    |
|  |               |         components are running |                |  |              |                                 |
|  |               |                                |                |  |              |                                 |
|  |               |                                |                |  |              |                                 |
|  +---------------+                                |                |  +------+-------+                                 |
|                                                   |                |         | Updates routing         "Slave Node"    |
|                                                   |                |         |                                         |
|                                                   |                +---------v-----------------------------------------+
|                                                   |                |                                                   |
|                               "Master Node"       |                | vRouter kernel module                "Kernel"     |
|                                                   |                |                                                   |
+---------------------------------------------------+                +---------------------------------------------------+

```
## Setup information :

Setup is done in two parts DCOS installation and contrail installation. For DCOS setup you can follow \
this website : https://dcos.io/install/.
For contrail installation follow :
https://github.com/Juniper/contrail-ansible-deployer. Make sure you fill out inventory file and set \
orchestrator as mesos.

Master Node consist of :
+ DCOS master components (https://docs.mesosphere.com/1.11/overview/architecture/components/)
+ Contrail master (Controller, Analytics, Config, UI)
+ Mesos Master

Slave/Agent Node consist of :
+ Contrail Agent
+ Contrail vRouter kernel module
+ Contrail CNI
+ DCOS slave components (https://docs.mesosphere.com/1.11/overview/architecture/components/)

## Components :

### 4.1 Contrail controller :

Contrail controller is the brain of contrail which does the decision making. You will find \
config management, analytics, UI and control plane components for network virtualization. \
You can find more information at https://github.com/Juniper/contrail-controller. Contrail expose \
API for creating configuration and updating virtual network components. In Mesos, mesos manager will \
update all information regarding task (universal docker) to Contrail Controller via API server.
All Contrail controller components are micro service docker.

### 4.2 Mesos Manager :

Mesos manager consist of two sub module :

 a. VNC server

 b. Marathon API

```
                           +----------------+
                           |                |
                           |                |
                           |                |
Interacts with             | Mesos Manager  |             Talks to Contrail VNC
marathon API & <-----------+                +-----------> API server
Server Side Events         |                |
                           |                |
                           |                |
                           |                |
                           |                |
                           +----------------+

```
 Mesos manager app runs inside a docker on the master. Mesos manager app when started it first tries to connect to \
 Marathon API server (master-ip:8080) and pulls all current running task. It parses only those tasks \
 which are registered as network "contrail-cni-plugin" and status as "TASK_RUNNING".
 More info on api at https://docs.mesosphere.com/1.11/deploying-services/marathon-api.
 Once it has all tasks data it checks against its VNC cached DB and updates/deleted task info which is stale.

 ```
     [
      {
        "id": "string",
        "containers": [
          {
            "name": "string",
            ....
          }
        ],
        ....
        "networks": [
          {
            "name": "contrail-cni-plugin",
            "mode": "container",
            "labels": {
              "additionalProp1": "string",
              "additionalProp2": "string",
              "additionalProp3": "string"
            }
          }
        ],
        "state":"TASK_RUNNING",
        ...
        "ipAddresses":[
            {
               "ipAddress":"172.17.0.3",
               "protocol":"IPv4"
            }
         ],
      }
    ]
 ```
 Now mesos manager subscribe to Server Side Events from Marathon which is real time event stream.
 More info at https://mesosphere.github.io/marathon/docs/event-bus.html. It subscribes to \
 status_update_event for task where taskStatus is "TASK_RUNNING", "TASK_FINISHED", "TASK_FAILED" or\
 "TASK_KILLED". For pods, it subscribes to  instance_changed_event and check condition as "Created" or "Failed".
 Filtering and subscribtion works as follow:
 ```
 curl -H "Accept: text/event-stream"  <MARATHON_HOST>:<MARATHON_PORT>/v2/events?event_type=status_update_event\&event_type=instance_changed_event
 ```

 Sample events:
 ```
 event: status_update_event
data: {"slaveId":"2275bb07-f40c-4d1e-884a-3d1332aed113-S4","taskId":"pod-with-virtual-network.instance-f575f481-caa6-11e8-a129-70b3d5800001.sleep1","taskStatus":"TASK_KILLING","message":"","appId":"/pod-with-virtual-network","host":"192.168.65.111","ipAddresses":[{"ipAddress":"9.0.1.3","protocol":"IPv4"}],"ports":[],"version":"2018-10-08T03:05:06.521Z","eventType":"status_update_event","timestamp":"2018-10-08T03:09:57.000Z"}

 event: instance_changed_event
 data: {"instanceId":"pod-multi.instance-a5f2bd75-caaa-11e8-a129-70b3d5800001","condition":"Created","runSpecId":"/pod- multi","agentId":"2275bb07-f40c-4d1e-884a-3d1332aed113-S4","host":"192.168.65.111","runSpecVersion":"2018-10-08T03:31:31.171Z","timestamp":"2018-10-08T03:31:31.222Z","eventType":"instance_changed_event"}

 event: instance_changed_event
 data: {"instanceId":"pod-multi.instance-a5f2bd75-caaa-11e8-a129-70b3d5800001","condition":"Failed","runSpecId":"/pod-multi","agentId":"2275bb07-f40c-4d1e-884a-3d1332aed113-S4","host":"192.168.65.111","runSpecVersion":"2018-10-08T03:31:31.171Z","timestamp":"2018-10-08T03:31:31.438Z","eventType":"instance_changed_event"}
```

### 4.3 Contrail CNI
Loction is CNI is /opt/mesosphere/active/cni/contrail-cni-plugin it a run to completion executable.
Config is located at /opt/mesosphere/etc/dcos/network/cni/contrail-cni-plugin.conf

Sample config file:
```conf
{
    "cniVersion": "0.3.1",
    "contrail" : {
        "vrouter-ip"    : "<slave-ip>",
        "vrouter-port"  : 9091,
        "cluster-name"  : "<slave-hostname>",
        "config-dir"    : "/var/lib/contrail/ports/vm",
        "poll-timeout"  : 15,
        "poll-retries"  : 5,
        "log-file"      : "/var/log/contrail/cni/opencontrail.log",
        "log-level"     : "debug"
    },

    "name": "contrail-cni-plugin",
    "type": "contrail-cni-plugin"
}
```

Mesos agent would invoke Contrail CNI when custom/host network provider is mentioned in the task
description. Where network provider is mentioned as "contrail-cni-plugin". CNI would then poll \
contrail agent for IP address and mac info and create a tap interface in a container.
Example : http://{slave-ip}:9091/vm-cfg/660a305f-10e4-47fd-a5fc-50e48147423b


Sample Task information updated to CNI :
```
{"args":{"org.apache.mesos":{"network_info":{"ip_addresses":[{"protocol":"IPv4"}],"labels":{"labels":[{"key":"networks","value":"blue-network"},{"key":"network-subnets","value":"5.5.5.0\/24"},{"key":"project-name","value":"default"},{"key":"domain-name","value":"default-domain"}]},"name":"contrail-cni-plugin"}}},"cniVersion":"0.2.0","contrail":{"cluster-name":"b2s27","config-dir":"\/var\/lib\/contrail\/ports\/vm","log-file":"\/var\/log\/contrail\/cni\/opencontrail.log","log-level":"info","poll-retries":5,"poll-timeout":15,"vrouter-ip":"10.84.22.27","vrouter-port":9091},"name":"contrail-cni-plugin","type":"contrail-cni-plugin"}
```


### 4.3 DNS and Load Balancer

In DCOS Minuteman acts as a layer 4 load balancer. It maps single virtual IP to multiple port and address.
It can be a name instead of IP address and its created automatically when service is installed.

VIPs follow this naming convention:

<service-name>.marathon.l4lb.thisdcos.directory:<port>

You can specify VIP in your task creation:
```
    "portDefinitions": [
        {
            "port": 0,
            "protocol": "tcp",
            "name": "http",
            "labels": {"VIP_0": "10.0.0.1:80"}
        }
    ],

    and
        "portDefinitions": [
        {
            "port": 0,
            "protocol": "tcp",
            "name": "http",
            "labels": {"VIP_0": "/my-service:80"}
        }
    ],
```
This will automatically create DNS and IPVS entry for routing.
https://docs.mesosphere.com/1.11/networking/load-balancing-vips/


Mesos/DCOS can also use Marathon LB as load balancer. It can be configured either internal or external.
External or internal could be differentiated using labels in task configuration. You can set both.
External load balancer should be configured on the public node.
More info : https://docs.mesosphere.com/services/marathon-lb/

```
  "labels":{
    "HAPROXY_GROUP":"internal"
  }
  or
    "labels":{
    "HAPROXY_GROUP":"external"
  }
  or
    "labels":{
    "HAPROXY_GROUP":"external,internal"
  }
```
## Minor Components :

### Logging :
Existing common/logger.py code would be used for logging.

### HA Setup :
Contrail can be configured as HA and one of the mesos manager would be elected as master and will connect to
a marathon API server, which can also be provided as a list. Mesos manager will traverse list and gets connected to
one of the API server. This is still not committed for 5.1.

### VNC cache :
A local vnc cache would be maintained inside mesos manager. So in case of restart Mesos will resync from VNC DB.
Also before querying to VNC API all calls would first query local cache and then connect to VNC API server.

### Windows support :
Windows support is not committed for 5.1.


# 5. Performance and scaling impact
Nothing so far.

# 6. Upgrade
Not applicable.

# 7. Deprecations
Not applicable.

# 8. Dependencies

# 9. Debugging

Curl to master IP to get status of all pods and apps.
```shell
curl http://{MASTER_IP}/marathon/v2/apps {/pods}
{
  "apps": [
    {
      "id": "/app-no-1",
      "acceptedResourceRoles": [
        "slave_public"
      ],
      "backoffFactor": 1.15,
      "backoffSeconds": 1,
      "container": {
        "type": "MESOS",
        "docker": {
          "forcePullImage": true,
          "image": "ubuntu-upstart",
          "parameters": [],
          "privileged": false
        },
        "volumes": [],
        "portMappings": [
          {
            "containerPort": 0,
            "labels": {},
            "name": "default",
            "protocol": "tcp",
            "servicePort": 10000
          }
        ]
      },
      "cpus": 0.1,
      "disk": 0,
      "executor": "",
      "instances": 1,
      "labels": {},
      "maxLaunchDelaySeconds": 3600,
      "mem": 128,
      "gpus": 0,
      "networks": [
        {
          "name": "contrail-cni-plugin",
          "mode": "container"
        }
      ],
      "requirePorts": false,
      "upgradeStrategy": {
        "maximumOverCapacity": 1,
        "minimumHealthCapacity": 1
      },
      "version": "2018-09-27T00:37:18.286Z",
      "versionInfo": {
        "lastScalingAt": "2018-09-27T00:37:18.286Z",
        "lastConfigChangeAt": "2018-09-27T00:37:18.286Z"
      },
      "killSelection": "YOUNGEST_FIRST",
      "unreachableStrategy": {
        "inactiveAfterSeconds": 0,
        "expungeAfterSeconds": 0
      },
      "tasksStaged": 0,
      "tasksRunning": 0,
      "tasksHealthy": 0,
      "tasksUnhealthy": 0,
      "deployments": [
        {
          "id": "7a72867d-1ad6-49c5-8321-c141f010466b"
        }
      ]
    }
  ]
}

```

You can also use dcos cli to retrieve status.
```shell
dcos marathon app list --json

[
  {
    "acceptedResourceRoles": [
      "slave_public"
    ],
    "backoffFactor": 1.15,
    "backoffSeconds": 1,
    "container": {
      "docker": {
        "forcePullImage": true,
        "image": "ubuntu-upstart",
        "parameters": [],
        "privileged": false
      },
      "portMappings": [
        {
          "containerPort": 0,
          "labels": {},
          "name": "default",
          "protocol": "tcp",
          "servicePort": 10000
        }
      ],
      "type": "MESOS",
      "volumes": []
    },
    "cpus": 0.1,
    "deployments": [
      {
        "id": "7a72867d-1ad6-49c5-8321-c141f010466b"
      }
    ],
    "disk": 0,
    "executor": "",
    "gpus": 0,
    "id": "/app-no-1",
    "instances": 1,
    "killSelection": "YOUNGEST_FIRST",
    "labels": {},
    "maxLaunchDelaySeconds": 3600,
    "mem": 128,
    "networks": [
      {
        "mode": "container",
        "name": "contrail-cni-plugin"
      }
    ],
    "requirePorts": false,
    "tasksHealthy": 0,
    "tasksRunning": 0,
    "tasksStaged": 0,
    "tasksUnhealthy": 0,
    "unreachableStrategy": {
      "expungeAfterSeconds": 0,
      "inactiveAfterSeconds": 0
    },
    "upgradeStrategy": {
      "maximumOverCapacity": 1,
      "minimumHealthCapacity": 1
    },
    "version": "2018-09-27T00:37:18.286Z",
    "versionInfo": {
      "lastConfigChangeAt": "2018-09-27T00:37:18.286Z",
      "lastScalingAt": "2018-09-27T00:37:18.286Z"
    }
  }
]

```

CNI logs:
```shell
/var/log/contrail/cni/opencontrail.log
```

Mesos manager logs (inside container):
```shell
/var/log/contrail/contrail-mesos-manager.log
 ```

# 10. Testing
## 10.1 Unit tests
## 10.2 Dev tests
## 10.3 System tests

# 11. Documentation Impact

# 12. References
