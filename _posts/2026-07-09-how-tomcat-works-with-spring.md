---
title: Tomcat은 Spring 요청을 어떻게 처리할까
date: 2026-07-09 00:00:00 +0900
categories: [Spring]
tags: [tomcat, spring, servlet, coyote, catalina, jasper, dispatcher-servlet, backend]
---

## Tomcat을 한 문장으로 보면

Tomcat은 HTTP 요청을 받아 Servlet으로 전달하고, Servlet의 실행 결과를 HTTP 응답으로 돌려주는 Servlet Container다.

Spring MVC를 사용하더라도 이 사실은 변하지 않는다.

Spring Controller가 Tomcat에 직접 등록되는 것이 아니다.

Tomcat이 아는 것은 Servlet이고, Spring MVC는 `DispatcherServlet`이라는 Servlet을 통해 Tomcat 안에서 동작한다.

```text
Client
 -> Tomcat
 -> Servlet
 -> DispatcherServlet
 -> Spring Controller
```

그래서 Tomcat 위에서 Spring이 어떻게 동작하는지 보려면 먼저 Servlet Container의 요청 처리 흐름을 봐야 한다.

이번 글에서는 외부 Tomcat에 Spring 애플리케이션을 WAR로 배포해서 사용한다고 가정하고, 요청 하나가 어디를 거쳐 Controller까지 도착하는지 따라가 본다.

## Tomcat의 큰 구성 요소

Tomcat은 크게 세 가지 이름으로 나누어 이해하면 좋다.

```text
Tomcat
  - Coyote
  - Catalina
  - Jasper
```

각 역할은 다음과 같다.

- Coyote: 클라이언트의 HTTP 요청을 받는 Connector 계층
- Catalina: Servlet Container의 본체
- Jasper: JSP를 Servlet으로 바꾸고 실행하는 JSP 엔진

요청 처리 흐름만 놓고 보면 보통 다음과 같이 이어진다.

```text
Client
 -> Coyote
 -> Catalina
 -> Filter Chain
 -> DispatcherServlet
 -> Spring Controller
 -> Response
```

만약 Spring MVC에서 JSP View를 사용한다면 응답을 만드는 과정에 Jasper가 들어온다.

```text
Spring Controller
 -> ViewResolver
 -> JSP
 -> Jasper
 -> Servlet
 -> HTML Response
```

REST API처럼 JSON만 반환하는 애플리케이션이라면 Jasper는 거의 등장하지 않는다.

반대로 JSP를 View 기술로 사용한다면 Jasper는 중요한 역할을 한다.

## Spring을 Tomcat에 올린다는 뜻

외부 Tomcat을 사용할 때는 보통 Spring 애플리케이션을 WAR 파일로 빌드해서 배포한다.

흐름은 다음과 같다.

```text
Spring 애플리케이션 빌드
 -> WAR 생성
 -> Tomcat webapps에 배포
 -> Tomcat이 웹 애플리케이션 Context 생성
 -> DispatcherServlet 등록
 -> 요청 처리 가능
```

Spring Boot를 사용하더라도 외부 Tomcat에 배포하려면 WAR 패키징을 사용할 수 있다.

반대로 Spring Boot의 기본 방식은 embedded Tomcat이다.

```text
외부 Tomcat
  Tomcat이 먼저 실행된다.
  그 위에 Spring WAR가 배포된다.

Embedded Tomcat
  Spring Boot 애플리케이션이 실행된다.
  애플리케이션 내부에서 Tomcat이 함께 시작된다.
```

실행 주체는 다르지만 중요한 구조는 같다.

Spring MVC는 결국 Servlet Container 위에서 동작하고, 그 입구는 `DispatcherServlet`이다.

## Coyote는 HTTP 요청이 들어오는 입구다

Coyote는 Tomcat의 Connector 계층이다.

클라이언트는 Tomcat 내부 구조를 알지 못한다.

브라우저나 API 클라이언트는 단지 HTTP 요청을 보낼 뿐이다.

```http
GET /blog/posts/1 HTTP/1.1
Host: example.com
Accept: application/json
```

이 요청을 가장 먼저 받는 쪽이 Connector이고, Tomcat에서 HTTP Connector를 담당하는 구현 계층이 Coyote다.

Coyote는 대략 다음 일을 한다.

