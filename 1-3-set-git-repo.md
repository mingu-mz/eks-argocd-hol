# Git 저장소 구성(10분)

이 장에서는 플랫폼 엔지니어로서 Argo CD가 Git 저장소와 상호 작용할 수 있도록 필요한 메타데이터와 자격 증명을 구성합니다.


## 클러스터 주석 삽입(5분)

"애플리케이션" 장의 앞부분에서 ArgoCD 애플리케이션 매니페스트에서 repoURL을 수동으로 업데이트했습니다. 이 작업을 더욱 동적이고 재사용 가능하게 만들려면 클러스터 객체의 어노테이션이나 메타데이터에서 repoURL과 같은 환경 값을 가져오는 ApplicationSet을 사용할 수 있습니다.

애노테이션은 ArgoCD 클러스터 객체에 연결된 키-값 쌍입니다. 허브-클러스터에는 GitOps Bridge에서 삽입된 애노테이션이 이미 포함되어 있습니다. ArgoCD 대시보드 > 설정 > 클러스터 > 허브-클러스터로 이동하여 이러한 애노테이션을 확인할 수 있습니다.

![](images/2025-10-29-14-40-51.png)

> 레이블은 생성기 조건을 충족하는 객체 컬렉션을 찾는 데 사용할 수 있습니다. 어노테이션은 저장소 URL과 같은 추가 정보를 제공합니다.

다음 장에서는 Git 저장소를 참조하는 ApplicationSet을 생성합니다. ArgoCD에서는 애노테이션을 사용하여 이러한 저장소를 동적으로 참조할 수 있습니다. 예를 들어 workload_repo_url 애노테이션을 참조하려면 다음과 같이 합니다.

```sh
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
...
template:
   spec:
      source:
        repoURL:'{{.metadata.annotations.workload_repo_url}}'
```

저장소(플랫폼, 워크로드, 애드온)에 대한 정보는 AWS Secrets(eks-blueprints-workshop-gitops-platform, eks-blueprints-workshop-gitops-workloads, eks-blueprints-workshop-gitops-addons)에 이미 입력되어 있습니다. 다음은 eks-blueprints-workshop-gitops-platform의 AWS Secret 값입니다. 이 Secret은 플랫폼 Git 저장소의 메타데이터를 저장합니다.

![](images/2025-10-29-14-42-09.png)

이 장에서는 이러한 값을 주석으로 복사하여 ArgoCD ApplicationSet에서 참조할 수 있도록 합니다.

1. Secrete Manaser

    ```sh
    cat <<'EOF' >> ~/environment/hub/main.tf

    # Retrieve Git repository metadata from AWS Secrets Manager for platform, workload, and addon repositories

    data "aws_secretsmanager_secret" "git_data_addons" {
      name = var.secret_name_git_data_addons
    }
    data "aws_secretsmanager_secret_version" "git_data_version_addons" {
      secret_id = data.aws_secretsmanager_secret.git_data_addons.id
    }
    data "aws_secretsmanager_secret" "git_data_platform" {
      name = var.secret_name_git_data_platform
    }
    data "aws_secretsmanager_secret_version" "git_data_version_platform" {
      secret_id = data.aws_secretsmanager_secret.git_data_platform.id
    }
    data "aws_secretsmanager_secret" "git_data_workload" {
      name = var.secret_name_git_data_workloads
    }
    data "aws_secretsmanager_secret_version" "git_data_version_workload" {
      secret_id = data.aws_secretsmanager_secret.git_data_workload.id
    }

    EOF
    ```

