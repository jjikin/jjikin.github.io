---
title: Terraform Study(T101) - VPC & Subnet 생성하기
date: 2022-10-23 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## VPC 생성

- 코드 파일 작성

  ```bash
  # vpc.tf
  provider "aws" {
    region  = "ap-northeast-2"
    profile = "ljyoon"
  }
  
  resource "aws_vpc" "ljyoon-vpc" {
    cidr_block       = "10.10.0.0/16"
  	enable_dns_support   = true
    enable_dns_hostnames = true
    tags = {
      Name = "ljyoon-vpc"
    }
  }
  ```

  - enable_dns_support : VPC가 Amazon에서 제공하는 DNS 서버를 통해 DNS 확인을 지원하는지 여부를 결정합니다.
  - enable_dns_hostnames : VPC가 퍼블릭 IP 주소가 있는 인스턴스에 퍼블릭 DNS 호스트 이름을 할당하도록 지원할 여부를 결정합니다.

- 배포 및 결과

  ```bash
  # 배포
  terraform init && terraform plan
  terraform apply -auto-approve
  
  # 결과
  terraform state list
  	aws_vpc.ljyoon-VPC
  
  aws ec2 describe-vpcs --profile ljyoon --output yaml
  	Vpcs:
  	- CidrBlock: 10.10.0.0/16
  	  CidrBlockAssociationSet:
  	  - AssociationId: vpc-cidr-assoc-08f56122961da0197
  	    CidrBlock: 10.10.0.0/16
  	    CidrBlockState:
  	      State: associated
  	  DhcpOptionsId: dopt-5fe58634
  	  InstanceTenancy: default
  	  IsDefault: false
  	  OwnerId: '*************'
  	  State: available
  	  Tags:
  	  - Key: Name
  	    Value: ljyoon-VPC
  		...
  ```

<br>

## Subnet 및 관련 리소스 생성

- 코드 파일 작성

  ```bash
  # vpc.tf 에 추가합니다.
  
  # 서브넷 생성
  resource "aws_subnet" "ljyoon-pub-a-sn" {
    vpc_id     = aws_vpc.ljyoon-vpc.id
    cidr_block = "10.10.1.0/24"
  
    availability_zone = "ap-northeast-2a"
  
    tags = {
      Name = "ljyoon-pub-a-sn"
    }
  }
  
  resource "aws_subnet" "ljyoon-pub-c-sn" {
    vpc_id     = aws_vpc.ljyoon-vpc.id
    cidr_block = "10.10.2.0/24"
  
    availability_zone = "ap-northeast-2c"
  
    tags = {
      Name = "ljyoon-pub-c-sn"
    }
  }
  
  # 인터넷 게이트웨이 생성
  resource "aws_internet_gateway" "ljyoon-igw" {
    vpc_id = aws_vpc.ljyoon-vpc.id
  
    tags = {
      Name = "ljyoon-igw"
    }
  }
  
  # 라우팅 테이블 생성
  resource "aws_route_table" "ljyoon-pub-rt" {
    vpc_id = aws_vpc.ljyoon-vpc.id
  
    tags = {
      Name = "ljyoon-pub-rt"
    }
  }
  
  # 라우팅 테이블에 서브넷 연결
  resource "aws_route_table_association" "ljyoon-pub-a-rt-association" {
    subnet_id      = aws_subnet.ljyoon-pub-a-sn.id
    route_table_id = aws_route_table.ljyoon-pub-rt.id
  }
  
  resource "aws_route_table_association" "ljyoon-pub-c-rt-association" {
    subnet_id      = aws_subnet.ljyoon-pub-c-sn.id
    route_table_id = aws_route_table.ljyoon-pub-rt.id
  }
  
  # 기본 라우팅 규칙 생성
  resource "aws_route" "ljyoon-pub-rt-rule" {
    route_table_id         = aws_route_table.ljyoon-pub-rt.id
    destination_cidr_block = "0.0.0.0/0"
    gateway_id             = aws_internet_gateway.ljyoon-igw.id
  }
  ```

