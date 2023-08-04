---
title: Terraform Study(T101) - Managing Secrets with Terraform
date: 2022-11-26 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## Secret Management Basics

배포 과정에서 민감 정보 (DB암호, API 키, TLS인증서, SSH키, GPG 키 등)를 안전하게 관리가 필요하며 절대 민감 정보를 평문으로 저장해서는 안됩니다.

테라폼에서 민감 정보가 노출되는 부분과 이를 어떻게 보완해야하는지 확인해봅니다.

**민감 정보의 종류**

- Personal secrets : 개인 소유의 암호, 예) 방문하는 웹 사이트 사용자 이름과 암호, SSH 키
- Customer secrets : 고객 소유의 암호, 예) 사이트를 운영하는데 고객의 사용자 이름과 암호, 고객의 개인정보 등 → 해싱 알고리즘 사용
- Infrastructure secrets : 인프라 관련 암호, 예) DB암호, API 키 등 → 암복호화 알고리즘 사용

**민감 정보의 저장 방법**

- File-based secret stores : 민감 정보를 암호화 후 저장 → 암호화 관련 키 관리는 AWS KMS, GCP KMS 혹은 PGP Key
- Centralized secret stores : 데이터베이스(MySQL, Psql, DynamoDB 등)에 비밀번호를 암호화하여 저장, 암호화 키는 서비스 자체 혹은 클라우드 KMS를 사용

<br>

## 테라폼에서의 위험 요소

### Provider Block에 인증 정보가 하드코딩 된 경우

- 예시

```hcl
provider "aws" {
	region = "ap-northeast-2"
	access_key = "AKIA***** ~ "
	secret_key = "3TaHkQ***** ~ "
```

