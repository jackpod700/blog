---
title: Tomcat의 요청 처리 스레드는 어떻게 동작할까
date: 2026-07-10 00:00:00 +0900
categories: [Spring]
tags: [tomcat, spring, thread, thread-pool, nio, connector, servlet, performance]
---

## 왜 Tomcat의 스레드를 알아야 할까

Spring MVC로 API를 만들다 보면 요청은 자연스럽게 Controller부터 시작하는 것처럼 보인다.

```text
Request
 -> Controller
 -> Service
 -> Repository
 -> Response
```

하지만 Controller를 실행하는 주체는 Spring이 새로 만든 스레드가 아니다.

Tomcat이 관리하는 요청 처리 스레드가 `DispatcherServlet`을 거쳐 Controller, Service, Repository까지 실행한다.

```text
Tomcat worker thread
 -> Filter Chain
 -> DispatcherServlet
 -> Controller
 -> Service
 -> Repository
```

그렇다면 다음과 같은 질문이 생긴다.

- 클라이언트 연결마다 스레드가 하나씩 생기는가?
- `maxThreads=200`이면 201번째 요청은 어디에서 기다리는가?
- DB 응답을 기다리는 동안 Tomcat 스레드는 무엇을 하는가?
- `maxConnections`와 `acceptCount`는 스레드 수와 어떤 관계인가?
- Spring MVC의 비동기 요청은 Tomcat 스레드를 어떻게 반환하는가?
- 가상 스레드를 켜면 `maxThreads`는 어떻게 되는가?

앞선 [Tomcat은 Spring 요청을 어떻게 처리할까]({% post_url 2026-07-09-how-tomcat-works-with-spring %})에서는 Coyote, Catalina, `DispatcherServlet`을 따라 요청의 전체 경로를 살펴봤다.

이번 글에서는 그 경로를 실행하는 스레드에 집중한다.

설명의 기준은 다음과 같다.

```text
Tomcat 10.1
NIO Connector
Spring MVC
동기식 요청 처리
플랫폼 스레드
```

외부 Tomcat에 WAR를 배포하든 Spring Boot의 embedded Tomcat을 사용하든, Connector가 시작된 이후의 핵심 스레드 모델은 같다.

HTTP/1.1 요청 하나를 기준으로 기본 모델을 먼저 이해하고, 뒤에서 Servlet 비동기 처리와 가상 스레드가 무엇을 바꾸는지 살펴본다.

## 연결, 요청, 스레드는 서로 다른 개념이다

Tomcat 스레드를 이해할 때 가장 먼저 분리해야 하는 것은 연결, 요청, 스레드다.

| 개념 | 의미 | 생명주기 |
|---|---|---|
| Connection | 클라이언트와 서버 사이의 TCP 연결 | 여러 HTTP 요청보다 오래 유지될 수 있다 |
| Request | 연결을 통해 전달되는 HTTP 요청 한 건 | 응답을 만들면 끝난다 |
| Worker Thread | 요청 처리 코드를 실행하는 스레드 | 풀에서 꺼내 쓰고 다른 요청에 재사용한다 |

HTTP keep-alive를 사용하면 하나의 TCP 연결로 여러 요청을 순서대로 보낼 수 있다.

```text
TCP Connection A
  -> Request 1
  -> Response 1
  -> Request 2
  -> Response 2
```

이 연결이 열려 있는 전체 시간 동안 worker thread 하나가 계속 붙어 있는 것은 아니다.

NIO Connector에서는 다음 요청이 도착하기를 기다리는 keep-alive 연결을 `Poller`가 감시한다.

실제로 읽고 처리할 일이 생겼을 때만 작업을 worker thread pool에 넘긴다.

```text
keep-alive 연결에서 다음 요청을 기다리는 중
 -> Poller가 감시
 -> worker thread를 점유하지 않음

새 요청 데이터가 도착함
 -> worker thread에 처리 작업 제출
 -> 요청 처리 동안 worker thread 사용
```

따라서 다음 등식은 성립하지 않는다.

```text
동시 연결 수 = worker thread 수
```

HTTP/2에서는 하나의 연결 안에서 여러 stream이 동시에 이동할 수도 있으므로 연결과 요청의 구분은 더 중요해진다.

## NIO Connector에는 역할이 다른 스레드가 있다

Tomcat의 NIO Connector를 단순화하면 세 역할로 나눌 수 있다.

```text
Acceptor
Poller
Worker
```

### Acceptor는 연결을 받는다

`Acceptor`는 서버 소켓에서 새로운 TCP 연결을 받아들인다.

```text
Client
 -> TCP connect
 -> ServerSocket
 -> Acceptor
```

Acceptor가 하는 핵심 일은 연결을 수락하고 소켓을 Tomcat이 관리할 수 있는 형태로 준비하는 것이다.

Controller나 Service 같은 애플리케이션 코드를 실행하는 스레드가 아니다.

연결 수가 `maxConnections`에 도달하면 Acceptor는 더 많은 연결을 계속 처리하지 못한다.

이때 새 연결 요청은 운영체제의 listen backlog에서 기다릴 수 있고, 그 길이에 관여하는 설정이 `acceptCount`다.

### Poller는 소켓의 이벤트를 감시한다

NIO의 핵심은 많은 소켓을 각각 전용 스레드로 기다리지 않는 데 있다.

`Poller`는 Java NIO의 `Selector`를 사용해 여러 소켓을 감시한다.

