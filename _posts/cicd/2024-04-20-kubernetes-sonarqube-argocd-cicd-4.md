---
layout: single
title: "[티어리스트] (4) Kubernets, ArgoCD, GitHub Action, Sonarqube를 통한 CICD 구축기"
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
# Sonarqube 배포하기

![](https://i.imgur.com/rD4aT7s.png)

Sonarqube란 정적 분석을 통해 코드 품질을 높이고, 유지보수를 할 때 도움을 준다. 코드 컨벤션부터 잠재적인 위험 사항까지 체크를 해주어 품질 향상에 도움을 준다.

쿠버네티스 클러스터에 Sonarqube를 배포해보자. Argocd를 설치했으니 GitOps를 통해 Sonarqube를 배포해보자.

## ArgoCD로 Sonarqube 배포하기

가장 먼저 GitOps가 될 GitHub 레포지토리를 설정해야한다. 해당 레포지토리는 ArgoCD가 주시하고 있다가 변경이 생기면 레포지토리의 매니페스트 파일에 따라 배포를 진행해준다.

![](https://i.imgur.com/QkE2xOi.png)

위와 같이 `sonarqube` 디렉토리를 레포지토리 루트에 생성한다. 다른 디렉토리는 신경쓰지 말자.

### 쿠버네티스 Secret 생성

Sonarqube와 Sonarqube가 사용할 Postgres의 패스워드를 쿠버네티스 Secret으로 생성하자.

가장 먼저 bas64를 통해 비밀을 인코딩 해야한다.

```sh
echo '[원하는 패스워드]' | base64
```

위에서 나온 결과를 복사해둔다. 그리고 `posrgres-password-secret.yaml`을 다음과 같이 작성한다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sonarqube-postgres
type: Opaque
data:
  password: [base64 인코딩 결과]
```

다음과 같이 Secret를 적용한다.

```sh
kubectl create ns sonarqube
kubectl apply -f posrgres-password-secret.yaml -n sonarqube
```

네임스페이스를 만들고 Secret를 생성했다.

### 쿠버네티스 PV 생성

Sonarqube 매니페스트에서 PVC를 이용해 볼륨을 요청할 것이다. 따라서 바인딩 될 PV가 필요하다. postgres용 pv, sonarqube 데이터 용 pv, sonarqube 익스텐션 용 pv 3개를 생성하자.

`sonar-data-pv.yaml`을 다음과 같이 작성하자.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sonarqube-data-pv
spec:
  storageClassName: sonarqube-data
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /sonarqube/data
```

sonarqube에서 권장하는 용량이 있지만 리소스 문제로 인해 작게 설정하였다. 리소스가 충분한 경우 30기가 이상으로 설정해주자.

`sonar-extentions-pv.yaml`을 다음과 같이 작성하자.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sonarqube-extentions-pv
spec:
  storageClassName: sonarqube-extentions
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /sonarqube/extentions
```

적당히 5기가로 생성했다.

마지막으로 postgres를 위한 pv 매니페스트를 작성해보자.

`sonar-postgres-pv.yaml`을 다음과 같이 작성한다.

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: sonarqube-postgres-pv
spec:
  storageClassName: sonarqube-postgres-data
  capacity:
    storage: 8Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /data/postgresql
```

다음 명령어로 작성한 매니페스트 파일을 적용하자.

```
kubectl apply -f sonar-postgres-pv.yaml sonar-extentions-pv.yaml sonar-data-pv.yaml
```

### Sonarqube Postgres 매니페스트 작성

Sonarqube를 그냥 띄우면 H2 인메모리 DB를 사용한다. 장기간 사용시 문제가 될 수 있으니 Postgres를 연결해 사용하는 것이 안전하다.

가장 먼저 postgres Deployment를 작성하자. `sonar-posrgres-deployment.yaml`로 작성했다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarqube-postgres
  namespace: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      name: sonarqube-postgres
  template:
    metadata:
      name: sonarqube-postgres
      labels:
        name: sonarqube-postgres
    spec:
      containers:
        - image: postgres:16.2
          name: sonarqube-postgres
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-password
                  key: password
            - name: POSTGRES_USER
              value: sonar
          ports:
            - containerPort: 5432
              name: postgresport
          volumeMounts:
            # This name must match the volumes.name below.
            - name: data-disk
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data-disk
          persistentVolumeClaim:
            claimName: postgres-pvc
```

`env`의 패스워드 부분은 쿠버네티스 Secret이 사용될 것이다.

다음은 서비스를 작성하자.

`sonar-postgres-service.yaml`을 다음과 같이 작성하자.

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: sonarqube
  labels:
    name: sonarqube-postgres
  name: sonarqube-postgres
spec:
  ports:
    - port: 5432
  selector:
    name: sonarqube-postgres
```

마지막으로 postgres의 PVC를 작성하자.

`sonar-postgres-pvc.yaml`를 다음과 같이 작성한다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: sonarqube
spec:
  storageClassName: sonarqube-postgres-data
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

### Sonarqube 매니페스트 작성


소나큐브가 PR에 분석 결과를 Comment로 달 수 있게 플러그인을 추가할 것이다. [sonarqube-community-branch-plugin](https://github.com/mc1arke/sonarqube-community-branch-plugin)이다. 커뮤니티 버전에서도 PR Comment을 사용할 수 있게 해준다. README를 살펴보면 Sonarqube 10.3.0버전은 플러그인 버전 1.18.0을 지원하므로 1.18.0버전으로 설치해보자.

`args`에 브랜치 플러그인 관련 설정 정보를 넣어준다.

`sonar-deployment-yaml`을 다음과 같이 작성한다. Secret에서 DB Password를 가져온다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarqube-deployment
  labels:
    app: sonarqube
  namespace: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      containers:
        - name: sonarqube
          image: sonarqube:10.3.0-community
          args:
            - -Dsonar.web.javaAdditionalOpts=-javaagent:./extensions/plugins/sonarqube-community-branch-plugin-1.18.0.jar=web
            - -Dsonar.ce.javaAdditionalOpts=-javaagent:./extensions/plugins/sonarqube-community-branch-plugin-1.18.0.jar=ce
          volumeMounts:
            - mountPath: "/opt/sonarqube/data/"
              name: sonarqube-data
            - mountPath: "/opt/sonarqube/extensions/"
              name: sonarqube-extensions
          env:
            - name: SONAR_JDBC_URL
              value: jdbc:postgresql://sonarqube-postgres:5432/sonar
            - name: SONAR_JDBC_USERNAME
              value: sonar
            - name: SONAR_JDBC_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: sonarqube-postgres
                  key: password
          ports:
            - containerPort: 9000
              protocol: TCP
      volumes:
        - name: sonarqube-data
          persistentVolumeClaim:
            claimName: sonarqube-data
        - name: sonarqube-extensions
          persistentVolumeClaim:
            claimName: sonarqube-extensions
```

다음은 sonarqube가 사용할 data를 위한 pvc를 작성해보자.

`sonar-data-pvc.yaml`을 다음과 같이 작성한다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarqube-data
  namespace: sonarqube
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: sonarqube-data
  resources:
    requests:
      storage: 10Gi
```

sonarqube가 사용할 플러그인 extentions를 위한 pvc를 작성해보자.

`sonar-extentions-pvc.yaml`을 다음과 같이 작성한다.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: sonarqube-extensions
  namespace: sonarqube
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: sonarqube-extentions
  resources:
    requests:
      storage: 5Gi
```

다음은 접속을 위한 서비스 매니페스트를 작성하자.

`sonar-service.yaml`을 다음과 같이 작성한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: sonarqube
  labels:
    app: sonarqube
  name: sonarqube-service
spec:
  ports:
    - name: sonar
      port: 80
      protocol: TCP
      targetPort: 9000
  selector:
    app: sonarqube
```

마지막으로 SSL인증을 통해 특정 도메인으로 접속하기 위해 다음과 같이 `sonar-ingress.yaml`을 작성한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sonarqube-ingress
  namespace: sonarqube
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
  - host: [원하는 host]
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sonarqube-service
            port:
             number: 9000
  tls:
    - hosts:
      -  [원하는 host]
      secretName: sonarqube-cert
```

도메인이 없는 경우에는 [ArgoCD 배포하기]()를 참고해 Ingress를 설정하자.

## 플러그인 설치

컨트롤 플레인의 `/sonarqube/extentions`에 Sonarqube 플러그인 볼륨을 매핑했다. 여기에 커뮤니티 브랜치 플러그인을 설치해야한다.

`plugins`폴더를 생성하고 `jar`파일을 다운받는다.

```
sudo apt install wget
mkdir -p /sonarqube/extensions/plugin
cd /sonarqube/extensions/plugin
wget https://github.com/mc1arke/sonarqube-community-branch-plugin/releases/download/1.18.0/sonarqube-community-branch-plugin-1.18.0.jar
```

## ArgoCD APP 생성

ArgoCD에 접속해 `NEW APP` 버튼을 눌러 sonarqube에 대한 GitOps를 적용해보자.

![](https://i.imgur.com/ALNnfJl.png)

기본적으로 프로젝트 정보를 위와 같이 설정한다.

![](https://i.imgur.com/BD2MTPM.png)

APP을 생성하면 SYNC를 시작할 것이다. 레포지토리의 매니페스트 파일을 읽어 쿠버네티스 클러스터로 배포시킨다.

![](https://i.imgur.com/sLDV5zT.png)

위와 같이 잘 배포되면 성공이다.

## 접속하기

Ingress에서 설정한 URL에 들어가 기본 ID/PW인 admin/admin으로 접속할 수 있다.

![](https://i.imgur.com/QSjghcg.png)

패스워드를 변경해준다.

Administration - General - Server Base URL에서 설정한 도메인으로 URL을 변경하면 Sonarqube를 사용하기 위한 준비를 마쳤다.

## 오류 해결
### 오류해결 (vm.max_map_count)

Sonarqube는 ElasticSearch를 사용한다. ElasticSearch가 사용하는 최소 `max_map_count`가 최소 요구사항보다 작아서 오류가 발생해 파드가 작동하지 않을 수도 있다. 따라서 컨트롤플레인에서 `vm.max_map_count`를 변경해 실행하면 오류를 해결할 수 있다.

```sh
# 값 확인
sudo sysctl -a | grep vm.max_map_count

# 값 변경
sudo systcl -w vm.max_map_count=262144

# 변경된 값 확인
sudo sysctl -a | grep vm.max_map_count
```

### 오류해결 (권한 문제)

pv의 권한 문제로 작동이 안될 수 있다. `sudo chmod 755 -R [pv 디렉토리 경로]` 명령어를 통해 권한을 활성화 해주고 Argocd에서 문제되는 Pod를 삭제해 재생성되도록 해보자.

## 다음으로

ArgoCD와 Sonarqube를 쿠버네티스 클러스터에 배포했다. 이제 Source Repository에서 GitHub Action을 통해 CI/CD 과정을 진행하면 된다. 다음 포스트에서는 Sonarqube 브랜치 플러그인을 사용하기 위한 준비를 진행해보자.