- 클라이언트 연결을 받는다.
- HTTP 요청 메시지를 파싱한다.
- 요청 라인, 헤더, 바디를 해석한다.
- Tomcat 내부에서 사용할 Request, Response 객체를 준비한다.
- 요청을 Catalina 쪽으로 넘긴다.

흐름을 단순화하면 다음과 같다.

```text
Client Socket
 -> Endpoint
 -> Processor
 -> CoyoteAdapter
 -> Catalina
```

Endpoint는 클라이언트와의 네트워크 입구에 가깝다.

Processor는 HTTP 메시지를 읽고 요청 정보를 파악한다.

그리고 `CoyoteAdapter`는 Coyote 계층에서 처리한 요청을 Catalina가 이해할 수 있는 Servlet 처리 흐름으로 넘겨준다.

즉, Coyote는 HTTP 세계와 Servlet Container 세계를 연결하는 입구다.

## Catalina는 Servlet Container의 본체다

Coyote가 요청을 받아 Catalina로 넘기면, 이제 Tomcat의 Servlet Container 본체가 움직인다.

Catalina는 어떤 웹 애플리케이션이 이 요청을 처리해야 하는지, 그 안의 어떤 Servlet이 실행되어야 하는지를 찾아간다.

Tomcat의 Container 구조는 보통 다음 계층으로 이해할 수 있다.

```text
Server
 -> Service
 -> Engine
 -> Host
 -> Context
 -> Wrapper
 -> Servlet
```

각 계층의 의미는 다음과 같다.

- Server: Tomcat 전체를 나타내는 최상위 구성 요소
- Service: Connector와 Engine을 묶는 단위
- Engine: 하나의 Service 안에서 실제 요청 처리를 담당하는 엔진
- Host: 가상 호스트, 예를 들면 `localhost`나 `example.com`
- Context: 하나의 웹 애플리케이션
- Wrapper: 하나의 Servlet을 감싸는 Container
- Servlet: 실제 요청을 처리하는 Servlet 객체

Spring 애플리케이션 기준으로 보면 더 구체적으로 연결된다.

```text
Context
  하나의 Spring 웹 애플리케이션

Wrapper
  DispatcherServlet을 감싸는 객체

Servlet
  DispatcherServlet
```

예를 들어 `/blog/posts/1` 요청이 들어왔다고 하자.

Tomcat은 먼저 어떤 Host의 요청인지 확인한다.

그 다음 URL 경로를 보고 어떤 Context에 속하는지 찾는다.

그리고 그 Context 안에서 어떤 Servlet 매핑이 이 요청을 처리할지 찾는다.

Spring MVC 애플리케이션에서는 대부분의 요청이 `DispatcherServlet`으로 매핑된다.

```text
GET /blog/posts/1

Host: example.com
Context: /blog
Wrapper: DispatcherServlet
Servlet: DispatcherServlet
```

여기서 중요한 점은 Tomcat이 `PostController`를 찾는 것이 아니라는 점이다.

Tomcat이 찾는 것은 Servlet이다.

Spring Controller를 찾는 일은 그 다음 단계에서 `DispatcherServlet`이 맡는다.

## Pipeline과 Valve

Catalina는 요청을 찾은 Servlet으로 바로 던지지 않는다.

Container 계층마다 Pipeline이 있고, 그 안에 Valve가 들어갈 수 있다.

```text
Engine Pipeline
 -> Host Pipeline
 -> Context Pipeline
 -> Wrapper Pipeline
 -> Filter Chain
 -> Servlet
```

Pipeline은 요청이 지나가는 통로이고, Valve는 그 통로에 끼워 넣는 처리 장치라고 볼 수 있다.

대표적인 예시는 Access Log다.

```text
AccessLogValve
  요청 정보와 응답 상태를 access log로 남긴다.
```

또 다른 예로 Tomcat 기본 에러 페이지를 만드는 Valve도 있다.

```text
ErrorReportValve
  Tomcat 레벨에서 에러 응답 화면을 만든다.
```

Valve는 Tomcat Container 레벨에서 동작한다.

반면 Filter는 웹 애플리케이션 레벨에서 동작한다.

둘 다 요청 전후에 공통 처리를 넣을 수 있지만 위치가 다르다.

```text
Valve
  Tomcat Container 레벨의 공통 처리

Filter
  웹 애플리케이션 레벨의 공통 처리
```

