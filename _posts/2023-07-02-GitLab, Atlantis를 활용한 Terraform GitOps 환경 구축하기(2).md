---

title: GitLab, Atlantis를 활용한 Terraform GitOps 환경 구축하기(2)
date: 2023-07-02 15:33:44 +09:00
categories: [DevOps, gitlab, atlantis]
tags: [gitlab, atlantis, gitops, terraform, iac]
image: /assets/img/posts/image-20230711012040124.png
---



이번 포스트에서는 구성 완료한 GitLab, Atlantis를 기반으로 Pull Request를 통해 Terraform Code를 배포하는 과정을 설명합니다.
<br>

## Pull Request 

### 사전 설정

기존의 Terraform Code를 배포하는 환경은 로컬PC에서 이루어지도록 구성되어 있습니다.  
따라서 정상적으로 Atlantis에서 Terraform Code를 배포하기 위해서는 몇 가지 설정 사항들을 변경해야합니다.

1. `infra` 코드 내 `profile` 옵션 주석 처리  
   기존 코드 배포는 로컬 PC에 `aws configure` 명령어를 통해 추가한 프로파일을 기반으로 Terraform Code 내 `profile` 옵션을 통해 AWS 리소스를 생성하도록 구성되어 있었습니다.  
   `profile` 옵션을 주석처리하여 Atlantis Pod가 가지고 있는 IRSA의 권한을 기반으로 코드 배포를 진행하도록 변경합니다.

   ```hcl
   # eks.tf
   ...
   ata "terraform_remote_state" "remote" { # VPC State를 가져온다.
     backend = "s3"
     config = {
       #profile        = "devops"   # 주석 처리
       bucket         = "devops-s3-tfstate"
       key            = "devops/terraform.tfstate"
       dynamodb_table = "devops-table-tfstate"
       region         = "ap-northeast-2"
     }
   ...
   ```

   ```hcl
   # main.tf
   provider "aws" {
     #profile = "devops"   # 주석 처리
     region = "ap-northeast-2"
   }
   ...
   ```

   <br>

2. KMS CMK 권한 변경  
   k8s etcd 암복호화를 위한 권한을 위해 CMK 키 관리자를 변경해야 합니다. 기존 프로파일로 설정되어있는 키 관리자를 Atlantis가 사용하는 Role로 변경합니다.

   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "KeyAdministration",
               "Effect": "Allow",
               "Principal": {
                   "AWS": "arn:aws:iam::111111111111:user/ljy"   # 삭제
                   "AWS": "arn:aws:iam::111111111111:role/devops-eks-atlantis-role"   # 추가             
               },
               "Action": [
   						...
   ```

   <br>

3. configmap `aws-auth` 에 Role 추가  
   Atlantis Pod에 부여된 `devops-eks-atlantis-role`을 통해 k8s 내 리소스를 생성 및 변경할 수 있도록 권한을 부여합니다.

   ```yaml
   # aws-auth configmap
   apiVersion: v1
   data:
     mapAccounts: |
       []
     mapRoles: |   # 추가
       - rolearn : arn:aws:iam::111111111111:role/devops-eks-atlantis-role
         username : atlantis
         groups :
           - system:masters
     mapUsers: |
   	- userARN: arn:aws:iam::111111111111:user/hunine8
         username: devops-admin
         groups:
         - system:masters
     ...
   ```



<br>

<br>

### 테스트

GitLab Repository에 소스 코드를 업로드한 직후 아래와 같이 GitLab Repository 상단에서 `Create merge request` 팝업을 확인할 수 있으며, 클릭하여 Pull Request를 진행합니다.
![image-20230716035422517](/assets/img/posts/image-20230716035422517.png)

{: .prompt-info }

> Pull Request와 Merge Request의 차이  
> Pull Request는 GitHub에서, Merge Request는 GitLab에서 사용하는 용어로 같은 의미로 쓰입니다.

<br>

PR의 제목과 설명, 검토자 등을 설정 후 생성합니다.

![image-20230722160553516](/assets/img/posts/image-20230722160553516.png)

<br>

팀원으로부터 Merge를 승인 받았다면 Atlantis가 정상적으로 연동되어있는지 확인하기위해 Activity 탭에 `atlantis help` 명령어를 Comment로 작성합니다.  
(👀 이모티콘 확인)

![image-20230722161032346](/assets/img/posts/image-20230722161032346.png)

<br>

Comment에 대해 Atlantis Bot이 정상적으로 응답하고 있음을 확인할 수 있습니다.  
(GitLab User로 AtlantisBot이 자동 생성됩니다.)

![image-20230722161207871](/assets/img/posts/image-20230722161207871.png)

<br>

`atlantis plan` 을 실행한 후 이상 없음을 확인합니다.

![image-20230722163009998](/assets/img/posts/image-20230722163009998.png)

<br>

`atlantis apply` 명령어를 실행하여 적용합니다.

![image-20230722163438878](/assets/img/posts/image-20230722163438878.png)

<br>

Atlantis Web에서는 `atlantis plan/apply` 간 S3에 저장된 상태 파일에 대한 Locking 여부 확인이 가능합니다.

![image-20230716024957121](/assets/img/posts/image-20230716024957121.png)

<br>

Altantis에서 상태 파일에 대한 Lock을 지원하므로 EKS 구축 시 S3 상태파일 Lock을 위해 생성했던 DynamoDB 리소스를 삭제합니다.

```hcl
# ~/backend/init.tf
module "dynamodb_table" {   # 삭제
  source   = "terraform-aws-modules/dynamodb-table/aws"

  name     = "devops-table-tfstate"
  hash_key = "LockID"
  billing_mode = "PAY_PER_REQUEST"  # On-demand, 요청만큼만 지불하는 방식
  attributes = [
    {
      name ="LockID"
      type = "S"  # String
    }
  ]
}
```

```hcl
# ~/infra/eks.tf
...
data "terraform_remote_state" "remote" { # VPC State를 가져온다.
  backend = "s3"
  config = {
    profile        = "devops"
    bucket         = "devops-s3-tfstate"
    key            = "devops/terraform.tfstate"
    #dynamodb_table = "devops-table-tfstate"   # 삭제
    region         = "ap-northeast-2"
  }

  depends_on = [module.vpc]
}
...
```

```hcl
# ~/infra/main.tf
...
  backend "s3" {
    #profile        = "devops"
    bucket         = "devops-s3-tfstate"
    key            = "devops/terraform.tfstate"
    #dynamodb_table = "devops-table-tfstate"   # 삭제
    region         = "ap-northeast-2"
    encrypt        = true
  }
}
...
```



<br>

다음 포스트 [GitLab, Atlantis를 활용한 Terraform GitOps 환경 구축하기(3)](https://jjikin.com/posts/GitLab,-Atlantis%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-Terraform-GitOps-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0(3)/)에서 이어집니다.
