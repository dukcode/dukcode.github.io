---
layout: single
title: "[톰캣 최종분석 01] 간단한 웹 서버"
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

# 간단한 웹 서버 만들기

웹 서버는 클라이언트(주로 웹 브라우저)와 통신할 때 HTTP를 사용하기 때문에 HTTP 서버라고도 한다. 자바 기반의 웹 서버는 2개의 중요한 클래스 `java.net.Socket`과 `java.net.ServerSocket`을 사용해 웹 서버를 구현한다.

우리의 웹 서버 어플리케이션은 다음과 같은 3개의 클래스로 구성된다.

- `HttpServer`
- `Request`
- `Response`

애플리케이션의 시작점은 HttpServer 클래스에 있다. `main` 메서드는 `HttpServer` 인스턴스를 생성하고  `await` 메서드를 호출한다. `await` 메서드는 지정한 포트에서의 HTTP 요청을 기다리고, 처리하고, 클라이언트에게 응답을 보내는 메서드이다. `await` 메서드는 중지 명령을 받기 전까지 계속해서 대기상태를 유지한다.

우리가 만들 웹서버는 특정 디렉토리에 있는 정적인 자원을 전달하는 기능만 구현할 것이다. 헤더 정보는 전송하지 않는다. 또 수신된 HTTP 요청의 바이트 스트림을 콘솔에 출력한다.

## `HttpServer` 클래스

```java
package com.dukcode.chap01;  
  
import java.io.File;  
import java.io.IOException;  
import java.io.InputStream;  
import java.io.OutputStream;  
import java.net.InetAddress;  
import java.net.ServerSocket;  
import java.net.Socket;  
  
public class HttpServer {  
  
  public static final String WEB_ROOT =  
      System.getProperty("user.dir") + File.separator + "webroot";  
  
  private static final String SHUTDOWN_COMMAND = "/SHUTDOWN";  
  
  private boolean shutdown = false;  
  
  public static void main(String[] args) {  
    HttpServer server = new HttpServer();  
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
        response.sendStaticResource();  
  
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

간단한 웹 서버의 코드이다. 핵심 메서드는 `await` 메서드이며 `ServerSocket`을 열어 요청을 받으면 `Socket`을 생성한다. `Socket`의 정보를 가지고 `Request` 클래스를 통해 내용을 파싱하고 파싱한 결과물을 `Response` 클래스로 받아서 요청한 정적자원을 전송한다.

또한 요청 URL 경로가 `/SHUT_DOWN`이면 서버를 종료하도록 설계되어 있다.

## `Request` 클래스

```java
package com.dukcode.chap01;  
  
import java.io.IOException;  
import java.io.InputStream;  
  
public class Request {  
  
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
}

```

`parse` 메서드에서는 넘겨 받은 `InputStream`에서 정보를 읽고 URI를 파싱한다. URI 파싱하는 부분을 살펴보면 상당히 간단하게 구성되어 있다.

HTTP 요청 메서드의 모양은 다음과 같다.

![](https://i.imgur.com/sUbywaC.png)

따라서 start line의 첫 공백 문자와 두번째 공백문자의 위치를 찾아 사이의 문자를 리턴한다.

## `Response` 클래스

```java
package com.dukcode.chap01;  
  
import java.io.File;  
import java.io.FileInputStream;  
import java.io.IOException;  
import java.io.OutputStream;  
  
public class Response {  
  
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
      File file = new File(HttpServer.WEB_ROOT, request.getUri());  
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
}

```

핵심 메서드는 `sendStaticResource`이다. `Request`에서 요청한 URI를 받아  `HttpServer`에서 정의해놓은 `WEB_ROOT` 경로에 파일이 존재하면 파일을 읽어와 `OutputStream`에 쓴다. 정적 리소스가 존재하지 않으면 하드 코딩되어있는 `errorMessage`를 전송한다.

## 동작 확인

![](https://i.imgur.com/Uq6nIBV.png)

먼저 `index.html`과 `images` 디렉토리를 [Intellij 샘플 코드](https://github.com/wujichao/how-tomcat-works)로부터 가져오자.

![](https://i.imgur.com/z7wWNr0.png)

우리의 `Response` 클래스의 코드는 그림과 같은 HTTP Response Message규격을 따르지 않고 body만 보내고 있다. body만 보내는 것은 [HTTP 0.9 Response](https://http.dev/0.9#responses)의 규격이다. 따라서 우리가 실제로 정적 리소스를 요청할 때는  웹브라우저 대신 `curl`을 이용해 `--http0.9`옵션을 붙여야 한다. 요즘 웹 브라우저에서는 HTTP 0.9에 대한 지원이 종료되었기 때문이다.

```sh
curl http://localhost:8080/index.html --http0.9
```

위의 명령어를 입력하면 다음과 같이 `index.html`을 잘 가져오는 것을 확인할 수 있다.

![](https://i.imgur.com/sQ7fBG7.png)

하지만 존재하지 않는 정적 리소스에 대한 요청은 `errorMessage`가 HTTP 1.1 형식으로 보내므로 일반 웹브라우저로 요청 가능하다.

![](https://i.imgur.com/s4jD3xe.png)
`errorMessage`에 적힌 내용이 전송되는 것을 확인할 수 있다.

이번 장에서 간단한 웹 서버의 동작 방식을 살펴보았다. 이제 이 클래스들을 기반으로 정적 리소스 외에 동적인 리소스를 처리할 수 있도록 코드를 발전시켜보자.