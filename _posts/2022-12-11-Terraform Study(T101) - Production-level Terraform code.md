---
title: Terraform Study(T101) - Production-level Terraform code
date: 2022-12-11 00:00:00 +09:00
categories: [Terraform, t101]
tags: [terraform]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

## By production-grade infrastructure 란?

- 서버, 로드 밸런서, 보안 기능, 모니터링 및 경고 도구, 파이프라인 구축 및 비즈니스 운영에 필요한 기타 모든 기술들이 유기적으로 결합된 구조
- 이중화 및 장애 대응이 가능한 인프라
- 전체 아키텍처를 프로덕션 수준의 인프라로 구성하기 위해선 규모에 따라 대략적으로 6~36개월의 시간이 소요된다.

<br>

## 프로덕션 수준 인프라 구축에 오랜 시간이 걸리는 이유

1. DevOps 산업은 현재까지 여전히 발전하고 있으며 아직 안정기에 접어들지 않았음.
   - Cloud Computing, IaC, DevOps, Docker, k8s’ 등 도구의 출현과 기술이 빠르게 변하고 있으며, 충분히 성숙되지 않았음

1. DevOps는 야크 털 깎기(Yak Shaving)에 취약하다.
   - 어떤 목적을 달성하기 위해 원래 목적과 전혀 상관없는 일들을 계속하는 작업을 의미합니다.
   - 원래 목적 및 의도를 망각하고 엉뚱한 일을 하면서 의미없는 시간이 될 수 있으므로 만약 프로젝트에 신규 기능이 필요할 경우 무작정 구현부터 하기보다 잘 검증된 외부 라이브러리나 제품을 먼저 검토해 보는 것이 필요합니다.

1. 수행해야 하는 작업의 체크 리스트가 너무 많다.
   - 문제는 대다수 개발자가 체크 리스트에 있는 대부분의 항목을 알지 못하기 때문에 프로젝트를 평가할 때 중요하고 시간이 많이 걸리는 세부 사항을 놓치는 경우가 많습니다.

<br>

## 프로덕션 수준 인프라 체크 리스트

