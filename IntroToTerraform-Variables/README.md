# Getting Started with Terraform on Azure: Variables

### Introduction

Reusability is one of the major benefits of Infrastructure as Code. When we codify our networking, storage, and compute in a way that allows for us to reuse the same code to deploy our resources, we can move very fast. Requests for infrastructure that normally take a week or two to complete now take a couple hours because we already have a standard template mapped out for that infrastructure. We also gain the ability to allow for teammates of any skill set to deploy complex infrastructure that's already been defined in code. For example, we no longer need a SQL expert to install and configure SQL services each time it is needed. Now a standard build can be defined in code and re-deployed by anyone. 

This is why it is important to keep in mind the reusability of our code when designing Terraform configurations. In Terraform, we can use variables to allow our configurations to become more dynamic. This means we are no longer hard coding every value straight into the configuration. In this guide we will use the different types of *input variables* to parameterize our configuration that we created in the [first article of this series](https://cloudskills.io/blog/terraform-azure-01). 
 

## Prerequisites

Before you begin, you'll need to set up the following:
 - [Azure subscription](https://azure.microsoft.com/en-us/).
 - Azure Cloud Shell. Be sure to check out the prerequisites on ["Getting Started with Terraform on Azure: Deploying Resources"](https://cloudskills.io/blog/terraform-azure-01) for a guide on how to set this up.


## Step 1 — Input Variables

Input variables serve the same purpose as a parameter would for a script. They allow us to parameterize the Terraform configuration so that we can input the values that are required upon deployment to customize our build. 

When creating Terraform configurations, it's best practice to separate out parts of our configuration into individual `.tf` files. This gives us better organization and readability. Remember, the code we are creating is also a form of living documentation, so make it pretty for the next person that needs to read it. For defining input variables, it's typical to create a separate `variables.tf` file and store the variable configurations in there:


```
development
└── server
    └── main.tf
    └── variables.tf
```
With the folder structure above, if we ran Terraform in the server directory, it would scan through any `.tf` files in that directory and see them as one whole configuration. We could put everything all in one `main.tf` file but as our configuration gets bigger this would cause the configuration to be harder to understand and debug. 

A variable is defined in Terraform by using a variable block with a label. The label must be a unique name, you cannot have variables with the same name in a configuration. It is also good practice to include a description and type. The variable type specifies the [type constraint](https://www.terraform.io/docs/configuration/types.html) that the defined variable will accept. This allows us to validate that we aren't using a string value for a variable that needs to be a number or bool type. Below are two variable blocks for `system` and `location` both of the type string. The `location` variable has an additional `default` argument with the `westus2` expression. This means that if a value for the location variable is not specified when we run our `terraform apply`, it will default to `westus2`:

```
variable "system" {
    type = string
    description = "Name of the system or environment"
}

variable "location" {
    type = string
    description = "Azure location of terraform server environment"
    default = "westus2"

}
```

Now that we've declared our variables in the `variables.tf` file. We can use the variables in our `main.tf` file by calling that variable in the `var.<name>` format. So the system variable would be represented as `var.system` and the value of our system variable that is is assigned during a `terraform apply` or `terraform plan` can then be referenced in our `azurerm_resource_group` resource block:        

```
provider "azurerm" {
  version = "1.38.0"
}

#create resource group
resource "azurerm_resource_group" "rg" {
    name     = "rg-${var.system}"
    location = var.location
    tags      = {
      Environment = var.system
    }
}
```
Also note that we can also include our variable name as part of a string by wrapping it in curly brackets like `${var.system}`. In the example above, this would cause our name argument to be the value of `var.system` but prefixed with `rg-`. This is extremely useful when dynamically naming resources like a storage account or subnet that should have a naming scheme that includes the system name.

So now we've declared some variables in our `variables.tf` file and we are referencing them in the `main.tf` file. How do we actually assign values to these variables we've created? In Terraform, there are several methods for assigning variable values and they all have their use case. The most simple way to assign a value to a variable is using the `-var` option in the command line when running our `terraform apply` or `terraform plan`:

```
terraform apply -var="system=terraformdemo" -var="location=eastus"
```
If we have many variable values to input we can define them in a variable definition file named `terraform.tfvars` in the following format:
```
system ="terraformdemo"
location = "eastus"

```
Terraform will automatically load the variable values from the variable definition file if it is named `terraform.tfvars` or ends in `.auto.tvfars` and placed in the same directory as the other configuration files like below:
```
development
└── server
    └── main.tf
    └── variables.tf
    └── terraform.tfvars
```
We can also place variable values in environment variables of the OS that is going to be executing the Terraform config. Terraform will scan the system for any variables that start with `TF_VAR_` and use those as variable values for Terraform. This is useful if you are going to be running multiple Terraform configurations on a system. If we were to set the value for the `system` variable, we would create an environment variable that looks like the following:
```
 export TF_VAR_system=terraformdemo
 
```
For our PowerShell friends it would look more like this:

```
$Env:TF_VAR_system="terraformdemo"
```

Now that we know how to declare variables in our configurations and assign values, let's take a look at more complex variable types.

## Step 2 — Maps, Lists, and Objects


A map is a collection of values, each value is labeled by a string. The values can be various element types like bool, int, string. However, they must all be of the same type. In the example below is a map variable named `managed_disk_type`. In the `default` block are two string values, `"Premium_LRS"` and `"Standard_LRS"` with a named label to identify each one `westus2` and `eastus`. This allows us to choose a value based on the label name. So, for our example, we have a string key value for `westus2` and `eastus`. We can use this grouping to define the type of storage we want to use based on the region in Azure. The example use case would be if we wanted servers in `westus2` to host our production workloads and use Premium_LRS disk for fast performance while servers hosted in `eastus` would be used for disaster recovery workloads and use Standard_LRS disk to save on costs:
```
variable "managed_disk_type" { 
    type = map
    description = "Disk type Premium in Primary location Standard in DR location"

    default = {
        westus2 = "Premium_LRS"
        eastus = "Standard_LRS"
    }
}
```
Now that we have our map variable defined, we still need to create the logic for which disk to use in our `main.tf` file. In the `storage_os_disk` block of the `azurerm_virtual_machine` resource we would use the `lookup` function, which follows the `lookup(map, key, default)` format. First, we declare the map variable that we want to lookup, which would be `var.managed_disk_type` in this example. Next, we will retrieve our map value based on the key we specify which is `var.location`. This variable should resolve to `westus2` or whatever location value we input for that variable. If it matches with its key value pair in our map it will retrieve that value. So our example should retrieve `Premium_LRS` if our `var.location` variable is configured with the `westus2` value. If the key value pair does not exist because the `var.location` variable doesn't contain either `westus2` or `eastus` it will default to `Standard_LRS`:
```
 storage_os_disk {
    name              = "stvm${var.servername}os"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = lookup(var.managed_disk_type, var.location, "Standard_LRS")
  }
```

Now that we've created a map variable, we'll create another variable type, list variables. List variable types are a sequence of values that can be any element type as long as they are all of the same type. Variable lists are useful for assigning values that require a list type. In the example below we are using a list variable for `vnet_address_space` this attribute can take list values, so declaring a list for the variable allows us to specify a list of addresses to configure:

```
variable "vnet_address_space" { 
    type = list
    description = "Address space for Virtual Network"
    default = ["10.0.0.0/16"]
}

```
We specified a default argument with `[10.0.0.0/16]` as the default address space. Notice that it is wrapped in square brackets. This represents a value that is a list. To pass this variable on to our `main.tf` we would simply use the `var.vnet_address_space` expression just like with a regular string variable. 

Another variable type is the object type. Object types are a structural type that allow for each attribute to have it's own unique type, unlike a map or list. In this example we are declaring an OS variable with the object type for the OS image that we want our VM to use. Object type variables are great for transferring a common set of variable values between modules. We could potentially have another module that outputs this required image information and easily pass it into the OS variable reducing the number of variable blocks needed:

```
variable "os" {
    description = "OS image to deploy"
    type = object({
        publisher = string
        offer = string
        sku = string
        version = string
  })
}    
```
When we use the object type variable in our `main.tf` configuration. We will need to specify the `os` variable and each attribute inside of it in order to reference each value:
```
storage_image_reference {
    publisher = var.os.publisher
    offer     = var.os.offer
    sku       = var.os.sku
    version   = var.os.version
  }
```
Lists, maps, and objects are the three most common complex variable types. They are all great for their specific use cases. In the next section, we will look at how to create an output for our configuration.

## Step 3 — Output Values

Outputs allow for us to define values in the configuration that we want to share with other resources or modules. For example, we could pass on the output information for the public IP address of a server to another process like building a firewall rule. Even if we are using a dynamic public IP, after the configuration is successfully deployed, the value of the public IP address is retrieved and defined as an output. An output block is defined with the following syntax:
```
output "pip" {
    description = "Public IP Address of Virtual Machine"
    value = azurerm_public_ip.publicip.ip_address
}
```
Note, the output parameter string must be unique in the configuration. We could then reference this output from the state file and use it in another configuration which we will go into more detail in the next post in this series.


## Step 4 — Deploying the Complete Configuration

Now that we know how variables and outputs works, lets refactor our configuration for deploying a Virtual Machine and convert it into a dynamic configuration with variables. In Azure Cloud Shell, we will create the four files for our variables and outputs. The directory and file structure will look like this:
```
terraformdemo
    └── main.tf
    └── variables.tf
    └── terraform.tfvars
    └── output.tf

```
Below are the complete configurations for each file including the changes to make our configuration more dynamic. Copy each configuration and paste it into the Azure Cloud Shell editor using the `code <filename>` command to create each file:

###**main.tf**
```
provider "azurerm" {
  version = "1.38.0"
}

#create resource group
resource "azurerm_resource_group" "rg" {
    name     = "rg-${var.system}"
    location = var.location
    tags      = {
      Environment = var.system
    }
}

#Create virtual network
resource "azurerm_virtual_network" "vnet" {
    name                = "vnet-dev-${var.location}-001"
    address_space       = var.vnet_address_space
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
}

# Create subnet
resource "azurerm_subnet" "subnet" {
  name                 = "snet-dev-${var.location}-001 "
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefix       = "10.0.0.0/24"
}

# Create public IP
resource "azurerm_public_ip" "publicip" {
  name                = "pip-${var.servername}-dev-${var.location}-001"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
}


# Create network security group and rule
resource "azurerm_network_security_group" "nsg" {
  name                = "nsg-sshallow-${var.system}-001 "
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

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
}

# Create network interface
resource "azurerm_network_interface" "nic" {
  name                      = "nic-01-${var.servername}-dev-001 "
  location                  = azurerm_resource_group.rg.location
  resource_group_name       = azurerm_resource_group.rg.name
  network_security_group_id = azurerm_network_security_group.nsg.id

  ip_configuration {
    name                          = "niccfg-${var.servername}"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "dynamic"
    public_ip_address_id          = azurerm_public_ip.publicip.id
  }
}

# Create virtual machine
resource "azurerm_virtual_machine" "vm" {
  name                  = var.servername
  location              = azurerm_resource_group.rg.location
  resource_group_name   = azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.nic.id]
  vm_size               = "Standard_B1s"

  storage_os_disk {
    name              = "stvm${var.servername}os"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = lookup(var.managed_disk_type, var.location, "Standard_LRS")
  }

  storage_image_reference {
    publisher = var.os.publisher
    offer     = var.os.offer
    sku       = var.os.sku
    version   = var.os.version
  }

  os_profile {
    computer_name  = var.servername
    admin_username = var.admin_username
    admin_password = var.admin_password
  }

  os_profile_linux_config {
    disable_password_authentication = false
  }
}
```

###**variables.tf**
```
variable "system" {
    type = string
    description = "Name of the system or environment"
}

variable "servername" {
    type = string
    description = "Server name of the virtual machine"
}

variable "location" {
    type = string
    description = "Azure location of terraform server environment"
    default = "westus2"

}

variable "admin_username" {
    type = string
    description = "Administrator username for server"
}

variable "admin_password" {
    type = string
    description = "Administrator password for server"
}

variable "vnet_address_space" { 
    type = list
    description = "Address space for Virtual Network"
    default = ["10.0.0.0/16"]
}

variable "managed_disk_type" { 
    type = map
    description = "Disk type Premium in Primary location Standard in DR location"

    default = {
        westus2 = "Premium_LRS"
        eastus = "Standard_LRS"
    }
}

variable "vm_size" {
    type = string
    description = "Size of VM"
    default = "Standard_B1s"
}

variable "os" {
    description = "OS image to deploy"
    type = object({
        publisher = string
        offer = string
        sku = string
        version = string
  })
}      
```

###**terraform.tfvars**
```
system = "terraexample"
servername = "vmterraform"
location = "westus2"
admin_username = "terraadmin"
vnet_address_space = ["10.0.0.0/16","10.1.0.0/16"]
os = {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "16.04.0-LTS"
    version   = "latest"
}
```

###**output.tf**
```
output "pip" {
    description = "Public IP Address of Virtual Machine"
    value = azurerm_public_ip.publicip.ip_address
}
```

Now that we have our directory created with the four configuration files, we will run our `terraform init` to initialize the directory and download our provider plugins. We will follow it up with a `terraform apply`. Notice that we are now prompted for the Administrator password variable. This because we followed best practices and left the local admin password value out of the `terraform.tfvars` file and never set a default value in the `variables.tf` file. Terraform will automatically prompt for variables that aren't defined and do not have a default value configured:

![adminpassword](/Images/passvariable.png)


After typing in a password, confirm the apply to start deploying the infrastructure:
```
azurerm_resource_group.rg: Creating...
azurerm_resource_group.rg: Creation complete after 2s [id=/subscriptions/f7c32571-81bd-4e23-5321-7e216164bae3/resourceGroups/rg-terraexample]
azurerm_network_security_group.nsg: Creating...
azurerm_virtual_network.vnet: Creating...
azurerm_public_ip.publicip: Creating...
azurerm_public_ip.publicip: Creation complete after 3s [id=/subscriptions/f7c32571-81bd-4e23-5321-7e216164bae3/resourceGroups/rg-terraexample/providers/Microsoft.Network/publicIPAddresses/pip-vmterraform-dev-westus2-001]
...
```
When our VM is successfully built, we will see the public IP address assigned to our VM is generated in the output:
```
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

pip = 52.183.4.46
```
We now have a reusable Terraform configuration for deploying virtual machines into Azure! We can now modify the `terraform.tfvars` with different values to customize our deployments.

## Conclusion

In this article we learned about variables and how we can use them to make our Terraform configurations reusable. We also went over complex variable types like lists, maps, and objects that can be used to populate data dynamically into our Terraform configurations. Variables are a core part of Terraform and should be heavily utilized in most Terraform configurations. Hard coding values into Terraform configurations should be discouraged if they are preventing the code from becoming a reusable tool.

In the next article we will dig into remote state which must be thoroughly understood before using Terraform in production. 