2. AWS Secrets에서 Git 메타데이터 검색

    각 비밀에는 url, basepath, path, revision과 같은 키가 포함되어 있습니다.

    다음 블록을 main.tf 파일에 추가하여 비밀을 구문 분석하고 로컬 변수에 값을 할당합니다.

    ```sh
    cat <<'EOF' >> ~/environment/hub/main.tf

    locals{

      gitops_addons_url      = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_addons.secret_string).url
      gitops_addons_basepath = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_addons.secret_string).basepath
      gitops_addons_path     = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_addons.secret_string).path
      gitops_addons_revision = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_addons.secret_string).revision


      gitops_platform_url      = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_platform.secret_string).url
      gitops_platform_basepath = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_platform.secret_string).basepath
      gitops_platform_path     = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_platform.secret_string).path
      gitops_platform_revision = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_platform.secret_string).revision


      gitops_workload_url      = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_workload.secret_string).url
      gitops_workload_basepath = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_workload.secret_string).basepath
      gitops_workload_path     = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_workload.secret_string).path
      gitops_workload_revision = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_workload.secret_string).revision

      annotations = merge(
        #enableaddonmetadata module.eks_blueprints_addons.gitops_metadata,
        {
          aws_cluster_name = module.eks.cluster_name
          aws_region = local.region
          aws_account_id = data.aws_caller_identity.current.account_id
          aws_vpc_id = local.vpc_id
          aws_vpc_name = data.terraform_remote_state.vpc.outputs.vpc_name
        },
        {
          #enableirsarole argocd_iam_role_arn = aws_iam_role.argocd_hub.arn
          argocd_namespace = local.argocd_namespace
        },
        {
          addons_repo_url = local.gitops_addons_url
          addons_repo_basepath = local.gitops_addons_basepath
          addons_repo_path = local.gitops_addons_path
          addons_repo_revision = local.gitops_addons_revision
        },
        {
          platform_repo_url = local.gitops_platform_url
          platform_repo_basepath = local.gitops_platform_basepath
          platform_repo_path = local.gitops_platform_path
          platform_repo_revision = local.gitops_platform_revision
        },
        {
          workload_repo_url = local.gitops_workload_url
          workload_repo_basepath = local.gitops_workload_basepath
          workload_repo_path = local.gitops_workload_path
          workload_repo_revision = local.gitops_workload_revision
        },
        #enableeso{
        #enableeso  external_secrets_service_account = local.external_secrets.service_account
        #enableeso  external_secrets_namespace = local.external_secrets.namespace
        #enableeso}    
      )
    }

    EOF
    ```

3. 주석 삽입

    어노테이션은 GitOps Bridge 모듈을 사용하여 허브-클러스터 객체에 적용됩니다. 다음 명령을 사용하여 메타데이터 줄(7번째 줄)의 주석 처리를 제거하고 어노테이션 주입을 활성화하세요.

    ```sh
    sed -i "s/#enableannotation //g" ~/environment/hub/main.tf
    ```

    위의 명령은 main.tf의 메타데이터 줄(7번째 줄)의 주석 처리를 해제하여 주석 주입을 활성화합니다.

    ```sh
    module "gitops_bridge_bootstrap" {
      source = "gitops-bridge-dev/gitops-bridge/helm"
      version = "0.0.1"
      cluster = {
        cluster_name = module.eks.cluster_name
        environment = local.environment
        metadata = local.annotations
        #enableaddons addons = local.labels
    }
    ```

4. Terraform 적용

    ```sh
    cd ~/environment/hub
    terraform apply --auto-approve
    ```

5. 주석 업데이트 검증

    Argo CD 대시보드에서 **Setting > Cluster > hub-cluster** 로 이동하여 hub-cluster 객체를 확인하세요. 이를 통해 GitOps Bridge가 주석을 성공적으로 업데이트했는지 확인할 수 있습니다.

    ![](images/2025-10-29-14-45-24.png)

## ArgoCD Git 저장소(5분)

Gitea에 Git 저장소가 호스팅되어 있지만 Argo CD에서는 아직 접근할 수 없습니다. 이 장에서는 ArgoCD가 GitOps 저장소에 접근할 수 있도록 구성합니다. 여러 가지 방법이 있습니다. Argo CD가 이러한 저장소에 액세스할 수 있도록 합니다. 사용자 이름과 비밀번호 방식을 사용합니다.

