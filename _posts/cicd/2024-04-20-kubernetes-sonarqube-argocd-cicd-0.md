---
layout: single
title: "[티어리스트] (0) Kubernets, ArgoCD, GitHub Action, Sonarqube를 통한 CICD 구축기"
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
# Kubernetes, ArgoCD, GitHub Action, Sonarqube를 통한 CICD 구축기

![](https://i.imgur.com/rD4aT7s.png)


지금까지 여러 프로젝트를 맛보면서 CI/CD를 구축하는 것은 험난한 일이었다.지금까지의 방식은 번잡하며 구축하는 데에 여러 오류를 해결해야 했다.

Jenkins를 이용해 코드를 빌드 및 테스트하고 Jenkins 플러그인을 통해 직접 서버로 SSH 접속 후 Shell Script를 이용해 컨테이너의 Rolling Update를 진행하는 것이었다.

위의 방법은 잘되는가 싶더라도 배포 과정 중 오류가 생기면 이를 해결하는데 한세월이 걸렸다.

따라서 이번 티어리스트 프로젝트를 진행하면서 컨테이너 오케이스트레이션의 대명사인 **Kubernetes**를 적용해보려 한다.

쿠버네티스를 통한 CI/CD를 구축하면서 쿠버네티스의 장점인 확장성, 유연성, 자동화, 안정성 등을 느낄 수 있었다.

먼저 목표하고자 하는 CI/CD 파이프라인을 살펴보자.

## CI/CD 파이프라인

### PR 요청 시 파이프라인

![](https://i.imgur.com/1ygYXoM.png)


이번 프로젝트는 현재 백엔드는 혼자 진행중이어서 정적 분석을 통한 피드백을 꼭 받고 싶었다. Sonarqube를 통해 정적 분석을 받은 뒤 PR Comment로 다음과 같이 피드백을 받고 Sonarqube에 존재하는 Issue들을 해결해 나가면서 좋은 코드를 만들 수 있다.

![](https://i.imgur.com/qGVjQTC.png)

아름답지 않은가? 정적분석을 통한 피드백을 받으며 좋은 코드를 만들수 있게 되었다.

### Merge 시 파이프라인

![](https://i.imgur.com/rD4aT7s.png)

이번 CI/CD를 구축하면서 GitOps라는 개념을 처음 접했다. ArgoCD라는 툴은 Kubernetes의 선언적 접근법을 잘 실현하는 도구이다.

ArgoCD는 GitOps 레포지토리의 메니페스트 파일의 변경이 생기면 쿠버네티스 클러스터에 반영해준다. 즉, 컨테이너들을 `YAML`파일로 관리할 수 있게 된 것이다. ArgoCD를 도입하면서 배포작업을 쉽게 자동화 할 수 있게 되었다.

Kubernetes의 Deployment로 컨테이너를 배포하면 `YAML`파일에 따라서 Rolling Update나 Blue Green 방식 등을 쉽게 관리할 수 있게 되었다. 따라서 이전에 Jenkins를 통해 직접 SSH로 접속해서 Rolling Update를 진행하는 부분을 쉽게 해결할 수 있게 되었다.

새로운 이미지에 문제가 있으면 이전 상태를 유지하므로 서비스의 다운 타임 걱정을 덜어주며 컨테이너를 가용성있게 운영할 수 있게 되었다.

## 다음으로

나는 CI/CD를 구축하며 집에 있는 M2 맥미니를 통해 위를 구성하였다. UTM을 통해 Ubuntu VM을 띄우는 작업부터 시작해보자.