- 해결 방법

  ⇒ 환경 변수를 사용한다. 

  ```bash
  export AWS_ACCESS_KEY_ID="AKIA***** ~ "
  export AWS_SECRET_ACCESS_KEY="3TaHkQ***** ~ "
  ```

  <br>⇒ 설정 파일을 사용한다. (+ aws-vault 툴 사용 - [링크](https://www.44bits.io/ko/post/securing-aws-credentials-with-aws-vault))

  ```bash
  $ aws configure list
        Name                    Value             Type                      Location
        ----                    -----             ----                      --------
     profile                <not set>             None                      None
  access_key                AKIA***** ~           shared-credentials-file    None
  secret_key                3TaHkQ***** ~         shared-credentials-file    None
      region           ap-northeast-2      config-file    ~/.aws/config
  ```

  <br>⇒ 임시 보안자격증명을 사용한다.

  - AWS가 주체가 되어 임시 보안자격증명 발급
  
    ![Untitled](/assets/img/posts/image-20221126230121620.png)
    _https://dev.classmethod.jp/articles/what-is-aws-iam-kr_

  - 외부 서비스(Github OIDC)가 주체가 되어 임시 보안자격증명 발급
  
    ![Untitled](/assets/img/posts/image-20221126230121621.png)
    _https://github.blog/2021-11-23-secure-deployments-openid-connect-github-actions-generally-available_

<br>

#### 실습(1) - EC2 Instance running Jenkins as a CI server, with IAM roles

Role(EC2FullAccess) Switch 를 통해, EC2를 사용할 수 있습니다.

<details markdown="1">
  <summary>코드 접기/펼치기</summary>

- code

  ```hcl
  # sg.tf
  resource "aws_security_group" "stg_mysg" {
    name        = "T101 SG"
    description = "T101 Study SG"
  }
  
  resource "aws_security_group_rule" "stg_mysginbound" {
    type              = "ingress"
    from_port         = 0
    to_port           = 22
    protocol          = "tcp"
    cidr_blocks       = ["0.0.0.0/0"]
    security_group_id = aws_security_group.stg_mysg.id
  }
  
  resource "aws_security_group_rule" "stg_mysgoutbound" {
    type              = "egress"
    from_port         = 0
    to_port           = 0
    protocol          = "-1"
    cidr_blocks       = ["0.0.0.0/0"]
    security_group_id = aws_security_group.stg_mysg.id
  }
  
  # iamec2.tf
  provider "aws" {
    region = "ap-northeast-2"
  	profile = "ljyoon"
  }
  
  resource "aws_instance" "example" {
    ami           = "ami-0eddbd81024d3fbdd"
    instance_type = "t2.micro"
    key_name      = "ljy_0208"
    vpc_security_group_ids      = ["${aws_security_group.stg_mysg.id}"]
  
    # Attach the instance profile
    iam_instance_profile = aws_iam_instance_profile.instance.name
  }
  
  # Create an IAM role
  resource "aws_iam_role" "instance" {
    name_prefix        = var.name
    assume_role_policy = data.aws_iam_policy_document.assume_role.json
  }
  
  # Allow the IAM role to be assumed by EC2 instances
  data "aws_iam_policy_document" "assume_role" {
    statement {
      effect  = "Allow"
      actions = ["sts:AssumeRole"]
  
      principals {
        type        = "Service"
        identifiers = ["ec2.amazonaws.com"]
      }
    }
  }
  
  # Attach the EC2 admin permissions to the IAM role
  resource "aws_iam_role_policy" "example" {
    role   = aws_iam_role.instance.id
    policy = data.aws_iam_policy_document.ec2_admin_permissions.json
  }
  
  # Create an IAM policy that grants EC2 admin permissions
  data "aws_iam_policy_document" "ec2_admin_permissions" {
    statement {
      effect    = "Allow"
      actions   = ["ec2:*"]
      resources = ["*"]
    }
  }
  
  # Create an instance profile with the IAM role attached
  resource "aws_iam_instance_profile" "instance" {
    role = aws_iam_role.instance.name
  }
  
  # outputs.tf
  output "instance_id" {
    value       = aws_instance.example.id
    description = "The ID of the EC2 instance"
  }
  
  output "instance_ip" {
    value       = aws_instance.example.public_ip
    description = "The public IP of the EC2 instance"
  }
  
  output "iam_role_arn" {
    value       = aws_iam_role.instance.arn
    description = "The ARN of the IAM role"
  }
  
  # variables.tf
  variable "name" {
    description = "The name used to namespace all the resources created by this module"
    type        = string
    default     = "ec2-iam-role-example"
  }
  ```

</details>

- 결과

  ![Untitled](/assets/img/posts/image-20221126230121622.png)

  생성된 Instance 접속 후 별도의 Accesskey 정보가 없음에도 IAM Role 확인 및 describe vpc 시 정상적으로 정보를 가져올 수 있었습니다.
  

<br>

#### 실습(2) - GitHub Actions as a CI server, with OIDC

github actions에서 S3 접근을 위한 OIDC 사용 실습으로 대체

<details markdown="1">
  <summary>코드 접기/펼치기</summary>

- code

  ```hcl
  # oidc.tf
  provider "aws" {
    region = "ap-northeast-2"
  	profile = "ljyoon"
  }
  
  resource "aws_iam_openid_connect_provider" "github_oidc" {
    url = "https://token.actions.githubusercontent.com"
  
    client_id_list = [
      "sts.amazonaws.com"
    ]
  
    thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
  }
  
  data "aws_iam_policy_document" "github_allow" {
    statement {
      effect  = "Allow"
      actions = ["sts:AssumeRoleWithWebIdentity"]
      principals {
        type        = "Federated"
        identifiers = [aws_iam_openid_connect_provider.github_oidc.arn]
      }
  
      condition {
        test     = "StringLike"
        variable = "token.actions.githubusercontent.com:sub"
        values   = ["repo:jjikin/github-actions-workflow:*"]
      }
      condition {
        test     = "StringEquals"
        variable = "token.actions.githubusercontent.com:aud"
        values   = ["sts.amazonaws.com"]
      }
    }
  }
  
  data "aws_iam_policy_document" "policy_bucket_list" {
    statement {
      actions = [
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation",
      ]
  
      resources = [
        "arn:aws:s3:::*",
      ]
    }
  }
  
  resource "aws_iam_role" "github_role" {
    name               = "GithubActionsRole"
    assume_role_policy = data.aws_iam_policy_document.github_allow.json
  }
  
  resource "aws_iam_role_policy" "github_policy" {
    role   = aws_iam_role.github_role.id
    policy = data.aws_iam_policy_document.policy_bucket_list.json
  }
  
  # gitub-action-workflow-s3_ls.yml
  name: aws-s3-ls
  
  on:
    push:
      branches: [ "main" ]
  
  jobs:
   build:
     name: build
     permissions:
       id-token: write
       contents: write
  
     runs-on: ubuntu-18.04
  
     steps:
       - name: Checkout
         uses: actions/checkout@v2
  
       - name: Configure AWS Credentials
         uses: aws-actions/configure-aws-credentials@master
         with:
           aws-region: ap-northeast-2
           role-to-assume: arn:aws:iam::************:role/GithubActionsRole
           role-session-name: GithubActionsSession
  
       - name: list bucket
         run: |
           aws s3 ls
  ```

</details>

- 결과

  ![Untitled](/assets/img/posts/image-20221126230121623.png)

<br>

### Resource 또는 Data Source block에 민감 정보가 하드코딩 된 경우(Database 암호 등)

- 예시

  ```hcl
  resource "aws_db_instance" "staging-rds" {
    identifier             = "staging-rds"
  	...
    db_name                = var.db_name
    username               = admin
    password               = password123!
  }
  ```

  <br>⇒ 환경 변수를 사용한다.

  ```bash
  $  export TF_VAR_db_username=(DB_USERNAME)
  $  export TF_VAR_db_password=(DB_PASSWORD)
  ```

  <br>⇒ 입력 변수를 사용한다. 

  ```hcl
  variable "db_username" {
    description = "The username for the database"
    type        = string
    sensitive   = true
  }
  
  variable "db_password" {
    description = "The password for the database"
    type        = string
    sensitive   = true
  }
  
  resource "aws_db_instance" "staging-rds" {
    identifier             = "staging-rds"
    ...
    db_name                = var.db_name
    username               = var.db_username
    password               = var.db_password
  }
  ```

  - 장점 : 모든 언어에서 환경 변수를 설정 및 쉽게 사용할 수 있으며 추가 비용이 들지 않음
  - 단점 : 변수의 수동 관리가 필요하며 여러 환경에서 사용할 경우 표준화 및 에러 발생 가능성이 존재

  <br>⇒ 클라우드 공급자(AWS KMS)를 통해 민감정보를 암호화할 키를 발급받아 사용한다.
  
  {: .prompt-info }
  
  > AWS KMS  
AWS Key Management Service(AWS KMS)는 데이터를 보호하는 데 사용하는 암호화 키를 쉽게 생성하고 제어할 수 있게 해주는 관리형 서비스입니다.

#### <br>실습(1) - Encrypted files with AWS KMS

AWS KMS 서비스를 통해 암호화 키를 생성하고 RDS 생성 시 활용 실습

- AWS KMS 실습

  ```bash
  # 키 생성
  CREATE_KEY_JSON=$(aws kms create-key --description "my text encrypt descript demo" --profile ljyoon)
  echo $CREATE_KEY_JSON | jq
  {
    "KeyMetadata": {
      "AWSAccountId": "************",
      **"KeyId": "eff54082-6ce7-4132-bdd3-************",**
      "Arn": "arn:aws:kms:ap-northeast-2:111111111111:key/eff54082-6ce7-4132-bdd3-************",
      "CreationDate": "2022-11-20T01:38:20.534000+09:00",
      "Enabled": true,
      "Description": "my text encrypt descript demo",
      "KeyUsage": "ENCRYPT_DECRYPT",
      "KeyState": "Enabled",
      "Origin": "AWS_KMS",
      "KeyManager": "CUSTOMER",
      "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT",
      "EncryptionAlgorithms": [
        "SYMMETRIC_DEFAULT"
      ],
      "MultiRegion": false
    }
  }
  
  # 키 Alias 설정
  export ALIAS_SUFFIX=jjikin
  aws kms create-alias --alias-name alias/$ALIAS_SUFFIX --target-key-id $KEY_ID --profile ljyoon
  
  ...
  {
    "AliasName": "alias/jjikin",
    "AliasArn": "arn:aws:kms:ap-northeast-2:111111111111:alias/jjikin",
    "TargetKeyId": "eff54082-6ce7-4132-bdd3-************",
    "CreationDate": "2022-11-20T01:38:20.534000+09:00",
    "LastUpdatedDate": "2022-11-20T01:38:20.534000+09:00"
  }
  
  # 평문 암호화
  echo "babo123!" > secret.txt
  aws kms encrypt --key-id alias/$ALIAS_SUFFIX --cli-binary-format raw-in-base64-out --plaintext file://secret.txt --profile ljyoon --output text --query CiphertextBlob | base64 --decode > secret.txt.encrypted
  
  # 암호문 복호화
  aws kms decrypt --ciphertext-blob fileb://secret.txt.encrypted --output text --query Plaintext --profile ljyoon | base64 --decode
  ```

  ![KMS Key로 평문을 암호화한 결과](/assets/img/posts/image-20221126230121624.png)
  _KMS Key로 평문을 암호화한 결과_

  ![KMS Key로 암호문을 복호화한 결과](/assets/img/posts/image-20221126230121625.png)
  _KMS Key로 암호문을 복호화한 결과_

  <br>

- RDS 생성 및 Key 활용 실습

    <details markdown="1">
    <summary>코드 접기/펼치기</summary>


  ```hcl
  # 민감 정보가 포함된 ~/week6/db-creds.yml 생성
  username: admin
  password: password
  
  # ~/week6/main.tf
  provider "aws" {
    region = "ap-northeast-2"
    profile = "ljyoon"
  }
  
  # Look up the details of the current user
  data "aws_caller_identity" "self" {}
  
  # 현재 사용자가 KMS 권한을 가질 수 있도록 정책 설정
  data "aws_iam_policy_document" "cmk_admin_policy" {
    statement {
      **effect    = "Allow"
      resources = ["*"]
      actions   = ["kms:*"]
      principals {
        type        = "AWS"
        identifiers = [data.aws_caller_identity.self.arn]**
      }
    }
  }
  
  resource "aws_kms_key" "cmk" {
    policy = data.aws_iam_policy_document.cmk_admin_policy.json
  }
  
  resource "aws_kms_alias" "cmk" {
    name          = "alias/jjikin2"
    target_key_id = aws_kms_key.cmk.id
  }
  ```

  ```bash
  # 암호화
  export ALIAS_SUFFIX=jjikin2
  aws kms encrypt --key-id alias/$ALIAS_SUFFIX --cli-binary-format raw-in-base64-out --plaintext file://db-creds.yml --profile ljyoon --output text --query CiphertextBlob | tee db-creds.yml.encrypted2
  
  # db-creds.yml 삭제
  rm db-creds.yml
  ```

  ```hcl
  # ~/week6/main.tf
  ...
  # 테라폼 코드에서 암호화된 파일 사용
  data "aws_kms_secrets" "creds" {
    secret {
      name    = "db"
      payload = file("${path.module}/db-creds.yml.encrypted2")
    }
  }
  
  locals {
    db_creds = yamldecode(data.aws_kms_secrets.creds.plaintext["db"])
  }
  
  resource "aws_db_subnet_group" "stg-db-sn-group" {
    name       = "stg-db-sn-group"
    subnet_ids = [aws_subnet.pri-a-sn.id, aws_subnet.pri-c-sn.id]
  
    tags = {
      Name = "stg-db-sn-group"
    }
  }
  
  resource "aws_security_group" "stg-rds-sg" {
    vpc_id      = aws_vpc.jjikin-vpc.id
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
  
  resource "aws_db_instance" "staging-rds" {
    identifier             = "staging-rds"
    engine                 = "mysql"
    allocated_storage      = 10
    instance_class         = "db.t2.micro"
    db_subnet_group_name   = aws_db_subnet_group.stg-db-sn-group.name
    vpc_security_group_ids = [aws_security_group.stg-rds-sg.id]
    skip_final_snapshot    = true
    db_name                = var.db_name
    **username               = local.db_creds.username
    password               = local.db_creds.password**
  }
  
  # 초기 생성 리소스
  resource "aws_vpc" "jjikin-vpc" {
    cidr_block       = "192.168.0.0/16"
    enable_dns_hostnames = true
    tags = {
      Name = "jjikin-stg-vpc" }
  }
  
  resource "aws_subnet" "pri-a-sn" {
    vpc_id     = aws_vpc.jjikin-vpc.id
    cidr_block = "192.168.30.0/24"
    availability_zone = "ap-northeast-2a"
    tags = {
      Name = "stg-pri-a-sn"
    }
  }
  
  resource "aws_subnet" "pri-c-sn" {
    vpc_id     = aws_vpc.jjikin-vpc.id
    cidr_block = "192.168.40.0/24"
    availability_zone = "ap-northeast-2c"
    tags = {
      Name = "stg-pri-c-sn"
    }
  }
  
  resource "aws_route_table" "pri-rt" {
    vpc_id = aws_vpc.jjikin-vpc.id
    tags = {
      Name = "stg-pri-rt"
    }
  }
  
  resource "aws_route_table_association" "pri-rt-a-asso" {
    subnet_id      = aws_subnet.pri-a-sn.id
    route_table_id = aws_route_table.pri-rt.id
  }
  
  resource "aws_route_table_association" "pri-rt-c-asso" {
    subnet_id      = aws_subnet.pri-c-sn.id
    route_table_id = aws_route_table.pri-rt.id
  }
  
  variable "db_name" {
    description = "The name to use for the database"
    type        = string
    default     = "stagingrds"
  }
  
  output "address" {
    value       = aws_db_instance.staging-rds.address
    description = "Connect to the database at this endpoint"
  }
  
  output "port" {
    value       = aws_db_instance.staging-rds.port
    description = "The port the database is listening on"
  }
  ```
    </details>

- 결과

  ![MasterUsername 항목이 ‘admin’으로 생성된 것을 확인할 수 있으며 이는 암호화된 파일 정보로 RDS가 생성되었음을 의미합니다.](/assets/img/posts/image-20221126230121626.png)

  MasterUsername 항목이 ‘admin’으로 생성된 것을 확인할 수 있으며 이는 암호화된 파일 정보로 RDS가 생성되었음을 의미합니다.

  - 장점 : 클라우드 KMS를 활용 가능
  - 단점 : 서비스에 대한 학습 필요, 키 암호화 복호화 과정이 필요하므로 자동화 환경에 적용 시 어려움이 존재, 감사 로그 취약 등
  
  <br>
  
- 리소스 삭제

  ```bash
  terraform destroy -auto-approve
  
  # 생성한 키 ID 변수 지정
  KEY_ID=$(echo $CREATE_KEY_JSON | jq -r ".KeyMetadata.KeyId")
  
  # 키 비활성화
  aws kms disable-key --key-id $KEY_ID
  
  # 키 삭제 예약 : 대기 기간(7일) 
  aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
  ```
  

<br>

#### 실습(2) - Secret Stores with AWS Secret Manager

3주차 [Terraform에서 민감 정보를 다루는 방법](https://jjikin.com/posts/Terraform-Study(T101)-Terraform에서-민감-정보를-다루는-방법/)에서 중앙 집중식 비밀저장소(AWS Secret Manager)를 이용한 민감 정보 관리 방법에 대해 실습하였습니다.

<br>

### State files 또는 Plan files에 민감 정보 노출

- **State files** : 3주차 ‘Terraform에서 민감 정보를 다루는 방법’에서 언급한 것 처럼, 테라폼 리소스와 데이터 소스에 전달되는 모든 **민감정보**는 테라폼 상태 파일에 **평문**으로 저장되므로 아래와 같이 Backend에 대한 접근제어가 필요합니다.
  - **백엔드 저장소에 저장 시 암호화** Store Terraform state in a backend that supports encryption
  - **백엔드 액세스에 대한 접근 통제** Strictly control who can access your Terraform backend

- **Plan files** : 암호화 및 파일 통제 필요

  ```bash
  # plan 출력 내용을 파일로 저장
  terraform plan -out=t101.plan
  #(참고) plan 출력 파일로 apply
  ## terraform apply t101.plan
  
  # plan 출력 내용을 파일 확인
  cat t101.plan
  
  # 평문
  terraform show t101.plan > t101.ansi
  less t101.ansi
  ```

<br>

### 참고 사항

**A comparison of methods for machine users (e.g., a CI server) to pass secrets to Terraform providers**

|                                     | Stored credentials | IAM roles                  | OIDC           |
| ----------------------------------- | ------------------ | -------------------------- | -------------- |
| Example                             | CircleCI           | Jenkins on an EC2 Instance | GitHub Actions |
| Avoid manually managing credentials | X                  | O                          | O              |
| Avoid using permanent credentials   | X                  | O                          | O              |
| Works inside of cloud provider      | X                  | O                          | X              |
| Works outside of cloud provider     | O                  | X                          | O              |
| Widely supported as of 2022         | O                  | O                          | X              |

**A comparison of methods for passing secrets to Terraform resources and data sources**

|                                            | Environment variables | Encrypted files | Centralized secret stores |
| ------------------------------------------ | --------------------- | --------------- | ------------------------- |
| Keeps plain-text secrets out of code       | O                     | O               | O                         |
| All secrets management defined as code     | X                     | O               | O                         |
| Audit log for access to encryption keys    | X                     | O               | O                         |
| Audit log for access to individual secrets | X                     | X               | O                         |
| Rotating or revoking secrets is easy       | X                     | X               | O                         |
| Standardizing secrets management is easy   | X                     | X               | O                         |
| Secrets are versioned with the code        | X                     | O               | X                         |
| Storing secrets is easy                    | O                     | X               | O                         |
| Retrieving secrets is easy                 | O                     | O               | X                         |
| Integrating with automated testing is easy | O                     | X               | X                         |
| Cost                                       | 0                     | $               | $$$                       |

<br>

참고

- 가시다님 스터디 자료
