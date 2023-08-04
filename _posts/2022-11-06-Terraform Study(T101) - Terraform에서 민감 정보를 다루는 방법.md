---
title: Terraform Study(T101) - Terraform에서 민감 정보를 다루는 방법
date: 2022-11-06 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## Terraform Variable sensitive option

RDS에 접근하기 위해서는 사용자명과 비밀번호가 필요합니다. 일반적으로 사용자명과 비밀번호는 민감한 정보이며 외부에 공개되어서는 안됩니다. 

Terraform에서는 이를 위해 기본적으로 변수에 sensitive 옵션을 지원합니다.

이전 ‘Terraform 상태 파일을 격리하는 방법’ 포스팅에서 RDS 구성 시 아래 `variable.tf` 내 sensitive 값을 true로 설정했었습니다.

```json
# stage/db/variables.tf
# REQUIRED
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

# OPTIONAL
variable "db_name" {
  description = "The name to use for the database"
  type        = string
  default     = "tstudydb"
}
```

<br>하지만 sensitive 옵션은 테라폼 코드 내에서 해당 옵션이 적용된 변수를 참조하거나(${db.password}) 출력 할 때, plan 또는 apply 할 때 아래와 같이 값을 가려주는 역할만 제공합니다.

```bash
# sensitive = false
Error: Output refers to sensitive values

	on outputs.tf line 11:
	11: output "db_password" {

	To reduce the risk of accidentally exporting sensitive data that was intended to be only internal,
	Terraform requires that any root module output containing sensitive data be explicitly marked as
	sensitive, to confirm your intent.

	If you do intend to export this data, annotate the output value as sensitive by adding the
	following argument:
	sensitive = true

# sensitive = true
...
Changes to Outputs:
  + db__password = (sensitive value)
...
```

<br>.tfvars 파일을 통해 민감 정보들을 별도의 파일로 관리하는 방법도 있지만, 두가지 방법 모두 민감한 정보를 코드 내에 하드코딩하는 방식으로 의도치않게  외부로 유출될 위험이 존재합니다.

<br>

## AWS Secrets Manager를 이용한 민감 정보 관리

위 문제를 해결하기 위해 AWS에서는 Secrets Manager 서비스를 제공합니다.

{: .prompt-tip}

> Secrets Manager는 코드의 암호를 포함해 하드 코딩된 자격 증명을 Secrets Manager에서 프로그래밍 방식으로 보안 암호를 검색하도록 하는 API 호출로 바꿀 수 있습니다. 이렇게 하면 보안 암호가 코드에 더 이상 존재하지 않기 때문에 코드를 검사하는 누군가에 의해 보안 암호가 손상되지 않도록 방지할 수 있습니다. 
> 또한 사용자가 지정한 일정에 따라 Secrets Manager가 자동으로 보안 암호를 교체하도록 구성할 수 있습니다. 따라서 단기 보안 암호로 장기 보안 암호를 교체할 수 있어 손상 위험이 크게 줄어듭니다.

<br>다음은 AWS Secrets Manager를 사용하여 보안 암호(Secret)을 생성하는 코드입니다.

```hcl
# 랜덤 암호 생성
resource "random_password" "password" {
  length           = 10
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}
 
# 보안 암호 이름
resource "aws_secretsmanager_secret" "secret_db" {
   name = "secret_db"
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
```

- locals : local 변수는 선언된 코드 내에서만 사용되는 변수입니다. 주로 다른 변수와의 연산 결과를 하나의 변수로 만들어야 할 때 사용됩니다.

<br>생성한 보안암호를 이전 ‘Terraform 상태 파일을 격리하는 방법’ 포스팅 내 RDS 구성 코드에 추가합니다.

```hcl
# stage/db/mysql/main.tf
...
resource "aws_db_subnet_group" "db-sn-group" {
  name       = "db-sn-group"
  subnet_ids = [aws_subnet.pri-a-sn.id, aws_subnet.pri-c-sn.id]

  tags = {
    Name = "db-sn-group"
  }
}

### 추가 ###
# 랜덤 암호 생성
resource "random_password" "password" {
  length           = 10
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}
...
locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.creds.secret_string)
###########

resource "aws_db_instance" "staging-rds" {
  identifier             = "staging-rds"
  engine                 = "mysql"
  allocated_storage      = 10
  instance_class         = "db.t2.micro"
  db_subnet_group_name   = aws_db_subnet_group.db-sn-group.name
  vpc_security_group_ids = [aws_security_group.rds-sg.id]
  skip_final_snapshot    = true

  db_name                = var.db_name
  username               = local.db_creds.username
  password               = local.db_creds.password
}

# stage/db/variables.tf
# db_username, db_password 변수 삭제
variable "db_name" {
  description = "The name to use for the database"
  type        = string
  default     = "tstudydb"
}

# 배포 및 확인
terraform init && terraform plan 
terraform apply -auto-approve
```

<br>적용 후 코드나 별도의 파일에 민감한 정보를 하드코딩하지 않고 RDS에 접근할 수 있습니다.

![Untitled](/assets/img/posts/image-20221106203703240.png)

<br>

## tfstate에 저장된 민감한 정보의 암호화 유무 확인

RDS를 구성하는데 사용되었던 정보들은 S3 backend 내 상태파일에 저장됩니다. AWS Secrets Manager 서비스를 이용해 민감한 정보는 암호화하였으므로 상태 파일 내 정보도 암호화되어야 합니다.

S3에 저장되어있는  `tfstate` 파일을 직접 다운받아 확인해보면 AWS Secrets Manager로 민감한 정보를 암호화했음에도 모두 평문으로 저장되는 것을 확인할 수 있습니다.

![Untitled](/assets/img/posts/image-20221106203703241.png)

해당 원인은 Terraform의 특성 때문입니다. 구성을 마지막으로 적용한 이후 변경했는지 여부를 알 수 있도록 상태를 저장해야하는데 이 정보들이 암호화된다면 Terraform에서는 구성이 변경되는지 확인할 수 없습니다.

<br>이러한 한계를 대비하기 위해서는 backend에 사용되는 S3의 접근제어 설정이 권장됩니다.

- aws_s3_bucket_server_side_encryption_configuration 리소스의 사용

  ```hcl
  resource "aws_s3_bucket_server_side_encryption_configuration" "default" {
    bucket = aws_s3_bucket.terraform_state.id
  
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
  ```

- S3 버킷의 퍼블릭 엑세스 차단

  ```hcl
  resource "aws_s3_bucket_public_access_block" "public_access" {
    bucket                  = aws_s3_bucket.terraform_state.id
    block_public_acls       = true
    block_public_policy     = true
    ignore_public_acls      = true
    restrict_public_buckets = true
  }
  ```

<br>

참고

- Terraform Docsㅣ[secrets-vault](https://developer.hashicorp.com/terraform/tutorials/secrets/secrets-vault)
- Terraform Docsㅣ[Resource: random_passwordFunction](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- AWS Docsㅣ[AWS Secrets Manager](https://docs.aws.amazon.com/ko_kr/secretsmanager/latest/userguide/intro.html)
- [how-to-create-secrets-in-aws-secrets-manager-using-terraform-in-amazon-account - ](https://automateinfra.com/2021/03/24/how-to-create-secrets-in-aws-secrets-manager-using-terraform-in-amazon-account/)
- [how-to-manage-terraform-state](https://blog.gruntwork.io/how-to-manage-terraform-state-28f5697e68fa)
- 가시다님 스터디 자료
