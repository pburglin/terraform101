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

## Scenario 1 - create a single EC2 instance

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

2. Check what terraform will do before you apply it:
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

That's it! You can now create as many EC2 instances as you want via command line.  However, these EC2 instances don't do anything. Let's destroy it and check the next scenario below to learn how to get our EC2 instances setup as web servers.

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

## Scenario 2 - create the simplest web server

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

## Scenario 3 - introducing variables!

In Atom, edit file ec2instance.tf and replace it with this content:
```
provider "aws" {
  region = "us-east-1"
  access_key = "AKIA****************"
  secret_key = "9lII************************************"
}

variable "server_port" {
  description = "The port the server will use for HTTP requests"
  default = 8080
}

resource "aws_security_group" "instance" {
  name = "debug-webserver"
  ingress {
    from_port = "${var.server_port}"
    to_port = "${var.server_port}"
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
              nohup busybox httpd -f -p "${var.server_port}" &
              EOF

  tags {
    Name = "simple-webserver"
  }
}
```

**Important:** again, replace the access_key and secret_key with the credentials generated for your user **terraform**.

This time, we:
* Introduced a variable for our webserver port. If user provides no value, it defaults to 8080;

1. Review what terraform will create:
```
terraform plan -var server_port="8081"
```

2. Apply the changes:
```
terraform apply -var server_port="8081"
```

3. Log back to AWS console, go to EC2 and confirm you now have a new EC2 instance created by terraform.

Click on the new instance, and in the bottom right copy the Public IP address. Paste it into your browser, add ":8081" and confirm you can see the webserver's output "Hello world".

4. Discard the resources:
```
terraform destroy
```

## Scenario 4 - removing single point of failures

In Atom, edit file ec2instance.tf and replace it with this content:
```
provider "aws" {
  region = "us-east-1"
  access_key = "AKIA****************"
  secret_key = "9lII************************************"
}

variable "server_port" {
  description = "The port the server will use for HTTP requests"
  default = 8080
}

output "elb_dns_name" {
  value = "${aws_elb.webserver.dns_name}"
}

data "aws_availability_zones" "all" {}

resource "aws_elb" "webserver" {
  name = "simple-webserver"
  security_groups = ["${aws_security_group.elb.id}"]
  availability_zones = ["${data.aws_availability_zones.all.names}"]

  health_check {
    healthy_threshold = 2
    unhealthy_threshold = 2
    timeout = 3
    interval = 30
    target = "HTTP:${var.server_port}/"
  }

  listener {
    lb_port = 80
    lb_protocol = "http"
    instance_port = "${var.server_port}"
    instance_protocol = "http"
  }
}

resource "aws_security_group" "elb" {
  name = "simple-webserver-elb-sg"

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = "80"
    to_port = "80"
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_autoscaling_group" "webserver" {
  launch_configuration = "${aws_launch_configuration.webserver.id}"
  availability_zones = ["${data.aws_availability_zones.all.names}"]
  min_size = 2
  max_size = 5

  load_balancers = ["${aws_elb.webserver.name}"]
  health_check_type = "ELB"

  tag {
    key = "Name"
    value = "simple-webserver"
    propagate_at_launch = true
  }
}

resource "aws_launch_configuration" "webserver" {
  image_id = "ami-aa2ea6d0"
  instance_type = "t2.micro"
  security_groups = ["${aws_security_group.instance.id}"]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello world" > index.html
              nohup busybox httpd -f -p "${var.server_port}" &
              EOF

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group" "instance" {
  name = "simple-webserver-instance-sg"
  ingress {
    from_port = "${var.server_port}"
    to_port = "${var.server_port}"
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

**Important:** again, replace the access_key and secret_key with the credentials generated for your user **terraform**.

This time, we:
* Set a launch configuration for as many EC2 instances we need; The EC2 instances will all listed on port 8080
* Set an auto scaling group with min 2 EC2 instances and max of 5;
* Set a security group for our load balancer with ingress open on port 80;
* Set a load balancer on all availability zones forwarding traffic to port 8080 (or whatever you set for variable server_port);
* Set healthchecks in the load balancer to run every 30 seconds to remove any failing webserver from rotation;
* Ask Terraform to output at the end of the process the load balancer's dynamic DNS name;

1. Review what terraform will create:
```
terraform plan
```

2. Apply the changes:
```
terraform apply
...
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

elb_dns_name = simple-webserver-46581985.us-east-1.elb.amazonaws.com
```

Copy this output for "elb_dns_name", we will use it in just a bit.

3. Log back to AWS console, go to EC2 and confirm you now have two new EC2 instances created by terraform.

Click on Load Balancers, confirm we now have a "simple-webserver" load balancer running.

Scroll down and review "Port Configuration: 80 (HTTP) forwarding to 8080 (HTTP)".

4. In your browser, enter the DNS name you received from Terraform for "elb_dns_name". You should receive the response "Hello world".

Note: it might take couple minutes for the DNS to propagate. If it doesn't work in the first try, just give it a minute and try again.

5. Test failover and self-healing:

In the AWS console, go to EC2 / Instances and terminate one of the two webserver instances.

Once one of the instances go down, try hitting the "elb_dns_name" DNS again - it should work since we still have one remaining EC2 instance.

Now give it a minute, and then refresh the EC2 / Instances screen - you should get a new EC2 instance created automatically!

6. Discard the resources:
```
terraform destroy
```

That's it for this intro, you now know how to leverage Terraform to automate various manual tasks in the AWS console.
