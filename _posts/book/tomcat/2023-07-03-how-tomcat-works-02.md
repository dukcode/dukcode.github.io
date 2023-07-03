---
layout: single
title: "[톰캣 최종분석 02] 간단한 서블릿 컨테이너"
categories: how-tomcat-works
tag: [tomcat, 톰캣_최종분석, how_tomcat_works, 톰캣_만들기, 톰캣_클론코딩]
toc: true
toc_sticky: true
author_profile: true
sidebar:
    nav: "docs"
header:
  teaser: "https://i.imgur.com/YQHJ2cj.png"
# search: false
---

# 간단한 서블릿 컨테이너 만들기

서블릿은 클라이언트의 요청을 처리하고, 그 결과를 반환하는 자바의 표준 인터페이스이다. 서블릿 인터페이스는 Java Servlet API 안에 존재한다.

우리는 서블릿 인터페이스를 구현하여 요청을 원하는 방식으로 처리할 수 있다. 서블릿 컨테이너는 HTTP 요청을 받아 필요한 전처리를 한 후, 우리가 구현한 서블릿 클래스를 가져와 실행하고 응답을 받아 전송하게 된다. 스프링 MVC의 시작점인 `DispatcherServlet`도 서블릿 인터페이스를 구현한 클래스이다.

서블릿 인터페이스를 구현하기 때문에 우리가 구현한 서블릿 클래스를 톰캣이나 제티 등의 서블릿 컨테이너 어디에서든 코드 변경 없이 사용할 수 있다.

## `Servlet` 인터페이스란?

서블릿 프로그래밍은 `javax.servlet` 패키지와 `javax.servlet.http` 패키지의 클래스와 인터페이스를 통해 이뤄진다. 이 패키지에서 가장 중요한 인터페이스가 `Servlet`이라고 할 수 있다. 모든 서블릿은 이 인터페이스를 구현해야 하기 때문이다.

`Servlet` 인터페이스에는 다음과 같은 5개의 메서드가 있다.

- `public void init(ServletConfig config)`
- `public void service(ServletRequest request, ServletResponse response)`
- `public void destory()`
- `public ServletConfig getServletConfig()`
- `public String getServletInfo()`

`init`, `service`, `destroy`는 서블릿 생명주기와 관련된 메서드이다.

`init` 메서드는 서블릿 컨테이너가  서블릿 클래스를 인스턴스화 한 뒤 호출한다. 서블릿 컨테이너가 이 메서드를 호출하면 해당 서블릿이 서비스를 할 수 있는 준비가 됨을 보장해야 한다. 예를들어 데이터베이스 드라이버나 어떤 초기값을 로드하는 일과 같은 코드를 `init` 메서드 안에 구현해야 한다. 딱히 초기화가 필요하지 않는 경우에는 이 메서드를 비워두는 것이 보통이다.

서블릿 컨테이너는 요청이 있을 때마다 해당 서블릿의 `service` 메서드를 호출한다. 이 때 서블릿 컨테이너는 `ServletRequest`와 `ServletResponse`를 전당한다. `ServletRequest`에는 클라이언트의 HTTP 요청에 관한 정보를 담고, `ServletResponse`에는 응답에 관한 정보를 캡슐화한다. `service` 메서드는 서블릿의 생명주기 동안 자주 호출된다.

서블릿 컨테이너는 서비스 영역으로부터 서블릿 인스턴스를 제거하기에 앞서 해당 서블릿의 `destroy` 메서드를 호출한다. `destroy` 메서드는 서블릿이 소유하고 있던 메모리, 파일 핸들, 스레드 등과 같은 자원을 깨끗이 반환할 수 있는 기회를 제공한다.

## 사전 준비

