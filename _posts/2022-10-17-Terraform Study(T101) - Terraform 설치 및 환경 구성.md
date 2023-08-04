---
title: Terraform Study(T101) - Infrastructure As Code
date: 2022-10-17 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## Terraform 구성 요소

- provider : Infrastructure 공급자를 정의합니다. (AWS, GCP, Azure…)

  ```json
  provider "aws" {
    region = "ap-northeast-2"
    profile = "ljyoon"
  }
  ```

- resource : 생성할 인프라 자원을 정의합니다. (VPC, EC2, ELB…)

  ```json
  resource "aws_instance" "example" {
    ami           = "ami-0e9bfdb247cc8de84"
    instance_type = "t2.micro"
  ```

- output : 인프라 프로비저닝 `terraform apply` 후 생성된 리소스를 확인하기 위해 사용합니다.

  ```json
  output "public_ip" {
    value       = aws_instance.example.public_ip
    description = "The public IP of the Instance"
  }
  ```

- backend : Terraform의 상태를 저장하는 공간입니다. 정의하지 않으면 로컬에 저장됩니다.

  ```json
  backend "s3" {
      bucket = "tfstate-ljyoon"
      key    = "workspaces-default/terraform.tfstate"
      region = "ap-northeast-2"
      dynamodb_table = "terraform-locks-ljyoon"
    }
  ```

- module : 공통적으로 활용할 수 있는 코드를 모듈로 정의합니다.

  ```json
  module "webserver_cluster" {
    source = "../../../modules/services/webserver-cluster"
  }
  ```

- remote state : 다른 경로의 state를 참조합니다. backend에서 output 정보들을 가져올 때 사용합니다.

  ```json
  data "terraform_remote_state" "vpc" {
    backend = "remote" 
  
    config = { 
      organization = "hashicorp" 
      workspaces = { name = "vpc-prod" } 
    }
  }
  ```

- variable

- data

<br>

## Terraform 동작 방식

Terraform은 API(응용 프로그래밍 인터페이스)를 통해 클라우드 플랫폼 및 기타 서비스에서 리소스를 생성하고 관리합니다.

![Untitled](/assets/img/posts/image-20221017163033190.png)
_https://developer.hashicorp.com/terraform/tutorials/aws-get-started/infrastructure-as-code?in=terraform%2Faws-get-started_


1. Write : 서버, 로드밸런서 등과 같은 인프라 내 리소스를 정의합니다. `main.tf`
2. Initialize(초기화/준비) : Terraform에서 인프라를 관리하는 데 필요한 플러그인을 설치합니다.
   - 지정한 backend에 상태 저장을 위한 `terraform.tfstate` 파일이 생성됩니다.
   - local에는 `tfstate`에 정의된 내용을 담은 `.terraform` 가 생성됩니다.
   - 동시성 처리를 위한 `.terraform.lock.hcl` 파일도 생성됩니다.
3. Plan : 정의한 코드로 만들어질 인프라 내역에 대한 예측 결과를 미리 보여줍니다. 결과에 에러가 없다고 하더라도, Apply 시 에러가 발생할 수 있습니다.
4. Apply : 정의한 코드로 인프라를 배포합니다. Apply가 완료되면 AWS 상에 실제로 해당 인프라가 생성되고 작업 결과가 backend의 `.tfstate`, local의 `.terraform` 에도 저장됩니다.

<br>

## Terraform, AWS CLI 설치 및 환경 구성

```bash
# homebrew를 통한 설치
brew install terraform

# 버전 확인
terraform version
	Terraform v1.3.2

# terraform 명령어 자동완성 설정
terraform -install-autocomplete
cat ~/.zshrc
autoload -U +X bashcompinit && bashcompinit
complete -o nospace -C /usr/local/bin/terraform terraform

# homebrew를 통한 설치
brew install awscli

# profile 설정
cat .aws/credentials
	[ljyoon]
	aws_access_key_id = ****************
	aws_secret_access_key = ***********************************

cat .aws/config
	[profile ljyoon]
	region = ap-northeast-2
	output = json

# pager 비활성화 
# awscli 명령어 결과 출력 시 pager를 통해 편리하게 읽기 및 검색이 가능합니다. 하지만 단순히 결과 출력만 필요한 경우 불편하므로 비활성화합니다.
export AWS_PAGER=""
set AWS_PAGER=""

# 실습에 유용한 Tool 설치
brew install tree jq watch
```

- pager 활성화

  ![Untitled](/assets/img/posts/image-20221017163033191.png)

- pager 비활성화

  ![Untitled](/assets/img/posts/image-20221017163033192.png)

<br>참고

- [https://developer.hashicorp.com/terraform/intro](https://developer.hashicorp.com/terraform/intro)
- Terraform UP & Running
- 가시다님 스터디 자료
