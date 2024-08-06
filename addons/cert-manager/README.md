# cert-manager 설치 및 Route 53 DNS01 인증 설정 가이드 (Helm)

이 가이드는 Kubernetes 클러스터에서 웹사이트 보안을 위한 SSL/TLS 인증서를 자동으로 발급하고 관리하는 cert-manager를 Helm을 사용하여 설치하고, Amazon Route 53 DNS 서비스를 통해 도메인 소유권을 확인하는 방법을 설명합니다.

## 사전 조건
1. **Kubernetes 클러스터**: EKS, K3s 등 Kubernetes 환경
2. **Helm 3**: Helm 패키지 관리 도구
3. **Route 53 등록 도메인**: Amazon Route 53에 등록된 도메인
4. **AWS IAM 자격 증명**: Route 53 DNS 설정 변경 권한이 있는 AWS 계정 정보 (Access Key ID, Secret Access Key)

## 목차

  - [1. Cert-Manager 설치 준비](#1-cert-manager-설치-준비)
  - [2. Cert-Manager 설치](#2-cert-manager-설치)
  - [3. AWS Route 53 정보 등록](#3-aws-route-53-정보-등록)
  - [4. 인증서 발급 기관 설정](#4-인증서-발급-기관-설정)
  - [5. 인증서 발급 요청](#5-인증서-발급-요청)
  - [6. 외부 접속 설정](#6-외부-접속-설정)
  - [7. 내부 환경에 인증서 적용](#7-내부-환경에-인증서-적용)
  - [8. 브라우저에서 도메인으로 접근](#8-브라우저에서-도메인으로-접근)
  - [참고 사항](#참고-사항)
  - [추가 정보](#추가-정보)


## 1. Cert-Manager 설치 준비

Cert-Manager 설치 파일이 있는 곳을 Helm에게 알려주고, 최신 정보를 가져옵니다.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

## 2. Cert-Manager 설치

Helm을 사용하여 Cert-Manager를 설치합니다.

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

## 3. AWS Route 53 정보 등록

Route 53에 접근하기 위한 AWS 계정 정보를 Kubernetes에 안전하게 저장합니다.

```bash
kubectl create secret generic route53-credentials-secret \
  --namespace cert-manager \
  --from-literal=aws-access-key-id=<YOUR_ACCESS_KEY_ID> \
  --from-literal=aws-secret-access-key=<YOUR_SECRET_ACCESS_KEY>
```

## 4. 인증서 발급 기관 설정

Let's Encrypt라는 무료 인증 기관을 사용하여 인증서를 발급받도록 설정합니다.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <YOUR_EMAIL>
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        route53:
          region: <YOUR_AWS_REGION>
          hostedZoneID: <YOUR_HOSTED_ZONE_ID>
          accessKeyIDSecretRef:
            name: route53-credentials-secret
            key: aws-access-key-id
          secretAccessKeySecretRef:
            name: route53-credentials-secret
            key: aws-secret-access-key
```

위 내용을 `clusterissuer.yaml` 파일로 저장한 후, 다음 명령어로 적용합니다.

```bash
kubectl apply -f clusterissuer.yaml
```

## 5. 인증서 발급 요청

어떤 도메인에 대한 인증서를 발급받을지 Cert-Manager에게 알려줍니다.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-cert
  namespace: default
spec:
  secretName: my-cert-secret
  duration: 90d
  renewBefore: 30d
  commonName: myapp.example.com
  dnsNames:
  - myapp.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

위 내용을 `certificate.yaml` 파일로 저장한 후, 다음 명령어로 적용합니다.

```bash
kubectl apply -f certificate.yaml
```

## 6. 외부 접속 설정

Kubernetes Ingress를 설정하여 외부에서 웹사이트에 접속할 수 있도록 하고, 암호화된 통신을 위해 발급받은 인증서를 사용합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: my-cert-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

위 내용을 `ingress.yaml` 파일로 저장한 후, 다음 명령어로 적용합니다.

```bash
kubectl apply -f ingress.yaml
```

## 7. 내부 환경에 인증서 적용

외부 환경에서 발급받은 인증서를 내부 환경의 Kubernetes 클러스터에 시크릿으로 적용합니다.

```bash
kubectl create secret tls my-tls-secret --key tls.key --cert tls.crt -n <your-namespace>
```

## 8. 브라우저에서 도메인으로 접근

브라우저에서 `https://myapp.example.com`으로 접근하여 Let's Encrypt 인증서를 사용한 HTTPS 연결을 테스트합니다.

## 참고 사항

* Let's Encrypt 외 다른 인증 기관을 사용하려면 `ClusterIssuer` 설정을 변경해야 합니다.
* `example.com`은 실제 도메인 이름으로 바꿔주세요.
* AWS 계정 정보는 유출되지 않도록 주의하세요.

## 추가 정보

이 가이드는 기본적인 설정 방법을 안내하며, 실제 운영 환경에서는 보안 및 성능을 위해 추가 설정이 필요할 수 있습니다. 자세한 내용은 [Cert-Manager 공식 문서](https://cert-manager.io/docs/)를 참고하세요.
```

이 `README.md` 파일을 통해 Cert-Manager 설치 및 인증서 발급 과정을 명확하게 안내할 수 있습니다. 필요한 내용을 포함하여 간결하게 작성했습니다. 이 파일을 Cert-Manager 폴더에 추가하여 사용하면 됩니다.

[def]: #사전-조건
