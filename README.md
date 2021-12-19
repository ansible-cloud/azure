# Ansible Azure Demo

This repository contains automation playbooks that can be used to test using Ansible with Microsoft Azure.

The directory structure of this project follows [directory conventions for Ansible Runner](https://ansible-runner.readthedocs.io/en/stable/intro/). 

Instructions for this project are written from the perspective of running the automation on your local machine.  However, the project may also be used directly with Ansible Automation Controller.  If you are using the later, then you will use Ansible Automation Controller credentials, job templates, etc. to setup the proper deployment.

# Requirements

## Applications

You will need to have the following installed and configured on your local host.

- Podman or Docker
- Python 3.8+
- Ansible, Ansible Runner, Ansible Navigator

## Azure Tenancy and Subscription

This demo assumes that you have permissions to create, manage, and destroy Azure resources such as Resource Groups, Virtual Machines, Networking, etc.  If you do not have access to a tenancy and subscription with access to those resources, then you will receive errors when attempting to run automation in later steps.

## Azure CLI

This guide will assume that the Azure CLI is using the default path `$HOME/.azure` as its path.

If you have the Azure CLI already installed on your local machine, then run `az login` to ensure that you have an active session.  You can skip to the next section.

If you do not have the Azure CLI installed on your local machine, then we can use a container to setup the CLI authentication without having to install the CLI on your local PC.

1. Create a directory in your home directory for the Azure configuration: `mkdir $HOME/.azure`
2. Pull a container with the Azure CLI: `docker pull bitnami/azure-cli:latest`
3. Login to Azure with the CLI in the container: `docker run -it --rm -v $HOME/.azure:/.azure bitnami/azure-cli:latest login`
4. Follow the instructions to login to Azure with your web browser. Once logged in, be sure to wait until the CLI recognizes the login.

## Create a Service Principal

1. Create a service principal for Ansible operations on Azure.
    - Running the CLI on your host: `az ad sp create-for-rbac --name ansible --role Contributor`
    - Running the CLI in a container: `docker run -it --rm -v $HOME/.azure:/.azure bitnami/azure-cli:latest ad sp create-for-rbac --name ansible --role Contributor`
2. Edit a new text file at `$HOME/.azure/credentials`
3. Paste the following replacing the values with the output of command in step 1.

```plaintext
[default]
subscription_id=xxxxxxx-xxxxx-xx-xxxxx
client_id=xxxxxxx-xxxxx-xx-xxxxx
secret=xxxxxxx-xxxxx-xx-xxxxx
tenant=xxxxxxx-xxxxx-xx-xxxxx
```

4. Save the file and exit.

# Instructions

## Setup 

There are a few steps that are required to configure this project.  Follow these steps to enable the automations.

1. Run the following command to create an `env` folder and environment files: `mkdir env; touch env/extravars`
2. Open the `env/extravars` file and add the following text replacing `<SSH-PUBLIC-KEY>` with your ssh public key. This will be the key that you use to ssh into deployed Linux servers.
```yaml
---
resource_group_name: "ansible_test"
region: "eastus"
vnet_cidr: "10.0.0.0/16"
subnet_cidr: "10.0.1.0/24"
vnet_name: "demo_vnet"
subnet_name: "demo_subnet"
network_sec_group_name: "demo_sec_group"
rhel_admin_user: "azureuser"
rhel_public_ip_name: "rhel_demo_ip"
rhel_nic_name: "rhel_demo_nic"
rhel_vm_name: "RHEL8-ansible"
rhel_vm_size: "Standard_DS1_v2"
rhel_vm_sku: "8.1"
rhel_public_key: "<SSH-PUBLIC-KEY>"

survey_public_ip: "True"

win_admin_user: "azureuser"
win_admin_password: "ChangeMeOnStartup12345"
win_vm_name: "WIN-ansible"
win_vm_sku: "2022-Datacenter"
win_vm_size: "Standard_DS1_v2"
win_public_ip_name: "win_demo_ip"
win_nic_name: "win_demo_nic"
```

## Run Playbooks

Each of the playbooks in this project can now be run using `ansible-navigator` or `ansible-runner`; just be sure to change the name of the YAML file to the name of the file that you want to run and add any required environment variables for the playbook that you need to run.

### Create a RHEL 8 Linux VM

The following command should be run from the root directory of this project as the example expects certain file paths following ansible runner directory conventions.  The playbook will create a RHEL 8 VM and all of the dependent resources to enable the VM that do not already exist.

```bash
ansible-navigator run project/create_rhel_vm_demo.yml -i inventory/hosts \
--pae false \
--extra-vars "@env/extravars" \
--mode stdout \
--ecmd vim \
--eei quay.io/scottharwell/azure-execution-env:latest \
--eev $HOME/.azure:/home/runner/.azure
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

If you get authentication errors when the automation runs, then you may need to perform the `docker run -it --rm bitnami/azure-cli:latest login` step again to enable a valid session.

### Create a Windows VM

The following command should be run from the root directory of this project as the example expects certain file paths following ansible runner directory conventions.  The playbook will create a Windows VM and all of the dependent resources to enable the VM that do not already exist.  If you intend to keep this server, then be sure to change the password once your VM is created.

```bash
ansible-navigator run project/create_windows_vm_demo.yml -i inventory/hosts \
--pae false \
--extra-vars "@env/extravars" \
--mode stdout \
--ecmd vim \
--eei quay.io/scottharwell/azure-execution-env:latest \
--eev $HOME/.azure:/home/runner/.azure
```

## Destroying Resources

Once resources are deployed, then you may incur charges in your Azure tenancy.  You may run the `destroy_resource_group.yml` playbook to remove all resources deployed with this demo to ensure that you're only charged for resources while testing.

```bash
ansible-navigator run project/destroy_resource_group.yml -i inventory/hosts \
--pae false \
--extra-vars "@env/extravars" \
--mode stdout \
--ecmd vim \
--eei quay.io/scottharwell/azure-execution-env:latest \
--eev $HOME/.azure:/home/runner/.azure
```
