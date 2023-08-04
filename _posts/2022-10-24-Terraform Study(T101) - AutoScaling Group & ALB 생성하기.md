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

## AutoScalingGroup을 구성하는 방법

AutoScalingGroup을 구성하는 방법에는 2가지가 있습니다.

- 시작 구성(Launch Configuration)
- 시작 템플릿(Launch Template)

시작 구성은 한 번 생성하면 변경할 수 없다는 특징을 가지고 있기 때문에 테라폼에서 이를 통해 ASG를 구성하고 사용하다보면 리소스 참조 문제가 발생합니다.

테라폼은 기본적으로 생성된 리소스를 먼저 삭제한 다음 대체 리소스를 생성하는데, ASG에서 이전 리소스에 대한 참조하고 있으므로 정상적으로 리소스를 삭제할 수 없기 때문입니다.

이를 해결하기 위해 테라폼에서는 licecycle 설정(create_before_destroy)을 제공합니다.

스터디는 시작 구성을 통해 진행되었지만, AWS에서는 더이상 시작 구성 사용을 권장하지 않으며 시작 템플릿을 사용하면 버전 관리 기능을 통한 업데이트가 가능하고 시작 구성에서 제공하지 않는 여러 추가 기능도 제공하므로 **시작 템플릿을 사용하여 ASG를 구성합니다.**

{: .prompt-info }

