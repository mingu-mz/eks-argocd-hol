# deploy-workshop 워크로드 온보딩(30분)

플랫폼 작업플랫폼 엔지니어님, 여러분의 임무는 "deploy-workshop" 애플리케이션을 온보딩하는 것입니다. 여기에는 네임스페이스를 생성하고 "deploy-workshop" 애플리케이션의 배포를 자동화하는 작업이 포함됩니다.

> 이 워크숍의 목적에 맞게 기존 워크숍과는 다른 내용으로 각색하였습니다. 
> 
> 이 후에나오는 각 이미지에서 webstore -> deploy-workshop 이라고 봐주시면됩니다.

## 네임스페이스

멀티 테넌트 환경에서는 공유 인프라에서 애플리케이션을 서로 격리하는 것을 목표로 합니다. 네임스페이스는 이러한 격리를 제공합니다. 시크릿, 구성 맵, 볼륨과 같은 모든 애플리케이션 객체는 애플리케이션 네임스페이스 내에 생성됩니다. 할당량과 제한 범위를 사용하여 각 애플리케이션이 사용하는 클러스터 리소스의 양을 제어할 수 있습니다. 또한 네트워크 정책과 RBAC(Resource Access Control)을 설정하여 애플리케이션을 더욱 격리할 수 있습니다.

![](images/2025-10-29-17-13-04.png)

이 시나리오에서는 Argo CD를 사용하여 실제 워크로드와 별도로 애플리케이션 네임스페이스를 미리 프로비저닝합니다. 워크로드를 배포할 때 Argo CD의 "CreateNamespace: true" 옵션은 사용하지 않는 것이 좋습니다. 이러한 문제 분리를 통해 플랫폼 팀과 애플리케이션 팀의 업무를 명확하게 구분할 수 있습니다.

플랫폼 팀은 각 클러스터 및 환경에 대한 가드레일과 기본값을 설정하여 일관되고 안전한 배포 방식을 보장할 책임을 맡습니다. RBAC 규칙을 정의하고, 리소스 할당량을 적용하고, 제한을 설정하고, 네트워크 정책을 구성하고, 각 네임스페이스 내에서 추가 가드레일 정책을 구현합니다. 이러한 가드레일은 규정 준수를 보장하고 클러스터 리소스 및 보안 태세에 대한 제어를 유지하는 프레임워크 역할을 합니다.

애플리케이션 팀은 프로비저닝된 네임스페이스 내에서 워크로드를 배포할 수 있는 권한을 부여받습니다. 하지만 플랫폼 팀이 적용하는 가드레일을 수정할 수는 없습니다. 이러한 책임 분리를 통해 플랫폼 팀은 클러스터의 전반적인 보안 및 리소스 관리를 제어하는 ​​동시에 애플리케이션 팀은 정의된 경계 내에서 워크로드를 배포하고 관리하는 데 집중할 수 있습니다.

이러한 접근 방식을 따르면 중앙 집중식 거버넌스와 분산형 애플리케이션 배포 간의 균형을 달성하여 안전하고 확장 가능한 Kubernetes 환경을 조성할 수 있습니다.

### deploy-workload 네임스페이스 만들기

이 장에서는 환경(개발)별 네임스페이스(UI, 주문, 체크아웃, 장바구니, 카탈로그, 자산)를 생성하기 위해 웹스토어 애플리케이션을 온보딩합니다.

'네임스페이스 및 워크로드 자동화 > 네임스페이스 자동화' 장의 자동화를 기반으로 config/deploy-workshop/namespace플랫폼 Git 저장소에 새 폴더를 추가합니다. 이 폴더와 해당 매니페스트가 Git에 푸시되면 기존 create-namespace ApplicationSet은 다음과 같이 작동합니다.

![](images/2025-10-29-17-14-54.png)

- 1. create-namespace ApplicationSet은 config/deploy-workshop/namespace 폴더를 감지합니다.
- 2. create-namespace-deploy-workshop라는 이름의 새로운 Argo CD 애플리케이션을 생성합니다.
- 3. 이 애플리케이션은 config/deploy-workshop/namespace의 매니페스트를 배포합니다.

