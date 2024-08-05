# Argo CD 설치 (Helm chart 사용)

이 문서는 Argo CD를 Helm chart를 사용하여 설치하는 방법을 설명합니다.

## 전제 조건

- infra-k3s 혹은 infra-eks 우선.
- prometheus metrics 설정 / 
  - namespace monitoring        
  - selector: 
    release: prometheus

## 설치 단계

1. **Argo CD 네임스페이스 생성**

    먼저 Argo CD를 위한 네임스페이스를 생성합니다.

    ```sh
    kubectl create namespace argocd
    ```

2. **Helm 저장소 추가 및 업데이트**

    Argo CD Helm chart를 사용하기 위해 Argo 프로젝트의 Helm 저장소를 추가하고 업데이트합니다.

    ```sh
    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo update
    ```

3. **Argo CD 설치**

    Helm chart를 사용하여 Argo CD를 설치합니다.

    ```sh
    helm install argo-cd argo/argo-cd --version 7.3.11  --namespace argocd --create-namespace

    ```

4. **접근 방법 설정**

    기본적으로 Argo CD 서버는 클러스터 내부에서만 접근 가능합니다. 외부에서 접근하려면 인그레스 리소스를 설정하거나 LoadBalancer 타입의 서비스를 사용할 수 있습니다. 다음은 인그레스 설정 예시입니다.

    ```yaml
    # pipelines/argo-cd/install/ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: argocd-server-ingress
      namespace: argocd
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      rules:
      - host: argocd.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
    ```

    위 파일을 적용하여 인그레스를 설정합니다.

    ```sh
    kubectl apply -f pipelines/argo-cd/install/ingress.yaml
    ```

5. **초기 관리자 비밀번호 확인**

    Argo CD의 초기 관리자 비밀번호를 확인합니다.

    ```sh
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```

6. **Argo CD UI 접속**

    브라우저에서 `http://argocd.example.com`에 접속하여 Argo CD UI에 로그인합니다. 기본 사용자 이름은 `admin`이며, 비밀번호는 위 단계에서 확인한 초기 비밀번호입니다.

## 추가 설정

설치 후 필요에 따라 추가 설정을 진행할 수 있습니다. 예를 들어, RBAC 설정, 인증 방법 변경 등을 설정할 수 있습니다.

자세한 설정 방법은 [Argo CD 공식 문서](https://argo-cd.readthedocs.io/en/stable/)를 참고하세요.
