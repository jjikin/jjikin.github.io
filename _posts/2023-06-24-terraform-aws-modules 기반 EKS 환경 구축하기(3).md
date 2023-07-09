---
title: terraform-aws-modules 기반 EKS 환경 구축하기(3)
date: 2023-06-24 21:15:15 +09:00
categories: [devops-study, eks]
tags: [aws, eks, kubenetes, k8s, terraform, iac, module]
image: /assets/img/posts/image-20230619231723025.png
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
img_path: /assets/img/posts/
image: /assets/img/posts/
---

이번 포스트에서는 terraform-aws-modules으로 구축했을 때 어떤 리소스들이 생성되는지 확인하고 미리 정의한 네이밍 규칙에 맞게 리소스를 재정의합니다.

또한 소스 코드 내 주석 처리 및 불필요한 부분들을 최적화합니다.



기존 포스트 내용대로 EKS를 구축하면 아래와 같이 IAM Role, Policy, SecurityGroup 등에서 prefix가 부여됩니다.

![image-20230709191027960](image-20230709191027960.png)

만약 다른 리전에서 동일한 코드 및 이름으로 클러스터를 생성해야 하는 경우 이러한 prefix가 필요하지만, 그 외의 경우 prefix는 불필요하고 가시성도 떨어집니다.

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



## Terraform Code

생성되는 리소스에 대한 네이밍 규칙을 정의하고 이에 맞게 기존 리소스 이름을 변경합니다. {프로젝트}-{리소스} 부분은 가능한 경우 ${local.name}으로 대체합니다.

- IAM Role : {프로젝트}-{리소스}-{용도}-role
- IAM Policy : {프로젝트}-{리소스}-{용도}-policy
- SecurityGroup : {프로젝트}-{리소스}-{용도}-sg

변경하는 방법은 EKS 모듈에서 해당하는 값을 선언해주면 되는데, 관련 옵션을 확인하려면 아래와 같이 terraform-aws-modules 내 variable.tf 파일을 참고합니다.
https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/modules/eks-managed-node-group/variables.tf



### IAM Role

- Cluster IAM Role `devops-eks-cluster-cluster-20230624062324814400000006`

  - 연결된 Policy
    - AmazonEKSClusterPolicy
    - AmaonEKSVPCResourceController
    - `devops-eks-cluster_encryption-policy20230709110325330300000012`
      : etcd 암호화를 위한 kms키 권한

  - 네이밍 변경 : `devops-eks-cluster-role`

  - 코드 내 변경 사항
    ```hcl
    module "eks" {
      source = "terraform-aws-modules/eks/aws"
      cluster_name                   = "${local.name}-cluster"
      cluster_version                = 1.24
      cluster_endpoint_public_access = true
      iam_role_name = "${local.name}-cluster-role"  # 추가
      iam_role_use_name_prefix = false           # 추가
      ...
    ```

    

- NodeGroup IAM Role ` devops-eks-app-ng-eks-node-group-2023062406233066610000000c`

  eks 모듈에서 노드 그룹별로 별도의 Role을 만들도록 설계되었으므로, 하나의 Role을 사용하기 위해서는 별도로 정의 후 arn을 지정해야 합니다.

  - 연결된 Policy
    - AmazonEKSWorkerNodePolicy
    - AmazonEKS_CNI_Policy
    - AmazonEC2ContainerRegistryReadOnlyAmazonEKSClusterPolicy
  - 네이밍 변경 : `devops-eks-node-role` - 하나의 노드그룹을 같이 사용하도록 설정

  - 코드 내 변경 사항

    ```hcl
    module "eks" {
      ...
      eks_managed_node_group_defaults = {
        ami_type       = "AL2_x86_64"
        ...
        create_iam_role            = false   											 								  # 추가
        iam_role_name              = "${local.name}-node-role"    											# 추가
        iam_role_arn               = module.iam_assumable_role_custom.iam_role_arn  # 추가
        iam_role_use_name_prefix   = false     																			# 추가
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
    
      create_role = true
      role_name   = "${local.name}-node-role"
      role_requires_mfa = false
      custom_role_policy_arns = [
        "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
        "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
        "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
      ]
    }
    ```




- IRSA Role

  `role_name` 내 ${local.name} 적용

  - devops-eks-vpc-cni-irsa-role → devops-eks-vpc_cni-role
  - devops-eks-lb-controller-irsa-role → devops-eks-lb_controller-role
  - devops-eks-lb-controller-tg-binding-only-irsa-role → devops-eks-lb_controller_tg-role
  - devops-eks-externaldns-irsa-role → devops-eks-external_dns-role





### IAM Policy

- encryption IAM Policy `devops-eks-cluster-cluster-ClusterEncryption202306240623305330300000012`

  - etcd 저장소에 저장되는 데이터들을 CMK로 암복호화하기 위한 권한

  - 네이밍 변경 : `devops-eks-cluster-encryption-policy`

  - 코드 내 변경 사항
    ```hcl
    module "eks" {
      ...
      iam_role_name = "${local.name}-cluster-role"
      iam_role_use_name_prefix = false
      cluster_encryption_policy_name = "${local.name}-cluster_encryption-policy"  # 추가
      cluster_encryption_policy_use_name_prefix = false												 # 추가
      ...
    ```

    

