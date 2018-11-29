# 1. Introduction

To provide Ansible playbooks for deployment of OpenStack and Contrail HELM charts.

# 2. Problem statement

As of today, OpenStack and Contrail HELM charts are deployed using multiple shell scripts. These deployment scripts are also different for Single node and Multi node environments. There are some manual steps that are performed as a pre-requisite for deployment. Also, the user has to continuously monitor for intermediate failures and there is no proper logging support. This entire deployment process is cumbersome and demands lot of user attention.

[ Blueprint ](https://blueprints.launchpad.net/juniperopenstack/+spec/helm-deployer-playbooks)

# 3. Proposed solution

The proposed solution is to use Ansible playbooks instead of shell scripts for deployment which removes the complexity and provides native logging facility.

## 3.1 Deployment Steps

The deployment can be divided into following steps and a playbook is provided for each step.
1. Configure nodes for Helm deployment
2. Install and setup Kubernetes cluster
3. Deploy OpenStack Helm charts
4. Deploy Contrail Helm charts

## 3.2 User Input

The user has to provide details about his/her Cluster environment (e.g. server details and their roles) and any required configuration details. To provide backward compatibility, the same input files that are used for deployment using shell scripts can be used here without any modification.

The inventory details should be specified in *helm_inventory.yaml* file present in **inventory** folder of the repo.
The configuration variables should be specified in *helm_vars.yaml* file present in **config** folder of the repo.

# 4. Alternatives considered

N/A

# 5. API schema changes

N/A

# 6. UI changes

N/A

# 7. Notification impact

N/A

# 8. Provisioning changes

OpenStack and Contrail HELM charts deployment is impacted. Below are the pre-requisites with this new deployment method.

## 8.1 Pre-requisite

Ansible playbooks should be present in contrail-helm-deployer repository so that user can get the playbooks by cloning the repo.
```
apt-get install git
git clone https://github.com/Juniper/contrail-helm-deployer.git
```

python should be present on all nodes for ansible to work.
sshpass is required for ansible to connect to nodes over SSH using a password.
```
apt-get install -y --no-install-recommends sshpass python-pip
```

Install Ansible on deployer node for running playbooks. Use the same Ansible version used by OpenStack Helm.
```
pip install --no-cache-dir --upgrade pip
hash -r
pip install --no-cache-dir --upgrade setuptools ansible==2.5.5
```

# 9. Implementation

The implementation provides Ansible playbooks and roles for each deployment step mentioned earlier.

# 10. Performance and scaling impact

N/A

# 11. Upgrade

N/A

# 12. Deprecations

N/A

# 13. Dependencies

N/A

# 14. Testing

N/A

# 15. Documentation Impact

As the deployment procedure is being modified from using shell scripts to running Ansible playbooks, the documentation should be modified completely to reflect the same.

# 16. References

N/A
