---
layout: post
title:  "CICD Pipeline for a LAMP application on AWS ECS using Terraform & CodePipeline"
date:   2022-12-29 00:00:00 +0100
categories: terraform AWS ECS IaC
author: "Darren Foley"
description: "Post discussing how to deploy a lamp stack to AWS ECS using Terraform and CodePipeline"
tags: ["Lamp", "AWS", "ECS", "Terraform"]
hero_darken: true
---

In order to follow along with this tutorial it is assumed that you have the following installed for your operating system:

1. [Docker](https://docs.docker.com/get-docker/) - The ability to build and create docker files.
2. [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) - Commit and pull code from github or AWS CodeCommit. 
3. [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) - Open source version should be sufficent here. Make sure you are comfortable with using/finding terraform modules.

You will also need access to an AWS Account - The Free tier account should be sufficent. Make sure to install aws cli V2 [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). We will be taking advantage of a number of different services from AWS; CodeCommit, CodePipeline, ECR, ECS, S3 etc.


In the article I will outline how to deploy a containerized lamp application to AWS ECS Service using terraform. The application we are deploying is a sample ecommerce application created by KodeKloud. It can be found on github [here](https://github.com/kodekloudhub/learning-app-ecommerce). 

<br>

## Step-0: Local Directory Setup

**Clone source Code**

First clone the KodeCloud code down into your local file system.

```
> mkdir aws_project && cd aws_project
> git clone https://github.com/kodekloudhub/learning-app-ecommerce
```

**Create IAM User**

Since we are going to push code to AWS CodeCommit we will need to set up an IAM user and create a CodeCommit user under this IAM user. 

Once you have an IAM user created go to the security credentials tab and scroll down to "HTTPS Git credentials for AWS CodeCommit", click "Generate Credentials".

![Go to Security credentials Tab](/images/security_credentials_section.png)


Next create a code commit user and download the username and password key. We will need these to push/pull code from CodeCommit.

![Create a code Commit User](/images/code_commit_user.png)



**Set up Terraform**

From within the working directory of the project, create a directory to hold your terraform configuration


```
> cd learning-app-ecommerce
> mkdir tf
> cd tf
```

The file structure should look something like the below directory tree structure. We will create the various files as we go along. For now just create a main.tf file within the tf directory.

![Terraform Project Structure](/images/terraform_structure.png)

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

The S3 bucket "ecomm-terraform-state-df" will store the terraform backend state under the network/ directory. Ensure that your bucket has a different name as it must be globally unique. I'm using eu-west-1 as the primary region.

From the command line run:

```
> terraform init
```

This will initialize the backend and download the AWS provider to the .terraform directory.


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

Our database backend will depend on a mariaDB docker image with a few extra options. Create a separate directory called db with a Dockerfile and an init-script like so:

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

**Build images and push to ECR**


## Create CodeCommit Repository


## Create Terraform Configuration

**Create VPC Network**

**Create ECS Task definition for ECS Fargate**

**Create Load Balancer**