| 작업                        | 설명                                                         | 사용 가능한 도구                                             |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 설치(Install)               | 소프트웨어 바이너리나 필요한 종속성을 설치                   | 배시, 셰프, 앤서블, 퍼핏                                     |
| 설정(Configure)             | 포트 설정, TLS 인증서, 서비스 디스커버리, 리더, 팔로워, 복제 등의 소프트웨어 설정 | 배시, 셰프, 앤서블, 퍼핏                                     |
| 프로비전(Provision)         | 서버, 로드 밸런서, 네트워크, 방화벽, IAM 권한설정 등의 인프라 제공 | 테라폼, 클라우드포메이션                                     |
| 배포(Deploy)                | 인프라 상위의 서비스를 배포중단 시간 없이 업데이트를 롤아웃. 블루그린, 카나리 배포 등 | 테라폼, 클라우드포메이션, 쿠버네티스, ECS                    |
| 고가용성(High availability) | 프로세스, 서버, 서비스, 데이터 센터, 리전 등의 장애에 대비   | 멀티데이터센터, 멀티리전, 복제, 오토스케일링, 로드 밸런싱    |
| 확장성(Scalability)         | 요청량에 따른 스케일업/아웃수평적 확장(더 많은 서버), 수직적 확장(더 큰 용량) | 밸런싱오토스케일링, 복제, 샤딩, 캐싱, 분할 정복              |
| 성능(Performance)           | CPU, 메모리, 디스크, 네트워크, GPU 용량 최적화쿼리 튜닝, 벤치마킹, 테스트, 프로파일링 | 다이나트레이스(Dynatrace), 밸그린드(valgrind), 비주얼VM(VisualVM), ab, 제이미터(Jmeter) |
| 네트워킹(Networking)        | 정적 혹은 동적 IP 설정, 포트, 서비스 디스커 버리, 방화벽, DNS, SSH 접속, VPN 연결 | VPC, 방화벽, 라우터, DNS regis- tars, OpenVPN                |
| 보안(Security)              | TLS를 통한 통신 중 데이터 암호화, 디스크 암호화, 인증, 인가, 보안 관리, 서버 하드닝 | ACM, Let's Encrypt, KMS, 코그니토(Cognito), 볼트, CIS        |
| 성능 지표(Metrics)          | 가용성, 비즈니스, 애플리케이션, 서버, 이벤트, 추적, 알림에 대한 메트릭 | 클라우드워치, DataDog, New Relic, Honeycomb                  |
| 로그(Logs)                  | 로그순환, 중앙으로 로그 데이터 수집                          | 클라우드워치 Logs, ELK, Sumo 수모로직(Sumo Logic), 페이퍼트레일 (Papertrail) |
| 백업 및 복구                |                                                              |                                                              |
| (Backup and Restore)        | DB, 캐시, 기타 데이터를 일정에 따라 백업 리전별, 계정별 복제 | RDS, ElatiCache, 복제                                        |
| 비용 최적화                 |                                                              |                                                              |
| (Cost Optimization)         | 적절한 인스턴스 유형 선택, 스팟 혹은 예약 인스턴스 사용, 오토스케일링, 사용하지 않는 리소스 정리 | 오토스케일링, 스팟 인스턴스, 예약 인스턴스                   |
| 문서화(Documentation)       | 코드, 아키텍처, 모든 내용을 문서화 장애 대응 내용 정리       | README, wikis, (Slack)                                       |
| 테스트(Tests)               | 인프라코드를 테스트 자동화, 항상 테스트 후에 배포            | 테라테스트, 인스펙(inspec), 서 버스펙(serverspec), 키친 테라폼 (kitchen-terraform) |

<br>

## 프로덕션 수준 인프라 모듈

5주차를 통해 작성한 테라폼 코드들 또한 여러 개의 작은 모듈로의 리팩터링 작업이 필요하며, 리팩터링 완료된 모듈 및 전체 코드는 아래와 같습니다.

