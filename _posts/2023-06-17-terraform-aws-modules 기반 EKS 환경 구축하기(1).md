---
title: terraform-aws-modules 기반 EKS 환경 구축하기(1)
date: 2023-06-17 00:00:00 +09:00
categories: [DevOps, eks]
tags: [aws, eks, kubenetes, k8s, terraform, iac, module]
image: /assets/img/posts/image-20230711103355989.png
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

Devops 스터디를 위한 EKS 환경을 모듈([terraform-aws-modules](https://github.com/terraform-aws-modules))을 활용하여 구축합니다.

테라폼 관련 github 커뮤니티에서 AWS 리소스를 생성할 때 필요한 테라폼 코드를 모듈화하여 제공하고 있어, 이를 활용하면 직접 코드를 작성하는 것 보다 간편하고 빠르게 리소스를 생성할 수 있습니다.

 

필요한 모듈을 검색하고,  
README.md와 Repository 내 example 디렉토리 내 코드를 참고하여 리소스 생성에 필요한 `Input`을 입력하면,  
리소스를 포함하여  `Output` 까지 자동으로 생성해주므로 특별한 케이스를 제외하면 별도로 정의할 필요가 없습니다.

<br>

<br>

## Terraform Code

### 레이아웃 구성

생성할 리소스가 많거나 복잡하지 않아 리소스별 상태 파일 격리는 별도로 하지 않았습니다.

```
devops-study
└── terraform
    ├── backend
    │   └── init.tf
    ├── infra
    │   ├── main.tf
    │   ├── vpc.tf
    │   └── eks.tf
    └── service
        ├── deploy
        │   └── kubernetes
        ├── ...
```

<br>

### init.tf

상태 파일 저장을 위한 원격 Backend는 S3와 DynamoDB를 통해 구성했습니다.   
Backend 구성 및 상태 저장을 위한 리소스는 미리 생성이 필요하므로, 디렉토리(backend) 및 파일을 별도로 분리했습니다.   
S3 및 DynamoDB 리소스 또한 terraform-aws-modules를 사용하여 생성했습니다.

<details markdown="1">
  <summary>코드 접기/펼치기</summary>


```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0.0"
    }
  }
}

provider "aws" {
  profile = "devops"
  region = "us-east-1"
}

module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "devops-s3-tfstate"
  acl    = "private"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}

module "dynamodb_table" {
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

</details>

<br>

### main.tf

프로바이더 정의와 상태 파일 저장을 위한 Backend, Local 변수를 정의합니다.

<details markdown="1">
  <summary>코드 접기/펼치기</summary>


```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0.0"
    }
  }

  backend "s3" {
    profile        = "devops"
    bucket         = "devops-s3-tfstate"
    key            = "devops/terraform.tfstate"
    dynamodb_table = "devops-table-tfstate"
    region         = "us-east-1"
    encrypt        = true
  }
}

provider "aws" {
  profile = "devops"
  region = "us-east-1"
}

locals {
  name              = "devops"
  vpc_id            = module.vpc.vpc_id
  subnet_ids        = module.vpc.public_subnets
  external_dns_arn  = "arn:aws:route53:::hostedzone/Z08574211BOF867DLRAI2"  # 개인용 Route53 HostingZone
  external_cert_arn = "arn:aws:acm:us-east-1:111111111111:certificate/725fd9d7-5e31-4750-a161-4f67cd6bb9f0"
  tags = {
    CreatedBy = "Terraform"
  }
}
```

</details>

<br>

### vpc.tf

여러 어카운트 간 리소스 연동이 필요하므로, 편의상 모든 리소스는 퍼블릭 영역에 생성합니다.

<details markdown="1">
  <summary>코드 접기/펼치기</summary>


```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "devops-vpc"
  cidr = "192.168.0.0/16"

  azs              = ["us-east-1a", "us-east-1c"]
  public_subnets   = ["192.168.0.0/20", "192.168.16.0/20"]
  public_subnet_names = ["devops-pub-a-sn", "devops-pub-c-sn"]
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1  # 해당 태그 지정 시, k8s 내에서 ingress 생성 시 서브넷 자동 지정
  }

  enable_nat_gateway = false
  enable_dns_hostnames = true
  enable_dns_support = true
  map_public_ip_on_launch = true  # 퍼블릭 서브넷 내 생성되는 리소스에 자동으로 퍼블릭 IP를 할당한다.

  tags = {
    CreatedBy = "Terraform"
  }
}

output "vpc_id" {
  description = "The ID of the VPC"
  value       = module.vpc.vpc_id
}