1. deploy-workshop 네임스페이스 구성 생성

    이제 deploy-workshop workload에 대한 환경별 네임스페이스 애플리케이션을 생성하기 위해 create-namespace-env-deploy-workshop라는 ApplicationSet을 정의하겠습니다.

    ```sh
    mkdir -p $GITOPS_DIR/platform/config/deploy-workshop/namespace
    cat > $GITOPS_DIR/platform/config/deploy-workshop/namespace/namespace-deploy-workshop-applicationset.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: create-namespace-env-deploy-workshop
      namespace: argocd
    spec:
      goTemplate: true
      syncPolicy:
        preserveResourcesOnDeletion: false
      generators:
        - clusters:
            selector:
              matchLabels:
                workload_deploy-workshop: 'true'
            values:
              workload: deploy-workshop
      template:
        metadata:
          name: 'namespace-{{ .metadata.labels.environment }}-deploy-workshop'
          labels:
            environment: '{{ .metadata.labels.environment }}'
            tenant: 'deploy-workshop'
            workloads: 'true'
        spec:
          project: default
          source:
            repoURL: '{{ .metadata.annotations.platform_repo_url }}'
            path: '{{ .metadata.annotations.platform_repo_basepath }}charts/namespace'
            targetRevision: '{{ .metadata.annotations.platform_repo_revision }}'
            helm:
              releaseName: 'deploy-workshop'
              ignoreMissingValueFiles: true
              valueFiles:
                - '../../config/deploy-workshop/namespace/values/default-values.yaml'
                - '../../config/deploy-workshop/namespace/values/{{ .metadata.labels.environment }}-values.yaml'
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

    - 라인 15: workload_deploy-workshop: 'true' 레이블이 있는 클러스터만 선택됩니다.
    - 29번째 줄: charts/namespace에 있는 Helm 차트의 네임스페이스를 가리킵니다.

    ![](images/2025-10-29-17-20-47.png)

    **Helm 차트에서 파일을 확인하세요:**
    ```sh
    ├── Chart.yaml
    ├── README.md
    ├── templates
    │   ├── _helpers.tpl
    │   ├── limitrange
    │   │   └── limitrange.yaml
    │   ├── namespace
    │   │   └── namespace.yaml
    │   ├── networkpolicy
    │   │   ├── egress
    │   │   │   ├── allow-dns.yaml
    │   │   │   └── deny-all.yaml
    │   │   ├── ingress
    │   │   │   └── deny-all.yaml
    │   │   └── networkpolicy.yaml
    │   ├── rbac
    │   │   ├── rolebinding.yaml
    │   │   └── role.yaml
    │   └── resourcequota
    │       └── resourcequota.yaml
    ├── values.schema.json
    ├── values-test.yaml
    └── values.yaml
    ```

    36-37행: 기본 및 선택적 환경별 Helm 값 파일을 지정합니다.

2. deploy-workshop 네임스페이스 기본값 생성

    ```sh
    mkdir -p $GITOPS_DIR/platform/config/deploy-workshop/namespace/values
    cat > $GITOPS_DIR/platform/config/deploy-workshop/namespace/values/default-values.yaml << 'EOF'
    name: deploy-workshop
    labels:
      environment: control-plane
    namespaces:
      bluegreen:
        labels:
          additionalLabels:
            app.kubernetes.io/created-by: eks-workshop
      canary:
        labels:
          additionalLabels:
            app.kubernetes.io/created-by: eks-workshop
      rolling:
        labels:
          additionalLabels:
            app.kubernetes.io/created-by: eks-workshop
    EOF
    ```

3. hub cluster 에 deploy-workshop 워크로드 활성화

    ApplicationSet 네임스페이스는 workload_deploy-workshop: 'true'로 레이블이 지정된 클러스터만 대상으로 합니다. 허브 클러스터에 이 레이블을 활성화해 보겠습니다.

    ```sh
    sed -i 's/{ workload_webstore = true }/{ workload_webstore = true },\n    { workload_deploy-workshop = true }/g' ~/environment/hub/main.tf
    ```

    코드 조각에 의한 변경 사항은 아래와 같습니다.
    ```sh
    locals{
      .
      .

      addons = merge(
        .
        .
        .
        { workload_deploy-workshop = true }  
      )

    }
    ```

4. Terraform 적용

    이렇게 하면 허브 클러스터에서 workload_webstore: 'true'라는 레이블이 설정됩니다.

    ```sh
    cd ~/environment/hub
    terraform apply --auto-approve
    ```


5. Git 커밋

    새로 추가된 ApplicationSet 및 값 파일을 Git에 푸시합니다.

    ```sh
    cd $GITOPS_DIR/platform
    git add .
    git commit -m "add deploy-workshop namespace applicationset and namespace values"
    git push
    ```

    > **메모**
    >
    > 'create-namespace-deploy-workshop' 애플리케이션은 몇 분 후에 표시됩니다.

    ![](images/2025-10-29-17-32-52.png)


    그러면 namespace-deploy-workshop 애플리케이션은 Argo CD가 기본값을 사용하여 네임스페이스 Helm 차트를 설치하도록 합니다.

6. 네임스페이스 검증

    > **메모**
    > 
    > 네임스페이스가 생성되는 데 몇 분 정도 걸릴 수 있습니다. 몇 분 정도 기다린 후 다시 시도해 보세요.

    이 설정을 사용하면 웹스토어 네임스페이스와 해당 정책(LimitRange 및 NetworkPolicies 등)이 간단한 Git 변경 사항으로 구동되는 Argo CD 및 Helm을 사용하여 자동으로 관리됩니다.

    ```sh
    kubectl get ns --context hub-cluster --context hub-cluster
    ```

    결과
    ```sh
    NAME               STATUS   AGE
    argo-rollouts      Active   28m
    argocd             Active   7h9m
    bluegreen          Active   3m57s
    canary             Active   3m57s
    default            Active   7h22m
    external-secrets   Active   75m
    ingress-nginx      Active   95m
    kube-node-lease    Active   7h22m
    kube-public        Active   7h22m
    kube-system        Active   7h22m
    rolling            Active   3m57s
    ```

    Argo CD 대시보드에서 애플리케이션 namespace-dev-deploy-workspace를 볼 수도 있습니다.

    ![](images/2025-10-29-17-39-01.png)

## 워크로드

이 장에서는 이전에 프로비저닝한 네임스페이스에 deploy-workshop 워크로드를 배포합니다. deploy-workshop 워크로드는 이 후 진행될 배포테스트를 위해 rolling, bluegreen, canary 3가지 애플리케이션으로 구성됩니다.

![](images/2025-10-30-10-15-11.png)

deploy-workshop 워크로드는 dev, 스테이징, 프로덕션을 포함한 여러 환경을 지원합니다. 환경별 구성은 Kustomization 파일을 통해 관리됩니다.

### deploy-workshop 배포 자동화

이 장에서는 deploy-workshop 워크로드 배포를 자동화합니다.

'네임스페이스 및 워크로드 자동화' > '워크로드 자동화' 장의 자동화를 기반으로 이제 deploy-workshop 워크로드에 적용해 보겠습니다. 구체적으로, 플랫폼 Git 저장소의 config/deploy-workshop/deployment에 새 폴더를 추가합니다. 이 폴더와 해당 매니페스트가 Git에 푸시되면 기존 create-deployment ApplicationSet은 다음과 같이 동작합니다.

deploy-workshop 워크로드 배포

1. create-deployment ApplicationSet은 config/deploy-workshop/deployment 폴더를 감지합니다.
2. create-deployment-deploy-workshop라는 이름의 새로운 Argo CD 애플리케이션을 생성합니다.
3. 이 애플리케이션은 config/deploy-workshop/deployment의 매니페스트를 배포합니다.

#### Dev deploy-workshop 배포 자동화

1. Dev 배포 애플리케이션 세트 생성

    개발 deploy-workshop 워크로드를 배포하는 역할을 하는 ApplicationSet을 만들어 보겠습니다.

    ```sh
    mkdir -p $GITOPS_DIR/platform/config/deploy-workshop/deployment
    cat > $GITOPS_DIR/platform/config/deploy-workshop/deployment/deployment-dev-deploy-workshop-applicationset.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: create-deployment-dev-deploy-workshop
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
                    matchExpressions:
                      - key: workload_deploy-workshop
                        operator: In
                        values: ['true']
                      - key: environment
                        operator: In
                        values: ['dev']                  
                  values:
                    workload: deploy-workshop
              - git:
                  requeueAfterSeconds: 30
                  repoURL: '{{ .metadata.annotations.workload_repo_url }}'
                  revision: '{{ .metadata.annotations.workload_repo_revision }}'
                  directories:
                    - path: '{{ .metadata.annotations.workload_repo_basepath }}deploy-workshop/*/dev'
      template:
        metadata:
          name: 'deployment-{{ .metadata.labels.environment }}-{{ index .path.segments 1 }}-deploy-workshop'
          labels:
            environment: '{{ .metadata.labels.environment }}'
            tenant: 'deploy-workshop'
            component: '{{ index .path.segments 1 }}'
            workloads: 'true'
        spec:
          project: default
          source:
            repoURL: '{{ .metadata.annotations.workload_repo_url }}'
            path: '{{ .path.path }}'
            targetRevision: '{{ .metadata.annotations.workload_repo_revision }}'
          destination:
            namespace: '{{ index .path.segments 1 }}'
            name: '{{ .name }}'
          syncPolicy:
            automated:
              allowEmpty: true
              prune: true
            retry:
              backoff:
                duration: 1m

    EOF
    ```

    - 라인 17: 웹스토어 워크로드는 `workload_deploy-workshop = true` 및 `environment = dev` 레이블이 있는 클러스터에만 배포됩니다.
    - 라인 27: metadata.annotations.workload_repo_url 즉, 허브 클러스터의 workload_repo_url 주석은 워크로드 git 저장소의 값을 갖습니다.
    - 30번째 줄: deploy-workshop/*/dev 에 매핑합니다 (deploy-workshop 아래의 각 마이크로서비스 개발 폴더)

    deploy-workshop 폴더 구조.

    ![](images/2025-10-30-13-51-30.png)

    - 3개의 서비스(bluegreen, canary, rolling)
    - 각 서비스에는
      - base: 모든 환경에 적용되는 공통 구성을 보관합니다.
      - 환경별 디렉토리(dev, staging, prod)는 환경별 구성을 보관하므로 쉽게 재정의하고 사용자 정의할 수 있습니다.
    - deploy-workshop 개발 버전을 배포하려면 dev 폴더에 있는 모든 서비스 kustomization.yaml을 배포해야 합니다.

2. Git 커밋

    ```sh
    cd $GITOPS_DIR/platform
    git add .
    git commit -m "add bootstrap workload applicationset"
    git push
    ```

3. 배포 검증

    > **메모**
    >
    > 'create-deployment-deploy-workshop' 애플리케이션은 몇 분 후에 표시됩니다.

    ArgoCD 대시보드 > Applications > Bootstrap 으로 이동하면 워크로드별 ArgoCD 애플리케이션(예: create-deployment-deploy-workshop)을 볼 수 있습니다.

    ![](images/2025-10-30-13-58-13.png)

    'create-deployment-deploy-workshop'를 클릭하면 개발자 전용 ArgoCD 애플리케이션(예: create-deployment-dev-deploy-workshop)이 표시됩니다. 이는 이 장에서 추가한 애플리케이션입니다. 이 애플리케이션 세트는 workload/deploy-workshop/*/dev 폴더에 ArgoCD 애플리케이션을 생성할 준비가 되었습니다. 애플리케이션 저장소에 아직 코드가 없으므로 어떤 애플리케이션도 배포하지 않았습니다.

    ![](images/2025-10-30-14-02-44.png)


4. 온보드 스테이징 및 프로덕션

    스테이징과 프로덕션을 모두 배포하기 위한 구성도 만들어 보겠습니다.

    ```sh
    sed -e 's/dev/staging/g' < ${GITOPS_DIR}/platform/config/deploy-workshop/deployment/deployment-dev-deploy-workshop-applicationset.yaml > ${GITOPS_DIR}/platform/config/deploy-workshop/deployment/deployment-staging-deploy-workshop-applicationset.yaml
    sed -e 's/dev/prod/g' < ${GITOPS_DIR}/platform/config/deploy-workshop/deployment/deployment-dev-deploy-workshop-applicationset.yaml > ${GITOPS_DIR}/platform/config/deploy-workshop/deployment/deployment-prod-deploy-workshop-applicationset.yaml
    ```

2. Git 커밋

    ```sh
    cd $GITOPS_DIR/platform
    git add .
    git commit -m "add bootstrap workload applicationset"
    git push
    ```

    > **메모**
    > 
    > '스테이징' 및 '프로덕션' 애플리케이션은 몇 분 후에 표시됩니다.

    ![](images/2025-10-30-14-06-22.png)

# deploy-workshop 워크로드 배포

이 장에서는 응용 프로그램 팀이 플랫폼 팀의 직접적인 참여 없이 웹스토어 애플리케이션을 독립적으로 배포합니다.

## deploy-workshop manifest 생성

이 장에서는 deploy-workshop 워크로드를 위한 kubernetes manifest를 생성합니다.

### deploy-workshop rolling 배포

Rolling 배포는 가장 기본적인 배포 전략으로, Pod를 하나씩 순차적으로 교체합니다.

1. base manifest

    ```sh
    mkdir -p $GITOPS_DIR/workload/deploy-workshop/rolling/base
    cat > $GITOPS_DIR/workload/deploy-workshop/rolling/base/rollout.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    metadata:
      name: rollout-rolling
    spec:
      revisionHistoryLimit: 2
      selector:
        matchLabels:
          app: rollout-rolling
      template:
        metadata:
          labels:
            app: rollout-rolling
        spec:
          containers:
          - name: rollouts-demo
            image: argoproj/rollouts-demo:blue
            imagePullPolicy: Always
            ports:
            - containerPort: 8080
            resources:  # ← HPA를 위해 필수!
              requests:
                cpu: 10m
                memory: 128Mi
      strategy:
        canary:
          # Pod를 순차적으로 교체
          steps:
          - setWeight: 33    # 1개 Pod 업데이트 (3개 중 33%)
          - pause: {duration: 10s}
          - setWeight: 67    # 2개 Pod 업데이트 (3개 중 67%)
          - pause: {duration: 10s}
          - setWeight: 100   # 모든 Pod 업데이트
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/rolling/base/nlb.yaml << 'EOF'
    apiVersion: v1
    kind: Service
    metadata:
      name: rollout-rolling
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: external
        service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
        service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
    spec:
      type: LoadBalancer
      ports:
        - port: 80
          targetPort: 8080
          name: http
      selector:
        app: rollout-rolling  # 모든 버전의 Pod 선택
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/rolling/base/hpa.yaml << 'EOF'
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: rollout-rolling-hpa
    spec:
      scaleTargetRef:
        apiVersion: argoproj.io/v1alpha1
        kind: Rollout
        name: rollout-rolling
      minReplicas: 1
      maxReplicas: 5
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/rolling/base/kustomization.yaml << 'EOF'
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - rollout.yaml
    - nlb.yaml
    - hpa.yaml
    EOF
    ```

2. dev manifest 생성

    ```sh
    mkdir $GITOPS_DIR/workload/deploy-workshop/rolling/dev
    cat > $GITOPS_DIR/workload/deploy-workshop/rolling/dev/hpa.yaml << 'EOF'
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: rollout-rolling-hpa
    spec:
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/rolling/dev/kustomization.yaml << 'EOF'
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - ../base

    patchesStrategicMerge:
    - rollout.yaml
    - hpa.yaml
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/rolling/dev/rollout.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    metadata:
      name: rollout-rolling
    spec:
      template:
        spec:
          containers:
          - name: rollouts-demo
            image: argoproj/rollouts-demo:blue
            imagePullPolicy: Always
            ports:
            - containerPort: 8080
            resources:  # ← HPA를 위해 필수!
              requests:
                cpu: 10m
                memory: 128Mi
    EOF
    ```

3. git push

    ```sh
    cd $GITOPS_DIR/workload
    git add .
    git commit -m "add rolling workload"
    git push
    ```

    ![](images/2025-10-30-14-13-20.png)

4. workload 검증

    > **중요**
    > 
    > Argo CD가 동기화되고 Karpenter가 추가 노드를 프로비저닝하는 데 몇 분이 걸립니다. 또한 로드 밸런서가 올바르게 프로비저닝되는 데도 몇 분이 걸립니다.

    ```sh
    export ROLLING_URL=$(kubectl --context hub-cluster get svc -n rolling rollout-rolling -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    echo "Rolling URL: http://$ROLLING_URL"
    ```

    ![](images/2025-10-30-14-43-49.png)


### deploy-workshop bluegreen 배포

Bluegreen 배포는 운영 환경에서 많이 사용되는 배포 전략으로 서비스의 중단 없이 다음 버전을 적용할 수 있다는 이점이 있습니다.

1. base manifest

    ```sh
    mkdir -p $GITOPS_DIR/workload/deploy-workshop/bluegreen/base
    cat > $GITOPS_DIR/workload/deploy-workshop/bluegreen/base/rollout.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    metadata:
      name: rollout-bluegreen
    spec:
      revisionHistoryLimit: 2
      selector:
        matchLabels:
          app: rollout-bluegreen
      strategy:
        blueGreen:
          activeService: rollout-bluegreen-active
          previewService: rollout-bluegreen-preview
          autoPromotionEnabled: false
      template:
        metadata:
          labels:
            app: rollout-bluegreen
        spec:
          containers:
          - name: rollouts-demo
            image: argoproj/rollouts-demo:blue
            imagePullPolicy: Always
            ports:
            - containerPort: 8080
            resources:  # ← HPA를 위해 필수!
              requests:
                cpu: 10m
                memory: 128Mi
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/bluegreen/base/nlb.yaml << 'EOF'
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: rollout-bluegreen-active
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: external
        service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
        service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
    spec:
      type: LoadBalancer
      ports:
        - port: 80
          targetPort: 8080
          name: http
      selector:
        app: rollout-bluegreen
      

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: rollout-bluegreen-preview
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: external
        service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
        service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
    spec:
      type: LoadBalancer
      ports:
        - port: 80
          targetPort: 8080
          name: http
      selector:
        app: rollout-bluegreen
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/bluegreen/base/hpa.yaml << 'EOF'
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: rollout-bluegreen-hpa
    spec:
      scaleTargetRef:
        apiVersion: argoproj.io/v1alpha1
        kind: Rollout
        name: rollout-bluegreen
      minReplicas: 1
      maxReplicas: 5
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/bluegreen/base/kustomization.yaml << 'EOF'
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - rollout.yaml
    - nlb.yaml
    - hpa.yaml
    EOF
    ```

2. dev manifest 생성

    ```sh
    mkdir -p $GITOPS_DIR/workload/deploy-workshop/bluegreen/dev
    cat > $GITOPS_DIR/workload/deploy-workshop/bluegreen/dev/kustomization.yaml << 'EOF'
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - ../base

    patchesStrategicMerge:
    - rollout.yaml
    - hpa.yaml
    EOF
    ```


    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/bluegreen/dev/rollout.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    metadata:
      name: rollout-bluegreen
    spec:
      template:
        spec:
          containers:
          - name: rollouts-demo
            image: argoproj/rollouts-demo:blue
            imagePullPolicy: Always
            ports:
            - containerPort: 8080
            resources:  # ← HPA를 위해 필수!
              requests:
                cpu: 10m
                memory: 128Mi
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/bluegreen/dev/hpa.yaml << 'EOF'
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: rollout-bluegreen-hpa
    spec:
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    EOF
    ```

3. git push

    ```sh
    cd $GITOPS_DIR/workload
    git add .
    git commit -m "add bluengreen workload"
    git push
    ```

    ![](images/2025-10-30-14-21-31.png)


4. workload 검증

    > **중요**
    > 
    > Argo CD가 동기화되고 Karpenter가 추가 노드를 프로비저닝하는 데 몇 분이 걸립니다. 또한 로드 밸런서가 올바르게 프로비저닝되는 데도 몇 분이 걸립니다.

    ```sh
    export BLUE_URL=$(kubectl --context hub-cluster get svc -n bluegreen rollout-bluegreen-active -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    export GREEN_URL=$(kubectl --context hub-cluster get svc -n bluegreen rollout-bluegreen-preview -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    echo "Blue/Grenn Blue URL: http://$BLUE_URL"
    echo "Blue/Grenn Green URL: http://$GREEN_URL"
    ```

    Blue(active)
    ![](images/2025-10-30-14-44-39.png)

    Green(preview): 현재는 첫 배포라 active, preview가 같습니다.
    ![](images/2025-10-30-14-45-03.png)


### deploy-workshop canary 배포

Canary 배포는 운영 환경에서 신규 버전을 점진적으로 배포하는 전략으로 실제 유저 요청을 살펴보며 승인단계에 따라 신규 버전의 비율을 높여가는 배포전략입니다.

1. base manifest

    ```sh
    mkdir -p $GITOPS_DIR/workload/deploy-workshop/canary/base
    cat > $GITOPS_DIR/workload/deploy-workshop/canary/base/rollout.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    metadata:
      name: rollout-canary
    spec:
      revisionHistoryLimit: 2
      selector:
        matchLabels:
          app: rollout-canary
      template:
        metadata:
          labels:
            app: rollout-canary
        spec:
          containers:
          - name: rollouts-demo
            image: argoproj/rollouts-demo:blue
            imagePullPolicy: Always
            ports:
            - containerPort: 8080
            resources:  # ← HPA를 위해 필수!
              requests:
                cpu: 10m
                memory: 128Mi
      strategy:
        canary:
          canaryService: rollout-canary-canary
          stableService: rollout-canary-stable
          steps:
          - setWeight: 20
          - pause: {}
          - setWeight: 40    # 전체 Pod의 40%를 카나리 버전으로
          - pause: {duration: 10s}
          - setWeight: 60
          - pause: {duration: 10s}
          - setWeight: 80
          - pause: {duration: 10s}
    EOF
    ```

    ```sh
    # 주의: Argo Rollouts가 자동으로 selector를 관리합니다
    # - stableService: stable 버전 Pod만 선택
    # - canaryService: canary 버전 Pod만 선택
    cat > $GITOPS_DIR/workload/deploy-workshop/canary/base/nlb.yaml << 'EOF'
    apiVersion: v1
    kind: Service
    metadata:
      name: rollout-canary-stable
    spec:
      type: ClusterIP
      ports:
        - port: 80
          targetPort: 8080
          name: http
      selector:
        app: rollout-canary
        # Argo Rollouts가 자동으로 추가: rollouts-pod-template-hash: <stable-hash>

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: rollout-canary-canary
    spec:
      type: ClusterIP
      ports:
        - port: 80
          targetPort: 8080
          name: http
      selector:
        app: rollout-canary
        # Argo Rollouts가 자동으로 추가: rollouts-pod-template-hash: <canary-hash>

    ---
    # 실제 사용자 트래픽을 받는 Service (stable + canary 모두)
    apiVersion: v1
    kind: Service
    metadata:
      name: rollout-canary
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: external
        service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
        service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
    spec:
      type: LoadBalancer
      ports:
        - port: 80
          targetPort: 8080
          name: http
      selector:
        app: rollout-canary  # stable과 canary Pod 모두 선택
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/canary/base/hpa.yaml << 'EOF'
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: rollout-canary-hpa
    spec:
      scaleTargetRef:
        apiVersion: argoproj.io/v1alpha1
        kind: Rollout
        name: rollout-canary
      minReplicas: 1
      maxReplicas: 5
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    EOF
    ```

    ```sh
    # $GITOPS_DIR/workload/deploy-workshop/canary/base/kustomization.yaml
    cat > $GITOPS_DIR/workload/deploy-workshop/canary/base/kustomization.yaml << 'EOF'
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - rollout.yaml
    - nlb.yaml
    - hpa.yaml
    EOF
    ```

2. dev manifest 생성

    ```sh
    mkdir $GITOPS_DIR/workload/deploy-workshop/canary/dev
    cat > $GITOPS_DIR/workload/deploy-workshop/canary/dev/hpa.yaml << 'EOF'
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: rollout-canary-hpa
    spec:
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/canary/dev/kustomization.yaml << 'EOF'
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - ../base

    patchesStrategicMerge:
    - rollout.yaml
    - hpa.yaml
    EOF
    ```

    ```sh
    cat > $GITOPS_DIR/workload/deploy-workshop/canary/dev/rollout.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    metadata:
      name: rollout-canary
    spec:
      template:
        spec:
          containers:
          - name: rollouts-demo
            image: argoproj/rollouts-demo:blue
            imagePullPolicy: Always
            ports:
            - containerPort: 8080
            resources:  # ← HPA를 위해 필수!
              requests:
                cpu: 10m
                memory: 128Mi
    EOF
    ```

3. git push

    ```sh
    cd $GITOPS_DIR/workload
    git add .
    git commit -m "add canary workload"
    git push
    ```

    ![](images/2025-10-30-14-39-33.png)

4. workload 검증

    > **중요**
    > 
    > Argo CD가 동기화되고 Karpenter가 추가 노드를 프로비저닝하는 데 몇 분이 걸립니다. 또한 로드 밸런서가 올바르게 프로비저닝되는 데도 몇 분이 걸립니다.

    ```sh
    export CANARY_URL=$(kubectl --context hub-cluster get svc -n canary rollout-canary -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    echo "Canary Main URL: http://$CANARY_URL"
    ```

    ![](images/2025-10-30-14-42-36.png)