1. **small-modules/modules/cluster/asg-rolling-deploy**  
   무중단 롤링 배포를 수행할 수 있으며 ASG를 배포하기 위한 재사용 가능한 일반 독립형 모듈
   
   <details markdown="1">
   <summary>코드 접기/펼치기</summary>

   - main.tf

     ```bash
     terraform {
       required_version = ">= 1.0.0, < 2.0.0"
     
       required_providers {
         aws = {
           source  = "hashicorp/aws"
           version = "~> 4.0"
         }
       }
     }
     
     resource "aws_launch_configuration" "example" {
       image_id        = var.ami
       instance_type   = var.instance_type
       security_groups = [aws_security_group.instance.id]
       user_data       = var.user_data
     
       # Required when using a launch configuration with an auto scaling group.
       lifecycle {
         create_before_destroy = true
         precondition {
           condition     = data.aws_ec2_instance_type.instance.free_tier_eligible
           error_message = "${var.instance_type} is not part of the AWS Free Tier!"
         }
       }
     }
     
     resource "aws_autoscaling_group" "example" {
       name                 = var.cluster_name
       launch_configuration = aws_launch_configuration.example.name
     
       vpc_zone_identifier  = var.subnet_ids
     
       # Configure integrations with a load balancer
       target_group_arns    = var.target_group_arns
       health_check_type    = var.health_check_type
     
       min_size = var.min_size
       max_size = var.max_size
     
       # Use instance refresh to roll out changes to the ASG
       instance_refresh {
         strategy = "Rolling"
         preferences {
           min_healthy_percentage = 50
         }
       }
     
       tag {
         key                 = "Name"
         value               = var.cluster_name
         propagate_at_launch = true
       }
     
       dynamic "tag" {
         for_each = {
           for key, value in var.custom_tags:
           key => upper(value)
           if key != "Name"
         }
     
         content {
           key                 = tag.key
           value               = tag.value
           propagate_at_launch = true
         }
       }
     
       lifecycle {
         postcondition {
           condition     = length(self.availability_zones) > 1
           error_message = "You must use more than one AZ for high availability!"
         }
       }
     
     }
     
     resource "aws_autoscaling_schedule" "scale_out_during_business_hours" {
       count = var.enable_autoscaling ? 1 : 0
     
       scheduled_action_name  = "${var.cluster_name}-scale-out-during-business-hours"
       min_size               = 2
       max_size               = 10
       desired_capacity       = 10
       recurrence             = "0 9 * * *"
       autoscaling_group_name = aws_autoscaling_group.example.name
     }
     
     resource "aws_autoscaling_schedule" "scale_in_at_night" {
       count = var.enable_autoscaling ? 1 : 0
     
       scheduled_action_name  = "${var.cluster_name}-scale-in-at-night"
       min_size               = 2
       max_size               = 10
       desired_capacity       = 2
       recurrence             = "0 17 * * *"
       autoscaling_group_name = aws_autoscaling_group.example.name
     }
     
     resource "aws_security_group" "instance" {
       name = "${var.cluster_name}-instance"
     }
     
     resource "aws_security_group_rule" "allow_server_http_inbound" {
       type              = "ingress"
       security_group_id = aws_security_group.instance.id
     
       from_port   = var.server_port
       to_port     = var.server_port
       protocol    = local.tcp_protocol
       cidr_blocks = local.all_ips
     }
     
     resource "aws_cloudwatch_metric_alarm" "high_cpu_utilization" {
       alarm_name  = "${var.cluster_name}-high-cpu-utilization"
       namespace   = "AWS/EC2"
       metric_name = "CPUUtilization"
     
       dimensions = {
         AutoScalingGroupName = aws_autoscaling_group.example.name
       }
     
       comparison_operator = "GreaterThanThreshold"
       evaluation_periods  = 1
       period              = 300
       statistic           = "Average"
       threshold           = 90
       unit                = "Percent"
     }
     
     resource "aws_cloudwatch_metric_alarm" "low_cpu_credit_balance" {
       count = format("%.1s", var.instance_type) == "t" ? 1 : 0
     
       alarm_name  = "${var.cluster_name}-low-cpu-credit-balance"
       namespace   = "AWS/EC2"
       metric_name = "CPUCreditBalance"
     
       dimensions = {
         AutoScalingGroupName = aws_autoscaling_group.example.name
       }
     
       comparison_operator = "LessThanThreshold"
       evaluation_periods  = 1
       period              = 300
       statistic           = "Minimum"
       threshold           = 10
       unit                = "Count"
     }
     
     data "aws_ec2_instance_type" "instance" {
       instance_type = var.instance_type
     }
     
     locals {
       tcp_protocol = "tcp"
       all_ips      = ["0.0.0.0/0"]
     }
     ```

   - outputs.tf

     ```bash
     output "asg_name" {
       value       = aws_autoscaling_group.example.name
       description = "The name of the Auto Scaling Group"
     }
     
     output "instance_security_group_id" {
       value       = aws_security_group.instance.id
       description = "The ID of the EC2 Instance Security Group"
     }
     ```

   - variables.tf

     ```bash
     # ---------------------------------------------------------------------------------------------------------------------
     # REQUIRED PARAMETERS
     # You must provide a value for each of these parameters.
     # ---------------------------------------------------------------------------------------------------------------------
     
     variable "cluster_name" {
       description = "The name to use for all the cluster resources"
       type        = string
     }
     
     variable "ami" {
       description = "The AMI to run in the cluster"
       type        = string
     }
     
     variable "instance_type" {
       description = "The type of EC2 Instances to run (e.g. t2.micro)"
       type        = string
     
       validation {
         condition     = contains(["t2.micro", "t3.micro"], var.instance_type)
         error_message = "Only free tier is allowed: t2.micro | t3.micro."
       }
     }
     
     variable "min_size" {
       description = "The minimum number of EC2 Instances in the ASG"
       type        = number
     
       validation {
         condition     = var.min_size > 0
         error_message = "ASGs can't be empty or we'll have an outage!"
       }
     
       validation {
         condition     = var.min_size <= 10
         error_message = "ASGs must have 10 or fewer instances to keep costs down."
       }
     }
     
     variable "max_size" {
       description = "The maximum number of EC2 Instances in the ASG"
       type        = number
     
       validation {
         condition     = var.max_size > 0
         error_message = "ASGs can't be empty or we'll have an outage!"
       }
     
       validation {
         condition     = var.max_size <= 10
         error_message = "ASGs must have 10 or fewer instances to keep costs down."
       }
     }
     
     variable "subnet_ids" {
       description = "The subnet IDs to deploy to"
       type        = list(string)
     }
     
     variable "enable_autoscaling" {
       description = "If set to true, enable auto scaling"
       type        = bool
     }
     
     # ---------------------------------------------------------------------------------------------------------------------
     # OPTIONAL PARAMETERS
     # These parameters have reasonable defaults.
     # ---------------------------------------------------------------------------------------------------------------------
     
     variable "target_group_arns" {
       description = "The ARNs of ELB target groups in which to register Instances"
       type        = list(string)
       default     = []
     }
     
     variable "health_check_type" {
       description = "The type of health check to perform. Must be one of: EC2, ELB."
       type        = string
       default     = "EC2"
     
       validation {
         condition     = contains(["EC2", "ELB"], var.health_check_type)
         error_message = "The health_check_type must be one of: EC2 | ELB."
       }
     }
     
     variable "user_data" {
       description = "The User Data script to run in each Instance at boot"
       type        = string
       default     = null
     }
     
     variable "custom_tags" {
       description = "Custom tags to set on the Instances in the ASG"
       type        = map(string)
       default     = {}
     }
     
     variable "server_port" {
       description = "The port the server will use for HTTP requests"
       type        = number
       default     = 8080
     }
     ```
   </details>
   <br>
   
