# 네임스페이스 및 워크로드 자동화(20분)

애플리케이션 배포는 2단계 프로세스입니다.

- 플랫폼 팀은 네임스페이스를 만들고 배포를 자동화하여 애플리케이션을 온보딩합니다.
- 개발자는 제공된 자동화를 사용하여 애플리케이션을 배포합니다.

이 섹션에서는 다음 역할을 맡게 됩니다. 플랫폼 엔지니어를 고용하고 애플리케이션 온보딩을 쉽고 반복 가능하게 만드는 자동화를 설정합니다.

다음 폴더 구조를 **플랫폼 Git** 저장소에 사용하여 애플리케이션을 온보딩합니다.

![](images/2025-10-29-16-30-24.png)

이 구조는 새 애플리케이션 온보딩을 구성 파일을 올바른 위치에 커밋하는 것만큼 간편하게 만들어 줍니다. Argo CD는 이 설정에 따라 자동으로 네임스페이스를 생성하고 애플리케이션을 배포합니다.

## 네임스페이스 자동화

이 장의 목표는 각 워크로드에 대한 네임스페이스 생성을 관리하는 ArgoCD 애플리케이션을 만드는 것입니다. 이는 각 워크로드 namespace폴더에 있는 매니페스트를 배포하는 부트스트랩 수준 애플리케이션입니다.

예를 들어, `create-namespace-workload-a` ArgoCD 애플리케이션은 `workload-a/namespace`폴더에 있는 매니페스트를 배포하는 역할을 합니다.

![](images/2025-10-29-16-32-36.png)

각 워크로드에 대한 ArgoCD 네임스페이스 애플리케이션을 만들려면 ApplicationSet을 사용합니다. 이전 부트스트랩 장에서는 플랫폼 Git 저장소에서 `bootstrap/`폴더를 지속적으로 모니터링하는 ArgoCD 애플리케이션을 만들었습니다. 이 장에서는 해당 폴더에 네임스페이스 ApplicationSet을 추가합니다.

![](images/2025-10-29-16-33-44.png)

