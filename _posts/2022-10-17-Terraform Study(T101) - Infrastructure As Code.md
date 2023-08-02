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

## IaC 란

Infrastructure As Code의 약자로써, 인프라의 생성 및 수정, 배포를 코드를 통해 관리하는 것을 의미합니다.



## IaC의 종류

1. 애드혹 스크립트(Ad-hoc Script)

   bash, python과 같은 프로그래밍 언어를 통해 작성되어 서버 단에서 실행되는 스크립트입니다. 
   가장 간단한 자동화 방법이면서 일회성 작업에 적합합니다.

   ```bash
   #!/bin/bash
   sudo apt-get update
   sudo apt-get install -y php apache2
   sudo git clone https://github.com/xxxx/xxxxx.git /var/www/html/app
   sudo service apache2 start
   ```

2. 구성 관리 도구

   서버에 소프트웨어를 설치하여 사용하는 방식으로 다수의 서버에서 동일한 코드를 실행하여 구성 및 변경을 자동화합니다. 대표적인 툴로 Ansible, Chef, Puppet 등이 있으며 애드혹 스크립트 대비 아래의 장점을 가집니다.

   - 코딩 규칙(coding convention)
     : 도구마다 일관되고 예측가능한 구조(레이아웃, 매개변수 등)
   - 멱등성(Idempotence)
     : 실행 횟수에 관계없이 항상 같은 결과값을 얻을 수 있는 성질
   - 분산형 구조(Distribution)
     : 다수의 서버 관리

   ```yaml
   # Ansible 
   - name: Update the apt-get cache
     apt:
       update_cache: yes
   
   - name: Install PHP
     apt:
       name: php
   
   - name: Install Apache
     apt:
       name: apache2
   
   - name: Git clone Repository
     git: repo=https://github.com/foo/php-app dest=/var/www/html/app
   
   - name: Start Apache
     systemd: 
       name: apache2 
       state: started
   ```

3. 서버 템플릿 도구

   운영체제, 어플리케이션 등의 서비스를 위해 필요한 모든 구성 요소를 포함하는 스냅샷으로 이미지를 생성합니다.
   생성된 이미지는 불변 인프라 요소이며 구성 관리 도구를 통해 사용됩니다.
   대표적인 툴로 Docker, Vagrant, Packer 등이 있습니다.

   ```yaml
   # DockerFile
   FROM ubuntu:20.04  
   
   RUN apt-get update && apt-get -y install nginx
   
   WORKDIR /etc/nginx
   
   CMD ["nginx", "-g", "daemon off;"]
   
   EXPOSE 80
   
   # Build image
   docker build -t nginx .
   ```

   > 불변 인프라(immutable infrastructure)
   >  : 한번 배포된 리소스는 다시 변경할 수 없으며 변경을 위해서는 신규 생성 후 배포해야하는 구조

4. 오케스트레이션 도구

   가상머신(VM), 컨테이너의 효율적 관리를 위한 도구입니다.
   대표적으로 Kubernetes, Docker Swarm 등이 있으며, 서버 템플릿 도구를 통해 생성한 이미지들을 코드화된 설정파일을 사용하여 관리합니다.

   ```yaml
   # create pod on Kubernetes
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx
   spec:
     containers:
     - name: nginx
       image: nginx:1.14.2
       ports:
       - containerPort: 80
   ```

   오케스트레이션 도구 기능

   - 효율적인 하드웨어 활용
   - 다양한 배포 전략을 통한 효율적 업데이트 및 롤백
   - Monitoring and Self Recovery
   - AutoScaling
   - LoadBalancing
   - Service Discovery
     : 서로 다른 네트워크에 있는 리소스 간 통신

5. 프로비저닝 도구

   구성 관리, 서버 템플릿 및 오케스트레이션 도구가 각 서버에서 실행되는 코드를 정의한다면, 프로비저닝 도구는 서버 자체를 생성합니다. 또한 로드밸런서나 서브넷, 방화벽, 라우팅 규칙 등의 대부분의 인프라 리소스들을 생성할 수 있습니다.
   대표적인 툴로는 Terraform, AWS CloudFormation 등이 있습니다.

   ```json
   **provider** "aws" {
     region = "ap-northeast-2"
   }
   
   **resource** "aws_instance" "example" {
     ami                    = "ami-0e9bfdb247cc8de84"
     instance_type          = "t2.micro"
   
     **user_data** = <<-EOF
                 **#!/bin/bash
                 echo "Hello, T101 Study" > index.html
                 nohup busybox httpd -f -p 8080 &**
                 EOF
   
     tags = {
       Name = "terraform-Study-101"
     }
   }
   EOT
   ```

<br>

## IaC의 장점

- 자급식 배포(Self-service) 
  : 배포 프로세스를 자동화 할 수 있으며, 개발자는 필요할 때마다 자체적으로 배포를 진행 할 수 있습니다.
- 속도와 안정성(Speed and safety)
  : 자동화된 프로세스는 사람이 진행하는 것 보다 빠르며, 일관되고 반복 가능하므로 안전합니다.
- 문서화(Documentation)
  : 누구나 읽을 수 있는 소스 파일로 인프라 상태를 나타낼 수 있습니다.
- 버전 관리(Version control)
  : 인프라의 변경 사항이 모두 코드형 소스 파일을 형태로 저장되므로 버전 관리 및 원복이 용이합니다.
- 유효성 검증(Validation)
  : 인프라 상태가 코드로 정의되어 있으면 코드 변경 시 검증 및 일련의 자동화된 테스트를 실행합니다.
- 재사용성(Reuse) 
  : 인프라를 재사용 가능한 모듈로 패키징할 수 있어 검증된 모듈로 일관되게 배포할 수 있습니다.

<br>

참조

- Terraform UP & Running
- 가시다님 스터디 자료
