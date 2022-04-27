# Getting Started with Terraform on Azure: Modules

### Introduction

When creating production-grade Terraform configurations, modules are an absolute must. One of the more apparent benefits of using them is that they allow our code to be DRY. DRY is a software development term that stands for Don't Repeat Yourself. The idea is to reduce the amount of repetition in our code. In Terraform, we can create modules to build re-usable components of our infrastructure. For example, we can have a module for SQL servers and a separate one for Virtual Machines. We can then re-use each module to deploy services and build out the infrastructure for various environments.

Modules should also be used as a way to split up a large environment into smaller components. We don't want to have a single main.tf file with over 1000 lines of code. Splitting up our code into smaller modules allows us to make changes to our environment safely without affecting large chunks of code. Also, by splitting our environment up into modules, we now have pieces of our infrastructure separated into a testable module. The ability to use software development testing practices to test our Terraform code is an enormous benefit of having infrastructure defined in code in the first place. 

Lastly, modules also provide a way for Terraform users to share their configurations either privately or within the Terraform community. In 2019 HCL was the [3rd fastest-growing programming language on GitHub](https://octoverse.github.com/), which validates the accelerated adoption of the HashiCorp product stack. 

In this guide, we are going to create a module and learn how to integrate it into our Terraform configurations.

## Prerequisites

Before you begin, you'll need to set up the following:
 - [Azure subscription](https://azure.microsoft.com/en-us/).
 - Azure Cloud Shell. Be sure to check out the prerequisites on ["Getting Started with Terraform on Azure: Deploying Resources"](https://cloudskills.io/blog/terraform-azure-01) for a guide on how to set this up.


## Step 1 — Module Architecture

In a real-world Terraform environment, we wouldn't want to re-create the same code over and over again for deploying infrastructure. This would create a large amount of redundancy in our Terraform code. Instead, we would want to break up our Terraform configurations into modules; typically, the best practice is a module for each component. For example, we could create a module for SQL databases that contain all of our configurations for deploying SQL with our needs. We could then re-use that module whenever a SQL database is needed and call it within our Terraform configurations. 

The diagram below demonstrates the strategy of splitting up the various Azure services by component modules. By creating four modules for each service in this environment, we can also re-use the same code in both Dev, QA, and Prod. This practice ensures accurate infrastructure comparisons between each environment throughout each stage of development. We are no longer copying and pasting our code from dev to QA to Prod. Instead, we parameterize our modules to allow us to customize slightly for each environment, such as resource names and networking subnets:

![UsingModules](/IntroToTerraform-Modules/Images/UsingModules.png)

Creating a module for each cloud service also allows us to re-use modules in other projects as well. Our Terraform modules turn into building blocks that can be used over and over again to create infrastructure on demand.


## Step 2 — Creating Modules

Creating modules in Terraform is very easy; all we need are input variables and a standard configuration of resources. In the example, we are going to create our first module for a storage account. We will start by creating a module folder and then reference that module in another Terraform configuration. We will begin with a folder hierarchy like the following:

```
terraformdemo
    └──modules 
            └──storage-account
                    └── main.tf
                    └── variables.tf
```

Copy the code for the `main.tf` and `variables.tf` configurations and create each file. We'll place each file according to the directory structure above.

The `variables.tf` file contains our input variables. The input variables are the parameters that our module accepts to customize its deployment. For our storage account module, we are keeping it as simple as possible for the example by receiving inputs for the storage account name, location, and resource group:

####**variables.tf**
```
variable "saname" {
    type = string
    description = "Name of storage account"
}
variable "rgname" {
    type = string
    description = "Name of resource group"
}
variable "location" {
    type = string
    description = "Azure location of storage account environment"
    default = "westus2"
}
```
The `main.tf` file contains the code for creating a storage account. In the example we only have a small set of arguments for our storage account to keep things simple. However, in a real production environment, we would possibly want to implement network policies as well as logging options. We could then use our module to define the 'standards' for how we want all our storage accounts to be configured:

####**main.tf**
```
resource "azurerm_storage_account" "sa" {
  name                     = var.saname
  resource_group_name      = var.rgname
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

}
```
Next, we will create another `main.tf` file at the root of our `terraformdemo` folder which will reference our newly created storage account module directory:

```
terraformdemo
    └── main.tf
    └──modules 
            └──storage-account
                    └── main.tf
                    └── variables.tf
```
In the root `main.tf`, we call our module using a `module` block followed by a string parameter. Inside the block, we need to reference the module that we are using by declaring a `source` argument. In this example, we are merely referencing the module in our `modules` subfolder, so the path is `./modules/storage-account`. We also need to include any required variable inputs for our storage account module. These are the same variables that we created in the `variables.tf` file in our storage account modules directory:
>**Note:** The storage account name must be unique and no more than 24 characters long or you may run into failures during deployment. 
####**main.tf**
```
provider "azurerm" {
  version = "1.38.0"
}

#create resource group
resource "azurerm_resource_group" "rg" {
    name     = "rg-MyFirstTerraform"
    location = "westus"
}

#Create Storage Account
module "storage_account" {
  source    = "./modules/storage-account"

  saname    = "statfdemosa234"
  rgname    = azurerm_resource_group.rg.name
  location  = azurerm_resource_group.rg.location
}
```

In our `main.tf` file, we also include the `azurerm` provider block. It is best practice to specify the provider at the root module file; that way, all modules that are called will then inherit this provider. When creating modules, try not to include a provider inside the module itself as much as possible. This can cause further complexity and make modules brittle. When we run our `terraform init` in the `terraformdemo` directory we can see that the module is initialized:
```
Initializing modules...
- storage_account in modules/storage-account

Initializing the backend...

Initializing provider plugins...
- Checking for available provider plugins...
- Downloading plugin for provider "azurerm" (hashicorp/azurerm) 1.38.0...
```
When we run `terraform apply`, it will reference the `storage-account` module to create our storage account with the settings we declared in the module input. Also, we can use the same module multiple times in a configuration with a different parameter string:

```
provider "azurerm" {
  version = "1.38.0"
}

#create resource group
resource "azurerm_resource_group" "rg" {
    name     = "rg-MyFirstTerraform"
    location = "westus"
}

#Create Storage Account
module "storage_account" {
  source    = "./modules/storage-account"

  saname    = "statfdemosa234"
  rgname    = azurerm_resource_group.rg.name
  location  = azurerm_resource_group.rg.location
}

#Create Storage Account
module "storage_account2" {
  source    = "./modules/storage-account"

  saname    = "statfdemosa241"
  rgname    = azurerm_resource_group.rg.name
  location  = azurerm_resource_group.rg.location
}
```
We just created our first module. This module is straightforward, however, for more complex scenarios like deploying a Virtual Machine with encrypted disks, a module can be perfect for abstracting all the complexity away with just a few inputs. 



## Step 3 —  Module Outputs 

Variable inputs are not the only important part of a module. Outputs are just as important as well. They allow us to transfer information to and from modules so that they can build off of each other. Creating an output for a module is the same process as with a regular Terraform configuration. Create an `output.tf` file and use an `output` block to declare our output values. Below we are creating an output block for our storage account primary access key so we can store it in an Azure Key Vault after it is created:

```
output "primary_key" {
    description = "The primary access key for the storage account"
    value = azurerm_storage_account.sa.primary_access_key
    sensitive   = true
}

```
Also note, we are using the `sensitive` argument to specify that the `primary_access_key` output for our storage account contains sensitive data. Terraform will treat this information as confidential and hide it from the console display when running `terraform apply`. This does not protect the value from within a Terraform's state file; it will still be in cleartext, which is why in a real-world production scenario, we would want to use remote state. 

To use a module's output values in another resource, specify the values by referencing it in the `module.<modulename>.<outputname>` format:

```
resource "azurerm_key_vault_secret" "stasecret" {
  name         = "statfdemosa234-secret"
  value        = module.storage_account.primary_key
  key_vault_id = azurerm_key_vault.kv.id

}
```


## Step 4 — Git Modules

If we plan to share this module throughout multiple environments, its best practice to put the module in a source control repository, we then get all the benefits of source control for our module like change tracking. Additionally, we also get version tagging. Tagging modules is a best practice because it allows us to "pin" a stable working version of our module to a Terraform configuration. This prevents any breaking changes from affecting configurations that are already in production. 

To use a Terraform module from a git repository, change the `source` argument to the git URL. In our example, I have uploaded our storage account module to an Azure DevOps Repo. This is a public git repo and will not require any authentication configuration. We can use the https URL and prefix it with `git::`:

```
#Create Storage Account
module "storage_account" {
  source    = "git::https://allanore@dev.azure.com/allanore/TerraformModulesExample/_git/TerraformModulesExample?ref=v0.1"

  saname    = "tfdemosa23432"
  rgname    = azurerm_resource_group.rg.name
  location  = azurerm_resource_group.rg.location
}

```
If we run a `terraform init` we can see in the console output that the module is downloaded from the git repo and saved to the `.terraform/modules` local directory:
```
...
Please install a compatible extension version or remove it.
Initializing modules...
Downloading git::https://allanore@dev.azure.com/allanore/TerraformModulesExample/_git/TerraformModulesExample?ref=v0.1 for storage_account...
- storage_account in .terraform/modules/storage_account
Downloading git::https://allanore@dev.azure.com/allanore/TerraformModulesExample/_git/TerraformModulesExample?ref=v0.1 for storage_account2...
- storage_account2 in .terraform/modules/storage_account2

Initializing the backend...
...

```


Also, if we wanted to use a private Azure Repo with SSH, we could reference our module in the `source` argument via an SSH URL like below. We would also need to [generate](https://docs.microsoft.com/en-us/azure/devops/repos/git/use-ssh-keys-to-authenticate?view=azure-devops&tabs=current-page) and install the SSH certificate for authentication:
```
git::git@ssh.dev.azure.com:v3/allanore/TerraformModulesExample/TerraformModulesExample?ref=v0.1
```
For using a Terraform module source from a GitHub repo, use the URL to the GitHub project. In the example below, I uploaded our module over to a Github repo:

```
#Create Storage Account
module "storage_account" {
  source    = "github.com/allanore/TerraformModulesExample"

  saname    = "tfdemosa23432"
  rgname    = azurerm_resource_group.rg.name
  location  = azurerm_resource_group.rg.location
}

```

The recommended folder structure for a Terraform module repo looks like the following. We have our root module configuration files at the root of our repository directory, which in this example, is `storage-account`. We also have a README.md at the root folder. This is a markdown file that contains the information about our module. It's recommended to have README.md files for every Terraform configuration to describe what it is and how it is used. Next, we have our `modules` folder, which contains any sub-modules that would be needed to perform additional tasks, for example, configuring Private Link or setting up a Static Website. We also have our `examples` directory, which should contain examples of every possible scenario of our module. Lastly, we have our `test` folder, which includes test files written in Golang to test our module using the examples from the example folder; we will go more into testing modules in a later article in this series:
```
storage-account
    └── README.md
    └── main.tf
    └── variables.tf
    └── outputs.tf
    ├───modules 
    │     ├──private-link
    │     │       └── README.md
    │     │       └── main.tf
    │     │       └── outputs.tf
    │     │       └── variables.tf
    │     └──static-website
    │             └── README.md
    │             └── main.tf
    │             └── outputs.tf
    │             └── variables.tf 
    ├───examples
    │     ├───public
    │     │       └── README.md
    │     │       └── main.tf
    │     │       └── outputs.tf
    │     │       └── variables.tf
    │     ├──private-link
    │     │       └── README.md
    │     │       └── main.tf
    │     │       └── outputs.tf
    │     │       └── variables.tf 
    │     └──static-website
    │             └── README.md
    │             └── main.tf
    │             └── outputs.tf
    │             └── variables.tf
    └───test
         └── sa_public_test.go
         └── sa_private_test.go
         └── sa_static_website_test.go
```
This module structure is how we can create production-grade Terraform modules that can be used for every project. It seems like a lot of work for creating a module and can be overwhelming; however, start small and slowly build out your module. A complex module can take an experienced developer several months to build. 

## Step 5 - Terraform Registry
Building a module can take a long time; however, there are thousands of modules shared by the community that you can take advantage of by using them as a base or just using them on their own. The [Terraform Registry](https://registry.terraform.io/) is a centralized place for community-made Terraform modules. It is a good idea to check the Terraform Registry before building your own module to save time. This is also a great learning tool since you can also view the project on GitHub and see how the module is done and the logic used behind it. 

A Terraform Registry can also be private and used via Terraform Cloud. The modules that are on the public Terraform Registry can be used by referencing them in the `<namespace>/<name>/<provider>` format. In the example below, we are using a [module to deploy Azure Functions](https://registry.terraform.io/modules/InnovationNorway/function-app/azurerm/0.1.2?tab=resources) from the Terraform Registry:

```
resource "azurerm_resource_group" "rg" {
  name     = "rg-MyFirstTerraform"
  location = "westus"
}

module "function-app" {
  source  = "InnovationNorway/function-app/azurerm"
  version = "0.1.2"

  function_app_name = "func-terrademo"
  resource_group_name = azurerm_resource_group.rg.name
  location = azurerm_resource_group.rg.location
}


```
When we run `Terraform init`, the module is automatically downloaded and used during the `terraform apply`.


## Conclusion

In this article, we learned about modules and how we can use them to abstract our Terraform configurations. We went over how to create a module and how to reference the output from a module. We also looked at how to store our modules in a git repository like GitHub and Azure Repos. Lastly, we learned about the Terraform Registry and the community-made modules stored there. 

In the next article, we will learn about more advanced HCL concepts like for loops, operators, and functions, which will allow us to perform more advanced infrastructure deployments. 