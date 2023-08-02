---
title: terraform-aws-modules 기반 EKS 환경 구축하기(3)
date: 2023-06-24 21:15:15 +09:00
categories: [DevOps, eks]
tags: [aws, eks, kubenetes, k8s, terraform, iac, module]
image: /assets/img/posts/image-20230711103355989.png
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

이번 포스트에서는 terraform-aws-modules으로 구축했을 때 어떤 리소스들이 생성되는지 확인하고 미리 정의한 네이밍 규칙에 맞게 리소스를 재정의합니다.

또한 소스 코드 내 주석 처리 및 불필요한 부분들을 정리합니다.

<br>

기존 포스트 내용대로 EKS를 구축하면 아래와 같이 IAM Role, Policy, SecurityGroup 등에서 prefix가 부여됩니다.

![image-20230709191027960](/assets/img/posts/image-20230709191027960.png)

만약 다른 리전에서 동일한 코드 및 이름으로 클러스터를 생성해야 하는 경우 이러한 prefix가 필요하지만, 
그 외의 경우 prefix는 불필요하고 가시성도 떨어집니다.

terraform으로 생성된 리소스들을 `terraform state list` 명령어를 통해 확인한 후 하나씩 변경해보겠습니다.

<details markdown="1">
  <summary>코드 접기/펼치기</summary>



