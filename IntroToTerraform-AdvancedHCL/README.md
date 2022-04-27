# Getting Started with Terraform on Azure: Functions, Expressions, and Loops


As we start building out infrastructure with Terraform, the logic used to build interlinking modules becomes more complicated. This is why it's essential to understand the options and capabilities of HCL (Hashicorp Configuration Language). Not only will this make it easier to create the necessary logic flow in our code, but we can also optimize our configurations. Writing a Terraform configuration in a different way can potentially provide a performance boost in deployment. 

Understanding how to create loops and advanced expressions can give our code a cleaner look and provide our teammates with a more precise understanding of the infrastructure when reading the Terraform configuration. Remember, our Terraform code is not meant to just deploy our infrastructure and get the job done. The goal of HCL is to bridge the gap between human and machine-friendly interfaces by providing code that is easy to read and work well with the command line at the same time. 

In this guide, we are going to create a module and learn how to use functions, expressions, and loops in our Terraform configurations.

## Prerequisites

If you'd like to follow along with the concepts in this guide, you'll need to set up the following:
 - [Azure subscription](https://azure.microsoft.com/en-us/).
 - Azure Cloud Shell. Be sure to check out the prerequisites on ["Getting Started with Terraform on Azure: Deploying Resources"](https://cloudskills.io/blog/terraform-azure-01) for a guide on how to set this up.

## Step 1 — Functions

HCL contains built-in functions that can be called within an expression to logically create values for our arguments. With Terraform, we can use an interactive console for testing our Terraform expressions instead of having to run our Terraform configuration each time. Simply type in the following syntax where Terraform is installed. This works in Azure Cloud Shell as well:

```
terraform console
```
You will be directed to the Terraform console, which will display a `>`. You may test some of these function examples in this guide in your own Terraform console. There are many [built-in functions](https://www.terraform.io/docs/configuration/functions.html), but we will go over the most commonly used ones:

**Lower** is useful for values that need to always be lowercased like storage account names:
```
> lower("TFStoragesta")
tfstoragesta

```

**Replace** is useful for manipulating naming schemes or number formats of certain values. The format is the following:
```
replace(string, substring, replacement)
```

A simple use case would be replacing a string in a text:
```
> replace("Luke likes CloudSkills", "likes", "loves")
Luke loves CloudSkills
```
A more practical use case would be using the `replace` function with a regex query to swap out any subscription ID on a resource ID string with a specific subscription:

```
> replace("/subscriptions/f8c37571-4325-4a97-977b-7a216e64bae3/resourceGroups/rg-remotestatetfc", "/[0-9A-Fa-f]{8}-(?:[0-9A-Fa-f]{4}-){3}[0-9A-Fa-f]{12}/", "c1c37531-2355-4a57-947b-4e236e64bae3")
/subscriptions/c1c37531-2355-4a57-947b-4e236e64bae3/resourceGroups/rg-remotestatetfc
```

**Min** and **max** are used to find the highest and lowest values in a set of numbers, which can be useful for determining the largest or smallest resource:
```
> max(32,64,99)
99
```

```
> min (32,128,1024)
32
```

**Split** can be used to get two string values out of a single value. For example, a `Standard_LRS` storage type string can be split up for the `azurerm_storage_account` resource that takes the following two arguments for `account_teir` and `account_replication_type`:
```
resource "azurerm_storage_account" "example" {
  name                = "storageaccountname"
  resource_group_name = azurerm_resource_group.example.name

  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```
We could take a variable input of `Standard_LRS` and use split to get each value reducing the number of variables needed:
```
> split("_","Standard_LRS")
[
  "Standard",
  "LRS",
]
```
**Element** can then be used to access each index of the list by wrapping our split function with an element function and specifying the index value:
```
> element(split("_","Standard_LRS"), 0)
Standard
```
We could then write our resource block like the following if we created an input variable for the `Standard_LRS` string:
```
resource "azurerm_storage_account" "example" {
  name                = "storageaccountname"
  resource_group_name = azurerm_resource_group.example.name

  location                 = azurerm_resource_group.example.location
  account_tier             = element(split("_","var.storage_type"), 0)
  account_replication_type = element(split("_","var.storage_type"), 1)
}
```

**Coalesce** will take a series of arguments and return the first one that isn't null or an empty string `""`:

```
> coalesce("40", "50", "20")
40

> coalesce( "", "50", "20")
50
```

This can be useful for creating a set of optional variables and selecting the one that contains a value:
> **Note:** There are several configurations in this article that will be truncated by a `...` to reduce the amount of code in this article.
```
resource "azurerm_virtual_machine" "main" {
  name                  = "${var.prefix}-vm"
  location              = azurerm_resource_group.main.location
  resource_group_name   = azurerm_resource_group.main.name
  network_interface_ids = [azurerm_network_interface.main.id]
  vm_size               = coalesce( var.small, var.medium, var.large)
...
```

**Length** returns the numerical length of a string, list, or map:
```
> length(["32GB", "64GB"])
2
```
This is very useful to get the number of an item. For the example above, we can determine that we have two disk sizes, and we will be requiring 2 disks to be created. We could then feed this value into a loop that we will demo later in this guide.

**Setproduct** will display all possible combinations of a given set:
```
> setproduct(["development", "staging", "production"], ["EastUS", "WestUS"])
[
  [
    "development",
    "EastUS",
  ],
  [
    "development",
    "WestUS",
  ],
  [
    "staging",
    "EastUS",
  ],
  [
    "staging",
    "WestUS",
  ],
  [
    "production",
    "EastUS",
  ],
  [
    "production",
    "WestUS",
  ],
]
```

**File** will read the contents of a file and return it as a string:
```
> file("${path.module}/test.ps1")
#This is an empty PS1

```
This is very useful when using the `template_file` data source type, which allows you to insert variables into your files and then pass them into another resource for use. Below we have a `test.ps1` file that will set the DNS addresses of a server. We have two variables inserted into our PowerShell script represented by `${DNS_Server1}` and `${DNS_Server2}`: 
```
# test.ps1
Set-DnsClientServerAddress -InterfaceIndex 12 -ServerAddresses ("${DNS_Server1}","{DNS_Server2}")
```
Next, we can use the `template_file` resource to specify the `test.ps1` script and replace the `${DNS_Server1}` and `${DNS_Server2}` variables inside the script with their respective values declared in the `vars` block. We can then reference the newly transformed `test.ps1` file that includes the proper values for the DNS servers by using `data.template_file.init.rendered` in the `custom_data` attribute of a virtual machine resource block:
```
data "template_file" "init" {
  template = "${file("./test.ps1")}"
  vars = {
    DNS_Server1 = "10.0.0.1"
    DNS_Server2 = "10.0.0.2"
  }

resource "azurerm_virtual_machine" "main" {
...
  os_profile {
      computer_name  = "myserver"
      admin_username = "adminuser"
      admin_password = "badpassword123"
      custom_data = data.template_file.init.rendered
    }
...
}
```
Functions are a great way to manipulate the data in our configurations, however, sometimes there is a need to include more logic into our code. This is where expressions come into play. 

## Step 2 — Complex Expressions

Expressions are used to compute the values in our HCL configurations. Complex expressions allow us to provide additional logic to our resources deployed. Below are some examples of popular expressions we can use in our code:

##### Operators

Operators either combine the result of two values and produce a third or take one value and transform it. The typical operators used in configurations are the following:

- **Equal** `a==b` if `a` is equal to `b` then the result is true. 
- **Not Equal** `a != b` if `a` does not equal `b` then the result is true.
- **Either Or**  `a || b`  if either `a` or `b` is true then the result is true.
- **Both True**  `a && b` if both `a` and `b` are true then the result is true.
- **Not True** `!a` if a is false then the result is true.

##### Conditions

Conditions determine the value based on the result of two bool (true or false) expressions:

```
condition ? true : false
```

This is typically used to set up a default value in a Terraform configuration. For example, if we have a virtual machine module and would like to provide the option to deploy a VM to a different location than it's resource group, we could create a variable for `vm_location` and set a default for `""` like below:

```
variable "vm_location" {
    type = string
    description = "Azure location of terraform server environment"
    default = ""

}
```
In our virtual machine resource block, we could then use a conditional expression for the `location` argument. The condition expression provides the logic to use the value for `var.vm_location` if it is not an empty string. If `var.vm_location` is still an empty string because the user of the module never specified one, then the condition expression provides the logic to default to the location of our resource group:

```
resource "azurerm_virtual_machine" "main" {
  name                  = "MyVMName"
  location              = var.vm_location != "" ? var.vm_location : azurerm_resource_group.main.location
  ...
```

##### For Expressions

For expressions allow us to take a value and change it. We can use this to iterate over a list of values and modify them. For example, we can obtain a list of subnets from the `azurerm_virtual_network` data source and use a `for` expression to iterate over each subnet and change the output formatting it with a lowercase:

```
data "azurerm_virtual_network" "example" {
  name                = "LukeLab-NC-Prod-Vnet"
  resource_group_name = "NetworkingTest-RG"
}

output "subnets" {
  value = [for s in data.azurerm_virtual_network.example.subnets : lower(s)]
}
```
When running this config the output is the following:

```
Outputs:

subnets = [
  "lukeapp-nc-prod-subnet",
  "lukeapp5-nc-prod-subnet",
  "lukeapp4-nc-prod-subnet",
  "lukeapp3-nc-prod-subnet",
  "lukeapp2-nc-prod-subnet",
]
```
Notice the for expression is wrapped in `[]`. This will give us a tuple output. We can also change the output type to an object by wrapping our for expression in `{}` instead. Also we must include two result expressions using the `=>` symbol to separate them:

```
data "azurerm_virtual_network" "example" {
  name                = "LukeLab-NC-Prod-Vnet"
  resource_group_name = "NetworkingTest-RG"
}

output "subnets" {
  value = {for s in data.azurerm_virtual_network.example.subnets : s => lower(s)}
}
```

The output then changes to an object type with out two results:

```
Outputs:

subnets = {
  "LukeApp-NC-Prod-Subnet" = "lukeapp-nc-prod-subnet"
  "LukeApp2-NC-Prod-Subnet" = "lukeapp2-nc-prod-subnet"
  "LukeApp3-NC-Prod-Subnet" = "lukeapp3-nc-prod-subnet"
  "LukeApp4-NC-Prod-Subnet" = "lukeapp4-nc-prod-subnet"
  "LukeApp5-NC-Prod-Subnet" = "lukeapp5-nc-prod-subnet"
}
```

We can also perform more advanced logic with our for expressions. We can filter out our results by including an if statement followed by some logic. In this example, we are only listing subnets that are not an empty string:
```
[for s in data.azurerm_virtual_network.example.subnets : lower(s) if s != ""
```

For a more advanced example of this, we can use `regexall` to filter our results to only include subnets that meet our regex requirements. In this example, we are selecting only subnets with LukeApp 2-4 in the name. Also, note we are wrapping our `regexall` function with a `length` function and then checking to ensure that it is greater than 0. This is the recommended way to validate a string that matches the regex requirements. If we were to skip using `length` the if expression would error out when the regex query doesn't match the other subnets:

```
data "azurerm_virtual_network" "example" {
  name                = "LukeLab-NC-Prod-Vnet"
  resource_group_name = "NetworkingTest-RG"
}

output "subnets" {
  value = [for s in data.azurerm_virtual_network.example.subnets : s if length(regexall("LukeApp[2-4]-NC-Prod-Subnet", s)) > 0]

}
```
When we run this, we get only the subnets that matched our regex statement:
```
Outputs:

subnets = [
  "LukeApp4-NC-Prod-Subnet",
  "LukeApp3-NC-Prod-Subnet",
  "LukeApp2-NC-Prod-Subnet",
]
```

##### Splat Expressions
Splat expressions are a cleaner way of performing some of the same tasks that we can do with a `for` expression. In our previous `for` expression example, we listed subnets of a virtual network like the following:
```
[for s in data.azurerm_virtual_network.example.subnets : s]
```
Instead, we can use a splat expression like so. The [*] symbol directs us to iterate over all the elements in the list:
```
data.azurerm_virtual_network.example[*].subnets
```

We could also reference the index of the attribute to select the first subnet in the list:
```
data.azurerm_virtual_network.example[*].subnets[0]
```

Complex expressions are a great way to add logic to our Terraform configurations and allow us to create modules that can be versatile. For example, if we wanted all VMs deployed in the development environment to use slower, less expensive disk types, we could code the logic with a conditional expression. Using a variable `var.environment` to state the environment we want to deploy to, we could then reference that variable to conditionally decide to use either the `Standard_LRS` or `Premium_LRS` disk type:

```
resource "azurerm_virtual_machine" "main" {
  ...

  storage_os_disk {
    name              = "myosdisk1"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = var.environment == "Development" ? "Standard_LRS" :  "Premium_LRS"
  }
  ...
```
Functions and complex expressions can do a lot of our heavy lifting, however there is one more tool in our tool belt that can help simplify our code. In the next section we will cover loops. 


## Step 3 —  Loops

Loops in Terraform can help us in many ways. We can quickly iterate through a collection of components with a loop. We can also scale our resources efficiently using loops. Here are some of the ways we can loop through resources in Terraform:


##### Count

Count is a very popular technique in Terraform configurations. It allows us to create multiple instances of a resource block. Almost all resource blocks can use the `count` attribute. Below is an example of an `azurerm_resource_group` resource block that has the `count` attribute set to `3`. This tells Terraform to create this resource three times. Also note, we are modifying the name of the resource group each time; otherwise, our code would error out because we can't have resource groups with duplicate names. We are using the `count.index` variable, which is accessible only when the `count` argument is used. `count.index` specifies the index value of our resource, which is the number assigned to that resource's "spot in the list". The index numbering starts at 0. So if we are deploying a resource with a `count` of `3` we will have three indexes of 0,1, and 2:

```
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "main" {
  count = 3
  name     = "rg-MyResourceGroup-${count.index}"
  location = "West US 2"
}
```
When we run `terraform plan` against this configuration we can see that we will be deploying three resource groups:

```
Terraform will perform the following actions:

  # azurerm_resource_group.main[0] will be created
  + resource "azurerm_resource_group" "main" {
      + id       = (known after apply)
      + location = "westus2"
      + name     = "rg-MyResourceGroup-0"
    }

  # azurerm_resource_group.main[1] will be created
  + resource "azurerm_resource_group" "main" {
      + id       = (known after apply)
      + location = "westus2"
      + name     = "rg-MyResourceGroup-1"
    }

  # azurerm_resource_group.main[2] will be created
  + resource "azurerm_resource_group" "main" {
      + id       = (known after apply)
      + location = "westus2"
      + name     = "rg-MyResourceGroup-2"
    }

Plan: 3 to add, 0 to change, 0 to destroy.

```

Count is used for conditional logic as well. If we set the `count` value of a resource to `0`, Terraform will deploy 0 copies of that resource. This is handy for creating the logic to deploy resources only in certain circumstances. In the example below, we have a bool variable for `bootdiag_storage`. We also have a resource block for azure storage to be used for our boot diagnostic storage. Notice the `count` argument now has a conditional expression to evaluate if the `bootdiag_storage` variable is true or not. If the variable is set to true, set the `count` argument to `1`, indicating that we will deploy this resource. If the variable is set to false, `count` is then set to `0` indicating that the `azure_storage_account` resource will not be deployed:

```
provider "azurerm" {
  features {}
}

variable "bootdiag_storage" {
  type = bool
  description = "Enter the name of the boot diagnostic storage account if one is desired"
  default = false
}

resource "azurerm_resource_group" "rg" {
  
  name     = "rg-MyResourceGroup"
  location = "West US 2"
}

resource "azurerm_storage_account" "bootdiag" {
  count = var.bootdiag_storage ? 1 : 0

  name                     = "myterraformvmsadiag"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
}
```


##### For_each

For_each loops allow us to iterate through a list or map. This allows us to have cleaner code. For example, instead of creating an `azurerm_network_security_rule` resource for each rule we want to create for our NSG, we can simply create a map variable called `inbound_rules`. We can then use `for_each` to loop through each item in our map and create a rule for each one. Also, note we are using `each.key` and `each.value` variables in our configuration to specify the key and value pairs in our map:
```
provider "azurerm" {
  features {}
}

variable "inbound_rules" {
  type = map
  description = "A map of allowed inbound ports and their priority value"
  default = {
    101 = 3389
    102 = 22
    103 = 443

  }
}

resource "azurerm_resource_group" "rg" {
  
  name     = "rg-mytesourcegroup"
  location = "West US 2"
}

resource "azurerm_network_security_group" "nsg" {
  name                = "nsg-mysecuritygroup"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_network_security_rule" "nsg_rule" {
  for_each = var.inbound_rules
  name                        = "port_${each.value}"
  priority                    = each.key
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = each.value
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.rg.name
  network_security_group_name = azurerm_network_security_group.nsg.name
}
```
When we run `terraform plan` we can see all our NSG rule resource blocks will be created with the values from our map:

```
Terraform will perform the following actions:

  # azurerm_network_security_group.nsg will be created
  + resource "azurerm_network_security_group" "nsg" {
      + id                  = (known after apply)
      + location            = "westus2"
      + name                = "nsg-mysecuritygroup"
      + resource_group_name = "rg-mytesourcegroup"
      + security_rule       = (known after apply)
    }

  # azurerm_network_security_rule.nsg_rule["101"] will be created
  + resource "azurerm_network_security_rule" "nsg_rule" {
      + access                      = "Allow"
      + destination_address_prefix  = "*"
      + destination_port_range      = "3389"
      + direction                   = "Inbound"
      + id                          = (known after apply)
      + name                        = "port_3389"
      + network_security_group_name = "nsg-mysecuritygroup"
      + priority                    = 101
      + protocol                    = "Tcp"
      + resource_group_name         = "rg-mytesourcegroup"
      + source_address_prefix       = "*"
      + source_port_range           = "*"
    }

  # azurerm_network_security_rule.nsg_rule["102"] will be created
  + resource "azurerm_network_security_rule" "nsg_rule" {
      + access                      = "Allow"
      + destination_address_prefix  = "*"
      + destination_port_range      = "22"
      + direction                   = "Inbound"
      + id                          = (known after apply)
      + name                        = "port_22"
      + network_security_group_name = "nsg-mysecuritygroup"
      + priority                    = 102
      + protocol                    = "Tcp"
      + resource_group_name         = "rg-mytesourcegroup"
      + source_address_prefix       = "*"
      + source_port_range           = "*"
    }

  # azurerm_network_security_rule.nsg_rule["103"] will be created
  + resource "azurerm_network_security_rule" "nsg_rule" {
      + access                      = "Allow"
      + destination_address_prefix  = "*"
      + destination_port_range      = "443"
      + direction                   = "Inbound"
      + id                          = (known after apply)
      + name                        = "port_443"
      + network_security_group_name = "nsg-mysecuritygroup"
      + priority                    = 103
      + protocol                    = "Tcp"
      + resource_group_name         = "rg-mytesourcegroup"
      + source_address_prefix       = "*"
      + source_port_range           = "*"
    }

  # azurerm_resource_group.rg will be created
  + resource "azurerm_resource_group" "rg" {
      + id       = (known after apply)
      + location = "westus2"
      + name     = "rg-mytesourcegroup"
    }

Plan: 5 to add, 0 to change, 0 to destroy.
```



##### Dynamic Blocks

Resources that contain repeatable configuration blocks can make use of dynamic blocks. For example, the `azurerm_virtual_machine` resource includes a `storage_data_disk` block for defining the configuration of our VM data disks. This block can be used multiple times within `azurerm_virtual_machine` to specify multiple data disks. Instead of repeating the same block structure over and over again, we can use dynamic blocks to simplify our code. This is done by prefixing the block with `dynamic` and using `for_each` to iterate through our `storage_data_disk` block. 

In this example, we have a variable `disk_size_gb` that can accept a string of comma-separated numbers to indicate the size of the disks to add. So if we wanted three disks that are sized as 32GB, 64GB, and 128GB we would set this variable to "32,64,128" in our `terraform.tfvars` file. 

In the `for_each` argument underneath our `dynamic storage_data_disk` block, we are using the `split` function to separate our disk size values and iterate through each one. We are using `storage_data_disk.value` to reference the value of each number in `disk_size_gb` fo each disk size. Also, we are using `storage_data_disk.key` to reference the index number of our `storage_data_disk` block and using that value in the `name` and `lun` arguments to provide unique values:


```

variable "disk_size_gb" {
  description = "Size of disks in GBs. Choose multiple disks by comma separating each size"
  type        = string
  default = "64"
}

# Create virtual machine
resource "azurerm_virtual_machine" "vm" {
  name                  = var.servername
  location              = azurerm_resource_group.rg.location
  resource_group_name   = azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.nic.id]
  vm_size               = "Standard_D2s_v3"
...
  dynamic storage_data_disk {
    for_each = split(",", var.disk_size_gb)
      content {
        name              = "${var.servername}_datadisk_${storage_data_disk.key}"
        create_option     = "Empty"
        lun               = storage_data_disk.key
        disk_size_gb      = storage_data_disk.value
        managed_disk_type = "StandardSSD_LRS"
      }
  }
...
```
When we run `terraform plan` against this configuration we get the following output specifying the `storage_data_disk` blocks for our three disks:

```
 # azurerm_virtual_machine.vm will be created
  + resource "azurerm_virtual_machine" "vm" {
      + availability_set_id              = (known after apply)
      + delete_data_disks_on_termination = false
      + delete_os_disk_on_termination    = false
      + id                               = (known after apply)
      + license_type                     = (known after apply)
      + location                         = "westus2"
      + name                             = "vmterraform"
      + network_interface_ids            = (known after apply)
      + resource_group_name              = "rg-terraexample"
      + tags                             = (known after apply)
      + vm_size                          = "Standard_D2s_v3"

      + identity {
          + identity_ids = (known after apply)
          + principal_id = (known after apply)
          + type         = (known after apply)
        }

      + os_profile {
          + admin_password = (sensitive value)
          + admin_username = "joeblow"
          + computer_name  = "vmterraform"
          + custom_data    = "c991e3a089d3ad26ec85af93f698c375db0933f2"
        }

      + os_profile_windows_config {
          + enable_automatic_upgrades = false
          + provision_vm_agent        = true

          + additional_unattend_config {
              + component    = "Microsoft-Windows-Shell-Setup"
              + content      = (sensitive value)
              + pass         = "oobeSystem"
              + setting_name = "FirstLogonCommands"
            }
          + additional_unattend_config {
              + component    = "Microsoft-Windows-Shell-Setup"
              + content      = (sensitive value)
              + pass         = "oobeSystem"
              + setting_name = "AutoLogon"
            }
        }

      + storage_data_disk {
          + caching                   = (known after apply)
          + create_option             = "empty"
          + disk_size_gb              = 32
          + lun                       = 0
          + managed_disk_id           = (known after apply)
          + managed_disk_type         = "StandardSSD_LRS"
          + name                      = "vmterraform_datadisk_0"
          + write_accelerator_enabled = false
        }
      + storage_data_disk {
          + caching                   = (known after apply)
          + create_option             = "empty"
          + disk_size_gb              = 64
          + lun                       = 1
          + managed_disk_id           = (known after apply)
          + managed_disk_type         = "StandardSSD_LRS"
          + name                      = "vmterraform_datadisk_1"
          + write_accelerator_enabled = false
        }
      + storage_data_disk {
          + caching                   = (known after apply)
          + create_option             = "empty"
          + disk_size_gb              = 128
          + lun                       = 2
          + managed_disk_id           = (known after apply)
          + managed_disk_type         = "StandardSSD_LRS"
          + name                      = "vmterraform_datadisk_2"
          + write_accelerator_enabled = false
        }

      + storage_image_reference {
          + offer     = "WindowsServer"
          + publisher = "MicrosoftWindowsServer"
          + sku       = "2019-Datacenter"
          + version   = "latest"
        }

      + storage_os_disk {
          + caching                   = "ReadWrite"
          + create_option             = "FromImage"
          + disk_size_gb              = (known after apply)
          + managed_disk_id           = (known after apply)
          + managed_disk_type         = "Premium_LRS"
          + name                      = "stvmvmterraformos"
          + os_type                   = (known after apply)
          + write_accelerator_enabled = false
        }
    }
```

Loops are great for reducing repetitive code and creating scalable resources. However, they can also add additional complexity for others reading the Terraform configuration, so choose wisely when using loops within Terraform code. 

## Conclusion

In this article, we learned about functions, complex expressions, and loops. We reviewed examples and discussed their benefits and use-cases within our Terraform code. The HCL language is both human-readable and machine-friendly enough to perform complex logic described in this guide. However, there is a balance to writing infrastructure code that will not only get the job done but also provide documentation to another team member. Use them where it makes sense but not to the point where code is hard to understand. Remember, simplicity is a crucial component of creating long-lasting automated solutions. 
