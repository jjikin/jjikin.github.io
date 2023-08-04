---
title: Terraform Study(T101) - Terraform 상태 관리하기
date: 2022-10-29 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## Terraform State 란

Terraform은 인프라 구성에 대한 상태를 파일 형태로 저장해야 합니다. 상태파일은 코드로 정의한 리소스가 실제 리소스로 매핑될 수 있도록 JSON 형식으로 기록됩니다. 

상태 파일은 배포할 때마다 변경되는 프라이빗 API로, 오직 테라폼 내부에서 사용하기 위한 것입니다.
따라서 테라폼 상태 파일을 직접 편집하는 것은 권장되지 않으며`terraform import` 또는 `terraform state` 명령어를 사용하여 관리해야 합니다.

<br>

### 팀 단위로 테라폼을 사용하는 경우

상태는 기본적으로 "terraform.tfstate"라는 로컬 파일에 저장되지만 팀 단위로 테라폼을 운영하는 경우 여러 문제점들이 발생합니다.

1. 로컬 파일로는 각 팀원들 간 동일한 상태를 유지하기 어렵습니다.
2. 두 팀원이 동시에 `apply` 시 상태 파일이 동시에 업데이트되어 충돌이 발생할 수 있습니다.
3. 여러 환경을 구성하여 사용하는 경우 (dev/staging/production) 각 환경에 대한 격리가 필요합니다.

따라서 팀 단위로 상태파일을 공유할 수 있는 저장소가 필요합니다.

<br>

### 상태 파일 공유를 위해 버전 관리 시스템을 사용하는 경우(Git 등)

1. 수동 오류 : `apply` 전 최신 변경 사항을 가져오거나 `apply` 후 push가 누락될 수 있습니다.
   - 팀의 누군가가 이전 버전의 상태 파일로 테라폼을 실행하고, 그 결과 실수로 이전 버전으로 롤백하거나 이전에 배포된 인프라를 복제하는 문제가 발생 할 수 있음.
2. 잠금 기능의 부재
   - 대부분의 버전 관리 시스템은 여러 명의 팀 구성원이 동시에 하나의 상태 파일에 `terraform apply` 명령을 실행하지 못하게 하는 잠금 기능이 제공되지 않음.
3. 민감 정보의 노출 가능성
   - 테라폼 상태 파일의 모든 데이터는 평문으로 저장되므로 개인정보와 같은 중요 데이터를 저장하는 경우 노출될 위험이 존재합니다.

위 3가지 문제들을 해결하기 위해서는 **Terraform에서 제공하는 원격 backend 기능 사용이 권장**됩니다.

<br>

## AWS S3 + DynamoDB를 활용한 원격 Backend 구성

스터디에서는 AWS S3 + DynamoDB를 사용합니다.

![Untitled](/assets/img/posts/image-20221029193904170.png)
_https://github.com/binbashar/terraform-aws-tfstate-backend_

- S3 Bucket을 생성하는 코드 작성

  ```bash
  mkdir tfstate-backend && cd tfstate-backend
  
  # backend.tf
  provider "aws" {
    region = "ap-northeast-2"
    profile = "ljyoon"
  }
  
  resource "aws_s3_bucket" "s3-backend" {
    bucket = "jjikin-t101study-tfstate"
  }
  
  # 버전 관리 활성화
  resource "aws_s3_bucket_versioning" "s3-backend_versioning" {
    bucket = aws_s3_bucket.s3-backend.id
    versioning_configuration {
      status = "Enabled"
    }
  }
  
  output "s3_bucket_arn" {
    value       = aws_s3_bucket.s3-backend.arn
    description = "The ARN of the S3 bucket"
  }
  ```

- 배포 및 확인

  ```bash
  terraform init && terraform plan 
  terraform apply -auto-approve
  
  terraform state list
  	aws_s3_bucket.s3-backend
  	aws_s3_bucket_versioning.s3-backend_versioning
  	
  # S3 버킷 확인
  aws s3 ls --profile ljyoon
  	2022-10-29 19:27:55 jjikin-t101study-tfstate
  	...
  ```

