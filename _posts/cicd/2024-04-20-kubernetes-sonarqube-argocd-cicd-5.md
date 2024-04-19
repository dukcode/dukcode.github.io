---
layout: single
title: "[티어리스트] (5) Kubernets, ArgoCD, GitHub Action, Sonarqube를 통한 CICD 구축기"
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
# Sonarqube 설정하기

Sonarqube가 GitHub Repository에서 작동할 수있게 여러 설정들을 해주어야 한다.

## GitHub APP 생성

이전 포스트에서 Sonarqube의 커뮤니티 브랜치 플러그인을 설치했다. 해당 플러그인이 작동하기 위해서는 GitHub APP을 만들어야 한다. 차근차근 하나씩 진행해보자.

[공식 문서](https://docs.sonarsource.com/sonarqube/latest/devops-platform-integration/github-integration/setting-up-integration/)의 내용을 참고해 진행하면 된다.

GitHub Organization이나 프로필 페이지에 들어가 Develops settings에 들어간다.

![](https://i.imgur.com/pkt5unZ.png)

아래와 같은 화면을 확인할 수 있다.

![](https://i.imgur.com/oACVThl.png)

New GitHub App을 눌러 앱을 추가한다.

![](https://i.imgur.com/E8ivQ7Y.png)

Homepage URL과 Callback URL에 Sonarqube URL을 설정한다.

![](https://i.imgur.com/iM9NuKe.png)

웹훅은 보안을 위해 끄라고 공식문서에 나와있다.

이제 Permissions를 설정해보자. 중요한 부분이다.

- Repository permissions
  -  Checks : Read & Write
  - GitHub.com : Read-only
  - Pull Requests : Read & write
  - Contents: Read-only (private repo인 경우 설정)
- Organization permissions
  - Members : Read-only
  - Projects : Read-only
- Account permissions
  - Email address : Read-only (GitHub Authentication을 사용하는 경우 설정)

**위와 같이 설정했다면 [공식 문서](https://docs.github.com/en/apps/using-github-apps/installing-your-own-github-app)를 보고 GitHub APP을 설치해주자. Organization이나 프로필의 Developer settings에서 설치할 수 있다.**

## Sonarqube 프로젝트 생성

Sonarqube에서 프로젝트를 생성해야 한다. 10.3.0 버전 이전 버전에서는 GitHub 앱과 연동하기 위해 설정을 해주어야 했지만 이 버전에서는 프로젝트를 생성할 때 자동으로 설정해준다.

![](https://i.imgur.com/6t9ORzb.png)

Import from GitHub를 선택한다.

![](https://i.imgur.com/vUBz5zw.png)

위와 같이 프로젝트 이름을 설정하고 GitHub API URL을 적는다. GitHub 엔터프라이즈 버전이 아니라면 `https://api.github.com/`을 입력한다.

![](https://i.imgur.com/jQc85sz.png)

GitHub App ID와 Client ID, Client Secret, Private Key를 입력해준다. 만든 앱의 설정에서 위와 같이 확인할 수 있다. WebHook Secret은 설정하지 않는다.

![](https://i.imgur.com/Yj3KCsi.png)

위와 같이 분석을 원하는 레포지토리를 설정한 후 Import 버튼을 누른다.

![](https://i.imgur.com/TRG2eO0.png)

global settings를 체크하고 프로젝트를 생성한다.

![](https://i.imgur.com/S5qU6IM.png)

이제 프로젝트 설정을 해주어야 한다. With Github Actions를 클릭하고 가이드에 따라 진행한다.

- `SONAR_TOKEN` 설정
- `SONAR_HOST_URL` 설정

위 설정을 마치고 빌드 툴을 선택한다. 티어리스트 프로젝트는 Gradle을 사용하므로 Gradle을 선택했다.

가이드에 따라 `build.gradle`을 설정한다. GitHub action yaml 파일 작성은 미루고 다음 포스트에서 진행한다.

## 플러그인 작동 확인

![](https://i.imgur.com/FEGSU1j.png)

Sonarqube 에서 Administration - DevOps Platform Integretions에서 Check Configuration을 눌러 위와 같이 정상적으로 작동하는지 확인한다.

## Quality Gates 설정

Sonarqube 상단의 Quality Gates를 눌러 새로운 퀄리티 게이트를 생성하자.

![](https://i.imgur.com/n18ptER.png)

Create 버튼을 통해 퀄리티 게이트를 만들 수 있다. Condition들을 입맛에 맞게 설정하고 프로젝트에서 해당 퀄리티 게이트를 연결해주자.

![](https://i.imgur.com/X3xx230.png)

프로젝트의 Project Settings에서 Quality Gate를 선택 후 만들어준 퀄리티 게이트에 연결해준다.

![](https://i.imgur.com/ASHQzwb.png)

이제 GitHub Action 수행 시 해당 퀄리티게이트를 통과하는지 확인하고 결과를 내보내 줄 것이다.

## 다음으로

이제 Sonarqube에서 정적 분석을 진행할 수 있게 되었다. 다음 포스트에서는 GitHub action을 통해 CICD 워크 플로우를 구성해보자.