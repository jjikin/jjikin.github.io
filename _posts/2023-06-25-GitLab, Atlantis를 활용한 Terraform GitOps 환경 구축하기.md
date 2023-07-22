---
title: GitLab, Atlantis를 활용한 Terraform GitOps 환경 구축하기
date: 2023-07-01 15:33:44 +09:00
categories: [devops-study, gitlab, atlantis, gitpos, terraform]
tags: [gitlab, atlantis, gitops, terraform, iac]
image: /assets/img/posts/image-20230711012040124.png
---

GitLab과 Terraform Pull Request 과정을 자동화해주는 Atlantis를 활용하여 스터디 간 Terraform 코드에 대한 변경 이력 관리와 협업을 위한 GitOps 환경을 구축합니다. 

{: .prompt-info }

  > Pull Request란?
  > 코드 변경 사항에 대한 branch를 생성하여 팀원들에게 검토 후 main branch에 병합을 요청하는 것

<br>

[Workflow 그려서 넣기]

<br>

## GitLab

GitLab은 지속적 통합/지속적 배포(CI/CD) 및 협업을 위한 여러 기능들을 제공하는 웹 기반 DevOps 플랫폼입니다.  
GitLab Community Edition은 오픈소스로 무료로 사용할 수 있고, SaaS형이 아닌 자체적으로 설치(Self-Managed)해서 사용할 수 있기에 선택하게 되었습니다.



### 설치 사양

GitLab 설치에 필요한 최소 사양은 CPU 4Core + Mem 4GB 이상으로, 이에 맞게 Spec을 산정하여 생성합니다.

- Instance Type : t3a.xlarge(4C/16M)
- AMI : Amazon Linux 2 (kernal 5.10.179-171.711.amzn2.x86_64)
- Storage : 30GiB

<br>

### 설치 방법