```text
Socket A: 아직 읽을 데이터 없음
Socket B: 요청 데이터 도착
Socket C: 응답을 이어서 쓸 수 있음
```

읽기나 쓰기가 가능한 소켓이 생기면 Poller는 해당 이벤트를 처리할 `SocketProcessor` 작업을 만든다.

그리고 그 작업을 worker thread pool의 `Executor`에 제출한다.

```text
Poller
 -> read/write ready event
 -> SocketProcessor
 -> Executor
```

Poller도 Spring Controller를 직접 실행하지 않는다.

어떤 소켓에 실제 처리할 일이 생겼는지 찾아 worker 쪽으로 넘기는 역할에 가깝다.

### Worker는 실제 요청 처리 코드를 실행한다

Executor에서 `SocketProcessor`를 가져와 실행하는 쪽이 worker thread다.

기본 NIO Connector에서는 로그나 thread dump에서 다음과 비슷한 이름을 볼 수 있다.

```text
http-nio-8080-exec-1
http-nio-8080-exec-2
http-nio-8080-exec-3
```

이 스레드가 Coyote의 HTTP 처리부터 Catalina, Servlet Filter, Spring MVC까지 이어서 실행한다.

```text
Worker Thread
 -> SocketProcessor
 -> Coyote Processor
 -> CoyoteAdapter
 -> Catalina
 -> Filter Chain
 -> DispatcherServlet
 -> Controller
```

Acceptor와 Poller가 네트워크 이벤트를 관리하고, worker가 애플리케이션 요청을 실행한다고 구분하면 전체 구조가 선명해진다.

## 요청 하나가 worker thread를 만나는 과정

HTTP 요청 하나의 흐름을 스레드 관점에서 따라가 보자.

### 1. Acceptor가 연결을 수락한다

클라이언트가 8080 포트로 연결하면 Acceptor가 소켓을 받는다.

```text
Client
 -> connect :8080
 -> Acceptor
```

Tomcat은 연결 수를 관리하고, NIO 소켓을 non-blocking 모드로 준비한 뒤 Poller에 등록한다.

### 2. Poller가 요청 데이터 도착을 감지한다

Poller는 소켓에 읽을 데이터가 생길 때까지 `Selector`로 감시한다.

```text
Poller
 -> OP_READ 감지
```

읽기 이벤트가 발생하면 해당 소켓을 무작정 Poller 스레드에서 처리하지 않는다.

`SocketProcessor` 작업을 Executor에 제출한다.

### 3. worker thread가 요청을 실행한다

여유 worker가 작업을 가져가면 HTTP 메시지를 처리하고 Servlet Container로 요청을 전달한다.

```text
http-nio-8080-exec-7
 -> HTTP 요청 파싱
 -> Catalina Pipeline
 -> Filter Chain
 -> DispatcherServlet
 -> Controller
```

Spring MVC의 동기식 Controller라면 Service와 Repository 호출도 같은 스레드에서 이어진다.

### 4. 응답을 만든다

Controller가 반환한 객체는 `HttpMessageConverter` 등을 거쳐 응답 body가 된다.

worker thread는 응답을 소켓에 쓰는 과정도 수행한다.

```text
Controller return
 -> HttpMessageConverter
 -> HTTP Response
 -> Socket write
```

Tomcat 10.1의 Connector 비교표를 보면 NIO Connector는 요청 header를 non-blocking으로 읽고 다음 keep-alive 요청을 non-blocking으로 기다리지만, 요청 body 읽기와 일반적인 응답 쓰기는 blocking 방식이다.

따라서 큰 요청 body를 읽거나 느린 클라이언트에 응답을 쓰는 상황도 worker를 오래 점유할 수 있다.

### 5. 연결을 닫거나 Poller로 돌려보낸다

응답 후 연결을 닫아야 한다면 소켓을 정리한다.

keep-alive 연결이라면 다음 요청을 기다리도록 다시 Poller에 등록한다.

```text
Response 완료
 -> Connection: close
    -> 소켓 종료

Response 완료
 -> keep-alive
    -> Poller에 다시 등록
    -> worker thread 반환
```

이제 worker thread는 풀로 돌아가 다른 요청을 처리할 수 있다.

## 동기식 Spring MVC는 한 worker가 끝까지 실행한다

기본적인 Spring MVC 요청은 thread-per-request 모델로 이해할 수 있다.

요청 하나가 worker를 배정받으면 특별한 비동기 경계를 만나지 않는 한 요청 처리가 끝날 때까지 같은 worker가 호출 스택을 이어간다.

```text
http-nio-8080-exec-7
  Filter.doFilter()
  DispatcherServlet.doDispatch()
  PostController.findPost()
  PostService.findPost()
  PostRepository.findById()
```

직접 확인하려면 Controller와 Service에서 현재 스레드 이름을 출력해 볼 수 있다.

```java
@RestController
@RequiredArgsConstructor
public class PostController {

    private final PostService postService;

    @GetMapping("/posts/{id}")
    public PostResponse findPost(@PathVariable Long id) {
        log.info("controller thread={}", Thread.currentThread().getName());
        return postService.findPost(id);
    }
}
```

```java
@Service
public class PostService {

    public PostResponse findPost(Long id) {
        log.info("service thread={}", Thread.currentThread().getName());
        return loadPost(id);
    }
}
```

동기식 요청이라면 두 로그에 보통 같은 이름이 찍힌다.

