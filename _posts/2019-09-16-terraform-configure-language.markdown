---
layout: post
title:  "Terraform Configuration Language"
date:   2019-09-16 17:33:34 +0800
categories: Terraform
---

# Configuration Language

The Terraform language is **declarative**, describing an **intended goal** rather than the steps to reach that goal.

The main purpose of the Terraform language is declaring **resources**

A **resource** describes a **single** infrastructure object, while a **module** might describe a **set of objects** and the necessary relationships between them in order to create a higher-level system.

A Terraform configuration consists of a **root module** where evaluation begins, along with a **tree of child modules** created when one module calls another.

Example
```hcl
resource "aws_vpc" "main" {
  cidr_block = var.base_cidr_block
}

<BLOCK TYPE> "<BLOCK LABEL>" "<BLOCK LABEL>" {
  # Block body
  <IDENTIFIER> = <EXPRESSION> # Argument
}
```

The `Terraform command line interface` (`CLI`) is a general engine for evaluating and applying Terraform configurations.
This general engine has no knowledge about specific types of infrastructure objects.
Instead, Terraform uses plugins called `providers` that each define and manage a set of resource types.


## Resources
```hcl
provider "aws" {
  region = "us-east-1"
}
provider "aws" {
  alias  = "west"
}

variable "subnet_ids" {
  type = list(string)
}

resource "aws_instance" "web" {
  ami           = "ami-a1b2c3d4"
  instance_type = "t2.micro"
  count         = length(var.subnet_ids)
  subnet_id     = var.subnet_ids[count.index]
}

resource "aws_instance" "server" {
  provider      = aws.west
  for_each      = toset(var.subnet_ids)
  ami           = "ami-a1b2c3d4"
  instance_type = "t2.micro"
  subnet_id     = each.key # note: each.key and each.value are the same for a set
}
```

### Meta-Arguments
- depends_on: Explicit Resource Dependencies
- count: Multiple Resource Instances By Count. (`count.index` is available, referring instances by `<TYPE>.<NAME>[<INDEX>]`)
- for_each: Multiple Resource Instances Defined By a Map, or Set of Strings (`each.key`, `each.value` are available, referring instances by `<TYPE>.<NAME>[<KEY>]`)
- provider: Selecting a Non-default Provider Configuration
- lifecycle: Lifecycle Customizations. (`create_before_destroy`(bool), `prevent_destroy`(bool), `ignore_changes`(by `tags` or `all`))
- provisioner and connection: Resource Provisioners

### Local-only Resources
While most resource types correspond to an infrastructure object type that is managed via a remote network API,
there are certain specialized resource types that operate only within Terraform itself, calculating some results and saving those results in the state for future use.

### Operation Timeouts
`Some` resource types provide a special timeouts nested block argument that allows you to customize how long certain operations are allowed to take before being considered to have failed.
```hcl
resource "aws_db_instance" "example" {
  # ...

  timeouts {
    create = "60m"
    delete = "2h"
  }
}
```


## Providers
- initialization: `terraform init`
- meta-arguments: `version`, `alias`(referring by `<PROVIDER NAME>.<ALIAS>`)

### Third-party Plugins
- See [Writing Custom Providers](https://www.terraform.io/docs/extend/writing-custom-providers.html)
- Install to `~/.terraform.d/plugins`
- Plugin Names and Versions: `terraform-provider-<NAME>_vX.Y.Z`


## Input Variables
```hcl
#Declare
variable "docker_ports" {
  type = list(object({
    internal = number
    external = number
    protocol = string
  }))
  default = [
    {
      internal = 8300
      external = 8300
      protocol = "tcp"
    }
  ]
  description = "docker port mappings"
}

#Using
resource "aws_instance" "example" {
  instance_type = "t2.micro"
  ami           = var.image_id
}
```

### Type
`string`, `number`, `bool`, `list(<TYPE>)`, `set(<TYPE>)`, `map(<TYPE>)`, `object({<ATTR NAME> = <TYPE}, ...)`, `tuple([<TYPE, ...>])`, `any`
If no type constraint is set then a value of `any` type is accepted.

### Assigning Values to Root Module Variables

#### Environment Variables
```bash
export TF_VAR_image_id=ami-abc123
```

#### Variables on the Command Line
```bash
terraform apply -var="image_id=ami-abc123"
terraform apply -var='image_id_list=["ami-abc123","ami-def456"]'
terraform apply -var='image_id_map={"us-east-1":"ami-abc123","us-east-2":"ami-def456"}'
```

#### Variable Definitions (`.tfvars`) Files
```bash
terraform apply -var-file="testing.tfvars"
```

testing.tfvars
```
image_id = "ami-abc123"
availability_zone_names = [
  "us-east-1a",
  "us-west-1c",
]
```

Terraform also automatically loads a number of variable definitions files if they are present:
- Files named exactly terraform.tfvars or terraform.tfvars.json.
- Any files with names ending in .auto.tfvars or .auto.tfvars.json.

terraform.tfvars.json
```
{
  "image_id": "ami-abc123",
  "availability_zone_names": ["us-west-1a", "us-west-1c"]
}
```


### Variable Definition Precedence
- Environment variables
- The `terraform.tfvars` file, if present.
- The `terraform.tfvars.json` file, if present.
- Any `*.auto.tfvars` or `*.auto.tfvars.json` files, processed in lexical order of their filenames.
- Any `-var` and `-var-file` options on the command line, in the order they are provided. (This includes variables set by a Terraform Cloud workspace.)


## Output Values
Output values are like the return values of a Terraform module, and have several uses:
- A child module can use outputs to expose a subset of its resource attributes to a parent module.
- A root module can use outputs to print certain values in the CLI output after running terraform apply.
- When using remote state, root module outputs can be accessed by other configurations via a terraform_remote_state data source.

```hcl
output "db_password" {
  value       = aws_db_instance.db.password
  description = "The password for logging in to the database."
  sensitive   = true
  depends_on  = []
}
```

Accessing Child Module Outputs: `module.<MODULE NAME>.<OUTPUT NAME>`


## Local Values

```hcl
locals {
  # Ids for multiple sets of EC2 instances, merged together
  instance_ids = concat(aws_instance.blue.*.id, aws_instance.green.*.id)
}

locals {
  # Common tags to be assigned to all resources
  common_tags = {
    Service = local.service_name
    Owner   = local.owner
  }
}
```


## Modules
A module is a container for multiple resources that are used together.
Every Terraform configuration has at least one module, known as its `root module`, which consists of the resources defined in the .tf files in the main working directory.

```hcl
# Calling a Child Module
module "servers" {
  source  = "./app-cluster"
  servers = 3
  servers = 5
  providers = {
    aws = "aws.usw2"
  }
}

# Accessing Module Output Values
resource "aws_elb" "example" {
  instances = module.servers.instance_ids
}
```


## Data Sources
Data sources allow data to be fetched or computed for use elsewhere in Terraform configuration. 
Use of data sources allows a Terraform configuration to make use of information defined outside of Terraform, or defined by another separate Terraform configuration.

```hcl
# Declare the data source
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "primary" {
  availability_zone = "${data.aws_availability_zones.available.names[0]}"
  # ...
}
```

