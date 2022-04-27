# Getting Started with Terraform on Azure: Importing Existing Infrastructure

When first introduced to Terraform, we can see how easy it is to build new environments and manage them with software development practices. We start to experience the numerous benefits that come with infrastructure as code such as deployment speed, stability through templatized environments, and transparency through code documentation. However, all these benefits emerge from the new infrastructure we are creating with Terraform. What about our old pre-existing infrastructure? How can we manage the environments we've already built by hand with code?

One of the main principles with infrastructure as code is to "define everything in code". If this principle only applies to new environments, we are greatly diminishing the benefits gained by limiting this process to only a small scope of the environment. This is why it's essential to retroactively return to pre-existing environments and convert them over to code.

Terraform can import pre-existing resources into a state file, which then allows Terraform to manage those resources with a configuration file. However, this process is still in its infancy stage and is actively being improved upon by Hashicorp. As of right now, Terraform cannot automatically generate code based on existing infrastructure. The import process included creating configuration files by hand, then importing the existing resources via the Terraform command line. Another caveat currently is that only a single resource can be imported into a state file at a time. 

In this guide, we walk through the process of importing pre-existing infrastructure into Terraform. First, we deploy some infrastructure with Azure CLI and then import it into a state file to be managed by Terraform.

## Prerequisites

