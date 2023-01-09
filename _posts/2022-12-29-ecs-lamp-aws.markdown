---
layout: post
title:  "Blue/Green deployment of an ecommerce application to ECS using Terraform & Circleci"
date:   2022-12-29 00:00:00 +0100
categories: terraform AWS ECS IaC circleci
author: "Darren Foley"
description: "Post discussing how to deploy a PHP web application to AWS ECS using Terraform and Circleci"
tags: ["Docker", "AWS", "ECS", "Terraform", "blue/green", "Circleci", "CI/CD"]
hero_image: /images/sigmund-4CNNH2KEjhc-unsplash.jpg
hero_darken: true
---

In order to follow along with this tutorial it is assumed that you have the following installed for your operating system.

1. [Docker](https://docs.docker.com/get-docker/) - The ability to build and create docker files.

2. [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) - Commit and pull code from github.

3. [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) - The open source version should be sufficent here. Make sure you are comfortable with using/finding terraform modules.

4. **AWS Account** - You will also need access to an AWS Account - The Free tier account should be sufficent. Make sure to install aws cli V2 [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). We will be taking advantage of a number of different services from AWS; ECR, ECS, S3 etc.

5. **Circleci Account** - If you do not already have a Circleci account, setup a free account [here](https://circleci.com/docs/first-steps/). For open source projects, you can get 400,000 free credits for CI builds.

In this article I will outline how to deploy a containerized ecommerce application to AWS ECS Service using terraform. The application we are deploying is a sample ecommerce website created by KodeKloud. It can be found on github [here](https://github.com/kodekloudhub/learning-app-ecommerce).

Source code for the entire project can be found on my github [TheLinuxEnthusiast](https://github.com/TheLinuxEnthusiast/ecomm-aws-bluegreen).

<br>

## Step-0: Local Directory Setup

#### Clone source code

First clone the KodeCloud repostory to your local file system.

```
> mkdir aws_project && cd aws_project
> git clone https://github.com/kodekloudhub/learning-app-ecommerce
```

<br>

#### Create a new github repo for the project

Go to github and create a new repository to hold the new project. If you wish you can add your public SSH key to the project or setup a personal access token. 

Repoint git to this new repo using

```
git remote remove origin
git remote add origin <new github uri>
git add .
git commit -m "First commit"
git push origin master
```

Alternatively, you could simply fork the existing repository.

<br>

#### Create IAM User for Terraform

We are going use Terraform to spin up various resources for this project such as ECS services, task Definitions & networking resources. We will need to create an IAM user with the required permissions.

For testing purposes, you can give this user admin rights. For production, you would need the appropriate role based access control following the principle of least priviledge.

Run the "aws configure" command to add your access key and secret access key for the local aws cli. 

<br>

#### Set up Terraform

From within the working directory of the project, create a directory to hold your terraform configuration.


```
> cd learning-app-ecommerce
> mkdir tf
> cd tf
```

The file structure should look something like the below directory tree structure. We will create the various files as we go along. For now just create a main.tf file within the tf directory.

![Directory Structure](/images/Directory_structure.png)

Add the following code to the tf/main.tf file. 

```
terraform {

   backend "s3" {
     bucket = "ecomm-terraform-state-df"
     key    = "network/terraform.state"
     region = "eu-west-1"
   }
 
   required_providers {
     aws = {
       source  = "hashicorp/aws"
       version = "~> 4.0"
     }
   }
}
 
provider "aws" {
   region = "eu-west-1"
}

```

The S3 bucket "ecomm-terraform-state-df" will store the terraform backend state under the network/ directory. Ensure that your bucket has a different name as it must be globally unique. I'm using eu-west-1 as the primary region. The CI/CD pipeline will not be able to run without a remote backend in S3.

From the command line run:

```
> terraform init
```

This will initialize the backend and download the AWS provider into the .terraform directory.


<br>

## Step-1: Dockerize Application Components

In order to run this application code in ECS we must first create two docker container images.

1. Application Container which will run the front end web server.
2. Backend database running MariaDB which will persist product data for the application. 


Create a Dockerfile with the following Code. Change LABEL details as appropriate.

```
FROM ubuntu:20.04

LABEL version="1.0"
LABEL description="Simple PHP based ecommerce application container running on port 80"
LABEL author="Darren Foley"
LABEL email="darrenfoley015@gmail.com"

ENV DEBIAN_FRONTEND noninteractive
ENV DEBCONF_NONINTERACTIVE_SEEN true

# Resolve timezone information for apache2
RUN echo "tzdata tzdata/Areas select Europe" > /tmp/preseed.txt; \
    echo "tzdata tzdata/Zones/Europe select Dublin" >> /tmp/preseed.txt; \
    debconf-set-selections /tmp/preseed.txt && \
    apt update -y && \
    apt install -y tzdata

RUN apt install -y apache2 php php-mysql

RUN apt clean

# Change configuration for /etc/apache2/apache2.conf
RUN sed -i "/^<Directory \/var\/www\/>/a\\\tOrder allow,deny" /etc/apache2/apache2.conf && \
	sed -i "/^<Directory \/var\/www\/>/a\\\tallow from all" /etc/apache2/apache2.conf && \
	sed -i "/^<Directory \/var\/www\/>/a\\\tDirectoryIndex index.php index.html" /etc/apache2/apache2.conf && \
	sed -i "s/Options Indexes FollowSymLinks$/& MultiViews/" /etc/apache2/apache2.conf && \
	sed -i "175s/AllowOverride None/AllowOverride All/" /etc/apache2/apache2.conf


COPY . /var/www/html/


EXPOSE 80

RUN /sbin/service apache2 restart

CMD [ "apache2ctl", "-D", "FOREGROUND"]
```

Our database backend will depend on a mariaDB docker image with a few extra options. The MariaDB image can be bootstrapped with an SQL script by copying the script into the directory "/docker-entrypoint-initdb.d" of the container which will execute the script during build time. Create a separate directory called db with a Dockerfile and an init-script like so:

![DB Directory](/images/db_directory.png)


The db Dockerfile should look something like this. Provide root/database username & passwords as shown below. 

```
FROM mariadb:jammy

ENV MYSQL_USER=ecomuser \
    MYSQL_PASSWORD=ecompassword \
    MYSQL_DATABASE=ecomdb \
    MYSQL_ROOT_PASSWORD=12345

COPY ./init_script.sql /docker-entrypoint-initdb.d

EXPOSE 3306
```


Create an init script with the following SQL:

```
CREATE DATABASE IF NOT EXISTS ecomdb;

CREATE USER IF NOT EXISTS 'ecomuser'@'127.0.0.1' IDENTIFIED BY 'ecompassword';

GRANT ALL PRIVILEGES ON ecomdb.* TO 'ecomuser'@'127.0.0.1';

FLUSH PRIVILEGES;

##############

USE ecomdb;

CREATE TABLE products (id mediumint(8) unsigned NOT NULL auto_increment,Name varchar(255) default NULL,Price varchar(255) default NULL, ImageUrl varchar(255) default NULL,PRIMARY KEY (id)) AUTO_INCREMENT=1;

INSERT INTO products (Name,Price,ImageUrl) VALUES ("Laptop","100","c-1.png"),("Drone","200","c-2.png"),("VR","300","c-3.png"),("Tablet","50","c-5.png"),("Watch","90","c-6.png"),("Phone Covers","20","c-7.png"),("Phone","80","c-8.png"),("Laptop","150","c-4.png");

```

<br>

#### Build images and push to ECR

Create an ECR repository to hold our images. We will need two repositories one for the frontend web application and one for the mariadb backend.

![ECR Repositories](/images/ECR_repo.png)

Build the images locally first and push to ECR. You will need to point Docker to ECR and authenticate with AWS ECR. The push commands are as follows:

```
# Push commands for ECR
aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-ecr-repo-uri>
docker build -t ecomm-lamp-app .
docker tag ecomm-lamp-app:latest <your-ecr-repo-uri>/ecomm-lamp-app:latest
docker push  <your-ecr-repo-uri>/ecomm-lamp-app:latest

```

Repeat the same process for the mariadb backend container. 

```

cd db/
aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-ecr-repo-uri>
docker build -t mariadb .
docker tag mariadb:latest <your-ecr-repo-uri>/mariadb:latest
docker push  <your-ecr-repo-uri>/mariadb:latest

```

<br>

## Step-2: Create Terraform Configuration

In this section we will build our terraform configuration for blue green deployments. My project has the following structure:

![Terraform project structure](/images/Terraform_structure.png)

I created four modules; green, blue, ecs & load balancer plus one community module for creating networking resources. 

The AWS VPC module (see in ./tf/main.tf) is an existing community module created and maintained by hashicorp/aws. It is recommended that you use this for spinning up networking resources unless you have very specific requirements.

The load balancer is a custom module that spins up an Application load balancer, listener, security group and blue/green target groups. 

The ECS handles the creation of the cluster, task definition, cloud watch logging etc.

The blue and green modules contains the ECS services that we will toggle on/off depending on the feature toggle passed to the module in the code like so:

For more information on module structure and building custom modules see this article from [hashicorp](https://developer.hashicorp.com/terraform/language/modules/develop).


```
module "green" {
  source                = "./modules/green"
  count                 = var.is_green ? 1 : 0 # When "is_green" is true, count=1 and the resource is created, when count=0 resource is destroyed
  type                  = "green"
  prefix                = var.prefix
  suffix                = random_string.suffix.result
  ecomm_vpc_id          = module.vpc.vpc_id
  private_subnets       = module.vpc.private_subnets
  ecomm_app_group_green = module.load_balancer_config.ecomm_target_group_arn_green
  ecs_cluster_id        = module.ecs.ecs_cluster_id
  task_definition_id    = module.ecs.ecs_task_definition_id
  security_group_id     = module.ecs.security_group_id
  ecomm_alb_listener    = module.load_balancer_config.ecomm_alb_listener
  depends_on            = [module.ecs]
}
```

<br>

#### Create VPC Network


```
# Network for High Availability
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "${var.prefix}-vpc-${random_string.suffix.result}"
  cidr = var.ecomm_vpc_cidr

  azs             = var.azs
  private_subnets = var.ecomm_private_subnets
  public_subnets  = var.ecomm_public_subnets

  enable_nat_gateway = true
  enable_vpn_gateway = false

  tags = {
    Environment = "${terraform.workspace}"
    Name        = "${var.prefix}-application-${random_string.suffix.result}"
  }
}
```

<br> 

#### Create ECS Task definition for ECS Fargate

Within ./tf/modules/ecs/main.tf the task definition of the ECS Service is defined along with the ecs cluster, logging and security groups. I have embedded the task definition inline using the jsonencode function to encode the string as json. This allows me to dynamically pass the task definition URI's into the definition for greater flexibility. We are running onto of ECS Fargate and giving the entire application ~1G of memory and 512 units (.5 vCPU).


```
resource "aws_ecs_task_definition" "ecomm_app_task" {
  family                   = "ecomm-lamp-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  execution_role_arn       = data.aws_iam_role.exeution_role_arn_ecs.arn
  cpu                      = 512
  memory                   = 1024
  container_definitions = jsonencode([
    {
      "logConfiguration" : {
        "logDriver" : "awslogs",
        "options" : {
          "awslogs-group" : "/ecs/ecomm-application",
          "awslogs-region" : "eu-west-1",
          "awslogs-stream-prefix" : "ecs"
        }
      },
      "portMappings" : [
        {
          "hostPort" : 80,
          "protocol" : "tcp",
          "containerPort" : 80
        }
      ],
      "cpu" : 256,
      "memoryReservation" : 512,
      "image" : "${local.frontend_repo}",
      "essential" : true,
      "name" : "ecomm-lamp-app"
    },
    {
      "logConfiguration" : {
        "logDriver" : "awslogs",
        "options" : {
          "awslogs-group" : "/ecs/ecomm-application",
          "awslogs-region" : "eu-west-1",
          "awslogs-stream-prefix" : "ecs"
        }
      },
      "portMappings" : [
        {
          "hostPort" : 3306,
          "protocol" : "tcp",
          "containerPort" : 3306
        }
      ],
      "cpu" : 256,
      "memoryReservation" : 512,
      "image" : "${local.backend_repo}",
      "essential" : true,
      "name" : "ecomm-db"
    }
  ])
  tags = local.tags
}
```

<br>

#### Create Load Balancer

Within ./tf/modules/load_balancer/main.tf we define the alb, security groups, listeners and target groups. For simplicity I'll highlight the most important resources for blue/green are shown below. Within the listener we are doing a lookup to a map of traffic distribution values (traffic_dist_map). This allows us to dynamically change how much traffic is sent to each target group after a terraform apply. 


```
## within ./tf/modules/load_balancer/variables.tf
locals {
  traffic_dist_map = {
    blue = {
      blue  = 100
      green = 0
    }
    blue-90 = {
      blue  = 90
      green = 10
    }
    split = {
      blue  = 50
      green = 50
    }
    green-90 = {
      blue  = 10
      green = 90
    }
    green = {
      blue  = 0
      green = 100
    }
  }
}
```

```
## within ./tf/modules/load_balancer/main.tf
resource "aws_alb_target_group" "ecomm_app_group_green" {
  name        = "terraform-alb-target-green"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.ecomm_vpc_id
  target_type = "ip"

  tags = {
    Environment = "${terraform.workspace}"
    Name        = "${var.prefix}-green-${var.suffix}"
  }
}

resource "aws_alb_target_group" "ecomm_app_group_blue" {
  name        = "terraform-alb-target-blue"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.ecomm_vpc_id
  target_type = "ip"
  tags = {
    Environment = "${terraform.workspace}"
    Name        = "${var.prefix}-blue-${var.suffix}"
  }
}

resource "aws_alb_listener" "ecomm_listener_http" {
  load_balancer_arn = aws_alb.ecomm_alb.arn
  port              = "80"
  protocol          = "HTTP"

  #default_action {
  #  type             = "forward"
  #  target_group_arn = aws_lb_target_group.blue.arn
  #}
  default_action {
    type = "forward"

    forward {
      target_group {
        arn    = aws_alb_target_group.ecomm_app_group_blue.arn
        weight = lookup(local.traffic_dist_map[var.traffic_distribution], "blue", 100)
      }

      target_group {
        arn    = aws_alb_target_group.ecomm_app_group_green.arn
        weight = lookup(local.traffic_dist_map[var.traffic_distribution], "green", 0)
      }

      stickiness {
        enabled  = false
        duration = 1
      }
    }
  }

  tags = {
    Environment = "${terraform.workspace}"
    Name        = "${var.prefix}-${var.suffix}"
  }
}
```

<br>

#### Create Blue and green modules

For simplicity, I will show the green module configuration. Within ./tf/modules/green/main.tf we are creating an ECS Service which will run our task definition linked to a particular target group, in this case "ecomm_app_group_green".

```
resource "aws_ecs_service" "ecomm_service_green" {
  name             = "ecomm-lamp-app-${var.type}"
  cluster          = var.ecs_cluster_id     #aws_ecs_cluster.ecomm_cluster.id
  task_definition  = var.task_definition_id #aws_ecs_task_definition.ecomm_app_task.id
  desired_count    = 2
  launch_type      = "FARGATE"
  platform_version = "LATEST"

  network_configuration {
    security_groups = [var.security_group_id] #[aws_security_group.ecomm_sg.id]
    subnets         = var.private_subnets
  }

  load_balancer {
    target_group_arn = var.ecomm_app_group_green #var.alb_target_group_arn
    container_name   = "ecomm-lamp-app"
    container_port   = 80
  }

  depends_on = [var.ecomm_alb_listener]

}
```

<br>

## Step-3: Create CICD Build Pipeline

This is what the full pipeline should look like. 

![Full pipeline](/images/circleci_full_pipeline.png)

1. First we lint our dockerfile syntax using an open source tool called hadolint. You can find out more about the tool [here](https://hadolint.github.io/hadolint/).

2. We build and push our docker images to ECR.

3. Deploy a new green environment. This will be another ECS service running in parallel to the existing service.

4. Move 10% of the traffic over to the green environment. 90% stays on blue.

5. Run a smoke test to ensure traffic can still be served to the user.

6. Move 100% of traffic to green

7. Destroy old blue environment

<br>

#### Link the github repo with circleci

In order to use circleci in your project, you must allow circleci to follow your project. This will authorize circleci to view your existing repositories on your account should you wish to link them for CI builds. From the projects tab, click the blue "Set up project" button which will ask you to setup a basic project structure .circleci/config.yml file. Circleci has a tutorial on their website for setting up a basic project [here](https://circleci.com/docs/getting-started/).

![Circleci project](/images/circleci_project.png)

<br>

#### Add secrets to circleci

From the project tab click the ellipsis symbol (...) and click "project settings". On the right hand side menu click "Environmental Variables" and add the following to your project.

1. AWS_ACCESS_KEY_ID
2. AWS_DEFAULT_REGION
3. AWS_ECR_REGISTRY_ID
4. AWS_SECRET_ACCESS_KEY
5. DOCKERHUB_PASSWORD
6. DOCKERHUB_USERNAME
7. ECR_URI

![Circleci_env_var](/images/circleci_env_var.png)

<br>

#### CI/CD - Pipeline steps

1 - Lint Dockerfiles

```
lint_dockerfiles:
     docker:
      - image: circleci/python:3.9
     steps:
      - checkout
      - run: 
         name: "Install hadolint from source"
         command: |
            sudo wget -O /bin/hadolint https://github.com/hadolint/hadolint/releases/download/v1.16.3/hadolint-Linux-x86_64
            sudo chmod +x /bin/hadolint
      - run: 
         name: "Lint Dockerfiles"
         command: |
            /bin/hadolint ./Dockerfile
            /bin/hadolint ./db/Dockerfile
      - run:
         name: "On failure"
         command: |
            echo "Linting Dockerfiles has failed"
         when: on_fail
```

2 - Build and push our docker images to ECR.

In this section we are using a prebuilt aws orb for building/pushing to ECR.

```
orbs:
  aws-ecr: circleci/aws-ecr@8.2.1
```

From the workflows section we are adding this section.
```
- aws-ecr/build-and-push-image:
          name: "frontend_build"
          path: .
          aws-access-key-id: AWS_ACCESS_KEY_ID
          aws-secret-access-key: AWS_SECRET_ACCESS_KEY
          dockerfile: Dockerfile
          registry-id: AWS_ECR_REGISTRY_ID
          region: ${AWS_DEFAULT_REGION}
          repo: ecomm-lamp-app
          tag: "latest,${CIRCLE_SHA1:0:7}"
          requires:
            - lint_dockerfiles
          filters:
            branches:
              only: master
```

Same for the mariadb image.

```
- aws-ecr/build-and-push-image:
          name: "mariadb_build"
          path: ./db
          workspace-root: ./db
          aws-access-key-id: AWS_ACCESS_KEY_ID
          aws-secret-access-key: AWS_SECRET_ACCESS_KEY
          dockerfile: Dockerfile
          registry-id: AWS_ECR_REGISTRY_ID
          region: ${AWS_DEFAULT_REGION}
          repo: mariadb
          tag: "latest,${CIRCLE_SHA1:0:7}"
          requires:
            - lint_dockerfiles
          filters:
            branches:
              only: master
```

3 - Deploy a new green environment. 

The code checks the toggle.txt file (pulled from s3) which indicates whether we are currently toggled to blue or green target group.

```
  deploy_new_env:
    docker:
      - image: circleci/python:3.9
    steps:
      - checkout
      - install_terraform
      - copy_toggle_from_s3
      - run:
         name: "Initialize Terraform"
         working_directory: ~/project/tf
         command: |
           terraform init
      - run:
         name: "Deploy Green ECS Service"
         working_directory: ~/project/tf
         command: |
           export ENV_TOGGLE=$(cat ~/project/toggle.txt)
           echo "ENV TOGGLE is set to ${ENV_TOGGLE}"
           if [ "${ENV_TOGGLE}" = "blue" ];then
              terraform apply -var-file=variables/development.tfvars -var is_green="true" -auto-approve
           else
              terraform apply -var-file=variables/development.tfvars -var is_blue="true" -var is_green="true" -auto-approve
           fi
      - wait_60_sec
      - run:
         name: "Failed to Deploy new Environment"
         command: |
           echo "Failed to deploy new Environment"
         when: on_fail
      - rollback_deployment
```

4 - Move 10% of the traffic over to the green environment.

```
  apply_traffic_90_10:
    docker:
      - image: circleci/python:3.9
    steps:
      - checkout
      - install_terraform
      - copy_toggle_from_s3
      - run:
         name: "Initialize Terraform"
         working_directory: ~/project/tf
         command: |
           terraform init
      - run:
         name: "Move 10% of Traffic to new env"
         working_directory: ~/project/tf
         command: |
           export ENV_TOGGLE=$(cat ~/project/toggle.txt)
           echo "ENV TOGGLE is set to ${ENV_TOGGLE}"
           if [ "${ENV_TOGGLE}" = "blue" ]; then
               terraform apply -var-file=variables/development.tfvars -var is_green="true" -var traffic_distribution="blue-90" -auto-approve
           else
               terraform apply -var-file=variables/development.tfvars -var is_blue="true" -var is_green="true" -var traffic_distribution="green-90" -auto-approve
           fi
      - wait_60_sec
      - run:
         name: "Failed to redistribute traffic 90/10"
         command: |
            echo "Failed to redistribute traffic 90/10, rolling back to previous state"
         when: on_fail
      - rollback_deployment
```

5 - Run Smoke test

This runs a simple smoke test to check if we are still receiving traffic on port 80/http.

```
smoke_test:
    docker:
      - image: circleci/python:3.9
    steps:
      - checkout
      - install_terraform
      - run:
         name: "Initialize Terraform"
         working_directory: ~/project/tf
         command: |
           terraform init
      - copy_toggle_from_s3
      - smoke_test_curl ## Referenced below

```

The below is added in the commands section so it can be used a number of times.
```
smoke_test_curl:
    steps:
      - run:
         name: "curl endpoint and check if valid HTTP 200 is returned"
         working_directory: ~/project/tf
         command: |
          export HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $(terraform output -raw alb_dns_name))
         
          if [ "${HTTP_STATUS}" != "200" ];
          then
             exit 1
          fi
      - run:
         name: "Smoke test failed, start rollback"
         command: |
            echo "Smoke test failed, roll back to previous state"
         when: on_fail
      - rollback_deployment
```

6 - Move 100% of traffic to green

```
  apply_traffic_0_100:
    docker:
      - image: circleci/python:3.9
    steps:
      - checkout
      - install_terraform
      - copy_toggle_from_s3
      - run:
         name: "Initialize Terraform"
         working_directory: ~/project/tf
         command: |
            terraform init
      - run:
         name: "Move 100% of Traffic to new env"
         working_directory: ~/project/tf
         command: |
           export ENV_TOGGLE=$(cat ~/project/toggle.txt)
           echo "ENV TOGGLE is set to ${ENV_TOGGLE}"
           if [ "${ENV_TOGGLE}" = "blue" ]; then
              terraform apply -var-file=variables/development.tfvars -var is_green="true" -var traffic_distribution="green" -auto-approve
           else
              terraform apply -var-file=variables/development.tfvars -var is_blue="true" -var is_green="true" -var traffic_distribution="blue" -auto-approve
           fi
      - wait_60_sec
      - run:
         name: "Failed to redistribute traffic"
         command: |
            echo "Failed to redistribute traffic, rolling back to previous state"
         when: on_fail
      - rollback_deployment
```

7 - Destroy old environment

```
  destroy_old_env:
    docker:
      - image: circleci/python:3.9
    steps:
      - checkout
      - install_terraform
      - copy_toggle_from_s3
      - run:
         name: "Initialize Terraform"
         working_directory: ~/project/tf
         command: |
            terraform init
      - run:
         name: "Destroy old environment"
         working_directory: ~/project/tf
         command: |
           export ENV_TOGGLE=$(cat ~/project/toggle.txt)
           echo "ENV TOGGLE is set to ${ENV_TOGGLE}"
           if [ "${ENV_TOGGLE}" = "blue" ]; then
              terraform apply -var-file=variables/development.tfvars -var is_green="true" -var is_blue="false" -var traffic_distribution="green" -auto-approve
           else
              terraform apply -var-file=variables/development.tfvars -var is_green="false" -var is_blue="true" -var traffic_distribution="blue" -auto-approve
           fi
      - run:
         name: "Update toggle file"
         command: |
           export ENV_TOGGLE=$(cat ~/project/toggle.txt)
           echo "ENV TOGGLE is set to ${ENV_TOGGLE}"
           if [ "${ENV_TOGGLE}" = "blue" ]; then
              echo "green" > ~/project/toggle.txt
           else
              echo "blue" > ~/project/toggle.txt
           fi
      - copy_toggle_to_s3
      - run:
         name: "Failed to redistribute traffic"
         command: |
            echo "Failed to redistribute traffic, rolling back to previous state"
         when: on_fail
      - rollback_deployment
```

<br>

## Step-4: Run and Test pipeline

We can trigger our pipeline manually from the circleci UI or we can submit a pull request to the master branch. You should see the following from circleci.

![Successful Circleci build](/images/Successful_build.png)

<br>

When you go to the dns of the load balancer you should see the following.

![Successful Front end view]()