2. **small-modules/modules/networking/alb**  
   ALB를 배포하기 위한 재사용 가능한 일반 독립형 모듈
   
   <details markdown="1">
   <summary>코드 접기/펼치기</summary>
   
   - main.tf
   
     ```bash
     terraform {
       required_version = ">= 1.0.0, < 2.0.0"
     
       required_providers {
         aws = {
           source  = "hashicorp/aws"
           version = "~> 4.0"
         }
       }
     }
     
     resource "aws_lb" "example" {
       name               = var.alb_name
       load_balancer_type = "application"
     
       subnets            = var.subnet_ids
     
       security_groups    = [aws_security_group.alb.id]
     }
     
     resource "aws_lb_listener" "http" {
       load_balancer_arn = aws_lb.example.arn
       port              = local.http_port
       protocol          = "HTTP"
     
       # By default, return a simple 404 page
       default_action {
         type = "fixed-response"
     
         fixed_response {
           content_type = "text/plain"
           message_body = "404: page not found"
           status_code  = 404
         }
       }
     }
     
     resource "aws_security_group" "alb" {
       name = var.alb_name
     }
     
     resource "aws_security_group_rule" "allow_http_inbound" {
       type              = "ingress"
       security_group_id = aws_security_group.alb.id
     
       from_port   = local.http_port
       to_port     = local.http_port
       protocol    = local.tcp_protocol
       cidr_blocks = local.all_ips
     }
     
     resource "aws_security_group_rule" "allow_all_outbound" {
       type              = "egress"
       security_group_id = aws_security_group.alb.id
     
       from_port   = local.any_port
       to_port     = local.any_port
       protocol    = local.any_protocol
       cidr_blocks = local.all_ips
     }
     
     locals {
       http_port    = 80
       any_port     = 0
       any_protocol = "-1"
       tcp_protocol = "tcp"
       all_ips      = ["0.0.0.0/0"]
     }
     ```
   
   - outputs.tf
   
     ```bash
     output "alb_dns_name" {
       value       = aws_lb.example.dns_name
       description = "The domain name of the load balancer"
     }
     
     output "alb_http_listener_arn" {
       value       = aws_lb_listener.http.arn
       description = "The ARN of the HTTP listener"
     }
     
     output "alb_security_group_id" {
       value       = aws_security_group.alb.id
       description = "The ALB Security Group ID"
     }
     ```
   
   - variables.tf
   
     ```bash
     # ---------------------------------------------------------------------------------------------------------------------
     # REQUIRED PARAMETERS
     # You must provide a value for each of these parameters.
     # ---------------------------------------------------------------------------------------------------------------------
     
     variable "alb_name" {
       description = "The name to use for this ALB"
       type        = string
     }
     
     variable "subnet_ids" {
       description = "The subnet IDs to deploy to"
       type        = list(string)
     }
     ```
   </details>
   <br>
   