```json
data.aws_availability_zones.available
data.aws_caller_identity.current
data.aws_eks_cluster_auth.eks
data.terraform_remote_state.remote
aws_launch_template.launch_template
aws_security_group.devops-office-sg
aws_security_group.remote_access
helm_release.aws-load-balancer-controller
helm_release.external_dns
kubernetes_service_account.aws-load-balancer-controller
kubernetes_service_account.external-dns
module.ebs_csi_driver_irsa_role.data.aws_caller_identity.current
module.ebs_csi_driver_irsa_role.data.aws_iam_policy_document.ebs_csi[0]
module.ebs_csi_driver_irsa_role.data.aws_iam_policy_document.this[0]
module.ebs_csi_driver_irsa_role.data.aws_partition.current
module.ebs_csi_driver_irsa_role.data.aws_region.current
module.ebs_csi_driver_irsa_role.aws_iam_policy.ebs_csi[0]
module.ebs_csi_driver_irsa_role.aws_iam_role.this[0]
module.ebs_csi_driver_irsa_role.aws_iam_role_policy_attachment.ebs_csi[0]
module.eks.data.aws_caller_identity.current
module.eks.data.aws_eks_addon_version.this["aws-ebs-csi-driver"]
module.eks.data.aws_eks_addon_version.this["coredns"]
module.eks.data.aws_eks_addon_version.this["kube-proxy"]
module.eks.data.aws_eks_addon_version.this["vpc-cni"]
module.eks.data.aws_iam_policy_document.assume_role_policy[0]
module.eks.data.aws_iam_session_context.current
module.eks.data.aws_partition.current
module.eks.data.tls_certificate.this[0]
module.eks.aws_cloudwatch_log_group.this[0]
module.eks.aws_eks_addon.before_compute["vpc-cni"]
module.eks.aws_eks_addon.this["aws-ebs-csi-driver"]
module.eks.aws_eks_addon.this["coredns"]
module.eks.aws_eks_addon.this["kube-proxy"]
module.eks.aws_eks_cluster.this[0]
module.eks.aws_iam_openid_connect_provider.oidc_provider[0]
module.eks.aws_iam_policy.cluster_encryption[0]
module.eks.aws_iam_role.this[0]
module.eks.aws_iam_role_policy_attachment.cluster_encryption[0]
module.eks.aws_iam_role_policy_attachment.this["AmazonEKSClusterPolicy"]
module.eks.aws_iam_role_policy_attachment.this["AmazonEKSVPCResourceController"]
module.eks.aws_security_group.cluster[0]
module.eks.aws_security_group.node[0]
module.eks.aws_security_group_rule.cluster["ingress_nodes_443"]
module.eks.aws_security_group_rule.node["egress_all"]
module.eks.aws_security_group_rule.node["ingress_cluster_443"]
module.eks.aws_security_group_rule.node["ingress_cluster_4443_webhook"]
module.eks.aws_security_group_rule.node["ingress_cluster_6443_webhook"]
module.eks.aws_security_group_rule.node["ingress_cluster_8443_webhook"]
module.eks.aws_security_group_rule.node["ingress_cluster_9443_webhook"]
module.eks.aws_security_group_rule.node["ingress_cluster_kubelet"]
module.eks.aws_security_group_rule.node["ingress_nodes_ephemeral"]
module.eks.aws_security_group_rule.node["ingress_self_coredns_tcp"]
module.eks.aws_security_group_rule.node["ingress_self_coredns_udp"]
module.eks.kubernetes_config_map_v1_data.aws_auth[0]
module.eks.time_sleep.this[0]
module.external_dns_irsa_role.data.aws_caller_identity.current
module.external_dns_irsa_role.data.aws_iam_policy_document.external_dns[0]
module.external_dns_irsa_role.data.aws_iam_policy_document.this[0]
module.external_dns_irsa_role.data.aws_partition.current
module.external_dns_irsa_role.data.aws_region.current
module.external_dns_irsa_role.aws_iam_policy.external_dns[0]
module.external_dns_irsa_role.aws_iam_role.this[0]
module.external_dns_irsa_role.aws_iam_role_policy_attachment.external_dns[0]
module.iam_assumable_role_custom.data.aws_caller_identity.current
module.iam_assumable_role_custom.data.aws_iam_policy_document.assume_role[0]
module.iam_assumable_role_custom.data.aws_partition.current
module.iam_assumable_role_custom.aws_iam_role.this[0]
module.iam_assumable_role_custom.aws_iam_role_policy_attachment.custom[0]
module.iam_assumable_role_custom.aws_iam_role_policy_attachment.custom[1]
module.iam_assumable_role_custom.aws_iam_role_policy_attachment.custom[2]
module.iam_assumable_role_custom.aws_iam_role_policy_attachment.custom[3]
module.iam_assumable_role_custom.aws_iam_role_policy_attachment.custom[4]
module.key_pair.aws_key_pair.this[0]
module.key_pair.tls_private_key.this[0]
module.load_balancer_controller_irsa_role.data.aws_caller_identity.current
module.load_balancer_controller_irsa_role.data.aws_iam_policy_document.load_balancer_controller[0]
module.load_balancer_controller_irsa_role.data.aws_iam_policy_document.this[0]
module.load_balancer_controller_irsa_role.data.aws_partition.current
module.load_balancer_controller_irsa_role.data.aws_region.current
module.load_balancer_controller_irsa_role.aws_iam_policy.load_balancer_controller[0]
module.load_balancer_controller_irsa_role.aws_iam_role.this[0]
module.load_balancer_controller_irsa_role.aws_iam_role_policy_attachment.load_balancer_controller[0]
module.load_balancer_controller_targetgroup_binding_only_irsa_role.data.aws_caller_identity.current
module.load_balancer_controller_targetgroup_binding_only_irsa_role.data.aws_iam_policy_document.load_balancer_controller_targetgroup_only[0]
module.load_balancer_controller_targetgroup_binding_only_irsa_role.data.aws_iam_policy_document.this[0]
module.load_balancer_controller_targetgroup_binding_only_irsa_role.data.aws_partition.current
module.load_balancer_controller_targetgroup_binding_only_irsa_role.data.aws_region.current
module.load_balancer_controller_targetgroup_binding_only_irsa_role.aws_iam_policy.load_balancer_controller_targetgroup_only[0]
module.load_balancer_controller_targetgroup_binding_only_irsa_role.aws_iam_role.this[0]
module.load_balancer_controller_targetgroup_binding_only_irsa_role.aws_iam_role_policy_attachment.load_balancer_controller_targetgroup_only[0]
module.vpc.aws_default_network_acl.this[0]
module.vpc.aws_default_route_table.default[0]
module.vpc.aws_default_security_group.this[0]
module.vpc.aws_internet_gateway.this[0]
module.vpc.aws_route.public_internet_gateway[0]
module.vpc.aws_route_table.public[0]
module.vpc.aws_route_table_association.public[0]
module.vpc.aws_route_table_association.public[1]
module.vpc.aws_subnet.public[0]
module.vpc.aws_subnet.public[1]
module.vpc.aws_vpc.this[0]
module.vpc_cni_irsa_role.data.aws_caller_identity.current
module.vpc_cni_irsa_role.data.aws_iam_policy_document.this[0]
module.vpc_cni_irsa_role.data.aws_iam_policy_document.vpc_cni[0]
module.vpc_cni_irsa_role.data.aws_partition.current
module.vpc_cni_irsa_role.data.aws_region.current
module.vpc_cni_irsa_role.aws_iam_policy.vpc_cni[0]
module.vpc_cni_irsa_role.aws_iam_role.this[0]
module.vpc_cni_irsa_role.aws_iam_role_policy_attachment.vpc_cni[0]
module.eks.module.eks_managed_node_group["devops-eks-app-ng"].data.aws_caller_identity.current
module.eks.module.eks_managed_node_group["devops-eks-app-ng"].data.aws_partition.current
module.eks.module.eks_managed_node_group["devops-eks-app-ng"].aws_eks_node_group.this[0]
module.eks.module.eks_managed_node_group["devops-eks-batch-ng"].data.aws_caller_identity.current
module.eks.module.eks_managed_node_group["devops-eks-batch-ng"].data.aws_partition.current
module.eks.module.eks_managed_node_group["devops-eks-batch-ng"].aws_eks_node_group.this[0]
module.eks.module.eks_managed_node_group["devops-eks-front-ng"].data.aws_caller_identity.current
module.eks.module.eks_managed_node_group["devops-eks-front-ng"].data.aws_partition.current
module.eks.module.eks_managed_node_group["devops-eks-front-ng"].aws_eks_node_group.this[0]
module.eks.module.eks_managed_node_group["devops-eks-mgmt-ng"].data.aws_caller_identity.current
module.eks.module.eks_managed_node_group["devops-eks-mgmt-ng"].data.aws_partition.current
module.eks.module.eks_managed_node_group["devops-eks-mgmt-ng"].aws_eks_node_group.this[0]
module.eks.module.kms.data.aws_caller_identity.current
module.eks.module.kms.data.aws_iam_policy_document.this[0]
module.eks.module.kms.data.aws_partition.current
module.eks.module.kms.aws_kms_alias.this["cluster"]
module.eks.module.kms.aws_kms_key.this[0]
```
</details>

