# Learn Terraform data sources | APP repository

I forked this repo in order to complete [this tutorial](https://learn.hashicorp.com/tutorials/terraform/data-sources?in=terraform/certification-associate-tutorials) which is part of the [Terraform Associate Tutorial List](https://learn.hashicorp.com/collections/terraform/certification-associate-tutorials).

This repo relies on [meshi-va/learn-terraform-data-sources-vpc](https://github.com/meshi-va/learn-terraform-data-sources-vpc), hence it should be only used with `terraform apply` once the configuration in the previously mentioned repo is applied. 

In this README I'll try to document the tasks that were the most challenging for me to understand, and explain them in a more digestable way.

## Objectives

1. In this tutorial, you will use Terraform to deploy a workspace containing a VPC and security groups on AWS in the `us-east-1` region. 
2. Next, you will use the `aws_availability_zones` data source to configure your VPC's Availability Zones (AZs), allowing you to deploy this configuration in any AWS region.
3. Then, you will use the `terraform_remote_state` data source to deploy another workspace containing your application infrastructure to your VPC. 
4. Finally, you will use the `aws_ami` data source to configure the correct AMI for the current region.

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

---

## Detailed tasks

One of the easier tasks was to change the instance count to 2 instances per private subnets:

```
resource "aws_instance" "app" {
count = var.instances_per_subnet * length(data.terraform_remote_state.vpc.outputs.private_subnet_ids)
}
```
Since the `var.instances_per_subnet` is hard coded as `2` and I defined the private subnet to be a list of 2, the `length(data.terraform_remote_state.vpc.outputs.private_subnet_ids)` will be `2` as well.

---

The other task was a bit harder to understand, but I'll do my best to explain it.

So the tutorial instructs me to add `data.terraform_remote_state.vpc.outputs.private_subnet_ids[count.index % length(data.terraform_remote_state.vpc.outputs.private_subnet_ids)]` as the `subnet_id` for the `aws_instance.app` resource:

```
resource "aws_instance" "app" {
 // ...
 subnet_id              = data.terraform_remote_state.vpc.outputs.private_subnet_ids[count.index % length(data.terraform_remote_state.vpc.outputs.private_subnet_ids)]
 // ...
}
```

This line of code is to define which private subnet ID to use for each instance.

But how does it exactly does it do that?

In the previous repo, I defined an output block for the private subnet IDs. These IDs form a list of 2:

```
private_subnet_ids = [
  "subnet-0478ce1fe723c6c1a",
  "subnet-05a921f34e27a88de",
]
```

`count.index`
: The distinct index number (starting with 0) of each instance. 

`length(data.terraform_remote_state.vpc.outputs.private_subnet_ids)`
: The length of the list of private subnet ids (2 in our case)

`count.index % length`
: The remainder of `count.index` and `length` which indicates the index of `outputs.private_subnet_ids` that will be used.

---

**A visual representation of how Terraform will allocate the recources:**

| instance | count.index | length | remainder |  private_subnet_ids | subnet id |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| aws_instance.app[0] | 0 | 2 | 0 | private_subnet_ids[0] | subnet-0478ce1fe723c6c1a |
| aws_instance.app[1] | 1 | 2 | 1 | private_subnet_ids[1] | subnet-05a921f34e27a88de |
| aws_instance.app[2] | 2 | 2 | 0 | private_subnet_ids[0] | subnet-0478ce1fe723c6c1a |
| aws_instance.app[3] | 3 | 2 | 1 | private_subnet_ids[1] | subnet-05a921f34e27a88de |

To make the above table readable, I replaced `data.terraform_remote_state.vpc.outputs.private_subnet_ids[]` with `private_subnet_ids`.

## Completed tasks

- Added a `terraform_remote_state` data source block to `main.tf`
- Changed the `aws` provider `region` to the region output from the VPC workspace
- Configured the load balancer security group and subnet with the corresponding outputs from the VPC workspace
- Updated `main.tf` to use multiple EC2 instances per subnet
- Configured region-specific AMIs by adding the `aws_ami` data source block and changing the the `ami` id of the `aws_instance` resource
- Configured EC2 subnet and security groups, which I described in detail above
