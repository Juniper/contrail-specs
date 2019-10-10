# 1. Introduction

Contrail Multicloud architecture is built on the foundational building blocks of Contrail Networking and Security. This document will present the details of data model, network plumbing and automation to enable contrail multicloud gateway to extend contrail fabric to Google Cloud. Contrail Multicloud Gateway is the contrail vrouter with IPSec, SSL and BGP capabilities to enable secure connectivity 

# 2. Scope

Scope of this change is introduction of GCP-based VPCs on Contrail Multicloud. This ranged from backend/API support to UI changes.

# 3. Use cases

# 3.1 Connection of GCP-based VPC with other VPCs
Contrail Multicloud will extension of an already existing Contrail cluster to a public GCP VPC.

# 3.2 Connection of multiple GCP-based VPCs
Contrail Multicloud will allow transfer between different, GCP-based VPCs, by providing a secure VPN and routing broadcasting.

# 3.3 Connection of GCP-based VPC with other VPCs
Contrail Multicloud will allow transfer between GCP-based VPCs, and VPCs on other providers' clouds, such as AWS and Azure, by providing a secure VPN and routing broadcasting.

# 4. Functional Representation
Contrail Multicloud enables users to manage cloud-to-cloud networking by using Contrail vRouter deployed on public cloud gateways and compute nodes. This enables the user to manage and monitor networking inter and intra cluster.

Contrail Multicloud's deployment is automated. This is done by using Python scripts, Ansible and Terraform.

Contrail Multicloud enables cloud-to-cloud connectivity using user's choice of either VxLAN, IPSec or OpenVPN tunnels. VxLAN tunnels are not encrypted, while IPSec and OpenVPN are secure tunnels.
Routing on Contrail Multicloud is handled by using BGP protocol, provided by BGPBird daemon.
Multicloud supports HA in a active-standby gateway model. This is done by using VRRP.

Contrail Command enables user to user Contrail Multicloud in a friendly, intuitive way, create new cloud deployments as well as configure existing ones.

# 5. Design Goals and Constraints

The design goals for secure connectivity to Google Cloud Platform are:

* Connect GCP VPCs to existing Contrail installations
* Provide secure transmission between GCP VPCs and other Contrail Multicloud clouds
* Automate as much work as is possible for the user
* Create, update and delete GCP VPCs on demand
* Interpret user-provided topology request into cloud resources consistent with other, already-implement clouds and deploy them
* Advertise routing changes between GCP-based cloud and other Contrail clusters
* Recover from failures
* Idempotent creation of clouds is the final goal.
* Apply GCP routing requirements to TOR switches OnPremise

Constraints specific to Google Cloud Platform integration are:

* To be consistent with other clouds, GCP VMs require specially prepared images. These can be done by packer, which is another feature or created by the user manually (not supported via UI)

# 6. Software Design

## 6.1 Models

Terraform data model abstrated for the generic objects and will have specific implementation for the encapsulated google cloud objects. 

class BaseResourceGCP will be the abstract class and class ProviderGCP will provide specific implementation. 

Each Type of objects in google cloud plugin of terraform is represented by a class. 

Ex: TYPE: google_compute_network - class ComputeNetworkGCP.

Objects will be initialized as part of processing the topology.yml for generting the terraform templates for each provider.

Models used for GCP support:
* BaseResourceGCP
* ProviderGCP
* ComputeNetworkGCP
* ComputeSubnetGCP
* FirewallGCP
* InstanceGCP
* AddressGCP
* RouteGCP
* VPCPeeringGCP (not yet usable from UI)
* ImageFilterGCP (required Packer changes)

## 6.2 Schema and API