<br>


## Terraform Code

생성되는 리소스에 대한 네이밍 규칙을 정의하고 이에 맞게 기존 리소스 이름을 변경합니다. {프로젝트} 부분은 가능한 경우 ${local.name}으로 대체합니다.

- {프로젝트}-{서비스}-{용도}-{리소스}
  - IAM Role : devops-eks-cluster-role
  - IAM Policy : devops-eks-encryption-policy
  - SecurityGroup : devops-eks-node-sg


변경하는 방법은 EKS 모듈에서 해당하는 값을 선언해주면 되는데, 관련 옵션을 확인하려면 [링크](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/modules/eks-managed-node-group/variables.tf)와 같이 terraform-aws-modules 내 variable.tf 파일을 참고합니다.
<br>

<br>

### vpc.tf

라우팅 테이블, 인터넷 게이트웨이 이름(Tag)을 재정의합니다.

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "${local.name}-vpc"
  cidr = "192.168.0.0/16"

  azs                 = ["us-east-1a", "us-east-1c"]
  public_subnets      = ["192.168.0.0/20", "192.168.16.0/20"]
  public_subnet_names = ["${local.name}-pub-a-sn", "${local.name}-pub-c-sn"]
  
  public_subnet_tags       = {"kubernetes.io/role/elb" = 1}
  public_route_table_tags  = {"Name" = "${local.name}-vpc-public-rt"}   # 추가
  default_route_table_tags = {"Name" = "${local.name}-vpc-default-rt"}  # 추가
  igw_tags                 = {"Name" = "${local.name}-vpc-public-igw"}  # 추가
  
  enable_nat_gateway       = false
  enable_dns_hostnames     = true
  enable_dns_support       = true
  map_public_ip_on_launch  = true

  tags = local.tags  # 추가
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

