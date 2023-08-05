---
title: Terraform Study(T101) - Terraform Module
date: 2022-11-17 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## Terraform Module

모듈은 하나의 디렉토리 내 .tf파일로 구성된 파일 모음이며 여러 테라폼 리소스를 하나의 논리적 그룹으로 관리하는 방법입니다.

- 루트 모듈(Root Module)

  현재 작업 디렉터리 내 테라폼 코드 모음

- 차일드 모듈(Child Module)

  다른 모듈의 테라폼 코드 내에서 참조하기 위한 목적으로 작성된 코드 모음(대부분의 모듈)이며 이후로 언급하는 모듈은 차일드 모듈입니다.

<br>

## Terraform Module의 필요성

1. 중첩 루프(Nested Loop)의 해소

   여려개의 IAM 유저를 생성하는 경우, IAM 유저를 생성할 때마다 관련 리소스(Group, Policy 등)를 선언해줘야합니다.

   ```hcl
   resource "aws_iam_user" "jjikin" {
   	name = "jjikin"
   }
   
   resource "aws_iam_user_group_membership" "jjikin" {
   	user = aws_iam_user.jjikin.name
   	groups = ["admin"]
   }
   
   resource "aws_iam_user_policy_attachment" "jjikin_policy" {
   	user = aws_iam_user.jjikin.name
   	policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
   }
   
   resource "aws_iam_user" "jjikin2" {
   	name = "jjikin"
   }
   ...
   
   resource "aws_iam_user" "jjikin3" {
   	name = "jjikin"
   }
   ...
   (반복)
   ```

1. 캡슐화

   - 객체지향 프로그래밍의 핵심 개념중 하나로 객체의 응집도와 독립성을 높이기 위해 객체의 모듈화를 지향하는 것
   - 객체의 모듈화가 잘 이루어지면 모듈 단위의 재사용이 용이하므로 유지보수가 간편

<br>

## Local Values

모듈 내에서 사용할 수 있는 값으로 locals block을 통해 선언할 수 있으며, 동일한 값을 여러 번 사용하거나 값에 이름을 부여하여 관리하고 싶을 때 사용합니다.  
block 안에 다양한 type의 variables를 선언할 수 있으며 코드 내에서 local.`<`이름`>`으로 참조할 수 있습니다.

```hcl
# iamuser.tf
provider "aws" {
  region = "ap-northeast-2"
  profile = "ljyoon"
}

locals {
  name = "iamuser"
  team = {
    group = "dev"
  }
}

resource "aws_iam_user" "iamuser1" {
  name = "${local.name}1"
  tags = local.team
}

resource "aws_iam_user" "iamuser2" {
  name = "${local.name}2"
  tags = local.team
}

# check
terraform init && terraform plan && terraform apply -auto-approve
aws iam list-users | jq

# delete
terraform destroy -auto-approve
```

![Untitled](/assets/img/posts/image-20221117204201270.png)

![Untitled](/assets/img/posts/image-20221117204201271.png)

<br>

## 모듈 생성하기

3주차 ‘파일 레이아웃을 통한 격리’에서 사용한 레이아웃 중에서 stage/services/webserver-cluster 내 ASG, ALB, SG 등 리소스 코드를 재사용 가능한 모듈로 변경하여 staging, production 환경에서 모듈을 사용하도록 합니다.

