
# 1. Introduction

Contrail helm deployer allows you to do a complete life cycle management
(installation, update and delete) of micro-services architecture based
contrail docker containers and also gives a way to configure various features
of contrail

# 2. Problem statement

With contrail 5.0 micro-services architecture, there should be a way to deploy
these contrail docker container as an application on top of kubernetes and use
kubernetes features to do a better life cycle management of contrail services.


# 3. Proposed solution

To create contrail helm charts. Helm is a software which helps you manage
kubernetes application. Helm uses a packaging format called charts. Contrail
helm charts, will have collection of charts. Each chart is a collection of
files which helps you define contrail services as a set of kubernetes resources.


## 3.1 Alternatives considered

## 3.2 API schema changes

## 3.3 User workflow impact

## 3.4 UI changes

## 3.5 Notification impact


# 4. Implementation

## 4.1 Contrail Helm Deployer Charts architecture

Contrail Helm Deployer is divided into six different charts

  * helm-toolkit:

    Is a chart imported by openstack-helm project which has common
    templates/functions defined which will be used by every other contrail chart

  * contrail-thirdparty:

    This chart we have defined below thirdparty containers as kubernetes
    resources

      * RabbitMQ
      * ZooKeeper
      * Cassandra
      * Kafka
      * Redis

  * contrail-controller

    In contrail-controller chart we have defined below contrail components as
    kubernetes resources

      * Control
      * Config
      * Webui

  * contrail-analytics

    This chart we have defined contrail analytics components as kubernetes
    resources

  * contrail-vrouter

    This chart helps to define contrail vrouter components as kubernetes
    resources

  * Contrail-superset

    This chart is a super set of all other charts, using this one chart we will
    be able to install all kubernetes resources defined in other contrail Charts


## 4.2 Contrail kubernetes resource implementation

All contrail charts follow a similar approach to implementation of kubernetes
resources. Each of the contrail 5.0 containers expect configuration input to be
given in as environment variable. Contrail helm charts use values.yaml file to
obtain input from users. Users should use variable `.Values.contrail_env` to
define environment variables for the containers

```yaml
contrail_env:
  CONTROLLER_NODES: 10.87.65.248
  LOG_LEVEL: SYS_NOTICE
  CLOUD_ORCHESTRATOR: openstack
  AAA_MODE: cloud-admin
```

All these environment variables are stored in a kubernetes resource known as
configmap. These configmaps are loaded into specific containers as environment
variables.

As contrail is an infrastructure level application, every pod of contrail will
be hosted on host network namespace. Because of which we use daemonset
controller to define all contrail pods which makes sure that it bring up these
contrail pods on different nodes to avoid port conflicts.

### 4.3 Standard Contrail pods to be deployed on kubernetes

Contrail helm deployer gives an option to logically group containers into pods
or deploy each container as different pod. It checks the value of
`.Values.manifests.each_container_is_pod` variable, if it is set to `true` then
each container will be deployed as pod or if it is set to `false` then it will
do logical grouping of containers into a pod.

For example:

If `.Values.manifests.each_container_is_pod` is `true` then below pods and
respective containers are created within it

  ```yaml
    pods:
      - contrail-control
        containers:
          - contrail-control
          - contrail-dns
          - contrail-named
          - control-nodemgr
      - contrail-config
        containers:
          - config-api
          - schema-transformer
          - svc-monitor
          - device-manager
          - config-nodemgr
      - contrail-webui
        containers:
          - contrail-webui
          - contrail-middleware
      - contrail-analytics
        containers:
          - analytics-api
          - analytics-colletor
          - snmp-collector
          - query-engine
          - alarm-gen
          - contrail-topology
      - contrail-vrouter
        containers:
          - vrouter-kernel/vrouter-dpdk/vrouter-sriov
          - vrouter-agent
          - vrouter-nodemgr
  ```
If `.Values.manifests.each_container_is_pod` is `false` then for each container
it will create a pod.

Note: For contrail-thirdparty chart, by default it will create a separate pod
for each of the thirdparty service

### 4.4 Support for different orchestrators

Contrail helm deployer will support deploying contrail for below orchestrators

  * Openstack
  * Kubernetes


# 5. Performance and scaling impact

# 6. Upgrade

Upgrade of contrail cluster using helm charts will be covered as part of
contrail issu spec for 5.0


# 7. Deprecations

# 8. Dependencies

# 9. Testing
## 9.1 Unit tests
Unit tests will be added, which will validate that contrail charts work as
expected when it is installed


# 10. Documentation Impact

# 11. References