3. **small-modules/modules/services/hello-world-app**

   <details markdown="1">
   <summary>코드 접기/펼치기</summary>
   
   - main.tf

     ```bash
     terraform {
       # Require any 1.x version of Terraform
       required_version = ">= 1.0.0, < 2.0.0"
     
       required_providers {
         aws = {
           source  = "hashicorp/aws"
           version = "~> 4.0"
         }
       }
     }
     
     module "asg" {
       source = "../../cluster/asg-rolling-deploy"
     
       cluster_name  = "hello-world-${var.environment}"
       ami           = var.ami
       instance_type = var.instance_type
     
       user_data     = templatefile("${path.module}/user-data.sh", {
         server_port = var.server_port
         db_address  = data.terraform_remote_state.db.outputs.address
         db_port     = data.terraform_remote_state.db.outputs.port
         server_text = var.server_text
       })
     
       min_size           = var.min_size
       max_size           = var.max_size
       enable_autoscaling = var.enable_autoscaling
     
       subnet_ids        = data.aws_subnets.default.ids
       target_group_arns = [aws_lb_target_group.asg.arn]
       health_check_type = "ELB"
       
       custom_tags = var.custom_tags
     }
     
     module "alb" {
       source = "../../networking/alb"
     
       alb_name   = "hello-world-${var.environment}"
       subnet_ids = data.aws_subnets.default.ids
     }
     
     resource "aws_lb_target_group" "asg" {
       name     = "hello-world-${var.environment}"
       port     = var.server_port
       protocol = "HTTP"
       vpc_id   = data.aws_vpc.default.id
     
       health_check {
         path                = "/"
         protocol            = "HTTP"
         matcher             = "200"
         interval            = 15
         timeout             = 3
         healthy_threshold   = 2
         unhealthy_threshold = 2
       }
     }
     
     resource "aws_lb_listener_rule" "asg" {
       listener_arn = module.alb.alb_http_listener_arn
       priority     = 100
     
       condition {
         path_pattern {
           values = ["*"]
         }
       }
     
       action {
         type             = "forward"
         target_group_arn = aws_lb_target_group.asg.arn
       }
     }
     
     data "terraform_remote_state" "db" {
       backend = "s3"
     
       config = {
         bucket = var.db_remote_state_bucket
         key    = var.db_remote_state_key
         region = "us-east-2"
       }
     }
     
     data "aws_vpc" "default" {
       default = true
     }
     
     data "aws_subnets" "default" {
       filter {
         name   = "vpc-id"
         values = [data.aws_vpc.default.id]
       }
     }
     ```

   - outputs.tf

     ```bash
     output "alb_dns_name" {
       value       = module.alb.alb_dns_name
       description = "The domain name of the load balancer"
     }
     
     output "asg_name" {
       value       = module.asg.asg_name
       description = "The name of the Auto Scaling Group"
     }
     
     output "instance_security_group_id" {
       value       = module.asg.instance_security_group_id
       description = "The ID of the EC2 Instance Security Group"
     }
     ```

   - variables.tf

     ```bash
     # ---------------------------------------------------------------------------------------------------------------------
     # REQUIRED PARAMETERS
     # You must provide a value for each of these parameters.
     # ---------------------------------------------------------------------------------------------------------------------
     
     variable "environment" {
       description = "The name of the environment we're deploying to"
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
     
     variable "enable_autoscaling" {
       description = "If set to true, enable auto scaling"
       type        = bool
     }
     
     variable "db_remote_state_bucket" {
       description = "The name of the S3 bucket used for the database's remote state storage"
       type        = string
     }
     
     variable "db_remote_state_key" {
       description = "The name of the key in the S3 bucket used for the database's remote state storage"
       type        = string
     }
     
     # ---------------------------------------------------------------------------------------------------------------------
     # OPTIONAL PARAMETERS
     # These parameters have reasonable defaults.
     # ---------------------------------------------------------------------------------------------------------------------
     
     variable "ami" {
       description = "The AMI to run in the cluster"
       type        = string
     }
     
     variable "instance_type" {
       description = "The type of EC2 Instances to run (e.g. t2.micro)"
       type        = string
       default     = "t2.micro"
     }
     
     variable "server_text" {
       description = "The text the web server should return"
       default     = "Hello, World"
       type        = string
     }
     
     variable "server_port" {
       description = "The port the server will use for HTTP requests"
       type        = number
       default     = 8080
     }
     
     variable "custom_tags" {
       description = "Custom tags to set on the Instances in the ASG"
       type        = map(string)
       default     = {}
     }
     ```

   - user-data.sh

     ```bash
     #!/bin/bash
     
     cat > index.html <<EOF
     <h1>${server_text}</h1>
     <p>DB address: ${db_address}</p>
     <p>DB port: ${db_port}</p>
     EOF
     
     nohup busybox httpd -f -p ${server_port} &
     ```
   </details>
   <br>
   