<br>

<br>

### eks.tf

#### IAM Role

- Cluster IAM Role

  - 기존 리소스 : `devops-eks-cluster-cluster-20230624062324814400000006`

  - 연결된 Policy
    - AmazonEKSClusterPolicy
    - AmaonEKSVPCResourceController
    - devops-eks-cluster_encryption-policy20230709110325330300000012
    
      : etcd 암호화를 위한 kms키 권한
    
  - 네이밍 변경 : `devops-eks-cluster-role`

  - 코드 내 변경 사항
    ```hcl
    module "eks" {
      source = "terraform-aws-modules/eks/aws"
      cluster_name                   = "${local.name}-eks-cluster"
      cluster_version                = 1.27
      cluster_endpoint_public_access = true
      iam_role_name = "${local.name}-eks-cluster-role"   # 추가
      iam_role_use_name_prefix = false   # 추가
      ...
    ```
    <br>

- NodeGroup IAM Role

  eks 모듈에서 노드 그룹별로 별도의 Role을 만들도록 설계되었으므로, 하나의 Role을 사용하기 위해서는 별도로 정의 후 arn을 지정해야 합니다.

  - 기존 리소스 : `devops-eks-app-ng-eks-node-group-2023062406233066610000000c`

  - 연결된 Policy
    - AmazonEKSWorkerNodePolicy
    - AmazonEKS_CNI_Policy
    - AmazonEC2ContainerRegistryReadOnlyAmazonEKSClusterPolicy
    
  - 네이밍 변경 : `devops-eks-node-role`

  - 코드 내 변경 사항

    ```hcl
    module "eks" {
      ...
      eks_managed_node_group_defaults = {
        ami_type       = "AL2_x86_64"
        ...
        create_iam_role            = false   # 추가
        iam_role_name              = "${local.name}-eks-node-role"   # 추가
        iam_role_arn               = module.iam_assumable_role_custom.iam_role_arn   # 추가
        iam_role_use_name_prefix   = false   # 추가
        iam_role_attach_cni_policy = true
        use_name_prefix            = false
        use_custom_launch_template = false
        ...
      }
      ...
    }
    ```

    ```hcl
    # node용 IAM Role 추가
    module "iam_assumable_role_custom" {
      source = "terraform-aws-modules/iam/aws//modules/iam-assumable-role"
    
      trusted_role_services = [
        "ec2.amazonaws.com"
      ]
    
      create_role             = true
      role_name               = "${local.name}-eks-node-role"
      role_requires_mfa       = false
      custom_role_policy_arns = [
        "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
        "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
        "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
      ]
    }
    ```
<br>


- IRSA Role

  role_name 내 `${local.name}` 적용

  - devops-eks-vpc-cni-irsa-role → `${local.name}-eks-vpc_cni-role`
  - devops-eks-lb-controller-irsa-role → `${local.name}-eks-lb_controller-role`
  - devops-eks-lb-controller-tg-binding-only-irsa-role → `${local.name}-eks-lb_controller_tg-role`
  - devops-eks-externaldns-irsa-role → `${local.name}-eks-external_dns-role`



