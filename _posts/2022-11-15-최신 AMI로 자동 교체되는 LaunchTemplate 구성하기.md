---
title: 최신 AMI로 자동 교체되는 Launch Template 구성하기
date: 2022-11-15 00:00:00 +09:00
categories: [AWS, Autoscaling]
tags: [aws, autoscaling, ami, launchtemplate, lambda]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---



## 구성 배경

기존에는 앱푸시, 라이브 방송 등의 이벤트로 인한 트래픽 증가가 예상되는 경우 WEB 인스턴스 타입을 Scale-up / down 하는 방식으로 대응해왔습니다.  
하지만 위 방법은 로드밸런서에서 인스턴스를 분리하고 재시작 하는 과정이 필요해서 번거롭고, 비용적인 측면에서도 효율적이지 않았습니다.  
이후 이벤트가 시간 단위로 반복해서 발생하게 되면서 AutoscalingGroup과 Launch Template을 통해 이벤트 기간에만 예약된 작업 기능을 통해 자동으로 Scale-out / in 할 수 있도록 구성할 예정입니다.

<br>

## 고려 사항

- 고객 요청으로 크기 조정 정책은 구성하지 않습니다.

- 최신 버전으로 배포된 AMI로 Scale-out 되어야 합니다.
  
   {: .prompt-warning }

   > Scale-out 시에는 Launch Template에 설정되어있는 AMI로 인스턴스가 생성됩니다  
  CodeDeploy와 AutoScalingGroup을 통합한 경우 오래된 인스턴스를 최신 상태로 유지하기 위해 자동으로 후속 배포가 진행되는 기능이 존재하지만, CodeDeploy를 사용하지 않는 환경이므로 배포 시 AMI의 수동 변경이 필요합니다.


  <br>

  

## 구성 내용

![Untitled](/assets/img/posts/image-20221115140010490.png)

<br>

### Launch Template 생성

![Untitled](/assets/img/posts/image-20221115140010491.png)

![Untitled](/assets/img/posts/image-20221115140010492.png)

![Untitled](/assets/img/posts/image-20221115140010493.png)

<br>

### AutoScalingGroup 생성

![Untitled](/assets/img/posts/image-20221115140010494.png)

![Untitled](/assets/img/posts/image-20221115140010495.png)

![Untitled](/assets/img/posts/image-20221115140010496.png)

<br>

### Lambda 함수 작성

- Launch Template 내 AMI를 최신으로 자동 교체하기 위한 람다 함수를 작성합니다.

- 일반 구성

  - 함수명 : jjikin-web-lt-ami-replace-automation
  - 런타임 : Python 3.9
  - 제한 시간 : 10초

- 트리거 : EventBridge

- 권한 : jjikin-lt-ami-replace-automation-Role

  ```json
  # jjikin-lt-ami-replace-automation-Policy
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Action": "logs:CreateLogGroup",
              "Resource": "arn:aws:logs:ap-northeast-2:111111111111:*"
          },
          {
              "Effect": "Allow",
              "Action": [
                  "logs:CreateLogStream",
                  "logs:PutLogEvents"
              ],
              "Resource": [
                  "arn:aws:logs:ap-northeast-2:111111111111:log-group:/aws/lambda/jjikin-web-lt-ami-replace-automation:*"
              ]
          },
          {
              "Effect": "Allow",
              "Action": [
                  "ec2:CreateLaunchTemplateVersion",
                  "ec2:ModifyLaunchTemplate",
                  "ec2:GetLaunchTemplateData",
                  "ec2:DescribeLaunchTemplateVersions",
                  "ec2:DescribeLaunchTemplates",
                  "ec2:DescribeImageAttribute",
                  "ec2:DescribeImages"
              ],
              "Resource": "*"
          }
      ]
  }
  ```

- 환경 변수

  ![Untitled](/assets/img/posts/image-20221115140010497.png)

  - EC2_INSTANCE_NAME : AMI의 Name Tag 입니다.
  - SG_IDS : 여러 보안그룹을 입력할 경우 띄어쓰기없이 ,로 구분하여 입력합니다.

- [코드](https://gist.github.com/jjikin/69a5171e1a01f4ec4fa8a0a0408731c3)

  - Line 9~15 : 코드의 재사용성을 위해 시작 템플릿 설정에 필요한 항목들을 환경변수로 처리했습니다.

  - Line 26~40 : AWS Backup을 통해 백업된 AMI는 이름이 ‘AWSbackup_’로 시작하며, 매일마다 1번 생성되므로 CreationDate를 통해 최신 AMI를 구분합니다.

  - Line 45~46 : describe 메소드를 통해 가져온 json을 creation-date 기준으로 내림차순 정렬 후 파싱하도록 작성했는데 이 방식보단 필터에서 creation-date를 추가, json 결과에 1개의 AMI 정보만 출력하는 방식이 더 간단했습니다. 만약 배포로 인해 2개 이상의 백업이 생성되는 경우 시간대도 체크해야하므로 남겨두었습니다.

  - Line 57 : boto3 문서를 참고하여 SecurityGroups를 사용했는데, AutoScalingGroup 생성 시 아래와 같은 오류가 발생했습니다.

    {: .prompt-danger }

    > 1단계에서 잘못된 시작 템플릿 지정: The parameter groupName cannot be used with the parameter subnet
    
    확인 시, SecurityGroups는 default-VPC에서 사용하는 파라미터이며, 그 외 사용자가 생성한 VPC는 **SecurityGroupIds**를 사용해야합니다.

<br>

### EventBridge Rule 생성

- 매일 AM 02:00에 람다 함수를 실행하도록 Rule을 생성합니다.

  ![Untitled](/assets/img/posts/image-20221115140010498.png)

<br>

## 테스트

- 예약된 작업을 생성합니다.

  ![Untitled](/assets/img/posts/image-20221115140010499.png)

- 예약 시간 이후 Scale-out된 인스턴스를 확인하고 테스트합니다.

  ![Untitled](/assets/img/posts/image-20221115140010410.png)

  ![Untitled](/assets/img/posts/image-20221115140010411.png)
