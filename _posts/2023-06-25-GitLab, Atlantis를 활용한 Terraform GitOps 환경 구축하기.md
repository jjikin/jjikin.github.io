---
title: 2023-06-25-GitLab, Atlantis를 활용한 Terraform GitOps 환경 구축하기
date: 2023-06-25 15:33:44 +09:00
categories: [devops-study, git]
tags: [git, gitlab, atlantis, gitops, terraform, iac]
img_path: /assets/img/posts/
image: /assets/img/posts/
---

스터디간 Terraform 코드에 대한 변경 이력 관리와 협업을 위해 terraform pull request 과정을 자동화해주는 
Atlantis와 ~한 Gitlab을 통해 IaC 의 GitOps? CI/CD 환경을 구축합니다.



각 어카운트간 연동은 취소되어 EKS Cluster를 public


Pull Request는 변경사항에 대한 branch를 생성 후 검토 및 병합을 요청하는 것













깃랩 설치의 최소 사양이 CPU 4 Core, Mem 4GB 이상이라 t3a.xlarge로 생성
amazon linux2

깃랩 설치 
https://about.gitlab.com/install/#amazonlinux-2

알람 이메일을 받으려면 Postfix 설치 및 구성 진행
```shell
sudo yum install -y postfix
```

gitlab 패키지 저장소 추가
```shell
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh
```

편리한 사용을 위해 Route53에서 별도 도메인 생성 후 환경변수로 추가하여 설치
root의 초기 패스워드도 설정 (미설정 시 설치 완료 후 /etc/gitlab/initial_root_password에서 확인)
```shell
sudo yum update
sudo GITLAB_ROOT_PASSWORD="root" EXTERNAL_URL="https://gitlab.jjikin.com" yum install -y gitlab-ee
```


user & private 프로젝트 생성
![](image-20230627204235643.png)

api, read_api scope 부여한 Access Token 생성 glpat-TaktyZyqy_cVdq4ojDt4
![](image-20230627204759036.png)

Root 계정에서 webhook 관련 Outbound request 허용
![](image-20230627212234426.png)

아래 URL을 통해 Random Secret 생성 후 Webhook 생성
https://www.browserling.com/tools/random-string
!!! URL 수정 필요
```
wfZZJK7uGgLDqPRJvdUgFU3xXNqNkyyiIGA039VMHKvHaf77znC0WpSbJ70FIDXo
```
![](image-20230627210210371.png)

~events 로 설정해야함



모든 아틀란티스 설치는 아래 docs
https://www.runatlantis.io/docs

Atlantis 배포

GitLab에서는 Atlantis로 Webhook을 보낼 수 있어야하고,
Atlantis는 GitLab에 API 호출을 할 수 있어야함. (이건 사설망으로)

Atlanstis는 Terraform init시 모듈과 plan을 위한 코드를 로컬 디스크에 저장한다. 모듈 용량이 클경우 디스크양을 적절히 부여해야한다.



helm에 runatlantis helm 차트 저장소 추가
```shell
helm repo add runatlantis https://runatlantis.github.io/helm-charts
```


액세스 토큰, 시크릿 입력을 위한 values.yaml 생성
```
helm inspect values runatlantis/atlantis > values.yaml
```


Webhook를 허용할 리포지토리를 입력
```yaml
...
# Replace this with your own repo allowlist:
orgAllowlist: gitlab.jjikin.com/*
...
```

value.yaml 내에 아래 주석 해제 후 액세스 토큰, 시크릿 입력
```yaml
...
# If using GitLab, specify like the following:
gitlab:
  user: jjikin
  token: glpat-TaktyZyqy_cVdq4ojDt4
  secret: wfZZJK7uGgLDqPRJvdUgFU3xXNqNkyyiIGA039VMHKvHaf77znC0WpSbJ70FIDXo
# GitLab Enterprise only:
  hostname: https://gitlab.jjikin.com
...
```

atlantis 계정 설정
```yaml
basicAuth: # atlantis account info
  username: "atlantis"
  password: "atlantis"
```

ingress 설정
atlantis만을 위한 별도의 ALB 생성은 불필요하므로, 내부 서비스`sockshop` 생성 시 같이 생성했던 ALB를 사용한다.
```yaml
...
ingress:
  enabled: true
  ingressClassName: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/group.name: devops-pub-alb # IngressGroups ALB 공유
    alb.ingress.kubernetes.io/target-type: instance # ALB -> NodePort -> Cluster IP(Port) -> Pod(TargetPort). 파드가 1개라 ip도 상관없긴 함
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
  host: atlantis.jjikin.com
...
```

![](image-20230627231746712.png)


이외 추가할 변수들은 아래 링크에서 확인 후 추가 및 변경
https://github.com/runatlantis/helm-charts#customization

배포
```
kubectl create namespace atlantis
helm install atlantis runatlantis/atlantis -f atlantis_values.yaml -n atlantis
```


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

![](image-20230626111750670.png)
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