**모듈이 적용되지 않은 stage 전체 코드 첨부**

  <details markdown="1">
  <summary>코드 접기/펼치기</summary>

  - stage/vpc

    ```hcl
    # main.tf
    provider "aws" {
      region  = "ap-northeast-2"
      profile = "ljyoon"
    } 
    
    terraform {
      backend "s3" {
        profile = "ljyoon"
        bucket  = "jjikin-tfstate-s3"
        key     = "stage/vpc/terraform.tfstate"
        region  = "ap-northeast-2"
        dynamodb_table = "tfstate-db-table"
      }
    }
    
    resource "aws_vpc" "jjikin-stg-vpc" {
      cidr_block       = "192.168.0.0/16"
      enable_dns_hostnames = true
      tags = {
        Name = "jjikin-stg-vpc" }
    }
    
    resource "aws_subnet" "stg-pub-a-sn" {
      vpc_id     = aws_vpc.jjikin-stg-vpc.id
      cidr_block = "192.168.10.0/24"
      availability_zone = "ap-northeast-2a"
      tags = {
        Name = "stg-pub-a-sn"
      }
    }
    
    resource "aws_subnet" "stg-pub-c-sn" {
      vpc_id     = aws_vpc.jjikin-stg-vpc.id
      cidr_block = "192.168.20.0/24"
    
      availability_zone = "ap-northeast-2c"
    
      tags = {
        Name = "stg-pub-c-sn"
      }
    }
    
    resource "aws_internet_gateway" "stg-igw" {
      vpc_id = aws_vpc.jjikin-stg-vpc.id
    
      tags = {
        Name = "stg-igw"
      }
    }
    
    resource "aws_route_table" "stg-pub-rt" {
      vpc_id = aws_vpc.jjikin-stg-vpc.id
    
      tags = {
        Name = "stg-pub-rt"
      }
    }
    
    resource "aws_route_table_association" "stg-pub-rt-a-asso" {
      subnet_id      = aws_subnet.stg-pub-a-sn.id
      route_table_id = aws_route_table.stg-pub-rt.id
    }
    
    resource "aws_route_table_association" "stg-pub-rt-c-asso" {
      subnet_id      = aws_subnet.stg-pub-c-sn.id
      route_table_id = aws_route_table.stg-pub-rt.id
    }
    
    resource "aws_route" "default-route" {
      route_table_id         = aws_route_table.stg-pub-rt.id
      destination_cidr_block = "0.0.0.0/0"
      gateway_id             = aws_internet_gateway.stg-igw.id
    }
    
    resource "aws_subnet" "stg-pri-a-sn" {
      vpc_id     = aws_vpc.jjikin-stg-vpc.id
      cidr_block = "192.168.30.0/24"
      availability_zone = "ap-northeast-2a"
      tags = {
        Name = "stg-pri-a-sn"
      }
    }
    
    resource "aws_subnet" "stg-pri-c-sn" {
      vpc_id     = aws_vpc.jjikin-stg-vpc.id
      cidr_block = "192.168.40.0/24"
      availability_zone = "ap-northeast-2c"
      tags = {
        Name = "stg-pri-c-sn"
      }
    }
    
    resource "aws_route_table" "stg-pri-rt" {
      vpc_id = aws_vpc.jjikin-stg-vpc.id
      tags = {
        Name = "stg-pri-rt"
      }
    }
    
    resource "aws_route_table_association" "stg-pri-rt-a-asso" {
      subnet_id      = aws_subnet.stg-pri-a-sn.id
      route_table_id = aws_route_table.stg-pri-rt.id
    }
    
    resource "aws_route_table_association" "stg-pri-rt-c-asso" {
      subnet_id      = aws_subnet.stg-pri-c-sn.id
      route_table_id = aws_route_table.stg-pri-rt.id
    }
    
    # outputs.tf
    output "stg-pub-a-sn" { 
      value = aws_subnet.stg-pub-a-sn.id
    }
    output "stg-pub-c-sn" { 
      value = aws_subnet.stg-pub-c-sn.id
    }
    
    output "stg-pri-a-sn" { 
      value = aws_subnet.stg-pri-a-sn.id
    }
    output "stg-pri-c-sn" { 
      value = aws_subnet.stg-pri-c-sn.id
    }
    
    output "stg-vpc-id" {
      value = aws_vpc.jjikin-stg-vpc.id
    }
    ```

  - stage/db

    ```hcl
    # main.tf
    provider "aws" {
      region  = "ap-northeast-2"
      profile = "ljyoon"
    } 
    
    terraform {
      backend "s3" {
        profile = "ljyoon"
        bucket = "jjikin-tfstate-s3"
        key    = "stage/db/mysql/terraform.tfstate"
        region = "ap-northeast-2"
        dynamodb_table = "tfstate-db-table"
      }
    }
    
    data "terraform_remote_state" "vpc" {
      backend = "s3"
      config = {
        profile = "ljyoon"
        bucket = "jjikin-tfstate-s3"
        key    = "stage/vpc/terraform.tfstate"
        region = "ap-northeast-2"
      }
    }
    
    resource "aws_db_subnet_group" "stg-db-sn-group" {
      name       = "stg-db-sn-group"
      subnet_ids = [data.terraform_remote_state.vpc.outputs.stg-pri-a-sn, data.terraform_remote_state.vpc.outputs.stg-pri-c-sn]
    
      tags = {
        Name = "stg-db-sn-group"
      }
    }
    
    resource "aws_security_group" "stg-rds-sg" {
      vpc_id      = data.terraform_remote_state.vpc.outputs.stg-vpc-id
      name        = "stg-rds-sg"
      description = "stg-rds-sg"
    }
    
    resource "aws_security_group_rule" "stg-rds-sg-inbound" {
      type              = "ingress"
      from_port         = 0
      to_port           = 3389
      protocol          = "tcp"
      cidr_blocks       = ["0.0.0.0/0"]
      security_group_id = aws_security_group.stg-rds-sg.id
    }
    
    resource "aws_security_group_rule" "stg-rds-sg-outbound" {
      type              = "egress"
      from_port         = 0
      to_port           = 0
      protocol          = "-1"
      cidr_blocks       = ["0.0.0.0/0"]
      security_group_id = aws_security_group.stg-rds-sg.id
    }
    
    # 랜덤 암호 생성
    resource "random_password" "password" {
      length           = 10
      special          = true
      override_special = "!#$%&*()-_=+[]{}<>:?"
    }
     
    # 보안 암호 이름
    resource "aws_secretsmanager_secret" "secret_db" {
       name = "secret_db_stg"
    }
     
    # 보안 암호 버전 설정
    resource "aws_secretsmanager_secret_version" "secret_version" {
      secret_id = aws_secretsmanager_secret.secret_db.id
      secret_string = <<EOF
       {
        "username": "cloudneta",
        "password": "${random_password.password.result}"
       }
    EOF
    }
     
    # 생성한 보안 암호의 arn 가져오기
    data "aws_secretsmanager_secret" "secret_db" {
      arn = aws_secretsmanager_secret.secret_db.arn
    }
    data "aws_secretsmanager_secret_version" "creds" {
      secret_id = data.aws_secretsmanager_secret.secret_db.arn
    }
     
    locals {
      db_creds = jsondecode(data.aws_secretsmanager_secret_version.creds.secret_string)
    }
    
    resource "aws_db_instance" "staging-rds" {
      identifier             = "staging-rds"
      engine                 = "mysql"
      allocated_storage      = 10
      instance_class         = "db.t2.micro"
      db_subnet_group_name   = aws_db_subnet_group.stg-db-sn-group.name
      vpc_security_group_ids = [aws_security_group.stg-rds-sg]
      skip_final_snapshot    = true
      db_name                = var.db_name
      username               = local.db_creds.username
      password               = local.db_creds.password
    }
    
    # outputs.tf
    output "address" {
      value       = aws_db_instance.staging-rds.address
      description = "Connect to the database at this endpoint"
    }
    
    output "port" {
      value       = aws_db_instance.staging-rds.port
      description = "The port the database is listening on"
    }
    
    # variables.tf
    variable "db_name" {
      description = "The name to use for the database"
      type        = string
      default     = "tstudydb"
    }
    ```

  - stage/service

    ```hcl
    # main.tf
    provider "aws" {
      region  = "ap-northeast-2"
      profile = "ljyoon"
    } 
    
    terraform {
      backend "s3" {
        profile = "ljyoon"
        bucket = "jjikin-tfstate-s3"
        key    = "stg/services/webserver-cluster/terraform.tfstate"
        region = "ap-northeast-2"
        dynamodb_table = "tfstate-db-table"
      }
    }
    
    # vpc tfstate 파일 참조
    data "terraform_remote_state" "vpc" {
      backend = "s3"
      config = {
        profile = "ljyoon"
        bucket = "jjikin-tfstate-s3"
        key    = "stage/vpc/terraform.tfstate"
        region = "ap-northeast-2"
      }
    }
    
    # db tfstate 파일 참조
    data "terraform_remote_state" "db" {
      backend = "s3"
      config = {
        profile = "ljyoon"
        bucket = "jjikin-tfstate-s3"
        key    = "stage/db/mysql/terraform.tfstate"
        region = "ap-northeast-2"
      }
    }
    
    resource "aws_security_group" "stg-web-sg" {
      vpc_id      = data.terraform_remote_state.vpc.outputs.stg-vpc-id
      name        = "stg-web-sg"
      description = "stg-web-sg"
    }
    
    resource "aws_security_group_rule" "stg-web-sg-inbound" {
      type              = "ingress"
      from_port         = 8080
      to_port           = 8080
      protocol          = "tcp"
      cidr_blocks       = ["0.0.0.0/0"]
      security_group_id = aws_security_group.stg-web-sg.id
    }
    
    resource "aws_security_group_rule" "stg-web-sg-outbound" {
      type              = "egress"
      from_port         = 0
      to_port           = 0
      protocol          = "-1"
      cidr_blocks       = ["0.0.0.0/0"]
      security_group_id = aws_security_group.stg-web-sg.id
    }
    
    data "template_file" "user_data" {
      template = file("${path.module}/user-data.sh")
    
      vars = {
        server_port = 8080
        db_address  = data.terraform_remote_state.db.outputs.address
        db_port     = data.terraform_remote_state.db.outputs.port
      }
    }
    
    data "aws_ami" "amazonlinux2" {
      most_recent = true
      filter {
        name   = "owner-alias"
        values = ["amazon"]
      }
    
      filter {
        name   = "name"
        values = ["amzn2-ami-hvm-*-x86_64-gp2"]
      }
    
      owners = ["amazon"]
    }
    
    resource "aws_launch_template" "stg-web-template" {
      name            = "stg-web-template"
      image_id        = data.aws_ami.amazonlinux2.id
      instance_type   = "t2.micro"
      network_interfaces {
        associate_public_ip_address = true
        security_groups = [aws_security_group.stg-web-sg.id]
      }  
    
      # Render the User Data script as a template
      user_data = base64encode(templatefile("${path.module}/user-data.sh", {
        server_port = 8080
        db_address  = data.terraform_remote_state.db.outputs.address
        db_port     = data.terraform_remote_state.db.outputs.port
      }))
    }            
    
    resource "aws_autoscaling_group" "stg-web-asg" {
      name                 = "stg-web-asg"
      vpc_zone_identifier  = [data.terraform_remote_state.vpc.outputs.stg-pub-a-sn, data.terraform_remote_state.vpc.outputs.stg-pub-c-sn]
      desired_capacity = 2
      min_size = 2
      max_size = 10
     
      # ALB 연결
      target_group_arns = [aws_lb_target_group.stg-web-alb-tg.arn]
      health_check_type = "ELB"
    
      launch_template {
        id      = aws_launch_template.stg-web-template.id
        version = "$Latest"
      }
    
      tag {
        key                 = "Name"
        value               = "stg-web-asg"
        propagate_at_launch = true
      }
    }
    
    # 기본사항 정의
    resource "aws_lb" "stg-web-alb" {
      name               = "stg-web-alb"
      load_balancer_type = "application"
      subnets            = [data.terraform_remote_state.vpc.outputs.stg-pub-a-sn, data.terraform_remote_state.vpc.outputs.stg-pub-c-sn]
      security_groups = [aws_security_group.stg-web-sg.id]
    
      tags = {
        Name = "stg-web-alb"
      }
    }
    
    # 리스너 정의
    resource "aws_lb_listener" "stg-web-http" {
      load_balancer_arn = aws_lb.stg-web-alb.arn
      port              = 8080
      protocol          = "HTTP"
    
      # By default, return a simple 404 page
      default_action {
        type = "fixed-response"
    
        fixed_response {
          content_type = "text/plain"
          message_body = "404: page not found - T101 Study"
          status_code  = 404
        }
      }
    }
    
    # 타겟그룹 정의
    resource "aws_lb_target_group" "stg-web-alb-tg" {
      name = "stg-web-alb-tg"
      port     = 8080
      protocol = "HTTP"
      vpc_id   = data.terraform_remote_state.vpc.outputs.stg-vpc-id
    
      health_check {
        path                = "/"
        protocol            = "HTTP"
        matcher             = "200-299"
        interval            = 5
        timeout             = 3
        healthy_threshold   = 2
        unhealthy_threshold = 2
      }
    }
    
    # 리스너 규칙 정의
    resource "aws_lb_listener_rule" "stg-web-alb-rule" {
      listener_arn = aws_lb_listener.stg-web-http.arn
      priority     = 100
    
      condition {
        path_pattern {
          values = ["*"]
        }
      }
    
      action {
        type             = "forward"
        target_group_arn = aws_lb_target_group.stg-web-alb-tg.arn
      }
    }
    
    output "stg-web-alb_dns" {
      value       = aws_lb.stg-web-alb.dns_name
      description = "The DNS Address of the ALB"
    }
    
    # user-data.sh
    #!/bin/bash
    wget https://busybox.net/downloads/binaries/1.31.0-defconfig-multiarch-musl/busybox-x86_64
    mv busybox-x86_64 busybox
    chmod +x busybox
    
    cat > index.html <<EOF
    <h1>T101 Study</h1>
    <p>My RDS DB address: ${db_address}</p>
    <p>My RDS DB port: ${db_port}</p>
    EOF
    nohup ./busybox httpd -f -p ${server_port} &
    ```
