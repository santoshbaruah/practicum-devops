## Objective of the Amazon linux2 EC2 instance is to have the RoR and PostgreSQL DB Using Teeraform

## Source Code File Details
- `main.tf` contains the beginning section of terraform code
- So we have to define `terraform` with `required_providers` and we have mentioned `aws` since we are going to create infra in AWS

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

# Configure the AWS Provider
$ export AWS_ACCESS_KEY_ID=""
$ export AWS_SECRET_ACCESS_KEY=""
```

- Rest of the `main.tf` should have the resource definition required for creating a `AWS EC2` instance
- We need to have below resources for creating an EC2 instance
  1. VPC
  2. Internet Gateway
  3. Subnet
  4. Route table
  5. Security Group
  6. EC2 instance definition

```
# Create a VPC
resource "aws_vpc" "app_vpc" {
  cidr_block = var.vpc_cidr

  tags = {
    Name = "app-vpc"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.app_vpc.id

  tags = {
    Name = "vpc_igw"
  }
}

resource "aws_subnet" "public_subnet" {
  vpc_id            = aws_vpc.app_vpc.id
  cidr_block        = var.public_subnet_cidr
  map_public_ip_on_launch = true
  availability_zone = "eu-central-1a"

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.app_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "public_rt"
  }
}

resource "aws_route_table_association" "public_rt_asso" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}


## Cloud Init and User Data
- Objective of the Amazon linux2 EC2 instance is to have the LNMP stack (Linux, Nginx, MySQL, PHP) installed on it, when the instance is created
- So we are providing a shell script in `user_data` section to install the LNMP
- The script added in `user_data` section will be invoked via `Cloud Init` functionality when the AWS server gets created

resource "aws_instance" "web" {
  ami             = "ami-0233214e13e500f77" 
  instance_type   = var.instance_type
  key_name        = var.instance_key
  subnet_id       = aws_subnet.public_subnet.id
  security_groups = [aws_security_group.sg.id]
```
user_data = <<-EOF
  #!/bin/bash
  echo "*** Install Ruby"
  sudo apt-get update
  sudo apt-get install -y curl gnupg build-essential
  sudo apt-get install gnupg2 -y
  curl -sSL https://get.rvm.io | bash -s stable — ruby=2.3.1p11
  sudo gpg2 — keyserver hkp://pool.sks-keyservers.net — recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
  curl -sSL https://get.rvm.io | sudo bash -s stable
  sudo usermod -a -G rvm ‘santoshbaruah’
  sudo apt install ruby -y
  rvm — default use ruby 2.3.1p11
  gem install bundler — no-rdoc — no-ri
  sudo apt-get install -y nodejs &&
> sudo ln -sf /usr/bin/nodejs /usr/local/bin/node
  sudo apt-get install -y dirmngr gnupg
  sudo apt-key adv — keyserver hkp://keyserver.ubuntu.com:80 — recv-keys 561F9B9CAC40B2F7
  sudo apt-get install -y apt-transport-https ca-certificates
  sudo sh -c ‘echo deb https://oss-binaries.phusionpassenger.com/apt/passenger bionic main > /etc/apt/sources.list.d/passenger.list’
  sudo apt-get update
  sudo apt-get install -y libnginx-mod-http-passenger
  sudo apt-get install -y libnginx-mod-http-passenger
  if [ ! -f /etc/nginx/modules-enabled/50-mod-http-passenger.conf ]; then sudo ln -s /usr/share/nginx/modules-available/mod-http-passenger.load /etc/nginx/modules-enabled/50-mod-http-passenger.conf ; fi
  sudo ls /etc/nginx/conf.d/mod-http-passenger.conf
  sudo apt install nginx-core
  sudo service nginx restart
  sudo apt-get update
  sudo apt-get install -y git
  sudo apt-get install mysql-server mysql-client
  sudo apt-get install libmysqlclient-dev
  curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
  echo “deb https://dl.yarnpkg.com/debian/ stable main” | sudo tee /etc/apt/sources.list.d/yarn.list
  sudo apt-get update && sudo apt-get install yarn
  cd /var/www
  sudo git clone [https://github.com/santoshbaruah/practicum-devops.git]
  sudo chown [santoshbaruah] -R [practicum-devops]
  cd [practicum-devops]
  vim config/master.key
  bundle install — deployment — without development test
  bundle exec rake db:create RAILS_ENV=production
  bundle exec rake assets:precompile RAILS_ENV=production
  chmod 700 config db
  chmod 600 config/database.yml config/master.key
  config.assets.js_compressor = Uglifier.new(harmony: true) 

```
  tags = {
    Name = "web_instance"
  }

  volume_tags = {
    Name = "web_instance"
  } 
}
```
- `variables.tf` file should have the customised variables, a user wanted to provide before running the infra creation
- User can also define default value for each variable in the file
resource "aws_security_group" "sg" {
  name        = "allow_ssh_http"
  description = "Allow ssh http inbound traffic"
  vpc_id      = aws_vpc.app_vpc.id

  ingress {
    description      = "SSH from VPC"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  ingress {
    description      = "HTTP from VPC"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "allow_ssh_http"
  }
}
```
- We can define `output.tf` file to see expected output values like `ipaddress` of instances and `hostname` etc.

