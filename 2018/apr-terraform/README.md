<!-- $theme: gaia -->
<!-- template: invert -->

# Swarm with Terraform

---
# Table of contents
* What is the Terraform?
* iac 3 parts
  * server templating tool
  * configuration management tool
  * provisioning tool
    * Terraform is this position
* snipet ec2 + web service?
* swarm cluster

---
# What is the *Terraform*
Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.

* Infrastructure as Code
* written by GO
* HCL declarative language: `.tf`

---
# Getting started
```
$ brew install terraform
$ export AWS_ACCES_KEY_ID
$ export AWS_SECRET_ACCESS_KEY

# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "example" {
  ami           = "ami-40d28157"
  instance_type = "t2.micro"
}

$ terraform init
$ terraform plan
```

---
# graph 
```
terraform graph > out.dot
```

### local
```
brew install graphviz
dot -T png -O out.dot
```
### online
```
https://dreampuf.github.io/GraphvizOnline/
```

---
### `terraform.tfstate`
* s3: encrypt + version
* terraform remote config (automatically pull & push)

### 문제:
* 모든 팀원이 프로젝트마다 remote 셋팅을...
* 동시에 수정하게 될 경우.

### 해결:
* terraform pro, enterprise
* using build server
* terragrunt using DynamoDB
 

---
# 정리해야하는데... 손안뎀..

# Terraform 노트
* written by GO
* `.tf`: HCL declarative language

### `multiple cloud` with same syntax and tool
* AWS
* Azure
* Google Cloud
* Digital Ocean
* OpenStack

### `IAC` (Infrastructure as code)
* server
* database
* load balancers
* network topology

### configuration management tool
* Puppet
* Chef
* Ansible
* SaltStack

### provisioning tool
* Terraform
* CloudFormation
* OpenStack Heat

### server templating tool
* docker
* packer

# Trade-offs
* configuration management VS provisioning
* mutable infrastructure VS immutable
* procedual language VS declarative
* master VS masterless
* agent VS agentless
* large community VS small
* Mature VS cutting-edge



### mutable infrastructure
config change ex) update openssl...: update all exist servers
* hard to reproducable

management change ex) dev server/ live server
* live server accumulated months of changes
* dev servers are not reflected!!

### procedual vs declarative
> To make instances 10 to 15

#### procedual
* 10 instances
* 5 instances (depends on previous state)
* result 15 instances

#### declarative
* 10 instances
* 15 instances
* result 15 instances

코드의 관리 관점으로 본다면 declarative가 훨씬 나아보인다.


### community measure point
* issue: open, close, all, active change in month
* stars
* contributors
* #commit in close months
* libraries?
* stackoverflow
* jobs
* indeed.com mention