설치는 [GitLab Docs](https://about.gitlab.com/install/#amazonlinux-2)를 참고하여 진행했습니다.

1. EC2 Instance 생성 시 설정한 보안그룹에 SSH 및 GitLab 접속을 위한 보안그룹 규칙을 설정합니다.
   ![image-20230713202810944](/assets/img/posts/image-20230713202810944.png)
   <br>

2. GitLab Instance에 Elastic IP를 할당합니다.
   ![image-20230715182553036](/assets/img/posts/image-20230715182553036.png)
   <br>

3. Route53에서 GitLab에 사용할 레코드를 생성한 후 Elastic IP로 라우팅하도록 설정합니다.
    ![image-20230713201840303](/assets/img/posts/image-20230713201840303.png)
    <br>

4. GitLab Instance 접속 후 GitLab 설치를 위한 패키지 저장소를 추가합니다.
   ```bash
   curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
   ```
   <br>

5. Route53에서 추가한 레코드와 root의 초기 패스워드를 환경변수로 추가하여 설치합니다.  
   (패스워드 미설정 시 설치 완료 후 /etc/gitlab/initial_root_password에서 확인해야합니다.)
   ```shell
   sudo yum update
   sudo GITLAB_ROOT_PASSWORD='패스워드 입력' EXTERNAL_URL='https://gitlab.jjikin.com' yum install -y gitlab-ce
   ```
    <br>

6. 설치 완료까지 약간의 시간이 소요됩니다.
   ![image-20230715182731486](/assets/img/posts/image-20230715182731486.png)
   <br>

7. 설정한 도메인 주소와 계정 정보로 GitLab에 접속합니다.
    ![image-20230713211140295](/assets/img/posts/image-20230713211140295.png)
    <br>

8. 사용할 신규 User를 생성한 후 로그인합니다.
    ![image-20230713211815781](/assets/img/posts/image-20230713211815781.png)
   <br>

9. Private Project를 생성합니다.
    ![image-20230713212228814](/assets/img/posts/image-20230713212228814.png)

<br>

### Atlantis 연동을 위한 설정

1. Atlantis에서 GitLab API 호출을 위한 Access Token 생성  
   프로젝트 선택 - Settings - Access Token에서 아래와 같이 입력 후 토큰을 생성하면 상단에 토근값이 출력되며 기록해둡니다.
   ![image-20230713215855683](/assets/img/posts/image-20230713215855683.png)
   <br>

2. Secret 생성  
   GitLab으로부터 수신한 Webhook이 올바른 요청인지 확인하기 위한 Secret Token을 생성해야합니다. 공식 문서에서 제공한 [링크](https://www.browserling.com/tools/random-string)에서 아래 설정으로 Random String을 생성합니다.
   - Format : a-zA-Z mixed case
   
   - Length : 32~128

{: .prompt-warning }
> String에 특수문자가 있거나 28문자보다 짧을 경우 400 Error(Unauthorized & did not match expected secret)가 발생할 수 있습니다.
<br>

3. Webhook 설정  
   생성한 Secret Token을 포함하여 Webhook을 보낼 Atlantis URL과 트리거를 입력합니다.
   ![image-20230715194410691](/assets/img/posts/image-20230715194410691.png)
   <br>

<br>


## Atlantis

Atlantis는 Pull Request를 통해 Terraform Workflow를 자동화해주는 오픈소스 Tool입니다.  
앞으로 진행될 스터디에서 팀원들 간 EKS를 구성하는 Terraform Code의 관리와 협업을 위해 꼭 필요한 툴이기에 선택하였으며, EKS 내 설치합니다.

<br>

### 사전 설정

- Atlantis는 PV를 사용하므로 ebs-csi-driver Add-on 설치가 필요합니다. 아래와 같이 코드 추가 후 재배포 합니다.
  
   ```hcl
   # eks.tf
   module "eks" {
     source  = "terraform-aws-modules/eks/aws"
     ...
     cluster_addons = {
       coredns = {
         most_recent       = true
         resolve_conflicts = "OVERWRITE"
       }
       ...
       aws-ebs-csi-driver = {  # 추가
         most_recent = true
         service_account_role_arn = module.ebs_csi_driver_irsa_role.iam_role_arn
       }
     }
    ...
   }  
   ```
   ```hcl
   # IRSA Module 추가
   module "ebs_csi_driver_irsa_role" {
     source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
    
     role_name                     = "${local.name}-eks-ebs_csi-role"
     policy_name_prefix            = "${local.name}-eks-"  
     attach_ebs_csi_policy = true
     
     oidc_providers = {
       main = {
         provider_arn               = module.eks.oidc_provider_arn
         namespace_service_accounts = ["kube-system:ebs-csi-controller-sa"]
       }
     }
     
     tags = local.tags
   }
   ```

<br>

### 설치 방법
Atlantis는 EKS 내 helm chart를 통해 배포할 예정이며, [Atlantis Docs](https://www.runatlantis.io/docs)를 참고하여 진행했습니다.

1. helm에 runatlantis helm 차트 저장소 추가
   ```shell
   helm repo add runatlantis https://runatlantis.github.io/helm-charts
   ```
   <br>

2. Access Token, Secret 설정을 위한 values.yaml 생성
   ```
   helm inspect values runatlantis/atlantis > atlantis_values.yaml
   ```
   <br>

3. atlantis_value.yaml 파일을 수정합니다.

   - Webhook를 허용할 리포지토리를 입력합니다.
   ```yaml
   # Replace this with your own repo allowlist:
   orgAllowlist: gitlab.jjikin.com/jjikin/devops  # {hostname}/{owner}/{repo}
   ```
   <br>

   - GitLab 연동을 위한 정보를 입력합니다.
   ```yaml
   # If using GitLab, specify like the following:
   gitlab:
     user: jjikin
     token: glpat-****_**************
     secret: ********************************
   # GitLab Enterprise only:
     hostname: https://gitlab.jjikin.com
   ```
   <br>

   - Atlantis에 로그인하기 위한 계정 정보를 설정합니다.
   ```yaml
   basicAuth: # atlantis account info
     username: "atlantis"
     password: "atlantis"
   ```
   <br>

   - ingress 설정  
     Atlantis만을 위한 별도의 ALB 생성은 불필요하므로, 내부 서비스 `sockshop` 생성 시 같이 생성했던 ALB를 사용합니다.
     
     ```yaml
     ingress:
       enabled: true
       ingressClassName: alb
       annotations:
         alb.ingress.kubernetes.io/scheme: internet-facing
         alb.ingress.kubernetes.io/group.name: devops-pub-alb # IngressGroups ALB 공유
         alb.ingress.kubernetes.io/target-type: instance
         alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
         alb.ingress.kubernetes.io/ssl-redirect: '443'
       host: atlantis.jjikin.com
       
       ## in case we need several hosts:
       hosts:
       #   - host: chart-example.local
       #     paths: ["/"]
       #     service: chart-example1
       #   - host: chart-example.local2
       #     service: chart-example1
       #     paths: ["/lala"]
       tls: []
       #  - secretName: chart-example-tls
       #    hosts:
       #      - chart-example.local
       labels: {}
     
     ```
     <br>
   
   - PV 설정  
     Atlanstis는 `Terraform init` 실행 시 필요한 Module을 PV에 저장합니다. 모듈 용량이 클 경우 용량 부족으로 에러가 발생할 수 있으므로 디스크 용량을 적절하게 부여해야합니다.
   
     ```yaml
     volumeClaim:
       enabled: true
       ## Disk space for to check out repositories
       dataStorage: 20Gi
       storageClassName: gp2
     ```
     <br>
   
   - ServiceAccount 설정
     ```yaml
     serviceAccount:
       create: true
       mount: true
       name: runatlantis
       annotations: 
         eks.amazonaws.com/role-arn: "arn:aws:iam::371604478497:role/devops-atlantis-role" # 직접 설정 필요
     ```
     <br>
   
   - 이외 추가할 변수들은 [링크](https://github.com/runatlantis/helm-charts#customization)를 통해 확인 후 추가 및 변경합니다.
     <br>
   
4. Helm을 통한 배포
   ```bash
   kubectl create namespace atlantis
   helm install atlantis runatlantis/atlantis -f atlantis_values.yaml -n atlantis
   ```

<br>



## Terraform Code 업로드



코드에 assume_role 추가

https://www.runatlantis.io/docs/provider-credentials.html#assume-role-session-names

https://www.runatlantis.io/docs/provider-credentials.html

https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-files.html

GitLab에 생성한 Repository에 Terraform Code를 업로드 합니다.

```shell
# Terraform Code가 존재하는 디렉토리로 이동
cd ~/Documents/Study/devops-study/v3(gitlab+atlantis)/infra

git init
git add .
git status
git commit -m 'init'
git remote add origin https://gitlab.jjikin.com/jjikin/devops.git
git branch -M main
git push -u origin +main
```



Push 설정 Protected 설정 branch 스샷

![image-20230716014840309](/Users/mzc01-ljyoon/Documents/blog/jjikin.github.io/assets/img/posts/image-20230716014840309.png)

altantis에 설정한 IRSA 은 ? -> atlantis-pod에다가 Admin 권한을 준 것.

따라서 올린 코드에 설정한 `profile=devops`를 삭제해야 pod가 가지고 있는 권한을 기반으로 `terraform init`을 실행한다.

![image-20230716035422517](/Users/mzc01-ljyoon/Documents/blog/jjikin.github.io/assets/img/posts/image-20230716035422517.png)

실행 후 에러 발생

```plaintext
│ Error: reading KMS Key (77270bdb-91e1-4576-ae99-46bcffb63a3a): reading KMS Key (77270bdb-91e1-4576-ae99-46bcffb63a3a): AccessDeniedException: User: arn:aws:sts::371604478497:assumed-role/devops-eks-atlantis-role/1689432777102819771 is not authorized to perform: kms:DescribeKey on resource: arn:aws:kms:ap-northeast-2:371604478497:key/77270bdb-91e1-4576-ae99-46bcffb63a3a because no resource-based policy allows the kms:DescribeKey action
│ 	status code: 400, request id: dc59db19-6d9b-4ed7-adfd-22645248b6a6
│ 
│   with module.eks.module.kms.aws_kms_key.this[0],
│   on .terraform/modules/eks.kms/main.tf line 8, in resource "aws_kms_key" "this":
│    8: resource "aws_kms_key" "this" {
```



kms 키 정책 확인 후 변경 -> root 로 

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "KeyAdministration",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::371604478497:user/ljy"
            },



pull request 후  atlantis plan 시 아래 에러 발생

```plaintext
Error: Unauthorized
│ 
│   with kubernetes_service_account.external-dns,
│   on eks.tf line 401, in resource "kubernetes_service_account" "external-dns":
│  401: resource "kubernetes_service_account" "external-dns" {
```



AWS -> EKS 내 리소스를 생성할 때 권한 오류라면 aws-auth configmap을 봐야한다.

```yaml
apiVersion: v1
data:
  mapAccounts: |
    []
  mapRoles: |
    - "groups":
      - "system:bootstrappers"
      - "system:nodes"
      "rolearn": "arn:aws:iam::371604478497:role/devops-eks-node-role"
      "username": "system:node:{{EC2PrivateDNSName}}"
    - rolearn : arn:aws:iam::371604478497:role/devops-eks-atlantis-role
      username : atlantis
      groups :
        - system:masters
  mapUsers: |
    []
```



위와 같이 추가 후 plan 성공.



plan 및 apply에 꽤 많은 시간이 소요되지 조금 기다려야함



apply 성공.

![image-20230716024854068](/Users/mzc01-ljyoon/Documents/blog/jjikin.github.io/assets/img/posts/image-20230716024854068.png)

![image-20230716024957121](/Users/mzc01-ljyoon/Documents/blog/jjikin.github.io/assets/img/posts/image-20230716024957121.png)

--- 이하 작성 중 ---

GitLab에 Terraform code 업로드

```shell
cd ~/devops-study/terraform/infra

git init
vim .gitignore # .DS_Store, .terraform* 제외
git branch -m main # master -> main
git add .
git commit -m 'terraform code upload'
git remote add origin https://gitlab.jjikin.com/jjikin/devops.git

git pull origin main --allow-unrelated-histories
git push origin main
```






---

에러! atlantis pod가 pending.
Watitng 관련 에러 

describe 시 볼륨할당 안됨. vpc ebs 드라이버 깔아서 ebs 할당해줘야함.
EKS 1.23버전 이후부터는 **AWS EBS CSI Driver**([https://github.com/kubernetes-sigs/aws-ebs-csi-driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver))를 설치 하지 않으면 EBS를 PV로 사용할 수 없습니다.

 mzc01-ljyoon  ~/Documents/Study/devops-study/terraform/service/deploy/kubernetes  kubectl get pvc
NAME                       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
atlantis-data-atlantis-0   Pending                                      gp2            83m





ebs csi 드라이버 설치

```hcl
module "ebs_csi_driver_irsa_role" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name          = "${local.name}-ebs_csi_driver-role"
  policy_name_prefix = "devops-eks-"  
  attach_ebs_csi_policy = true  # 이 Input을 기준으로 목적에 맞는 Role이 생성됨.

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-ebs-csi-driver"]
    }
  }

  tags = local.tags
}

resource "kubernetes_service_account" "aws-ebs-csi-driver" {
  metadata {
    name        = "aws-ebs-csi-driver"
    namespace   = "kube-system"
    annotations = {
      "eks.amazonaws.com/role-arn" = module.ebs_csi_driver_irsa_role.iam_role_arn  # irsa 생성 모듈에서 output으로 iam_role_arn을 제공한다.
    }
  }

  depends_on = [module.ebs_csi_driver_irsa_role]
}

resource "helm_release" "aws-ebs-csi-driver" {
  name       = "aws-ebs-csi-driver"
  namespace  = "kube-system"
  repository = "https://kubernetes-sigs.github.io/aws-ebs-csi-driver"
  chart      = "aws-ebs-csi-driver"

  set {
    name = "serviceAccount.create"
    value = false
  }
  set {
    name = "serviceAccount.name"
    value = "aws-ebs-csi-driver"
  }  
}
```


설치후에도 

failed to provision volume with StorageClass "gp2": rpc error: code = Internal desc = Could not create volume "pvc-151141cd-0e83-4273-bcc0-acce7022a014": could not create volume in EC2: UnauthorizedOperation: You are not authorized to perform this operation.

에러 내용
Could not create volume "pvc-4d2f8505-76d1-4fd9-aff5-12af75a14c95": could not create volume in EC2: WebIdentityErr: failed to retrieve credentials  caused by: AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity

-> SA 문제 였음 


적절한 label을 부여하면 ebs-csi-controller-sa가 생성되는지 테스트 
-> 그래도 생성됨 ㅠ

에러 모음 
Events:
  Type     Reason            Age   From               Message

----     ------            ----  ----               -------

  Warning  FailedScheduling  62s   default-scheduler  0/2 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/2 nodes are available: 2 No preemption victims found for incoming pod..

kubectl get pvc -A
NAMESPACE   NAME                       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
atlantis    atlantis-data-atlantis-0   Pending                                      atlantis-sc    72s

value.yaml 내 StorageClassName을 별도로 작성해도 자동으로 생성해주지 않는다.
아래 sc와 같이 eks 생성 시 기본적으로 생성되는 sc(gp2)를 사용하거나 별도로 생성해야한다.
kubectl get sc
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  36m




  Warning  ProvisioningFailed    3s                ebs.csi.aws.com_ebs-csi-controller-868956d75-2hxgf_acc84c1f-2ca5-45af-b70a-2dcf14e8f60e  failed to provision volume with StorageClass "gp2": rpc error: code = Internal desc = Could not create volume "pvc-ccd339eb-ab56-463d-82f2-ae0a13834b34": could not create volume in EC2: WebIdentityErr: failed to retrieve credentials
caused by: AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity
  status code: 403, request id: 0ad800fb-63be-4f87-84d5-32810a0f4957

-> atlantis의 SA와, IAM Role(Admin권한)는 문제가 없다.

-> ebs-csi 자체에서 권한이 없어 ebs를 생성하고 있지 못하는 상태임

당연히 내가만든 aws-ebs-csi-driver를 썼는데, 이게 아니라 자동으로만들어진 ebs-csi-controller-sa를 사용해야함

```yaml
# ebs-csi-controller-sa (자동 생성된 SA)
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::371604478497:role/devops-eks-ebs_csi_driver-role
  creationTimestamp: "2023-06-25T15:18:26Z"
  labels:
    app.kubernetes.io/component: csi-driver
    app.kubernetes.io/managed-by: EKS
    app.kubernetes.io/name: aws-ebs-csi-driver
    app.kubernetes.io/version: 1.19.0
  name: ebs-csi-controller-sa
  namespace: kube-system
  resourceVersion: "1632"
  uid: 5df5555a-2adf-4071-a334-054272a0ea11
```

```yaml
# aws-ebs-csi-driver(내가 생성한 SA)
apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::371604478497:role/devops-eks-ebs_csi_driver-role
  creationTimestamp: "2023-06-25T15:15:58Z"
  name: aws-ebs-csi-driver
  namespace: kube-system
  resourceVersion: "960"
  uid: 7c341671-d71e-4cd3-9dce-0f872f6a362a
```

차이점은 labels 부분인데, 직접 label을 추가후 테스트해도 여전히 권한 에러가 발생함.

왜 내가 생성한 sa를 안쓰고 자동 생성한 sa를 바라보는건지 궁금.

ebs csi 관련 문서를 찾아보니 아래와 같은 정보를 확인할 수 있었습니다.

https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/csi-iam-role.html
Amazon EBS CSI 드라이버 IAM 역할을 생성했으므로 [Amazon EBS CSI 추가 기능 추가](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/managing-ebs-csi.html#adding-ebs-csi-eks-add-on)로 진행할 수 있습니다. 해당 절차에서 플러그인이 배포되면 `ebs-csi-controller-sa`라는 서비스 계정을 생성하고 해당 계정을 사용하도록 구성됩니다. 이 서비스 계정은 필요한 Kubernetes 권한이 할당되어 있는 Kubernetes `clusterrole`에 바인딩됩니다.

자동으로 생성되는 서비스 계정을 기준으로 ebs-csi 관련 pod들이 올라갔는데인데, 테라폼 모듈로 이걸 만들어버리면 아래와 같이 labels에서만 내가 생성한 aws-ebs-csi-driver 가 붙고,
실제 controller 파드에 설정된 SA는 ebs-csi-controller-sa다...

```yaml
# kubectl edit pod -n kube-system ebs-csi-controller-868956d75-2hxgf
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2023-06-25T15:18:27Z"
  generateName: ebs-csi-controller-868956d75-
  labels:
    app: ebs-csi-controller
    app.kubernetes.io/component: csi-driver
    app.kubernetes.io/managed-by: EKS
    app.kubernetes.io/name: aws-ebs-csi-driver
    app.kubernetes.io/version: 1.19.0
    pod-template-hash: 868956d75
  name: ebs-csi-controller-868956d75-2hxgf
  namespace: kube-system
  ...
  serviceAccount: ebs-csi-controller-sa
  serviceAccountName: ebs-csi-controller-sa
  terminationGracePeriodSeconds: 30
  tolerations:
```

위와 같이 ebs-csi-controller-sa로 설정되어있는데, SA가 IAM Role을 통해 ebs를 생성하도록하는 IRSA 설정은 모듈로 인해 내가 생성한 aws-ebs-csi-driver가 설정되어 권한 에러가 발생한것임.

정말 이 부분이 원인인지 검증하기 위해 cluster- add-on이 아닌 helm을 통해 설치
설치해도 sa는 자동생성된걸로 나온다.

helm release 등 관련 문서를 찾아봐도 다른 플러그인고 ㅏ달리 service account 를 설정할 수가 없네
즉 별도로 변경할 수 없는 거 같음

![](/Users/mzc01-ljyoon/Documents/blog/image-20230626111750670.png)
https://malwareanalysis.tistory.com/598







직접 헬름 으로 설치해보기 -> terraform으로 자동설치하기 순으로 



신규 IAM User에 대한 k8s api 호출 허용하려면? 
aws-auth configmap을 수정해줘야함 

```
# aws-auth configmap
apiVersion: v1
data:
  mapAccounts: |
    []
  mapRoles: |
    ...
  mapUsers: |
	- userARN: arn:aws:iam::371604478497:user/hunine83@gmail.com
      username: devops-admin-1
      groups:
      - system:masters
    - userARN: arn:aws:iam::371604478497:user/andy741023@gmail.com
      username: devops-admin-2
      groups:
      - system:masters
```







https://github.com/kubernetes-sigs/aws-ebs-csi-driver/issues/1033
그냥 Add-on으로 설치하면 편리함

워크플로우

Terrform 코드 변경 후 GitLab push하여 신규 Branch 생성
검토 후 branch를 main branch로 merge 요청
이를 기준으로 terraform plan 수행
검토후 승인 altantis apply 적용
apply 완료 되면 main merge 완료 


GLa7sCj/xC/si3HSFUQURR+He2OcEeqLMtatGoKZ7to=


설치 위치 EC2 VS k8s

k8s 내에 설치하면 k8s 관련 지식이 요구되며, 지금과 같은 소규모의 스터디용도로는 적합하지 않고 대규모 배포 등에 더 적합함. 일반 설치보다 더 많은 리소스가 필요하고

깃랩을 사설 레포용도로만 사용할 예정 깃랩 러너가 별로다, 깃랩을 쓰는 이유는 커뮤니티에디션이 무료라서



s3, dynamodb + atlantis vs terraform cloud 


atlantis 설치
https://www.runatlantis.io/docs/installation-guide.html

꼭 mgmt 노드에 atlantis 리소스 넣기 노드셀렉터든 어피니티든


참고사이트
https://medium.com/nerd-for-tech/terraforming-the-gitops-way-9417cf4abf58


https://joshkodroff.com/post/2018_07_04_ci_cd_pipeline_terraform_atlantis/

https://techblog.yogiyo.co.kr/gitops-%EA%B8%B0%EB%B0%98%EC%9D%98-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0-1%EB%B6%80-terraform-cloud-github-action-%EC%A0%81%EC%9A%A9-92a0a0ffcba0





Root 계정에서 webhook 관련 Outbound request 허용
![](/Users/mzc01-ljyoon/Documents/blog/image-20230627212234426.png)



altnatis error

kubectl logs atlantis-0 -n atlantis
`Error: initializing server: GET https://gitlab.com/api/v4/version: 401 {message: 401 Unauthorized}`



-> 권한문제인줄알고 Repository 토큰 권한 권한설정만 봤었는데, atlantis 설정에

\#GitLab Enterprise only: 부분을 주석처리해줘야한다. 커뮤니티 버전을 사용한다면





atlantis 파드가 ALB TargetGroup에 Binding 되지 않는 문제



AWS LoadBlancer Controller 파드의 로그 확인

```shell
{"level":"error","ts":"2023-07-15T12:12:00Z","msg":"Reconciler error","controller":"targetGroupBinding","controllerGroup":"elbv2.k8s.aws","controllerKind":"TargetGroupBinding","TargetGroupBinding":{"name":"k8s-atlantis-atlantis-cfb44d0014","namespace":"atlantis"},"namespace":"atlantis","name":"k8s-atlantis-atlantis-cfb44d0014","reconcileID":"eb007d1f-eaa7-4878-9991-dd46a6f243e4",

"error":"expect exactly one securityGroup tagged with kubernetes.io/cluster/devops-eks-cluster for eni eni-0d0d20b446f9a678b, got: [sg-06fdd3101d57d1a06 sg-08dc8e319a2fb64f0] (clusterName: devops-eks-cluster)"}
```



https://github.com/kubernetes-sigs/aws-load-balancer-controller/issues/1897



첫 EKS 생성을 위한 Terraform Code 작성 당시 [링크](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/alb-ingress.html)의 내용처럼 보안 그룹에 태깅이 요구되어 추가했었는데, 

다시 확인해보니 노드에 클러스터 보안그룹이 자동으로 할당되며, 이 보안그룹에 해당 태그가 이미 할당되어있다.

노드에 연결된 보안그룹에 해당 태그는 1개만 있어야하므로, 임의로 추가했던 remote_access용에서 태그를 삭제했다.

```
resource "aws_security_group" "remote_access" {
  name = "${local.name}-eks-remote_access-sg"
  description = "Allow remote SSH access"
  vpc_id      = local.vpc_id

  ingress {
    description = "SSH access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    "CreatedBy" = "Terraform",
    "kubernetes.io/cluster/devops-eks-cluster" = "owned"  # AWS LB Controller 사용을 위한 요구 사항
  }
}
```





atlantis targetgroup에서 401 unhealthy 발생 건

https://github.com/runatlantis/helm-charts/issues/106





route 53 도메인이 자꾸 바뀌는 문제

팀원분이 다른 클러스터의 exteral-dns에서 서로 업데이트하면서 레코드가 삭제 생성이 반복되었음

txt ownerid를 변경하여 해결

