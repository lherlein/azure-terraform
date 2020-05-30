# Creating VM's in Microsoft Azure using Terraform
## Step 1: Sign up for Azure
Create a free Azure subscription using this [link](https://azure.microsoft.com/en-us/free/?ref=microsoft.com&utm_source=microsoft.com&utm_medium=docs&utm_campaign=visualstudio).
## Step 2: Configure Azure Command Line Interface (CLI)
The Azure CLI allows the user to manage Azure infrastructure from the command line. Follow the steps at this [link](https://docs.bitnami.com/azure/faq/administration/install-az-cli/) to connect your Azure account to the command line. 
## Step 3: Install and Configure Terraform
### Install Terraform
First download Terraform from [here](https://www.terraform.io/downloads.html). Make sure to define a global path to the executable just downloaded. 

- [Mac OS / Linux](https://stackoverflow.com/questions/14637979/how-to-permanently-set-path-on-linux-unix)
- [Windows](https://stackoverflow.com/questions/1618280/where-can-i-set-path-to-make-exe-on-windows)

Verify that the path is correct by trying to run a Terraform command.

```bash
user@server:~$ terraform
Usage: terraform [--version] [--help] <command> [args]
```
### Create an Azure AD Service Principal
Next up: give Terraform access to azure. To do this an Azure AD service principal must be created. This will give Terraform access to your Azure account. 

If you have multiple Azure accounts, use az account list to find the subscription id and tenant id of the account you want to use.

```bash
az account list --query "[].{name:name, subscriptionId:id, tenantId:tenantId}"
```
Next, select the subscription you want to use for this section with az account set. Make sure to set the SUBSCRIPTION_ID environment variable to hold the value of the subscription id you want to use. 

```bash
az account set --subscription="${SUBSCRIPTION_ID}"
```
Now you can create a service principal using the az ad sp create-for-rbac command. This command with return the appID, password, sp_name, and tenant. Make note of each, especially appID and password. 

```bash
az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/${SUBSCRIPTION_ID}"
```
### Configure Environment Variables
In order for Terraform to use the Azure service principal, environment variables must be set so that they can be used by Azure Terraform Modules. The variables that need to be set are:

```bash
- export ARM_SUBSCRIPTION_ID = your_subscription_id
- export ARM_CLIENT_ID = your_appID
- export ARM_CLIENT_SECRET = your_password
- export ARM_TENANT_ID = your_tenant_id
- export ARM_ENVIRONMENT = public 

# ARM_ENVIRONMENT=public is required for the US, German and Chinese Government. 
```
### Run a Sample Script
Create a test.tf file in a new directory and paste the following script.

```HashiCorp Configuration Language
provider "azurerm" {
  # The "feature" block is required for AzureRM provider 2.x. 
  # If you are using version 1.x, the "features" block is not allowed.
  version = "~>2.0"
  features {}
}
resource "azurerm_resource_group" "rg" {
        name = "testResourceGroup"
        location = "westus"
}
```
Save the file. Initialize the Terraform deployment. This downloads the Azure modules needed to create a resource group.

```bash
terraform init
```
You should get something close to the following output

```console
* provider.random: version = "~> 2.2"

Terraform has been successfully initialized!
```
If you want, you can preview what the terraform script will do with terraform plan. 

```bash
terraform plan
```
When ready, apply the plan with terraform apply.

```bash
terraform apply
```
The output should look something like:

```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + azurerm_resource_group.rg
      id:       <computed>
      location: "westus"
      name:     "testResourceGroup"
      tags.%:   <computed>

azurerm_resource_group.rg: Creating...
  location: "" => "westus"
  name:     "" => "testResourceGroup"
  tags.%:   "" => "<computed>"
azurerm_resource_group.rg: Creation complete after 1s
```
## Step 4: Create a Linux VM with Infrastructure
### Create the Terraform Script
Create a file called terraform-azure.tf and place it in an empty directory. Paste the following script into terraform-azure.tf. This script generates the required infrastructure for a Linux VM and then creates the VM itself. It creates an Azure resource group called myResourceGroup that stores all of the infrastructure for the VM. It also creates a user on the VM called azureuser. Important note: In the "create virtual machine" section (the last one) make sure to change the public_key file to the location of the public ssh key you want to use. 

```HashiCorp
# Configure the Microsoft Azure Provider
provider "azurerm" {
    # The "feature" block is required for AzureRM provider 2.x. 
    # If you're using version 1.x, the "features" block is not allowed.
    version = "~>2.0"
    features {}

    subscription_id = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    client_id       = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    client_secret   = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    tenant_id       = "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}

# Create a resource group if it doesn't exist
resource "azurerm_resource_group" "myterraformgroup" {
    name     = "myResourceGroup"
    location = "eastus"

    tags = {
        environment = "Terraform Demo"
    }
}

# Create virtual network
resource "azurerm_virtual_network" "myterraformnetwork" {
    name                = "myVnet"
    address_space       = ["10.0.0.0/16"]
    location            = "eastus"
    resource_group_name = azurerm_resource_group.myterraformgroup.name

    tags = {
        environment = "Terraform Demo"
    }
}

# Create subnet
resource "azurerm_subnet" "myterraformsubnet" {
    name                 = "mySubnet"
    resource_group_name  = azurerm_resource_group.myterraformgroup.name
    virtual_network_name = azurerm_virtual_network.myterraformnetwork.name
    address_prefix       = "10.0.1.0/24"
}

# Create public IPs
resource "azurerm_public_ip" "myterraformpublicip" {
    name                         = "myPublicIP"
    location                     = "eastus"
    resource_group_name          = azurerm_resource_group.myterraformgroup.name
    allocation_method            = "Dynamic"

    tags = {
        environment = "Terraform Demo"
    }
}

# Create Network Security Group and rule
resource "azurerm_network_security_group" "myterraformnsg" {
    name                = "myNetworkSecurityGroup"
    location            = "eastus"
    resource_group_name = azurerm_resource_group.myterraformgroup.name
    
    security_rule {
        name                       = "SSH"
        priority                   = 1001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
    }

    tags = {
        environment = "Terraform Demo"
    }
}

# Create network interface
resource "azurerm_network_interface" "myterraformnic" {
    name                      = "myNIC"
    location                  = "eastus"
    resource_group_name       = azurerm_resource_group.myterraformgroup.name

    ip_configuration {
        name                          = "myNicConfiguration"
        subnet_id                     = azurerm_subnet.myterraformsubnet.id
        private_ip_address_allocation = "Dynamic"
        public_ip_address_id          = azurerm_public_ip.myterraformpublicip.id
    }

    tags = {
        environment = "Terraform Demo"
    }
}

# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
    network_interface_id      = azurerm_network_interface.myterraformnic.id
    network_security_group_id = azurerm_network_security_group.myterraformnsg.id
}

# Generate random text for a unique storage account name
resource "random_id" "randomId" {
    keepers = {
        # Generate a new ID only when a new resource group is defined
        resource_group = azurerm_resource_group.myterraformgroup.name
    }
    
    byte_length = 8
}

# Create storage account for boot diagnostics
resource "azurerm_storage_account" "mystorageaccount" {
    name                        = "diag${random_id.randomId.hex}"
    resource_group_name         = azurerm_resource_group.myterraformgroup.name
    location                    = "eastus"
    account_tier                = "Standard"
    account_replication_type    = "LRS"

    tags = {
        environment = "Terraform Demo"
    }
}

# Create virtual machine
resource "azurerm_linux_virtual_machine" "myterraformvm" {
    name                  = "myVM"
    location              = "eastus"
    resource_group_name   = azurerm_resource_group.myterraformgroup.name
    network_interface_ids = [azurerm_network_interface.myterraformnic.id]
    size                  = "Standard_DS1_v2"

    os_disk {
        name              = "myOsDisk"
        caching           = "ReadWrite"
        storage_account_type = "Premium_LRS"
    }

    source_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "16.04.0-LTS"
        version   = "latest"
    }

    computer_name  = "myvm"
    admin_username = "azureuser"
    disable_password_authentication = true
        
    admin_ssh_key {
        username       = "azureuser"
        public_key     = file("/home/azureuser/.ssh/authorized_keys")
    }

    boot_diagnostics {
        storage_account_uri = azurerm_storage_account.mystorageaccount.primary_blob_endpoint
    }

    tags = {
        environment = "Terraform Demo"
    }
}
```
### Build and Deploy the Infrastructure
Now that the correct Terraform script has been created, it must be initialized. 

```
terraform init
```
Next, you want Terraform to review and validate the script. This is done by using the plan command. This will tell you what the script does but will not create or deploy resources in Azure.

```bash
terraform plan
```
The plan command should output something like: 

```
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


...

Note: You didn't specify an "-out" parameter to save this plan, so when
"apply" is called, Terraform can't guarantee this is what will execute.
  + azurerm_resource_group.myterraform
      <snip>
  + azurerm_virtual_network.myterraformnetwork
      <snip>
  + azurerm_network_interface.myterraformnic
      <snip>
  + azurerm_network_security_group.myterraformnsg
      <snip>
  + azurerm_public_ip.myterraformpublicip
      <snip>
  + azurerm_subnet.myterraformsubnet
      <snip>
  + azurerm_virtual_machine.myterraformvm
      <snip>
Plan: 7 to add, 0 to change, 0 to destroy.
```
Make sure that everything looks correct. If it does, it's time to deploy the infrastructure and VM. 

```
terraform apply
```
Once everything is complete, you should be able to obtain the IP address of your new VM using az vm show:

```
az vm show --resource-group myResourceGroup --name myVM -d --query [publicIps] --output tsv
```
Finally, ssh to your VM!

```
ssh azureuser@<publicIP>
```
