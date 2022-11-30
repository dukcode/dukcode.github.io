---
layout: single
title: "Gradle Project에 JaCoCo 설정하기"
categories: cicd
tag: [cicd, jacoco]
toc: true
toc_sticky: true
author_profile: true
sidebar:
    nav: "docs"
# search: false
---

## JaCoCo란?

`JaCoCo`란 Java Code Coverage의 약자로 Java 코드의 커버리지를 체크하는 라이브러리이다. 테스트코드가 해당 코드를 얼마나 커버하는지를 확인해준다. 또한 사용자의 설정으로 원하는 커버리지 기준을 설정하고 html이나 xml, csv의 형태로 코드 커버리지를 확인할 수 있는 툴이다.

## Gradle 프로젝트에 설정하는법

JaCoCo는 Java프로젝트를 하면서 사용하는 사람들이 많아서 인지 Gradle이 Plugin Reference로 제공하고 있다. [LINK](https://docs.gradle.org/current/userguide/plugin_reference.html) 를 확인해 보면 `JaCoCo` 플러그인이 존재하는 것을 볼 수 있다.

공식 문서를 확인하며 `JaCoCo`를 Gradle 프로젝트에 적용해보자

```groovy
plugins {
    id 'jacoco'
}
```

위와 같이 플러그인을 추가해주면 `jacocoTestReport`와 `jacocoTestCoverageVerification`이라는 새 태스크가 생긴다. 

![](../../images/Pasted%20image%2020221108191627.png){: .align-center}

`jacocoTestCoverageVerification`과 `jacocoTestReport` 두개의 태스크가 생기는 것을 볼 수 있다.

### 의존성 설정

- `jacocoTestReport` : 커버리지 결과를 사람이 읽기 좋은 형태의 리포트로 저장한다.
- `jacocoTestCoverageVerification` : 해당 태스크는 **최소 코드 커버리지 수준**을 설정할 수 있고, 이를 통과하지 못하면 태스크가 실패한다.

> ![](../../images/Pasted%20image%2020221109001134.png){: .align-center}

> 보고서를 생성하기 전에 테스트를 실행해야 하지만 jacocoTestReport 작업은 테스트 작업에 종속되지 않습니다.

공식문서에 위처럼 나와있다. 즉, 테스트 하고난 후에 `jacocoTestReport`를 실행하도록 설정해주어야 한다는 의미이다. 또한 `jacocoTestCoverageVerification`도 마찬가지이다.

`jacocoTestReport` 태스크는 **테스트에 대한 리포트를 생성**하는 태스크이므로 `Test` 태크스가 선행되어야 한다. 또한 `jacocoTestCoverageVerification`은 **설정한 커버리지를 만족하는지 리포트를 보고 판단**해야 하므로 순서는 `test` - `jacocoTestReport` - `jacocoTestCoverageVerification` 이 되어야 한다.

따라서 `test`가 gradle에서 실행되었을 때 `test` 후 `jacocoTestReport`와 `jacocoTestCoverageVerification`이 순서대로 실행되게 설정할 수 있다.

```groovy
tasks.named('test') {
    finalizedBy 'jacocoTestReport' // test를 실행하면 마지막에 jacocoTestReport 태스트가 실행되도록 설정
}

jacocoTestReport {
    dependsOn 'test' // 해당 태스크를 실행하면 test가 먼저 수행되도록 설정
	finalizedBy 'jacocoTestCoverageVerification' // 해당 태스크 후에 실행
}
```

위와 같이 `build.gradle`에 설정해준다. 그러면 

```sh
$ ./gradlew test
$ ./gradlew jacocoTestReport
```

콘솔에서 위와 같이 실행했을 때 (두 줄 중 아무거나) `test` - `jacocoTestReport` - `jacocoTestCoverageVerification` 순서로 태스크가 실행된다.

> 순수 test만 진행하고 싶은 경우 새로운 태스크를 만들어 의존성을 설정해주면 된다. 그러면 test 태스크를 실행했을 때 test만 진행된다.

### JaCoCo 설정

이제 추가된 두개의 태스크에 대한 설정을 해주기 전에, [JaCoCo 플러그인 자체 설정](https://docs.gradle.org/current/dsl/org.gradle.testing.jacoco.plugins.JacocoPluginExtension.html) 을 해주어야 한다.

`JacocoPluginExtention`을 통해 `JaCoCo` 플러그인에 대한 설정을 해줄 수 있다. 종류는 다음 과 같다.

- `reportsDirectory` : 리포트 파일이 생성될 디렉토리를 설정한다.
- `toolVersion` : JaCoCo의 버전을 설정한다. (현재 최신 버전은 0.8.8 버전이다. ([최신버전 확인](https://mvnrepository.com/artifact/org.jacoco/org.jacoco.agent)))

> `reportsDirctory`를 설정하지 않았을 때 디폴트 값은 `$buildDir/reports/jacoco`이다.

```groovy
jacoco {
	toolVersion = '0.8.8'
	reportsDir = $buildDir/reports/jacoco
}
```

위와 같이 설정해 줄 수 있지만 필자는 디폴트 설정을 따르겠다.

### `jacocoTestReport` 설정하기

`jacocoTestReport`는 다양한 형식의 리포트를 생성할 수 있다. `xml`, `csv`, `html` 세가지 인데, 나는 나중에 `SonarQube`에 적용하기위한 `xml`과 편하게 볼 수 있는 `html`을 설정하겠다.

설정 항목은 다양하게 존재하는데 [공식 문서](https://docs.gradle.org/current/dsl/org.gradle.testing.jacoco.tasks.JacocoReport.html) 로 대체하겠다.

```groovy
jacocoTestReport {
    dependsOn test
    reports {
        xml.enabled true
        csv.enabled false
        html.enabled true
    }
    finalizedBy 'jacocoTestCoverageVerification'
}
```


xml과 html만 enable처리를 해놓았다.

### `jacocoTestCoverageVerifiaction` 설정하기

`jacocoTestCoverageVerification`의 설정으로 **최소 코드 커버리지 수준**을 설정할 수 있다.

`violationRules`메서드를 통해 커버리지 기준을 설정하는 rule들을 정의할 수 있다.

```groovy
jacocoTestCoverageVerification {
    violationRules {
        rule {
            enabled = true

            // 룰을 체크할 단위는 클래스 단위
            element = 'CLASS'

            // 브랜치 커버리지를 최소한 80% 만족시켜야 합니다.
            limit {
                counter = 'BRANCH'
                value = 'COVEREDRATIO'
                minimum = 0.80
            }

            // 라인 커버리지를 최소한 80% 만족시켜야 합니다.
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.80
            }
        }
    }
}

```

또한 rule의 설정 항목으로는 다음이 존재한다.

- enable : 해당 룰을 설정/해제함
- element : 커버리지를 체크할 기준을 설정, 다음의 6개의 기준이 존재한다. 디폴트 설정은 BUNDLE이다.
	- BUNDLE : 패키지 번들(프로젝트 모든 파일을 합친 것)
	-  CLASS : 클래스
	-  GROUP : 논리적 번들 그룹
	-  METHOD : 메서드
	-  PACKAGE : 패키지
	-  SOURCEFILE : 소스 파일
- limit : 커버리지를 제한한다.
- includes : 적용 대상을 패키지 수준으로 정의할 수 있다. 디폴트는 전체 패키지
- excludes : 제외할 대상을 **클래스** 수준으로 정의할 수 있다.

여기서 가장 중요한 `limit`을 알아보자

#### limit

##### counter

커버리지 측정의 최소 단위이다. BRANCH, CLASS, COMPLEXITY, INSTRUCTION, METHOD, LINE이 있으며 의미는 [공식 문서](https://www.eclemma.org/jacoco/trunk/doc/counters.html) 에서 자세히 확인할 수 있다.

##### value
`value`는  **측정한 커버리지를 어떠한 방식으로 보여줄 것**인지를 말한다. 총 5개의 방식이 존재한다.

-   COVEREDCOUNT : 커버된 개수
-   COVEREDRATIO : 커버된 비율, 0부터 1사이의 숫자로 1이 100%이다.
-   MISSEDCOUNT : 커버되지 않은 개수
-   MISSEDRATIO : 커버되지 않은 비율, 0부터 1사이의 숫자로 1이 100%이다.
-   TOTALCOUNT : 전체 개수

값을 지정하지 않은 경우 Default 값은 **COVEREDRATIO**이다.

##### minimum

counter 값을 value에 맞게 표현했을 때의 최소값을 말한다. 이 값을 통해 태스크의 성공/실패 여부가 결정된다.

> 해당 값은 `BigDemical` 값이므로 유효숫자만큼 값이 설정된다. 따라서 80%의 코드커버리지를 만족하고, value 값에서 1%까지 보고싶을 때 0.8이 아닌 0.80으로 설정해주어야 87%가 87%로 보인다. 0.8로 설정하면 87%도 80%로 보인다.

### test Task 설정

`JaCoCo` 플러그인은 Test 유형의 모든 작업에 [JacocoTaskExtension](https://docs.gradle.org/current/dsl/org.gradle.testing.jacoco.plugins.JacocoTaskExtension.html)을 추가할 수 있다.

공식문서에 따르면 디폴트 설정을 다음과 같다.

```groovy
test {
    jacoco {
        enabled = true
        destinationFile = layout.buildDirectory.file("jacoco/${name}.exec").get().asFile
        includes = []
        excludes = []
        excludeClassLoaders = []
        includeNoLocationClasses = false
        sessionId = "<auto-generated value>"
        dumpOnExit = true
        classDumpDir = null
        output = JacocoTaskExtension.Output.FILE
        address = "localhost"
        port = 6300
        jmx = false
    }
}
```

일단은 따로 건들것은 없는것 같으므로 넘어가겠다.

## 실행해보기

설정이 완료되었으므로 gradle을 이용해 jacoco report를 받아보자

```sh
$ ./gradlew clean
$ ./gradlew test
```

위를 콘솔에 입력해보면 

![](../../images/Pasted%20image%2020221109010510.png){: .align-center}

실패했다.. 설정한 커버리지를 넘기지 못했기 때문이다. html을 확인해 보면

![](../../images/Pasted%20image%2020221109010543.png){: .align-center}

기본적으로 coverage가 매우 낮기 때문이다. 또한 커버리지를 측정하고 싶지 않은 패키지도 보인다. 특히 querydsl을 사용한다면 Qdomain은 빼는게 좋겠다.

![](../../images/Pasted%20image%2020221109010936.png){: .align-center}

이로써 설정한대로 JaCoCo가 잘 작동함을 확인할 수 있다.

## 커버리지에서 제외할 클래스 설정

#### 일반 클래스

`JaCoCo`로 테스트 코드 수행 시 테스트가 필요없는 부분들도 커버리지에 잡하기 때문에 제외할 필요성이 있다.

제외할 클래스들의 일반적인 예시로는 다음과 같다.

- 각종 Config 클래스
- `@SpringBootApplication`가 붙은 클래스
- DTO 클래스
- Exception 및 관련 클래스
- Constant 클래스
- Interceptor 및 관련 클래스
- Filter 및 관련 클래스
- Querydsl에 사용되는 Qdomain 클래스

나의 경우에는 아래를 추가했다.

- logback관련 Appender 클래스
- 특정 컨트롤러 클래스


Qdomain 클래스들은 Q뒤에 원래 클래스의 이름이 오기 때문에 QA.. 부터 QZ.. 까지의 모든 클래스를 추가시키면 `Queen`과 같은 클래스를 만들었다고 해도 테스트리포트에 정상적으로 포함된다.

```groovy
jacocoTestReport {
    dependsOn 'test'
    reports {
        xml.enabled true
        csv.enabled false
        html.enabled true
    }

	def Qdomains = []  
	for (qPattern in '**/QA'..'**/QZ') {  
	    Qdomains.add(qPattern + '*')  
	}

    afterEvaluate {
        classDirectories.setFrom(
                files(classDirectories.files.collect {
                    fileTree(dir: it, excludes: [
			// 개인이나 팀 약속에 맞춰 exlude 패턴 추가
			"**/config/**/*",      // config 클래스
			"**/*Application*",  // @SpringBootApplication이 붙은 클래스
			"**/*DTO*",          // DTO 클래스
			"**/*Exception*",   // Exception 및 관련 클래스
			"**/constant/*",    // constant 클래스
			"**/*Interceptor*", // interceptor 클래스
			"**/filter/*",      // filter 및 관련 클래스
			"**/util/*",

			"**/*Appender*",    // logback 관련 Appender 클래스
			"**/WebRestController.class"  // 특정 컨트롤러 클래스
                    ] + Qdomains)
                })
        )
    }
}

```

따라서 위와 같이 설정하면

![](../../images/Pasted%20image%2020221109015552.png){: .align-center}

이전에는 보였던 Qdomain들이 보이지 않고 제외한 클래스들이 모두 제외된 것을 확인할 수 있다.

또한 `jacocoTestCoverageVerification`에서도 excludes를 설정해주어야한다. 이 태스크의 excludes는 **클래스 수준**으로 표기해야 하므로 다음과 같이 설정한다.

```groovy
jacocoTestCoverageVerification {
    def Qdomains = []
    for (qPattern in '*.QA'..'*.QZ') {
        Qdomains.add(qPattern + '*')
    }

    violationRules {
        rule {
            enabled = true

            element = 'CLASS'

            limit {
                counter = 'BRANCH'
                value = 'COVEREDRATIO'
                minimum = 0.80
            }

            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.80
            }

            excludes = [
                    "**.config.**.*",      // config 클래스
                    "**.*Application*",  // @SpringBootApplication이 붙은 클래스
                    "**.*DTO*",          // DTO 클래스
                    "**.*Exception*",   // Exception 및 관련 클래스
                    "**.constant.*",    // constant 클래스
                    "**.*Interceptor*", // interceptor 클래스
                    "**.filter.*",      // filter 및 관련 클래스
                    "**.util.*",

                    "**.*Appender*",    // logback 관련 Appender 클래스
                    "**.WebRestController"  // 특정 컨트롤러 클래스
            ] + Qdomains } }
}

```

비로소 `jacocoTestCoverageVerification`까지 통과됨을 볼 수 있다.

#### Lombok으로 생성된 코드

lombok을 사용하면 getter, builder등의 메서드를 generate하게 되는데 이를 제외할 필요가 있다.

루트 폴더 아래에 `lombok.config`파일을 만들자

![](../../images/Pasted%20image%2020221109020013.png){: .align-center}

위의 파일에

```
lombok.addLombokGerneratedAnnotation = true
```

를 추가해주면된다. 그러면 lombok이 메서드를 생성할 때 `@Generated` 어노테이션을 메서드에 붙여준다. 그러면 `JaCoCo`에서는 기본설정으로 `@Generated`가 붙은 메서드를 제거해준다.

![](../../images/Pasted%20image%2020221109020424.png){: .align-center}

그러면 위에서 보던 lombok으로 생성한 Builder가 제거되는 것을 확인할 수 있다.

---
1. [우아한형제들 기술블로그 - Gradle 프로젝트에 JaCoCo 설정하기](https://techblog.woowahan.com/2661/)
2. [Gradle JaCoCo Plugin 공식문서](https://docs.gradle.org/current/userguide/jacoco_plugin.html#jacoco_plugin)