output "public_subnets" {
  description = "List of IDs of public subnets"
  value       = module.vpc.public_subnets
}
```

</details>

<br>

### eks.tf

EKS 버전은 2023.06 기준 최신 버전인 v1.27으로 설정했으며, 편의 및 서비스 구분을 위해 관리형 노드 그룹 app, mgmt 2개로 구분했습니다.  
app 그룹의 노드에는 서비스 리소스, mgmt 그룹의 노드에는 devops 관련 툴(Atlantis, argoCD 등) 리소스를 할당하도록 NodeSeletor를 사용했습니다.

k8s 내부 통신을 위한 컴포넌트(kubenet, kube-proxy, coredns)는 EKS에서 기본적으로 제공하는 Add-on으로 구성하고 이외 컴포넌트(loadbalancer-controller, external-dns)는 Helm을 통해 설치합니다.

EKS에서 k8s 내 리소스에 접근할 수 있는 권한을 부여하기 위해 configmap(aws-auth)을 자동으로 생성하도록 정의하고, 반대의 경우 AWS에서 제공하는 OIDC Provider를 구성, IRSA를 사용하여 k8s 내 Pod에 IAM Role을 할당합니다.

내부 서비스는 ALB를 통해 제공합니다.  
k8s Ingress와 ALB 연동을 위해 loadbalancer-controller 컴포넌트를 설치 및 구성하고, HTTPS 통신을 위해 별도의 인증서 생성하여 적용했습니다.

서비스 접근을 위한 주소는 기본적으로 ALB의 DNS주소를 사용하므로, 별도 도메인을 통해 접근할 수 있도록 externl-dns 컴포넌트 설치 후 Route53과 연동했습니다.

<details markdown="1">
  <summary>코드 접기/펼치기</summary>

```hcl
# Terraform에서 k8s에 접근할 수 있도록 인증 정보를 제공한다.
provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  token                  = data.aws_eks_cluster_auth.eks.token
}
# Terraform에서 helm을 통해 k8s 내 Add-on를 설치할 수 있도록 인증 정보를 제공한다.
provider "helm" {
  kubernetes {
    host                   = module.eks.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
    token                  = data.aws_eks_cluster_auth.eks.token
  }
}

data "aws_availability_zones" "available" {}

# Terraform에서 AWS의 계정 ID를 참조하기 위해 정의한다. 사용은 ${data.aws_caller_identity.current.account_id}
data "aws_caller_identity" "current" {}

# EKS 클러스터와 통신하기 위한 인증 토큰을 가져온다.
data "aws_eks_cluster_auth" "eks" {name = module.eks.cluster_name}

# VPC 상태를 가져온다.
data "terraform_remote_state" "remote" {
  backend = "s3"
  config = {
    profile = "devops"
    bucket         = "devops-s3-tfstate"
    key            = "devops/terraform.tfstate"
    dynamodb_table = "devops-table-tfstate"
    region         = "us-east-1"
  }
}

locals {
  name              = "devops-eks"
  vpc_id            = data.terraform_remote_state.remote.outputs.vpc_id
  subnet_ids        = data.terraform_remote_state.remote.outputs.public_subnets
  external_dns_arn  = "arn:aws:route53:::hostedzone/Z08574211BOF867DLRAI2"  # 개인용 Route53 HostingZone
  external_cert_arn = "arn:aws:acm:us-east-1:111111111111:certificate/725fd9d7-5e31-4750-a161-4f67cd6bb9f0"
  tags = {
    CreatedBy = "Terraform"
  }
}

################################################################################
### EKS Module
################################################################################
module "eks" {
  source  = "terraform-aws-modules/eks/aws"

  cluster_name                   = "${local.name}-cluster"
  cluster_version                = 1.27
  cluster_endpoint_public_access = true

  # EKS Add-On 정의
  cluster_addons = {
    coredns = {
      most_recent       = true
      resolve_conflicts = "OVERWRITE"
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent              = true
      before_compute           = true  # 워커 노드가 프로비저닝되기 전 vpc-cni가 배포되어야한다. 배포 전 워커 노드가 프로비저닝 되면 파드 IP 할당 이슈 발생
      resolve_conflicts        = "OVERWRITE"
      service_account_role_arn = module.vpc_cni_irsa_role.iam_role_arn  # IRSA(k8s ServiceAccount에 IAM 역할을 사용한다)
      configuration_values     = jsonencode({
        env = {
          ENABLE_PREFIX_DELEGATION = "true"  # prefix assignment mode 활성화
          WARM_PREFIX_TARGET       = "1"  # 기본 권장 값
        }
      })
    }
  }

  vpc_id     = local.vpc_id
  subnet_ids = local.subnet_ids

  # aws-auth configmap
  manage_aws_auth_configmap = true  # AWS -> EKS 접근을위한 configmap 자동 생성