```text
controller thread=http-nio-8080-exec-7
service thread=http-nio-8080-exec-7
```

여기서 중요한 점은 worker가 요청 전용으로 새로 만들어진 스레드가 아니라는 것이다.

요청이 끝나면 같은 스레드가 나중에 전혀 다른 사용자의 요청을 처리한다.

```text
exec-7
 -> 사용자 A의 요청 처리
 -> pool로 반환
 -> 사용자 B의 요청 처리
```

그래서 애플리케이션이 직접 `ThreadLocal`이나 MDC에 값을 넣었다면 반드시 정리해야 한다.

```java
try {
    requestContext.set(requestId);
    filterChain.doFilter(request, response);
} finally {
    requestContext.remove();
}
```

정리하지 않으면 이전 요청의 값이 같은 worker를 재사용한 다음 요청에 남을 수 있다.

## Blocking I/O 동안 worker는 반환되지 않는다

동기식 Controller가 DB를 조회한다고 해 보자.

```java
@Transactional(readOnly = true)
public PostResponse findPost(Long id) {
    Post post = postRepository.findById(id)
        .orElseThrow();
    return PostResponse.from(post);
}
```

JDBC 호출이 DB 응답을 기다리는 동안 코드는 실행할 일이 거의 없다.

하지만 플랫폼 worker thread는 요청을 맡은 상태로 대기한다.

```text
Tomcat worker
 -> JDBC query 전송
 -> DB 응답 대기
 -> worker 점유 상태 유지
 -> 응답 도착
 -> 요청 처리 계속
```

외부 API를 동기식 HTTP Client로 호출할 때도 같다.

Redis, 파일 I/O, lock 획득, DB connection pool 대기 역시 worker를 붙잡을 수 있다.

```text
worker를 오래 점유하는 대표 원인
  - 느린 SQL
  - DB connection pool 대기
  - 외부 API 응답 대기
  - 느린 request body 업로드
  - 느린 response write
  - synchronized lock 경합
  - 긴 sleep 또는 재시도
```

따라서 Tomcat의 처리량은 단순히 CPU 연산 속도만으로 결정되지 않는다.

요청이 어떤 자원을 얼마나 오래 기다리는지가 worker의 회전 속도를 결정한다.

## Tomcat thread pool은 어떻게 커질까

공유 Executor를 따로 지정하지 않으면 Connector는 내부 Executor를 만든다.

플랫폼 스레드를 사용하는 내부 풀의 핵심 값은 다음과 같다.

```text
minSpareThreads
maxThreads
maxQueueSize
```

`minSpareThreads`는 유지할 최소 worker 수다.

`maxThreads`는 Connector가 생성할 요청 처리 worker의 최대 수다.

`maxQueueSize`는 모든 worker가 바쁠 때 실행을 기다릴 `Runnable` 작업의 최대 수다.

Tomcat은 일반적인 무제한 큐 기반 `ThreadPoolExecutor`와 조금 다른 `TaskQueue`를 사용한다.

여유 worker가 없다면 작업을 곧바로 큐에 쌓기보다 `maxThreads`까지 worker를 늘릴 수 있도록 동작한다.

흐름을 단순화하면 다음과 같다.

```text
작업 제출
 -> 놀고 있는 worker가 있음
    -> 기존 worker가 실행

 -> 놀고 있는 worker가 없음
    -> 현재 worker 수 < maxThreads
       -> worker 생성 후 실행

    -> 현재 worker 수 = maxThreads
       -> TaskQueue에서 대기

    -> TaskQueue도 가득 참
       -> 작업 거부
```

즉, 기본 플랫폼 스레드 모델에서 풀은 부하에 따라 최소 수에서 최대 수까지 커지고, 최대 수에 도달한 뒤 작업이 큐에 쌓인다.

큐 용량까지 다 쓰면 Executor가 작업을 거부한다.

이때 클라이언트가 항상 예쁜 `503 Service Unavailable` 응답을 받는다고 가정하면 안 된다.

거부가 발생한 시점과 소켓 상태에 따라 연결 종료, reset, timeout 같은 형태로 보일 수 있고 Tomcat 로그에는 작업 거부가 남을 수 있다.

## maxThreads, maxQueueSize, maxConnections, acceptCount

Tomcat 과부하 설정에서 가장 많이 혼동하는 네 값이 있다.

Tomcat 10.1 NIO Connector의 기본값과 의미를 정리하면 다음과 같다.

| 설정 | 기본값 | 제한하는 대상 |
|---|---:|---|
| `maxThreads` | 200 | 동시에 실행할 요청 처리 worker |
| `maxQueueSize` | `Integer.MAX_VALUE` | worker를 기다리는 실행 작업 |
| `maxConnections` | 8192 | Tomcat이 받아서 관리하는 연결 |
| `acceptCount` | 100 | 연결 한도 이후 OS가 대기시킬 연결 요청 |

이 기본값은 Tomcat 10.1의 내부 플랫폼 Executor를 사용하는 NIO Connector 기준이다.

Tomcat 버전, Connector 구현, 공유 Executor, Spring Boot 버전에 따라 실제 적용값은 달라질 수 있으므로 운영 환경의 설정과 JMX 값을 확인해야 한다.

### maxThreads는 실행 동시성의 상한이다

동기식 요청은 처리되는 동안 worker 하나가 필요하다.

`maxThreads=200`이면 Connector 내부 풀은 최대 200개의 요청 처리 플랫폼 스레드를 만든다.

