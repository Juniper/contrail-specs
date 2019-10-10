# 1. Introduction

Contrail Multicloud architecture is built on the foundational building blocks of Contrail Networking and Security. This document will present the details of data model, network plumbing and automation to enable contrail multicloud gateway to extend contrail fabric to Google Cloud. Contrail Multicloud Gateway is the contrail vrouter with IPSec, SSL and BGP capabilities to enable secure connectivity.

# 2. Problem statement

Currently Contrail Multicloud deployment involves many static configuration and preparation steps like downloading third party dependencies (f.e. python, docker, etc.), downloading container images (f.e. contrail-nodemgr, contrail-multicloud-bird, etc.).
All these steps have to be repeated during CRUD operations (f.e. new compute node, or new gateway node has to download all dependencies). Additionally, these steps depend on third party repositories.
Temporary unavailability of these repositories can cause a failure in the multicloud deployment.

Via introduction of prebuilt os images with all dependencies and static configuration these issues can be mitigated and the Contrail Multicloud deployment can be stabilised and speeded up.
Introduced software will create os images for contrail-cni, contrail-controller, contrail-gateway and contrail-k8s-master and their combinations that can be made available from the market place of the public cloud providers.

Additionally, introduction of the prebuilt images will enable Contrail Multicloud deployment on the Google Cloud. On Google Cloud images used for deployment require a guest os feature 'MULTI_IP_SUBNET'.
This feature is not enabled on the publicly available RHEL, Ubuntu and Centos images, so separate images for the Contrail Multicloud deployment need to be created.


### Data model
Prebuilt images are stored using EBS-Backed AMIs on AWS, Google Compute Engine Images and Azure Compute Images.

Google Compute Engine Images can be used globally, so image built in one region can be used for starting instances in any other region.

EBS-Backed AMIs and Azure Compute Images are tied to specific regions, so image building process has to be repeated for each region used.

An example topology that would trigger a build images based on RHEL 7 os, on AWS and Azure:

AWS compute AMIs that will be built:
* contrail-gateway in the us-east-1 region
* contrail-gateway in the us-east-2 region
* image combining contrail-controller and contrail-k8s-master configurations in the us-east-2 region
* contrail-cni in the us-east-1 region
* contrail-cni in the us-east-2 region

Azure compute images that will be built:
* contrail-gateway in the CentralUS region
* contrail-cni in the CentralUS region


```
- provider: aws
  organization: juniper
  project: image-builder
  prebuild: r1911
  tags:
    owner: juniper
    Project: contrail_multicloud
    build_id: latest
  regions:
    - name: us-east-2
      vpc:
        - name: image-vpc-1
          cidr_block: 172.16.0.0/23
          subnets:
            - name: subnet_1
              cidr_block: 172.16.0.0/24
              availability_zone: a
          security_groups:
            - name: all_in
              ingress:
                from_port: 0
                to_port: 0
                protocol: "-1"
                cidr_blocks:
                  - "0.0.0.0/0"
            - name: all_out
              egress:
                from_port: 0
                to_port: 0
                protocol: "-1"
                cidr_blocks:
                  - "0.0.0.0/0"
          instances:
            - name: GW1
              availability_zone: a
              roles:
                - gateway
              provision: true
              os: rhel7
              instance_type: t2.large
              volume_size: 10
              security_groups:
                - all_out
                - all_in
              subnets: subnet_1
              interface: eth1

            - name: controller
              availability_zone: a
              provision: true
              roles:
                - controller
                - k8s_master
              os: rhel7
              instance_type: t2.xlarge
              volume_size: 100
              security_groups:
                - all_out
                - all_in
              subnets: subnet_1
              interface: eth0

            - name: Compute1
              availability_zone: a
              provision: true
              roles:
                - compute_node
              os: rhel7
              instance_type: t2.medium
              volume_size: 24
              security_groups:
                - all_out
                - all_in
              subnets: subnet_1
              interface: eth0

    - name: us-east-1
      vpc:
        - name: image-vpc-2
          cidr_block: 192.168.0.0/23
          subnets:
            - name: subnet_2
              cidr_block: 192.168.0.0/24
              availability_zone: a
          security_groups:
            - name: all_in_2
              ingress:
                from_port: 0
                to_port: 0
                protocol: "-1"
                cidr_blocks:
                  - "0.0.0.0/0"
            - name: all_out_2
              egress:
                from_port: 0
                to_port: 0
                protocol: "-1"
                cidr_blocks:
                  - "0.0.0.0/0"
          instances:
            - name: GW2
              availability_zone: a
              roles:
                - gateway
              provision: true
              os: rhel7
              instance_type: t2.large
              security_groups:
                - all_out_2
                - all_in_2
              subnets: subnet_2
              interface: eth1

            - name: compute2
              availability_zone: a
              provision: true
              roles:
                - compute_node
              os: rhel7
              instance_type: t2.medium
              security_groups:
                - all_out_2
                - all_in_2
              subnets: subnet_2
              interface: eth0
              
- provider: azure
  organization: juniper
  project: image-builder
  prebuild: r1911
  tags:
    owner: juniper
    Project: contrail_multicloud
    build_id: latest
  regions:
    - name: CentralUS
      resource_group: contrail-multicloud
      vnet:
        - name: image-vnet-1
          cidr_block: 172.22.0.0/23
          subnets:
            - name: subnet1testbuild
              cidr_block: 172.22.0.0/24
              security_group: allowallprotocolstestbuild
          security_groups:
            - name: allowallprotocolstestbuild
              rules:
                - name: allinbuild
                  direction: inbound
                - name: alloutbuild
                  direction: outbound
          instances:
            - name: gw2azuretestbuild
              roles:
                - gateway
              provision: true
              os: rhel7
              instance_type: Standard_F2
              subnets: subnet1testbuild
              interface: eth1
              username: redhat
            - name: gw1azuretestbuild
              roles:
                - gateway
              provision: true
              os: rhel7
              instance_type: Standard_F2
              subnets: subnet1testbuild
              interface: eth1
              username: redhat
            - name: compute2azuretestbuild
              roles:
                - compute_node
              provision: true
              os: rhel7
              instance_type: Standard_F2
              subnets: subnet1testbuild
              interface: eth1
              username: redhat
            - name: compute3azuretestbuild
              roles:
                - compute_node
              provision: true
              os: rhel7
              instance_type: Standard_F2
              subnets: subnet1testbuild
              interface: eth1
              username: redhat
```