> 시작 구성은 사용하지 않는 것이 좋습니다. Amazon EC2 Auto Scaling 또는 Amazon EC2의 일부 구성을 제공하지 않기 때문입니다. 시작 구성은 시작 구성에서 시작 템플릿으로 아직 마이그레이션하지 않은 고객을 위해 제공하고 있습니다. - [Link(https://docs.aws.amazon.com/ko_kr/autoscaling/ec2/userguide/launch-configurations.html)]

<br>

## AutoScalingGroup 생성

- Launch Template을 사용하는 코드 파일 작성

  ```bash
  # asg.tf
  data "aws_ami" "amazonlinux2" {
    most_recent = true
    filter {
      name   = "owner-alias"
      values = ["amazon"]
    }
  
    filter {
      name   = "name"
      values = ["amzn2-ami-hvm-*-x86_64-gp2"]
    }
  
    owners = ["amazon"]
  }
  
  resource "aws_launch_template" "webserver-template" {
    name            = "webserver-template"
    image_id        = data.aws_ami.amazonlinux2.id
    instance_type   = "t2.micro"
    vpc_security_group_ids = [aws_security_group.webserver-sg.id]
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
  }            
  
  resource "aws_autoscaling_group" "webserver-asg" {
    name                 = "webserver-asg"
    vpc_zone_identifier  = [aws_subnet.ljyoon-pub-a-sn.id, aws_subnet.ljyoon-pub-c-sn.id]
    desired_capacity = 2
    min_size = 2
    max_size = 10
    
    launch_template {
      id      = aws_launch_template.webserver-template.id
      version = "$Latest"
    }
  
    tag {
      key                 = "Name"
      value               = "webserver-asg"
      propagate_at_launch = true
    }
  }
  ```

- 배포 진행

  - Error #1 : `plan` 에서는 문제가 없었으나 `apply` 시 아래 에러가 발생했습니다.

    ![Untitled](/assets/img/posts/image-20221024180254450.png)

    일반적인 Instance와 시작 템플릿 간 유저 데이터 처리하는 방식이 달라 발생하는 것으로 보여, 별도의 파일로 분할 후 테라폼에서 제공하는 `filebase64` 함수를 사용하여 해결했습니다.

    ```bash
    # user_data.sh
    #!/bin/bash
    wget https://busybox.net/downloads/binaries/1.31.0-defconfig-multiarch-musl/busybox-x86_64
    mv busybox-x86_64 busybox
    chmod +x busybox
    RZAZ=$(curl http://169.254.169.254/latest/meta-data/placement/availability-zone-id)
    IID=$(curl 169.254.169.254/latest/meta-data/instance-id)
    LIP=$(curl 169.254.169.254/latest/meta-data/local-ipv4)
    echo "<h1>RegionAz($RZAZ) : Instance ID($IID) : Private IP($LIP) : Web Server</h1>" > index.html
    nohup ./busybox httpd -f -p 80 &
    
    # asg.tf
    ...
    resource "aws_launch_template" "webserver-template" {
      name_prefix     = "webserver-template-"
      image_id        = data.aws_ami.amazonlinux2.id
      instance_type   = "t2.micro"
      vpc_security_groups_ids = [aws_security_group.webserver-sg.id]
      user_data = filebase64("user_data.sh")          
    ...
    ```

  - Error #2 : public IP 추가를 위해 terraform docs를 참고하여 아래와 같이 코드를 변경 후 배포했습니다.

    ```bash
    # asg.tf
    resource "aws_launch_template" "webserver-template" {
      name            = "webserver-template"
      image_id        = data.aws_ami.amazonlinux2.id
      instance_type   = "t2.micro"
      vpc_security_group_ids = [aws_security_group.webserver-sg.id]
      user_data = filebase64("user_data.sh")
    
      network_interfaces {
    		associate_public_ip_address = true
      }          
    }            
    ```

    배포는 문제없이 완료되었고, 아래와 같이 시작 템플릿 버전이 변경되었습니다.

    ![Untitled](/assets/img/posts/image-20221024180254451.png)

    기존 생성되었던 인스턴스의 버전 업데이트를 위해 인스턴스 새로고침을 시도하였으나 아래와 같이 에러가 발생했습니다.

    ![Untitled](/assets/img/posts/image-20221024180254452.png)

    시작 템플릿에 네트워크 인터페이스 블록을 사용하여 정의하게되면, 보안그룹도 해당 블록으로 넣어주어야 했습니다. 아래와 같이 코드 변경 후 해결되었습니다.

    ```bash
    # asg.tf
    resource "aws_launch_template" "webserver-template" {
      name            = "webserver-template"
      image_id        = data.aws_ami.amazonlinux2.id
      instance_type   = "t2.micro"
      user_data = filebase64("user_data.sh")
      network_interfaces {
    		associate_public_ip_address = true
        security_groups = [aws_security_group.webserver-sg.id]
      }
    }            
    ```

    ![1대씩 최신 버전으로 교체 진행](/assets/img/posts/image-20221024180254453.png)

    1대씩 최신 버전으로 교체 진행

    ![Untitled](/assets/img/posts/image-20221024180254454.png)

    ![Untitled](/assets/img/posts/image-20221024180254455.png)

- 테스트

  ![Untitled](/assets/img/posts/image-20221024180254456.png)

- ASG 구성 확인

  ```bash
  aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names --profile ljyoon | jq
  aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names --profile ljyoon--output table
  aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names myasg --profile ljyoon| jq
  aws autoscaling describe-scaling-activities --auto-scaling-group-name myasg --profile ljyoon | jq
  ```

<br>

## Application LoadBalancer 생성

ALB 구성을 위해서는 Listener, Listener rule, Target group 정의가 필요합니다.

- 코드 파일 작성

  ```bash
  # 기본사항 정의
  resource "aws_lb" "web-alb" {
    name               = "web-alb"
    load_balancer_type = "application"
    subnets            = [aws_subnet.ljyoon-pub-a-sn.id, aws_subnet.ljyoon-pub-c-sn.id]
    security_groups = [aws_security_group.webserver-sg.id]
  
    tags = {
      Name = "web-alb"
    }
  }
  
  # 리스너 정의
  resource "aws_lb_listener" "web-http" {
    load_balancer_arn = aws_lb.web-alb.arn
    port              = 80
    protocol          = "HTTP"
  
    # By default, return a simple 404 page
    default_action {
      type = "fixed-response"
  
      fixed_response {
        content_type = "text/plain"
        message_body = "404: page not found - T101 Study"
        status_code  = 404
      }
    }
  }
  
  # 타겟그룹 정의
  resource "aws_lb_target_group" "web-alb-tg" {
    name = "web-alb-tg"
    port     = 80
    protocol = "HTTP"
    vpc_id   = aws_vpc.ljyoon-vpc.id
  
    health_check {
      path                = "/"
      protocol            = "HTTP"
      matcher             = "200-299"
      interval            = 5
      timeout             = 3
      healthy_threshold   = 2
      unhealthy_threshold = 2
    }
  }
  
  # 리스너 규칙 정의
  resource "aws_lb_listener_rule" "web-alb-rule" {
    listener_arn = aws_lb_listener.web-http.arn
    priority     = 100
  
    condition {
      path_pattern {
        values = ["*"]
      }
    }
  
    action {
      type             = "forward"
      target_group_arn = aws_lb_target_group.web-alb-tg.arn
    }
  }
  
  output "web-alb_dns" {
    value       = aws_lb.web-alb.dns_name
    description = "The DNS Address of the ALB"
  }
  ```

- 배포 및 확인

  ```bash
  terraform plan
  terraform apply -auto-approve
  ...
  Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
  
  Outputs:
  	web-alb_dns = "web-alb-852605745.ap-northeast-2.elb.amazonaws.com"
  ```

  ![Untitled](/assets/img/posts/image-20221024180254457.png)

<br>

## ALB 와 ASG 연결하기

ALB의 Targetgroup을 ASG에 연결하고, Healthcheck 설정을 ELB로 변경합니다.

- 코드 수정

  ```bash
  # asg.tf
  ...
  resource "aws_autoscaling_group" "webserver-asg" {
    name                 = "webserver-asg"
    vpc_zone_identifier  = [aws_subnet.ljyoon-pub-a-sn.id, aws_subnet.ljyoon-pub-c-sn.id]
    desired_capacity = 2
    min_size = 2
    max_size = 10
    target_group_arns = [aws_lb_target_group.web-alb-tg.arn]
    health_check_type = "ELB" 
  
    launch_template {
      id      = aws_launch_template.webserver-template.id
      version = "$Latest"
    }
    ...
  ```

{: .prompt-info }

> **EC2 Healthcheck   VS   ELB Healthcheck**
  EC2 상태 확인은 하이퍼바이저(underlying hardware issue 등) 및 네트워킹 수준(잘못된 인스턴스 구성)에서 인스턴스 상태를 체크합니다.  
반면 ELB 상태 확인은 인스턴스가 Listening 하는 TCP 포트 연결 상태 또는 웹 페이지가 HTTP 2xx 코드를 응답하는지 체크합니다.  
  즉, 인스턴스에서 서비스 중인 어플리케이션의 상태 체크가 가능합니다.


- 배포 및 확인

  ```bash
  terraform plan
  terraform apply -auto-approve
  ...
  Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
  
  Outputs:
  web-alb_dns = "web-alb-852605745.ap-northeast-2.elb.amazonaws.com"
  ```

  ![ALB DNS를 통해 테스트 시 로드밸런싱을 통해 ASG내 인스턴스가 균등하게 응답했음을 확인할 수 있습니다.](/assets/img/posts/image-20221024180254458.png)

  ALB DNS를 통해 테스트 시 로드밸런싱을 통해 ASG내 인스턴스가 균등하게 응답했음을 확인할 수 있습니다.

  ![타겟그룹에 ASG가 등록되었습니다.](/assets/img/posts/image-20221024180254459.png)

  타겟그룹에 ASG가 등록되었습니다.

<br>

추후 도전과제

- 위 ASG/ALB/EC2 코드 내용 중 일부 값를 **Variable** 로 정의하여 테라폼 코드 작성 및 실습해보기 - [Link](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb#network-load-balancer)
- ALB 에 **HTTPS**(ACM 인증서) 리스너 설정하여 **SSL Offload**(HTTPS → HTTP로 타켓 대상 연결) 연결하는 테라폼 코드 작성 및 실습해보기 - [Link](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/acm_certificate)
- ALB 대신 **NLB** 를 생성하여 ASG에 연결하는 테라폼 코드 작성 및 실습해보기 - [Link](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb#network-load-balancer)

<br>

참고

- [Terraform Docs Launch Template](https://registry.terraform.io/providers/aaronfeng/aws/latest/docs/resources/launch_template)
- [Terraform Docs Function - filebase64](https://developer.hashicorp.com/terraform/language/functions/filebase64)
- [stackoverflow - cant-add-security-group-to-launch-template](https://stackoverflow.com/questions/66825815/cant-add-security-group-to-launch-template)
- [Terraform Docs aws_lb_listener](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener)
- [Terraform Docs lb_target_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group)
- 가시다님 스터디 자료
