---
layout: single
title: "[티어리스트] 트러블슈팅 - ControllerAdvice 예외 핸들링 우선순위"
categories: spring
tag: [spring, exceptionhandler, controlleradvice]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/r38jcdy.png"
# search: false
---

![](https://i.imgur.com/r38jcdy.png)

## 문제 상황

티어리스트 프로젝트에서는 `@RestControllerAdvice`를 이용해 예외응답을 생성하고 있다. 스프링 프레임워크가 던지는 예외를 핸들링하는 `GlobalExceptionHandler`와 도메인 레이어에서 던지는 예외를 핸들링하는 `BusinessExceptionHandler`를 Bean으로 등록해 사용하고 있다.

사용자는 중복된 이메일과 중복된 닉네임으로 가입할 수 없기 때문에 중복된 이메일이나 중복된 닉네임으로 가입을 시도할 시 `EmailDuplicationException`이나 `NicknameDuplicationException`을 던진다.

이를 `BusinessExceptionHandler`에서 받아서 다음과 같이 처리한다.

```java
@Slf4j  
@RestControllerAdvice  
public class BusinessExceptionHandler {  

  // 생략
  
  @ExceptionHandler(DuplicationException.class)  
  public ResponseEntity<ErrorResponse> handleDuplicationException(  
      DuplicationException duplicationException) {  
    ErrorCode errorCode = duplicationException.getErrorCode();  
    return ResponseEntity.status(HttpStatus.CONFLICT)
        .body(ErrorResponse.from(errorCode));  
  }  

  // 생략
  
}
```

Local에서 Postman으로 이메일 중복 검사 API에 중복된 이메일로 요청하면 다음과 같이 예상한 응답이 나왔다.

![](https://i.imgur.com/hkyC2uk.png)

하지만 개발 서버에 배포된 쿠버네티스 파드에 같은 요청을 보냈을 때 `GlobalExceptionHandler`에서 처리됨을 확인할 수 있었다.

```java
@Slf4j  
@RestControllerAdvice  
public class GlobalExceptionHandler {  

  // 생략
  
  @ExceptionHandler(Throwable.class)  
  public ResponseEntity<ErrorResponse> handleThrowable(Throwable e) {  
    log.error(e.getMessage(), e);  
    final ErrorCode errorCode = ErrorCode.INTERNAL_SERVER_ERROR;  
    return ResponseEntity.internalServerError()
        .body(ErrorResponse.from(errorCode));  
  }  
  
}
```

응답은 다음과 같다.

![](https://i.imgur.com/yWq10QG.png)

같은 코드임에도 불구하고 어디서 작동되냐에 따라 예외 핸들링 클래스가 달라지는 것은 이상하게 느껴졌다.

## 문제 원인

### ControllerAdvice 선택

`ControllerAdvice`의 선택 과정을 디버깅 해보자. 어떤 핸들러를 선택할 것인지는 `ExceptionHandlerExceptionResolver`의 `getExceptionHandlerMethod`메서드 호출 과정에서 결정된다.

```java
@Nullable  
protected ServletInvocableHandlerMethod getExceptionHandlerMethod(  
   @Nullable HandlerMethod handlerMethod, Exception exception) {  

  // 생략

  for (Map.Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry : this.exceptionHandlerAdviceCache.entrySet()) {  
   ControllerAdviceBean advice = entry.getKey();  
   if (advice.isApplicableToBeanType(handlerType)) {  
    ExceptionHandlerMethodResolver resolver = entry.getValue();  
    Method method = resolver.resolveMethod(exception);  
    if (method != null) {  
     return new ServletInvocableHandlerMethod(advice.resolveBean(), method, this.applicationContext);  
    }  
   }  
  }  
  
  return null;  
}
```


`exceptionHandlerAdviceCache`의 `entrySet`에서 등록된 `ControllerAdvice`를 하나씩 꺼내와 순서대로 탐색한다. 던져진 예외에 일치 핸들러가 존재하는지 확인한 후 `ExceptionHandler`의 메서드를 반환한다.

즉, 처음 만나는 일치하는 핸들러의 메서드를 반환하게 된다.

![](https://i.imgur.com/JIyuR0C.png)

`exceptionHandlerAdviceCache`는 `LinkedHashMap`으로 구현되어 있다. `LinkedHashMap`은 보통 다른 설정이 없다면 기본 설정으로 삽입되는 `Key`값의 순서를 기반으로 정렬된다.

개발 서버에서는 `GlobalExceptionHandler`가 먼저 `exceptionHandlerAdviceCache`에 등록되었다고 추측할 수 있다.

### ExceptionHandler 등록 순서

`exceptionHandlerAdviceCache`에 어드바이스들이 등록되는 순서를 살펴보자.

![](https://i.imgur.com/r5nu6ru.png)

`ExceptionHandlerExceptionResolver`의 `initExceptionHandlerAdviceCache`메서드를 살펴보면 `exceptionHandlerAdviceCache`를 초기화 함을 알 수 있다.

`ControllerAdviceBean.findAnnotatedBeans(getApplicationContext())`에서 어드바이스 빈을 가져와 차례로 등록시킨다.

해당 메서드를 살펴보자.

![](https://i.imgur.com/s0qh68o.png)

`BeanFactoryUtils`의 메서드를 통해 빈들을 가져온 후 마지막에 `OrderComparator`를 통해 해당 빈들을 정렬한다. `BeanFactoryUtils`의 메서드는 순서를 보장하지 않는다. **즉, 원하는 `ControllerAdvice` 간의 적용 순서를 보장하려면 `@Order` 애노테이션을 통해 순서를 강제해야한다.**

> 로컬과 개발 서버의 유일한 설정 차이는 다음과 같다. 배포중인 이미지는 GitHub Action의 DockerX action을 통해 빌드가 이루어진다. 아마도 이 부분에서 차이가 생겼을 것이라고 추측된다.

## 문제 해결

위의 디버깅 결과로 `@Order` 애노테이션을 통해 `ControllerAdvice`간의 순서를 보장해 주어야 함을 알았다.

새로운 `ControllerAdvice`를 추가해야할 때 이를 누락하는 실수를 할 수 있기 때문에 커스텀한 `ControllerAdvice` 애노테이션을 만들어 사용해보자.

### 커스텀 @ControllerAdvice 애노테이션 구현

`@GlobalControllerAdvice`와 `@BusinessControllerAdvice`를 다음과 같이 구현했다.

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Order(100)
@RestControllerAdvice
public @interface GlobalControllerAdvice {
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Order(Ordered.HIGHEST_PRECEDENCE)
@RestControllerAdvice
public @interface BusinessControllerAdvice {

}
```


`@GlobalControllerAdvice`에 낮은 우선순위를 부여하고, `@BusinessControllerAdvice`에 가장 높은 우선순위를 부여했다.

앞으로 추가될 `ControllerAdvice`에는 적당한 우선순위를 부여해주면 될 것이다. 이 두 애노테이션을 기존 `ControllerAdvice`에 적용했다.

## 결과

트러블슈팅 결과 환경에 따라 예외 응답 처리 클래스가 달라지지 않았고 우선순위가 잘 적용됨을 확인했다.

![](https://i.imgur.com/0Iubysg.png)