The topology file can be manually edited using example templates from the contrail-multicloud-deployer OR use UI / GoAPI workflow that would render the templates and create the topology definition. 

### Provisioning
UI / GoAPI will invoke contrail-multi-cloud API / functions which will build the relevant images to the various regions of the public cloud provider. The images that will be built will be:

- contrail-cni: Contains contrail vrouter and k8s node
- contrail-gateway: Contains, BGP, openvpn, strongswan and contrail vrouter
- contrail-controller: Contains contrail config, control and analytics
- contrail-k8s-master: Container k8s master and contrail kube manager

Based on the roles that are provided in the instances, the appropriate images are build and initialized

#### Deployment model:

The images are built at the time of deployment based on the instance's os, specified roles and region of the instance (region is irrelevant on the Google Cloud as Google Compute Images are globally available).
Created images are assigned tags with information about it's base os, prebuild_id (prebuild field from the topology) and assigned roles. These tags are then used by terraform filters that look for appropriate
images that should be used for creation of particular instance.


#### Required Software

boto3 - 1.9.11 - Python SDK for AWS API

google-api-python-client - 1.7.9 - Python SDK for Google Cloud API

azure-mgmt-compute - 6.0.0 - Python SDK for Azure Compute Management API

Packer - 1.3.5 - Hashicorp's tool that automates provisioning and saving of the os images

# 3. Proposed solution

1. Additional script is executed during the Contrail Multicloud Deployment. The additional 'image_builder' script scans the topology file, looks for required images that need to be created on different cloud providers
and on different regions. Script prepares a temporary file for the Hashicorp's Packer tool with descriptions of needed images and Ansible playbooks that need to be executed on them as part of the provisioning. Additional
'prebuild' field is added to the topology file, next to the 'provisioner' field as can be seen in the Data Model section. When looking for required images, 'image_builder' script scans only the fields listed under 
provider with the 'prebuild' field specified.

2. Packer is executed, taking the file created in the 1st step as input. It creates temporary instances, one for each image that has to be built. These instances are provisioned by scripts specified in the input file.
After provisioning, Packer saves snapshots of these images. Created images are tied to the user's account. 

3. Contrail Multicloud deployment continues as before. Only difference is that the Terraform tool used for creating cloud resources, instead for creating instances based on clean os images (f.e. on base RHEL 7 image),
uses the images created by Packer in the 2nd step. Ansible tasks that download dependencies or perform some static configuration will be skipped as they have been already executed during the 2nd step. 


## 3.1 Alternatives considered

There are plans to publish the created os images to the AWS, GCP and Azure image Marketplaces, so that the 1st and 2nd step form the previous section will not be executed by the user. It will make the future releases 
much more stable as the created images will have all third party dependencies included, so they will be available even when their original sources (package repositories, docker registries) will be taken down.


## 3.2 GO API schema changes
GoAPI will require changes in the templates to trigger the backend to process the prebuild tags and build the images. Section 3 - Topology Definition, 1 and 2 describes the changes that is required in the topology file.

## 3.3 User workflow impact
There is no deviation to the user workflow to create and use the images. Image builder and using the images will be transparent to the user.

## 3.4 UI changes
Additional field 'Version Id' is added to the Cloud creation screen. It corresponds to the 'prebuild' field added to the topology file. The 'Version Id' field has a tooltip that explains to the user that the field
determines the tag assigned to created images.

## 3.5 Notification impact
#### N/A

# 4. Implementation

## 4.1 Image builder

The 'image_builder' script renders the required templates from the values in the topology to build the images. The those templates will be used as the input for the Hashicorp's Packer tool that will create and
provision the required images. 

# 5. Performance and scaling impact

## 5.1 API

## 5.2 Provisioning

With the pre-build images, provisioning of the instances will be faster as the required microservices will be readily packaged in the images.

#### N/A

## 5.2 Forwarding performance
#### N/A

# 6. Upgrade
#### Backward compatible changes

# 7. Deprecations
#### N/A

# 8. Dependencies

# 9. Testing
## 9.1 Unit tests
Contrail multicloud cloud has UT, code style and extensive gate jobs to ensure code coverage. Image builder tests are run as part of CI jobs.

## 9.2 Dev tests
Contrail-multi-cloud will run extensive gate jobs to ensure code coverage and CrUD test cases for building the images and using the images in AWS / Google and Azure clouds.
Dev team also runs End-to-End tests as part of the tasks.

## 9.3 System tests

# 10. Documentation Impact
The additional 'Version Id' field on the Cloud creation screen will be included in the documentation.

# 11. References
[Architecture]: (https://docs.google.com/document/d/1bQ0Qm8zHgF9YwOwZ8NEKymoEaDiSjD5TYw4Gj1BPU-c/edit#)