Add a new supported provider to `topology.yml`:
```yml
- provider: google
  organization: <organization name>
  project: <project name>
  regions:
    - name: <region name>
      vpc:
        - name: <vpc name>
          subnets:
            - name: <subnet name>
              cidr_block: <cidr>
          firewalls_external:
            - name: <firewall name>
              allow:
                protocol: <protocol name>
            - name: <firewall name>
              allow:
                protocol: <protocol name>
          firewalls_internal:
            - name: <firewall name>
              allow:
                protocol: <protocol name>
            - name: <firewall name>
              allow:
                protocol: <protocol name>
          instances:
            - name: <instance name>
              roles:
               - gateway
               - compute
               - controller
               - k8s_master
              provision: <true|false>
              username: <username>
              instance_type: <instance type>
              subnets: <subnet name>
              protocols_mode: # optional, only for gateway
                - ipsec_client
                - ipsec_server
                - ssl_client
                - ssl_server
            - name: <instance name>
              provision: <true|false>
              username: <username>
              roles:
               - gateway
               - compute
               - controller
               - k8s_master
              os: <ubuntu16 | centos7>
              instance_type: <instance type>
              subnets: <subnet name>
              count: <count of instance with same config>
              protocols_mode: # optional, only for gateway
                - ipsec_client
                - ipsec_server
                - ssl_client
                - ssl_server
```

Add a `gcp` type to enum of supported `cloud_provider` types in GoAPI.
```yml
id: cloud_provider
metadata:
  category: cloud
parents:
  cloud:
    operations: "CRUD"
    description: "Parent for cloud provider"
    presence: "optional"
plural: cloud_providers
prefix: /
schema:
  properties:
    type:
      description: Cloud Provider type
      default: private
      enum:
      - aws
      - azure
      - gcp
      - private
```
Add a `google_credential` field to `cloud_user` in GoAPI

```yml
description: Cloud User Details
extends:
- base
id: cloud_user
metadata:
  category: cloud
references:
  credential:
    operations: "CRUD"
    description: "Reference to SSH credential object."
    presence: "optional"
plural: cloud_users
prefix: /
schema:
  properties:
    aws_credential:
      presence: "optional"
      description: "AWS Credential Details"
      $ref: "cloud_types.json#/definitions/AWSCredential"
    gcp_credential:
      presence: "optional"
      description: "GCP Credential Details"
      $ref: "cloud_types.json#/definitions/GCPCredential"
  type: object
singular: cloud_user
title: Cloud User
type: ""
```

## 6.3 Workflow

## 6.4 Algorithms and Design patterns

Bridge design pattern is used to expose the abstract functions as facade and provide concrete implementation for the objects, specific to Google cloud.

Builder -> BuilderGCP
BaseResouce -> BaseResourceGCP
Provider -> ProviderGCP

Defination and implementation of dictionary, List and map based on Python is used to satisfy the object creation, lookup and computation.

# 7. Process View

Command container has the Go process, SQL engine / process for postgres and Ansible. Contrail Multicloud  has Python modules to generate the terraform templates and the inventory that is required for ansible to provision the instances in the public cloud.

## 7.1 Functionality

Public cloud credentials: Get the public cloud account credentials. This can be either provided to the deployer along with the ssh keys OR command / deployer will have the LDAP hooks to lookup the account credentials for the user.

Deployer:  This is the container, that has all the dependencies pre-built and ready for interacting with the Multicloud environments. It has the contrail-ansible-deployer and the scripts that will provide single click provisioning of the entire Multicloud environment. 

Define Topology: This is the topology definition that includes all the resources that has to be defined in the different public cloud environments. Objects such as VPC / VNET, subnet, routes, security groups, instances / EC2 / virtual machines can be defined.

Topology Compiler:  This will take in the topology definition and generates the terraform classes required for applying the topology with all the objects defined onto the corresponding public cloud provider

Terraform Apply: Terraform from HashiCorp is an orchestration tool that is Go-lang based. It has all the required API definitions to call the public cloud providers API to create the resources as defined in the topology definition. 

Inventory Generator: The logical to physical mapping of resources is done in the inventory generator. Once terraform apply completes the execution, the logical objects defined in the topology definition get the physical attributes like, IP, hostname, etc.   Inventory files have the required details for Ansible provisioning. 

Ansible provisioning: There are 2 parts to this Ansible provisioning. 

1- Gateway Provisioning: As part of gateway provisioning, the objects in the different public cloud environments are initialized. This entails, loading up of the required drivers, plumbing the interfaces, configuring interfaces, setting up the cloud specific routing objects, installing docker, creating loopback interfaces for running OSPF, BFD and BGP, configure the IPSec IKEv2 and or SSL, configure BGP, configure iptables, SNAT and HA agent. 