- 배포 및 결과

  ```bash
  # 배포
  terraform init && terraform plan
  terraform apply -auto-approve
  
  # 결과
  terraform state list
  	aws_internet_gateway.ljyoon-igw
  	aws_route.ljyoon-pub-rt-rule
  	aws_route_table.ljyoon-pub-rt
  	aws_route_table_association.ljyoon-pub-a-rt-association
  	aws_route_table_association.ljyoon-pub-c-rt-association
  	aws_subnet.ljyoon-pub-a-sn
  	aws_subnet.ljyoon-pub-c-sn
  	aws_vpc.ljyoon-vpc
  ```

<br>

## Security Group 생성

- 코드 파일 작성

  ```bash
  # sg.tf
  resource "aws_security_group" "webserver-sg" {
    vpc_id      = aws_vpc.ljyoon-vpc.id
    name        = "webserver-sg"
    description = "T101 Study webserver-sg"
  }
  
  resource "aws_security_group_rule" "webserver-sg-inbound" {
    type              = "ingress"
    from_port         = 0
    to_port           = 80
    protocol          = "tcp"
    cidr_blocks       = ["0.0.0.0/0"]
    security_group_id = aws_security_group.webserver-sg.id
  }
  
  resource "aws_security_group_rule" "webserver-sg-outbound" {
    type              = "egress"
    from_port         = 0
    to_port           = 0
    protocol          = "-1"
    cidr_blocks       = ["0.0.0.0/0"]
    security_group_id = aws_security_group.webserver-sg.id
  }
  ```

- 배포 및 결과

  ```bash
  # 배포
  terraform plan
  terraform apply -auto-approve
  
  # 결과
  terraform state list
    ...
  	aws_security_group_rule.webserver-sg-inbound
  	aws_security_group_rule.webserver-sg-outbound
  	...
  ```

<br>

## 테스트용 Instance 생성

- 코드 파일 작성

  ```bash
  # ec2.tf
  data "aws_ami" "amazonlinux2" {
    most_recent = true
    filter {
      name   = "owner-alias"
      values = ["amazon"]
    }
  
    filter {
      name   = "name"
      values = ["amzn2-ami-hvm-*-x86_64-ebs"]
    }
  
    owners = ["amazon"]
  }
  
  resource "aws_instance" "webserver2" {
  
    depends_on = [
      aws_internet_gateway.ljyoon-igw
    ]
  
    ami                         = data.aws_ami.amazonlinux2.id
    associate_public_ip_address = true
    instance_type               = "t2.micro"
    vpc_security_group_ids      = [aws_security_group.webserver-sg.id]
    subnet_id                   = aws_subnet.ljyoon-pub-a-sn.id
  
    user_data = <<-EOF
                #!/bin/bash
                wget https://busybox.net/downloads/binaries/1.31.0-defconfig-multiarch-musl/busybox-x86_64
                mv busybox-x86_64 busybox
                chmod +x busybox
                RZAZ=$(curl http://169.254.169.254/latest/meta-data/placement/availability-zone-id)
                IID=$(curl 169.254.169.254/latest/meta-data/instance-id)
                LIP=$(curl 169.254.169.254/latest/meta-data/local-ipv4)
                echo "<h1>RegionAz($RZAZ) : Instance ID($IID) : Private IP($LIP) : Web Server</h1>" > index.html
                nohup ./busybox httpd -f -p 80 &
                EOF
  
    user_data_replace_on_change = true
  
    tags = {
      Name = "webserver2"
    }
  }
  
  output "webserver2_public_ip" {
    value       = aws_instance.webserver2.public_ip
    description = "The public IP of the Instance"
  }
  ```

- 배포 및 결과

  ```bash
  # 배포
  terraform plan
  terraform apply -auto-approve
  
  # 결과
  terraform state list
  	data.aws_ami.amazonlinux2
  	aws_instance.webserver2
  	...
  ```

  ![Untitled](/assets/img/posts/image-20221023175356605.png)

- Instance 삭제

```bash
rm -rf ec2.tf && terraform apply -auto-approve
```

<br>

참고

- [VPC의 DNS 속성](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/vpc-dns.html#vpc-dns-support)
- 가시다님 스터디 자료