<br>

#### IAM Policy

- encryption IAM Policy

  : etcd 저장소에 저장되는 데이터들을 CMK로 암복호화하기 위한 권한

  - 기존 리소스 : `devops-eks-cluster-cluster-ClusterEncryption202306240623305330300000012`

  - 네이밍 변경 : `devops-eks-cluster-encryption-policy`

  - 코드 내 변경 사항
    ```hcl
    module "eks" {
      ...
      iam_role_name = "${local.name}-eks-cluster-role"
      iam_role_use_name_prefix = false
      cluster_encryption_policy_name = "${local.name}-eks-cluster-encryption-policy"  # 추가
      cluster_encryption_policy_use_name_prefix = false  # 추가
      ...
    ```

    <br>

- IRSA Policy

  IRSA를 생성해주는 모듈`iam-role-for-service-accounts-eks`에서는 Role에 부여되는 Policy 이름을 변경할 수 있는 Variable을 제공하지 않았습니다.  
  Policy를 별도로 생성한 후 arn을 모듈로 전달하는 방식으로 해결가능하지만 추후 컴포넌트 추가 시 Policy도 생성해줘야하는 번거로움이 있어 prefix만 변경했습니다.

  - prefix 변경 `AmazonEKS_CNI_Policy-20230624062324810600000001`  
    → `devops-eks-CNI_Policy-20230624062324810600000001`
    
    ```hcl
    module "vpc_cni_irsa_role" { 
      source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
    
      role_name             = "${local.name}-eks-vpc_cni-role"
      policy_name_prefix = "${local.name}-eks-"  # 추가
      ...
    
    # 아래 모듈에도 동일하게 적용
    module "load_balancer_controller_irsa_role"
    module "load_balancer_controller_targetgroup_binding_only_irsa_role"
    module "external_dns_irsa_role"
    ```



<br>

#### SecurityGroup

- 클러스터 보안 그룹 `eks-cluster-sg-devops-eks-cluster-977726102` - [Link](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/sec-group-reqs.html)

  EKS에 의해 자동 생성되며, 인바운드 소스를 Self(자기 자신)으로 설정해서 컨트롤 플레인과 워커 노드에 연결하여 둘 간의 통신이 항상 가능하도록 허용합니다.

  - 네이밍 변경 : EKS에서 자동생성하는 리소스로 변경이 불가능합니다.

    <br>

- 컨트롤 플레인 보안 그룹(=추가 보안 그룹) `devops-eks-cluster-cluster-20230709113604226200000003`

  - 네이밍 변경 : `devops-eks-cluster-sg`

  - 코드 내 변경 사항

    ```hcl
    module "eks" {
      ...
      iam_role_name                  = "${local.name}-eks-cluster-role"
      iam_role_use_name_prefix       = false
      cluster_encryption_policy_name = "${local.name}-eks-cluster_encryption-policy"
      cluster_encryption_policy_use_name_prefix = false
      cluster_security_group_name    = "${local.name}-eks-cluster-sg"  # 추가
      cluster_security_group_use_name_prefix = false  # 추가
      cluster_security_group_tags    = {"Name" = "${local.name}-eks-cluster-sg"}  # 추가
      ...
    ```
<br>


- 워커 노드 보안 그룹 `devops-eks-cluster-node-20230709113604228700000005`

  - 네이밍 변경 : `devops-eks-node-sg`

  - 코드 내 변경 사항

    ```hcl
    module "eks" {
      ...
      cluster_security_group_name    = "${local.name}-eks-cluster-sg" 			           
      cluster_security_group_use_name_prefix = false       			
      cluster_security_group_tags    = {"Name" = "${local.name}-eks-cluster-sg"}
      node_security_group_name			 = "${local.name}-eks-node-sg"  # 추가
      node_security_group_use_name_prefix = false  # 추가
      node_security_group_tags       = {"Name" = "${local.name}-eks-node-sg"}  # 추가
      ...
    ```
    <br>