```text
active worker <= 200
```

하지만 이것이 서버가 TCP 연결을 200개만 받을 수 있다는 뜻은 아니다.

idle keep-alive 연결은 Poller가 worker 없이 관리할 수 있기 때문이다.

### maxQueueSize는 실행 작업의 대기열이다

worker가 모두 바쁘면 새 `SocketProcessor` 작업은 실행 큐에서 기다릴 수 있다.

```text
Worker 200개 모두 사용 중
 -> 새 실행 작업
 -> TaskQueue
```

기본값은 사실상 매우 큰 값이다.

큰 큐는 순간적인 부하를 흡수할 수 있지만, 처리 능력보다 요청이 빠르게 들어오는 상태가 계속되면 긴 대기 시간과 메모리 사용 증가를 숨길 수 있다.

```text
처리량 < 유입량
 -> queue 증가
 -> 응답 지연 증가
 -> 클라이언트 timeout
 -> 재시도로 부하 증가
```

무조건 큰 큐가 안전한 것은 아니다.

서비스가 허용할 수 있는 최대 대기 시간과 실패 전략을 기준으로 제한된 큐를 검토해야 한다.

### maxConnections는 열린 연결의 상한이다

`maxConnections`는 Tomcat이 동시에 받아서 처리 상태로 관리하는 연결 수를 제한한다.

여기에는 요청을 실행 중인 연결뿐 아니라 다음 요청을 기다리는 keep-alive 연결도 포함될 수 있다.

```text
active request connections
+ idle keep-alive connections
+ other accepted connections
<= maxConnections
```

그래서 `maxConnections`는 보통 `maxThreads`보다 클 수 있다.

NIO가 많은 연결을 적은 Poller와 제한된 worker pool로 관리하는 이유가 여기에 있다.

### acceptCount는 HTTP 요청 큐가 아니다

`acceptCount`는 Tomcat이 관리하는 요청 작업 큐의 길이가 아니다.

Tomcat이 `maxConnections`까지 연결을 받은 뒤, 아직 `accept()`되지 못한 새 TCP 연결 요청을 운영체제가 보관하는 listen backlog와 관련된 값이다.

```text
Tomcat accepted connections
 -> maxConnections 도달

새 TCP 연결 요청
 -> OS listen backlog
 -> acceptCount가 크기에 관여
```

운영체제가 설정값을 그대로 적용하지 않을 수도 있다.

backlog까지 가득 차면 새 연결은 거절되거나 timeout이 날 수 있다.

따라서 다음 설명은 정확하지 않다.

```text
maxThreads가 200이므로 201번째 HTTP 요청은 acceptCount에서 기다린다.
```

201번째 요청의 연결이 이미 Tomcat에 받아들여졌다면 worker 실행 큐나 소켓 이벤트 처리 단계에서 기다릴 수 있다.

`acceptCount`가 등장하는 경계는 `maxThreads`가 아니라 기본적으로 `maxConnections` 이후의 새 연결이다.

## 200개의 느린 요청이 들어오면 어떤 일이 생길까

다음과 같이 설정했다고 가정하자.

```text
maxThreads=200
maxQueueSize=100
maxConnections=1000
acceptCount=100
```

그리고 모든 요청이 느린 외부 API를 5초 동안 기다린다고 하자.

### 실행 중인 요청

worker가 최대 200개까지 늘어나고 각 worker가 요청 하나를 맡는다.

```text
200 requests
 -> 200 workers
 -> 외부 API 응답 대기
```

플랫폼 스레드는 대부분 기다리고 있지만 모두 점유된 상태다.

### worker를 기다리는 작업

그 뒤 읽기 가능한 소켓에서 새 처리 작업이 만들어지면 최대 100개까지 TaskQueue에 대기할 수 있다.

```text
100 runnable tasks
 -> TaskQueue에서 worker 반환 대기
```

이 요청들의 실제 비즈니스 로직은 아직 시작하지 못했지만, 사용자 관점의 응답 시간은 이미 흐르고 있다.

### 실행 큐까지 가득 찬 경우

큐가 가득 찬 뒤 제출되는 작업은 Executor에서 거부될 수 있다.

관련 소켓은 닫힐 수 있고 클라이언트에서는 연결 오류나 timeout으로 관찰될 수 있다.

### 연결 수까지 가득 찬 경우

열린 연결이 1000개에 도달하면 `maxConnections` 경계가 적용된다.

그 이후 새 TCP 연결 요청은 운영체제 backlog에서 최대 `acceptCount`의 영향을 받으며 기다릴 수 있다.

이 예시는 네 설정이 하나의 직선 큐가 아니라 서로 다른 계층의 한도라는 점을 보여 준다.

```text
실행 계층
  maxThreads
  maxQueueSize

연결 계층
  maxConnections
  acceptCount
```

## Tomcat thread pool만 키우면 빨라질까

`maxThreads`를 늘리면 동시에 더 많은 요청이 애플리케이션 코드로 들어올 수 있다.

하지만 병목이 다른 곳에 있다면 처리량은 늘지 않는다.

예를 들어 다음과 같은 구성을 보자.

```text
Tomcat maxThreads = 200
HikariCP maximumPoolSize = 20
```

모든 요청이 DB connection을 필요로 한다면 동시에 DB를 사용하는 요청은 connection pool 크기의 영향을 받는다.

