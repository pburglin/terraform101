# Terraform 101
Intro to Terraform

# Description

Terraform is an open-source tool designed to enable infrastructure as code. It is similar to AWS CloudFormation, but supports multiple platforms and services.

# Pre-requirements

## 1. An AWS Account

If you still do not have an AWS account, create a free one:
* Go to the [AWS page](https://aws.amazon.com/)
* Review the details under "AWS Free Tier Details"
* Click the "Create a Free Account" button and follow the forms to create your account

NOTE: Concerned with unexpected bills for usage above the free tier? If you are careful using only free tier eligible resources and clean up after your tests this is typically not a problem. Still, you can always consider using a reloadable prepaid card instead of an actual credit card.

## 2. AWS CLI:

Follow the instructions [here](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) to install the AWS CLI and include it in your command line path.

## 3. Terraform

### Mac OS X
```
# install homebrew
ruby -e "$(curl -fsSL https://raw.github.com/mxcl/homebrew/go)"

# install terraform
brew install terraform
```

### Linux
```
wget https://releases.hashicorp.com/terraform/0.11.0/terraform_0.11.0_linux_amd64.zip
unzip terraform_0.11.0_linux_amd64.zip
sudo mv terraform /usr/bin/
```

### Other operating systems

You can find the latest version and packages for other operating systems here:
https://www.terraform.io/downloads.html

## 4. Atom + Terraform syntax checker

This step is optional, but useful when editing terraform files. Feel free to use an alternative text editor instead.
```
brew cask install atom
apm install language-terraform
```

## Validation:

```
aws --version
aws-cli/1.11.97 Python/2.7.14 Darwin/17.2.0 botocore/1.5.60

terraform version
Terraform v0.11.0
```

Note: if your environment has more recent versions of these components it should be okay, but this article was tested with the versions above.

# Lab

Review Terraform modules here: https://www.terraform.io/docs/modules/sources.html

## Setup terraform user in AWS

In AWS console, create a new user named **terraform** with appropriate access:

* AWS Console: https://console.aws.amazon.com/console/home
* Go to IAM / Users / Add User
* Create user named "terraform"
* Check box "Programmatic access"
* Attach existing policies directly, then select policy "AmazonEC2FullAccess"
* Download user credentials as CSV file

Example:
```
Username: terraform
Access key ID: AKIA****************
Secret access key: 9lII************************************
```

## Use case 1 - create a single EC2 instance

In Atom, create a new file ec2instance.tf with this content:
```
provider "aws" {
  region = "us-east-1"
  access_key = "AKIA****************"
  secret_key = "9lII************************************"
}

resource "aws_instance" "webserver" {
  ami = "ami-aa2ea6d0"
  instance_type = "t2.micro"
}
```

**Important:** replace the access_key and secret_key with the credentials generated for your user **terraform**.

AMI "ami-aa2ea6d0" refers to Ubuntu Server 16.04 LTS (HVM), SSD Volume Type.

Instance type t2.micro is eligible for AWS free tier, if you are using a free AWS account.

1. Initialize terraform:

```
terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "aws" (1.3.1)...
...
Terraform has been successfully initialized!
```

2. Check what terraform will do if you apply it:
```
terrafform plan

Refreshing Terraform state in-memory prior to plan...
...
Terraform will perform the following actions:

  + aws_instance.webserver
      ami:                          "ami-aa2ea6d0"
      instance_type:                "t2.micro"
...
Plan: 1 to add, 0 to change, 0 to destroy.
```

3. Apply changes:
```
terraform apply

Terraform will perform the following actions:
  + aws_instance.webserver
Plan: 1 to add, 0 to change, 0 to destroy.
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.
  Enter a value: yes
aws_instance.webserver: Creation complete after 22s (ID: i-0b69af13fcaafff19)
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

4. Log back to AWS console, go to EC2 and confirm you now have a new EC2 instance created by terraform.

That's it! You can now create as many EC2 instances as you want via command line.  However, these EC2 instances don't do anything. Let's destroy it and check the next use case below to learn how to get our EC2 instances setup as web servers.

5. Destroy changes:
```
terraform destroy

Terraform will perform the following actions:
  - aws_instance.webserver
Plan: 0 to add, 0 to change, 1 to destroy.
Do you really want to destroy?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.
  Enter a value: yes
aws_instance.webserver: Destruction complete after 1m11s
Destroy complete! Resources: 1 destroyed.
```

## Use case 2 - create the simplest web server

In Atom, edit file ec2instance.tf and replace it with this content:
```
provider "aws" {
  region = "us-east-1"
  access_key = "AKIA****************"
  secret_key = "9lII************************************"
}

resource "aws_security_group" "instance" {
  name = "debug-webserver"
  ingress {
    from_port = 8080
    to_port = 8080
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "webserver" {
  ami = "ami-aa2ea6d0"
  instance_type = "t2.micro"

  vpc_security_group_ids = ["${aws_security_group.instance.id}"]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello world" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  tags {
    Name = "simple-webserver"
  }
}
```

**Important:** again, replace the access_key and secret_key with the credentials generated for your user **terraform**.

This time, we:
* Set parameter "user_data" with commands executed as root as soon the EC2 instance starts. These commands start a lightweight webserver on port 8080 that responds to browser requests with the word "Hello world";
* Set a tag to name our EC2 instance as "simple-webserver";
* Set a security group to allow traffic to port 8080 to our webserver;

1. Review what terraform will create:
```
terraform plan
```

2. Apply the changes:
```
terraform apply
```

3. Log back to AWS console, go to EC2 and confirm you now have a new EC2 instance created by terraform.

Click on the new instance, and in the bottom right copy the Public IP address. Paste it into your browser, add ":8080" and confirm you can see the webserver's output "Hello world".

4. Discard the resources:
```
terraform destroy
```