그래서 Access Log처럼 Tomcat 자체가 관리하는 기능은 Valve로 들어가고, 인증, 인코딩, 로깅처럼 애플리케이션과 가까운 기능은 Filter로 들어가는 경우가 많다.

## Filter Chain은 Spring으로 들어가기 전 단계다

Tomcat이 최종적으로 실행할 Servlet을 찾으면, Servlet이 바로 실행되기 전에 Filter Chain이 먼저 동작한다.

Servlet Filter는 Java 웹 애플리케이션 표준 기능이다.

요청 전후에 공통 로직을 넣을 수 있다.

```text
Request
 -> CharacterEncodingFilter
 -> Spring Security Filter Chain
 -> Logging Filter
 -> DispatcherServlet
```

Spring 애플리케이션에서 Filter Chain을 이해해야 하는 대표적인 이유는 Spring Security 때문이다.

Spring Security는 Controller 앞단에서 동작한다.

인증이 실패하면 요청이 `DispatcherServlet`까지 가지 않을 수 있다.

```text
Request
 -> Spring Security Filter
 -> 인증 실패
 -> 401 Response
```

이 경우 Controller 메서드는 호출되지 않는다.

`@ControllerAdvice`나 `@ExceptionHandler`에서 처리되지 않는 인증 실패가 생기는 이유도 여기에 있다.

요청이 아직 Spring MVC의 Controller 영역에 들어가기 전이기 때문이다.

## DispatcherServlet은 Spring MVC의 입구다

Filter Chain을 통과하면 드디어 `DispatcherServlet`이 실행된다.

`DispatcherServlet`은 Spring MVC의 Front Controller다.

Tomcat 입장에서 보면 그냥 하나의 Servlet이다.

하지만 Spring MVC 입장에서 보면 모든 웹 요청의 입구다.

```text
Tomcat
 -> DispatcherServlet
 -> HandlerMapping
 -> HandlerAdapter
 -> Controller
 -> Service
 -> Repository
```

`DispatcherServlet`은 먼저 `HandlerMapping`을 사용해 어떤 Controller 메서드가 요청을 처리할지 찾는다.

```java
@RestController
public class PostController {

    @GetMapping("/posts/{id}")
    public PostResponse findPost(@PathVariable Long id) {
        return postService.findPost(id);
    }
}
```

다음 요청이 들어왔다고 하자.

```http
GET /posts/1
```

Spring MVC는 이 요청을 다음 메서드에 매핑한다.

```java
@GetMapping("/posts/{id}")
public PostResponse findPost(@PathVariable Long id)
```

그 다음 `HandlerAdapter`가 실제 Controller 메서드를 호출한다.

메서드가 객체를 반환하면 JSON 응답에서는 보통 `HttpMessageConverter`가 동작한다.

```java
public record PostResponse(
    Long id,
    String title,
    String content
) {
}
```

반환 객체는 JSON body로 변환된다.

```json
{
  "id": 1,
  "title": "Tomcat",
  "content": "Tomcat 요청 처리 흐름"
}
```

정리하면 Tomcat의 역할은 `DispatcherServlet`을 실행하는 데까지다.

그 이후 Controller를 찾고 호출하는 일은 Spring MVC가 맡는다.

## Jasper는 JSP를 Servlet으로 바꾼다

Jasper는 Tomcat의 JSP 엔진이다.

JSP를 사용하지 않는 REST API 서버라면 이 부분은 거의 등장하지 않는다.

하지만 Spring MVC에서 JSP를 View로 사용한다면 Jasper가 필요하다.

예를 들어 Controller가 다음처럼 View 이름을 반환한다고 하자.

```java
@Controller
public class PostPageController {

    @GetMapping("/posts/{id}")
    public String postDetail(@PathVariable Long id, Model model) {
        model.addAttribute("post", postService.findPost(id));
        return "posts/detail";
    }
}
```

Spring MVC는 `ViewResolver`를 통해 View 이름을 실제 JSP 경로로 바꾼다.

```text
"posts/detail"
 -> /WEB-INF/views/posts/detail.jsp
```

그 다음 JSP는 그대로 실행되지 않는다.

Jasper가 JSP를 Java Servlet 코드로 변환하고, 컴파일하고, 그 Servlet을 실행한다.