```text
20 workers
 -> DB connection 획득
 -> SQL 실행

나머지 workers
 -> HikariCP에서 connection 반환 대기
```

DB connection을 기다리는 worker도 Tomcat 기준으로는 바쁜 스레드다.

`maxThreads`를 400으로 늘리면 DB 처리량이 두 배가 되는 것이 아니라, DB connection을 기다리는 스레드만 더 늘어날 수 있다.

외부 API connection pool이 50개인 경우도 같다.

```text
Tomcat worker 200
 -> outbound HTTP connection 50
 -> 나머지는 connection 대기 가능
```

오히려 과도하게 큰 플랫폼 thread pool은 다음 비용을 늘릴 수 있다.

- 각 스레드의 stack과 네이티브 자원
- CPU context switching
- lock 경합
- DB와 외부 시스템으로 전달되는 동시 부하
- 장애 시 대기 중인 요청 수와 복구 시간

Tomcat pool은 혼자 튜닝할 수 있는 섬이 아니다.

DB pool, HTTP Client pool, Redis pool, downstream 처리량, timeout, CPU와 함께 봐야 한다.

## 필요한 동시성은 응답 시간과 함께 결정된다

필요한 동시 요청 수를 거칠게 생각할 때 Little's Law를 사용할 수 있다.

```text
평균 동시 요청 수 ≈ 초당 요청 수 × 평균 응답 시간(초)
```

평균 100 RPS를 처리하고 응답 시간이 200ms라면 다음과 같다.

```text
100 × 0.2 = 20
```

평균적으로 약 20개의 요청이 동시에 처리 중이라고 볼 수 있다.

같은 100 RPS인데 downstream 장애로 평균 응답 시간이 2초가 되면 결과가 달라진다.

```text
100 × 2 = 200
```

응답 시간이 열 배 느려졌을 뿐인데 필요한 동시성은 200까지 증가한다.

이 상태에서 `maxThreads=200`이면 worker가 모두 차기 쉽다.

그래서 thread pool 포화는 유입 트래픽이 갑자기 늘어서만 발생하지 않는다.

DB나 외부 API가 느려져서 요청 하나가 worker를 오래 잡고 있어도 발생한다.

다만 이 식만으로 `maxThreads`를 바로 정하면 안 된다.

평균값은 순간 burst와 긴 tail latency를 숨기고, 실제 요청은 CPU 작업과 여러 I/O 대기를 섞어서 수행한다.

부하 테스트에서 다음 값을 함께 관찰해야 한다.

- 처리량과 평균, p95, p99 응답 시간
- busy worker와 최대 worker 수
- 실행 큐 대기
- DB connection pool 사용량과 대기 시간
- 외부 API connection pool과 timeout
- CPU, 메모리, GC, context switch
- 오류율과 클라이언트 재시도

## Spring Boot에서 thread pool 설정하기

embedded Tomcat을 사용하는 Spring Boot에서는 다음처럼 설정할 수 있다.

```yaml
server:
  tomcat:
    threads:
      min-spare: 20
      max: 200
      max-queue-capacity: 100
    max-connections: 1000
    accept-count: 100
    connection-timeout: 5s
    keep-alive-timeout: 20s
```

각 값은 다음 경계를 조정한다.

```text
threads.max
  -> worker 최대 수

threads.max-queue-capacity
  -> 실행 작업 queue 용량

max-connections
  -> Tomcat이 관리할 연결 수

accept-count
  -> OS backlog에 관여
```

`server.tomcat.threads.max-queue-capacity`는 Spring Boot 3.3부터 추가되었다.

다른 버전을 사용한다면 해당 버전의 application properties와 실제 Tomcat 버전을 먼저 확인해야 한다.

설정값은 예시일 뿐 권장 정답이 아니다.

운영 환경에서는 앞단 load balancer의 연결 및 timeout, 컨테이너 CPU 제한, downstream pool과 함께 결정해야 한다.

## 외부 Tomcat에서 설정하기

외부 Tomcat의 내부 Connector Executor를 사용할 때는 `server.xml`의 Connector에 설정한다.

```xml
<Connector
    port="8080"
    protocol="org.apache.coyote.http11.Http11NioProtocol"
    minSpareThreads="20"
    maxThreads="200"
    maxQueueSize="100"
    maxConnections="1000"
    acceptCount="100"
    connectionTimeout="5000"
    keepAliveTimeout="20000" />
```

여러 Connector가 thread pool을 공유하게 만들고 싶다면 별도의 Executor를 정의할 수 있다.

```xml
<Executor
    name="tomcatThreadPool"
    namePrefix="tomcat-exec-"
    minSpareThreads="20"
    maxThreads="200"
    maxQueueSize="100" />

<Connector
    port="8080"
    protocol="org.apache.coyote.http11.Http11NioProtocol"
    executor="tomcatThreadPool"
    maxConnections="1000"
    acceptCount="100" />
```

Executor 요소는 이를 참조하는 Connector보다 먼저 선언되어야 한다.

그리고 공유 Executor를 연결하면 Connector에 적은 `maxThreads`, `minSpareThreads` 같은 thread 설정은 무시된다.

이 경우 thread 관련 값은 Executor에 설정해야 한다.

```text
Connector 내부 Executor 사용
 -> Connector의 thread 설정 적용

공유 Executor 사용
 -> Executor의 thread 설정 적용
 -> Connector의 thread 설정 무시
```