  # 관리형 노드그룹에 사용할 공통 사항 정의
  eks_managed_node_group_defaults = {
    ami_type                   = "AL2_x86_64"
    instance_types             = ["t3.medium"]
    capacity_type              = "SPOT"
    iam_role_attach_cni_policy = true
    use_name_prefix            = false  # false하지 않으면 리소스 이름 뒤 임의의 난수값이 추가되어 생성됨
    use_custom_launch_template = false  # AWS EKS 관리 노드 그룹에서 제공하는 기본 템플릿을 사용
    block_device_mappings = {
      xvda = {
        device_name = "/dev/xvda"
        ebs = {
          volume_size           = 30
          volume_type           = "gp3"
          delete_on_termination = true
        }
      }
    }
    remote_access = {  # Remote access cannot be specified with a launch template
      ec2_ssh_key               = module.key_pair.key_pair_name
      source_security_group_ids = [aws_security_group.remote_access.id]
      tags = {
        "kubernetes.io/cluster/devops-eks-cluster" = "owned"  # AWS LB Controller 사용을 위한 요구 사항
      }
    }

    tags = local.tags
  }

  # 관리형 노드 그룹 정의
  eks_managed_node_groups = {
    devops-eks-app-ng = {
      name         = "${local.name}-app-ng"
      labels = {
        nodegroup = "app"
      }
      desired_size = 1
      min_size     = 1
      max_size     = 1
    }

    devops-eks-mgmt-ng = {
      name         = "${local.name}-mgmt-ng"
      labels = {
        nodegroup = "mgmt"
      }      
      desired_size = 1
      min_size     = 1
      max_size     = 1
    } 
  }
}

# 각종 Add-on에 필요한 IRSA 생성해주는 모듈
# https://github.com/terraform-aws-modules/terraform-aws-iam/tree/master/modules/iam-role-for-service-accounts-eks
module "vpc_cni_irsa_role" { 
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name             = "${local.name}-vpc-cni-irsa-role"
  attach_vpc_cni_policy = true
  vpc_cni_enable_ipv4   = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-node"]
    }
  }

  tags = local.tags
}

module "load_balancer_controller_irsa_role" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name                              = "${local.name}-lb-controller-irsa-role"
  attach_load_balancer_controller_policy = true  # 이 Input을 기준으로 목적에 맞는 Role이 생성됨.
  
  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }

  tags = local.tags
}

module "load_balancer_controller_targetgroup_binding_only_irsa_role" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name = "${local.name}-lb-controller-tg-binding-only-irsa-role"
  attach_load_balancer_controller_targetgroup_binding_only_policy = true  # 이 Input을 기준으로 목적에 맞는 Role이 생성됨.

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }

  tags = local.tags
}

module "external_dns_irsa_role" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name                     = "${local.name}-externaldns-irsa-role"
  attach_external_dns_policy    = true  # 이 Input을 기준으로 목적에 맞는 Role이 생성됨.
  external_dns_hosted_zone_arns = [local.external_dns_arn]

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:external-dns"]
    }
  }

  tags = local.tags
}


module "key_pair" {
  source  = "terraform-aws-modules/key-pair/aws"
  version = "~> 2.0"

  key_name_prefix    = "devops-eks-cluster"
  create_private_key = true
}

resource "aws_security_group" "remote_access" {
  name_prefix = "${local.name}-cluster-remote-access"
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
}

resource "kubernetes_service_account" "aws-load-balancer-controller" {
  metadata {
    name        = "aws-load-balancer-controller"
    namespace   = "kube-system"
    annotations = {
      "eks.amazonaws.com/role-arn" = module.load_balancer_controller_irsa_role.iam_role_arn  # irsa 생성 모듈에서 output으로 iam_role_arn을 제공한다.
    }

    labels = {
      "app.kubernetes.io/component" = "controller"
      "app.kubernetes.io/name" = "aws-load-balancer-controller"
    }

  }

  depends_on = [module.load_balancer_controller_irsa_role]
}

resource "kubernetes_service_account" "external-dns" {
  metadata {
    name        = "external-dns"
    namespace   = "kube-system"
    annotations = {
      "eks.amazonaws.com/role-arn" = module.external_dns_irsa_role.iam_role_arn
    }
  }

  depends_on = [module.external_dns_irsa_role]
}

### Helm
# https://github.com/GSA/terraform-kubernetes-aws-load-balancer-controller/blob/main/main.tf
# https://registry.terraform.io/providers/hashicorp/helm/latest/docs/resources/release
# https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.5/
resource "helm_release" "aws-load-balancer-controller" {
  name       = "aws-load-balancer-controller"
  namespace  = "kube-system"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"

  set {
    name = "clusterName"
    value = module.eks.cluster_name
  }
  set {
    name = "serviceAccount.create"
    value = false
  }
  set {
    name = "serviceAccount.name"
    value = "aws-load-balancer-controller"
  }  

  #depends_on = [kubernetes_service_account.aws-load-balancer-controller]
}

