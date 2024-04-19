---
layout: single
title: "[티어리스트] (3) Kubernets, ArgoCD, GitHub Action, Sonarqube를 통한 CICD 구축기"
categories: cicd
tag: [cicd, argocd, github-action, sonarqube,  kubernetes]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/rD4aT7s.png"
# search: false
---
# ArgoCD 배포하기

![](https://i.imgur.com/rD4aT7s.png)

쿠버네티스 클러스터 구성까지 마쳤다. 이제 GitOps를 실현하는 ArgoCD를 쿠버네티스 클러스터에 배포해보자.

## ArgoCD 배포

ArgoCD 매니페스트 파일을 제공한다. 따라서 명령어 한줄로 쿠버네티스 클러스터에 ArgoCD를 배포할 수 있다.

다음과 같이 `argocd` 네임스페이스를 만들고 해당 네임스페이스에 ArgoCD를 배포할 수 있다.

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```

다음의 명령어로 배포가 잘 되었는지 확인할 수 있다.

```sh
kubectl get all -n argocd
```

![](https://i.imgur.com/KyoyReP.png)

## Nginx Ingress Controller 설치

Ingress란 다음과 같은 기능을 제공해준다.

- 외부에서 접속가능한 URL 사용
- 트래픽 로드밸런싱
- SSL 인증서 처리
- 도메인 기반 가상 호스팅 제공

우리는 외부에서 도메인 기반으로 ArgoCD에 접속할 수 있게 하고 SSL인증서 처리를 위해 위를 적용할 것이다.

Ingress를 이용하기 위해서는 Ingress Controller가 필요한데 여러 서드파티가 컨트롤러를 제공한다. 많이 익숙한 Nginx가 만든 Nginx Controller를 사용해보자.

[공식 문서](https://kubernetes.github.io/ingress-nginx/deploy/)에서 제공하는 Quick Start 가이드를 참고하면 다음과 같이 설치할 수 있다.

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

매니페스트 파일을 이용해 위와 같이 쉽게 설치할 수 있다.

```sh
# type: NodePort로 변경
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "NodePort"}}'
```

NodePort로 변경하면 Ingress Controller를 통해 IP로 접속할 수 있다.

![](https://i.imgur.com/9rhXRtb.png)

컨트롤러 서비스가 31816으로 포트포워딩된 것을 확인할 수 있다. 따라서 서비스에 접속할 때 `192.168.0.100:31816`으로 접속할 수 있다. 만약 도메인이 있다면 설정하지 않아도 된다.

## cert-manager 설치

우리는 ArgoCD의 접속에 SSL인증서를 적용할 것이다. cert-manager는 인증서를 생성하고 갱신해주는 역할을 한다.

### helm 설치

helm을 통해 cert-manager를 설치할 것이다. helm이란 쿠버네티스 플랫폼의 패키지 매니저이다. helm을 이용하면 애플리케이션과 애플리케이션에 필요한 여러 서비스들을 한번에 배포할 수 있는 장점이 있다.

helm을 다음 명령어로 설치할 수 있다.

```sh
# helm 설치
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

helm version # helm 버전 확인
```

### cert-manager 설치

helm을 통해 다음 명령어로 `cert-manager`를 설치할 수 있다.

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0 \
  --set installCRDs=true
```

### letsencrypt SSL 인증서 발급

[Let’s Encrypt](https://letsencrypt.org/)는 SSL 인증서를 무료로 발급해주는 CA(Certificate Authorities)입니다. [여러 글로벌 기업의 후원]을 받고 있으며 모질라(Mozilla) 재단에서 ‘신뢰할 수 있는 인증 기관(Trusted CA)’ 으로 인증도 받았다. 따라서 개발용으로는 안심하고 이용할 수 있다.

Cluster Issuer 매니페스트 파일 작성해 이를 적용해보자. 

`letsencrypt-product.yaml`를 다음과 같이 작성한다.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # ACME서버 지정(도메인의 소유권을 자동으로 검증, 발급, 갱신해주는 프로토콜)
    # Letencrypt ACME 서버 URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # 자주 사용하는 이메일 주소 지정
    email: duk9741@gmail.com
    privateKeySecretRef:
      # ACME 서버와의 통신에 사용되는 ACME 계정의 Private Key를 저장할 Secret 이름
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            # Ingress Controller Class 지정
            class: nginx
```

다음 명령어로 매니페스트 파일 적용할 수 있다.

```sh
kubectl apply -f letsencrypt-product.yaml
```

### ArgoCD SSL 인증서 적용

다음과 같이 `argocd-ingress.yaml` 작성한다. host엔 원하는 도메인을 적는다. 나는 `argocd.[도메인 주소]`로 설정하고 도메인 발급 사이트에서 DNS 설정을 해주었다.

만약 도메인이 없다면 host를 `192.168.0.100/argocd`와 같이 설정하고 `annotations`에
`nginx.ingress.kubernetes.io/rewrite-target: /`를 추가하자. `/argocd/123`과 같은 접속정보를 `/123`으로 변경해준다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: [원하는 host 주소]
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
      - [원하는 host 주소]
      # do not change, this is provided by Argo CD
      secretName: argocd-secret
```

아래의 명령어로 위의 매니페스트 파일을 적용한다.

```sh
kubectl apply -f argocd-ingress.yaml
```

이제 설정한 host주소로 접속하면 보안 설정이 완료된 연결을 할 수 있다.

![](https://i.imgur.com/g6LMqT3.png)

## ArgoCD 접속

초기 패스워드를 다음과 같이 얻을 수 있다.

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

- Username : admin
- Password: 위에서 얻은 패스워드

위를 이용해 ArgoCD에 접속할 수 있다.

## 다음으로

GitOps를 돕는 ArgoCD를 배포해보았다. 이제 정적 분석을 돕는 Sonarqube를 쿠버네티스 클러스터에 배포해보자.