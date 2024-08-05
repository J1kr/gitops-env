# Prometheus 및 Grafana 설치 (kube-prometheus-stack)

이 문서는 Prometheus와 Grafana를 포함하는 kube-prometheus-stack을 Helm chart를 사용하여 설치하는 방법을 설명합니다.

## 전제 조건

- Kubernetes 클러스터가 준비되어 있어야 합니다.
- Helm이 설치되어 있어야 합니다. (Helm 설치 방법은 [Helm 공식 문서](https://helm.sh/docs/intro/install/)를 참고하세요.)

## 설치 단계

1. **Helm 저장소 추가 및 업데이트**

    Prometheus Community의 Helm chart 저장소를 추가하고 업데이트합니다.

    ```sh
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    ```

2. **kube-prometheus-stack 설치**

    Helm chart를 사용하여 kube-prometheus-stack을 설치합니다. 이때 네임스페이스도 함께 생성합니다.

    ```sh
    helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace -f addons/prometheus-kube-stack/values.yaml
    ```

3. **접근 방법 설정**

    기본적으로 Prometheus와 Grafana는 클러스터 내부에서만 접근 가능합니다. 외부에서 접근하려면 인그레스 리소스를 설정하거나 LoadBalancer 타입의 서비스를 사용할 수 있습니다. 다음은 인그레스 설정 예시입니다.

    ```yaml
    # addons/prometheus-kube-stack/ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: grafana-ingress
      namespace: monitoring
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      rules:
      - host: grafana.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80
    ```

    위 파일을 적용하여 인그레스를 설정합니다.

    ```sh
    kubectl apply -f addons/prometheus-kube-stack/ingress.yaml
    ```

4. **Grafana 초기 관리자 비밀번호 확인**

    Grafana의 초기 관리자 비밀번호를 확인합니다.

    ```sh
    kubectl get secret --namespace monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
    ```

5. **Grafana UI 접속**

    브라우저에서 `http://grafana.example.com`에 접속하여 Grafana UI에 로그인합니다. 기본 사용자 이름은 `admin`이며, 비밀번호는 위 단계에서 확인한 초기 비밀번호입니다.

## 추가 설정

설치 후 필요에 따라 추가 설정을 진행할 수 있습니다. 예를 들어, Grafana 대시보드를 설정하거나 Prometheus 알림 규칙을 설정할 수 있습니다.

자세한 설정 방법은 [kube-prometheus-stack 공식 문서](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)를 참고하세요.
