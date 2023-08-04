---
title: Terraform Study(T101) - Terraform 코드와 실제 배포된 형상이 일치하지 않는 경우
date: 2022-10-20 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## Terraform 코드와 실제 배포된 형상이 일치하지 않는 경우

### 테스트 환경 생성

```bash
# 신규 폴더 생성
mkdir sync-state && cd sync-state

# provider.tf
provider "aws" {
  region = "ap-northeast-2"
  profile = "ljyoon"
}

# main.tf(S3 Bucket 생성)
resource "aws_s3_bucket" "t101study" {
    bucket = "jjikin-t101study-bucket"
}

terraform init
terraform plan
terrafrom apply -auto-approve

# 모니터링
while true; do aws s3 ls --profile ljyoon; echo "------------------------------"; date; sleep 1; done
```

- 별도 backend를 지정하지 않았으므로 작업 결과는 로컬에 있는 `terraform.tfstate` 파일에 저장됩니다.

  ![Untitled](/assets/img/posts/image-20221020174519650.png)

  ```json
  # terraform.tfstate
  {
    "version": 4,
    "terraform_version": "1.3.2",
    "serial": 8,
    "lineage": "1715d2c0-e36f-b04b-9c1c-75a202eebf7f",
    "outputs": {},
    "resources": [
      {
        "mode": "managed",
        "type": "aws_s3_bucket",
        "name": "t101study",
        "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
        "instances": [
          {
            "schema_version": 0,
            "attributes": {
              "acceleration_status": "",
              "acl": null,
              "arn": "arn:aws:s3:::jjikin-t101study-bucket",
              "bucket": "jjikin-t101study-bucket",
              "bucket_domain_name": "jjikin-t101study-bucket.s3.amazonaws.com",
              "bucket_prefix": null,
              "bucket_regional_domain_name": "jjikin-t101study-bucket.s3.ap-northeast-2.amazonaws.com",
  ...
  ```

<br>

### 강제로 코드 파일(.tf)을 삭제한다면

- 리소스가 정의된 코드 파일은 삭제, tfstate에는 S3가 존재, 실제 리소스도 존재하는 경우

  ```bash
  rm -rf main.tf
  
  terraform plan
  	aws_s3_bucket.t101study: Refreshing state... [id=jjikin-t101study-bucket]
  	
  	Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with
  	the following symbols:
  	  - destroy
  	
  	Terraform will perform the following actions:
  	  # aws_s3_bucket.t101study will be destroyed
  	  # (because aws_s3_bucket.t101study is not in configuration)
  	  ...
  
  terraform apply
  	aws_s3_bucket.t101study: Destroying... [id=jjikin-t101study-bucket]
  	aws_s3_bucket.t101study: Destruction complete after 0s
  ```

  - Apply 시 실제 리소스가 삭제됩니다.

<br>

### 강제로 상태 파일(.tfstate)을 삭제한다면

- 리소스가 정의된 코드 파일 존재, tfstate 삭제, 실제 리소스는 존재하는 경우

  ```bash
  # main.tf (S3 Bucket 재생성)
  resource "aws_s3_bucket" "t101study" {
      bucket = "jjikin-t101study-bucket-2"
  }
  terraform apply -auto-approve
  
  # 상태파일 삭제
  rm -rf terraform.tfstate
  
  terraform plan
  	Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with
  	the following symbols:
  	  + create
  	
  	Terraform will perform the following actions:
  	
  	  # aws_s3_bucket.t101study will be created
  	  + resource "aws_s3_bucket" "t101study"
  
  terraform apply
  	aws_s3_bucket.t101study: Creating...
  	
  	Error: creating Amazon S3 (Simple Storage) Bucket (jjikin-t101study-bucket-2):
    BucketAlreadyOwnedByYou: Your previous request to create the named bucket succeeded 
    and you already own it.
  	 	status code: 409, request id: FJE0977Z5MW5YK5R, host id: RnBaw4u3eK+tpZRCZhQjaA51KeY/QFP0ducYUk8lHv7CPfrHou/fnqicZq1PE3du4ywSfW2N0lw=
  	...
  
  # terraform.tfstate
  {
    "version": 4,
    "terraform_version": "1.3.2",
    "serial": 2,
    "lineage": "beaddca2-6d89-bf43-6d69-65a038ae54a1",
    "outputs": {},
    "resources": [],
    "check_results": []
  }
  ```

  - Plan 시 버킷을 생성한다고 표시되지만, 실제 Apply 시 이미 존재하는 버킷이므로 에러가 발생합니다.
  - 다시 생성된 tfstate 파일에도 버킷 정보가 없음을 확인할 수 있습니다.

- 이러한 불일치 상황을 해결하려면 `terraform import`를 통해 강제로 상태를 맞춰줘야 합니다.

  ```bash
  terraform import aws_s3_bucket.t101study jjikin-t101study-bucket-2
  ```

  ![Untitled](/assets/img/posts/image-20221020174519651.png)

<br>

### `terraform import` 시에도 중복 에러가 발생한다면

- 테라폼 코드를 통해 RDS를 생성한 후, S3 Backend 설정하는 과정에서 자격 증명 오류로 상태 파일이 누락되었습니다.

  이후 `terraform apply` 시 관련 리소스의 중복 에러가 발생하며, import 시도해도 동일합니다.

  ![Untitled](/assets/img/posts/image-20221020174519652.png)

  이 경우 import 하려는 리소스가 들어갈 공간을 만들어 줘야합니다. 아래와 같이 리소스 정의 부분에서 관련 내용들을 모두 삭제합니다.

  ```bash
  # 삭제 전
  resource "aws_db_subnet_group" "db-sn-group" {
    name       = "db-sn-group"
    subnet_ids = [data.terraform_remote_state.vpc.outputs.pri-a-sn, data.terraform_remote_state.vpc.outputs.pri-c-sn]
  
    tags = {
      Name = "db-sn-group"
    }
  }
  
  # 삭제 후
  resource "aws_db_subnet_group" "db-sn-group" {
  
  }
  ```

  삭제 후 import 시 정상적으로 리소스 정보를 가져오는지 확인합니다.

  ![Untitled](/assets/img/posts/image-20221020174519653.png)

  해당 케이스는 리소스 정보를 가져오지 못한 경우입니다.

  ![Untitled](/assets/img/posts/image-20221020174519654.png)

다시 `terraform apply` 시 리소스 정보를 가져온 것을 확인할 수 있습니다. 하지만 apply 과정에서 실제 리소스가 삭제 후 재생성될 가능성이 있으므로 주의해야합니다.

![Untitled](/assets/img/posts/image-20221020174519655.png)

<br>

참고 

- 가시다님 스터디 자료