- IRSA Policy

  IRSA를 생성해주는 모듈`iam-role-for-service-accounts-eks`에서는 Role에 부여되는 Policy 이름을 변경할 수 있는 Variable을 제공하지 않았습니다. 
  Policy를 별도로 생성한 후 arn을 모듈로 전달하는 방식으로 해결가능하지만 추후 컴포넌트 추가 시 Policy도 생성해줘야하는 번거로움이 있어 prefix만 변경했습니다.

  - prefix 변경 `AmazonEKS_CNI_Policy-20230624062324810600000001` → `devops-eks-CNI_Policy-20230624062324810600000001`
    ```hcl
    module "vpc_cni_irsa_role" { 
      source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
    
      role_name             = "${local.name}-vpc_cni-role"
      policy_name_prefix = "${local.name}-"										# 추가
      ...
      
    module "load_balancer_controller_irsa_role" { ...
    module "load_balancer_controller_targetgroup_binding_only_irsa_role" { ...
    module "external_dns_irsa_role" { ...
    ...
      
    ```





### SecurityGroup

- 클러스터 보안 그룹 `eks-cluster-sg-devops-eks-cluster-977726102` - [Link]([https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/sec-group-reqs.html)

  EKS에 의해 자동 생성되며, 인바운드 소스를 Self(자기 자신)으로 설정해서 컨트롤 플레인과 워커 노드에 연결하여 둘 간의 통신이 항상 가능하도록 허용합니다.

  - 네이밍 변경 : EKS에서 자동생성하는 리소스로 변경이 불가능합니다.

    

- 컨트롤 플레인 보안 그룹(=추가 보안 그룹) `devops-eks-cluster-cluster-20230709113604226200000003`

  - 네이밍 변경 : `devops-eks-cluster-sg`

  - 코드 내 변경 사항

    ```hcl
    module "eks" {
      ...
      iam_role_name                  = "${local.name}-cluster-role"
      iam_role_use_name_prefix       = false
      cluster_encryption_policy_name = "${local.name}-cluster_encryption-policy"
      cluster_encryption_policy_use_name_prefix = false
      cluster_security_group_name    = "${local.name}-cluster-sg" 			           # 추가
      cluster_security_group_use_name_prefix = false       								     # 추가
      cluster_security_group_tags    = {"Name" = "${local.name}-cluster-sg"}      # 추가
      ...
    ```



- 워커 노드 보안 그룹 `devops-eks-cluster-node-20230709113604228700000005`

  - 네이밍 변경 : `devops-eks-node-sg`

  - 코드 내 변경 사항

    ```hcl
    module "eks" {
      ...
      cluster_security_group_name    = "${local.name}-cluster-sg" 			           
      cluster_security_group_use_name_prefix = false       			
      cluster_security_group_tags    = {"Name" = "${local.name}-cluster-sg"}
      node_security_group_name			 = "${local.name}-node-sg" 							   # 추가
      node_security_group_use_name_prefix = false                              # 추가
      node_security_group_tags       = {"Name" = "${local.name}-node-sg"}      # 추가
      ...
    ```



- 원격 액세스를 위한 보안 그룹 `eks-remoteAccess-0cc49d3d-1761-a381-d042-32af9ba4cfce`

  - 네이밍 변경 : `devops-eks-remote_access-sg`

  - 코드 내 변경 사항

    ```hcl
    resource "aws_security_group" "remote_access" {
      name = "${local.name}-remote_access-sg"
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





### KeyPair

- 원격 액세스를 위한 Key `devops-eks-cluster20230624135747050900000009`

  - 네이밍 변경 : `devops-ssh-keypair`

  - 코드 내 변경 사항

    ```hcl
    module "key_pair" {
      source  = "terraform-aws-modules/key-pair/aws"
      version = "~> 2.0"
    
      key_name           = "devops-ssh-keypair"
      create_private_key = true
    }
    ```







EC2 인스턴스 네임 태그 -> 시작템플릿을 통해 부여가능


terraform destory 과정에서 설치한 플러그인에서 connection refused 에러 발생 
https://github.com/terraform-aws-modules/terraform-aws-eks/issues/911
-> terraform에서 리소스를 삭제하는 과정에서 k8s 내 aws-auth configmap 리소스 삭제하고 클러스터를 삭제해야하는데, 이미 클러스터가 삭제상태에 들어간 상태에서 configmap을 삭제하기 위한 리소스 읽기 요청이 실패

아래아 같이 상태파일에서 aws-auth configmap 관련 리소스를 수동으로 삭제해 준후 재시도한다.
`terraform state rm module.eks.kubernetes_config_map_v1_data.aws_auth`
-> 필요한가?

`terraform destroy --auto-approve -refresh=false`
-> 즉, 삭제 전 테라폼이 aws-auth configmap 을 최신상태로 업데이트한 후 삭제하지 않고 상태파일에 저장된 상태를 기준으로 삭제하도록 refresh 옵션을 비활성화하여 삭제

![](/Users/mzc01-ljyoon/Documents/blog/jjikin.github.io/assets/img/posts/image-20230625132119421.png)



를참고하여  아래 추가 

```hcl
data "aws_eks_cluster" "cluster" {
  name = module.eks.cluster_id
}

data "aws_eks_cluster_auth" "cluster" {
  name = module.eks.cluster_id
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
  load_config_file       = false
  version                = "~> 1.12"
}
```





21