설정 파일에 값이 존재한다는 사실과 실제로 적용된다는 사실은 다를 수 있으므로 JMX에서 런타임 값을 확인하는 편이 안전하다.

## timeout은 worker를 보호하는 경계다

스레드가 무한히 기다리지 않게 만드는 것도 pool 크기만큼 중요하다.

다음 대기에는 각각 별도의 timeout이 있을 수 있다.

```text
Client
 -> Load Balancer timeout
 -> Tomcat connection timeout
 -> Spring request 처리
 -> DB connection timeout
 -> JDBC query timeout
 -> 외부 HTTP connect/read timeout
```

외부 API의 read timeout이 없으면 장애가 난 downstream을 worker들이 계속 기다릴 수 있다.

DB connection timeout이 너무 길면 connection pool이 고갈됐을 때 Tomcat worker가 줄지어 대기한다.

실행 큐가 크고 모든 timeout이 길면 서버는 실패하지 않는 것이 아니라 매우 늦게 실패한다.

이때 앞단 클라이언트가 먼저 timeout을 내고 재시도하면, 서버는 이미 버려진 요청을 계속 처리하면서 더 큰 부하를 받을 수도 있다.

따라서 timeout은 단순한 예외 설정이 아니라 worker 점유 시간을 제한하는 용량 관리 수단이다.

## Spring MVC 비동기 처리는 무엇이 다를까

지금까지의 설명은 동기식 요청 기준이다.

Spring MVC는 Servlet 비동기 처리를 이용해 container worker를 요청 완료 전에도 반환할 수 있다.

대표적인 반환 타입은 `Callable`, `DeferredResult`, `WebAsyncTask`다.

### Callable

Controller가 `Callable`을 반환하는 예를 보자.

```java
@GetMapping("/reports/{id}")
public Callable<ReportResponse> findReport(@PathVariable Long id) {
    return () -> reportService.findReport(id);
}
```

대략적인 흐름은 다음과 같다.

```text
Tomcat worker A
 -> DispatcherServlet
 -> Controller가 Callable 반환
 -> request.startAsync()
 -> Filter와 DispatcherServlet 종료
 -> worker A를 Tomcat pool에 반환

Spring AsyncTaskExecutor thread
 -> Callable 실행
 -> 결과 생성

Tomcat worker B
 -> ASYNC dispatch
 -> DispatcherServlet 재진입
 -> 결과로 응답 완성
```

긴 작업을 실행하는 동안 최초 Tomcat worker를 점유하지 않는다는 점이 핵심이다.

하지만 작업이 사라진 것은 아니다.

`Callable`은 Spring MVC에 설정된 `AsyncTaskExecutor`의 다른 스레드에서 실행된다.

그 Executor의 pool, queue, timeout을 별도로 관리하지 않으면 병목을 다른 곳으로 옮길 뿐이다.

Spring 공식 문서도 기본 비동기 Executor가 부하 상황의 운영 환경에 적합하지 않으므로 명시적으로 설정하라고 안내한다.

### DeferredResult

`DeferredResult`는 결과를 나중에 다른 실행 흐름에서 넣을 수 있다.

```java
@GetMapping("/events/{id}")
public DeferredResult<EventResponse> findEvent(@PathVariable Long id) {
    DeferredResult<EventResponse> result = new DeferredResult<>(5_000L);
    eventService.register(id, result);
    return result;
}
```

Controller가 `DeferredResult`를 반환하면 Spring MVC는 async mode를 시작하고 Tomcat worker를 돌려보낸다.

나중에 다른 스레드가 `setResult()`를 호출하면 Servlet Container로 async dispatch가 발생하고, worker를 다시 배정받아 응답을 마무리한다.

응답이 완료되지 않았더라도 연결은 계속 열려 있다.

따라서 비동기 처리는 worker 점유를 줄일 수 있지만 `maxConnections`, async timeout, 메모리, 열린 요청 수까지 없애 주지는 않는다.

### @Async는 Servlet async와 같지 않다

Service 메서드에 `@Async`를 붙였다고 Tomcat worker가 자동으로 반환되는 것은 아니다.

```java
@GetMapping("/reports/{id}")
public ReportResponse findReport(@PathVariable Long id) {
    return reportService.findReportAsync(id).join();
}
```

위 코드처럼 Controller가 `join()`으로 결과를 기다리면 작업은 다른 스레드에서 실행되더라도 Tomcat worker는 기다리는 상태로 남는다.

```text
Tomcat worker
 -> @Async 작업 제출
 -> join() 대기
 -> worker 반환 안 됨
```

Controller가 Spring MVC가 지원하는 비동기 반환 타입을 그대로 반환하고 기다리지 않아야 Servlet async 처리로 연결될 수 있다.

```java
@GetMapping("/reports/{id}")
public CompletableFuture<ReportResponse> findReport(@PathVariable Long id) {
    return reportService.findReportAsync(id);
}
```

비동기 경계를 넘으면 실행 스레드가 바뀌므로 `ThreadLocal`, MDC, Security Context 같은 문맥 전파도 별도로 확인해야 한다.

## 비동기 Spring MVC와 WebFlux는 같은가

같지 않다.

Spring MVC 비동기 처리는 Servlet의 async 기능을 사용한다.

초기 Filter-Servlet chain에서 빠져나와 container worker를 반환한 뒤, 결과가 준비되면 다시 dispatch한다.

