---
title: terraform-aws-modules을 활용한 AWS EKS 환경 구축하기
date: 2023-06-12 09:00:39 +09:00
categories: [devops-study, eks]
tags: [aws, eks, kubenetes, k8s, terraform, iac, module]
img_path: ../
image: ../assets/img/posts/image-20230618154229292.png
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

devops 스터디를 위한 EKS 환경을 terraform을 통해 구축합니다.

테라폼 관련 github 커뮤니티에서 AWS 리소스를 생성할 때 필요한 테라폼 코드를 모듈화하여 제공([terraform-aws-modules](https://github.com/terraform-aws-modules))하고 있어, 
이를 활용하면 직접 코드를 작성하는 것 보다 간편하고 빠르게 리소스를 생성할 수 있습니다.

 



## Terraform Code

### VPC.tf

여러 어카운트 간 리소스 연동이 필요하므로, 모든 리소스는 퍼블릭 영역에 생성합니다.

```json
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "devops-vpc"
  cidr = "192.168.0.0/16"

  azs              = ["us-east-1a", "us-east-1c"]
  public_subnets   = ["192.168.0.0/20", "192.168.16.0/20"]
  # private_subnets  = ["192.168.32.0/20", "192.168.48.0/20"]

  public_subnet_names = ["devops-pub-a-sn", "devops-pub-c-sn"]
  # private_subnet_names = ["devops-pri-a-sn", "devops-pri-c-sn"]
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1  # 서브넷에 해당 주석을 달아주면, k8s 내에서 ingress 생성 시 서브넷 지정이 필요없음
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

output "private_subnets" {
  description = "List of IDs of private subnets"
  value       = module.vpc.private_subnets
}
```



참고 사이트

- https://github.com/terraform-aws-modules/terraform-aws-vpc
- https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest













Chirpy 테마 설치 [방법](https://chirpy.cotes.page/posts/getting-started/)에는 Chirpy Starter와 GitHub Fork 방식이 존재합니다.
Chirpy Starter의 경우 빠르게 구성하여 블로깅할 수 있는 장점이 있지만 커스터마이징이 제한적입니다.<br>따라서 이 포스트에서는 GitHub Fork 방식을 통해 설치합니다.    

1. [링크](https://github.com/cotes2020/jekyll-theme-chirpy/fork)를 통해 Repository를 Fork 합니다.<br>Repository name은 반드시 [github ID].github.io 형식으로 생성해야하며, 설정 후 Create Fork를 선택합니다.
   ![image-20230617194703359](../assets/img/posts/image-20230617194703359.png)

   

2. branch를 master에서 main으로 변경하고 Branch protection rule도 설정합니다.
   ![image-20230617201648065](../assets/img/posts/image-20230617201648065.png)
   ![image-20230618023426257](../assets/img/posts/image-20230618023426257.png)

   {: .prompt-info}

   >  변경하지 않는 경우 Github 배포 시 에러 발생하는 케이스가 있었습니다.

   

3. 로컬로 코드를 가져오기 위해 git clone합니다.

   ```shell
   git clone https://github.com/jjikin/jjikin.github.io.git
   ```

   ![image-20230617194724936](../assets/img/posts/image-20230617194724936.png)

   

4. jekyll 실행을 위해 필요한 모듈을 설치합니다.

   ```shell
    cd ~/Documents/blog/jjikin.github.io
    bundle
   ```

   ![image-20230617202547940](../assets/img/posts/image-20230617202547940.png)

   {: .prompt-danger }

   > bundle 실행 전 반드시 ruby 버전이 최소 3 버전 이상인지 체크해야 합니다.  
   > MacOS(Intel)에는 기본적으로 ruby 2.6 버전이 설치되어 있는데, 이 상태에서 bundle을 통해 모듈을 설치할 경우 Chirpy에서 사용하는 모듈과 호환되지 않아 블로그 기능(다크모드, 검색, 이미지 표시, 모바일 환경 비정상 동작 등)이 정상적으로 동작하지 않습니다.

   

5. npm을 통해 node.js 모듈을 설치합니다.

   ```shell
   npm install && npm run build
   ```

   {: .prompt-warning }

   > node.js 모듈을 설치하지 않으면 assets/js/dist/*.min.js Not Found 에러 발생과 함께 블로그 기능이 정상적으로 동작하지 않습니다.

   

6. 설치 완료 후 아래 명령어를 통해 로컬에서 jekyll을 실행합니다.
   ```shell
   jekyll serve
   ```

   ![image-20230617202606906](../assets/img/posts/image-20230617202606906.png)

   

7. 웹브라우저에서 127.0.0.1:4000 주소로 블로그가 정상적으로 표시되는지 확인하고 블로그 내 여러 메뉴 및 기능들도 정상 동작하는지 확인합니다. 

![image-20230617202644391](../assets/img/posts/image-20230617202644391.png)

  

  



### Github 배포

로컬에서 테스트한 소스 코드를 Github에 배포합니다.

1. 배포 전 아래와 같이 Settings - Pages - Build and deployment 에서 소스를 GitHub Actions로 변경합니다.

   ![image-20230617202846583](../assets/img/posts/image-20230617202846583.png)

   

2. Configure를 선택합니다.

   ![image-20230617203012976](../assets/img/posts/image-20230617203012976.png)

   

3. 별도의 수정 없이 Commit changes...를 선택 후 Commit changes 선택합니다.

   ![image-20230617203158302](../assets/img/posts/image-20230617203158302.png)

   ![image-20230617203223507](../assets/img/posts/image-20230617203223507.png)

   {: .prompt-warning }

   > GitHub Actions로 소스를 변경하지 않거나, Configure를 완료하지 않고 배포할 경우 아래와 같이 index.html 화면만 표시되니 주의합니다.

   ![image-20230617203510587](../assets/img/posts/image-20230617203510587.png)

   

4. 기존 배포 방식(Deploy form a branch)에 사용되던 파일(pages-deploy.yml.hook)을 삭제합니다.

   ![image-20230618140237706](../assets/img/posts/image-20230618140237706.png)

   

5. Github에서 jekyll.yml을 생성했으므로 git pull을 통해 로컬 리소스와 동기화를 먼저 진행합니다.

   ```shell
   cd ~/Documents/blog/jjikin.github.io
   git pull
   ```

   ![image-20230617204557101](../assets/img/posts/image-20230617204557101.png)

   

6. .gitignore 내 assets/js/dist 디렉토리 내 파일들의 Push가 무시되도록하는 설정을 주석처리 합니다.

   ```shell
   # Bundler cache
   .bundle
   vendor
   Gemfile.lock
   
   # Jekyll cache
   .jekyll-cache
   _site
   
   # RubyGems
   *.gem
   
   # NPM dependencies
   node_modules
   package-lock.json
   
   # IDE configurations
   .idea
   .vscode
   
   # Misc
   # assets/js/dist  ### 주석 처리
   ```

   {: .prompt-warning }

   > 로컬에서는 assets/js/dist/*.min.js 파일이 존재하여 정상 동작했지만, 위 설정을 하지 않고 배포할 경우 Github에는 해당 파일이 push되지 않으므로 블로그 기능이 정상 동작하지 않습니다.

   

7. git 배포를 위해 _posts 경로에 테스트용 포스트를 생성한 후 git push 합니다.

   ```shell
   git add -A
   git commit -m "test"
   git push
   ```

   ![image-20230618023030988](../assets/img/posts/image-20230618023030988.png)

   ![image-20230618022538483](../assets/img/posts/image-20230618022538483.png)

   

8. Github - Actions 탭에서 배포 워크플로우 실행을 확인할 수 있습니다.

   ![image-20230618021752649](../assets/img/posts/image-20230618021752649.png)

   

9. 테스트 페이지와 블로그 기능이 정상 동작하는지 확인합니다.

![image-20230618024152318](../assets/img/posts/image-20230618024152318.png)





## 마무리

루비 버전 때문에 정말 삽질을 많이 했다. Jekyll이 루비 기반으로 만들어진 엔진인데 이 부분을 간과했었다.

이외에도 많은 분들이 사용하는 테마인만큼 커스터마이징과 이슈 해결을 위한 구글링이 매우 수월했습니다.

Chirpy 예제 사이트에서 설치 및 md 작성 방법과 Favicon 등에 대한 소개 페이지를 확인 가능합니다.
https://chirpy.cotes.page/

또한 에러 발생 시, Github 내 Issue 검색을 통해 트러블슈팅하면 왠만한 에러들은 해결 가능했습니다.
https://github.com/cotes2020/jekyll-theme-chirpy/issues





## 커스터마이징 간 이슈 해결

Chirpy 테마의 커스터마이징은 하얀눈길님 [블로그](https://www.irgroup.org/categories/chirpy/)를 참고하여 진행하였습니다. 이후 커스터마이징 하면서 겪었던 내용들을 업데이트할 예정입니다.



- **avator 아이콘이 변경되지 않거나, 포스트 내 이미지에서 `The image could not be loaded.`  에러 발생하는 경우**

  _config.xml에서 cdn을 사용하도록 설정되어있으므로, 이를 주석처리