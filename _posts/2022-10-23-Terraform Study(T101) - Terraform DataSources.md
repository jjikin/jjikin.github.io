---
title: Terraform Study(T101) - Terraform DataSources
date: 2022-10-23 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## Terraform Data Sources 란

Data Source를 사용하면 Terraform으로 구성되지 않은 리소스 또는 다른 Terraform Code를 통해 생성된 리소스의 데이터를 가져올 수 있습니다.

AWS의 경우 VPC, subnet, AMI IDs, IP address ranges, user’s identity 등의 데이터를 제공합니다.

### 선언 방법

Data Source의 사용 방법과 사용할 수 있는 인수들은 아래와 같습니다.

> **data “<PROVIDER>_<TYPE>” “<NAME>” {
> [CONFIG …]
> }**

- **TYPE** : 사용하려는 데이터 소스의 유형입니다.
- **NAME** : 테라폼 코드에서 선언한 데이터 소스를 참조할 때 사용할 식별자입니다.
- **CONFIG** : Data를 가져오기 위해서는 해당 데이터 소스에 하나 이상의 값(Argument)이 필요합니다.
  AWS의 각 Type별 사용할 수 있는 Argument는 아래 Terraform Docs를 통해 확인 가능합니다.
  [https://registry.terraform.io/providers/hashicorp/aws/latest/docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

### 선언 예시

```bash
data "aws_ami" "webserver" {
  owners = ["ljyoon"]

  tags = {
    Name   = "webserver"
  }

  most_recent = true
}

data "aws_iam_user" "ljyoon" {
  user_name = "ljyoon"
}
```

### 사용 방법

- 데이터 소스에서 데이터를 가져오려면 선언한 Type의 속성(Attribute) 참조 구문을 사용합니다.

  > **data.<PROVIDER>_<TYPE>.<NAME>.<ATTRIBUTE>**

  AWS의 각 Type별 사용할 수 있는 속성은 위의 Terraform Docs를 통해 확인 가능합니다.

- 다른 데이터 소스와 결합하여 VPC 내 서브넷을 조회합니다.

  ```bash
  data "aws_subnets" "default" {
    filter {
      name   = "vpc-id"
      values = [data.aws_vpcs.default.id]
    }
  }
  ```

- ASG에 aws_subnets 데이터 소스에서 가져온 서브넷ID를 사용하도록 합니다.

  ```bash
  resource "aws_autoscaling_group" "example" {
    launch_configuration = aws_launch_configuration.example.name
    vpc_zone_identifier  = data.aws_subnets.default.ids
  
    min_size = 2
    max_size = 10
    ...
  ```

<br>

참고

- [Terraform data sources](https://velog.io/@gentledev10/terraform-data-source)
- [MZC Terraform Workshop](https://mzcdev.github.io/terraform-workshop)
- 가시다님 스터디 자료