```text
Spring MVC async
 -> Servlet 기반
 -> request.startAsync()
 -> ASYNC dispatch
```

Spring WebFlux는 애초에 Servlet의 요청당 스레드 모델을 기반으로 설계된 것이 아니다.

non-blocking 처리 모델과 event loop를 중심으로 동작한다.

```text
Spring WebFlux
 -> Reactive 기반
 -> event loop
 -> non-blocking pipeline
```

Spring MVC Controller가 `Mono`를 반환한다고 전체 요청 경로의 blocking I/O가 자동으로 non-blocking이 되는 것도 아니다.

JDBC 같은 blocking API를 그대로 사용한다면 그 대기는 여전히 어떤 실행 스레드를 점유한다.

## 가상 스레드를 사용하면 무엇이 바뀔까

Tomcat 10.1은 Java 21 이상에서 요청 작업을 가상 스레드로 실행할 수 있다.

플랫폼 스레드는 OS 스레드와 밀접하게 연결된 상대적으로 비싼 자원이다.

가상 스레드는 JVM이 많은 가상 스레드를 더 적은 carrier 플랫폼 스레드 위에 스케줄링한다.

blocking I/O에서 가상 스레드가 대기할 때 carrier를 반환할 수 있으므로 thread-per-request 스타일을 유지하면서 더 많은 I/O 대기 작업을 다루는 데 유리하다.

```text
Platform thread
 -> blocking I/O 동안 OS thread 점유

Virtual thread
 -> blocking I/O에서 가상 스레드 대기
 -> carrier는 다른 가상 스레드 실행 가능
```

Spring Boot에서는 다음 설정으로 가상 스레드 사용을 활성화할 수 있다.

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

외부 Tomcat의 내부 Executor를 사용한다면 Connector에서 활성화할 수 있다.

```xml
<Connector
    port="8080"
    protocol="org.apache.coyote.http11.Http11NioProtocol"
    useVirtualThreads="true"
    maxConnections="1000"
    acceptCount="100" />
```

Tomcat의 가상 스레드 Executor는 작업마다 새 가상 스레드를 사용한다.

플랫폼 스레드 풀처럼 가상 스레드를 최대 N개로 만들어 재사용하는 방식이 아니다.

그래서 가상 스레드가 활성화되면 `maxThreads`와 `minSpareThreads`는 기존 플랫폼 thread pool의 동시성 제한 역할을 하지 않는다.

그렇다고 동시성을 무제한으로 열어도 된다는 뜻은 아니다.

```text
가상 스레드 수 증가
 != DB connection 증가
 != 외부 API 처리량 증가
 != CPU core 증가
```

가상 스레드는 스레드 대기 비용을 줄여 줄 수 있지만 DB connection pool, outbound connection pool, 메모리와 downstream 용량은 그대로다.

CPU 집약 작업 자체가 빨라지는 것도 아니다.

Oracle 문서의 표현대로 가상 스레드는 더 빠른 스레드가 아니라 더 높은 동시성과 처리량을 위한 스레드다.

가상 스레드 환경에서는 제한된 thread pool 대신 semaphore, connection pool, rate limit, bulkhead 같은 명시적인 자원 한도가 더 중요해진다.

## 운영 환경에서는 무엇을 관찰해야 할까

설정값만 보고 병목을 판단하기는 어렵다.

실제 런타임 상태를 함께 봐야 한다.

### busy thread 비율

Spring Boot Actuator와 Micrometer를 사용하면 다음 Tomcat metric을 확인할 수 있다.

```text
tomcat.threads.busy
tomcat.threads.current
tomcat.threads.config.max
tomcat.connections.current
tomcat.connections.keepalive.current
tomcat.connections.config.max
```

플랫폼 스레드 환경에서 `tomcat.threads.busy`가 `tomcat.threads.config.max`에 계속 붙어 있고 응답 시간이 증가한다면 worker 포화를 의심할 수 있다.

```text
busy / max가 장시간 높음
+ latency 상승
+ timeout 또는 오류 증가
 -> thread pool 또는 downstream 병목 조사
```

busy 값만 높고 처리량이 안정적일 수도 있으므로 반드시 응답 시간, 오류율, downstream 상태와 같이 봐야 한다.

### thread dump

플랫폼 스레드의 실행 위치는 thread dump로 확인할 수 있다.

```bash
jcmd <pid> Thread.print
```

여러 worker가 같은 stack에서 기다린다면 병목 후보가 된다.

```text
http-nio-8080-exec-* 다수가
  HikariPool.getConnection()에서 대기
  -> DB connection pool 고갈 의심

http-nio-8080-exec-* 다수가
  외부 HTTP socket read에서 대기
  -> downstream 지연 또는 read timeout 확인

http-nio-8080-exec-* 다수가
  같은 monitor 진입에서 BLOCKED
  -> lock 경합 확인
```

thread dump에서 `RUNNABLE`이라고 표시된 스레드가 항상 CPU를 적극적으로 사용 중인 것은 아니다.

네이티브 socket read 같은 I/O 대기가 `RUNNABLE`로 보일 수 있으므로 stack trace와 CPU profile을 함께 봐야 한다.

가상 스레드는 다음처럼 파일로 dump할 수 있다.

```bash
jcmd <pid> Thread.dump_to_file -format=json threads.json
```

JFR을 사용하면 긴 가상 스레드 pinning이나 submit 실패 같은 이벤트도 관찰할 수 있다.

### queue와 downstream pool