```text
Controller
 -> return "posts/detail"
 -> ViewResolver
 -> /WEB-INF/views/posts/detail.jsp
 -> Jasper
 -> Servlet Java 코드 생성
 -> 컴파일
 -> Servlet 실행
 -> HTML Response
```

그래서 JSP의 정체는 HTML 파일이라기보다 Servlet으로 변환되는 템플릿에 가깝다.

처음 JSP에 접근할 때 상대적으로 느리게 느껴질 수 있는 이유도 여기에 있다.

JSP를 Servlet으로 변환하고 컴파일하는 과정이 필요할 수 있기 때문이다.

물론 한 번 준비된 JSP Servlet은 이후 요청에서 재사용될 수 있다.

## 요청 하나의 전체 흐름 다시 보기

이제 Spring MVC 요청 하나가 Tomcat 안에서 어떻게 흐르는지 처음부터 다시 보자.

```text
1. Client가 HTTP 요청을 보낸다.
2. Coyote가 요청을 받는다.
3. Coyote의 Processor가 HTTP 메시지를 파싱한다.
4. CoyoteAdapter가 요청을 Catalina로 넘긴다.
5. Catalina가 Engine, Host, Context, Wrapper를 따라 요청 대상을 찾는다.
6. Container의 Pipeline과 Valve를 통과한다.
7. 웹 애플리케이션의 Filter Chain이 실행된다.
8. DispatcherServlet이 실행된다.
9. HandlerMapping이 Controller 메서드를 찾는다.
10. HandlerAdapter가 Controller 메서드를 호출한다.
11. Controller가 Service를 호출하고 결과를 반환한다.
12. JSON 응답이면 HttpMessageConverter가 응답 body를 만든다.
13. JSP 응답이면 ViewResolver 이후 Jasper가 JSP를 Servlet으로 처리한다.
14. 응답이 다시 Tomcat을 통해 Client로 나간다.
```

그림처럼 짧게 보면 다음과 같다.

```text
Client
 -> Coyote
 -> Catalina
 -> Valve
 -> Filter
 -> DispatcherServlet
 -> Spring MVC
 -> Controller
 -> Response
```

JSP를 사용하는 경우에는 Controller 이후에 다음 흐름이 추가된다.

```text
Controller
 -> ViewResolver
 -> JSP
 -> Jasper
 -> HTML
```

## 404와 500은 어디서 나는 걸까

Tomcat 구조를 이해하면 에러가 어느 계층에서 발생했는지도 조금 더 잘 볼 수 있다.

예를 들어 요청한 Context 자체가 없다면 Spring MVC까지 가지 못한다.

```text
GET /wrong/posts/1

Tomcat
 -> Host는 찾음
 -> Context를 못 찾음
 -> Tomcat 레벨의 404
```

반대로 Context와 `DispatcherServlet`까지는 찾았지만, Spring MVC 안에서 매핑되는 Controller가 없다면 Spring MVC 레벨의 404가 될 수 있다.

```text
GET /blog/unknown

Tomcat
 -> Context 찾음
 -> DispatcherServlet 찾음
 -> Spring MVC에서 Handler를 못 찾음
 -> Spring MVC 레벨의 404
```

인증 실패도 위치에 따라 다르게 보인다.

Spring Security Filter에서 요청을 막으면 Controller는 호출되지 않는다.

```text
Request
 -> Security Filter
 -> 인증 실패
 -> Controller 호출 안 됨
```

Controller 안에서 예외가 발생하면 그때는 Spring MVC의 예외 처리 흐름이 동작할 수 있다.

```text
Request
 -> DispatcherServlet
 -> Controller
 -> 예외 발생
 -> @ControllerAdvice 처리 가능
```

즉, 같은 404나 500처럼 보여도 Tomcat 레벨에서 난 것인지, Filter에서 끝난 것인지, Spring MVC까지 들어간 뒤 난 것인지 구분해야 한다.

## 외부 Tomcat과 Embedded Tomcat을 다시 보면

외부 Tomcat과 Embedded Tomcat은 실행 방식이 다르다.

하지만 요청 처리의 본질은 같다.

외부 Tomcat에서는 Tomcat이 먼저 떠 있고, 그 위에 Spring WAR가 올라간다.

```text
Tomcat 실행
 -> WAR 배포
 -> Context 생성
 -> DispatcherServlet 등록
```

