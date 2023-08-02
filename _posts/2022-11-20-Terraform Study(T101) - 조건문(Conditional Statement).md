---
title: Terraform Study(T101) - 조건문(Conditional Statement)
date: 2022-11-20 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---



## 반복문과 조건문을 같이 사용하는 CASE

테라폼에서 사용할 수 있는 조건문의 구성은 아래와 같습니다.

- count parameter : 조건부 리소스에서 사용
- for_each, for expressions : 리소스 내의 조건부 리소스 및 인라인 블록에서 사용
- if string directive(문자열 지시어) : 문자열 내 조건문에 사용

<br>

## count + if

모듈의 오토스케일링 사용 여부를 특정 환경에 따라 조건부로 지정해야하는 경우

```hcl
# modules/services/webserver-cluster/variables.tf
variable "enable_autoscaling" {
  description = "If set to true, enable auto scaling"
  type        = bool
}

# stage/services/webserver-cluster/main.tf
module "webserver_cluster" {
  source = "../../../modules/services/webserver-cluster"
	env = "stage"
	instance_type = "t2.micro"
  min_size = 1
  max_size = 1
  enable_autoscaling = false
}

# modules/services/webserver-cluster/main.tf
resource "aws_autoscaling_schedule" "scale_out_during_business_hours" {
  count = var.enable_autoscaling ? 1 : 0
  scheduled_action_name = "scale-out-during-business-hours"
  min_size              = 2
  max_size              = 2
  desired_capacity      = 2
  recurrence            = "0 9 * * *"
	autoscaling_group_name = module.webserver_cluster.asg_name
}

resource "aws_autoscaling_schedule" "scale_in_at_night" {
  count = var.enable_autoscaling ? 1 : 0
  scheduled_action_name = "scale-in-at-night"
  min_size              = 0
  max_size              = 2
  desired_capacity      = 0
  recurrence            = "0 18 * * *"
	autoscaling_group_name = module.webserver_cluster.asg_name
}
```

- 입력 변수 enable_autoscaling의 값이 1(True)인 경우, 1번 반복하므로 리소스가 각각 한개씩 생성됩니다.
- 입력 변수 enable_autoscaling의 값이 0(False)인 경우, 반복하지 않으므로 리소스가 생성되지 않습니다.

<br>

## count + if-else

IAM User 생성 시 특정 AWS 서비스 Full Access 적용 여부를 결정해야 하는 경우

```hcl
variable "give_cloudwatch_full_access" {
  description = "If true, gets full access to CloudWatch"
  type        = bool
}

resource "aws_iam_user_policy_attachment" "cloudwatch_full_access" {
  count = var.give_cloudwatch_full_access ? 1 : 0

  user       = aws_iam_user.example[0].name
  policy_arn = aws_iam_policy.cloudwatch_full_access.arn
}

resource "aws_iam_user_policy_attachment" "cloudwatch_read_only" {
  count = var.give_cloudwatch_full_access ? 0 : 1

  user       = aws_iam_user.example[0].name
  policy_arn = aws_iam_policy.cloudwatch_read_only.arn
}
```

- resource에서 생성되는 출력 값(속성)들은 참조할 수 없는 단점이 존재합니다.

<br>

## for_each, for expresstion + if

```hcl
dynamic "tag" {
	for_each = { for key, value in var.custom_tags: key => upper(value) if key != "Name" }

    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
```

- 리소스를 조건부로 생성할 때는 count 를 사용할 수 있지만, 그 외 모든 유형의 반복문 또는 조건문에는 for_each 를 사용합니다.
- 리소스의 복사본을 여러 개 만들 때는 count 보다 for_each 를 사용
- 조건 논리의 경우 비어 있지 않은 컬렉션에 for_each 를 설정하는 것보다 count 를 0 또는 1로 설정하는 것이 간단

<br>

## string directive + if

```bash
%{ if <CONDITION> }<TRUEVAL>%{ endif }

#else
%{ if <CONDITION> }<TRUEVAL>%{ else }<FALSEVAL>%{ endif }
```

- CONDITION은 boolean 으로 평가되는 표현식이고, TRUEVAL은 CONDITION이 True로 평가되면 렌더링할 표현식입니다.

<br>

### 사용 예시

```hcl
variable "users" {
  description = "users to render"
  type        = list(string)
  default     = ["aaa", "bbb", "ccc"]
}

output "for_directive" {
  value = "%{ for name in var.users }\${name}, %{ endfor }"
}

output "for_directive_index" {
  value = "%{ for i, name in var.users }(\${i}) \${name}, %{ endfor }"
}

output "for_directive_index_if" {
  value = %{ for i, name in var.users }${name}%{ if i < length(var.users) - 1 }, %{ endif }%{ endfor }
}

output "for_directive_index_if_strip" {
  value = <<EOF
%{~ for i, name in var.names ~}
  \${name}%{ if i < length(var.names) - 1 }, %{ endif }
%{~ endfor ~}
EOF
}

output "for_directive_index_if_else_strip" {
  value = <<EOF
%{~ for i, name in var.names ~}
  \${name}%{ if i < length(var.names) - 1 }, %{ else }.%{ endif }
%{~ endfor ~}
EOF
}

# output
Outputs:
for_directive_index_if = <<EOT

  gasida,

  akbun,

  fullmoon

EOT
for_directive_index_if_strip = "  aaa,   bbb,   ccc"
for_directive_index_if_else_strip = "  aaa,   bbb,   ccc."

```

- 줄 바꿈, 스페이스 같은 공백을 없애기 위해 문자열 지시자의 앞이나 뒤에 물결표(~)를 사용할 수 있습니다.
- else를 활용하여 출력 마지막에 .를 찍어보기

<br>

참고

- Terraform Docsㅣ[Function](https://developer.hashicorp.com/terraform/language/functions)
- Terraform Docsㅣ[Expressions](https://developer.hashicorp.com/terraform/language/expressions)
- 가시다님 스터디 자료
- [악분님 Youtube](https://www.youtube.com/watch?v=fhgGcC7wJoc)
