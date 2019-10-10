# 1. Introduction

Contrail Multicloud enables connection of different public clouds with on-premise cloud through Contrail-based services and networking. This documents describes extension of supported public clouds to include Google Cloud Platform or GCP.

# 2. Problem statement

There is a need to use GCP-based instances together with Contrail Multicloud.

Usecases for this solution:

* User wants to connect many different GCP VPCs and use Contrail networking between them
* User wants to extend On-Prem Contrail Cluster to public cloud, using GCP provider.

# 3. Proposed solution

Add support for GCP-based clouds to Contrail Multicloud

# 4. Alternatives considered
N/A

# 5. Api/Schema changes
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

# 6. UI changes

User will be able to choose GCP provider in Add new cloud form. This form will contain relevant fields for creation of a new GCP cloud.

User will be able to upload GCP credentials file.

# 7. Notification impact

Consistent with other clouds, user will receive information about:
* When public cloud resources are created
* When public cloud is connected to on-premise cluster
* When there is an error in public cloud

# 8. Provisioning changes

Provisioning changes required by this feature employ creation of specific GCP image. This will be done by Packer (another blueprint).
Alternatively, supported only by API directly, user can create their own GCP image with MULTI_IP_SUBNET flag set up.

# 9. Implementation

## 9.1 Intent definition
Intent for GCP has to be defined in GCP domain language. User has to declare things such as firewalls, subnets and virtual machines, as well as roles for them. Sample can be seen in point #5.

Some considerations:
* GCP uses firewalls instead of Security Groups like AWS and Azure.
* Firewalls on GCP apply to the whole VPC
* Internal Firewall is used for traffic intra-VPC
* External Firewall is used for traffic inter-VPC, for example Internet connection


## 9.2 Preparation of images
Images have to be prepared before deployment. This will be done by packer automatically.

## 9.3 Cloud resources
Cloud resources will be created using Terraform's `google` plugin.
Based on the intent/topology, a terraform `google.tf.json` file will be created. This file will contain the intent transformed into terraform cloud resources. Terraform binary will read this file, accept user credentials and create relevant cloud resources.

## 9.4 Deployment
Deployment will be almost the same for GCP as for other cloud providers.

On GCP, some changes to provisoning are requied:
* gcloud to manage GCP resources
* Different VRRP scripts

## 9.5 VRRP
VRRP used to switch between active-standby mode on gateways is doing so by directly using google-provided tool to manage cloud resources.

## 10. Performance and scaling impact
This feature doesn't modify in any way lower bound performance scenario, as in all tests the bottleneck was Azure.

In regards to stability, no differences were found between GCP and other providers.

The performance of working GCP-based cloud is dependant mainly on user's choice of instance types.

## 10.1 Number of gateways
The only supported mode of operation based on Contrail Command is currently full mesh deployment. This means the complexity of setup can be described as O(n^2), where n is number of gateways.

Using API directly supports hub-and-spoke mode, where performance is O(n^2+n*m), where n is number of hubs and m is number of spokes. This can be sometimes be more effective.

## 10.2 Number of computes
The performance is not dependant on number of computes directly. However, assuming every compute will have the same traffic, Performance requirements are described as O(n) where n is number of computes

## 11. Upgrade
N/A

# 12. Deprecations
None

# 13. Dependencies
To fully support GCP deployments, Packer support is necessary. See references

# 14. Security Considerations
The credentials to Google Cloud Platform have to removed after every deployment. This means time where an attack is possible is minimal, only during execution of Terraform scripts.

Leak of these credentials can mean significant costs if those credentials are used by attackers to create cloud resources.

# 15. Testing
This change requires through testing.

# 15.1. Regression testing
Tests should be done if GCP support doesn't break non-GCP deployments.

# 15.2. General
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

# 15.3 Compatibility
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

# 15.4 High Availability
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

# 15.5 Sanity
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

# 16. Documentation Impact
The new alarms and config settings for vrouter and agent needs to be documented.


# 17. References

https://contrail-jws.atlassian.net/browse/CEM-9003 - Jira task for Blueprint
https://contrail-jws.atlassian.net/browse/CEM-9004 - Jira task for Tech Spec
https://contrail-jws.atlassian.net/browse/CEM-8998- Jira task for Blueprint for Packer
https://contrail-jws.atlassian.net/browse/CEM-8999 - Jira task for Blueprint for Packer