# https://tech.polyconseil.fr/external-dns-helm-terraform.html
# parameter https://github.com/kubernetes-sigs/external-dns/tree/master/charts/external-dns
resource "helm_release" "external_dns" {
  name       = "external-dns"
  namespace  = "kube-system"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "external-dns"
  wait       = false  ## 서비스가 완전히 올라올때 까지 대기
  set {
    name = "provider"
    value = "aws"
  }
  set {
    name = "serviceAccount.create"
    value = false
  }
  set {
    name = "serviceAccount.name"
    value = "external-dns"
  }
  set {
    name  = "policy"
    value = "sync"
  }     
}
```
</details>

<br>

## 내부 서비스(Sockshop)

![image-20230619182858040](/assets/img/posts/image-20230619182858040.png)

EKS에서 제공하는 서비스는 Weaveworks에서 무료로 제공하는 마이크로서비스 데모 애플리케이션인 [Sockshop](https://microservices-demo.github.io/)을 사용했습니다.

소스 코드는 이후 GitOps 기반 CI/CD 파이프라인을 구성하고 이를 통해 배포할 예정이므로, 지금은 수동으로 EKS에 배포합니다.

소스 코드 중 변경한 사항은 아래와 같습니다.

- 내부 서비스 리소스들을 정의하는 yaml 파일(complete-demo.yaml)

  - 각 Deployment에 NodeSelector 설정
    ```yaml
    ...
    nodeSelector:
      #beta.kubernetes.io/os: linux # 주석 처리
      nodegroup: app
    ```

  - Ingress 설정
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: alb-ingress
      namespace: sock-shop
      annotations:
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/load-balancer-name: devops-pub-alb
        alb.ingress.kubernetes.io/certificate-arn: "별도 생성 후 arn 입력"
        alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
        alb.ingress.kubernetes.io/ssl-redirect: '443'
    spec:
      ingressClassName: alb
      rules:
      - host: devops.jjikin.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: front-end
                port:
                  number: 80
    ```
    

<br>

<br>

## 구성 방법

### 환경 구성

1. Terraform 설치

   ```shell
   $ brew tap hashicorp/tap
   $ brew install hashicorp/tap/terraform
   ```

2. aws configure 명령어를 통해 프로파일 추가
   ```shell
   $ aws configure --profile devops
   AWS Access Key ID [None]: ******
   AWS Secret Access Key [None]: ******
   Default region name [None]: us-east-1
   Default output format [None]: json
   ```

3. Terraform 코드 `eks.tf` 내 local 변수 값 `external_dns_arn` 에 Route53 호스팅영역 ARN 변경

4. service > deploy > kubernetes > `complete-demo.yaml` 내 인증서 ARN, Domain 변경

   ![image-20230621125926221](/assets/img/posts/image-20230621125926221.png)

   <br>

### 인프라 생성

1. backend 디렉토리에서 `terraform init`  및 `terraform apply`  실행

2. infra 디렉토리에서 `terraform init`  및 `terraform apply`  실행


<br>

### 내부 서비스 배포

1. service > deploy > kubernetes 디렉토리 이동

2. kubectl 명령어 사용을 위해 EKS 클러스터 내 kubeconfig를 업데이트합니다.  
   `aws eks update-kubeconfig --name devops-eks-cluster --profile devops`
   
3. kubectl을 통한 API 요청이 정상인지 확인합니다.  
   `kubectl get nodes`
   
4. 서비스 소스 코드를 배포합니다.  
   `kubectl apply -f complete-demo.yaml`
   
5. 도메인(devops.jjikin.com)을 통해 접속 후 서비스를 확인합니다.

<br>

### 구성 삭제

1. 서비스 리소스를 삭제합니다.  
   `kubectl delete -f complete-demo.yaml`

2. Route53 Record, ALB 삭제되었는지 확인합니다.

   {: .prompt-info }

   > Record의 경우 externaldns가 Route53을 폴링하는 간격 차이로 삭제가 늦어질 수 있습니다.

3. 생성한 인프라를 삭제합니다.  
   `terraform destroy --auto-approve`

4. backend 디렉토리에서 생성한 인프라를 삭제합니다.  
   `terraform destroy --auto-approve`

<br>
<br>

다음 포스트 [2023-06-18-terraform-aws-modules 기반 EKS 환경 구축하기(2)](https://jjikin.com/posts/terraform-aws-modules-%EA%B8%B0%EB%B0%98-EKS-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0(2)/)에서 이어집니다.
