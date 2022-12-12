---
layout: single
title: "Gradle Project에 checkstyle 설정하기"
categories: cicd
tag: [cicd, checkstyle]
toc: true
toc_sticky: true
author_profile: true
sidebar:
    nav: "docs"
header:
  teaser: "https://i.imgur.com/RhIrIbF.png"
# search: false
---

![](https://i.imgur.com/RhIrIbF.png){: .align-center}

Gradle 프로젝트에 `checkstyle`을 적용해 보려고 한다.

체크스타일이란 프로그래머가 자바코드를 작성할 때 일반적으로 지켜야 하는 코딩 표준을 잘 준수할 수 있도록 지원하는 유틸이다. `checkstyle`을 적용하면 내가 짠 코드가 정해진 규칙에서 무엇을 어기고 있는지 보고서를 통해 보고 받을 수 있다.

google이나 naver같은 대기업은 물론 작은 기업도 협업을 할 때, 마치 한사람이 짠 코드인 것처럼 개행 컨벤션이나, 네임컨벤션 등 린트를 맞춘다. 대기업들은 이를 코드 컨벤션 규칙들을 xml 파일로 제공하고 있으며 대표적으로 google이 지원하고 있다.

> [LINK](https://google.github.io/styleguide/javaguide.html) 에서 구글의 자바 스타일 가이드를 확인할 수 있다.
> xml은 [LINK](https://github.com/google/styleguide) 에서 ide에 맞는것을 골라 사용하면 된다.

## gradle 프로젝트에 checkstyle적용하기

gradle에서 checkstyle을 플러그인으로 지원하고 있다. ([공식문서](https://docs.gradle.org/current/userguide/checkstyle_plugin.html))

가장 먼저 `build.gradle`에 다음과 같이 설정하면 플러그인을 불러올 수 있다.

```groovy
plugins {
    id 'checkstyle'
}
```

나는 네이버의 코딩 컨벤션을 적용하고자 한다. [checkstyle 네이버 xml 설정파일](https://github.com/naver/hackday-conventions-java/tree/master/rule-config) 해당 링크에 들어가면 naver에서 공식적으로 제공하는 checkstyle의 xml 파일들을 받을 수 있다.

rules와 suppressions 모두 받아서 저장해주자.

프로젝트 루트에 checkstyle 디렉토리를 만들고 해당 파일을 붙여넣는다.

![](../../images/Pasted%20image%2020221110133518.png){: .align-center}

[checkstyle config 공식문서](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.quality.CheckstyleExtension.html) 를 확인해보면 checkstyle을 설정할 수 있는 여러 항목들이 있다. 기본적인 기본들을 이용할 것이므로 다음과 같이 설정해준다. (임의의 설정이 더 필요하면 해당 문서를 확인하고 추가해준다.)

```groovy

compileJava.options.encoding = 'UTF-8'
compileTestJava.options.encoding = 'UTF-8'

tasks.withType(Checkstyle) {
    reports {
        xml.required = true
        html.required = true
    }
}

checkstyle {
    configFile = file("checkstyle/naver-checkstyle-rules.xml")
    configProperties = ["suppressionFile": "checkstyle/naver-checkstyle-suppressions.xml"]
}


```

`build.gradle`에 위와 같이 추가해준다.

위에서 설정한 부분은 다음과 같다.

- 자바를 컴파일할 때 `UTF-8`로 해주어 테스트에서 한글로 명명된 테스트에서 체크스타일 워닝이 발생하지 않도록 해준다.
- xml 리포트와 html 리포트 추출을 활성화 한다.
- `rule.xml` file과  `suppressions.xml` 파일을 설정한다.

이제 build를 해보면 `checkstyle` 관련 Task들이 실행되고 리포트 결과를 확인할 수 있다!

## Querydsl 사용 시 문제 해결하기

만약 `Querydsl`를 사용하고 있다면 gradle clean 후 build를 해보면 CompileQuerydsl에서 오류가 날 것이다. 또한 오류를 해결 하고 나면 checkstyle 리포트에 generate된 `Qdomain`들이 포함되어 있을 것이다. 이를 해결해 보자.

### `CompileQueryDsl` 오류 해결하기

pmd도 코드스타일 관련 도구인데 비슷한 문제가 생긴듯 하여 같은 해결법을 적용해 주었다. (관련 링크 :  [LINK](https://stackoverflow.com/questions/48988083/gradle-compile-querydsljava-failed))

```groovy
checkstyle {
    configFile = file("checkstyle/naver-checkstyle-rules.xml")
    configProperties = ["suppressionFile": "checkstyle/naver-checkstyle-suppressions.xml"]
    sourceSets = [sourceSets.main] // CompileQuerydsl 오류 해결
}
```

위 처럼 기존 설정에 `sourceSets` 관련 설정을 한 줄 추가해 주어 `CompileQuerydsl` 태스크 에서 생기는 문제를 해결할 수 있다.

### Querydsl 제외시키기

그리고 gradle에서 `checkstyleMain`을 실행시켜보면 현재 코드에 문제가 있다고 보고된다.

![](../../images/Pasted%20image%2020221110133654.png){: .align-center}

`build - reports - main.html` 파일을 열어보면

![](../../images/Pasted%20image%2020221110133814.png){: .align-center}

위의 결과에서 볼 수 있듯이, querydsl관련 코드도 codestyle에 잡힌 것을 확인할 수 있다.


`Querydsl` 관련 클래스를 제거하려면 checkstyle 설정에서 generated된 코드를 검사하지 않게 현재 소스만 검사하도록 설정하면 된다.

```groovy
checkstyleMain.source = fileTree('src/main/java')
```

위와 같이 설정하면 내가 작성한 소스파일만 해당되게 할 수 있다.

![](../../images/Pasted%20image%2020221110150823.png){: .align-center}

그러면 Queydsl 관련 파일이 체크되지 않아 에러가 줄어든 것을 확인할 수 있다.

최종 설정은 다음과 같다

```groovy
compileJava.options.encoding = 'UTF-8'
compileTestJava.options.encoding = 'UTF-8'

tasks.withType(Checkstyle) {
    reports {
        xml.required = true
        html.required = true
    }
}

checkstyle {
    configFile = file("checkstyle/naver-checkstyle-rules.xml")
    configProperties = ["suppressionFile": "checkstyle/naver-checkstyle-suppressions.xml"]
    sourceSets = [sourceSets.main]
}

checkstyleMain.source = fileTree('src/main/java')

```


## Intellij에 checkstyle적용하기

나는 checkstyle 플러그인은 설치하지 않고 fomatter 설정만 해줄 것이다.

[checkstyle 네이버 inteelij xml 설정파일](https://github.com/naver/hackday-conventions-java/tree/master/rule-config) 해당 링크에 들어가면 naver에서 공식적으로 제공하는 intellij의 fommater xml 파일들을 받을 수 있다.

해당 파일을 원하는 곳에 저장시키고 Intellij에서 다음의 설정에 들어간다.

![](../../images/Pasted%20image%2020221110154409.png){: .align-center}

받은 파일을 적용시켜 준다.

해당 포매터를 즉시 모든 파일에 적용시켜 보자

프로젝트 루트 폴더를 선택하고 `⌥ ⌘ L`을 누르면 다음과 같은 창이 뜬다. 모든 체크박스를 체크하고 `Run`을 눌러주자

gradle check를 돌려보면


![](../../images/Pasted%20image%2020221110154939.png){: .align-center}

아까와 다르게 25개의 에러가 있는 것이 확인된다. 나의 경우에는 변수의 이름에 `requestDTO`와 같은 카멜케이스를 무시한 이름이 문제가 되었다. 이를 바꿔주자

`DTO`를 `Dto`로 바꾸며 `jacocoTestCoverageVerification`에서 exclude할 클래스도 `DTO`에서 `Dto`로 바꿔주었다.

![](../../images/Pasted%20image%2020221110160237.png){: .align-center}

몇번의 수정을 거친 뒤 에러를 0으로 만들 수 있었다.
