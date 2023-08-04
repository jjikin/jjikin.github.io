---
title: Terraform Study(T101) - Terraform Variable
date: 2022-10-21 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## Terraform Variable 이란

Terraform은 HCL(HashiCorp Configuration Language)라는 개별 언어를 사용하며, 변수를 정의하고 주입해서 사용할 수 있습니다.

변수는 용도에 따라 입력 변수, 출력 변수로 구분됩니다.
입력 변수는 인프라를 구성하는 값을 정의하는 데 사용되며 출력 변수는 배포 후 인프라에 대한 정보를 얻는 데 사용됩니다.

<br>

## 입력 변수

### 선언 방법

입력 변수의 선언 방법과 사용할 수 있는 인수들은 아래와 같습니다.

> **variable “NAME” {
> [CONFIG …]
> }**

- **description** : 설명, 코드 내용 및 plan/apply 명령어를 실행할 때 출력됩니다.
- **default** : 변수 값을 전달하는 방법을 지정하지 않으면 기본값이 전달됨, 기본값이 없으면 대화식으로 사용자에게 변수에 대한 정보를 질의합니다.
  - 변수 값 전달 : 명령 줄(-var 옵션), 파일(-var-file 옵션), 환경 변수(테라폼은 이름이 ‘TF_VAR_<variable_name>’)
- **type** : 전달하려는 변수의 유형을 지정합니다. 유형을 지정하지 않으면 any로 지정됩니다.
      string number bool list map(key-value) set object tuple
- **sensitive** : 입력 변수가 사용될 때 출력 제한(암호 등 민감 데이터)
- **validation** : 변수 값에 사용자 지정 검사를 설정할 수 있습니다.

<br>

### 선언 예시

```bash
variable "string_example" {
  description = "An example of a string variable in Terraform"
  type        = string
  default     = "example"
}

variable "list_example" {
  description = "An example of a list in Terraform"
  type        = list
  default     = ["a", "b", "c"]
}

# 리스트 내 항목이 number인 list
variable "list_numeric_example" {
  description = "An example of a numeric list in Terraform"
  type        = list(number)
  default     = [1, 2, 3]
}

# object로 구조적 유형(structural type) 정의 가능
variable "object_example" {
  description = "An example of a structural type in Terraform"
  type        = object({
    name    = string
    age     = number
    tags    = list(string)
    enabled = bool
  })

  default = {
    name    = "value1"
    age     = 42
    tags    = ["a", "b", "c"]
    enabled = true
  }
}
```

<br>

### 입력 방법

- 변수 정의

  ```bash
  variable "webserver_port" {
    description = "The port the server will use for HTTP requests"
    type        = number
  }
  ```

- 입력 방법

  ```bash
  # 대화형 방식으로 입력
  terraform plan
  var.webserver_port
    The port the server will use for HTTP requests
  
    Enter a value: 
  
  # terraform 명령어에 -var 옵션으로 입력
  terraform plan -var "webserver_port=8080"
  
  # 환경변수를 통해 입력
  export TF_VAR_webserver_port=8080 && terraform plan
  
  # Default 값을 통해 입력
  variable "webserver_port" {
    description = "The port the server will use for HTTP requests"
    type        = number
    default     = 8080
  }
  ```

<br>

### 사용 방법

- var.<Variable_Name>

  ```bash
  resource "aws_security_group" "instance" {
    name = "terraform-example-instance"
  
    ingress {
      from_port   = var.server_port
      to_port     = var.server_port
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  ```

- ${var.<Variable_Name>}

  ```bash
  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p ${var.server_port} &
              EOF
  ```

  유저 데이터와 같이 문자열 내에서 참조가 필요한 경우 사용합니다.

<br>

## 출력 변수

출력 변수를 사용하면 배포 후 확인이 필요한 정보들을 쉽게 체크할 수 있습니다.

### 선언 방법

출력 변수의 선언 방법과 사용할 수 있는 인수들은 아래와 같습니다.

> **output “NAME” {
> [CONFIG …]
> }**

- NAME은 출력 변수의 이름이며, VALUE는 출력하려는 테라폼 표현식입니다.
- **description** : 출력 변수에 어떤 유형의 데이터가 포함되어있는지 출력됩니다.
- **sensitive** : 출력 변수 값이 암호 등 민감 데이터인 경우 출력을 제한합니다.
- **depends-on** : 출력 변수에 의존성이 존재하는 경우 사용하면 처리 순서를 보장할 수 있습니다.

### 선언 예시

```bash
output "public_ip" {
  value       = aws_instance.example.public_ip
  description = "The public IP address of the web server"
}
```

### 사용 방법

```bash
terraform apply
...
aws_instance.webserver: Creation complete after 21s [id=i-0e57686cd032fc0db]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:
public_ip = "13.125.215.173"

# 명령어를 통한 확인 방법
terraform output
...
public_ip = "13.125.215.173"
...

terraform output public_ip
"13.125.215.173"
```

<br>

## 포트 넘버를 입력받아 웹 서버 생성하기

- 코드 파일 작성 및 배포

  ```bash
  # main.tf
  data "aws_ssm_parameter" "amzn2_latest"  {
    name = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
  }
  
  resource "aws_instance" "webserver" {
    ami 			 = data.aws_ssm_parameter.amzn2_latest.value
    instance_type 	 = "t2.micro"
    vpc_security_group_ids = [aws_security_group.webserver.id]
  
    user_data = <<-EOF
      #!/bin/bash
      yum update -y && yum install httpd -y
      sudo sed -i "s/Listen 80/Listen ${var.webserver_port}/g" /etc/httpd/conf/httpd.conf
  	  echo "Hello, My name is Jiyoon. Port number is ${var.webserver_port}" > /var/www/html/index.html
      systemctl restart httpd
    EOF
  
    user_data_replace_on_change = true
  
    tags = {
      Name = "webserver1"
    }
  }
  
  resource "aws_security_group" "webserver" {
    name = var.security_group_name
  
    ingress {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
      from_port   = var.webserver_port
      to_port     = var.webserver_port
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    egress {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  ```

  ```bash
  # provider.tf
  provider "aws" {
    region = "ap-northeast-2"
    profile = "ljyoon"
  }
  
  # variable.tf
  variable "security_group_name" {
    description = "The security group name of webserver"
    type        = string
    default     = "webserver"
  }
  
  variable "webserver_port" {
    description = "The port the server will use for HTTP requests"
    type        = number
    default     = 5000
  }
  
  # output.tf
  output "public_ip" {
    value       = aws_instance.webserver.public_ip
    description = "The public IP of the Instance"
  }
  ```

- 실행 결과

  ![Untitled](/assets/img/posts/image-20221021175043547.png)

- 리소스 삭제

  ```bash
  terrafrom destroy && terraform init
  ```

<br>

참고

- [출력 변수](https://velog.io/@gentledev10/terraform-output-values)
- 가시다님 스터디 자료