- output.tf
```
output "web_instance_ip" {
    value = aws_instance.web.public_ip
}
```
- Since we have the custom variables defined in our terraform file, we have provide the values for those custom variables
- So we have to create a `tfvars` files and provide the custom variable values
- User has to provide the EC2 instance `pem file` key name in `instance_key` value
- aws.tfvars
```
region =  "eu-central-1"
instance_type = "t2.micro"
instance_key = "aws_ec2_pem_file_name"
creds = "~/.aws/credentials"
vpc_cidr = "178.0.0.0/16"
public_subnet_cidr = "178.0.10.0/24"
```
# RDS resources
#
resource "aws_db_instance" "postgresql" {
  allocated_storage               = var.allocated_storage
  engine                          = "postgres"
  engine_version                  = var.engine_version
  identifier                      = var.database_identifier
  snapshot_identifier             = var.snapshot_identifier
  instance_class                  = var.instance_type
  storage_type                    = var.storage_type
  iops                            = var.iops
  name                            = var.database_name
  password                        = var.database_password
  username                        = var.database_username
  backup_retention_period         = var.backup_retention_period
  backup_window                   = var.backup_window
  maintenance_window              = var.maintenance_window
  auto_minor_version_upgrade      = var.auto_minor_version_upgrade
  final_snapshot_identifier       = var.final_snapshot_identifier
  skip_final_snapshot             = var.skip_final_snapshot
  copy_tags_to_snapshot           = var.copy_tags_to_snapshot
  multi_az                        = var.multi_availability_zone
  port                            = var.database_port
  vpc_security_group_ids          = [aws_security_group.postgresql.id]
  db_subnet_group_name            = var.subnet_group
  parameter_group_name            = var.parameter_group
  storage_encrypted               = var.storage_encrypted
  monitoring_interval             = var.monitoring_interval
  monitoring_role_arn             = var.monitoring_interval > 0 ? aws_iam_role.enhanced_monitoring.arn : ""
  deletion_protection             = var.deletion_protection
  enabled_cloudwatch_logs_exports = var.cloudwatch_logs_exports

  tags = merge(
    {
      Name        = "DatabaseServer",
      Project     = var.project,
      Environment = var.environment
    },
    var.tags
  )
}


## Steps to run Terraform

terraform init
terraform plan -var-file=aws.tfvars
terraform apply -var-file=aws.tfvars -auto-approve

- Once the `terrform apply` completed successfully it will show the `public ipaddress` of the ROR , postgres as `output`

```