저장소(플랫폼, 워크로드, 애드온)에 대한 자격 증명은 이미 AWS Secrets(eks-blueprints-workshop-gitops-platform, eks-blueprints-workshop-gitops-workloads, eks-blueprints-workshop-gitops-addons)에 저장되어 있습니다. 다음은 eks-blueprints-workshop-gitops-platform secret의 AWS Secret 값입니다.

![](images/2025-10-29-14-54-01.png)


1. Git 자격 증명 읽기

    ```sh
    cat <<'EOF' >> ~/environment/hub/main.tf

    locals{
      gitops_workload_repo_username = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_workload.secret_string).username
      gitops_workload_repo_password = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_workload.secret_string).password

      gitops_platform_repo_username = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_platform.secret_string).username
      gitops_platform_repo_password = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_platform.secret_string).password

      gitops_addons_repo_username = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_addons.secret_string).username
      gitops_addons_repo_password = jsondecode(data.aws_secretsmanager_secret_version.git_data_version_addons.secret_string).password

    }

    EOF
    ```

2. Git 저장소에 대한 Argo CD 비밀번호 만들기

    시크릿을 생성하는 방법은 여러 가지가 있습니다. Secrets Manager에서 시크릿을 생성하고 외부 시크릿 연산자를 사용하여 클러스터에 동기화할 수 있습니다. 이 워크숍에서는 Terraform을 사용하여 시크릿을 생성합니다.

    ```sh
    cat <<'EOF' >> ~/environment/hub/main.tf

    resource "kubernetes_secret" "git_secrets" {
      depends_on = [kubernetes_namespace.argocd]
      for_each = {
        git-addons = {
          type     = "git"
          name     = "git-addons"
          url      = local.gitops_addons_url
          username = local.gitops_addons_repo_username
          password = local.gitops_addons_repo_password
        }
        git-platform = {
          type     = "git"
          name     = "git-platform"
          url      = local.gitops_platform_url
          username = local.gitops_platform_repo_username
          password = local.gitops_platform_repo_password
        }
        git-workloads = {
          type     = "git"
          name     = "git-workloads"
          url      = local.gitops_workload_url
          username = local.gitops_workload_repo_username
          password = local.gitops_workload_repo_password
        }
      }
      metadata {
        name      = each.key
        namespace = kubernetes_namespace.argocd.metadata[0].name
        labels = {
          "argocd.argoproj.io/secret-type" = "repository"
        }
      }
      data = each.value
    }
    EOF
    ```

3. Terraform 적용

    이 명령은 Terraform 구성을 적용하고 EKS 클러스터에 Argo CD 비밀을 프로비저닝합니다.

    ```sh
    cd ~/environment/hub
    terraform apply --auto-approve
    ```

    Argo CD 대시보드로 이동하여 설정 페이지에 접속하세요. "저장소"를 선택하면 gitops-platform 및 gitops-workload 저장소를 볼 수 있습니다.

    ![](images/2025-10-29-14-56-54.png)

    Argo CD의 Git 저장소 연결 데이터는 Kubernetes Secret에 저장됩니다. Terraform이 Git 저장소에 액세스하기 위한 구성을 포함하는 Secret 객체를 생성했는지 확인할 수 있습니다.

    ```sh
    kubectl get secret -n argocd --selector=argocd.argoproj.io/secret-type=repository --context hub-cluster
    ```

    예상 출력:

    ```sh
    NAME            TYPE     DATA   AGE
    git-addons      Opaque   5      4m36s
    git-platform    Opaque   5      4m36s
    git-workloads   Opaque   5      4m36s
    ```

    이 시점에서 Argo CD는 Kubernetes Secrets에 저장된 자격 증명을 사용하여 Git 저장소에서 안전하게 인증할 수 있습니다.