- 잠금 기능 사용을 위해 기본 키(LockID)가 존재하는 DynamoDB Table 생성 코드 추가

  ```bash
  # backkend.tf
  ...
  resource "aws_dynamodb_table" "dynamodbtable-backend" {
    name         = "dynamodbtable-backend"
    billing_mode = "PAY_PER_REQUEST"
    hash_key     = "LockID"
  
    attribute {
      name = "LockID"
      type = "S"
    }
  }
  ...
  output "dynamodb_table_name" {
    value       = aws_dynamodb_table.dynamodbtable-backend.name
    description = "The name of the DynamoDB table"
  }
  ```

- 배포 및 확인

  ```bash
  terraform init && terraform plan 
  terraform apply -auto-approve
  
  terraform state list
  	aws_dynamodb_table.dynamodbtable-backend
  	...
  	
  # DynamoDB Table 생성 확인
  export AWS_PAGER=""
  aws dynamodb list-tables --profile ljyoon --output text
  	TABLENAMES	dynamodbtable-backend
  
  aws dynamodb describe-table --table-name dynamodbtable-backend --profile ljyoon --output table
  ```

<br>

## Dev 환경 구성

- 환경 구성(VPC + SG + EC2)은 [이전 포스팅](https://jjikin.com/posts/Terraform-Study(T101)-VPC-&-Subnet-생성하기)에서 다뤘던 내용 및 코드를 참고합니다.

- dev 환경에 backend 구성을 위한 코드 파일 작성

  ```bash
  mkdir dev && cd dev 
  
  # dev_backend.tf
  terraform {
    backend "s3" {
      profile = "ljyoon"
      bucket  = "jjikin-t101study-tfstate"
      key     = "dev/terraform.tfstate"
      region  = "ap-northeast-2"
      dynamodb_table = "dynamodbtable-backend"
      #encrypt = true
    }
  }
  ```

- 배포(dev 환경에 backend 적용) 및 확인

  ![Untitled](/assets/img/posts/image-20221029193904171.png)

  ![`terraform.tfstate.backup` 파일이 생성됨](/assets/img/posts/image-20221029193904172.png)
  _`terraform.tfstate.backup` 파일이 생성됨_

  ![S3 Bucket 내 tfstate 생성 확인](/assets/img/posts/image-20221029193904173.png)
  _S3 Bucket 내 tfstate 생성 확인_

![dynamoDB 내 LockID 생성 확인](/assets/img/posts/image-20221029193904174.png)
  _dynamoDB 내 LockID 생성 확인_

<br>

## Terraform Backend 테스트

- 테라폼의 backend 블록에는 변수나 참조를 사용 할 수 없음 → 아래 코드 사용할 수 없음

  ![Untitled](/assets/img/posts/image-20221029193904175.png)

- 그 결과 S3 버킷 이름, 리전, DynamoDB 테이블 이름을 모두 테라폼 모듈에 수동으로 복사붙여녛어야 함, 심지어 **key** 값은 중복되면 안되며 고유하게 넣어야함

- [**partial configuration**](https://developer.hashicorp.com/terraform/language/settings/backends/configuration#partial-configuration)을 통해 일부 매개 변수를 전달해서 사용 할 수 있음

  ![Untitled](/assets/img/posts/image-20221029193904176.png)

- 다만, 이 경우에도 모듈마다 서로 다른 key 값을 설정해야 하기 때문에 key 매개 변수는 테라폼 코드에 있어야함

  ![Untitled](/assets/img/posts/image-20221029193904177.png)

<br>

### 1. Instance의 태그를 변경 및 배포 후 dynamoDB Table 확인하기

```bash
sed -i -e 's/HallsHolicker-jjang/gasida-jjangg/g' dev_ec2.tf

terraform plan && terraform apply
```

![잠금 상태 활성화와 함께 관련 Info를 json 형식으로 확인할 수 있다.](/assets/img/posts/image-20221029193904178.png)

잠금 상태 활성화와 함께 관련 Info를 json 형식으로 확인할 수 있다.

```json
{
   "ID":"7b8c75ae-2fed-cb9f-c5a3-76325b6ff59e",
   "Operation":"OperationTypeApply",
   "Info":"",
   "Who":"*********************"
   "Version":"1.3.2",
   "Created":"2022-10-29T21:45:13.25147Z",
   "Path":"jjikin-t101study-tfstate/dev/terraform.tfstate"
}
```

![S3에서도 변경된 사항이 저장된 버전이 생성되었음을 확인할 수 있습니다.](/assets/img/posts/image-20221029193904179.png)

S3에서도 변경된 사항이 저장된 버전이 생성되었음을 확인할 수 있습니다.

<br>

### 2. 로컬 상태파일 삭제하기

```bash
rm -rf terraform.tfstate terraform.tfstate.backup
sed -i -e 's/gasida-jjangg/jjikin-jjangg/g' dev_ec2.tf

terraform plan && terraform apply
```

![Untitiled](/assets/img/posts/image-20221029193904110.png)
_로컬에 있는 상태파일을 삭제했음에도 S3로부터 이전 태그(gasida-jjangg) 정보를 가져오는 것을 확인할 수 있습니다._

<br>

### 3. 별도의 환경(stg)을 생성 중(잠금 상태)일 때 apply 가능 여부 확인하기

- Dev 환경과 네이밍만 변경한 동일한 환경 생성

  ```bash
  mkdir stg && cd stg
  
  terraform init && terraform plan 
  terraform apply -auto-approve
  ```

- `terraform apply`  직후 다른 터미널에서 `terraform apply`  시도

  ![기존 터미널에서 stg 환경을 생성 중이므로, 잠금 상태에서는 apply가 불가능합니다.](/assets/img/posts/image-20221029193904111.png)
  _기존 터미널에서 stg 환경을 생성 중이므로, 잠금 상태에서는 apply가 불가능합니다._

  ![Untitled](/assets/img/posts/image-20221029193904112.png)
  _ 또한, 여러 환경을 구성하여 사용하더라도 Key(LockID)가 구분되어 생성되므로 각 환경에 대한 격리가 가능합니다._

<br>

## Terraform Backend 단점

- backend 블록에는 변수나 참조를 사용 할 수 없습니다.
  = backend 관련 리소스 이름을 모두 코드에 수동으로 입력해야 하며 key 값은 중복되지 않아야 합니다.

  ![Untitled](/assets/img/posts/image-20221029193904113.png)

- **partial configuration**을 통해 리소스 이름은 일부 매개 변수를 전달해서 사용 할 수 있지만, key 값의 경우각 환경별로 중복되지 않아야 하므로 수동 입력이 필요합니다.

![Untitled](/assets/img/posts/image-20221029193904176.png)

<br>

## 리소스 삭제

```bash
# 각 폴더에서 리소스 삭제
stg$ terraform destroy -auto-approve
dev$ terraform destroy -auto-approve

# S3 버킷에 객체 삭제
NICKNAME=jjikin
aws s3 rm s3://$NICKNAME-t101study-tfstate --recursive --profile ljyoon

# S3 버킷에 버저닝 객체 삭제 
aws s3api delete-objects \
    --profile ljyoon \
    --bucket $NICKNAME-t101study-tfstate \
    --delete "$(aws s3api list-object-versions \
    --bucket "${NICKNAME}-t101study-tfstate" --profile ljyoon \
    --output=json \
    --query='{Objects: Versions[].{Key:Key,VersionId:VersionId}}')"

# S3 버킷에 삭제마커 삭제
aws s3api delete-objects --bucket $NICKNAME-t101study-tfstate \
    --profile ljyoon \
    --delete "$(aws s3api list-object-versions --bucket "${NICKNAME}-t101study-tfstate" \
    --profile ljyoon \
    --query='{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}')"

# 백엔드 리소스 삭제
tfstate-backend$ terraform destroy -auto-approve
```

<br>

참고

- [Terraform State](https://developer.hashicorp.com/terraform/language/state)

  [Terraform Partical configuration](https://developer.hashicorp.com/terraform/language/settings/backends/configuration#partial-configuration)

- Terraform Up & Running

- 가시다님 스터디 자료
