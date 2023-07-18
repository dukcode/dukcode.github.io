---
layout: single
title: "[개발고민] Spring Custom Exception과 예외 처리 전략에 관한 고민"
categories: spring
tag: [개발고민, spring, spring-exception, custom-exception]
toc: true
toc_sticky: true
author_profile: true
sidebar:
    nav: "docs"
header:
  teaser: "https://i.imgur.com/iYc6wqg.png"
# search: false
---

  💡 **개발고민**은 개발을 공부하며 했던 저의 생각들 입니다. 정답이 아니며 정답을 찾아가는 과정이라고 봐주시면 감사하겠습니다. [Github Repository](https://github.com/dukcode/boilerplate)<br>
  {: .notice--info} 
  
# Spring Custom Exception과 예외 처리 전략

![](https://i.imgur.com/iYc6wqg.png){:.align-center width="500" }

프로젝트를 진행하면서 팀원들끼리 예외 처리에 대한 기준이 서로 다르다 보니 코드의 일관성이 떨어졌다.

어떤 팀원은 **Standard Exception**을 통해 예외를 발생시키고, 어떤 팀원은 **Custom Exception**을 생성해 이를 통해 예외를 발생시켰다. 또한 던져진 예외의 처리 방식도 달랐다. Validation에 대해서 어떤 팀원은 Controller의 메서드의 파라미터로 `BindingResult`를 받아 에러를 처리하고자 했고, 어떤 팀원은 `@ControllerAdvice`로 처리하고자 했다.

이는 다른 사람이 짠 코드를 볼때 한번에 **이해하기 쉽지 않았고**, 이에 따라 **코드의 유지보수성**도 떨어질게 뻔했다. 적절한 예외 생성 전략과 예외 처리 전략이 필요했고 이에 대해 고민해 보았다.

## Custom Exception의 필요성

**Effective Java**에서는 다음과 같은 이유로 **Standard Exception의 사용을 권장**한다.

- 우리의 API가 다른 사람이 익히고 사용하기 쉬워진다.
- 예외를 재사용하므로써 예외 클래스 수가 줄어들고 그에 따라 메모리의 사용량, 클래스를 적재하는 시간도 적게 든다.

하지만 다음과 같은 이유라면 **표준 예외를 확장해도 좋다**고 한다.

- 표준 예외가 **이름뿐만 아니라 예외가 던져지는 맥락도 부합하지 않다면** 표준 예외를 확장하라.
- **더 많은 정보를 제공**하기 원한다면 표준 예외를 확장해도 좋다.

우리가 Spring MVC Application을 만들 때, 주로 예외가 발생하는 맥락과 특별히 제공해야 하는 정보가 있는지 생각해보자.

### 예외의 맥락

**회원 가입 서비스**를 생각해 보자. 사용자는 생성될 이메일과 닉네임, 패스워드를 입력하고 우리의 서비스는 이를 처리한다. **아이디와 닉네임은 중복되면 안된다**는 요구사항이 있다고 가정하자. 만약 이메일과 닉네임이 중복된다면 우리는 어떤 표준 예외 던지는 것이 적절할까?

| 표준 예외                       | 주요 쓰임                             |
| -------------------------- | ------------------------------------- |
| `IllegalArgumentException` | 허용하지 않는 값이 인수로 건네졌을 때 |
| `IllegalStateException`    | 객체가 메서드를 수행하기에 적절하지 않은 상태일 때                                      |

아이디와 닉네임이 중복되었을 때 위의 두 표준 예외가 모두 해당될 수 있다고 본다. 즉, 개발자가 맥락을 어떻게 보느냐에 따라 `IllegalArgumentException`과 `IllegalStateException` **모두 선택될 수 있다고 본다.**

이는 전체적인 **코드의 일관성**을 떨어뜨린다. 만약 **Custom Exception을 생성해 처리**한다면 어떨까?

```java

public MemberSignupResponse createMember(MemberSignupRequest request) {
    if (memberRepository.exsitsByEmail(request.getEmail())) {
        throw new EmailDuplicateException();
    }

    if (memberRepository.exsitsByNickname(request.getNickname())) {
        throw new NicknameDuplicateException();
    }

    Member member = memberRepository.save(request.toEntity());
    return new MemberSignupResponse(member);
}
```

던져지는 **예외의 이름이 맥락과 정확하게 일치**해 코드의 **명확성**과 **가독성**이 높아지는 것을 확인할 수 있다. 개발자도 **선택의 여지 없이** 해당 맥락과 정확하게 일치하는 예외를 생성해 던지면 된다. **코드의 일관성**이 높아지는 것이다.

### 추가적인 정보

**표준 예외를 이용하면서도 가독성과 일관성을 높일 수 있는 방법**도 고민해 보았다. 팀원끼리 **하나의 예외를 골라 쓰도록** 하고 **메시지를 상수로 관리**해 명확성을 높이는 방법이다.

```java
public MemberSignupResponse createMember(MemberSignupRequest request) {
    if (memberRepository.exsitsByEmail(request.getEmail())) {
        throw new IllegalArgumentException(ErrorMessage.EMAIL_DUPLICATE);
    }
    
    if (memberRepository.exsitsByNickname(request.getNickname())) {
        throw new IllegalArgumentException(ErrorMessage.NICKNAME_DUPLICATE);
    }
    
    // ...
}
```

만약 **예외에 대한 추가적인 정보**를 받고 싶다면 어떨까? 위의 상황에서는 중복된 아이디나 중복된 닉네임이 무엇인지에 대해 정보를 받고 싶다고 가정하자.

```java
throw new IllegalStateException(ErrorMessage.EMAIL_DUPLICATE + " " + request.getEmail());
```

**표준 에러**를 사용한다면 **메시지를 받는 생성자**를 이용할 수 밖에 없다. 이에 따라 추가적인 **정보가 누락될 가능성**이 있고, 그에 따라 코드의 일관성이 떨어질 가능성이 있다. 즉, 추가적인 정보를 **생성자를 통해 강제**할 수 없다. 

또한 **테스트**의 경우에도 이메일 중복, 닉네임 중복에도 같은 예외가 던져지기 때문에 **메시지를 비교**해야 한다. 이 또한 비용이라고 생각된다.

결론은 이렇다. 예외 처리에서의 코드 명확성을 높이자. **Standard Exception**과 **메세지 상수**의 조합은 코드 명확성을 높일 수는 있지만 **생성자를 통한 추가 정보의 강제가 불가능** 하므로 코드의 일관성이 부족해 질 수 있다. 따라서 **Custom Exception**을 사용하는 것이 코드의 명확성과 일관성을 높이는데 유리하다.

## 일관된 Custom Exception 구현

예외를 던질때 외에도 Custom Exception을 **추가**할 때도 일관성을 유지하는 것이 필요했다. 서로 예외의 **구현 방식**이 다르다면 개발 비용도 올라갈 것이고 코드의 일관성도 떨어질 것이다.

일관된 Custom Exception 구현 방식을 따르기 위해 상속을 통해 예외의 **계층 구조**를 사용할 것이다. 최상위의 `BusinessException`을 먼저 구현해 보자. `RuntimeException`을 상속하며, `ErrorCode`를 멤버로 가지고 있다.

```java
@Getter  
public class BusinessException extends RuntimeException {  
  
  private final ErrorCode errorCode;  
  
  public BusinessException(String message, ErrorCode errorCode) {  
    super(message);  
    this.errorCode = errorCode;  
  }  
  
  @Override  
  public synchronized Throwable fillInStackTrace() {  
    return this;  
  }  
}
```

특이한 점은 `Trowable`의 `fillInStackTrace()` 메서드를 **오버라이딩** 했다는 것이다. 보통 서비스에서 예외를 발생시키는 경우는 예외의 추적보다는 유효하지 않는 값을 처리하려고 할 때 하위의 서비스 로직을 수행하지 못하기 위한 용도로 주로 사용된다. 따라서 일반적인 경우에는 **Stack Trace**가 불필요해 보인다.

예외 생성 비용은 비싸다. 성능에 영향을 미치는 큰 요소로는 **Stack Trace의 경로**가 있다고 한다. 특히 Stack의 Depth 10마다 **4000ns**가 들기 때문에 일반적으로 예외 생성에 1~5ms 가 소비된다. 이를 오버라이딩을 통해 해결하면 예외 생성에 **80ns**정도로 **성능을 향상**시킬 수 있다. ([참고](https://meetup.nhncloud.com/posts/47))

![](https://i.imgur.com/wSMfu5E.png){:.align-center }

이제 서비스에서 발생할 수 있는 **예외의 범주**를 나누는 작업이 필요하다. 일반적으로 서비스 시 나타날 수 있는 예외의 경우는 **중복 값이 허용되지 않을 때**, **없는 객체를 요청하는 경우**, **만료된 값을 요청**하는 경우 등이 있을 수 있다. 이는 서비스의 특성에 따라 다르므로 예외의 범주를 잘 나누는 작업이 필요하다.

예외의 범주를 나누었다면 `BussinessException`을 상속해 구현한다. 중복 불가능한 값을 요청에 포함하는 경우를 예외로 들어보겠다.

```java
@Getter
public class DuplicateException extends BusinessException {

  private String value;

  public DuplicateException(String value) {
    this(value, ErrorCode.DUPLICATE);
    this.value = value;
  }

  public DuplicateException(String value, ErrorCode errorCode) {
    super(value, errorCode);
    this.value = value;
  }
}

```

중복값인 `value`를 받아서 응답에 추가할 수 있도록 멤버 변수를 추가했다.

실제 서비스에서는 구체적인 상황인 이메일 중복, 닉네임 중복 등이 존재할 것이다. 더 구체적인 예외는 위의 클래스를 상속해서 구현한다. `BussinessException`를 직접 상속하지 않고 **중간 계층**을 두는 이유는 **구체적인 예외**들이 **유사한 방식으로 구현**되도록 하기 위해서다.

```java
public class EmailDuplicateException extends DuplicateException {  
  
  public EmailDuplicateException(final String email) {  
    super(email, ErrorCode.EMAIL_DUPLICATE);  
  }  
}

public class NicknameDuplicateException extends DuplicateException {  
  
  public NicknameDuplicateException(final String nickname) {  
    super(nickname, ErrorCode.NICKNAME_DUPLICATE);  
  }  
}
```

예외의 범주를 나누고 **중간 계층**을 두었기 때문에, 특정한 상황에 맞는 구체적인 예외 클래스들을 **쉽고 유사한 방식**으로 구현할 수 있게되었다. 중복에 관한 다른 예외 클래스를 추가해야 할 때도 **개발에 드는 비용**을 줄일 수 있게 되었다.

## 일관된 응답 구조

클라이언트에서 동일한 로직으로 예외 처리를 하려면 일관된 응답 구조가 필요하다. 또한 클라이언트에서 **메세지를 받아 예외를 처리**하는 것은 **에러 메세지의 수정**을 어렵게 만든다. 따라서 Http Status보다 **세부적인 정보**를 제공하기 위해 에러 코드를 **문서화**해 관리하는 것이 유리할 것이다.

### `ErrorResponse`

```json
{
    "status": 409,
    "code": "M-001",
    "message": "Duplicate Email Address",
    "values": ["duk9741@gmail.com"]
}
```

- **status** : 기본적으로 Http Status와 동일 값을 가진다. status는 클라이언트와 서버의 약속이기 때문에 특별한 값을 추가할 수 있을 것이다.
- **code** : 클라이언트가 서버와 약속된 코드를 통해 상황에 따른 예외처리를 할 수 있게 한다. 해당 코드는 문서화를 통해 커뮤니케이션의 오류 없이 클라이언트 개발자에게 전달되야 할 것이다.
- **message** : 클라이언트의 개발자가 상황을 이해할 수 있도록 돕는 메세지를 전송한다.
- **values** : 예외가 생성된 관련 값을 나타낸다. 예외가 생성된 이유가 여러가지 일 수 있기 때문에 배열로 값을 전달한다. 값이 존재하지 않으면 `null`값이 아닌 빈 배열을 전달하도록 하였다.

에러에 대한 응답을 위와 같이 동일하게 처리하기 위해 다음과 같은 객체를 구현한다. 추가적인 구현은 [코드 링크](https://github.com/dukcode/boilerplate/blob/main/src/main/java/com/dukcode/boilerplate/global/error/ErrorResponse.java)에서 확인할 수 있다.

```java
@Getter  
@NoArgsConstructor(access = AccessLevel.PROTECTED)  
public class ErrorResponse {  
  
  private int status;  
  private String code;  
  private String message;  
  private List<String> values = new ArrayList<>();  
  
  private ErrorResponse(final int status, final ErrorCode code) {  
    this.status = status;  
    this.message = code.getMessage();  
    this.code = code.getCode();  
  }  

  // ...

}
```

### `ErrorCode`

에러 코드와 기본적인 에러 메시지를 함께 관리하는 `ErrorCode`를 구현하자.

```java
@Getter
public enum ErrorCode {

  // Common
  INVALID_INPUT_VALUE("C-001", "Invalid Input Value"),
  METHOD_NOT_ALLOWED("C-002", "Method Not Allowed"),
  INTERNAL_SERVER_ERROR("C-004", "Server Error"),
  INVALID_TYPE_VALUE("C-005", "Invalid Type Value"),
  HANDLE_ACCESS_DENIED("C-006", "Access is Denied"),

  // Business
  DUPLICATE("B-001", "Duplicate Value"),
  NOT_FOUND("B-002", "Entity Not Found"),

  // Member
  EMAIL_DUPLICATE("M-001", "Duplicate Email Address"),
  NICKNAME_DUPLICATE("M-002", "Duplicate Nickname");

  // ...

  private final String code;
  private final String message;

  ErrorCode(final String code, final String message) {
    this.code = code;
    this.message = message;
  }
}
```

일반적으로 **Http Status**를 `ErrorCode`를 추가하는 방식이 많은 것 같다. 하지만 이는 **중복**이 많아지기에 예외 계층 구조와 `@ControllerAdvice`의 메서드를 통해 **일관되게 처리**하도록 한다.

### `@ControllerAdivce`

`@ControllerAdvice`로 모든 예외를 한 곳에서 처리할 수 있도록 한다. 메서드의 기본적인 동작구조는 예외를 파라미터로 전달받아 `ErrorResponse`의 값을 채워 반환하는 것이다. 우리가 만든 Custom Exception 외에도 스프링에서 일어날 수 있는 예외도 처리할 수 있게 구현했다. 추가적인 코드는 [코드 링크](https://github.com/dukcode/boilerplate/blob/main/src/main/java/com/dukcode/boilerplate/global/error/GlobalExceptionHandler.java)에서 확인할 수 있다.

```java
@Slf4j  
@ControllerAdvice  
public class GlobalExceptionHandler {  
  
  /**  
   * Validated 시 바인딩 에러가 존재할 때 발생
   */
   @ExceptionHandler(MethodArgumentNotValidException.class)  
  protected ResponseEntity<ErrorResponse> handleMethodArgumentNotValidException(  
      MethodArgumentNotValidException e) {  
    log.info("Handle MethodArgumentNotValidException", e);  
  
    final int status = HttpStatus.BAD_REQUEST.value();  
  
    final ErrorResponse response = ErrorResponse.of(status, ErrorCode.INVALID_INPUT_VALUE,  
        e.getBindingResult());  
  
    return new ResponseEntity<>(response, HttpStatus.valueOf(status));  
  }

  // ...
  
  // Spring에서 일어날 수 있는 일반적인 예외 처리
  
  // ...
  
  @ExceptionHandler(DuplicateException.class)
  protected ResponseEntity<ErrorResponse> handleDuplicationException(final DuplicateException e) {
    log.info("Handle DuplicationException", e);

    final ErrorCode errorCode = e.getErrorCode();
    final String value = e.getValue();
    final int status = HttpStatus.CONFLICT.value();

    final ErrorResponse errorResponse = ErrorResponse.of(status, errorCode, value);

    return new ResponseEntity<>(errorResponse, HttpStatus.valueOf(status));
  }
  
  @ExceptionHandler(BusinessException.class)  
  protected ResponseEntity<ErrorResponse> handleBusinessException(final BusinessException e) {  
    log.info("Handle BusinessException", e);  
  
    final ErrorCode errorCode = e.getErrorCode();  
    final int status = HttpStatus.BAD_GATEWAY.value();  
  
    final ErrorResponse errorResponse = ErrorResponse.of(status, errorCode);  
  
    return new ResponseEntity<>(errorResponse, HttpStatus.valueOf(status));  
  }  
  
  @ExceptionHandler(Exception.class)  
  protected ResponseEntity<ErrorResponse> handleException(Exception e) {  
    log.error("Handle Exception", e);  
  
    final int status = HttpStatus.INTERNAL_SERVER_ERROR.value();  
  
    final ErrorResponse response = ErrorResponse.of(status, ErrorCode.INTERNAL_SERVER_ERROR);  
  
    return new ResponseEntity<>(response, HttpStatus.INTERNAL_SERVER_ERROR);  
  }  
}
```

앞의 메서드들은 스프링 및 라이브러리에서 자체적으로 일어날 수 있는 예외를 처리한다. 하지만 모든 상황에 대해 어떤 예외가 발생하는지 알기는 어렵다. 따라서 마지막 메서드를 통해 예상하지 못한 모든 예외를 처리한다.

이로써 예외 처리의 서비스 개발의 일관성과 코드 명확성, 클라언트에 대한 일관된 응답을 구현할 수 있다.