- 원격 액세스를 위한 보안 그룹 `eks-remoteAccess-0cc49d3d-1761-a381-d042-32af9ba4cfce`

  - 네이밍 변경 : `devops-eks-remote_access-sg`

  - 코드 내 변경 사항

    ```hcl
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
      ...
    }
    ```



<br>

#### KeyPair

- 원격 액세스를 위한 Key `devops-eks-cluster20230624135747050900000009`

  - 네이밍 변경 : `devops-ssh-keypair`

  - 코드 내 변경 사항

    ```hcl
    module "key_pair" {
      source  = "terraform-aws-modules/key-pair/aws"
      version = "~> 2.0"
    
      key_name           = "${local.name}-ssh-keypair"
      create_private_key = true
    }
    ```

<br>

#### Tagging

- 각 모듈을 통해 생성된 리소스에 Tag 추가

  ```hcl
  locals {
    name              = "devops"
    vpc_id            = module.vpc.vpc_id
    subnet_ids        = module.vpc.public_subnets
    ...
    tags = {
      CreatedBy = "Terraform"  # 추가
    }
  }
  ```

  ```hcl
  module "eks" {
    source  = "terraform-aws-modules/eks/aws"
  
    cluster_name                   = "${local.name}-eks-cluster"
    cluster_version                = 1.27
    cluster_endpoint_public_access = true
    iam_role_name = "${local.name}-eks-cluster-role"
    iam_role_use_name_prefix = false
  
    tags = local.tags  # 추가
  ```

  ```hcl
  module "vpc_cni_irsa_role" { 
    source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  
    role_name             = "${local.name}-eks-vpc_cni-role"
    policy_name_prefix    = "${local.name}-eks-"
    ...
    tags = local.tags  # 추가
  }  
  
  # 아래 모듈에도 동일하게 적용
  module "load_balancer_controller_irsa_role"
  module "load_balancer_controller_targetgroup_binding_only_irsa_role"
  module "external_dns_irsa_role"
  module "iam_assumable_role_custom"
  module "key_pair"
  ```



- 별도 생성한 Resource에 Tag 추가

  ```hcl
  resource "aws_security_group" "remote_access" {
    name = "${local.name}-eks-remote_access-sg"
    description = "Allow remote SSH access"
    vpc_id      = local.vpc_id
    ...
    tags = local.tags  # 추가
  }
  
  # 아래 리소스는 k8s 내 생성되는 리소스이므로 태깅 제외
  resource "kubernetes_service_account" "aws-load-balancer-controller"
  resource "kubernetes_service_account" "external-dns"
  resource "helm_release" "aws-load-balancer-controller"
  resource "helm_release" "external_dns"
  ```


<br>

#### 워커 노드에 ec2 Name Tag 추가

- 워커 노드에 ec2 Name Tag를 추가하려면 사용자지정 시작 템플릿을 사용해야합니다.