Embedded Tomcat에서는 Spring Boot 애플리케이션이 먼저 시작되고, 내부에서 Tomcat을 생성한다.

```text
java -jar app.jar
 -> Spring Boot 시작
 -> Embedded Tomcat 생성
 -> DispatcherServlet 등록
```

어느 쪽이든 Spring MVC 요청은 Servlet 구조 위에서 처리된다.

차이는 Tomcat을 누가 먼저 시작하느냐와 배포 단위가 무엇이냐에 있다.

```text
외부 Tomcat
  배포 단위: WAR
  실행 주체: Tomcat

Embedded Tomcat
  배포 단위: 실행 가능한 JAR
  실행 주체: Spring Boot 애플리케이션
```

## Spring 개발자가 Tomcat 구조를 알아야 하는 이유

Spring만 보고 있으면 요청은 곧바로 Controller로 들어오는 것처럼 느껴진다.

하지만 실제 흐름은 더 길다.

```text
Client
 -> Coyote
 -> Catalina
 -> Valve
 -> Filter Chain
 -> DispatcherServlet
 -> Controller
```

이 구조를 알면 다음 문제를 더 잘 구분할 수 있다.

- 요청이 Controller까지 오지 않는 이유
- 인증 실패가 `@ControllerAdvice`에서 잡히지 않는 이유
- Context Path가 맞지 않아 404가 나는 이유
- Tomcat 기본 에러 페이지와 Spring 에러 응답이 다르게 보이는 이유
- JSP 첫 요청이 상대적으로 느릴 수 있는 이유
- WAR 배포 후 `/`, `/app`, `/blog` 경로가 다르게 동작하는 이유

특히 외부 Tomcat에 Spring을 올려서 운영한다면 Context Path를 자주 마주친다.

예를 들어 WAR 파일 이름이 `blog.war`라면 기본적으로 Context Path가 `/blog`가 될 수 있다.

```text
blog.war
 -> /blog
```

이 상태에서 Controller가 `/posts/{id}`로 매핑되어 있다면 실제 접근 경로는 다음과 같이 보인다.

```text
/blog/posts/1
```

Controller 입장에서는 `/posts/1`이지만, 클라이언트 입장에서는 Context Path까지 포함한 `/blog/posts/1`로 접근한다.

이 차이를 모르면 Tomcat 배포 후 "로컬에서는 됐는데 서버에서는 404가 난다" 같은 문제를 만나기 쉽다.

요청을 실제로 실행하는 Acceptor, Poller, worker thread pool과 `maxThreads`, `maxConnections`, `acceptCount`의 관계는 [Tomcat의 요청 처리 스레드는 어떻게 동작할까]({% post_url 2026-07-10-how-tomcat-threads-work-with-spring %})에서 이어서 살펴본다.

## 정리

Tomcat은 Spring Controller를 직접 실행하지 않는다.

Tomcat은 HTTP 요청을 받아 Servlet으로 전달하는 Servlet Container다.

Coyote는 클라이언트의 HTTP 요청을 받고, Catalina는 어떤 Host, Context, Wrapper, Servlet이 요청을 처리할지 찾는다.

그 과정에서 Pipeline과 Valve를 지나고, 웹 애플리케이션의 Filter Chain을 통과한 뒤 `DispatcherServlet`이 실행된다.

`DispatcherServlet`부터는 Spring MVC의 영역이다.

Spring MVC는 `HandlerMapping`으로 Controller를 찾고, `HandlerAdapter`로 Controller 메서드를 호출한다.

JSON 응답이면 `HttpMessageConverter`가 body를 만들고, JSP 응답이면 `ViewResolver` 이후 Jasper가 JSP를 Servlet으로 바꾸어 HTML을 만든다.

결국 전체 흐름은 다음 문장으로 정리할 수 있다.

```text
Coyote는 요청을 받고,
Catalina는 Servlet을 찾고,
DispatcherServlet은 Spring Controller를 호출한다.
JSP를 사용한다면 Jasper가 JSP를 Servlet으로 바꿔 실행한다.
```

Spring MVC는 Tomcat과 별개의 세계에서 혼자 동작하는 것이 아니다.

Tomcat의 Servlet 구조 안으로 `DispatcherServlet`이라는 입구를 만들어 들어오고, 그 안에서 우리가 익숙한 Controller, Service, Repository 흐름이 이어진다.