1. Bootstrap 네임스페이스 애플리케이션 세트 생성

    플랫폼 Git 저장소의 폴더 `bootstrap/`아래에 다음과 같은 파일을 만듭니다.`namespace-applicationset.yaml`

    ```sh
    cat > $GITOPS_DIR/platform/bootstrap/namespace-applicationset.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: create-namespace
      namespace: argocd
    spec:
      goTemplate: true
      syncPolicy:
        preserveResourcesOnDeletion: false
      generators:
        - matrix:
            generators:
              - clusters:
                  selector:
                    matchLabels:
                      fleet_member: hub
              - git:
                  repoURL: '{{ .metadata.annotations.platform_repo_url }}'
                  revision: '{{ .metadata.annotations.platform_repo_revision }}'
                  directories:
                    - path: 'config/*/namespace'
      template:
        metadata:
          name: 'create-namespace-{{ index .path.segments 1 }}'
          labels:
            environment: '{{ .metadata.labels.environment }}'
            tenant: '{{ index .path.segments 1 }}'
            workloads: 'true'
            
        spec:
          project: default
          source:
            repoURL: '{{ .metadata.annotations.platform_repo_url }}'
            path: '{{ .path.path }}'
            targetRevision: '{{ .metadata.annotations.platform_repo_revision }}'
          destination:
            name: '{{ .name }}'
          syncPolicy:
            automated:
              allowEmpty: true
            retry:
              backoff:
                duration: 1m
                #limit: 100
            syncOptions:
              - CreateNamespace=true
    EOF
    ```

    이 ApplicationSet은 모든 워크로드에 대한 네임스페이스별 ArgoCD 애플리케이션 생성을 시작합니다.

    - 12번째 줄 : 행렬 생성기는 내부 생성기(git과 cluster)의 출력을 결합하여 순열을 생성합니다.
    - 22번째 줄 : Git 생성기는 config/*/namespace플랫폼 Git 저장소 아래의 각 폴더를 반복합니다.
    - 35번째 줄 : {{ .path.path }}각 네임스페이스 폴더 경로에 매핑됩니다 config/*/namespace.
      - 예를 들어, 의 경우 workload-a경로는 .입니다 config/workload-a/namespace.
      - 아래에 폴더가 없으면 config/*아직 애플리케이션이 생성되지 않습니다.

2. Git 커밋

    ```sh
    cd $GITOPS_DIR/platform
    git add .
    git commit -m "add bootstrap namespace applicationset"
    git push
    ```

    Push 후 ArgoCD 대시보드로 이동하여 부트스트랩 애플리케이션을 엽니다. 새로 생성된 create-namespace ApplicationSet이 표시됩니다.

    > 메모
    > 
    > 'create-namespace' 애플리케이션 세트는 몇 분 후에 표시됩니다.

    ![](images/2025-10-29-16-38-39.png)

## 워크로드 자동화

이전 장의 네임스페이스 자동화와 유사하게, 이 장의 목표는 각 워크로드에 대한 ArgoCD 애플리케이션을 생성하여 워크로드 배포를 관리하는 것입니다. 이는 각 워크로드 deployment폴더에 있는 매니페스트를 배포하는 부트스트랩 수준 애플리케이션입니다.

예를 들어, `create-deployment-workload-a`ArgoCD 애플리케이션은 `workload-a/deployment`폴더에 있는 매니페스트를 배포하는 역할을 합니다.

![](images/2025-10-29-16-40-20.png)

각 워크로드에 대한 ArgoCD 배포 애플리케이션을 생성하려면 ApplicationSet을 사용합니다. 이전 부트스트랩 챕터에서는 bootstrap/플랫폼 Git 저장소의 폴더를 지속적으로 감시하는 ArgoCD 애플리케이션을 생성했습니다. 이번 챕터에서는 해당 폴더에 배포 ApplicationSet을 추가합니다.

1. 부트스트랩 워크로드 애플리케이션 세트 생성

    플랫폼 Git 저장소의 `bootstrap/`폴더 아래에 다음과 같은 파일을 만듭니다.`workload-applicationset.yaml`

    ![](images/2025-10-29-16-40-29.png)

    ```sh
    cat > $GITOPS_DIR/platform/bootstrap/workload-applicationset.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: create-deployment
      namespace: argocd
    spec:
      goTemplate: true
      syncPolicy:
        preserveResourcesOnDeletion: false
      generators:
        - matrix:
            generators:
              - clusters:
                  selector:
                    matchLabels:
                      fleet_member: 'hub'
              - git:
                  repoURL: '{{ .metadata.annotations.platform_repo_url }}'
                  revision: '{{ .metadata.annotations.platform_repo_revision }}'
                  directories:
                    - path: '{{ .metadata.annotations.platform_repo_basepath }}config/*/deployment'
      template:
        metadata:
          name: 'create-deployment-{{ index .path.segments 1 }}'
          labels:
            environment: '{{ .metadata.labels.environment }}'
        spec:
          project: default
          source:
            repoURL: '{{ .metadata.annotations.platform_repo_url }}'
            path: '{{ .path.path }}'
            targetRevision: '{{ .metadata.annotations.platform_repo_revision }}'
          destination:
            name: '{{ .name }}'
          syncPolicy:
            automated:
              allowEmpty: true
            retry:
              backoff:
                duration: 1m
                #limit: 100
            syncOptions:
              - CreateNamespace=true

    EOF
    ```

    이 ApplicationSet은 모든 워크로드에 대한 배포별 ArgoCD 애플리케이션 생성을 시작합니다.

    - 12번째 줄 : 행렬 생성기는 내부 생성기(git과 cluster)의 출력을 결합하여 순열을 생성합니다.
    - 22번째 줄 : Git 생성기는 config/*/deployment플랫폼 Git 저장소 아래의 각 폴더를 반복합니다.
    - 35번째 줄 : {{ .path.path }}각 네임스페이스 폴더 경로에 매핑됩니다 config/*/deployment.
        - 예를 들어, 의 경우 workload-a경로는 .입니다 config/workload-a/deployment.
        - 아래에 폴더가 없으면 config/*아직 애플리케이션이 생성되지 않습니다.

2. Git 커밋

    ```sh
    cd $GITOPS_DIR/platform
    git add .
    git commit -m "add bootstrap workload applicationset"
    git push
    ```

    > **메모**
    > 
    > 'create-deployment' 애플리케이션 세트는 몇 분 후에 표시됩니다.

    부트스트랩 폴더가 모니터링 되므로 workload-applicationset.yaml 과 같은 새 파일이 추가되면 처리됩니다.

    ![](images/2025-10-29-16-41-49.png)

## Karpenter Node 자동화

이전 장의 네임스페이스, 워크로드 자동화와 유사하게, 이 장의 목표는 각 Karpenter Node에 대한 ArgoCD 애플리케이션을 생성하여 관리하는 것입니다. 이는 karpenter-node워크로드를 만들어 폴더에 있는 매니페스트를 배포하는 부트스트랩 수준 애플리케이션입니다.

예를 들어, `create-karpenter-node`ArgoCD 애플리케이션은 `karpenter-node/`폴더에 있는 매니페스트를 배포하는 역할을 합니다.

각 워크로드에 대한 ArgoCD 배포 애플리케이션을 생성하려면 ApplicationSet을 사용합니다. 이전 부트스트랩 챕터에서는 bootstrap/플랫폼 Git 저장소의 폴더를 지속적으로 감시하는 ArgoCD 애플리케이션을 생성했습니다. 이번 챕터에서는 해당 폴더에 karpenter ApplicationSet을 추가합니다.

1. 부트스트랩 워크로드 애플리케이션 세트 생성

    플랫폼 Git 저장소의 `bootstrap/`폴더 아래에 다음과 같은 파일을 만듭니다.`karpenter-node-applicationset.yaml`

    ```sh
    cat > $GITOPS_DIR/platform/bootstrap/karpenter-node-applicationset.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: create-karpenter-node
      namespace: argocd
    spec:
      goTemplate: true
      syncPolicy:
        preserveResourcesOnDeletion: false
      generators:
        - matrix:
            generators:
              - clusters:
                  selector:
                    matchLabels:
                      fleet_member: 'hub'
              - git:
                  repoURL: '{{ .metadata.annotations.platform_repo_url }}'
                  revision: '{{ .metadata.annotations.platform_repo_revision }}'
                  directories:
                    - path: '{{ .metadata.annotations.platform_repo_basepath }}config/karpenter-node'
      template:
        metadata:
          name: 'create-karpenter-nod'
          labels:
            environment: '{{ .metadata.labels.environment }}'
        spec:
          project: default
          source:
            repoURL: '{{ .metadata.annotations.platform_repo_url }}'
            path: '{{ .path.path }}'
            targetRevision: '{{ .metadata.annotations.platform_repo_revision }}'
          destination:
            name: '{{ .name }}'
          syncPolicy:
            automated:
              allowEmpty: true
              prune: true
            retry:
              backoff:
                duration: 1m
                #limit: 100
    EOF
    ```

    이 ApplicationSet은 모든 워크로드에 대한 배포별 ArgoCD 애플리케이션 생성을 시작합니다.

2. Git 커밋

    ```sh
    cd $GITOPS_DIR/platform
    git add .
    git commit -m "add bootstrap karpenter-node applicationset"
    git push
    ```

    > **메모**
    > 
    > 'create-deployment' 애플리케이션 세트는 몇 분 후에 표시됩니다.

    부트스트랩 폴더가 모니터링 되므로 karpenter-node-applicationset.yaml 과 같은 새 파일이 추가되면 처리됩니다.

    ![](images/2025-10-29-16-55-29.png)