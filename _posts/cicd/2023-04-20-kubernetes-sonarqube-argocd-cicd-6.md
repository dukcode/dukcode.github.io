---
layout: single
title: "[티어리스트] (6) Kubernets, ArgoCD, GitHub Action, Sonarqube를 통한 CICD 구축기"
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
# GitHub Action을 통해 CICD 해보기

이제 CI/CD를 위한 모든 세팅이 끝났다. GitHub Action 스크립트를 작성해 실제로 CI/CD 파이프라인을 작동시키면 된다.

## Dockerfile 작성

가장 먼저 우리의 서비스 애플리케이션의 컨테이너 이미지를 만들기 위한 Dockerfile을 작성해보자.

### plain-jar 생성하지 않기

티어리스트 프로젝트는 gradle을 빌드 시스템으로 이용하고 있다. gradle을 빌드 시 jar와 plain-jar을 함께 생성한다. Dockerfile이 `/build/libs`에서 wildcard인 `*`를 통해 `jar`파일을 가져올 수 있게 plain-jar 생성을 막아보자.

`build.gradle`에 다음과 같이 설정하면 된다.

```groovy
jar {  
    enabled = false  
}
```

위와 같이 추가하고 다시 빌드해보면 plain-jar가 생성되지 않음을 확인할 수 있다.

### Dockerfile 작성

프로젝트 루트에 `Dockerfile`을 다음과 같이 작성한다.

```docker
FROM eclipse-temurin:17.0.10_7-jre-focal as build
WORKDIR /app
COPY  build/libs/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]

```

`eclipse-temurin:17.0.10_7-jre-focal`이미지를 기반으로 `jar`파일을 가져오고 컨테이너 실행 시 해당 `jar`파일을 실행한다. 포트는 8080을 열어두었다.

## GitHub Action 스크립트 작성

### PR 요청 시 파이프라인