</details>

<br>

실습 환경을 아래와 같이 재구성하여 모듈을 생성할 예정입니다.

1. 3주차 리소스들을 별도 디렉터리(week4)에 모두 복사

2. stage/services/webserver-cluster 경로 내 tf 파일을 module/services/webserver-cluster로 이동

3. 각 환경의 루트 모듈(global, stage, prod)에서 provider와 state backend를 선언할 예정이므로 모듈 내 main.tf에서는 provider, backend 구문을 삭제합니다.

4. 아래 내용을 따라 모듈화를 적용합니다.

5. global → vpc → db → service 순으로 리소스를 apply 합니다.

    {: .prompt-warning }

    > db 리소스 생성 간 Secret Manager 보안 암호의 경우 이전 실습 시 생성 및 삭제했던 암호는 즉시 삭제되지 않으므로(기본 7일) 네이밍 변경 후 생성합니다. - [Link](https://aws.amazon.com/ko/premiumsupport/knowledge-center/delete-secrets-manager-secret)

6. File Tree

   ```bash
   . ~/week4
   ├── global
   │   └── s3
   │       ├── main.tf
   │       └── outputs.tf
   ├── modules
   │   └── services
   │       └── webserver-cluster
   │           ├── main.tf (파일 내 provider, backend 선언 삭제)
   │           ├── outputs.tf
   │           ├── user-data.sh
   │           └── variables.tf
   ├── stage
   │   ├── db
   │   │   └── mysql
   │   │       ├── main.tf
   │   │       ├── outputs.tf
   │   │       └── variables.tf
   │   ├── services
   │   │   └── webserver-cluster
   │   │       └── main.tf (생성 예정)
   │   └── vpc
   │       ├── main.tf
   │       └── outputs.tf
   ├── prod
       ...(stage 환경과 동일)
   ```

<br>

## 모듈 사용하기

- 사용 방법

  module block을 통해 사용가능 합니다.

  ```hcl
  module "<NAME>" {
    source = "<SOURCE>"
  
    [CONFIG ...] 
  }
  ```

- 적용 방법

  stage/services/webserver-cluster 경로에 새로운 main.tf를 작성합니다.

  ```hcl
  # ~/week4/stage/service/webserver-cluster/main.tf
  provider "aws" {
    region = "ap-northeast-2"
    profile = "ljyoon"
  }
  
  terraform {
    backend "s3" {
      profile = "ljyoon"
      bucket = "jjikin-tfstate-s3"
      key    = "stage/services/webserver-cluster/terraform.tfstate"
      region = "ap-northeast-2"
      dynamodb_table = "tfstate-db-table"
    }
  }
  
  module "webserver_cluster" {
    source = "../../../modules/services/webserver-cluster"
  }
  
  ```

- stage 환경 테스트

  모듈을 적용하거나 source 파라미터를 수정하는 경우 반드시 terraform init 실행이 필요합니다.

  ```bash
  terraform init
  Initializing modules...
  - webserver_cluster in ../../../modules/services/webserver-cluster
  
  Initializing the backend...
  Initializing provider plugins...
  
  terraform plan
  ```

  ![Untitled](/assets/img/posts/image-20221117204201272.png)

  {: .prompt-danger }

  > stage 환경에서 terraform plan & apply 시 user-data.sh 파일 경로와 관련된 에러가 발생합니다.

  3주차 ‘AutoScaling Group & ALB 생성하기’에서 사용한 것 처럼 테라폼에서는 내장 함수 file을 사용하여 상대 경로로 파일을 읽을 수 있었습니다.   
  하지만 file 함수는 terraform plan & apply를 실행되는 경로를 기준으로 파일을 읽을 수 있으므로 모듈과 같이 참조되는 경우에는 사용이 불가능합니다.

  <br>이 경우 경로 참조 표현식 ${path.module}을 사용하여 아래와 같이 코드를 수정 후 apply 합니다.

  ```hcl
  # .../modules/service/webserver-cluster/main.tf
  
  101 data "template_file" "user_data" {
  102   template = file("${path.module}/user-data.sh")
  103
  104   vars = {
  105     server_port = 8080
  106     db_address  = data.terraform_remote_state.db.outputs.address
  107     db_port     = data.terraform_remote_state.db.outputs.port
  108   }
  109 }
  ...
  135   # Render the User Data script as a template
  136   user_data = base64encode(templatefile("${path.module}/user-data.sh", {
  137     server_port = 8080
  138     db_address  = data.terraform_remote_state.db.outputs.address
  139     db_port     = data.terraform_remote_state.db.outputs.port
  140   }))
  141 }
  ...
  
  terraform plan && terraform apply -auto-approve
  ```

  ![Untitled](/assets/img/posts/image-20221117204201273.png)

  apply 및 서비스가 정상적으로 실행되었음을 확인할 수 있습니다.

  - 경로 참조 표현식
    - path.module : 표현식이 정의된 모듈의 파일 시스템 경로를 반환
    - path.root : 루트 모듈의 파일 시스템 경로를 반환
    - path.cwd : 현재 작업 중인 디렉터리의 파일 시스템 경로를 반환

<br>

- prod 환경 테스트

  main.tf 내 backend key를 prod로 변경 후 apply

  ```bash
  terraform {
    backend "s3" {
      profile = "ljyoon"
      bucket  = "jjikin-tfstate-s3"
      key     = "prod/vpc/terraform.tfstate"
      region  = "ap-northeast-2"
      dynamodb_table = "tfstate-db-table"
    }
  }
  ```

  services 리소스 apply 과정에서 아래와 같이 AWS 리소스 이름이 모듈 내에 하드코딩되어 있어 중복 에러가 발생합니다.

  ![Untitled](/assets/img/posts/image-20221117204201274.png)

  이를 위해서는 모듈 내에 입력 변수를 활용하여 stage/prod 간 환경을 구분하도록 설정해줘야 합니다.

<br>

### 모듈에서 입력 변수 사용하기

모듈에도 입력 변수(input variable) 사용이 가능합니다.

1. modules/services/webserver-cluster/variables.tf 에 새로운 입력 변수를 추가합니다.

     ```bash
     # modules/services/webserver-cluster/variables.tf
     variable "env" {
       description = "Variables for environment separation"
       type        = string
     }
     
     variable "instance_type" {
       description = "The type of EC2 Instances to run (e.g. t2.micro)"
       type        = string
     }
     
     variable "min_size" {
       description = "The minimum number of EC2 Instances in the ASG"
       type        = number
     }
     
     variable "max_size" {
       description = "The maximum number of EC2 Instances in the ASG"
       type        = number
     }
     ```

2. modules/services/webserver-cluster/main.tf 에 하드코딩된 AWS 리소스 이름을 확인하고, 위에서 선언한 변수로 대체합니다.

     - 사용 환경(env) 적용

       ```bash
       ...
       # vpc tfstate 파일 참조
       data "terraform_remote_state" "vpc" {
         backend = "s3"
         config = {
           profile = "ljyoon"
           bucket = "jjikin-tfstate-s3"
           key    = "${var.env}/stage/vpc/terraform.tfstate"
           region = "ap-northeast-2"
         }
       }
       
       # db tfstate 파일 참조
       data "terraform_remote_state" "db" {
         backend = "s3"
         config = {
           profile = "ljyoon"
           bucket = "jjikin-tfstate-s3"
           key    = "${var.env}/db/mysql/terraform.tfstate"
           region = "ap-northeast-2"
         }
       }
       
       resource "aws_security_group" "web-sg" {
         vpc_id      = data.terraform_remote_state.vpc.outputs.vpc-id
         name        = "${var.env}-web-sg"
         description = "${var.env}-web-sg"
       }
       ...
       
       resource "aws_launch_template" "-web-template" {
         name            = "${var.env}-web-template"
         image_id        = data.aws_ami.amazonlinux2.id
         instance_type   = "t2.micro"
         network_interfaces {
           associate_public_ip_address = true
           security_groups = [aws_security_group.web-sg.id]
         }  
       ...
       
       resource "aws_autoscaling_group" "web-asg" {
         name                 = "${var.env}-web-asg"
         vpc_zone_identifier  = [data.terraform_remote_state.vpc.outputs.pub-a-sn, data.terraform_remote_state.vpc.outputs.pub-c-sn]
         desired_capacity = 2
         min_size = var.min_size
         max_size = var.max_size
       
         # ALB 연결
         target_group_arns = [aws_lb_target_group.web-alb-tg.arn]
         health_check_type = "ELB"
       
         launch_template {
           id      = aws_launch_template.web-template.id
           version = "$Latest"
         }
       
         tag {
           key                 = "Name"
           value               = "${var.env}-web-asg"
           propagate_at_launch = true
         }
       }
       
       # ALB 정의
       resource "aws_lb" "web-alb" {
         name               = "${var.env}-web-alb"
         load_balancer_type = "application"
         subnets            = [data.terraform_remote_state.vpc.outputs.pub-a-sn, data.terraform_remote_state.vpc.outputs.pub-c-sn]
         security_groups = [aws_security_group.web-sg.id]
       
         tags = {
           Name = "${var.env}-web-alb"
         }
       }
       ...
       ```

     - 인스턴스 타입, min, max size 적용
       ```bash
       resource "aws_launch_template" "-web-template" {
         name            = "${var.env}-web-template"
         image_id        = data.aws_ami.amazonlinux2.id
         instance_type   = "${var.instance_type}"
         network_interfaces {
           associate_public_ip_address = true
           security_groups = [aws_security_group.web-sg.id]
         }  
       ```

3. stage/services/webserver-cluster/main.tf 에 새로운 입력 변수를 설정합니다.

    ```bash
    # ~/week4/stage/service/webserver-cluster/main.tf
    ...

    module "webserver_cluster" {
      source = "../../../modules/services/webserver-cluster"

      env = "stage"
      instance_type = "t2.micro"
      min_size = 1
      max_size = 1

    }
    ```
<br>

### 모듈에서 로컬 변수 사용하기

로컬 변수(Local Variable)는 코드를 보다 쉽게 읽기 유지 관리할 수 있도록 도와줍니다. 또한 모듈에서 로컬 값 사용 시 이름은 모듈 내에서만 표시되므로 다른 모듈에는 영향을 미치지 않고 모듈 외부에서 이 값을 재정의할 수 없습니다.
로컬 변수를 사용하기 위해서는 local.<NAME> 으로 참조합니다.

모듈에 로컬 변수를 정의하고 ALB 보안그룹에 하드코딩 된 값을 로컬 변수로 변경합니다.

```bash
# module/services/webserver-cluster/main.tf
...
locals {
  http_port    = 8080
  any_port     = 0
  any_protocol = "-1"
  tcp_protocol = "tcp"
  all_ips      = ["0.0.0.0/0"]
}

resource "aws_security_group" "web-sg" {
  vpc_id      = data.terraform_remote_state.vpc.outputs.vpc-id
  name        = "${var.env}-web-sg"
  description = "${var.env}-web-sg"
}

resource "aws_security_group_rule" "web-sg-inbound" {
  type              = "ingress"
  from_port         = local.http_port
  to_port           = local.http_port
  protocol          = local.tcp_protocol
  cidr_blocks       = local.all_ips
  security_group_id = aws_security_group.web-sg.id
}

resource "aws_security_group_rule" "web-sg-outbound" {
  type              = "egress"
  from_port         = local.any_port
  to_port           = local.any_port
  protocol          = local.any_protocol
  cidr_blocks       = local.all_ips
  security_group_id = aws_security_group.web-sg.id
}
```

<br>

### 모듈에서 출력 변수 사용하기

모듈에서도 output 값을 활용할 수 있습니다.

- 스테이징 환경에서 ASG 스케쥴 기반 증감 정책 활용 - 업무 시간 2대 증가 → 업무 시간 후 0대로 감소

  ```bash
  # module/services/webserver-cluster/outputs.tf
  output "asg_name" {
    value       = aws_autoscaling_group.example.name
    description = "The name of the Auto Scaling Group"
  }
  
  # stage/services/webserver-cluster/main.tf
  resource "aws_autoscaling_schedule" "scale_out_during_business_hours" {
    scheduled_action_name = "scale-out-during-business-hours"
    min_size              = 2
    max_size              = 2
    desired_capacity      = 2
    recurrence            = "0 9 * * *"
  
  	autoscaling_group_name = module.webserver_cluster.asg_name
  }
  
  resource "aws_autoscaling_schedule" "scale_in_at_night" {
    scheduled_action_name = "scale-in-at-night"
    min_size              = 0
    max_size              = 2
    desired_capacity      = 0
    recurrence            = "0 18 * * *"
  
  	autoscaling_group_name = module.webserver_cluster.asg_name
  }
  
  ```

- ALB의 DNS 출력

  ```bash
  # module/services/webserver-cluster/outputs.tf
  output "alb_dns_name" {
    value       = aws_lb.example.dns_name
    description = "The domain name of the load balancer"
  }
  
  # stage/services/webserver-cluster/outputs.tf
  output "alb_dns_name" {
    value       = module.webserver_cluster.alb_dns_name
    description = "The domain name of the load balancer"
  }
  ```

 <br>

참고

- HashiTalks: Koreaㅣ[확장 가능한 테라폼 코드 관리(박병진님)](https://www.youtube.com/watch?v=yWhwZpzJ3no&t=2538s)
- 가시다님 스터디 자료