가장 먼저 서블릿 인터페이스가 포함된 Java Servlet API를 Dependency에 추가해야 한다. 최신 버전의 Java Servlet API는 [mvnrepository](https://mvnrepository.com/artifact/javax.servlet/javax.servlet-api)에서 작성일 기준 `4.0.1`버전 까지 나온 것으로 보인다.

하지만 이 책에서는 `2.4`버전을 사용하므로 `2.4`버전을 사용하자([mvnrepository](https://mvnrepository.com/artifact/javax.servlet/servlet-api)). `build.gradle`에 다음과 같이 추가한다.

```groovy
dependencies {  
    implementation 'javax.servlet:servlet-api:2.4' // servlet 의존성 추기
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.8.1'  
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.8.1'  
}
```

이제 서블릿 관련 클래스들을 사용할 수 있다.

### 원시 Servlet

우리는 서블릿 컨테이너를 작성할 것이다. 이를 테스트할 때 간단하게 사용할 수 있는 Servlet 클래스를 먼저 만들어 보자.

`./webroot`에 다음과 같이 작성한다.

```java
import java.io.IOException;
import java.io.PrintWriter;
import javax.servlet.Servlet;
import javax.servlet.ServletConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

public class PrimitiveServlet implements Servlet {

  @Override
  public void init(ServletConfig config) throws ServletException {
    System.out.println("init");
  }

  @Override
  public void service(ServletRequest req, ServletResponse res)
      throws ServletException, IOException {
    System.out.println("from service");
    PrintWriter out = res.getWriter();
    out.println("Hello. Roses are red.");
    out.print("Violets are blue.");
  }

  @Override
  public void destroy() {
    System.out.println("destroy");
  }

  @Override
  public ServletConfig getServletConfig() {
    return null;
  }

  @Override
  public String getServletInfo() {
    return null;
  }
}

```

아주 간단하게 하드 코딩된 메시지를 프린트하는 클래스이다. 응답의 마지막 줄은 body로 끝나야 하므로 마지막 줄엔 `print`메서드를 사용했다.

이제 이 클래스를 컴파일 해야한다. `PrimitiveServlet`클래스는 `Servlet`에 의존하고 있기 때문에 `Servlet`클래스를 가지고 있는 `jar`파일을 클래스 패스에 추가해야 컴파일 할 수 있다.

`jar`파일의 경로를 알기 위해 다음과 같이 복사한다.

![](https://i.imgur.com/NNjXNpa.png)

여기서 Absolute Path를 복사하면 된다.

![](https://i.imgur.com/7iEqfyK.png)

이제 복사한 경로를 `-cp`옵션에 넣고 `PrimitiveServlet`을 컴파일하자.

```sh
$ java -cp "javac -cp "[복사한 경로]" PrimitiveServlet.java
```

그러면 아래와 같이 `class`파일이 생성되고 우리의 애플리케이션에서 불러와 실행할 수 있다.

![](https://i.imgur.com/D8RfEfX.png)

## 첫 번째 애플리케이션

완전한 서블릿 컨테이너는 다음과 같이 HTTP 요청을 처리해야한다.

- 어떤 서블릿을 처음으로 요청받았을 떄, 해당 서블릿 클래스를 로드하고 서블릿의 `init` 메서드를 (딱 한 번) 호출한다.
- 각 요청에서 `ServletRequest`와 `ServletResponse` 인스턴스를 생성한다.
- `ServletRequest`와 `ServletResponse`를 전달해 서블릿의 `service` 메서드를 호출한다.
- 서블릿 클래스를 종룟하면서 서블릿의 `destroy` 메서드를 호출하고 서블릿 클래스를 언로드해야 한다.

하지만 첫번째 서블릿 컨테이너에서는 `init`메서드와 `destroy`메서드 호출을 없애기로 한다. 또한 `/servlet/**`에 대한 요청은 서블릿을 로드하여 처리하고, 나머지 요청은 정적 리소스를 반환하기로 한다.

먼저 전체적인 구조를 UML로 확인해 보자.

![](https://i.imgur.com/v8kZGPn.png)

애플리케이션의 진입점은 `HttpServer1` 클래스에 위치하며 `main`메서드는 `HttpServer1`의 인스턴스를 생성하고 `await` 메서드를 호출한다. `await` 메서드를 HTTP 요청을 기다리고, 모든 요청에 대해 `Request`와 `Response` 인스턴스를 생성한다. 그리고 정적 자원의 요청인지 서블릿 요청인지에 따라 두 객체를 `StaticResourceProcessor`나 `ServletProcessor` 인스턴스에 디스패치한다.

### `HttpServer1` 클래스

이 클래스는 [chap01](http://dukcode.github.org/how-tomcat-works/how-tomcat-works-01)의 `HttpServer` 클래스와 거의 유사하다. 다른 점은 정적 자원 말고도 서블릿 요청에 대한 처리도 할 수 있다는 점이다.

정적 자원에 요청하려면 다음과 같은 URL로 요청하면 된다.

```sh
$ curl http://localhost:8080/staticResourse --http0.9
```

우리가 만든 원시 Servlet에 요청하려면 다음과 같이 요청하면 된다.

```sh
$ curl http://localhost:8080/servlet/PrimitiveServlet --http0.9
```

이제 코드를 살펴보자.

```java
package com.dukcode.chap02;  
  
import java.io.IOException;  
import java.io.InputStream;  
import java.io.OutputStream;  
import java.net.InetAddress;  
import java.net.ServerSocket;  
import java.net.Socket;  
  
public class HttpServer1 {  
  
  private static final String SHUTDOWN_COMMAND = "/SHUTDOWN";  
  
  private boolean shutdown = false;  
  
  public static void main(String[] args) {  
    HttpServer1 server = new HttpServer1();  
    server.await();  
  }  
  
  private void await() {  
    ServerSocket serverSocket = null;  
    int port = 8080;  
    try {  
      serverSocket = new ServerSocket(port, 1, InetAddress.getByName("127.0.0.1"));  
    } catch (IOException e) {  
      e.printStackTrace();  
      System.exit(1);  
    }  
  
    while (!shutdown) {  
      Socket socket = null;  
      InputStream input = null;  
      OutputStream output = null;  
  
      try {  
        socket = serverSocket.accept();  
        input = socket.getInputStream();  
        output = socket.getOutputStream();  
  
        Request request = new Request(input);  
        request.parse();  
  
        Response response = new Response(output);  
        response.setRequest(request);  
  
        if (request.getUri().startsWith("/servlet/")) {  
          ServletProcessor1 processor = new ServletProcessor1();  
          processor.process(request, response);  
        } else {  
          StaticResourceProcessor processor = new StaticResourceProcessor();  
          processor.process(request, response);  
        }  
  
        socket.close();  
  
        shutdown = request.getUri().equals(SHUTDOWN_COMMAND);  
  
      } catch (IOException e) {  
        e.printStackTrace();  
        continue;  
      }  
    }  
  }  
  
}
```

기본적인 동작 방식은 [chap01](http://dukcode.github.org/how-tomcat-works/how-tomcat-works-01)의 `HttpServer`과 유사하다. 하지만 다른점은 `Response` 인스턴스에게 직접 응답을 전송하도록 시키는 것이 아닌 경우에 따라 `ServletProcessor1`과 `StaticResourceProcessor`에게 프로세스를 맡긴다.

`/servlet/`으로 시작하는 경우에 `ServletProcessor1`에게 프로세스를 맡기고 아닌 경우에 `StaticResourceProceccor`에게 프로세스를 맡긴다.

또 다른 점은 기존의 `WEB_ROOT`가 `Constants`라는 새로운 클래스로 옮겨 갔다는 것이다. `WEB_ROOT`는 `ServletProcessor1`에서 동적으로 클래스를 로드할때 필요한 경로이므로 외부 클래스에서 상수를 관리하는 것이 타탕해 보인다.

### `Request`와 `Response` 클래스

우리는 Servlet 표준이 제공하는 `Request`와 `Reponse`를 사용해 서블릿 요청을 처리할 것이므로 두 클래스 모두 `ServletRequest`, `ServletResponse`라는 인터페이스를 구현하도록 변경한다. 두 클래스의 로직은  [chap01](http://dukcode.github.org/how-tomcat-works/how-tomcat-works-01)의 로직과 대부분 유사하다. 

인스턴스에서 구현하도록 요구하는 메서드들은 일단 구현을 미루도록 빈 구현을 해놓도록 하자.

[`Request`]
```java
package com.dukcode.chap02;  
  
import java.io.BufferedReader;  
import java.io.IOException;  
import java.io.InputStream;  
import java.io.UnsupportedEncodingException;  
import java.util.Enumeration;  
import java.util.Locale;  
import java.util.Map;  
import javax.servlet.RequestDispatcher;  
import javax.servlet.ServletInputStream;  
import javax.servlet.ServletRequest;  
  
public class Request implements ServletRequest {  
  
  private InputStream input;  
  private String uri;  
  
  public Request(InputStream input) {  
    this.input = input;  
  }  
  
  public void parse() {  
    StringBuilder request = new StringBuilder(2048);  
    int i;  
    byte[] buffer = new byte[2048];  
    try {  
      i = input.read(buffer);  
    } catch (IOException e) {  
      e.printStackTrace();  
      i = -1;  
    }  
    for (int j = 0; j < i; j++) {  
      request.append((char) buffer[j]);  
    }  
    System.out.print(request.toString());  
    uri = parseUri(request.toString());  
  }  
  
  private String parseUri(String requestString) {  
    int index1;  
    int index2;  
  
    index1 = requestString.indexOf(' ');  
    if (index1 != -1) {  
      index2 = requestString.indexOf(' ', index1 + 1);  
      if (index2 > index1) {  
        return requestString.substring(index1 + 1, index2);  
      }  
    }  
    return null;  
  }  
  
  public String getUri() {  
    return uri;  
  }  
  
  /* 이하 SerlvetRequest의 빈 구현 */  
  @Override  
  public Object getAttribute(String name) {  
    return null;  
  }  
  
  // 생략...
  
  @Override  
  public int getLocalPort() {  
    return 0;  
  }  
}
```

`Request` 클래스는 `ServletRequest`를 구현하면서 기존의 로직은 그대로 가지고 있도록 구현한다.

[`Response`]
```java
package com.dukcode.chap02;  
  
import java.io.File;  
import java.io.FileInputStream;  
import java.io.IOException;  
import java.io.OutputStream;  
import java.io.PrintWriter;  
import java.util.Locale;  
import javax.servlet.ServletOutputStream;  
import javax.servlet.ServletResponse;  
  
public class Response implements ServletResponse {  
  
  private static final int BUFFER_SIZE = 1024;  
  Request request;  
  OutputStream output;  
  
  public Response(OutputStream output) {  
    this.output = output;  
  }  
  
  public void setRequest(Request request) {  
    this.request = request;  
  }  
  
  public void sendStaticResource() throws IOException {  
    byte[] bytes = new byte[BUFFER_SIZE];  
    FileInputStream fis = null;  
    try {  
      File file = new File(Constants.WEB_ROOT, request.getUri());  
      if (file.exists()) {  
        fis = new FileInputStream(file);  
        int ch = fis.read(bytes, 0, BUFFER_SIZE);  
        while (ch != -1) {  
          output.write(bytes, 0, ch);  
          ch = fis.read(bytes, 0, BUFFER_SIZE);  
        }  
      } else {  
        String errorMessage = """  
            HTTP/1.1 404 File Not Found\r  
            Content-Type: text/html\r  
            Content-Length: 23\r  
            \r  
            <h1>File Not Found</h1>""";  
        output.write(errorMessage.getBytes());  
      }  
    } catch (IOException e) {  
      e.printStackTrace();  
    } finally {  
      if (fis != null) {  
        fis.close();  
      }  
    }  
  }  
  
  @Override  
  public PrintWriter getWriter() throws IOException {  
    PrintWriter writer = new PrintWriter(output, true);  
    return writer;  
  }  
  
  /* 이하 SerlvetResponse의 빈 구현 */  
  @Override  
  public String getCharacterEncoding() {  
    return null;  
  }  
  
  // 생략...
  
  @Override  
  public void setLocale(Locale loc) {  
  
  }  
}
```

`Response`도 마찬가지고 `ServletResponse`를 구현한다. 하지만 거의 빈 구현이고 유일하게 `getWriter` 메서드를 구현한다.

![](https://i.imgur.com/Gh8r61U.png)


`PrintWriter` 생성자의 두 번째 파라미터는 `autoFlush` 여부이다. 인스턴스의 `println`, `printf`, `format` 메서드는 자동적으로 플러시 하지만, `print` 메서드는 자동적으로 플러시하지 않는다. 따라서 `PrimitiveServlet`의 마지막 줄인 `out.print("Violets are blue.");`는 플러시 하지 않아서 요청 시에 볼 수 없다. (HTTP 응답의 마지막 줄은 body로 끝나야 해서 `print`메서드를 사용함)

이 문제는 추후 애플리케이션을 개발하면서 해결할 것이다.

### `StaticResourceProcessor` 클래스

정적 자원에 대한 처리를 하는 간단한 클래스이다. `Response`의 `sendStaticResource`메서드를 호출 하는 기능이 전부이다.

```java
package com.dukcode.chap02;  
  
import java.io.IOException;  
  
public class StaticResourceProcessor {  
  
  public void process(Request request, Response response) {  
    try {  
      response.sendStaticResource();  
    } catch (IOException e) {  
      e.printStackTrace();  
    }  
  }  
}
```

### `ServletProcessor1` 클래스

`ServletProcessor1`클래스의 `process` 메서드의 역할은 `WEB_ROOT` 경로에서 URI 경로와 맞는 서블릿 클래스를 로드해서 `service`메서드를 호출하는 기능을 한다.

클래스를 동적으로 로드하는 코드는 처음 짜봐서 난해할 수 있지만, 사용법을 익힌다고 생각해보고 직접 작성해보자. 클래스로더는 [chap08](http://dukcode.github.org/how-tomcat-works/how-tomcat-works-08)에서 자세히 다룰 예정이다.

샘플 코드와 다른 점은 클래스를 읽어오는 부분이다. `Class.newInstance`는 예외 처리가 난해해 `JAVA 9`부터 Deprecated 되었다. `Class.getContructor` 메서드를 기본 생성자를 가져오고 기본 생성자를 통해 클래스를 로드하는 방식으로 변경했다.

로드한 클래스를 `Servlet`으로 캐스팅해 `service` 메서드를 호출한다.

```java
package com.dukcode.chap02;  
  
import java.io.File;  
import java.io.IOException;  
import java.lang.reflect.Constructor;  
import java.net.URL;  
import java.net.URLClassLoader;  
import java.net.URLStreamHandler;  
import javax.servlet.Servlet;  
import javax.servlet.ServletRequest;  
import javax.servlet.ServletResponse;  
  
public class ServletProcessor1 {  
  
  public void process(Request request, Response response) {  
    String uri = request.getUri();  
    String servletName = uri.substring(uri.lastIndexOf("/") + 1);  
    URLClassLoader loader = null;  
    try {  
      URL[] urls = new URL[1];  
      URLStreamHandler streamHandler = null;  
      File classPath = new File(Constants.WEB_ROOT);  
  
      String repository = new URL("file", null,  
          classPath.getCanonicalPath() + File.separator).toString();  
      urls[0] = new URL(null, repository, streamHandler);  
      loader = new URLClassLoader(urls);  
    } catch (IOException e) {  
      e.printStackTrace();  
    }  
  
    Class myClass = null;  
    try {  
      myClass = loader.loadClass(servletName);  
    } catch (ClassNotFoundException e) {  
      e.printStackTrace();  
    }  
  
    Servlet servlet = null;  
    try {  
      Constructor constructor = myClass.getConstructor();  
      servlet = (Servlet) constructor.newInstance();  
      servlet.service((ServletRequest) request, (ServletResponse) response);  
    } catch (Throwable e) {  
      e.printStackTrace();  
    }  
  }  
  
}
```

`URLCLassLoader`의 가장 간단한 생성자는 `URL`의 배열을 파라미터로 받는다. 이 때, `URL`의 경로가 `/`로 끝나는 경우에는 디렉토리를 참조하는 것으로 간주되고 , `/`로 끝나지 않으면 `jar`파일을 참조하는 것으로 간주된다.

일반적으로 서블릿 컨테이너에서 클래스 로더가 서블릿 클래스를 찾는 위치를 `repository`라고 명한다. `repository`가 `/`로 끝나는 것을 눈여겨 보자.

실제로 톰캣은 `URL`를 만들 때, `public URL(URL context, String spec, URLStreamHandler handler)` 생성자를 사용한다. 또한 다른 생성자도 존재한다. `public URL(String protocol, String host, String file)`의 형태이다.

우리는 톰캣의 구현을 따라 첫 번째의 생성자를 사용하고 싶다. 하지만 `new URL(null, repository, null)`의 형태로 작성한다면 컴파일러 입장에서 어떤 생성자를 사용할지 판단할 수 없다. 따라서 `URLStreamHandler`를 `null`로 선언해 넣어준 것이다.

### `Constants` 클래스

단순한 상수 클래스이다.

```java
package com.dukcode.chap02;  
  
import java.io.File;  
  
public class Constants {  
  
  public static final String WEB_ROOT =  
      System.getProperty("user.dir") + File.separator + "webroot";  
  
}
```

### 실행해보기

이제 정적 리소스와 서블릿 모두를 호출해보자.

정적 리소스를 호출하기 위해 다음과 같이 명령어를 입력한다.

```sh
$ curl http://localhost:8080/index.html --http0.9
```

아래와 같이 정적 리소스를 잘 불러오는 것을 확인할 수 있다.

![](https://i.imgur.com/sPXnRI6.png)

이제 서블릿을 호출해 보자. 다음과 같은 명령어로 서블릿을 호출한다.

```sh
$ curl http://localhost:8080/servlet/PrimitiveServlet --http0.9
```

아래와 같이 잘 작동하는 것을 확인할 수 있다. `PrimitiveServlet`의 마지막 `print`메서드는 플러시 되지 못한 것도 확인할 수 있다.

![](https://i.imgur.com/Gitxhc0.png)


## 두 번째 애플리케이션

첫번째 애플리케이션은 심각한 문제가 존재한다. `ServletProcessor1` 클래스의 `process`메서드를 확인해 보면 `Request` 인스턴스와 `Response` 인스턴스를 각각 `ServletRequest`와 `ServletResponse`로 업캐스팅해 전달하고 있다.

이런 방식은 보안과 관련한 문제를 야기한다. 서블릿 컨테이너의 내부를 알고있는 서블릿 프로그래머라면 서블릿 내부에서 `ServletRequest`와 `ServletResponse`를 각각 `Request`와 `Response`로 다운캐스팅해 퍼블릭 메서드를 호출할 수 있다. 예를들어 서블릿 내부에서 `Request`의 `parse`메서드를 호출하고, `Response`에서 `sendStaticResource`를 호출하는 등, 서블릿 컨테이너 개발자가 의도하지 않은대로 서블릿을 작성할 수 있게 된다.

이를 해결하기 위한 방법이 있다. `FACADE` 패턴을 사용하는 것이다. 기존 클래스 메서드를 퍼블릭으로 유지하면서 의도되지않은 서블릿 개발자의 사용을 막을 수 있다.

### `RequestFacade`와 `ResponseFacade` 클래스

`RequestFacade`와 `ResponseFacade` 클래스는 각각 `Request`와 `Response`를 인스턴스로 가지고 있는 `FACADE` 클래스이다. 외부에 공개하고 싶은 메서드만 위임 메서드로 구현한다. 코드는 다음과 같다.

[`RequestFacade`]
```java
package com.dukcode.chap02;  
  
import java.io.BufferedReader;  
import java.io.IOException;  
import java.io.UnsupportedEncodingException;  
import java.util.Enumeration;  
import java.util.Locale;  
import java.util.Map;  
import javax.servlet.RequestDispatcher;  
import javax.servlet.ServletInputStream;  
import javax.servlet.ServletRequest;  
  
public class RequestFacade implements ServletRequest {  
    
  private Request request;  
  
  public RequestFacade(Request request) {  
    this.request = request;  
  }  
  
  /* 이하 SerlvetRequest의 위임 메서드 구현 */  
  @Override  
  public Object getAttribute(String name) {  
    return request.getAttribute(name);  
  }  

  // 생략...

  @Override  
  public int getLocalPort() {  
    return request.getLocalPort();  
  }  
}
```

[`ResponseFacace`]
```java
package com.dukcode.chap02;  
  
import java.io.IOException;  
import java.io.PrintWriter;  
import java.util.Locale;  
import javax.servlet.ServletOutputStream;  
import javax.servlet.ServletResponse;  
  
public class ResponseFacade implements ServletResponse {  
    
  private Response response;  
  
  public ResponseFacade(Response response) {  
    this.response = response;  
  }  
  
  /* 이하 위임 메서드 구현 */  
  @Override  
  public PrintWriter getWriter() throws IOException {  
    return response.getWriter();  
  }  
    
  // 생략...
  
  @Override  
  public Locale getLocale() {  
    return response.getLocale();  
  }  
}
```

이제 `Request`의 `parseUri` 메서드와 `Response`의 `sendStaticResource` 메서드는 안전해졌다.

### `HttpServer2` 클래스

`HttpServer2` 클래스는 `await` 메서드에서 `ServletProcessor1`대신 `ServletProcessor2`를 사용한다는 점만 제외하면 `HttpServer1` 클래스와 같다.

[다른 부분]
```java
        if (request.getUri().startsWith("/servlet/")) {
          // ServeletProcessor2 사용
          ServletProcessor2 processor = new ServletProcessor2();
          processor.process(request, response);
        } else {
          // ...
        }

```

### `ServletProcessor2` 클래스

`ServletProcessor2` 클래스는 `Request`와 `Response`를 각각 `ServletRequest`와 `ServletResponse`로 바로 업캐스팅 하지 않고 `RequestFacade`와 `ResponseFacade`를 생성하고 업캐스팅하여 전달한다. 나머지 부분은 같다.

[다른 부분]
```java
    try {
      Constructor constructor = myClass.getConstructor();
      servlet = (Servlet) constructor.newInstance();

      // FACADE 클래스를 생성해 전달
      RequestFacade requestFacade = new RequestFacade(request);
      ResponseFacade responseFacade = new ResponseFacade(response);
      
      servlet.service((ServletRequest) requestFacade, (ServletResponse) responseFacade);
    } catch (Throwable e) {
      e.printStackTrace();
    }

```

### 실행해보기

이제 두 번째 애플리케이션도 첫 번째 애플리케이션과 같은 방법으로 실행할 수 있다. 다른 점은 서블릿 클래스에서 `Request`와 `Response`의 허용되지 않는 퍼블릭 메서드에 접근할 수 없다는 것이다.

[정적 리소스 접근]<br>
![](https://i.imgur.com/BZeUPJZ.png)

[서블릿 접근]<br>
![](https://i.imgur.com/9wZLb7H.png)
