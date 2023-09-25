---
layout: single
title: Spring REST Docs 사용법
categories: spring
tags:
  - spring
  - api-docs
  - spring-rest-docs
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: docs
header:
  teaser: https://i.imgur.com/qRmIrxY.png
---
# Spring REST Docs 제대로 알아보고 사용해보기

Spring 진영에서의 대표적인 API 문서화 자동화 도구에는 **Spring REST Docs**와 **Swagger**가 있다.

[Swagger VS Spring Rest Docs](https://dukcode.github.io/spring/swagger/#swagger-vs-spring-rest-docs)에서도 알아보았지만 **Spring REST Docs**는 Swagger와 달리 운영 코드에 비침투적이며, 테스트를 강제하기 때문에 많은 장점이 있다.

가장 먼저 **Spring REST Docs**의 작동 과정을 알아보자.

## Spring REST Docs 작동 과정

![](https://i.imgur.com/HcIM7u5.png)

테스트에서 `spring-rest-docs-mockmvc` 라이브러리를 이용하면 테스트 실행 시, 문서 조각인 스니펫을 얻을 수 있다. 개발자는 문서의 뼈대가 될 `adoc`파일을 따로 작성하고, 여기서 테스트 결과로 나온 스니펫을 include 시킨다.

두 종류의 파일이 완성되었으면, `asciidoctor` 태스크를 실행시킨다. 이 태크스는 `adoc`파일을 엮어 `html`형식으로 만들어 개발 문서를 완성시켜 주는 역할을 한다.

테스트 후 `asciidoctor` 플러그인을 실행시키는 것은 대단히 번거로워 보인다. Spring REST Docs 설정을 하면서 해당 과정을 자동화 할 수 있게 설정해보자.

## Spring REST Docs 설정

### 공식 문서 설정

Spring Boot를 사용한다면 설정은 [Spring REST Docs 공식 문서](https://docs.spring.io/spring-restdocs/docs/current/reference/htmlsingle/#getting-started-build-configuration)를 따라 다음과 같다.

```groovy
plugins {  
    // 생략
    id 'org.asciidoctor.jvm.convert' version '3.3.2' // (1)
}  
  
// 생략
  
configurations {  
    asciidoctorExt // (2)
    // 생략
}  
  
// 생략
  
ext {  
    set('snippetsDir', file("build/generated-snippets")) // (3)
    // 생략
}  
  
dependencies {  

    // 생략

    asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor' // (2)
    testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc' // (4)
}  

task.named('test') {
    outputs.dir snippetsDir // (3)
    // 생략
}

asciidoctor { // (5)
    input.dir snippetsDir
    configurations 'asciidoctorExt' // (2)
    depensOn test
}

bootJar { // (6)
    dependsOn asciidoctor

    from( "${asciidoctor.outputDir}") {
        into 'static/docs'
    }
}

```

1. Asciidoctor 플러그인을 추가한다. Asciidoctor 플러그인에 담긴 asciidoctor 태스크가 `adoc`파일과 스니펫을 조합해 `html`로 변경해주는 역할을 한다.
2. dependencies 블럭에 asciidoctorExt로 라이브러리를 불러올 수 있도록 선언한다. 해당 라이브러리는 **개발자가 작성하는 `adoc`파일에서 `{{snippet}}`을 통해 스니펫 경로에 있는 스니펫들을 쉽게 불러올 수 있도록 한다.** 해당 라이브러리가 있다면 직접 경로를 입력하지 않아도 되서 편리하게 문서를 작성할 수 있다. 그리고 이 라이브러리를 `asciidoctor` 태스크에 적용한다.
3. 스니펫들이 저장될 `snippetDir` 변수를 설정한다. `snippetDir`을 test의 `output.dir`로 설정해 스니펫들이 해당 경로에 저장되도록 설정한다.
4. `MockMvc`에 기반해서 **스니펫을 뽑아낼 수 있도록 하는 라이브러리**를 디펜던시에 추가한다. 만약 테스트 방식으로 `MockMvc`를 사용하지 않고 `WebTestClient`나 `REST Assured`를 사용하는 환경이라면 `spring-restdocs-webtestclient`나 `spring-restdocs-restassured`를 사용할 수 있다.
5. `adoc`파일을 `html`파일로 변환시켜주는 `asciidoctor` 설정을 한다. `input.dir`을 이전에 설정해둔 스니펫 경로인 `snippetDir`로 설정한다. test 태스크에 의존하도록 `depensOn` 설정을 하여 `asciidoctor` 태스크를 실행하면 동작 수행 전에 test 태스크를 수행해서 스니펫을 새로 만들고 새로운 `html`파일을 만들도록 설정한다.
6. `bootJar` 태스크를 실행하면 `asciidoctor` 태스크가 실행되게 한다. 또한 `asciidoctor` 태스크는 `test` 태스크에 의존하고 있기 때문에 `bootJar`를 실행하게 되면 `test` - `asciidoctor` - `bootJar` 순서로 실행된다. 또한 `html`로 만들어진 문서를 기본 `outputDir`인 `build/docs/asciidoc`에서 `static/docs`로 복사해 배포 시 `/docs/**` URL로 접속해 문서를 확인할 수 있게 한다.

위의 설정으로 `asciidoctor` 태스크를 실행하면 `build/docs/asciidoc`에서 `html`문서를 확인할 수 있다. 또한 배포 시 `bootJar` 태스크를 실행시키면 자동으로 `asciidoctor` 태스크가 실행되고 결과물이 `static/docs`에 담기기 때문에 구동 시 `/docs/**`로 접속하면 API문서를 확인할 수 있다.

하지만 개발자라면 문서를 작성하면서 잘 작성되고 있는지 눈으로 확인하고 싶을 것이다. 공식 문서의 설정을 추가해 보자.
### 추가 설정

기존 방식대로라면 문서를 작성하고 local에서 Intellij run을 돌려봐도 작성한 문서가 반영되지 않는다. `test` 태스크가 진행될 때 `asccidoctor` 태스크가 실행되고 결과물을 `/static/docs`로 복사해오도록 변경해 보자. 그러면 테스트를 돌리고 intellij run을 하면 우리가 작성한 문서가 반영될 것이다.

최종적인 `build.gradle`은 다음과 같다. 

```groovy
plugins {  
    // 생략
    id 'org.asciidoctor.jvm.convert' version '3.3.2'
}  
  
// 생략
  
configurations {  
    asciidoctorExt
    // 생략
}  
  
// 생략
  
ext {  
    set('snippetsDir', file("build/generated-snippets"))
    // 생략
}  
  
dependencies {  

    // 생략

    asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor'
    testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc'
}  
  
// 생략

  tasks.named('testClasses') { // (1)
    doFirst {
        delete file('build/docs/asciidoc')
    }
}

tasks.named('test') {  
    outputs.dir snippetsDir
    // 생략
    finalizedBy asciidoctor // (2)
}  
  
tasks.named('asciidoctor') {
    dependsOn test  
    configurations 'asciidoctorExt'  
    inputs.dir snippetsDir  
    finalizedBy copyDocument // (3)
    doFirst { // (4)
        delete file('src/main/resources/static/docs')  
    }  
}  
  
task copyDocument(type: Copy) { // (5)
    dependsOn asciidoctor  
    from file('build/docs/asciidoc')  
    into file('src/main/resources/static/docs')  
}

bootJar {  
    dependsOn asciidoctor  
    doFirst {  
        delete file('static/docs')  
    }  
    from("${asciidoctor.outputDir}") {  
        into 'static/docs'  
    }  
}
```

1. 개별 클래스 테스트가 진행되기 전에 이전에 생성되었던 html 파일을 삭제하도록 한다. 만약 파일 명을 바꾸거나 위치를 옮길 때, 바꾸기 전의 html파일이 원래 위치에 그대로 존재하는 것을 방지하기 위해 설정했다. (gradle이 증분 빌드를 사용하기 때문인 것으로 추측된다.)
2. `test` 태스크가 끝나면 `asccidoctor` 태스크가 실행되도록 `finalizedBy`를 통해 설정한다.
3. `asciidoctor` 태스크가 끝나면 새로 작성한 태스크인 `copyDocument`가 실행되도록 설정한다.
4. `asciidoctor` 태스크가 시작하기 전 `static/docs`에 있는 이전 문서들을 삭제한다.
5. `asciidoctor` 태크스의 결과물을 `static/docs`로 옮긴다. 따라서 intellij run 시에 `/docs/**` URL로 API 문서에 접근할 수 있다.
6. 기존 `bootJar`  태스크 설정에  기존 html 파일을 삭제하는 로직을 추가하였다. `build/resource/main/static/docs` 하위에 존재하는 이전 빌드의 파일을 삭제했다. 만약 파일의 위치를 변경하거나 이름을 변경했을 때 삭제 설정을 하지 않으면 기존의 파일이 그대로 남아있다. (아마도 gradle이 증분 빌드를 사용하기 때문인 것으로 추측된다.)

위처럼 설정하면 local 환경에서 API 문서를 서버를 통해 확인할 수 있다.

![](https://i.imgur.com/CZGHsPG.png){:.align-center width="400" }


- 문서 업데이트를 위해 `test` 태스크를 실행한다. (gradle task로 실행해야 한다. intellij에서 테스트를 실행하면 태스크 체인이 작동하지 않아 `asciidoctor`가 작동하지 않는다.)
2. 이제 gradle을 통해 서버를 구동하던 intellij run을 통해 서버를 구동하던 `/docs/**` URL을 통해 API 문서에 접근할 수 있다. (`/static/docs`에 이미 파일이 존재하므로)

많은 블로그 자료들에서 `build` 태스크가 실행될 때 `copyDocument`가 실행되도록 하기 위해서 다음처럼 설정하고 있다.

```groovy
build {
    dependsOn copyDocument
}
```

하지만 위는 의존관계의 순서가 잘못된 것이다. 만약 `build` 태스크를 실행했을 때 `copyDocument`가 작동하게 하려면 다음과 같이 입력해야 한다.

```groovy
build {
    finalizedBy copyDocument
}
```

## 테스트 메서드 작성

### 기본 작성법

기본적인 작성법은 `MockMvc`의 `andDo()`메서드 안에 `RestDocumentRequestHandler`를 생성하고 작성하고 싶은 내용을 넣는 것이다. 공식문서의 [Document your API](https://docs.spring.io/spring-restdocs/docs/current/reference/htmlsingle/#documenting-your-api) 섹션을 살펴보면 `RestDocumentRequestHandler`에 내용을 채우는 법을 알 수 있다.

가장 먼저 `@AutoConfigureRestDocs`를 설정해서 `MockMvc`에서 Spring REST Docs를 사용할 수 있도록 설정한다. 그리고 `andDo()`메서드 안에 문서의 내용을 작성할 수 있다.

다음은 간단한 회원 가입 API에 대한 Spring REST Docs 스니펫을 만들기 위한 테스트이다.

```java
@Import(SecurityConfiguration.class)  
@AutoConfigureRestDocs  
@WebMvcTest(MemberSignupController.class)  
class MemberSignupControllerTest2 {  
  
  @Autowired  
  protected MockMvc mockMvc;  
  @Autowired  
  protected ObjectMapper objectMapper;  
  
  @MockBean  
  private MemberSignupUseCase memberSignupUseCase;  
  
  @Test  
  public void signup_201() throws Exception {  
    // given  
  
    Member member = Member.withEncodedPassword(1L,  
        "duk9741@gmail.com",  
        "encoded-password",  
        "dukcode",  
        LocalDate.of(1995, 1, 10));  
  
    given(memberSignupUseCase.signup(BDDMockito.any())).willReturn(member);  
  
    // when  
    // then
    MemberSignupRequest request = new MemberSignupRequest(  
        "duk9741@gmail.com",  
        "123456",  
        "raw@password1",  
        "dukcode",  
        LocalDate.of(1995, 1, 10));  
  
    mockMvc.perform(post("/member")  
            .contentType(APPLICATION_JSON)  
            .content(objectMapper.writeValueAsString(request)))  
        .andExpect(status().isCreated())  
        .andExpect(jsonPath("$.id").value(1L))  
        .andExpect(jsonPath("$.email").value("duk9741@gmail.com"))  
        .andExpect(jsonPath("$.nickname").value("dukcode"))  
        .andDo(MockMvcResultHandlers.print()) // 요청, 응답 출력
        .andDo(document("{class-name}/{method-name}",  // 문서 이름 설정
            preprocessRequest(  
                modifyHeaders()  // 헤더 내용 수정
                    .remove("Content-Length")  
                    .remove("Host"),  
                prettyPrint()),  // 한 줄로 출력되는 json에 pretty 포멧 적용
            preprocessResponse(  
                modifyHeaders()  
                    .remove("Content-Length")  
                    .remove("X-Content-Type-Options")  
                    .remove("X-XSS-Protection")  
                    .remove("Cache-Control")  
                    .remove("Pragma")  
                    .remove("Expires")  
                    .remove("X-Frame-Options"),  
                prettyPrint()),  
            requestFields(  // 요청 필드 추가
                fieldWithPath("email")  // 필드 path 추가
                    .type(STRING)  // 타입 지정
                    .description("멤버 이메일") // 설명 지정 
                    // 추가 속성 지정
                    .attributes(new Attribute("constraints", "이메일 형식")),  
                fieldWithPath("nickname")  
                    .type(STRING)  
                    .description("멤버 닉네임")  
                    .attributes(new Attribute("constraints", "2자 이상 10자 이하 형식")),  
                fieldWithPath("password")  
                    .type(STRING)  
                    .description("멤버 패스워드")  
                    .attributes(  
                        new Attribute("constraints", "8자 이상 20자 이하 최소 1글자 이상의 영어, 숫자, 특수문자 포함")),  
                fieldWithPath("emailVerificationCode")  
                    .type(STRING)  
                    .description("이메일 인증 코드")  
                    .attributes(new Attribute("constraints", "6자리 숫자")),  
                fieldWithPath("birthday")  
                    .type(STRING)  
                    .description("멤버 생년월일")  
                    .optional()  
                    .attributes(new Attribute("constraints", "YYYY-MM-DD"))  
            ),
            responseFields(  // 응답 필드 추가
            fieldWithPath("id")
                .type(NUMBER)
                .description("멤버 고유 번호"),
            fieldWithPath("email")
                .type(STRING)
                .description("멤버 이메일")
            fieldWithPath("nickname")
                .type(STRING)
                .description("멤버 닉네임")
            )));  
  }  
}
```

위의 간단한 API에 대한 요청과 응답에 대한 테스트 메서드를 작성하고 이를 실행하면 `build/generated-snnipets`아래 `adoc`형식의 스니펫이 생성된다.

![](https://i.imgur.com/HKDioUj.png){:.align-center width="400" }

기본적으로는 다음과 같은 스니펫들이 생성된다.

- curl-request.adoc
- http-request.adoc
- httpie-request.adoc
- http-response.adoc
- request-body.adoc
- response-body.adoc

테스트 코드에서 어떤 snippet을 생성할건지에 따라 추가적인 조각이 생성될 수 있다.

- response-fields.adoc
- request-parameters.adoc
- request-parts.adoc
- path-parameters.adoc
- request-parts.adoc

이제 이를 우리가 작성할 뼈대 `adoc`에서 include해서 사용하면 된다.
### 테스트 코드 리팩토링(코드를 줄이기)

위는 간단한 API에 대한 테스트코드이다. 하지만 문서 설정 내용을 보면 해당 API에만 적용될 수 있는 내용이 있고, 전체 API에서 공통적으로 사용되어야 될 내용이 있다. 공통적으로 사용해야 할 내용을 분리해서 코드를 간결하게 만들어보자.

공통을 처리해주어야 할 내용을 다음과 같다.

- 요청과 응답 json에 pretty format을 적용하는 부분
- 문서 이름을 처리해 주는 부분
- header를 숨기는 부분

- `ObjectMapper`, `MockMvc` 등 API 문서 작성 필수 클래스 선언 부분
- 요청 응답 print 적용
- 추가 속성 지정의 객체 생성 반복 코드

위의 테스트 코드에서는 `@AutoConfigureRestDocs`를 통해 커스터마이즈된 `MockMvc`를 사용하고 default로 생성도니 `RestDocumentationRequestHandler`를 사용했다. 이를 미리 커스터마이즈해서 적용하면 공통 코드를 줄일 수 있을 것이다.

가장 먼저 `RestDocumentationRequestHandler`를 커스터마이즈해보자.

```java
@TestConfiguration  
public class RestDocsConfiguration {  
  
  @Bean  
  public RestDocumentationResultHandler restDocumentationResultHandler() {  
    return MockMvcRestDocumentation.document(  
        "{class-name}/{method-name}",  // 문서 이름 설정
        preprocessRequest(  // 공통 헤더 설정
            modifyHeaders()  
                .remove("Content-Length")  
                .remove("Host"),  
            prettyPrint()),  // pretty json 적용
        preprocessResponse(  // 공통 헤더 설정
            modifyHeaders()  
                .remove("Content-Length")  
                .remove("X-Content-Type-Options")  
                .remove("X-XSS-Protection")  
                .remove("Cache-Control")  
                .remove("Pragma")  
                .remove("Expires")  
                .remove("X-Frame-Options"),  
            prettyPrint())    // pretty json 적용
    );  
  }  
}
```

위와 같이 `test` 디렉토리 아래에 테스트 전용 설정 파일을 구성한다. **문서 이름**과 **공통적으로 설정할 헤더 설정**, **json pretty print**을 미리 해둔다. (실제로 테스트를 돌려보고 스니펫을 열어보면 개발 환경에 따라 다른 헤더가 나올 수 있다. 나는 Spring Security를 사용하고 있어서 `HeaderWriterFilter`에서 추가된 헤더들을 지워주고 있다.)

위의 설정으로 다음과 같은 공통 부분을 해결할 수 있다.

- 요청과 응답 json에 pretty format을 적용하는 부분
- 문서 이름을 처리해 주는 부분
- header를 숨기는 부분

이제는 미리 만들어둔 `RestDocumentaionRequestHandler`를 `MockMvc`에 적용해보자. `RestDocSupport` 추상 클래스를 선언하고 API 문서 테스트 클래스가 이를 상속하게 하자.

```java
@Disabled  
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)  
@Import({RestDocsConfiguration.class, SecurityConfiguration.class})  
@ExtendWith(RestDocumentationExtension.class)  
public class RestDocsTestSupport {  
  
  @Autowired  
  protected RestDocumentationResultHandler restDocs;  
  @Autowired  
  protected MockMvc mockMvc;  
  @Autowired  
  protected ObjectMapper objectMapper;  
  
  protected static Attribute constraints( // contraints Attribute 간단하게 추가
      final String value) {
    return new Attribute("constraints", value);
  }

  
  @BeforeEach  
  void setUp(final WebApplicationContext context,  
      final RestDocumentationContextProvider provider) {  
    this.mockMvc = MockMvcBuilders.webAppContextSetup(context)  
        .apply(MockMvcRestDocumentation.documentationConfiguration(provider))
        .alwaysDo(MockMvcResultHandlers.print()) // print 적용
        .alwaysDo(restDocs) // RestDocsConfiguration 클래스의 bean 적용
        .build();  
  }  
}
```

위의 내용을 [공식 문서](https://docs.spring.io/spring-restdocs/docs/current/reference/htmlsingle/#getting-started-documentation-snippets-setup)를 참고하면 더 세세하게 적용할 수 있다.

`@Disabled`로 테스트 할 클래스에서 해당 클래스 제외해준다. `@Import`로 작성한 `RestDocsConfiguration`을 추가해준다. `@ExtendWith`에 `RestDocumentationExtension`을 설정해주어 context를 제공해 Spring REST Docs가 잘 작동할 수 있도록 한다.

이 클래스를 상속한 클래스가 `MockMvc`와 `ObjectMapper`를 선언할 필요가 없도록 미리 설정해준다.

`Attribute`를 간단하게 추가할 수 있도록 메서드를 추가한다.

`@BeforeEach`에서 `MockMvc`의 커스터마이즈를 진행한다. `RestDocsConfiguration`에서 설정했던 `RestDocumentationResultHandler`를 적용하고, 테스트 시 요청과 응답을 출력할 수 있도록 `print()`메서드도 적용한다.

이제 위의 서포트 클래스를 적용해보자.

```java
package me.coonect.coonect.member.adapter.in.web;  
  
@WebMvcTest(MemberSignupController.class)  
class MemberSignupControllerTest extends RestDocsTestSupport {  
  
  @MockBean  
  private MemberSignupUseCase memberSignupUseCase;  
  
  @Test  
  public void signup_201() throws Exception {  
    // given  
  
    Member member = Member.withEncodedPassword(1L,  
        "duk9741@gmail.com",  
        "encoded-password",  
        "dukcode",  
        LocalDate.of(1995, 1, 10));  
  
    given(memberSignupUseCase.signup(BDDMockito.any())).willReturn(member);  
  
    // when  
    // then
    MemberSignupRequest request = new MemberSignupRequest(  
        "duk9741@gmail.com",  
        "123456",  
        "raw@password1",  
        "dukcode",  
        LocalDate.of(1995, 1, 10));  
  
    mockMvc.perform(post("/member")  
            .contentType(APPLICATION_JSON)  
            .content(objectMapper.writeValueAsString(request)))  
        .andExpect(status().isCreated())  
        .andExpect(jsonPath("$.id").value(1L))  
        .andExpect(jsonPath("$.email").value("duk9741@gmail.com"))  
        .andExpect(jsonPath("$.nickname").value("dukcode"))  
        .andDo(restDocs.document(  
            requestFields(  
                fieldWithPath("email")  
                    .type(STRING)  
                    .description("멤버 이메일")  
                    .attributes(constraints("이메일 형식")),  
                fieldWithPath("nickname")  
                    .type(STRING)  
                    .description("멤버 닉네임")  
                    .attributes(constraints("2자 이상 10자 이하 형식")),  
                fieldWithPath("password")  
                    .type(STRING)  
                    .description("멤버 패스워드")  
                    .attributes(constraints("8자 이상 20자 이하 최소 1글자 이상의 영어, 숫자, 특수문자 포함")),  
                fieldWithPath("emailVerificationCode")  
                    .type(STRING)  
                    .description("이메일 인증 코드")  
                    .attributes(constraints("6자리 숫자")),  
                fieldWithPath("birthday")  
                    .type(STRING)  
                    .description("멤버 생년월일")  
                    .optional()  
                    .attributes(constraints("YYYY-MM-DD"))  
            ),  
            responseFields(  
                fieldWithPath("id")  
                    .type(NUMBER)  
                    .description("멤버 고유 번호"),  
                fieldWithPath("email")  
                    .type(STRING)  
                    .description("멤버 이메일"),  
                fieldWithPath("nickname")  
                    .type(STRING)  
                    .description("멤버 닉네임")  
        )));  
  
  }  
  
}
```

코드가 핵심만 남고 간단해졌다.

```java
fieldWithPath("email")  
    .type(STRING)  
    .description("멤버 이메일")  
    .attributes(constraints("이메일 형식"))
```

위와 같은 부분도 추가적으로 공통 코드를 더 뽑아내서 `enum`으로 만들어 간단하게 만들 수 있겠지만 지금은 이정도까지만 진행해 보겠다.

### 문서화

이제 테스트 코드를 통해 스니펫을 뽑아낼 수 있게 되었다. 하지만 `request-fields.adoc`파일을 보면 우리가 설정했던 `constraints` 어트리뷰트가 표현되지 않은 것을 알 수 있다. 또한 optional로 작성된 부분도 표현되어 있지 않다. 이를 해결해보자.

#### 커스텀 스니펫 만들기

스니펫은 템플릿을 통해 생성된다. 기본 템플릿은 문서 앞에 `default`가 붙어있다. `default-request-fields.snippet`을 확인해 보면 다음과 같이 작성되어 있다.

```html
|===  
|Path|Type|Description  
  
{% raw %}{{#{% endraw %}fields}}
|{% raw %}{{#{% endraw %}tableCellContent}}`+{% raw %}{{p{% endraw %}ath}}+`{% raw %}{{/{% endraw %}tableCellContent}}
|{% raw %}{{#{% endraw %}tableCellContent}}`+{% raw %}{{t{% endraw %}ype}}+`{% raw %}{{/{% endraw %}tableCellContent}}
|{% raw %}{{#{% endraw %}tableCellContent}}{{description}}{% raw %}{{/{% endraw %}tableCellContent}}
  
{% raw %}{{/{% endraw %}fields}}
|===
```

`Mustache` 문법을 통해 작성되어 있다. path와 type, description만 불러와서 테이블로 만드는 것을 볼 수 있다. 우리가 추가한 어트리뷰트를 추가해서 커스텀 스니펫 템플릿을 만들어보자.

![](https://i.imgur.com/mTsZ9Px.png)


커스텀 스니펫은 src/test/resources/org/springframework/restdocs/templates 디렉토리 하위에 작성하면 된다. `request-fields.snippet`파일을 위의 경로에 만들자. 그러면 디폴트 스니펫 템플릿 대신 작동하게 된다.
다음과 같이 어트리뷰트를 추가하자.

```html
|===  
|필드명|타입|필수여부|제약조건|설명  
{% raw %}{{#{% endraw %}fields}}  
|{% raw %}{{#{% endraw %}tableCellContent}}`+{% raw %}{{p{% endraw %}ath}}+`{% raw %}{{/{% endraw %}tableCellContent}}  
|{% raw %}{{#{% endraw %}tableCellContent}}`+{% raw %}{{t{% endraw %}ype}}+`{% raw %}{{/{% endraw %}tableCellContent}}  
|{% raw %}{{#{% endraw %}tableCellContent}}{% raw %}{{^{% endraw %}optional}}O{% raw %}{{/{% endraw %}optional}}{% raw %}{{#{% endraw %}optional}}X{% raw %}{{/{% endraw %}optional}}{% raw %}{{/{% endraw %}tableCellContent}}  
|{% raw %}{{#{% endraw %}tableCellContent}}{% raw %}{{#{% endraw %}constraints}}{% raw %}{{{% endraw %}{% raw %}.{% endraw %}}}{% raw %}{{/{% endraw %}constraints}}{% raw %}{{/{% endraw %}tableCellContent}}  
|{% raw %}{{#{% endraw %}tableCellContent}}{% raw %}{{{% endraw %}description}}{% raw %}{{/{% endraw %}tableCellContent}}  
{% raw %}{{/{% endraw %}fields}}  
|===
```

위와 같이 작성후 테스트를 돌려보면 다음과 같이 우리가 추가한 어트리뷰트가 추가되어 스니펫이 작성되는 것을 확인할 수 있다. 아래와 같이 intellij에서 `adoc`파일의 프리뷰를 확인하려면 [AsciiDoc 플러그인](https://intellij-asciidoc-plugin.ahus1.de)을 설치하면 된다.

![](https://i.imgur.com/F6Y6jBH.png)

##### .snippet 파일을 AsciiDoc로 인식하지 않을 때 (IntelliJ)

![](https://i.imgur.com/U8X4YMN.png)

- File > Settings > Editor > File Types
- Recognized File Types 에서 AsciiDoc files을 선택
- File name patterns에 .snippet 추가

#### 문서 작성

이제 스니펫들을 include하여 뼈대 문서를 작성해보자. 문서의 경로는 `src/docs/asciidoc` 디렉토리를 생성해 작성하면 된다.

`member.adoc`을 작성하는 예시는 다음과 같다. [AsciiDoctor 가이드 공식 문서](https://asciidoctor.org/docs/asciidoc-writers-guide/#writing-in-asciidoc)를 참고해서 작성하면 된다.

```html
= Coonect API Document  
:doctype: book  
:icons: font  
:source-highlighter: highlightjs  
:toc: left  
:toclevels: 2  
  
== Member 관련 API  

=== 회원 가입  
  
==== 요청  
  
include::{snippets}/member-signup-controller-test/signup_201/http-request.adoc[]  
  
==== 요청 필드  
  
include::{snippets}/member-signup-controller-test/signup_201/request-fields.adoc[]  
  
==== 응답  
  
include::{snippets}/member-signup-controller-test/signup_201/http-response.adoc[]  
  
==== 응답 필드  
  
include::{snippets}/member-signup-controller-test/signup_201/response-fields.adoc[]
```

위와 같이 작성하고 gradle 테스트를 돌려 html 문서를 업데이트하고 서버를 실행해 `/docs/member.html`로 접속해보자.

![](https://i.imgur.com/rCdEQBh.png)


위와 같이 문서를 확인할 수 있다. AsciiDoctor문법을 참고하여 추가적인 설명을 덛붙힐 수 있다.

[다음 포스트](https://dukcode.github.io/spring/spring-rest-docs-2/)에서는 API문서의 공통 영역, 예를 들어 error response의 enum값을 문서화 하는 방법, 링크를 통해 adoc 파일을 관리하는 법 등을 알아보겠다.