Tomcat worker 수만 보지 말고 다음 queue를 함께 봐야 한다.

```text
Tomcat task queue
HikariCP connection waiters
HTTP Client pending connections
애플리케이션 Executor queue
Kafka 또는 메시지 처리 queue
```

앞 queue가 비어 있어도 뒤 자원에서 오래 기다리면 worker는 포화될 수 있다.

반대로 worker가 아직 남아 있어도 실행 queue가 빠르게 늘고 있다면 곧 지연이 커질 수 있다.

## thread pool을 조정하는 순서

운영 설정은 다음 순서로 접근하는 편이 좋다.

### 1. 요청 특성을 나눈다

CPU 중심 요청인지, DB나 외부 API 대기가 긴 I/O 중심 요청인지 확인한다.

평균값만 보지 말고 느린 endpoint와 p95, p99를 따로 본다.

### 2. 현재 병목을 찾는다

busy worker, thread dump, DB pool, outbound pool, CPU, GC를 함께 확인한다.

worker가 가득 찼다는 사실은 증상이고, worker가 무엇을 기다리는지가 원인일 수 있다.

### 3. timeout을 먼저 점검한다

무한 대기나 지나치게 긴 downstream timeout을 줄여 worker가 예측 가능한 시간 안에 반환되게 한다.

재시도가 있다면 횟수, backoff와 전체 timeout budget도 같이 본다.

### 4. downstream 용량과 맞춘다

Tomcat worker가 늘어날 때 DB, Redis, 외부 API가 감당할 수 있는지 확인한다.

모든 pool 크기를 같은 숫자로 맞출 필요는 없지만, 앞단 동시성만 훨씬 크게 열어 뒤에서 대기시키는 구조인지 확인해야 한다.

### 5. queue의 실패 전략을 정한다

무제한에 가까운 queue로 오래 기다릴지, 제한된 queue로 빠르게 거부하고 상위 계층에서 backpressure를 적용할지 서비스 특성에 맞게 선택한다.

### 6. 부하 테스트로 다시 검증한다

설정을 바꾼 뒤 처리량만 보지 말고 tail latency, 오류율, 자원 사용량과 장애 시 복구 시간을 비교한다.

```text
가설
 -> 설정 변경
 -> 부하 테스트
 -> metric / thread dump 확인
 -> 병목 재평가
```

`maxThreads`에 모든 서버에 통하는 마법의 숫자는 없다.

요청 시간과 downstream 용량을 측정한 결과가 설정의 근거가 되어야 한다.

## 전체 구조 다시 보기

지금까지 본 Tomcat과 Spring MVC의 스레드 흐름을 한 장으로 정리하면 다음과 같다.

![Tomcat NIO와 Spring MVC 스레드 처리 전체 구조](/assets/img/posts/2026-07-10-tomcat-thread-model.svg)

동기식 요청의 핵심 흐름은 다음과 같다.

```text
1. Acceptor가 연결을 받는다.
2. Poller가 소켓 이벤트를 감시한다.
3. 읽을 데이터가 생기면 SocketProcessor를 Executor에 제출한다.
4. worker가 Coyote, Catalina, Spring MVC를 실행한다.
5. 동기식 DB와 외부 API 대기 중에도 worker는 점유된다.
6. 응답이 끝나면 worker는 풀로 돌아간다.
7. keep-alive 연결은 다음 요청을 기다리도록 Poller로 돌아간다.
```

설정의 경계는 다음처럼 기억할 수 있다.

```text
maxThreads
  실행 중인 요청 작업의 상한

maxQueueSize
  worker를 기다리는 실행 작업의 상한

maxConnections
  Tomcat이 관리하는 연결의 상한

acceptCount
  그 뒤 OS에 대기하는 새 연결 요청의 한도
```

Spring MVC의 기본 요청은 Tomcat worker 하나가 Controller부터 Repository까지 이어서 실행한다.

그래서 DB와 외부 API의 지연은 곧 worker 점유 시간이고, worker 점유 시간은 서버가 감당할 수 있는 동시 요청 수와 직접 연결된다.

Servlet async는 처리 도중 container worker를 반환할 수 있지만 별도 Executor와 열린 연결을 관리해야 한다.

가상 스레드는 blocking I/O가 많은 thread-per-request 코드의 확장성을 높일 수 있지만 downstream 자원의 한도를 없애 주지는 않는다.

결국 Tomcat thread를 이해한다는 것은 단순히 `maxThreads` 값을 외우는 일이 아니다.

연결, 실행 작업, worker, queue, downstream 자원이 어디에서 기다리고 어떤 한도로 보호되는지 이해하는 일이다.

## 참고 자료

- [Apache Tomcat 10.1 HTTP Connector](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html)
- [Apache Tomcat 10.1 Executor](https://tomcat.apache.org/tomcat-10.1-doc/config/executor.html)
- [Apache Tomcat 10.1 NioEndpoint API](https://tomcat.apache.org/tomcat-10.1-doc/api/org/apache/tomcat/util/net/NioEndpoint.html)
- [Apache Tomcat TaskQueue source](https://github.com/apache/tomcat/blob/10.1.x/java/org/apache/tomcat/util/threads/TaskQueue.java)
- [Spring MVC Asynchronous Requests](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html)
- [Spring Boot Application Properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html)
- [Oracle Java Virtual Threads](https://docs.oracle.com/en/java/javase/25/core/virtual-threads.html)
