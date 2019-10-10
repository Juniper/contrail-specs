# 1. Introduction

Contrail Multicloud image builder module stabilizes and eases the deployment of the Contrail Multicloud topologies by creating preconfigured system images on AWS, GCP and Azure. 
This document will present the use cases, design and deployment process of the module.

# 2. Scope

Scope of this expansion is creation of an executable which creates specified images on cloud providers: GCP, AWS and Azure.

Created images contain most of the required data pre-downloaded, so the deployment using them will be much faster.

The scope also includes modifying MultiCloud scripts to support the images.

# 3. Use Case

## 3.1 Building images by user

As a user I want to deploy Contrail Multicloud using preconfigured system images with downloaded container images and other deployment dependencies.
Contrail Multicloud Image Builder will parse user's topology and prepare the required system images. Images can be used in the Contrail Multicloud deployment process. Furthermore, they will be used in the subsequent CRUD operations on the deployment.


# 4. Functional Representation

This module is executed after preparing the `topology.yml` file for `generate_topology.py`, but before actually launching `generate_topology.py`.

Execution of this module has been added as an additional function executed by the Python CLI, which abstracts the sequence of running different
scripts from the user.

The function provided by this step is to deploy a partial machine, save it to the cloud image and use it as a base in future deployments.

# 5. Design Goals and Constraints

The main factors considered during designing this solution ere:
* Portability to other cloud providers (ex. Azure)
* Speed of development
* Compatibility with the current Multicloud model
* Testability in CI
* Automation

Some constraints were:
* Time limit for building the images
* Compatibility with current Ansible model
* Low costs of the user
* Error resistance

# 6. Software Design

## 6.1 Models

Models used in the scripts:

### 6.1.1 ImageConfig

ImageConfig models a single Image to be built.

An object of this class aggregates information about  the target provider, region, distribution and other data. There will be many ImageConfigs generated per an execution.

### 6.1.2 PackerTemplateBuilder

PackerTemplateBuilder class is responsible for creating the Packer template file.

Packer template files are used as configuration by Packer. There are a few segments in such a file, each defining a stage of image building process:
* Provisioners - in case of this project, this is running ansible playbook on the intermediate machine
* Deprovisioner - responsible for cleaning up
* Builder - responsible for creation of cloud resources

### 6.1.3 CloudApiInstances

CloudApiInstances is a class used to provide a higher-level abstraction api for managing instances in the cloud.

This includes features such as copying images onto other regions or removing unnecessary images

### 6.1.4 CloudApiUtility

CloudApiUtility is an abstract class representing calls to a single cloud API provider.

Implementation of this class provide such features as listing existing images, tagging images, listing available images, etc.

There are two classes derived from CloudApiUtility at the moment:
* GCPApiUtility
* AwsApiUtility
* AzureApiUtility


## 6.2 Schema and API

Usage of this module is as follows:
```
usage: deployer all topology [-h] [-s SECRETS_PATH] [-t TOPOLOGY]
```
In all cases, `SECRETS_PATH` and `TOPOLOGY` has to be supplied. 


## 6.3 Workflow
The script checks, which APIs have to be used then initializes relevant objects.

Module then calculates required images and issues a request to the cloud API to check what images are already available.

After calculating the difference, which is required images, the script starts building relevant images and eventually copies them to other regions.

## 6.4 Algorithms and Design patterns
The script uses set arithmetic to calculate required images.

Api client classes are designed with singleton principle.

Api client classes are created using a base abstract class.

PackerTemplateBuilder class is designed using the builder design pattern.

# 7. Process View

## 7.1 Functionality

The intended use of this module is a single launched process, Python interpreter using the supplied file. This process will spawn a few other processes as noted in IPC.

## 7.2 IPC
The inter-process communication is achieved via calling `boto3` for AWS and `google-api-python-client` for GCP.

Packer itself is called as a child process, using standard Python process management.

## 7.3 Persistence

The artifacts of the build persist in the relevant cloud. Other byproducts (such as VMs) will be removed upon successful completion.

## 7.4 Failure handling

Possible-to-handle failures will be handled in software.

Due to the cloud API-facing nature of the software,  not all issues can be automatically resolved, such as insufficient quotas or inaccessible servers. In such cases, an error message is displayed.

# 8. Deployment view

## 8.1 Definition
After deployment, nodes launched using image builder should be indistinguishable from nodes launched using the classic method.

## 8.2 Provsioning
Provisioning is the same as in non-packer method.

# 9. Size and Performance
The deployment is much faster, thanks to not having to download Contrail and kubernetes container images, which is the major bottleneck of regular deployment.

# 10. Quality metrics

Sample quality metrics can be:
* Speed increase when using images compared to base deployment.
* Stability of nightly builds

## 10.1 Tests

Multicloud team implemented packer tests into CI, as well as a nightly image building job.

Multicloud team performed manual end-to-end tests to confirm that the feature works on GCP, Azure and AWS

More, manual tests can be performed by QA to assure quality.

# 11. Management and Monitoring

Management of the image building process is done via specyfing relevant options and providing intent (topology).

Monitoring is provided by log messages spread through the program. User can easily check the progress of all images currently being built.

Management and monitoring of the prepared images is done via specific cloud provider's tools, such as CLI or WebUI.