- 사용자지정 시작 템플릿을 사용하면 워커 노드에 접속하기 위해 eks module에 정의했던 remote_access 설정을 지원하지 않으므로 시작 템플릿에 별도 지정해야합니다.  
  여기서는 노드 Role `devops-eks-node-role` 에 SSM을 통해 접근할 수 있도록 설정합니다.

  ```hcl
  module "eks" {
    ...
    eks_managed_node_group_defaults = {
      ami_type                   = "AL2_x86_64"
      instance_types             = ["t3.medium"]  # 이동
      capacity_type              = "SPOT"
  
      create_iam_role            = false
      iam_role_name              = "${local.name}-eks-node-role"
      iam_role_arn               = module.iam_assumable_role_custom.iam_role_arn
      iam_role_use_name_prefix   = false
      iam_role_attach_cni_policy = true
      iam_role_additional_policies = {
        AmazonSSMManagedInstanceCore = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"  # 추가
      }   
      use_name_prefix            = false  # false 하지 않으면 리소스 이름 뒤 임의의 난수값이 추가되어 생성됨
      
      create_launch_template          = false  # 추가
      use_custom_launch_template      = true   # true로 변경 false:AWS EKS 관리 노드 그룹에서 제공하는 기본 템플릿을 사용
      enable_bootstrap_user_data      = true  # 추가 
      # 사용자 지정 템플릿을 노드그룹에 지정하는 경우 노드가 클러스터에 join 하기위한 부트스트랩이 자동 적용되지 않음. 따라서 해당 옵션 true 설정 필요
  
      block_device_mappings = {  # 이동
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            volume_size           = 30
            volume_type           = "gp3"
            delete_on_termination = true
          }
        }
      }
  
      remote_access = {  # 삭제
        ec2_ssh_key               = module.key_pair.key_pair_name
        source_security_group_ids = [aws_security_group.remote_access.id]
        tags = {
          "kubernetes.io/cluster/devops-eks-cluster" = "owned"  # 이동(AWS LB Controller 사용을 위한 요구 사항)
        }
      }
    }
    ...
  }
  ```
  
  <br>
  
- `eks_managed_node_group_defaults`에서 정의했던 내용을 시작 템플릿으로 이동 및 삭제한 후의 Code는 아래와 같습니다.

  ```hcl
  module "eks" {
    ...
    eks_managed_node_group_defaults = {
      ami_type                   = "AL2_x86_64"
  		capacity_type              = "SPOT"
  		
      create_iam_role            = false
      iam_role_name              = "${local.name}-eks-node-role"
      iam_role_arn               = module.iam_assumable_role_custom.iam_role_arn
      iam_role_use_name_prefix   = false
      iam_role_attach_cni_policy = true
      iam_role_additional_policies = {
        AmazonSSMManagedInstanceCore = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"  # 추가
      }      
      use_name_prefix            = false
      
      create_launch_template          = false
      use_custom_launch_template      = true
      enable_bootstrap_user_data      = true
    }
    ...
  }
  ```

<br>

- app, mgmt 노드용 custom launch template을 생성합니다.

  ```hcl  
  # main.tf에 variable 변수 생성
  variable "lt_resource_tags" {  # LT 리소스 태깅을 위한 변수
    type    = set(string)
    default = ["instance", "volume", "network-interface", "spot-instances-request"]
  }
  
  # app launch template, mgmt도 동일하게 생성
  resource "aws_launch_template" "app_launch_template" {
    name     = "${local.name}-eks-app_node-lt"
    instance_type = "t3.medium"
    instance_market_options { market_type = "spot" }
    key_name = module.key_pair.key_pair_name
    vpc_security_group_ids = [module.eks.cluster_primary_security_group_id, aws_security_group.remote_access.id]
  
    block_device_mappings {
      device_name = "/dev/xvda"
      ebs {
        volume_size           = 30
        volume_type           = "gp3"
        delete_on_termination = true
      }
    }  
  
    dynamic "tag_specifications" {
      for_each = var.lt_resource_tags
      content {
        resource_type = tag_specifications.key
        tags = {
          Name = "${local.name}-eks-app-node"
        }
      }
    }
    
    tags = local.tags # LT Tag
  }
  ```

  <br>

- eks 노드 그룹 정의에 시작템플릿을 추가합니다.

  ```hcl
    eks_managed_node_groups = {
      devops-eks-app-ng = {
        name         = "${local.name}-app-ng"
        launch_template_id = aws_launch_template.app_launch_template.id  # 추가
  			...
      }
  
      devops-eks-mgmt-ng = {
        name         = "${local.name}-mgmt-ng"
        launch_template_id = aws_launch_template.mgmt_launch_template.id  # 추가   
        ...
      } 
  ```

  