Before you begin, you'll need to set up the following:
 - [Azure subscription](https://azure.microsoft.com/en-us/).
 - Azure Cloud Shell. Be sure to check out the prerequisites on ["Getting Started with Terraform on Azure: Deploying Resources"](https://cloudskills.io/blog/terraform-azure-01) for a guide on setting up Azure Cloud Shell.

In this guide, we will be importing some pre-existing infrastructure into Terraform. Before we can walk through the import process, we will need some existing infrastructure in our Azure account. Below is a list of commands to run in Azure CloudShell using Azure CLI in the Bash environment. The Azure CLI commands deploy a resource group, network security group, virtual network, and subnets. You can copy the entire configuration below and paste it directly into Azure CloudShell to deploy everything all at once:
```
az group create --name rg-terraform --location eastus

az network vnet create \
  --name vnet-terraform-eastus-1 \
  --resource-group rg-terraform \
  --address-prefix 10.0.0.0/16 \
  --subnet-name subnet-terraform-10.0.0.0_24 \
  --subnet-prefix 10.0.0.0/24

az network nsg create -g rg-terraform -n nsg-terraform

az network nsg rule create -g rg-terraform --nsg-name nsg-terraform -n Allow-HTTPS --priority 100 \
    --source-address-prefixes '*' --source-port-ranges 443 \
    --destination-address-prefixes '*' --destination-port-ranges 443 --access Allow \
    --protocol Tcp --description "Allow HTTPS"

az network nsg rule create -g rg-terraform --nsg-name nsg-terraform -n Allow-HTTP --priority 110 \
    --source-address-prefixes '*' --source-port-ranges 80 \
    --destination-address-prefixes '*' --destination-port-ranges 80 --access Allow \
    --protocol Tcp --description "Allow HTTP"

az network vnet subnet create -g rg-terraform --vnet-name vnet-terraform-eastus-1 \
    -n subnet-terraform-10.0.1.0_24 \
    --address-prefixes 10.0.1.0/24  \
    --network-security-group nsg-terraform

```
We should now have a resource group with a network security group, virtual network, and two subnets. In the next steps we will walk through how to import this infrastructure into Terraform. 

## Step 1 — Simple Import

We will start by importing a resource group into Terraform. Remember, we can only import one resource at a time. To import a resource, we need to have a Terraform configuration file already built for that resource. The configuration file allows us to link the resource identifier used by Terraform to the resource identifier used in Azure. To import our resource group, we will create the following configuration in a `main.tf` file within Azure CloudShell:
```
provider "azurerm" {
    version="1.38.0"
}

# create resource group
resource "azurerm_resource_group" "rg"{
    name = "rg-terraform"
    location = "eastus"
}
```
The syntax to perform an import with Terraform uses the following format for Azure resources using the `terraform import` command:
```
terraform import <Terraform Resource Name>.<Resource Label> <Azure Resource ID>
```
We already have the resource block name of our resource group, which is `azurerm_resource_group`, according to the Azure Terraform provider. We also need to reference the given local name that we are calling our resource group block, which in our example is `rg`. Now we need the resource ID of the resource group in Azure to tell Terraform we want to import this item from Azure. To retrieve the resource ID, we can look up the properties of the `rg-terraform` resource group in the Azure portal, or we can use the following command in the Azure CloudShell to display the ID:
```
az group show --name rg-terraform
```

The output looks like the following, copy the ID of the resource group:
```
{
  "id": "/subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform",
  "location": "eastus",
  "managedBy": null,
  "name": "rg-terraform",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```
Now we have all the information we need to import our resource group into a Terraform state file. In the same directory as our `main.tf` file,  we need to run `terraform init` to download the plugin for the Azure provider before we can perform the import:
```
terraform init
```
After `terraform init` has completed, we are good to run `terraform import` with our Terraform and Azure identifiers. The import command inspects the `main.tf` file and the Azure environment to ensure those IDs are relevant. Then imports information about the resource into a state file:
```
terraform import azurerm_resource_group.rg /subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform
```
We can see the output indicating the import was successful:
```
azurerm_resource_group.rg: Importing from ID "/subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform"...
azurerm_resource_group.rg: Import prepared!
  Prepared azurerm_resource_group for import
azurerm_resource_group.rg: Refreshing state... [id=/subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform]

Import successful!
```
Now, let's confirm that our resource group is indeed in the state file by running `cat terraform.tfstate` to display the contents. We can see that the resource group is in the state file with the resource ID that we specified:
```
{
  "version": 4,
  "terraform_version": "0.12.23",
  "serial": 1,
  "lineage": "1377f481-dc7d-7da9-45aa-a40f007ebb3d",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "azurerm_resource_group",
      "name": "rg",
      "provider": "provider.azurerm",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "/subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform",
            "location": "eastus",
            "name": "rg-terraform",
            "tags": {}
          },
          "private": "eyJzY2hlbWFfdmVyc2lvbiI6IjAifQ=="
        }
      ]
    }
  ]
}
```

After using `terraform import`, it is a good idea to run `terraform plan` to validate that the configuration in the `main.tf` file matches the resource that imported. When we run `terraform plan` we want to see output indicating that there are no changes in the plan:
```
No changes. Infrastructure is up-to-date.

This means that Terraform did not detect any differences between your
configuration and real physical resources that exist. As a result, no
actions need to be performed.
```
Once the plan has been successfully validated and reports no changes between our `main.tf` and the current state, we can now deem this configuration as good and store it in our source control repo, as it now contains the configuration for live infrastructure. 

Had we configured our `main.tf` to specify a resource group in the westus2 location, even though the actual resource is in eastus, we would still be allowed to import the resource, and the state file would contain the correct eastus location of our resource group in Azure. However, if we ran `terraform plan`, the plan would indicate that a rebuild of the resource group would need to occur to match the resource configuration in the `main.tf` file:
```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # azurerm_resource_group.rg must be replaced
-/+ resource "azurerm_resource_group" "rg" {
      ~ id       = "/subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform" -> (known after apply)
      ~ location = "eastus" -> "westus2" # forces replacement
        name     = "rg-terraform"
      ~ tags     = {} -> (known after apply)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

This is why it's crucial to run a `terraform plan` after the `terraform import` to validate that the configuration and infrastructure are up to date. If the `main.tf` displays changes when running the `terraform plan`, there is a risk with using that configuration file to apply changes in the future.

## Step 2 — Complex Imports

The example of importing a resource group is defined as a **simple import**. However, resources that contain several resources within them are deemed as **complex imports**. An example of this would be a virtual network that contains subnets or a network security group that contains security rules. Both of these resources contain multiple child resources. It is important to be aware of child resources when importing these components. We must capture all the child resources for each resource in the `main.tf` terraform configuration file, or they will be removed when running `terraform apply`. 

Below is the Terraform configuration for importing our network security group and virtual network. Notice the child resources they both contain. Copy the configuration below and save over the previous `main.tf` we used to import the resource group in step 1:

```
provider "azurerm" {
    version="1.38.0"
}

# create resource group
resource "azurerm_resource_group" "rg"{
    name = "rg-terraform"
    location = "eastus"
}


resource "azurerm_network_security_group" "nsg" {
  name                = "nsg-terraform"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name


  security_rule {
    name                       = "Allow-HTTPS"
    description                = "Allow HTTPS"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "443"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow-HTTP"
    description                = "Allow HTTP"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "80"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}



resource "azurerm_virtual_network" "vnet" {
  name                = "vnet-terraform-eastus-1"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.0.0.0/16"]


  subnet {
    name           = "subnet-terraform-10.0.0.0_24"
    address_prefix = "10.0.0.0/24"
  }

  subnet {
    name           = "subnet-terraform-10.0.1.0_24"
    address_prefix = "10.0.1.0/24"
    security_group = azurerm_network_security_group.nsg.id
  }
}
```

We need the resource IDs of our network security group and virtual network. We could retrieve this information from the Azure portal, or we can type in the following two commands to get them from Azure CloudShell:
```
az network nsg show -g rg-terraform -n nsg-terraform

az network vnet show -g rg-terraform -n vnet-terraform-eastus-1

```
Next, we use `terraform import` for each resource specifying their Terraform resource block identifier and Azure resource ID:

```
terraform import azurerm_network_security_group.nsg /subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform/providers/Microsoft.Network/networkSecurityGroups/nsg-terraform

terraform import azurerm_virtual_network.vnet /subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform/providers/Microsoft.Network/virtualNetworks/vnet-terraform-eastus-1

```
Once `terraform import` is successful for our network security group and virtual network, we can run `cat terraform.tfstate` to confirm they are now in the state file. The last test is to run `terraform plan` to validate that our `main.tf` holds the correct configuration settings for our resources:
```
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

azurerm_resource_group.rg: Refreshing state... [id=/subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform]
azurerm_network_security_group.nsg: Refreshing state... [id=/subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform/providers/Microsoft.Network/networkSecurityGroups/nsg-terraform]
azurerm_virtual_network.vnet: Refreshing state... [id=/subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform/providers/Microsoft.Network/virtualNetworks/vnet-terraform-eastus-1]

------------------------------------------------------------------------

No changes. Infrastructure is up-to-date.
```
The plan output shows no changes, which means our `main.tf` is solid and can now be used to manage this infrastructure.

## Step 3 — Import Modules

Now that we know how to import existing resources into Terraform, how do we go about importing a module? 

Let's set up a module folder to create a module for the configuration we made in step 2 and test importing it into a state file. In the current directory where we performed the tasks in step 2, we will create a subfolder called **module** using the following directory structure:
```
testingimportfolder
    └── main.tf
    └── terraform.tfstate
    └── terraform.tfstate.backup
    └───module
          └── main.tf
```
The main.tf consists of a resource block for the Azure provider and a module resource block with the source argument pointing to the parent directory. The source argument is telling our module to use the `main.tf` in the directory above it. This is not the ideal folder structure for a normal in production module, but for the sake of demonstrating importing a module with very little pre-setup, the module subfolder works:
```
provider "azurerm" {
    version="1.38.0"
}

module "importlab" {
    source = "../"
}
```
Importing a module into a state file is similar to importing resources. However, we need to import each resource that the module configures. For our example, since we are just re-using the `main.tf` file that we created in step 2, we need to import the same three resources. But, we need to change the resource identifier on the Terraform configuration side to declare that we are using a module to manage these resources. We can do this by appending our module name to the beginning of each resource identifier, which ends up looking like `module.importlab.<resource>`. While in the module folder directory, run `terraform init` to initialize the directory and pull down the Azure provider. Then run `terraform import` with the following syntax to import the three resources managed by the `importlab` module: 

```
terraform import module.importlab.azurerm_resource_group.rg /subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform

terraform import module.importlab.azurerm_network_security_group.nsg /subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform/providers/Microsoft.Network/networkSecurityGroups/nsg-terraform

terraform import module.importlab.azurerm_virtual_network.vnet /subscriptions/c8f32571-82db-4a89-923b-7e149764bae3/resourceGroups/rg-terraform/providers/Microsoft.Network/virtualNetworks/vnet-terraform-eastus-1

```
After importing the three module resources, we can run `cat terraform.tfstate` to see the contents of the state file. We see our module resource is present along with the resources that it manages:
```
{
  "version": 4,
  "terraform_version": "0.12.23",
  "serial": 3,
  "lineage": "919aec3c-da13-c83a-a846-c7d1838e13ef",
  "outputs": {},
  "resources": [
    {
      "module": "module.importlab",
      "mode": "managed",
      "type": "azurerm_network_security_group",
      "name": "nsg",
      "provider": "module.importlab.provider.azurerm",
      "instances": [
        {
...
```

Now we can validate our configuration by running `terraform plan`. The plan output should state no changes in infrastructure, indicating that we now have our module configuration imported into Terraform state.  


## Step 4 — Remote State

We can use `terraform import` with either a local or remote state. Initially, we could have configured a remote backend at the beginning of this guide and imported all of our resources into a remote state file. However, some might like to manipulate a state file locally and then copy it up to their remote state location after they have a valid configuration. For this purpose, we will demonstrate migrating our newly imported local state over to an Azure storage account backend. To create an Azure storage account with a storage container, run the following commands in Azure CloudShell:

> **Note:** Make sure to use an externally unique name for the storage account, or Azure will error out when deploying one.

```
az group create --location eastus --name rg-terraformstate

az storage account create --name terrastatestorage2134 --resource-group rg-terraformstate --location eastus --sku Standard_LRS

az storage container create --name terraformstate --account-name terrastatestorage2134

```
To copy our state file over to the storage account, we will create an additional file called `backend.tf` in the modules folder:
```
module
   └── main.tf
   └── backend.tf
```
The `backend.tf` file contains the following code to direct our Terraform configuration to save its state to our storage container. Copy the code below and save it to `backend.tf` inside the module folder:

```
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraformstate"
    storage_account_name = "terrastatestorage2134"
    container_name       = "terraformstate"
    key                  = "testimport.terraform.tfstate"
  }
}
```
Next, we run `terraform init` in the modules folder and select `yes` to copy our current state file over to the Azure storage account:
```
Initializing modules...

Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend to the
  newly configured "azurerm" backend. No existing state was found in the newly
  configured "azurerm" backend. Do you want to copy this state to the new "azurerm"
  backend? Enter "yes" to copy and "no" to start with an empty state.

  Enter a value: yes


Successfully configured the backend "azurerm"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...

Terraform has been successfully initialized!

```

Our state is now safely stored in the Azure storage account, where the state files for our other infrastructure should be (don't use local state in production). If we wanted to double check, we can use the `terraform state list` command to display the resources in our remote state:
```
module.importlab.azurerm_network_security_group.nsg
module.importlab.azurerm_resource_group.rg
module.importlab.azurerm_virtual_network.vnet
```

Our pre-existing infrastructure has now been imported and saved in our remote state container to be managed by Terraform going forward.  

## Conclusion

As you can see, importing existing infrastructure into Terraform can be awkward and tedious. There is not a fully ironed out process for it yet. However, converting pre-existing infrastructure over to be managed by Terraform is worth the time. The benefits gained through "everything in code" will most likely outweigh the time spent on importing infrastructure.
 
This process can also be used as a learning experience for employees or team members just starting with Terraform. Following documented procedures for onboarding infrastructure into Terraform can get them well acquainted with how Terraform works with the state file and Azure infrastructure.

In the next article, we will go deep into the weeds of testing and walk through how to get started with testing our Terraform code. 

