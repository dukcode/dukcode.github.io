---
layout: single
title: "Spring Cloud Config 적용해보기"
categories: cicd
tag: [spring, spring-cloud, spring-cloud-config, properties, yml]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/BjWyzWg.png"
# search: false
---

# Spring Cloud Config를 이용해 안전하게 설정파일 관리하기

![](https://i.imgur.com/BjWyzWg.png)

팀 프로젝트를 진행하다 보면 프로젝트 설정 정보에 보안에 민감한 정보가 포함된다. 이를 팀원과 함께 안전하게 공유할 수 있는 방법으로 이전에 [git-secret 사용해보기](https://dukcode.github.io/git/git-secret/)라는 방법을 소개했다.

`git-secret`은 기존 레포지토리를 통해 안전하게 민감한 정보들을 공유할 수 있었다. 하지만 새로운 브랜치로 이동할때, 커밋 전이나 pull을 받았을 때 항상 까먹지 않고 민감한 파일의 암/복호화를 수동으로 진행해 주어야 했다.

`git-secret`은 이런 번거로움 때문인지 2022년 6월 `v0.5.0` 버전 이후에 업데이트 소식이 없다. 또한 모든 팀원은 `git-secret` 사용법을 익혀야 하고 암/복호화 실수가 생기는 경우에는 번거롭게 `git`을 조작하는 일이 다반사가 된다.

![](https://i.imgur.com/V5BNsSF.png)

spring-could-config를 사용하면 위의 그림처럼 리포지토리에 있는 민감 정보를 실시간으로 전달해주는 서버를 띄우게 된다. 따라서 우리의 어플리케이션은 spring-cloud-config 서버를 통해 설정 정보를 읽어올 수 있게 된다.

이는 기존에 소개했던 방법인 `git-secret`에서 했던 번거로운 수동 암/복호화 과정을 제거할 수 있고, MSA인 경우에도 유용하게 설정 정보를 한 곳에서 관리할 수 있게 된다. 또한, 팀원들도 새로운 툴에 대해 익힐 내용이 적기 때문에 휴먼에러도 줄일 수 있다. 또한 실시간으로 설정정보를 읽어오기 때문에 설정정보를 변경시켜야 할 때도 재배포하지 않아도 된다.

이제 spring-cloud-config 서버를 띄워 애플리케이션의 설정정보를 효과적으로 관리해보자.

## 설정 정보를 위한 GitHub Repository 만들기

설정 정보를 보관하기 위한 private Repository를 생성한다. 해당 Repository에서  `yml`파일을 관리하고 Spring Cloud Config 서버가 이를 읽어 클라이언트에게 제공할 것이다.

### Private Repository 생성

spring-cloud-config 서버의 기본 구현은 git에 연결되도록 하고 있기 때문에 git 호스팅 서비스를 제공하는 GitHub를 통해 설정 파일을 읽어올 수 있다. 따라서 팀원만 접근할 수 있도록 Private Repository를 생성하고 이를 이용해 설정파일을 관리해보자.

![](https://i.imgur.com/5aqpn7O.png)

위와 같이 설정 파일을 보관하기 위한 Private Repository를 생성한다.

### `yml`파일 업로드

간단하게 config 디렉토리를만들고 다음과 같은 `application.yml`파일을 작성할 것이다. 실제로는 프로젝트에 필요한 `yml`파일 여러개를 작성하고 push하면 된다.

```yaml
message:
  honorific: 님
  greeting: 반갑습니다!
  secret: 비밀정보
```

위와 같은 간단한 `yml`파일을 작성하고 읽어온 설정 정보를 클라이언트에서 사용해 볼 예정이다.

![](https://i.imgur.com/ZuhkOn8.png)

`creating a new file`을 클릭해 `yml`파일을 작성해보자.

![](https://i.imgur.com/PprNFZd.png)

`myapp.yml`이라는 파일을 프로젝트 루트에 작성했다.

`myapp-local.yml`로 파일을 추가로 작성하게 된다면 Client의 활성 프로파일이 `local`인 경우에만 읽어와진다. 따라서 상황에 맞게 `yml`파일을 작성하는 것이 중요하다.

- `myapp.yml` : 디폴트 설정으로 어느 프로파일이 활성화되던 불러와진다.
- `myapp-{profile}.yml` : `{profile}`이 활성화 될 때만 불러와진다.

### SSL 설정

이제 Spring Cloud Config 서버가 private Repository에 접근할 수 있도록 SSL연결 설정을 해주어야 한다.

터미널에서 다음과 같은 명령어를 입력해 비대칭 키를 생성하자.

```sh
$ ssh-keygen -m PEM -t ecdsa -b 256
```

생성 시 파일 이름과 passphrase 관련 질문을 모두 넘기면 다음과 같이 `~/.ssh`에 `id_ecdsa.pub`으로 공개키가 생성된다.

![](https://i.imgur.com/82g1DYX.png)

이제 이 공개키를 GitHub Repository에 등록하자.

[Repository - Settings- Security - Deploy Keys - Add deploy key] 버튼을 누르면 등록할 수 있다.

![](https://i.imgur.com/GiM3Bgy.png)

`id_ecdsa.pub`의 내용을 붙여넣고 deploy key를 생성한다. Spring Cloud Config 서버가 write access를 가질 필요는 없으므로 체크박스는 체크하지 않고 진행한다. 이제 spring cloud config 서버가 private key를 가지고 해당 repository에 접근할 수 있다.

## Spring Cloud Config 서버 만들기

### Spring Cloud Config 서버 생성

가장 먼저 [start.spring.io](https://start.spring.io)에서 아래와 같이 dependency를 추가하고 프로젝트를 생성한다.

![](https://i.imgur.com/kYhB4ts.png)

Depencency로 `Config Server`를 추가하면 된다.

```groovy
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.1.1'
	id 'io.spring.dependency-management' version '1.1.0'
}

group = 'com.myapp'
version = '0.0.1-SNAPSHOT'

java {
	sourceCompatibility = '17'
}

repositories {
	mavenCentral()
}

ext {
	set('springCloudVersion', "2022.0.3")
}

dependencies {
	implementation 'org.springframework.cloud:spring-cloud-config-server'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

dependencyManagement {
	imports {
		mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
	}
}

tasks.named('test') {
	useJUnitPlatform()
}

```

프로젝트를 열고 `build.gradle`을 열어보면 dependency가 추가된 것을 확인할 수 있다.

### `application.yml` 설정

`src/main/resources` 아래에 `application.yml`을 설정해주자.

```yaml
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        encrypt:
          enabled: false
        git:
          default-label: main
          uri: git@github.com:Plan-A-project/infli-server-config.git
          # search-paths:
          ignoreLocalSshSettings: true
          strictHostKeyChecking: false
          hostKey: # ssh-keyscan 명령어로 확인
          hostKeyAlgorithm: # ssh-keyscan 명령어로 확인
          privateKey: |
            -----BEGIN EC PRIVATE KEY-----
            XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
            XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
            XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
            -----END EC PRIVATE KEY-----
          passphrase: # 비대칭키 생성 시 입력한 값
```

#### server.port

서버의 포트를 설정한다. 공식 문서에서 8888로 설정하고 있으니 8888로 맞춰준다.

#### encrypt 관련 설정

`ecrypt.enabled`를 `false`로 설정하고 있다. 이는 서버에서 설정 정보를 읽어올 때 암호화된 문자열을 서버에서 복호화시킬지 여부에 대한 설정이다. `false`로 설정하면 클라이언트에서 암호화된 문자열을 받아와 직접 복호화한다. 서버에서 암호화된 정보를 넘겨주기 때문에 보안에 조금 더 안전하게 작동할 수 있다.

#### git 관련 설정

##### `default-label`

읽어올 설정 정보의 브랜치를 설정한다.

##### `uri`

SSH를 통해 git에 접속하기 위한 URL이다.

![](https://i.imgur.com/pPCkmo0.png)

위와 같이 SSH 주소를 복사해 넣어준다.

##### `search-paths`

repository에서 탐색을 시작할 경로를 설정한다. `yml`파일이 `config` 디렉토리 안에 있다면 `config`라고 입력하면 된다. `search-paths`에 `config/**`와 같이 AntPath로 작성하면 작동하지 않는다.

##### `hostKey`, `hostKeyAlgorithm`

`hostKey`와 `hostKeyAlgorithm`은 터미널에서 다음 명령어로 찾을 수 있다.

```sh
$ ssh-keyscan -t ecdsa github.com
```

![](https://i.imgur.com/DOa4vhG.png)

#### `privateKey`, `passphrase`

또한 `privateKey`는 `~/.ssh` 디렉토리에서 `id_ecdsa`를 열어 복사후 넣는다. `|`를 빼먹지 말고 줄을 맞춰 작성하자.

`passphrase`는 key를 생성할 때 넣었던 값을 작성하면 된다.

### `@EnableConfigServer

이제 마지막으로 Config 서버의 마지막 설정이다. 다음과 같이 `@EnableConfigServer` 어노테이션만 추가하면 Spring Cloud Config 서버가 완성된다.

```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServerTestApplication {

	public static void main(String[] args) {
		SpringApplication.run(InfliConfigServerApplication.class, args);
	}

}
```

`http://localhost:8888/myapp/default`로 요청을 보내면 다음과 같이 확인할 수 있다.

![](https://i.imgur.com/mbBoVm5.png)

잘 작동하는 것으로 보인다.

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

위와 같은 경로로 설정정보를 받아올 수 있다.

`/myapp/default`는 `myapp`의 `default` 프로파일의 설정정보를 받아오는 것이다.

하지만 지금은 우리의 비밀정보가 노출된다. 이를 해결해 보자.

## 암호화 관련 설정

spring cloud config 서버의 `yml`파일의 맨 밑에 다음과 같이 추가한다.

```yml
encrypt:
  key: my-secret # 사용할 암호
```

정보를 암호화할 키를 넣는다. 사용할 암호를 임의로 지정해 적용하면 된다.

다시 서버를 작동시키고 `http://localhost:8888/encrypt`에 다음과 같이 `POST` 요청을 보낸다.

![](https://i.imgur.com/CQePdqS.png)

암호화 하고 싶은 정보를 `Body`에 담아 보내면 결과를 응답해준다.

Private Repository에 해당 정보를 넣어보자.

```yml
message:
  honorific: 님
  greering: 반갑습니다!
  secret: "{cipher}aadfc7c7841c2ef32e1abcc4e6860755f22fb5be06cde48106ae95029413d794"
```

위와 같이 민감한 정보를 대체할 수 있다. 쌍따옴표 안에 `{cipher}`를 넣고 암호화된 문자를 넣는다.

이제 다시 요청을 보내보면 다음과 같이 암호화된 정보를 받을 수 있다.

![](https://i.imgur.com/bjywRtB.png)

만약 `encrypt.enabled`를 `true`로 설정하면 다음과 같이 복호화된 정보를 받을 수 있다.

![](https://i.imgur.com/ApwqAD5.png)

하지만 이는 spring cloud config서버가 public에 노출될 경우 민감한 정보를 바로 받을 수 있어 `encrypt.enabled`를 `false`로 지정해 놓고 클라이언트에서 복호화하여 사용하는 것을 추천한다.

## Spring Cloud Config  클라이언트 적용

### Client 프로젝트 생성

이제 해당 서버에 붙을 수 있는 클라이언트 프로젝트를 만들어보겠다.

![](https://i.imgur.com/TBoPvR5.png)

위와 같이 Dependency에 `Config Client`만 추가하면 된다. `Spring Boot Actuator`는 `refresh` 엔드포인트를 이용해 프로젝트의 재기동 없이 설정정보를 반영하기 위해 추가했다. `Spring Web`은 설정을 잘 받아오는지 확인하는 용도로 추가했다.

기존 프로젝트에 적용하고 싶다면 `build.gradle`에 다음과 같이 추가하면 된다. `refresh`기능이 필요 없다면 `actuator`는 추가하지 않아도 된다.

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-actuator'
  implementation 'org.springframework.cloud:spring-cloud-starter-config'
}
```

### Client 설정

Client에서 Spring Cloud Config 서버에서 정보를 받아오기 위해 `application.yml`파일을 다음과 같이 설정한다.

```yml
spring:
  application:
    name: myapp
  profiles:
    active: local
  config:
    import: optional:configserver:http://localhost:8888
management:
  endpoints:  
    web:  
      exposure:  
        include: refresh
```

#### `spring.application.name`

위에서 말했던 것 처럼 애플리케이션의 이름을 지정한다. 애플리케이션의 이름을 지정하면 클라이언트를 설정 정보를 요청 할 때 `/myapp/local`과 같은 경로로 서버에 요청하게 된다.

만약 설정하지 않으면 default로 `application`이라는 이름으로 불러와지게 된다.

#### `spring.config.import`

설정 정보를 받아올 서버의 주소를 입력한다. `optional`은 서버에 접속할 수 없어도 프로젝트를 실행시킨다. `optional`을 제거한다면 서버에서 설정 정보를 불러오지 못해도 프로젝트를 실행시킨다.

### `refresh` 엔드포인트 활성화

`Spring Boot Actuator`의 `refresh` 엔드포인트를 활성화해 변경된 설정 정보를 프로젝트 재기동 없이 실시간으로 변경될 수 있게 한다.

### 환경 변수 설정

우리의 Spring Cloud Config 서버는 암호화된 정보를 그대로 가져온다. 따라서 클라이언트에서 해당 정보를 복호화해 사용해야 한다.

환경변수에 `ENCRYPT_KEY=my-secret` 값을 다음과 같이 적용한다. 암호화 관련 설정에서 사용한 비밀키를 넣으면 된다.

![](https://i.imgur.com/lpGQMIW.png)

`Edit Configurations`에서 다음과 같이 설정한다.

![](https://i.imgur.com/QuZrKA5.png)


### 설정 정보 적용 확인

설정 정보가 프로젝트에 적용되었는지 확인해 보자. 클라이언트에 다음과 같은 코드를 작성해 확인해보자.

```java
@Getter
@Component
public class MessageManager {

	@Value("${message.greeting}")
	public String greeting;

	@Value("${message.honorific}")
	public String honorific;

	@Value("${message.secret}")
	public String secret;

	public String hello(String name) {
		return greeting + ", " + name + honorific + " " +
			"secret은 다음과 같습니다. : " + secret;
	}

}

@RequiredArgsConstructor
@RestController  
public class GreetingController {  
  
   private final GreetingManager greetingManager;  

   @GetMapping("/hello/{name}")  
   public String hello(@PathVariable String name) {  
      return greetingManager.hello(name);  
   }  
}
```

위와 같이 작성하고  `/hello/홍길동`으로 요청을 보내면 다음과 같이 설정 정보가 잘 불러와지고 암호화된 정보도 복호화되어 작동하는 것을 확인할 수 있다.

![](https://i.imgur.com/fwvS6ri.png)
## Client 작동 해보기

### 설정 정보 우선 순위

설정 파일은 다음의 순서대로 읽어진다. 나중에 읽어지는 것이 우선순위가 높다. 앞의 정보와 겹치게 된다면 덮어씌워지므로 주의하자.

- 프로젝트의 application.yml
- 설정 저장소의 {application name}.yml
- 프로젝트의 application-{profile}.yml
- 설정 저장소의 {application name}-{profile}.yml

### refresh 해보기

이제 `/refresh` 엔드포인트를 활용해 설정정보를 실시간으로 변경해보자.

먼저 Private Repository의 `yml`을 다음과 같이 변경해본다.

```yml
message:
  honorific: 회원님
  greeting: 반갑습니다!
  secret: "{cipher}6fdeed8a7ecd76d7e315320c5aa261e94774b111cc5a8cbd82899dd0dff48a54"

```

그리고 클라이언트에 `/actuator/refresh`로 `POST`요청을 보내보자.

![](https://i.imgur.com/6BIhT39.png)

변경된 항목을 `Body`로 전송해 준다.

다시 `http:/localhost:8080/hello/홍길동`으로 요청을 보내보면 다음과 같이 클라이언트 서버의 재기동 없이 설정정보를 반영시키는 것을 볼 수 있다.

![](https://i.imgur.com/jsNs5Gj.png)