![](https://i.imgur.com/1ygYXoM.png)

PR 요청 시 Sonarqube로 부터 피드백을 받을 수 있게 위와 같이 구성할 것이다.

1. 개발자가 source repository에 PR을 작성한다.
2. GitHub Action 트리거가 작동한다.
3. Build와 Test를 마치고 Sonarqube에 정적 분석을 요청한다.
4. Sonarqube의 플러그인은 정적 분석 결과를 PR Comment로 추가한다.
5. 개발자는 피드백에 따라 코드를 리팩토링 하고 푸시한다.
6. 다시 2번으로 돌아간다.

위와 같이 피드백을 받고 피드백을 수용하는 과정을 거칠 수 있도록 구성할 것이다.

프로젝트 루트에 `.github/workflows` 폴더를 만든다.

`pr-sonarbot.yaml`파일에 다음과 같이 작성한다.

```yaml
name: pr-sonarbot  
  
on:  
  pull_request:  
    types: [opened, synchronize, reopened]  
jobs:  
  check-code-smell:  
  
    runs-on: ubuntu-latest  
  
    steps:  
      - name: Checkout  
        uses: actions/checkout@v4  
  
      - name: Setup Java17  
        uses: actions/setup-java@v4  
        with:  
          distribution: temurin  
          java-version: 17  
  
      - name: Setup Gradle  
        uses: gradle/gradle-build-action@v3  
  
      - name: Cache SonarQube packages  
        uses: actions/cache@v4  
        with:  
          path: ~/.sonar/cache  
          key: ${{ runner.os }}-sonar  
          restore-keys: ${{ runner.os }}-sonar  
  
      - name: Cache Gradle packages  
        uses: actions/cache@v4  
        with:  
          path: ~/.gradle/caches  
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle') }}  
          restore-keys: ${{ runner.os }}-gradle  
  
      - name: Analyze with Sonarqube  
        env:  
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}  
        run: ./gradlew build sonar --info  
  
      - name: SonarQube Quality Gate check  
        id: sonarqube-quality-gate-check  
        uses: sonarsource/sonarqube-quality-gate-action@master  
        timeout-minutes: 5  
        with:  
          scanMetadataReportFile: build/sonar/report-task.txt  
        env:  
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }} #OPTIONAL
```

빌드 후 소나큐브에 정적 분석을 요청하는 간단한 워크플로우이다. 이제 해당 Repository에 PR을 날려보면 아래와 같이 정적 분석 결과를 코멘트로 확인할 수 있다.

![](https://i.imgur.com/qGVjQTC.png)

### Merge 시 파이프라인

![](https://i.imgur.com/rD4aT7s.png)

PR의 개발 및 리팩토링이 끝나고 머지가 되면 위와 같은 CI/CD 워크플로우가 작동한다.

1. develop 브랜치에 변경사항이 생기면 GitHub Action 트리거가 작동한다.
2. 서비스 애플리케이션을 빌드 테스트한 후 Sonarqube에 정적 분석을 맡긴다.
3. Quality Gate를 통과하지 못하면 워크플로우는 종료되고 Discord에 알림을 보낸다.
3. Quality Gate 통과 시 Dockerfile을 기반으로 컨테이너 이미지를 생성해 Docker Registry에 등록한다. 
4. GitOps 레포지토리에 Docker Image의 태그를 업데이트한다.
5. ArgoCD는 GitOps Repository에 변경을 감지하고 매니페스트 파일들을 쿠버네티스 클러스터에 배포한다.

프로젝트 루트에서 `.github/workflows/pr-sonarbot.yaml` 파일에 다음과 같이 작성한다.

```yaml
name: ci  
  
on:  
  push:  
    branches: [ develop ]  
jobs:  
  ci:  
  
    runs-on: ubuntu-latest  
  
    steps:  
      - name: Checkout  
        uses: actions/checkout@v4  
  
      - name: Setup Java17  
        uses: actions/setup-java@v4  
        with:  
          distribution: temurin  
          java-version: 17  
  
      - name: Setup Gradle  
        uses: gradle/gradle-build-action@v3  
  
      - name: Build With Gradle  
        run: ./gradlew clean build  
  
      - name: Test With Gradle  
        run: ./gradlew test  
  
      - name: Cache SonarQube packages  
        uses: actions/cache@v4  
        with:  
          path: ~/.sonar/cache  
          key: ${{ runner.os }}-sonar  
          restore-keys: ${{ runner.os }}-sonar  
  
      - name: Cache Gradle packages  
        uses: actions/cache@v4  
        with:  
          path: ~/.gradle/caches  
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle') }}  
          restore-keys: ${{ runner.os }}-gradle  
  
      - name: Analyze with Sonarqube  
        env:  
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}  
        run: ./gradlew build sonar --info  
  
      - name: SonarQube Quality Gate check  
        id: sonarqube-quality-gate-check  
        uses: sonarsource/sonarqube-quality-gate-action@master  
        timeout-minutes: 5  
        with:  
          scanMetadataReportFile: build/sonar/report-task.txt  
        env:  
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
  
      - name: Docker meta  
        id: meta  
        uses: docker/metadata-action@v5  
        with:  
          images: duk9741/tierlist-api  
          tags: |  
            ${{github.run_number}}  
  
      - name: Set up QEMU  
        uses: docker/setup-qemu-action@v3  
  
      - name: Set up Docker Buildx  
        uses: docker/setup-buildx-action@v3  
  
      - name: Login to DockerHub  
        if: github.event_name != 'pull_request'  
        uses: docker/login-action@v3  
        with:  
          username: ${{ secrets.DOCKERHUB_USERNAME }}  
          password: ${{ secrets.DOCKERHUB_TOKEN }}  
  
      - name: Build and push  
        uses: docker/build-push-action@v5  
        with:  
          context: .  
          push: ${{ github.event_name != 'pull_request' }}  
          tags: ${{ steps.meta.outputs.tags }}  
          labels: ${{ steps.meta.outputs.labels }}  
          platforms: |  
            linux/amd64            linux/arm64            linux/arm/v7  
      - name: Checkout Git Ops Repo  
        uses: actions/checkout@v4  
        with:  
          repository: tierlist-projects/tierlist-git-ops  
          token: ${{ secrets.GIT_PASSWORD }}  
  
      - name: Update Git Ops Repo  
        run: |  
          pwd          ls          git config --global user.email "actions@github.com"          git config --global user.name "GitHub Actions"          sed -i "s+duk9741/tierlist-api.*+${{ steps.meta.outputs.tags }}+g" app/Deployment.yaml  
          git add .          git commit -m "action: update image tag to ${{ steps.meta.outputs.tags }}"  
          git push origin main  
      - name: Send Webhook to Discord  
        uses: sarisia/actions-status-discord@v1  
        if: always()  
        with:  
          webhook: ${{ secrets.DISCORD_CICD_WEBHOOK }}
```

눈여겨 볼 점은 다음과 같다.

- 속도를 위해 캐싱을 사용한다.
- M2 Mac에서 동작하기 위해서 Docker Buildx를 통해  ARM 등의 여러 플랫폼으로 빌드한다.
- 도커 이미지의 태그는 GitHub Action `run_number`로 한다.
- 도커 레지스트리에 컨테이너 이미지를 등록할 때 `DOCKERHUB_USERNAME` 및 `DOCKERHUB_PASSWORD`를 사용한다. 이를 레포지토리 Secret에 등록해야 한다.
- 이미지 PUSH가 완료되면 GitOps 레포지토리에 접속해 이미지의 태그를 변경한다.
- 마지막으로 결과를 Discord로 보낸다. 세부 설정은 [sarisia/actions-status-discord](https://github.com/sarisia/actions-status-discord)를 확인해 보자. 레포지토리 Secret에 Discord Webhook을 생성해 넣어줘야 한다.

위와 같이 설정하면 GitHub action 설정을 마친 것이다.

### GitOps 레포지토리 매니페스트 등록

Spring Boot 앱을 위한 매니페스트를 GitOps Repository를 등록해보자.

레포지토리 루트에 `app`디렉토리를 만들고 `Deployment.yaml`파일을 다음과 같이 작성한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tierlist-api-deployment
  namespace: tierlist-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tierlist-api
  template:
    metadata:
      labels:
        app: tierlist-api
      annotations:
    spec:
      serviceAccountName: internal-app
      containers:
        - name: tierlist-api
          image: duk9741/tierlist-api:27
          env:
          - name: REDIS_HOST
            valueFrom:
              configMapKeyRef:
                name: tierlist-configmap
                key: redis_host
          - name: REDIS_PORT
            valueFrom:
              configMapKeyRef:
                name: tierlist-configmap
                key: redis_port
          - name: GMAIL_USERNAME
            valueFrom:
              secretKeyRef:
                name: tierlist-gmail-secret
                key: username
          - name: GMAIL_PASSWORD
            valueFrom:
              secretKeyRef:
                name: tierlist-gmail-secret
                key: password
          - name: JWT_SECRET
            valueFrom:
              secretKeyRef:
                name: tierlist-jwt-secret
                key: secret
          ports:
            - containerPort: 8080
```

눈여겨 볼점은 다음과 같다.

- repllcas는 1로 설정하고 있다.
- 환경변수를 위해 ConfigMap을 작성해 환경 변수를 설정하고 있다.
- 민감한 정보는 Secret을 이용해 설정하고 있다.

세부적인 배포 전략은 수정할 수 있다. replica를 늘린다던가 env로 profile정보를 넣을 수도 있다. 입맛에 맞게 변경해서 사용해보자.

**Secret은 민감한 정보이므로 GitOps 레포지토리에서 관리하면 안된다. `Vault`를 이용하던지 기본적인 Secret을 쿠버네티스에서 생성해 이용하자.**

`app` 디렉토리에 `ConfigMap.yaml`을 설정해 ArgoCD가 앱과 함께 배포할 수 있게 설정할 수 있다. 예시는 다음과 같다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: tierlist-api
  name: tierlist-configmap
data:
    redis_host: redis-service
    redis_port: '6379'
```

ConfigMap의 유의할 점은 숫자는 바로 쓰지 못하므로 따옴표로 감싸주어야 한다는 점이다.

이제 서비스와 Ingress를 설정해주자.

`Service.yaml`에 다음과 같이 작성했다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tierlist-api-service
  namespace: tierlist-api
spec:
  selector:
    app: tierlist-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: NodePort
```

`Ingress.yaml`에 다음과 같이 작성했다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tierlist-api-ingress
  namespace: tierlist-api
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
            name: tierlist-api-service
            port:
             number: 80
  tls:
    - hosts:
      - [원하는 host]
      secretName: tierlist-api-cert
```

특이점은 Spring Boot 앱이 http를 사용하기 때문에 `backend-protocol`을 `HTTP`로 설정해주었다는 것이다.

이제 마지막으로 해당 매니페스트들을 바라볼 수 있게 ArgoCD APP을 생성해보자. [Sonarqube ArgoCD APP 생성]()을 참고해 APP을 생성할 수 있다.

이제 develop에 Merge 또는 push를 하면 다음과 같이 GitHub Action 워크플로우가 작동한다.

![](https://i.imgur.com/ttTvAHy.png)

또한 ArgoCD를 확인해보면 아래와 같이 매니페스트들이 잘 배포되었음을 확인할 수 있다.

![](https://i.imgur.com/YAV5Nz9.png)

## 회고

이로써 Kubernets, ArgoCD, GitHub Action, Sonarqube를 통한 CICD 구축기를 마친다. 엄청난 오류와 씨름하고 얻어낸 결과이다. 하지만 다 해놓고 나니  안정적으로 서비스가 운영될 수 있음에 뿌듯함을 느꼇다.

추후에 도입하고 싶은 것은 다음과 같다.

- docker jib 도입
- vault 도입

위는 프로젝트가 어느정도 안정화가 된 후 천천히 도입해볼 생각이다. 이로써 길고 길었던 티어리스트 프로젝트 CICD 구축기를 마친다.