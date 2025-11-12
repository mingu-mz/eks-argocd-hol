# Bootstrap(10분)

이 장에서는 플랫폼 엔지니어로서 Argo CD와 GitOps Bridge를 사용하여 기본 자동화 패턴을 설정합니다.

## 플랫폼 저장소 부트스트랩

플랫폼 Git 저장소의 폴더를 감시하도록 애플리케이션을 구성합니다. 이 폴더의 모든 파일은 자동으로 처리됩니다.

![](images/2025-10-29-15-04-26.png)

다음 장에서는 애드온, 네임스페이스, 워크로드 자동화를 위해 이 폴더에 파일을 추가합니다.


### Bootstrap ApplicationSet 만들기

ApplicationSet은 플랫폼 Git 저장소의 platform/bootstrap 디렉토리를 가리키는 "bootstrap"이라는 이름의 새로운 ArgoCD 애플리케이션을 생성합니다.


1. Bootstrap ApplicationSet 만들기

    ```sh
    mkdir -p ~/environment/hub/bootstrap
    cat > ~/environment/hub/bootstrap/bootstrap-applicationset.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: bootstrap
      namespace: argocd
    spec:
      goTemplate: true
      syncPolicy:
        preserveResourcesOnDeletion: true
      generators:
        - clusters:
            selector:
              matchLabels:
                fleet_member: hub
      template:
        metadata:
          name: 'bootstrap'
        spec:
          project: default
          source:
            repoURL: '{{ .metadata.annotations.platform_repo_url }}'
            path: '{{ .metadata.annotations.platform_repo_basepath }}{{ .metadata.annotations.platform_repo_path }}'
            targetRevision: '{{ .metadata.annotations.platform_repo_revision }}'
            directory:
              recurse: true
          destination:
            namespace: 'argocd'
            name: '{{ .name }}'
          syncPolicy:
            automated:
              allowEmpty: true

    EOF
    ```

    참고로 22~24번째 줄은 ArgoCD 클러스터 비밀의 주석을 사용합니다.

    현재 환경에서 값을 확인할 수 있습니다.

    ```sh
    kubectl --context hub-cluster get secrets -n argocd hub-cluster -o json | jq ".metadata.annotations.platform_repo_url" -r
    kubectl --context hub-cluster get secrets -n argocd hub-cluster -o json | jq ".metadata.annotations.platform_repo_basepath" -r
    kubectl --context hub-cluster get secrets -n argocd hub-cluster -o json | jq ".metadata.annotations.platform_repo_path" -r
    kubectl --context hub-cluster get secrets -n argocd hub-cluster -o json | jq ".metadata.annotations.platform_repo_revision" -r
    ```

    출력 결과는 다음과 유사해야 합니다. 플랫폼 저장소의 bootstrap 폴더를 가리켜야 합니다. 이 워크숍에서는 platform_repo_basepath가 사용되지 않으므로 빈 문자열로 설정되어 있습니다.

    ```sh
    https://d3mkl1q53qn8v6.cloudfront.net/gitea/workshop-user/eks-blueprints-workshop-gitops-platform

    bootstrap
    HEAD
    ```

2. Terraform에서 Bootstrap ApplicationSet 참조

    ~/environment/hub/bootstrap/bootstrap-applicationset.yaml에 부트스트랩 애플리케이션 세트를 생성했습니다. 이를 참조하는 변수를 다시 만들어 보겠습니다.

    ```sh
    cat <<'EOF' >> ~/environment/hub/main.tf
    locals{
      argocd_apps = {
        bootstrap   = file("${path.module}/bootstrap/bootstrap-applicationset.yaml")
      }
    }
    EOF
    ```

3. GitOps Bridge를 사용하여 Bootstrap ApplicationSet 배포

    이전 단계에서 Bootstrap ApplicationSet을 참조하는 변수를 생성했습니다. GitOps Bridge를 사용하여 이 ApplicationSet을 생성하겠습니다.

    ```sh
    sed -i "s/#enableapps //g" ~/environment/hub/main.tf
    ```

    위에 제공된 코드는 GitOps Bridge의 주석 처리를 제거하여 ArgoCD 애플리케이션을 생성합니다. 이 경우 부트스트랩 애플리케이션을 생성합니다.

    ```sh
    module "gitops_bridge_bootstrap" {
      source = "gitops-bridge-dev/gitops-bridge/helm"
      version = "0.0.1"
      cluster = {
        cluster_name = module.eks.cluster_name
        environment = local.environment
        #enableannotation metadata = local.addons_metadata
        #enableaddons addons = local.addons
      }
      apps = local.argocd_apps
      argocd = {
        namespace = local.argocd_namespace
    ```

### 허브 클러스터에 레이블을 지정하세요

1. fleet_member 레이블 변수 추가

    ApplicationSet 클러스터 생성기(16번째 줄)는 fleet_member = hub라는 레이블이 있는 클러스터를 필터링합니다. 이 레이블을 허브 클러스터 정의에 추가해 보겠습니다.

    ![](images/2025-10-29-15-09-23.png)

    fleet_member 라벨을 추가해 보겠습니다.

    ```sh
    cat <<'EOF' >> ~/environment/hub/main.tf

    locals{
      addons = merge(
        { fleet_member = "hub" },
        { tenant = "tenant1" },
        #enableaddonvariable local.aws_addons,
        #enableaddonvariable local.oss_addons,
        #enablewebstore{ workload_webstore = true }  
      )

    }

    EOF
    ```

2. GitOps Bridge를 통해 레이블 주입 활성화

    GitOps Bridge를 사용하여 허브-클러스터 객체에 레이블을 추가합니다. GitOps Bridge는 지정된 클러스터 객체에 레이블을 추가하도록 구성되어 있습니다.

    ```sh
    sed -i "s/#enableaddons//g" ~/environment/hub/main.tf
    ```

    위 코드는 아래에 강조 표시된 것처럼 main.tf 파일의 addons 변수의 주석 처리를 제거합니다. GitOps Bridge addons(8번째 줄) 변수에 할당된 모든 값은 클러스터 레이블에 할당됩니다.

    ```sh
    module "gitops_bridge_bootstrap" {
      source = "gitops-bridge-dev/gitops-bridge/helm"
      version = "0.0.1"
      cluster = {
        cluster_name = module.eks.cluster_name
        environment = local.environment
        #enableannotation metadata = local.annotations
        addons = local.addons
    }
    ```

### Terraform 적용

```sh
cd ~/environment/hub
terraform apply --auto-approve
```

### ArgoCD에서 Bootstrap 애플리케이션 검증

UI에서 ArgoCD 대시보드로 이동하여 "애플리케이션"을 클릭하여 부트스트랩 애플리케이션이 성공적으로 생성되었는지 확인하세요 . 부트스트랩 ArgoCD 애플리케이션은 현재 bootstrap플랫폼 Git 저장소의 폴더를 가리키도록 구성되어 있습니다.

![](images/2025-10-29-15-16-33.png)

현재 폴더가 비어 있습니다. 다음 장에서는 애드온, 네임스페이스, 프로젝트 및 워크로드에 대한 ApplicationSet 파일로 폴더를 채울 것입니다.

![](images/2025-10-29-15-16-43.png)