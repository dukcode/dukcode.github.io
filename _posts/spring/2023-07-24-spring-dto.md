---
layout: single
title: "[개발고민] Spring DTO는 어떻게 작성하고 변환해야 할까?"
categories: spring
tag: [개발고민, spring, DTO, 생성자, getter, setter, mapstruct]
toc: true
toc_sticky: true
author_profile: true
sidebar:
    nav: "docs"
header:
  teaser: "https://i.imgur.com/aR5r9MH.png"
# search: false
---

  💡 **개발고민**은 개발을 공부하며 했던 저의 생각들 입니다. 정답이 아니며 정답을 찾아가는 과정이라고 봐주시면 감사하겠습니다. [Github Repository](https://github.com/dukcode/boilerplate)<br>
  {: .notice--info} 
  
# Spring DTO는 어떻게 작성하고 변환해야 할까?

![](https://i.imgur.com/aR5r9MH.png){:.align-center}

REST API 프로젝트를 진행하며 API가 증가할 수록, **DTO**(Data Transfer Object)의 숫자도 그에 따라 증가했다. 그 동안은 DTO를 구현하며 관성적으로 lombok의 `@Data` 애노테이션을 사용했다.

`@Data`는 강력한 애노테이션이며, **클래스의 모든 요소들을 열어 놓는다**. 하지만 이는 좋은 프로그래밍이 아니다. 좋은 프로그래밍은 클래스에 대한 **접근 권한을 최소화** 해야한다.

**DTO**가 필요한 **기능**은 하면서 외부에 대해 **최소한으로 열려있게** 하고 싶었다. 또, 그 기준을 만족한다면, **DTO**와 **Entity**의 **변환**은 어떻게 일어나야 효율적이고, 또 **어느 레이어**에서 일어나야 좋은 코드를 짤 수 있는지 알고 싶었다.

위의 주제에 대한 여러 내용을 공부해 보며 나름의 **DTO의 구현과 사용**에 대한 **Best Practice**를 만들고 싶었다.

- `@RequestBody`를 위한 DTO의 최소 조건
- `@ResponseBody`를 위한 DTO의 최소 조건
- `@RequestBody`를 위한 DTO의 최소 조건
- DTO와 Entity 사이의 변환 방식
- DTO와 Entity의 변환 레이어

직렬화와 역직렬화에 먼저 알아보고, 위의 5개의 주제에 대해 차례대로 살펴보자.

## 직렬화와 역직렬화

컨트롤러의 파라미터에 `@RequestBody`를 선언하면, 클라이언트가 HTTP Body에 담아 보낸 **JSON**은 **객체**로 매핑된다. 또한 `@ResponseBody`를 컨트롤러 메서드에 선언하면 리턴할 **객체**를 **JSON**으로 변환해 클라이언트에게 전달한다. `@ModelAttribute`는 컨트롤러 메서드의 객체 파라미터에 선언하면 클라이언트가 전송한 **쿼리 파라미터** 또는 `x-www-form-urlencoded` 형식의 HTTP Body를 **객체**로 매핑해준다.

**스프링**은 애노테이션을 감지해 객체를 데이터로, 데이터를 객체로 변환시킨다. 이를 직렬화 또는 역직렬화라고 한다.

- **직렬화** : 객체를 데이터로 변환 (ex. **객체** -> **JSON**)
- **역직렬화** : 데이터를 객체로 변환 (ex. **JSON** -> **객체**)


## Jackson

스프링부트는 JSON과 객체 사이를 직렬화/역직렬화를 해주는 라이브러리를 이미 내장하고 있다. 라이브러리의 이름은 **Jackson**이다. `@RequestBody`와 `@ResponseBody`의 직렬화/역직렬화를 위한 DTO의 최소 조건을 알아보려면 **Jackson**에 대해 먼저 알아야 한다.

![](https://i.imgur.com/wGIALTO.png){:.align-center width="500" }

스프링부트 `3.1.2`버전 기준 `Jackson`은 `2.15.2` 버전을 탑재하고 있다.

**Jackson**의 `ObjectMapper`는 실제로 **JSON**과 **객체**간의 변환을 담당하는 클래스이다. 리플렉션을 통해 JSON String과 객체 사이를 변환하는데, 다음과 같이 사용할 수 있다.

```java
ObjectMapper objectMapper = new ObjectMapper();

Member member = new Member("이름","닉네임");

String json = objectMapper.writeValueAsString(member);
Member deserializedMember = objectMapper.readValue(json, Member.class);
```

### Spring Boot AutoConfiguration

스프링부트는 이미 `ObjectMapper`를 AutoConfiguration을 통해 **추가 설정**을 한 후 Bean으로 자동으로 등록시킨다. 즉, 우리는 `@Autowired`를 통해 추가 설정된 `ObjectMapper` Bean을 가져올 수 있다. AutoConfiguration 코드를 따라가다 보면 어떤 부분을 추가 설정했는지 확인할 수 있다.

---

<details markdown="1">
<summary><b>[ObjectMapper AutoConfiguration 방식 알아보기]</b></summary>
<br>
AutoConfiguration은 단순히 기본 생성자를 통해 `ObjectMapper`를 생성하지 않고 팩토리를 통해 **추가 설정**을 마친 후 Bean을 생성한다. `JacksonAutoConfiguration`에서 다음의 코드를 찾을 수 있다.

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(Jackson2ObjectMapperBuilder.class)
static class JacksonObjectMapperConfiguration {

    @Bean
    @Primary
    @ConditionalOnMissingBean
    ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
        return builder.createXmlMapper(false).build();
    }

}
```

빌더를 통해 `ObjectMapper` Bean을 생성하고 있음을 알 수 있다. `Jackson2ObjectMapperBuilder`는 `ObjectMapper`를 쉽게 설정하고 생성하기 위해 스프링이 만든 클래스이다. 해당 클래스도 다음과 같이 Bean으로 생성된다.

![](https://i.imgur.com/rxnRHR5.png){:.align-center}


`customize(builder)`메서드를 통해 커스텀한 설정이 이루어 지고 있는 것을 알 수 있다. 코드는 다음과 같다.

![](https://i.imgur.com/rlgJ91l.png){:.align-center}

다시 `ObjectMapper`의 Bean 등록 메서드의 `build()`메서드를 타고 들어가보자. `build()`메서드를 타고 들어가면 `createXmlMapper`는 `false`로 설정했으니, `configure(mapper)`를 호출하는 것을 알 수 있다.

![](https://i.imgur.com/l8qca1o.png){:.align-center}

즉, `configure(mapper)`메서드의 설정과 `customize(builder)`의 메서드를 살펴보면 어떤 설정이 디폴트 설정과 다른지 알 수 있다. 하지만 직접 하나하나 비교해 볼 필요는 없고 어떤식으로 `ObjectMapper` Bean이 등록되는지만 알아두면 될 것같다.

</details>

---

결론은 Spring Boot AutoConfiguration은 `ObjectMapper`의 디폴트 설정에서 다음과 같은 값을 비활성화 시킨다. (참고 : [Baeldung](https://www.baeldung.com/spring-boot-customize-jackson-objectmapper))

- **MapperFeature.DEFAULT_VIEW_INCLUSION**
    - `ObjectMapper`는 `View`를 설정해 원하는 멤버 변수를 매핑에 포함시킬지 말지 결정할 수 있다. 객체 멤버변수의`@JsonView` 선언을 통해 어떤 `View`에 포함될 것인지 정하는데, 이 애노테이션이 없는 멤버 변수가 매핑에 포함될 것인지 아닌지 정하는 옵션이다.(default : true)
- **DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES**
    - 역직렬화에서 객체에 존재하지 않는 프로퍼티가 존재할  때, 역직렬화를 진행할 것인지 FAIL시킬 것인지 정하는 옵션이다. (default : true) 
- **SerializationFeature.WRITE_DATES_AS_TIMESTAMPS**
    - 시간에 관련된 객체를 변환할 때, 64비트의 numeric timestamps(ex. 1690193106)로 변환할 것인가 `String`(ex. "2023-07-24T10:05:58.179+00:00")으로 변환할 것인지 결정하는 옵션이다. (default: true)

우리는 DTO를 직렬화/역직렬화하기 위한 최소 조건을 알아보고 싶었다. 즉, `ObjectMapper`가 **어떻게 객체 정보를 가져오고 어떤 방법으로 객체에 값을 넣을 수 있는지** 궁금했다. 하지만 스프링 부트의 custom한 설정은 실질적인 데이터 바인딩과 관계가 없었다. 즉, **바인딩에 대한 부분은 `ObjectMapper`의 기본 설정값을 사용한다는 것이다.** 기본 설정이 어떻게 되어있는지 알아보자.

### `ObjectMapper` 기본 설정

데이터 바인딩에 관한 `ObjectMapper`의 기본 설정에 대해 알아보자. Jackson은 기본적으로 리플렉션을 이용해 동작한다. [jackson-databind 공식 문서](https://github.com/FasterXML/jackson-databind/wiki/Mapper-Features)의 **Mapper Features**에서 디폴트 설정에 대해 확인할 수 있다. 객체 정보를 어떤 방법으로 가져오고 값을 넣는지에 대한 설정만 주로 살펴보자. (`2.15.2` 버전 기준)

- **AUTO_DETECT_CREATORS** (default: true)
    - 생성자, 정적 팩토리 메서드를 자동으로 찾는다. 자동으로 디텍트되려면 `public`이어야 하고 `String`, `int`, `long`, `boolean`인 argument를 가져야 한다. 팩토리 메서드는 `valueOf`라는 이름을 가지고 있어야 자동으로 디텍트된다.
- **AUTO_DETECT_FIELDS** (default: true)
    - 멤버 변수를 자동으로 디텍트한다.
- **AUTO_DETECT_GETTERS** (default: true)
    - getter를 자동으로 디텍트한다.
- **AUTO_DETECT_IS_GETTERS** (default: true)
    - is_getter를 자동으로 디텍트한다. 예를들어 `boolean isEnabled()`와 같은 메서드
- **AUTO_DETECT_SETTERS** (default: true)
    - setter를 자동으로 디텍트한다.
- **REQUIRE_SETTERS_FOR_GETTERS** (default: false)
    - getter에 대응되는 setter가 필수인가
- **USE_GETTERS_AS_SETTERS** (default: true)
    - 역직렬화 시에 setter메서드 대신 getter메서드로 데이터 입력할 것인지 여부
- **CAN_OVERRIDE_ACCESS_MODIFIERS** (default: true)
    - `AccessibleObject#setAccessible` 메서드를 호출해 접근 제한자를 변경할 수 있다.

위의 기본 설정으로 미루어 보았을 때, **Jackson**은 생성자가 필요하고, getter또는 setter로 데이터를 바인딩 함을 할 수 있다. 기본적으로 setter보다는 getter를 통해 데이터에 접근하고 수정하는 것을 알 수 있다.

즉, **Jackson**이 작동할 수 있도록 최소한으로 DTO를 열어두려면 **생성자**와 **getter**가 필요함을 알 수 있다.

그럼 생성자와 getter의 **접근 제한자**는 어떻게 설정해야할까? `VisibilityChecker` 클래스에 답이 있다.

```java
public interface VisibilityChecker<T extends VisibilityChecker<T>> {  
  public static class Std implements VisibilityChecker<Std>, java.io.Serializable {  
    private static final long serialVersionUID = 1;  
  
      protected final static Std DEFAULT = new Std(  
        Visibility.PUBLIC_ONLY, // getter  
        Visibility.PUBLIC_ONLY, // is-getter  
        Visibility.ANY, // setter  
        Visibility.ANY, // creator
        Visibility.PUBLIC_ONLY // field  
        );  

      // ...
    }  
}
```

`ObjectMapper`의 디폴트 설정인 `Visibility`를 확인할 수 있다. 생성자는 `private`이어도 찾을 수 있고 getter는 `public`까지 볼 수 있는 것이 기본 설정이다. 즉, 데이터 바인딩을 위한 **생성자의 최소 범위는 `private`**이고 **getter의 최소 접근 제한 범위는 `public`**이다.

그렇다면 어떤 생성자를 선언해야할까? **Jackson**의 역직렬화에 관련된 코드를 살펴보면 확인할 수 있다. `BeanDeserializer`의 `deserializeFromObject()` 메서드를 주요한 코드를 중심으로 확인해보자.

```java
@Override
public Object deserializeFromObject(JsonParser p, DeserializationContext ctxt) throws IOException
{

    // 스탠다드 생성이 아닐 때 -> 기본 생성자 사용 X
    if (_nonStandardCreation) {
        // ...
        Object bean = deserializeFromObjectUsingNonDefault(p, ctxt);
        return bean;
    }
    // 스탠다드 생성일 때 -> 기본 생성자 사용
    final Object bean = _valueInstantiator.createUsingDefault(ctxt);
    p.setCurrentValue(bean);
    // ...
    if (p.hasTokenId(JsonTokenId.ID_FIELD_NAME)) {
        String propName = p.currentName();
        do {
            // 프로퍼티 채우기
        } while ((propName = p.nextFieldName()) != null);
    }
    return bean;
}

```

스탠다드 방식의 생성일 때, 기본 생성자를 통해 빈을 생성하고 `do-while`문을 통해 프로퍼티를 채우는 것을 확인할 수 있다. 즉, **Jackson**의 기본적인 빈 생성 방식은 기본 생성자를 이용한다는 의미이다.

그렇다면 기본 생성자가 없는 클래스는 어떻게 역직렬화 시킬까?

실제로 기본 생성자가 없는 클래스로 역직렬화 하려면 특별한 모듈을 설정해주어야 한다. **jackson-module-parameter-names**모듈을 이용하는 것이다. JDK8 이상에서는 컴파일 시에 Reflection API로 파라미터 정보를 가지고 올 수 있도록 컴파일된 클래스에 정보를 추가할 수 있는데 이를 이용할 수 있도록 하는 옵션이다.

클래스 컴파일 시에 `-parameters`옵션을 추가해 클래스에 정보를 추가할 수 있도록 해주면 해당 모듈이 이를 이용해 역직렬화를 진행할 수 있는 것이다.

![](https://i.imgur.com/JhkZISx.png){:.align-center width="500" }

실제로 Spring이 제공하는 `ObjectMapper`의 모듈 정보를 보면 **jackson-module-parameter-names**가 설정되어있는 것을 확인할 수 있다.

![](https://i.imgur.com/U1EZxvp.png){:.align-center width="500" }

`JacksonAutoConfiguration`클래스를 보면 Spring Boot가 `PrameterNamesModule`을 자동 설정하는 것을 확인할 수 있다. `-parameters`는 gradle의 java 플러그인에서 자동으로 설정해준다(관련 코드 : [LINK](https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-tools/spring-boot-gradle-plugin/src/main/java/org/springframework/boot/gradle/plugin/JavaPluginAction.java#L250))따라서 기본 생성자가 존재하지 않는 클래스도 역직렬화 할 수 있게 된다.

하지만 이는 스탠다드한 방법이 아니고 **Jackson**에 대해 잘 알지 못하면 예상치 못하게 역직렬화에 실패할 수 있으니 기본 생성자를 추가하는 것이 예상치 못한 문제를 막는 방법이라고 생각된다.

> 예상치 못한 문제의 예로, 멤버 변수가 하나이고 파라미터가 하나인 생성자만 존재할 때, 이를 Jackson이 이 메서드가 Delegating 생성자인지, Property-based creator인지 구분할 수 없어 이를 명시적으로 해결해 주어야 하는 이슈가 존재한다. ([LINK](https://github.com/fasterxml/jackson-module-parameter-names/issues/21))

## `@RequestBody`, `@ResponseBody`

`MappingJackson2HttpMessageConvertor` 클래스에서 `ObjectMapper`를 통해 직렬화/역직렬화가 이루어 진다.

위에서 알아본 이유에 따라 **역직렬화**를 위해서는 **private 기본 생성자**와 **public getter**가 필요하고, **직렬화**를 위해서는 **public getter**만 필요하다. 하지만 **기본 생성자를 없앨 수 는 없으므로 다음과 같이 private으로 작성**하면 될 것이다.

즉, `@RequestBody`, `@ResponseBody`를 위한 최소의 접근 제한자를 가진 DTO 클래스는 다음과 같이 작성할 수 있다.

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class Dto {
  private String name;
  private int age;
}
```

## `@ModelAttribute`

`@ModelAttribute`는 `ModelAttributeMethodProcessor` 클래스에서 데이터 바인딩이 이루어 진다. `resolveArgument()` 메서드를 따라가보면 동작 방식을 확인할 수 있다. 순서는 다음과 같다.

- 적절한 생성자를 찾아 새 인스턴스 생성
- 클라이언트가 요청한 파라미터를 기준으로 setter 메서드를 찾아서 실행

코드를 보며 좀 더 자세히 알아보자. 생성자를 다음과 같은 방법으로 찾는다. `BeanUtils`의 `getResolvableContructor()`메서드를 살펴보자.

```java
public static <T> Constructor<T> getResolvableConstructor(Class<T> clazz) {
    Constructor<T> ctor = findPrimaryConstructor(clazz);
    if (ctor != null) {
        return ctor;
    }

    Constructor<?>[] ctors = clazz.getConstructors();
    if (ctors.length == 1) {
        // A single public constructor
        return (Constructor<T>) ctors[0];
    }
    else if (ctors.length == 0) {
        // No public constructors -> check non-public
        ctors = clazz.getDeclaredConstructors();
        if (ctors.length == 1) {
            // A single non-public constructor, e.g. from a non-public record type
            return (Constructor<T>) ctors[0];
        }
    }

    // Several constructors -> let's try to take the default constructor
    try {
        return clazz.getDeclaredConstructor();
    }
    catch (NoSuchMethodException ex) {
        // Giving up...
    }

    // No unique constructor at all
    throw new IllegalStateException("No primary or single unique constructor found for " + clazz);
}
```

가장 먼저 public으로 선언된 생성자를 찾는다. public 생성자가 없다면 public이 아닌 기본 생성자를 먼저 찾고 다른 생성자가 있다면 매개변수가 가장 적은 생성자를 선택한다. 즉, 어떤 접근 제한자의 생성자가 존재하더라도 `@ModelAttribute`를 위한 객체를 생성하는 것에 문제가 없다.

이제는 값을 채우는 로직을 확인해 보자.

![](https://i.imgur.com/mws5Us7.png)

`resolveArgument()` 메서드에서 `bindRequestParameters()`메서드를 따라가보자.

![](https://i.imgur.com/v0pIcG3.png)

따라가다 보면 `AbstractPropertyAccessor`의 `setPropertyValues()`메서드를 확인해보면 `propertyValues`에 클라이언트의 요청에서 값을 가져와 바인딩을 시도하는 것을 볼 수 있다. 이때 클래스의 setter메서드가 호출된다.


`@ModelAttribute`는 setter와 생성자를 통해 데이터 바인딩이 진행되므로 다음과 같이 DTO를 작성하면 데이터 바인딩이 가능하면서, 최소한의 접근 제한자를 가지는 클래스로 작성할 수 있다.

```java
@Setter
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class Dto {
  private String name;
  private int age;
}
```

위는 **데이터 바인딩이 가능한 최소의 클래스**이다. 하지만, 아직은 DTO와 엔티티를 변환시킬 방법을 고려하지 않았다. DTO와 엔티티 사이를 어떻게 변환 시킬것인지, 테스트를 진행할 것인지에 따라 **필요한 생성자나, getter, setter등은 바뀔 수 있다**. 어떤 방식으로 변환할 것인지 고려해보고 최소 접근 권한을 가진 DTO를 다시 고려해보자.

## DTO의 변환 방법

DTO는 데이터를 전달하기 위한 객체이기 때문에 데이터 매핑은 DTO의 숙명이다. 데이터 변환을 위한 방법으로는 크게 세가지 방법을 생각해 볼 수 있을 것 같다.

- **변환 메서드 수동 작성**
- **ModelMapper 라이브러리 사용**
- **MapStruct 라이브러리 사용**

**ModelMapper**는 동적으로 Reflection API를 사용해 객체를 매핑하기 때문에 디버깅이 어렵고 성능 오버헤드가 크다고 한다. 또한 setter를 사용하기 때문에 setter를 강제해야하는 특성이 있다. 이런 이유로 **ModelMapper**는 사용 후보에서 제외하겠다.

**변환 메서드를 직접 작성**하는 방식의 단점은 **boilerplate 코드의 양이 기하급수적으로 늘어난다**는 것이다. 만약 엔티티에 멤버 변수를 추가해야 한다면 많은 변환 메서드를 수정해야할 수도 있을 것이다.

**MapStruct**를 사용하면 애노테이션을 통해 간단하게 데이터 매핑 클래스를 만들 수 있다. 컴파일 시점에 코드를 생성하기 때문에 직접 메서드를 작성하는 것과 같이 **타입 안정성과 성능을 보장**할 수 있다. 변환 메서드 수동 작성에서의 **boilerplate 코드를 줄일 수 있고 실수의 발생 가능성도 줄일 수 있다**.

따라서 간단한 프로젝트이거나 유지보수가 크게 필요하지 않은 프로젝트인 경우엔 직접 변환 메서드를 작성하고, 나머지 경우엔 **MapStruct**를 쓰는 것이 유지보수에 유리할 것 같다. **MapStruct**의 간단한 사용 방법을 알아보자.

### MapStruct 라이브러리 추가

```groovy
// lombok
compileOnly 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok'

// mapstruct
implementation 'org.mapstruct:mapstruct:1.5.5.Final'
annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'
annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
```

가장 먼저 `build.gradle`에 [mapstruct 공식 문서](https://mapstruct.org/documentation/stable/reference/html/#lombok)를 참고하여 위와 같이 설정해 의존성을 추가한다. 순서를 지키지 않으면 정상적으로 작동하지 않는 이슈가 있다고 아니 순서를 지켜 작성하자.

![](https://i.imgur.com/6VdOmx7.png)

롬복을 통해 빌더를 생성하면 **Mapstruct**와 충돌이 생기는데 이를 처리하기위해 추가해주어야 할 디펜던시가 `lombok-mapstruct-binding`이다. [lombok changelog](https://projectlombok.org/changelog)에서 위와 같이 확인할 수 있다.

### MapStruct에 필요한 생성자/메서드

**MapStruct**는 기본적으로 source객체의 getter를 이용해 데이터를 가져오기 때문에 **source 객체에 getter는 필수**이다.

**Mapstruct**는 타겟 객체를 생성할 때, 타겟 클래스에 빌더가 정의되어 있다면 빌더를 생성자보다 우선적으로 사용한다. 생성자를 활용할 것이라면 `@AllArgsConstructor` 를 지정해서 모든 속성을 포함하는 생성자를 만들어야 하고 `@NoArgsConstructor` 로 기본생성자만 만들 생각이라면 `@Setter`로 setter 메소드를 반드시 포함시켜 줘야 한다.

- **builder** (1순위)
- **전체 생성자** (2순위)
- **기본 생성자 + setter** (3순위)

setter는 최대한 지양해야 한다고 생각한다. 따라서 생성자나 builder를 이용할 것이다. 따라서 엔티티에서 DTO로 변환 될 target 객체인 `@ResponseBody`로 사용될 DTO에는 `@AllArgsContructor`를 추가해준다. builder를 사용하고 싶다면 여기에 `@Builder`를 추가한다.

Mapstruct [공식 문서](https://mapstruct.org)의 예를 기반으로 엔티티와 Request DTO, Response DTO를 작성해보자.

```java
@Getter  
@NoArgsConstructor(access = AccessLevel.PROTECTED)  
@Entity  
public class Car {  
  
  @Id  
  @GeneratedValue(strategy = GenerationType.IDENTITY)  
  private Long id;  
  
  private String name;  
  
  private int numberOfSeats;  
  
  private CarType type;  
  
  @Builder  
  public Car(String name, int numberOfSeats, CarType type) {  
    this.name = name;  
    this.numberOfSeats = numberOfSeats;  
    this.type = type;  
  }  
}
  
@Getter  
@Builder  
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PRIVATE)  
public class CarRequest {  
  
  private String name;  
  
  private int seatCount;  
  
  private String type;  
  
}

@Getter
@Builder
@AllArgsConstructor
public class CarResponse {

  private String name;

  private int seatCount;

  private String type;

}
```

`CarRequest`는 `@RequestBody`에서 데이터 바인딩이 잘 될 것이다. `@RequestBoody`에서의 데이터 바인딩을 위해서는 **private 기본 생성자**와 **getter**만 필요하기 때문이다. (`@ModelAttribute`에 사용될 것이라면 `@Setter`를 추가)

`CarRequest`는 Mapstruct에서 엔티티로 변환되는 source 객체로 사용되므로 생성자는 필요없고 getter만 필요하다. 따라서 데이터 바인딩과 매핑을 위해서 private 기본 생성자와 getter면 충분할 것이다.

위의 최소 조건을 갖추면 애플리케이션이 돌아가는데는 문제가 없다. 하지만 DTO를 생성할 수 없어 **테스트가 어렵다**는 단점이 있다. 따라서 **테스트를 위해 `@Builder`를 추가했다.**

`CarResponse`는 `@ResponseBody`에서 데이터 바인딩이 될 것이다. 바인딩을 위해서는 getter만 필요하다. Mapstruct가 `CarResponse`를 생성하기 위해서는 builder를 선택했다.

이제 엔티티와 DTO를 매핑해주는 간단한 매퍼를 만들어보자. 만약 특별한 로직이나 설정이 필요한 경우에는 Mapstruct의 공식 문서를 참고하여 설정하자.([LINK](https://mapstruct.org/documentation/stable/reference/html/))

```java
@Mapper(componentModel = "spring")
public interface CarMapper {

  @Mapping(source = "numberOfSeats", target = "seatCount")
  CarResponse toCarResponse(Car car);

  @Mapping(source = "seatCount", target = "numberOfSeats")
  Car toCar(CarRequest carRequest);
}

```

위와 같이 인터페이스를 선언하면 컴파일 시 `CarMapperImpl`클래스가 generated 디렉토리 안에 생기게 된다. 실제 클래스는 다음과 같이 생성된다.

```java
@Generated(  
    value = "org.mapstruct.ap.MappingProcessor",  
    date = "2023-08-09T18:41:17+0900",  
    comments = "version: 1.5.5.Final, compiler: javac, environment: Java 17.0.5 (Eclipse Adoptium)"  
)  
@Component  
public class CarMapperImpl implements CarMapper {  
  
    @Override  
    public CarResponse toCarResponse(Car car) {  
        if ( car == null ) {  
            return null;  
        }  
  
        CarResponse.CarResponseBuilder carResponse = CarResponse.builder();  
  
        carResponse.seatCount( car.getNumberOfSeats() );  
        carResponse.name( car.getName() );  
        if ( car.getType() != null ) {  
            carResponse.type( car.getType().name() );  
        }  
  
        return carResponse.build();  
    }  
  
    @Override  
    public Car toCar(CarRequest carRequest) {  
        if ( carRequest == null ) {  
            return null;  
        }  
  
        Car.CarBuilder car = Car.builder();  
  
        car.numberOfSeats( carRequest.getSeatCount() );  
        car.name( carRequest.getName() );  
        if ( carRequest.getType() != null ) {  
            car.type( Enum.valueOf( CarType.class, carRequest.getType() ) );  
        }  
  
        return car.build();  
    }  
}
```

실제로 위와 같은 클래스가 생성되고 **문제가 있다면 컴파일 오류가 발생하**기 떄문에 문제를 빠르게 찾아낼 수 있다. 아래와 같이 테스트 클래스를 작성할 수 있다.

```java
@SpringBootTest
public class CarMapperTest {

  @Autowired
  CarMapper carMapper;

  @Test
  public void shouldMapCarToResponse() {
    //given
    Car car = Car.builder()
        .name("Morris")
        .numberOfSeats(5)
        .type(CarType.SEDAN)
        .build();

    //when
    CarResponse carResponse = carMapper.toCarResponse(car);

    //then
    assertThat(carResponse).isNotNull();
    assertThat(carResponse.getName()).isEqualTo(car.getName());
    assertThat(carResponse.getSeatCount()).isEqualTo(car.getNumberOfSeats());
    assertThat(carResponse.getType()).isEqualTo(car.getType().name());
  }

  @Test
  public void shouldMapRequestToCar() {
    //given
    CarRequest carRequest = CarRequest.builder()
        .name("Morris")
        .seatCount(5)
        .type("SEDAN")
        .build();

    //when
    Car car = carMapper.toCar(carRequest);

    //then
    assertThat(car).isNotNull();
    assertThat(car.getName()).isEqualTo(carRequest.getName());
    assertThat(car.getNumberOfSeats()).isEqualTo(carRequest.getSeatCount());
    assertThat(car.getType()).isEqualTo(CarType.valueOf(carRequest.getType()));
  }
}


```

## 최종적인 DTO의 작성 방법

이제 스프링의 데이터 바인딩 방법, 데이터 매핑 방법, 테스트 편의성을 고려해서 잘 작동하는 최소의 **DTO**의 작성 방법을 정리해보자.

### `@RequestBody`

```java
@Getter // @RequesteBody를 위해 추가, Mapstruct를 위해 추가, 로직/테스트를 위해 추가
@Builder // (선택) 테스트를 위해 추가
@AllArgsConstructor // @Builder를 위해 추가
@NoArgsConstructor(access = AccessLevel.PRIVATE) // @RequestBody를 위해 추가
public class SomeRequest {
    private String name;
    private int value;
}
```

- `@NoArgsConstructor` : Jackson은 Standard Creation으로 기본 생성자를 사용한다. 이 때, 리플렉션 API를 사용하고, 생성자의 디폴트 Visibility는 `ALL`이다. 기본 생성자가 없다면 스탠다드한 방법을 사용하지 않아 예기치 못한 문제가 발생할 수 있다. Standart Creation을 사용하기 위해 기본 생성자를 선언해주고, 접근 권한을 최소화 하기 위해 private으로 설정한다.
- `@AllArgsConstructor` : 기본 생성자가 선언되었을 때, `@Builder`를 사용하려면 전체 생성자를 선언해 주어야 한다.
- `@Builder` : 테스트를 편하게 하기 위해 추가했다. **`@Builder` 사용은 선택이다.** `@Builder`가 없어도 테스트에서 전체 생성자를 사용하면 된다. 그저 편한 코딩을 위해 추가했다.
- `@Getter` : Jackson에서 getter의 디폴트 Visibility는 `PUBLIC`이다. 따라서 public으로 getter를 선언했다. 또한 Mapstruct에서 getter를 사용하기 때문에 추가하였다. 또한 로직과 테스트에서 값을 편하게 사용하기 위해 사용했다.

### `@ResponseBody`

```java
@Getter // @ResponseBody 위해 추가, 로직/테스트를 위해 추가
@Builder // (선택) Mapstruct를 위해 추가, 테스트를 위해 추가
@AllArgsConstructor // @Builder를 위해 추가
public class SomeResponse {
    private String name;
    private int value;
}
```

- `@AllArgsConstructor` : 기본 생성자가 선언되었을 때, `@Builder`를 사용하려면 전체 생성자를 선언해 주어야 한다.
- `@Builder` : target 객체 이기 때문에 Mapstruct에서 객체 생성을 위해 필요하다. 또한 테스트를 편하게 하기 위해 추가했다. **`@Builder` 사용은 선택이다.** `@Builder`가 없어도 Mapstruct는 `@AllArgsConstructor`를 이용해 코드를 생성하고, 테스트에서도 전체 생성자를 사용하면 된다. 그저 가독성과 편한 코딩을 위해 추가했다.
- `@Getter` : Jackson에서 getter의 디폴트 Visibility는 `PUBLIC`이다. 따라서 public으로 getter를 선언했다. 또한 로직과 테스트에서 값을 편하게 가져오기 위해 사용했다.

### `@ModelAttribute`

```java
@Getter // Mapstruct를 위해 추가, 로직/테스트를 위해 추가
@Setter // @ModelAttribute를 위해 추가
@Builder // (선택) Mapstruct를 위해 추가, 테스트를 위해 추가
@AllArgsConstructor // @ModelAttribute를 위해 추가, @Builder를 위해 추가
public class SomeResponse {
    private String name;
    private int value;
}
```

- `@AllArgsConstructor`, `@Builder` : `ModelAttributeMethodProcessor`에서는 가장 먼저 public 생성자를 찾아 객체를 생성한다. public 생성자가 존재하지 않는다면 non-public 생성자를 찾아 객체를 생성한다. 만약 해당 클래스를 Mapstruct를 통해 변경하지 않고, 테스트를 하지 않을 예정이라면 기본 생성자를 `PRIVATE`로 선언하는 것이 더 나은 방법일 것이다.
- `@Setter` : `ModelAttributeMethodProcessor`는 setter를 통해 데이터를 바인딩하기 때문에 추가하였다.
- `@Getter` : Mapstruct에서 source객체의 정보를 가져오기 위해 getter를 사용하기 때문에 추가하였다. 또한 로직과 테스트에서 값을 편하게 사용하기 위해 사용했다.

> **위의 모든 DTO들은 Mapstruct로 엔티티로 변환을 한다는 가정 하에 작성되었다. 만약 변환 메서드를 수동으로 작성하기로 했거나, 혹은 변환이 필요 없는 경우도 있을 것이다. 그렇다면 위의 설명에 따라 필요한 부분만 제거하거나 추가하면 상황에 맞는 적절한 클래스를 작성할 수 있을 것이다. 또한 엔티티로의 변환 외에도 외부 요청을 위한 DTO인 경우에도 적절하게 최소 접근 권한의 DTO클래스를 작성할 수 있을 것이다.**
