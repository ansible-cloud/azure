# Ansible Azure Demo

This repository contains automation playbooks that can be used to test using Ansible with Microsoft Azure.

The directory structure of this project follows [directory conventions for Ansible Runner](https://ansible-runner.readthedocs.io/en/stable/intro/).  Ansible Runner can be used to run playbooks both locally and with execution environments. 

Instructions for this project are written from the perspective of running the automation on your local machine.  However, the project may also be used directly with Ansible Automation Controller.  If you are using the later, then you will use Ansible Automation Controller credentials, job templates, etc. to setup 
# Requirements

## Applications

You will need to have the following installed and configured on your local host.

- Podman or Docker
- Python 3.8+
- Ansible and Ansible Runner

## Azure Tenancy and Subscription

This demo assumes that you have permissions to create, manage, and destroy Azure resources such as Resource Groups, Virtual Machines, Networking, etc.  If you do not have access to a tenancy and subscription with access to those resources, then you will receive errors when attempting to run automation in later steps.

## Azure CLI

This guide will assume that the Azure CLI is using the default path `$HOME/.azure` as its path.

If you have the Azure CLI already installed on your local machine, then run `az login` to ensure that you have an active session.  You can skip to the next section.

If you do not have the Azure CLI installed on your local machine, then we can use a container to setup the CLI authentication without having to install the CLI on your local PC.

1. Create a directory in your home directory for the Azure configuration: `mkdir $HOME/.azure`
2. Pull a container with the Azure CLI: `docker pull bitnami/azure-cli:latest`
3. Login to Azure with the CLI in the container: `docker run -it --rm bitnami/azure-cli:latest login`
4. Follow the instructions to login to Azure with your web browser. Once logged in, be sure to wait until the CLI recognizes the login.

# Instructions

## Setup 

There are a few steps that are required to configure this project.  Follow these steps to enable the automations.

1. Run the following command to create an `env` folder and environment files: `mkdir env; touch env/envvars; touch env/extravars`
2. Open the `env/envvars` file and add the following text replacing `<PASSWORD>` with a password that you want deployed to a deployed windows server
```yaml
---
WINDOWS_PASSWORD: "<PASSWORD>"
```
3. Open the `env/extravars` file and add the following text replacing `<SSH-PUBLIC-KEY>` with your ssh public key. This will be the key that you use to ssh into deployed Linux servers.
```yaml
---
resource_group_name: "ansible_test"
region: "eastus"
vnet_cidr: "10.0.0.0/16"
subnet_cidr: "10.0.1.0/24"
vnet_name: "demo_vnet"
subnet_name: "demo_subnet"
network_sec_group_name: "demo_sec_group"
rhel_public_ip_name: "rhel_demo_ip"
rhel_nic_name: "rhel_demo_nic"
rhel_vm_name: "RHEL8-ansible"
rhel_vm_size: "Standard_DS1_v2"
rhel_public_key: "<SSH-PUBLIC-KEY>"

survey_public_ip: "True"

win_vm_name: "WIN-ansible"
win_vm_size: "Standard_DS1_v2"
win_public_ip_name: "win_demo_ip"
win_nic_name: "win_demo_nic"
```

## Run Playbooks

Each of the playbooks in this project can now be run using the following command; just be sure to change the name of the YAML file to the name of the file that you want to run.

The following command should be run from the root directory of this project as `ansible-runner` expects the directory convention that has been created.

```bash
ansible-runner run \
--process-isolation \
--process-isolation-executable docker \
--container-image quay.io/scottharwell/azure-execution-env:0.0.2 \
--playbook create_rhel_vm_demo.yml \
--container-volume-mount=$HOME/.azure:/home/runner/.azure \
./
```

Output will be similar to running the playbook locally on your machine, but you have run the playbook in an execution environment!

```text
PLAY [Create Azure VM] *********************************************************

TASK [Create resource group] ***************************************************
changed: [localhost]

TASK [Create virtual network] **************************************************
changed: [localhost]

TASK [Add subnet] **************************************************************
changed: [localhost]

TASK [Create public IP address] ************************************************
changed: [localhost]

TASK [Dump public IP for VM which will be created] *****************************
ok: [localhost] => {
    "msg": "The public IP is 20.85.219.123"
}

TASK [Create Network Security Group that allows SSH and RDP] *******************
changed: [localhost]
...
```

If you get authentication errors when the automations run, then you may need to perform the `docker run -it --rm bitnami/azure-cli:latest login` step again to enable a valid session.

## Destroying Resources

Once resources are deployed, then you may incur charges in your Azure tenancy.  You may run the `destroy_resource_group.yml` playbook to remove all resources deployed with this demo to ensure that you're only charged for resources while testing.

```bash
ansible-runner run \
--process-isolation \
--process-isolation-executable docker \
--container-image quay.io/scottharwell/azure-execution-env:0.0.2 \
--playbook destroy_resource_group.yml \
--container-volume-mount=$HOME/.azure:/home/runner/.azure \
./
```
