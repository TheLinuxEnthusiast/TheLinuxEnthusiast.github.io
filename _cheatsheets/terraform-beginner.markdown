---
title: "Terraform Beginner Commands and Concepts"
layout: post
name: "terraform-beginner"
date: 2022-11-24 00:00:00 +0100
tags: ["Terraform", "IaC"]
author: "Darren Foley"
show_sidebar: false
menubar: cheatsheet_menu
---

<br>

<h4>Remote Backend and Provider version</h4>


<p>When working with terraform, its best practice to create both a remote backend location and to specify the version of the terraform providers you are using.</p>

Within the terraform block use a required providers block with a specific version.

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
```

Then add a backend block within the terraform block to specify which S3 bucket to use as a remote backend (if you are using the AWS provider).

```
terraform {

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }

  backend "s3" {
    bucket = "terraform-backend-bucket"
    key    = "backend/terraform.tfstate"
    region = "eu-west-1"
  }
}
```

Once completed run **terraform init** from the working directory to initialze the project.

You can move your local terraform.tfstate to a remote backend even with resources still active. Simply run the following after the backend block has been added to your config:

```
terraform init -migrate-state
```

<br>


<h4>Write, plan, apply</h4>

The standard Terraform workflow is write, plan & apply. 

1. WRITE: Create your configuration describing all resources, input variables and output variables.
2. PLAN: Create an execution plan to assess what resources will be created. **terraform plan**
3. APPLY: Run terraform apply to create the resources. **terraform apply**


<br>

<h4>Project Structure: beginner</h4>

<p>For a simple beginner projects your configuration should have the following structure.</p>

```
./tf-project
	main.tf
	variables.tf
	outputs.tf
	development.tfvars	
```

Terraform will look for any files ending in '.tf' so you could seperate the main.tf into seperate components/files which is also acceptable.

```
./tf-project
	network.tf
	ec2.tf
	load_balancer.tf
	variables.tf
	outputs.tf
	development.tfvars
```


<br>

<h4>Rewrite code with canonical format</h4>

Hashicorp recommends that you format your code to align with their style conventions as outlined [here](https://developer.hashicorp.com/terraform/language/syntax/style)

From your project root run the following.


```

terraform fmt

```

If your project contains sub-directories and modules your will need to add the -recursive argument

```

terraform fmt -recursive

```

<br>

<h4>Validate your terraform syntax</h4>

<p>To check that your terraform code is logically consistant and does not contain any missing parameters.</p>

```

terraform validate

```

<br>

<h4>Common Block types</h4>

The **resource** block is the most common block type. This describes any remote resource such as VPC's, virtual machines, containers or network route tables that you wish to create.

```

resource "aws_vpc" "custom_vpc" {
	cidr_block = "10.10.0.0/16"

}

```

*aws_vpc* refers to the resource type within the provider

*custom_vpc* is the custom string that describes the resource for that provider.

Each resouce block will contain mandatory & optional arguments. Terraform will attempt to infer optional arguments that were not specified.


<br>


The **variable** block will describe an input variable that the terraform project requires. If a variable is not supplied with a default value, you will be prompted at the command line for an input value.

```

variable "subnet_cidrs" {
	type = list(string)
	default = ["10.10.2.0/24", "10.10.3.0/24"]
}

```

<br>

The **output** block is used for returning output to the caller from a resource block. This is important when writing and using modules within terraform. 

```

output "instance_ip_addr" {
  value = aws_instance.server.private_ip
}

```

<br>

The **data** block is used for fetching information about remote resources such as AMI ids or availability zones within a region.


```

data "aws_ami" "example" {
  most_recent = true

  owners = ["self"]
  tags = {
    Name   = "app-server"
    Tested = "true"
  }
}

```