### **소형 모듈(**Small modules)

모든 인프라 환경들을 단일 파일, 대형 모듈로 정의하는 것은 아래의 이유로 권장되지 않습니다. 

- 속도 저하 : 모든 인프라가 하나의 모듈에 정의되어 있으면 명령 실행 속도가 저하되며 terraform plan에만 20분 걸리는 케이스도 발생합니다.
- 불안정성 : 어떤 것을 변경하려고해도 여러 권한 또는 모든 액세스 권한이 필요할 수 있습니다.
- 고위험성 : 예를 들어 스테이징 환경에서 프런트엔드 앱을 변경 시 오타나 잘못된 명령으로 프로덕션 데이터베이스를 삭제할 수 있습니다.
- 이해 및 리뷰의 어려움  : 한 곳에 코드가 많을수록 모든 것을 이해 및 리뷰하기가 더 어렵습니다.
  - terraform plan 실행 시 오래 걸리고 plan 명령의 출력이 수천 줄인 경우 아무도 코드를 읽으려 하지 않으며 중요한 알람(삭제 관련)을 놓칠 수 있습니다.
- 테스트의 어려움

위의 문제들을 해결하기 위해서는 소형 모듈로 코드를 작성해야합니다.

<br>

### **합성 가능한 모듈(Composable modules**)

- **외부에서 상태를 읽는 대신 입력 변수를 통해 전달하고, 외부에 상태를 쓰는 대신 출력 변수를 통해 계산 결과를 반환합니다.**
- 모든 것을 입력 변수를 통해 전달하고 모든 것을 출력 변수를 통해 반환하면 간단한 모듈들을 결합해 더 고도화된 모듈을 만들 수 있습니다.
- 실제 프로덕션 환경에서는 더 나은 합성과 재사용을 위해 아래 실습 내용 보다 모듈을 더욱 세분화해야 할 수도 있습니다.

