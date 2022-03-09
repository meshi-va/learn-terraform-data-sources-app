# Learn Terraform data sources | APP repository

I forked this repo in order to complete [this tutorial](https://learn.hashicorp.com/tutorials/terraform/data-sources?in=terraform/certification-associate-tutorials) which is part of the [Terraform Associate Tutorial List](https://learn.hashicorp.com/collections/terraform/certification-associate-tutorials).

This repo relies on [meshi-va/learn-terraform-data-sources-vpc](https://github.com/meshi-va/learn-terraform-data-sources-vpc), hence it should be only used with `terraform apply` once the configuration in the previously mentioned repo is applied. 

In this README I'll try to document the tasks that were the most challenging for me to understand, and explain them in more digestable way.

## Resource overview

### Defined in the VPC repo

In the [previous repo](https://github.com/meshi-va/learn-terraform-data-sources-vpc) I configured 2 private subnets for my VPC.

They are defined in the `module.vpc` 

```
module "vpc" {
  // ...
  private_subnets = slice(var.private_subnet_cidr_blocks, 0, 2)
  // ...
}
```
By slicing the `private_subnet_cidr_blocks` variable in the `variables.tf` file, I define the first and the third object as the subnets to be used:

```
variable "private_subnet_cidr_blocks" {
  description = "Available cidr blocks for private subnets"
  type        = list(string)
  default = [
    "10.0.101.0/24",
    "10.0.102.0/24",
    "10.0.103.0/24",
    "10.0.104.0/24",
    "10.0.105.0/24",
    "10.0.106.0/24",
    "10.0.107.0/24",
    "10.0.108.0/24"
  ]
}
```

I also wrote an output block for `public_subnet_ids` so that this workspace can read it.

### Defined in this repo

Since my VPC configuration is applied in another workspace, I have to link it with a data source:

```
data "terraform_remote_state" "vpc" {
  backend = "local"

  config = {
    path = "../learn-terraform-data-sources-vpc/terraform.tfstate"
  }
}
```
The amount of instances per subnet was already defined in the `variables.tf` file:

```
variable "instances_per_subnet" {
  description = "Number of EC2 instances in each private subnet"
  type        = number
  default     = 2
}
```

### More complex tasks

One of the easier tasks was to change the instance count to 2 instances per private subnets:

```
resource "aws_instance" "app" {
count = var.instances_per_subnet * length(data.terraform_remote_state.vpc.outputs.private_subnet_ids)
}
```
Since the `var.instances_per_subnet` is hard coded as `2` and I defined the private subnet to be a list of 2, hence the `length(data.terraform_remote_state.vpc.outputs.private_subnet_ids)` will be `2` as well.

---

The other task was a bit harder to understand, but I'll do my best to explain it.

So the tutorial instructs me to add `data.terraform_remote_state.vpc.outputs.private_subnet_ids[count.index % length(data.terraform_remote_state.vpc.outputs.private_subnet_ids)]` as the subnet_id for the `aws_instance.app` resource.

This line of code is to define which private subnet ID to use for each instance.