2- Once the above configuration is complete, secure connectivity is established across the different gateways that advertises the local prefix blocks to the peers. Contrail-ansible-deployer will now provision contrail and the orchestrator as per the roles defined in the topology definition.  

## 7.2 IPC

Inter process communication is via the RESTFull interface and GO API calls.

GoAPI communicated with Ansible by standard process execution.

## 7.3 Persistence

Multicloud objects is stored in the postgres database tables. In the container the objects is stored in the terraform state files.
Credentials are not stored and are deleted after every operation where they're required. This is for security reasons.

## 7.4 Failure handling

Multicloud backend has the implementation for handling the failures and auto re-try on failure while creating the terraform objects in the public cloud and the ansible playbook retries.

# 8. Deployment view

## 8.1 Definition
## 8.2 Provsioning

User -> Multi-cloud-deployer -> Terraform -> Google Cloud
     -> Multi-cloud-deployer -> Ansible -> Google Cloud

User -> Command UI -> GoAPI -> Multi-cloud backend -> Terraform -> Google Cloud
     -> Command UI -> GoAPI -> Multi-cloud backend -> Ansible -> Google Cloud

# 9. Size and Performance

Multicloud backend is tested to scale upto 1K computes across 10 GWs. 

# 10. Quality metrics
* Speed of provisioning
* Resilience in case of one/multiple VMs going down
* Speed of data transfer
* Reliability of provisioning

## 10.1 Tests
This change requires through testing.

### 10.1.1 Regression testing
Tests should be done if GCP support doesn't break non-GCP deployments.

### 10.1.2 General
Test shoud be done to create a VPC connected to Contrail in GCP. A sample scenario that will test the broadest scope:

* On-Prem cluster
  * 2 Compute Nodes
  * 2 Gateways
  * 2 Controller Nodes
  * Contrail Command Node
* GCP cluster
  * 2 Compute Nodes
  * 2 Gateways

  The expected scenario is:
  * Pods can be ran on any of the Compute Nodes and on none of other hosts
  * Pods can communicate with each other by default
  * Pods can no longer communicate after Contrail configuration to block such communication

### 10.1.3 Compatibility
Test should check whether GCP clouds interoperate with other Clouds

Sample scenario:
* On-Prem cluster
  * 2 Compute Nodes
  * 2 Gateways
  * 2 Controller Nodes
  * Contrail Command Node
* GCP cluster
  * 2 Compute Nodes
  * 2 Gateways
* AWS cluster
  * 2 Compute Nodes
  * 2 Gateways
* Azure cluster
  * 2 Compute Nodes
  * 2 Gateways

  The expected result is:
    * Pods can be ran on any of the Compute Nodes and on none of other hosts
  * Pods can communicate with each other by default
  * Pods can no longer communicate after Contrail configuration to block such communication

### 10.1.4 High Availability
VRRP should ensure HA on the cloud. Sample scenario:

* On-Prem cluster
  * 2 Compute Nodes
  * 2 Gateways
  * 2 Controller Nodes
  * Contrail Command Node
* GCP cluster
  * 2 Compute Nodes
  * 2 Gateways

The expected scenario is:
  * Pods can communicate with each other
  * After disabling either of gateways, the traffic still works. TCPDUMP can be used to verify the active routing is switched.

### 10.1.5 Sanity
Sanity checks have to be performed to ensure the resources are created correctly, on the  GCP.
Sample scenario:

* On-Prem cluster
  * 2 Compute Nodes
  * 2 Gateways
  * 2 Controller Nodes
  * Contrail Command Node
* GCP cluster
  * 2 Compute Nodes
  * 2 Gateways

The expected result is:
*  All entities can be seen in Google Cloud Console while logging in with the same account as the one used to provision CMC.


# 11. Management and Monitoring
## 11.1 Management
The setup can be managed via the UI or modyfing the topology file.
User can create additional VMs, remove them, create new VPC or security rules.
User can extend the setup to include public clouds of other providers, such as AWS or Azure

## 11.2 Monitoring
Contrail Command provides some information about the state of Contrail Multicloud, such as provisioning success/failure.

Current terraform status, such as which resources are being deployed can be seen in `/var/log/contrail/cloud.log` file on Contrail Command server.

Current provisioning status, such as failures or Ansible status can be seen in `/var/log/contrail/deploy.log` file on Contrail Command server.