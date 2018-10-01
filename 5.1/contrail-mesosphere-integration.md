# 1. Introduction
Mesos is built using the same principles as the Linux kernel, only at a different level of abstraction. The Mesos
kernel runs on every machine and provides applications (e.g., Hadoop, Spark, Kafka, Elasticsearch) with API’s for
resource management and scheduling across entire datacenter and cloud environments. Ref: http://mesos.apache.org/ 

# 2. Problem statement
Mesos containerized infrastructure has lowest elements as Application (group of tasks) and Pods. There is a need to address networking segment (which includes network assignment, load balancing and security) for seamless connectivity among pods and apps.

# 3. Proposed solution
Currently Mesos Marathon framework leverages custom network provider for differnt network providers. We we use this to insert contrail as networking components in the traditional mesos infrastructure. Contrail will bring in its features like virtual ips, load balancer, network isolation, security and policy based routing.

### Contrail as Network and IPAM provider
Contrail will assign IP's to task and pods also release when required. Contrail will plumb task and pods into its networking components and would help route packets according to 

### 3.1 Task/Pod network assignment
As per the the task request Task or Pod will be assigned to the designated network and will attach provided ip or ip from the subnet range.

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

### 3.2 Public/floating ip
As per the request a task or pod can be allocated a public ip from the public network you can also mention which ip to allocate from the public ip network subnet range.
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

### 3.3 Assigning security group on interface
An interface can be assign with a requested security group.
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
Extending contrail ip-fabric features contrail will be able to support Mesos-DNS and Marathon-lb &/ dcos-layer4-lb.

### 3.5 A sample marathon input file:
You can also mention as networks.
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

### 4.1 Contrail CNI
Mesos agent would invoke contrail CNI when custom/host network provider is mentioned in the task description. CNI would parse all argument provided and pass required info to contrail's mesos manager. CNI would then poll contrail agent for ip address and mac info and create a tap interface in container. Loction is CNI is /opt/mesosphere/active/cni/contrail-cni-plugin and config is /opt/mesosphere/etc/dcos/network/cni/contrail-cni-plugin.conf

### 4.2 Mesos Manager
Mesos manager would receive information from CNI on port 8080 regarding task/pod and accordingly process information and inform contrail using api's to contrail controller. Information includes network in which task/pod should be assigned, allocate a public ip/floating ip and security group to be assigned to. Mesos manager is running as a distributed application on docker on each slave.

### 4.3 DNS and load balancer
Mesos DNS and Mesos marathon lb would running as part of contrail network so that resolved ip address can be reached out via contrails vrouter. Minuteman, spartan, Mesos DNS and Navstar should would be configured to make them working.

# 5. Performance and scaling impact
Nothing so far.

# 6. Upgrade
Not applicable.

# 7. Deprecations
Not applicable.

# 8. Dependencies

# 9. Debugging

Curl to master ip to get status of all pods and apps.
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

You can also use dcos cli to retrive status.
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

Cni logs:
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
