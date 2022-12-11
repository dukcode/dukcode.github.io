---
layout: single
title: "SDKMAN로 여러 JDK 버전 쉽게 설치하고 관리하기"
categories: java
tag: [java-version, sdkman, JDK]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: https://i.imgur.com/pU1QRQv.png"
# search: false
---

여러가지 자바 설치 방법을 알아보고, `SDKMAN`을 이용해 여러 자바 버전을 쉽게 설치하고 관리해보자.


![](https://i.imgur.com/haFdb4s.png){: .align-center}

## 근본적인 설치 방법

JDK를 설치하고 관리해주는 툴은 여러가지가 있지만, **가장 근본적인 방법은 JDK를 제작한 회사가 제공하는 JAVA 패키지의 압축을 풀어 해당 디렉토리를 $JAVA_HOME에 환경변수로 지정하고, 폴더의 bin 디렉토리를 $PATH 환경변수에 추가하는 것이다. 또한 버전 변경을 위해서는 환경변수를 원하는 JDK 버전 경로로 수정해 주면 된다.**

설치 작업을 자동으로 해줄 수 있는 툴로는 범용 패키지 관리 도구가 있다. `macOS`의 `Homebrew`, `Linux`의 `YUM`이나 `APT`, `Windows`의 `Chocolatey`가 있다. 위와 같은 툴의 장점으로는 JDK의 다양한 배포판들을 다운로드하고 환경변수를 설정하는 과정을 **콘솔 한 줄**로 할 수 있다는 것이다.

> 하지만 이 방법은 여러 **JDK 버전을 전환해 가며 사용하고자 하는 경우** 항상 **환경변수를 수정**해주어야 한다. 또한, 패키지에 따라서 JDK가 존재하는 경로가 다양해 질 수 있어 **버전관리가 힘들어질 수 있다.** 또한, 해당 패키지 매니저에서 **설치할 수 없는 JDK의 배포판**이 존재한다. (물론 추가 작업을 거치면 해당 배포판도 설치할 수 있긴하다.)

## SDKMAN

![](https://i.imgur.com/pU1QRQv.png){: .align-center}

`SDKMAN`은 JVM에 관련한 다양한 개발도구를 설치할 수 있는 범용 패키지 관리 도구이다. JDK 외에도 `Maven`, `Gradle`, `Ant` 등의 도구를 설치할 수 있다.

위에서 소개한 `Homebrew`, `YUM/APT`, `Chocolatey`와 차별화 되는 가장 큰 장점은 다양한 버전을 관리하며, 사용할 버전을 명령어 한 줄로 변경할 수 있다는 점이있다. 또한 `Adopt Open JDK`, `Amazon Corretto`, `GraalVM`, `Zulu` 등의 주요 배포판들을 거의 모두 포함하고 있는 것이 장점이다.

또한 명령행에서 디폴트로 사용할 JDK 버전은 `~/.sdkman/candidates/java/current` 에서 심볼릭 링크로 관리되고, 이 링크가 환경변수 `$PATH`와 `$JAVA_HOME`에 추가 된다. 또한 실제 JDK 디렉토리는 `~/.sdkman/candidates/java`에서 관리 되기 때문에 JDK의 경로를 한 곳에서 관리할 수 있는 장점이 있다.

### direnv를 함께 사용(선택)

**아쉽게도 SDKMAN은 특정 디렉토리에서 특정 JDK버전을 사용하는 기능을 지원하지는 않는다. 해당 기능을 이용하고 싶다면 `direnv`라는 디렉토리별 환경 변수 지정 도구를 `SDKMAN`과 병행하여 사용하는 것을 추천한다.**

JDK관리 도구에는 아래와 같이 다양한 종류가 있고 각자의 기능이 조금씩 다르다. (`direnv`는 JDK관리 도구는 아니고 디렉토리별 환경 변수 지정 도구이다. 따라서 다른 환경 변수도 적용이 가능하다.)

|  이름  | JDK 설치 | 디폴트 버전 지정 기능 | 디렉토리별 버전 지정 기능 |
|:------:|:--------:|:---------------------:|:-------------------------:|
| SDKMAN |    O     |           O           |             X             |
| jabba  |    O     |           X           |             △             |
|  jEnv  |    X     |           O           |             O             |
| direnv |    X     |           X           |             O             |

`jabba`는 디폴트 버전 지정 기능이 존재하지 않는 JDK의 설치와 버전관리 전용 도구이다. 디렉토리별 버전 지정 기능을 제공하지만, 해당 디렉토리에서 `$ jabba use` 명령어를 사용해야 환경변수를 변경해준다. `jEnv`와 `direnv`는 해당 디렉토리에 접근할 때 자동적으로 환경변수를 바꾸어 준다.

`SDKMAN`과 `jEnv`를 함께 사용하는 것은 적합하지 않다. `$PATH` 환경변수의 경로 순서에 따라서 JDK 디폴트 버전 지정 결과가 의도치 않게 변경될 수 있기 때문이다.

결국 모든 기능을 충돌없이 사용하려면 `SDKMAN`과 `direnv`를 선택할 수 있다.

하지만 디렉토리 별로 JDK 버전을 지정하는 것은 특수한 경우이므로 이 포스트에서는 다루지 않을 예정이다.

### SDK 설치 및 사용 방법

SDKMAN에서 JDK를 설치하고, 디폴트 버전을 변경하는 법을 알아보자.

#### SDKMAN 설치

[LINK](https://sdkman.io/install)를 참조하여 설치를 진행한다.

```sh
$ curl -s "https://get.sdkman.io" | bash
```

위의 커맨드를 터미널에 입력하면 쉽게 설치가 가능하다.

#### 설치 가능한 JDK 버전 검색

```sh
$ sdk list java
```

위의 커맨드를 입력하면 다음과 같이 설치 가능한 다양한 버전의 JDK 리스트를 확인할 수 있다. 또한 설치된 JDK 버전의 Status를 알 수 있다. 목록은 `j`와 `k`키로 이동 가능하며 종료는 `q`이다.

```
================================================================================
Available Java Versions for macOS ARM 64bit
================================================================================
 Vendor        | Use | Version      | Dist    | Status     | Identifier
--------------------------------------------------------------------------------
 Corretto      |     | 19.0.1       | amzn    |            | 19.0.1-amzn
               |     | 17.0.5       | amzn    |            | 17.0.5-amzn
               |     | 11.0.17      | amzn    |            | 11.0.17-amzn
               |     | 8.0.352      | amzn    |            | 8.0.352-amzn
 Gluon         |     | 22.1.0.1.r17 | gln     |            | 22.1.0.1.r17-gln
               |     | 22.1.0.1.r11 | gln     |            | 22.1.0.1.r11-gln
 GraalVM       |     | 22.3.r19     | grl     |            | 22.3.r19-grl
               |     | 22.3.r17     | grl     |            | 22.3.r17-grl
               |     | 22.3.r11     | grl     |            | 22.3.r11-grl
               |     | 22.2.r17     | grl     |            | 22.2.r17-grl
               |     | 22.2.r11     | grl     |            | 22.2.r11-grl
               |     | 22.1.0.r17   | grl     |            | 22.1.0.r17-grl

...

```

위에서 원하는 버전의 JDK를 설치하면 된다.

#### 특정 JDK 버전 설치

```sh
$ sdk install java [Identifier]
$ sdk uninstall java [Identifier]
```

위의 명령어로 원하는 버전의 JDK를 설치하고 삭제할 수 있다.

#### JDK 디폴트 버전 변경

다음의 명령어로 설치된 JDK 목록을 확인하고, default로 변경, 적용된 default 버전을 확인할 수 있다.

```sh
# 설치된 JDK 목록 확인 (Status)
$ sdk list java

# 원하는 버전 default로 변경
$ sdk default java [Identifier]

# 현재 버전 확인
$ sdk current java
```