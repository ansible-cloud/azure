![Validation Checks](https://github.com/ansible-cloud/azure/actions/workflows/on-push.yml/badge.svg)

# Ansible Azure Demo

This repository contains automation playbooks that can be used to test using Ansible with Microsoft Azure.  This repository and the examples within are not supported by Red Hat and are for example purposes only.

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

```ini
[default]
subscription_id=xxxxxxx-xxxxx-xx-xxxxx
client_id=xxxxxxx-xxxxx-xx-xxxxx
secret=xxxxxxx-xxxxx-xx-xxxxx
tenant=xxxxxxx-xxxxx-xx-xxxxx
```

4. Save the file and exit.

# Instructions

## Resource Demos

This section applies to the resource demo playbooks.

* `create_resource_group.yml`
* `create_rhel_vm_demo.yml`
* `create_windows_vm_demo.yml`
* `destroy_resource_group.yml`
* `update_rhel_vms.yml`

There are a few steps that are required to configure this project if you intend to run the playbooks with `ansible-navigator`.  The extra vars can be directly copied to Ansible Controller templates when using Ansible Controller.  Not all variables are used in each playbook; you may omit unused extra vars in Ansible Controller on a per-template basis.

1. Run the following command to create an `env` folder and environment files: `mkdir env; touch env/extravars`
2. Open the `env/extravars` file and add the following text replacing `<SSH-PUBLIC-KEY>` with your ssh public key. This will be the key that you use to ssh into deployed Linux servers.

```yaml
---
resource_group_name: "ansible_test"
region: "eastus"
vnet_cidr: "172.16.1.0/24"
subnet_cidr: "172.16.1.0/24"
vnet_name: "demo_vnet"
subnet_name: "demo_subnet"
network_sec_group_name: "demo_sec_group"
rhel_admin_user: "azureuser"
rhel_public_ip_name: "rhel_demo_ip"
rhel_nic_name: "rhel_demo_nic"
rhel_vm_name: "RHEL8-ansible"
rhel_vm_size: "Standard_A1_v2"
rhel_vm_sku: "8_5"
rhel_public_key: "<SSH-PUBLIC-KEY>"

survey_public_ip: true

win_admin_user: "azureuser"
win_admin_password: "ChangeMeOnStartup12345"
win_vm_name: "WIN-ansible"
win_vm_sku: "2022-Datacenter"
win_vm_size: "Standard_DS1_v2"
win_public_ip_name: "win_demo_ip"
win_nic_name: "win_demo_nic"
```

### Running Tests

Each of the playbooks in this project can now be run using `ansible-navigator` or `ansible-runner`; just be sure to change the name of the YAML file to the name of the file that you want to run and add any required environment variables for the playbook that you need to run.

#### Create a RHEL 8 Linux VM

The following command should be run from the root directory of this project as the example expects certain file paths following ansible runner directory conventions.  The playbook will create a RHEL 8 VM and all of the dependent resources to enable the VM that do not already exist.

```bash
ansible-navigator run project/create_rhel_vm_demo.yml \
-i inventory/hosts \
--pae false \
--extra-vars "@env/extravars" \
--mode stdout \
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

If you get authentication errors when the automation runs, then you may need to ensure that your Service Principal is configured correctly.

#### Create a Windows VM

The following command should be run from the root directory of this project as the example expects certain file paths following ansible runner directory conventions.  The playbook will create a Windows VM and all of the dependent resources to enable the VM that do not already exist.  If you intend to keep this server, then be sure to change the password once your VM is created.

```bash
ansible-navigator run project/create_windows_vm_demo.yml \
-i inventory/hosts \
--pae false \
--extra-vars "@env/extravars" \
--mode stdout \
--eei quay.io/scottharwell/azure-execution-env:latest \
--eev $HOME/.azure:/home/runner/.azure
```

#### Destroying Resources

Once resources are deployed, then you may incur charges in your Azure tenancy.  You may run the `destroy_resource_group.yml` playbook to remove all resources deployed with this demo to ensure that you're only charged for resources while testing.

**Note:** _Only run this playbook when you are ready to remove the resources from your Azure subscription if you do not want to have to recreate them in order to test again._

```bash
ansible-navigator run project/destroy_resource_group.yml \
-i inventory/hosts \
--pae false \
--extra-vars "@env/extravars" \
--mode stdout \
--eei quay.io/scottharwell/azure-execution-env:latest \
--eev $HOME/.azure:/home/runner/.azure
```

## Private Networking

**Note:** _This section uses a different set of extra vars and implementation requirements than the examples from the previous section._

Cloud networking offers organizations many options for connecting multiple cloud networks, on-premises networks, and multi-cloud networks together.  Microsoft provides [informational documentation](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) and [troubleshooting documentation](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-troubleshoot-peering-issues#configure-virtual-network-peering-between-two-virtual-networks) about VNET peering and the concepts of different types of networking.

Many customers using Ansible on Azure will want to use a hub-and-spoke networking model that allows transit routing to-and-from AAP for access to the applications and for automation to traverse cloud and on-premises networks without the need for a bastion or jump host over the public internet.

![Hub-and-spoke networking diagram](https://docs.microsoft.com/en-us/azure/virtual-network/media/virtual-networks-peering-overview/local-or-remote-gateway-in-peered-virual-network.png)

In the image above, Ansible Automation Platform would sit on one of the spoke VNETs.

The `create_peer_network_demo.yml` playbook creates a demo deployment of this type of network configuration.  It is intended to be an example for routing traffic using the hub-and-spoke model implemented in the same fashion as the diagram above.  A production deployment would likely be much more complex and unique per the requirements of your organization; the variables in this playbook are for example only.

### Basic Operations

This playbook performs the following actions during deployment:

1. Creates a hub virtual network
2. Creates two spoke virtual networks
3. Creates a VPN Virtual Gateway for transit routing in the hub network
4. Creates a route table that routes the spoke subnets through the virtual gateway
5. Assigns the route table as the default route table for the spoke networks
6. Creates a RHEL virtual machine in each of the three VNETs
   * A VM in the hub VNET with a public IP address
   * A VM in the spoke 1 VNET with a private IP address
   * A VM in the spoke 2 VNET with a private IP address

Once the playbook completes, you should be able to SSH into the public VM and route to each of the VMs on spoke VNETs.  Traffic should also route between the two spoke VMs via the virtual gateway with the configured transit routing.

**Note:** _It can take over 30 mins for the VPN virtual gateway to deploy. The entire playbook typically takes about 45 mins to create all resources._

The following are required extra-vars needed to run the playbook.  Similar to the previous examples, you should create an extra-vars file in the `env` folder to store them.  You may create a file with a separate name, like `env/extravars-peering` to maintain both sets of variables.

```yaml
---
# Required
debug: false
resource_group_name: rg-peering-demo
region: eastus
hub_vnet_name: hub-vnet
hub_vnet_cidr: "172.16.0.0/23"
hub_subnet_name: hub-subnet
hub_subnet_cidr: "172.16.0.0/24"
spoke1_vnet_name: spoke-1-vnet
spoke1_vnet_cidr: "172.16.2.0/24"
spoke1_subnet_name: spoke-1-subnet
spoke1_subnet_cidr: "172.16.2.0/24"
spoke2_vnet_name: spoke-2-vnet
spoke2_vnet_cidr: "172.16.3.0/24"
spoke2_subnet_name: spoke-2-subnet
spoke2_subnet_cidr: "172.16.3.0/24"
virtual_gw_name: hub-gateway
virtual_gw_sku: Basic
virtual_gw_subnet_cidr: "172.16.1.0/26"
virtual_gw_vpn_type: route_based
route_table_name: "hub-and-spoke-route-table"
ssh_security_group_name: ssh-security-group
rhel_vm_sku: "8_5"
vm_size: "Standard_A1_v2"
vm_username: azureuser
ssh_pub_key: ""  # Add your RSA SSH public key here
# Optional
# node_pool_rg: ""  # Name of the node pool resource group that contains the route table for the AKS cluster
# managed_app_rg: ""  # Name of the managed app resource group
# managed_app_vnet_name: ""  # Name of the managed app node pool vnet
# managed_app_cidr: "192.168.0.0/26"  # CIDR of the managed app node pool vnet
# managed_app_route_table: ""  # The route table name for the managed app node pool vnet
# vpn_cidr: ""  # the CIDR range of your local VPN that could also be connected as a spoke to the newly created hub vnet
```

When ready, you may run the following command to begin the deployment.

```bash
ansible-navigator run project/peer_network_demo.yml \
-i inventory/hosts \
--pae false \
--mode stdout \
--ee true \
--eei quay.io/scottharwell/azure-execution-env:latest \
--extra-vars "@env/peer_network_extravars" \
--eev $HOME/.azure:/home/runner/.azure
```

### Advanced Operations

#### VPN Setup

This networking configuration sets up most of the Azure requirements to add an external network (on-premises network, other cloud, etc.) via VPN.  There is a extra variable `vpn_cidr` that can be issued during playbook run that will add the VPN CIDR to the route tables.  But, you will need to perform the VPN configuration directly in the Azure VPN Gateway to your on-prem network to establish connectivity. VPN configuration is not within the scope of this example.

#### Ansible Automation Platform on Azure

If you have Ansible Automation Platform on Azure installed as a managed application, then configuring the optional `managed_app_*` values above with will configure routing options between the previously created resources and your AAP deployment.  This will update the routing peering and routing table of the managed application to participate in the hub-and-spoke networking, and can be used to automate against hosts on the spoke networks configured in the route table, including on-premises networks or other clouds.

### Removing the Peer Network Demo

The peered network demo puts all resources into a single resource group for easy removal.  However, the AKS route table will have hanging configuration if just the resource group is deleted.  The `destroy_peer_network_demo.yml` will remove the hanging resources in the node pool resource group and any resources in the resource group created for the demo.