<br>

### **테스트 가능한 모듈(Testable modules**)

- asg-rolling-deploy 모듈을 사용하여 크기가 1인 ASG를 배포합니다.

  ```bash
  terraform {
    required_version = ">= 1.0.0, < 2.0.0"
  
    required_providers {
      aws = {
        source  = "hashicorp/aws"
        version = "~> 4.0"
      }
    }
  }
  
  provider "aws" {
    region = "us-east-2"
  }
  
  module "asg" {
    source = "../../modules/cluster/asg-rolling-deploy"
  
    cluster_name  = var.cluster_name
  
    ami           = data.aws_ami.ubuntu.id
    instance_type = "t2.micro"
  
    min_size           = 1
    max_size           = 1
    enable_autoscaling = false
  
    subnet_ids        = data.aws_subnets.default.ids
  }
  
  data "aws_vpc" "default" {
    default = true
  }
  
  data "aws_subnets" "default" {
    filter {
      name   = "vpc-id"
      values = [data.aws_vpc.default.id]
    }
  }
  
  data "aws_ami" "ubuntu" {
    most_recent = true
    owners      = ["099720109477"] # Canonical
  
    filter {
      name   = "name"
      values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
    }
  }
  
  # 배포
  cd ~/terraform-up-and-running-code/code/terraform/08-production-grade-infrastructure/small-modules/examples/**asg**
  terraform init
  terraform plan
  terraform apply -auto-approve
  
  # ALB 배포
  cd ~/terraform-up-and-running-code/code/terraform/08-production-grade-infrastructure/small-modules/examples/**alb**
  terraform init && terraform plan
  terraform apply -auto-approve
  ```

<br>

### **릴리스 가능한 모듈(**Versioned modules)

- 테라폼 0.13 validation blocks은 입력 변수를 체크할 수 있습니다.

  - instance_type 으로 t2.micro 와 t3.micro 만 사용할 수 있도록 제한합니다.

    ```bash
    variable "instance_type" {
      description = "The type of EC2 Instances to run (e.g. t2.micro)"
      type        = string
    
      validation {
        condition     = contains(["t2.micro", "t3.micro"], var.instance_type)
        error_message = "Only free tier is allowed: t2.micro | t3.micro."
      }
    }
    
    # check
    $ terraform apply -var instance_type="m4.large"
    │ Error: Invalid value for variable
    │
    │   on main.tf line 17:
    │    1: variable "instance_type" {
    │     ├────────────────
    │     │ var.instance_type is "m4.large"
    │
    │ Only free tier is allowed: t2.micro | t3.micro.
    │
    │ This was checked by the validation rule at main.tf:21,3-13.
    ```

  <br>

- Versioned Modules : 실행 시기에 상관없이 항상 동일한 결과를 얻을 수 있어야 함

  - 두 가지 유형 버전 고려 필요 : 모듈 의존적인 버전 관리, 모듈 자체의 버전 관리

  - 모듈 의존적인 버전 관리

    - 테라폼 실행 파일 버전(Terraform core) 고정 : Production 환경에서는 required_version 사용하여 버전 직접 지정 권장

      ```bash
      terraform {
        # Require any 1.x version of Terraform
        required_version = ">= 1.0.0, < 2.0.0"
      }
      ```

    - 프로바이더 버전 고정 : required_providers 아래 version 사용

      ```bash
      terraform {
        required_version = ">= 1.0.0, < 2.0.0"
      
        required_providers {
          aws = {
            source  = "hashicorp/aws"
            version = "~> 4.0"
          }
        }
      }
      ```

    - 모듈 버전 고정 : 시맨틱 버전 관리와 함께 깃 태그를 사용

<br>

### Beyond Terraform modules

- Provisioners
- Provisioners with null_resource
- External data source

<br>

참고

- 가시다님 스터디 자료 
