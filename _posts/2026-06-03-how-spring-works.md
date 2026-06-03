---
title: Spring은 처음부터 끝까지 어떻게 작동할까
date: 2026-06-03 00:00:00 +0900
categories: [Spring]
tags: [spring, java, backend, ioc, di, mvc, aop]
---

## Spring을 한 문장으로 보면

Spring은 Java 애플리케이션에서 객체 생성, 의존성 연결, 생명주기 관리, 트랜잭션, 웹 요청 처리 같은 공통 흐름을 컨테이너 중심으로 관리해 주는 프레임워크다.

개발자가 모든 객체를 직접 만들고 연결하는 대신, Spring 컨테이너가 애플리케이션에 필요한 객체를 만들고 보관하고 필요한 곳에 주입한다.

Spring의 핵심은 다음 두 가지다.

- IoC: 객체 생성과 제어 흐름의 주도권을 Spring 컨테이너가 가진다.
- DI: 객체가 필요로 하는 의존성을 Spring 컨테이너가 넣어 준다.

여기서는 Spring Boot가 아니라 순수 Spring을 기준으로, 애플리케이션이 시작되고 요청을 처리하기까지의 흐름을 따라가 본다.

## 1. Spring 컨테이너가 시작된다

Spring 애플리케이션은 먼저 컨테이너를 생성하면서 시작된다.

순수 Java 설정을 사용한다면 다음처럼 시작할 수 있다.

```java
ApplicationContext context =
    new AnnotationConfigApplicationContext(AppConfig.class);
```

웹 애플리케이션에서는 보통 WAS가 애플리케이션을 시작하면서 Spring의 웹 컨텍스트를 만든다.

```text
Tomcat 같은 WAS 시작
 -> web.xml 또는 WebApplicationInitializer 실행
 -> Spring 컨테이너 생성
```

Spring 컨테이너는 `ApplicationContext`라고 부른다.

`ApplicationContext`는 Bean을 생성하고, 의존성을 주입하고, 이벤트를 발행하고, 메시지나 리소스 같은 부가 기능도 제공한다.

즉, Spring 애플리케이션의 중심은 `ApplicationContext`다.

## 2. 설정 정보를 읽는다

컨테이너가 만들어지면 Spring은 설정 정보를 읽는다.

Spring 설정은 여러 방식으로 작성할 수 있다.

```java
@Configuration
public class AppConfig {
    @Bean
    public PostService postService() {
        return new PostService(postRepository());
    }

    @Bean
    public PostRepository postRepository() {
        return new MemoryPostRepository();
    }
}
```

또는 컴포넌트 스캔을 사용할 수도 있다.

```java
@Configuration
@ComponentScan(basePackages = "com.example.blog")
public class AppConfig {
}
```

예전 방식으로는 XML 설정도 가능하다.

```xml
<bean id="postService" class="com.example.blog.PostService"/>
```

방식은 달라도 목적은 같다.

Spring에게 "어떤 객체를 Bean으로 등록할 것인지", "객체끼리 어떻게 연결할 것인지" 알려 주는 것이다.

## 3. BeanDefinition을 만든다

Spring은 설정 정보를 읽자마자 바로 객체를 만드는 것이 아니다.

먼저 Bean을 만들기 위한 설계도인 `BeanDefinition`을 만든다.

`BeanDefinition`에는 이런 정보가 들어 있다.

- Bean으로 등록할 클래스
- Bean 이름
- scope
- 생성자 인자
- 의존성 정보
- 초기화 메서드
- 지연 초기화 여부

예를 들어 다음 클래스가 있다고 하자.

```java
@Service
public class PostService {
}
```

Spring은 이 클래스를 보고 대략 다음과 같은 정보를 만든다.

```text
Bean name: postService
Bean class: PostService
Scope: singleton
```

이 단계는 실제 객체를 만드는 단계가 아니라, 객체를 만들기 위한 메타데이터를 정리하는 단계다.

## 4. BeanFactoryPostProcessor가 실행된다

BeanDefinition이 준비되면 Spring은 Bean을 만들기 전에 BeanDefinition을 수정할 기회를 제공한다.

이때 실행되는 대표적인 확장 지점이 `BeanFactoryPostProcessor`다.

예를 들어 `PropertySourcesPlaceholderConfigurer`는 다음과 같은 값을 실제 설정값으로 치환할 수 있게 해 준다.

```java
@Value("${db.url}")
private String dbUrl;
```

즉, Spring은 객체를 만들기 전에 설정 메타데이터를 한 번 더 가공할 수 있다.

이 덕분에 Spring은 단순한 객체 생성기가 아니라, 설정을 해석하고 확장할 수 있는 컨테이너가 된다.

## 5. Bean을 생성한다

이제 Spring은 BeanDefinition을 바탕으로 실제 객체를 생성한다.

기본적으로 Spring Bean은 singleton scope다.

즉, 컨테이너 안에 하나만 만들어지고 여러 곳에서 공유된다.

```text
Spring Container
 ├─ postController
 ├─ postService
 └─ postRepository
```

예를 들어 다음 코드가 있다고 하자.

```java
@Controller
public class PostController {
    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }
}
```

```java
@Service
public class PostService {
    private final PostRepository postRepository;

    public PostService(PostRepository postRepository) {
        this.postRepository = postRepository;
    }
}
```

```java
@Repository
public class PostRepository {
}
```

Spring은 의존 관계를 분석해서 필요한 순서로 객체를 만든다.

```text
PostController
 └─ PostService
     └─ PostRepository
```

실제 생성 순서는 다음과 비슷하다.

```text
1. PostRepository 생성
2. PostService 생성
3. PostController 생성
```

## 6. 의존성을 주입한다

객체가 만들어질 때 Spring은 필요한 의존성을 넣어 준다.

이것이 DI, Dependency Injection이다.

가장 권장되는 방식은 생성자 주입이다.

```java
@Service
public class PostService {
    private final PostRepository postRepository;

    public PostService(PostRepository postRepository) {
        this.postRepository = postRepository;
    }
}
```

`PostService`는 `PostRepository`가 필요하지만 직접 만들지 않는다.

```java
new PostRepository()
```

대신 생성자를 통해 외부에서 받는다.

```java
public PostService(PostRepository postRepository) {
    this.postRepository = postRepository;
}
```

Spring 컨테이너 안에 `PostRepository` Bean이 있으면 Spring은 그 Bean을 찾아서 `PostService` 생성자에 넣어 준다.

생성자 주입의 장점은 이러한 것들이 있다.

- 필요한 의존성이 코드에 명확히 드러난다.
- `final` 필드를 사용할 수 있다.
- 테스트할 때 가짜 객체를 넣기 쉽다.
- 객체가 불완전한 상태로 생성되는 일을 줄일 수 있다.

## 7. Bean 생명주기가 진행된다

Spring Bean은 단순히 생성되고 끝나지 않는다.

대략 다음 생명주기를 거친다.

```text
1. 객체 생성
2. 의존성 주입
3. Aware 인터페이스 처리
4. BeanPostProcessor 전처리
5. 초기화 메서드 실행
6. BeanPostProcessor 후처리
7. Bean 사용
8. 컨테이너 종료 시 소멸 메서드 실행
```

예를 들어 초기화 작업이 필요하면 `@PostConstruct`를 사용할 수 있다.

```java
@Component
public class CacheStore {
    @PostConstruct
    public void init() {
        // Bean 생성 후 초기화 작업
    }
}
```

컨테이너가 종료될 때 정리 작업이 필요하면 `@PreDestroy`를 사용할 수 있다.

```java
@PreDestroy
public void close() {
    // 리소스 정리
}
```

Spring은 이렇게 Bean의 시작과 끝을 관리한다.

## 8. BeanPostProcessor가 부가 기능을 붙인다

Spring의 강력한 기능 중 하나는 Bean 생성 전후에 개입할 수 있다는 점이다.

이때 사용되는 대표적인 확장 지점이 `BeanPostProcessor`다.

`BeanPostProcessor`는 Bean 초기화 전후에 실행되며, 필요하다면 원래 Bean 대신 다른 객체를 반환할 수도 있다.

이 구조 덕분에 AOP 프록시 같은 기능이 가능해진다.

예를 들어 `@Transactional`이 붙은 서비스가 있다고 하자.

```java
@Service
public class PostService {
    @Transactional
    public void createPost(Post post) {
        // 게시글 저장
    }
}
```

Spring은 이 Bean을 그대로 사용하는 대신 프록시 객체로 감쌀 수 있다.

```text
Controller
 -> PostService 프록시
     -> 트랜잭션 시작
     -> 실제 PostService.createPost()
     -> 트랜잭션 커밋
```

예외가 발생하면 롤백한다.

```text
Controller
 -> PostService 프록시
     -> 트랜잭션 시작
     -> 실제 PostService.createPost()
     -> 예외 발생
     -> 트랜잭션 롤백
```

개발자는 비즈니스 로직에 집중하고, 트랜잭션 같은 공통 기능은 Spring이 프록시를 통해 처리한다.

## 9. 웹 애플리케이션에서는 DispatcherServlet이 준비된다

Spring MVC를 사용하는 웹 애플리케이션에서는 `DispatcherServlet`이 핵심이다.

`DispatcherServlet`은 모든 HTTP 요청을 받아 적절한 Controller로 보내는 프론트 컨트롤러다.

순수 Spring MVC에서는 보통 `web.xml`에 등록한다.

```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

또는 Java 코드로 등록할 수도 있다.

```java
public class WebAppInitializer implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext servletContext) {
        AnnotationConfigWebApplicationContext context =
            new AnnotationConfigWebApplicationContext();

        context.register(WebConfig.class);

        DispatcherServlet dispatcherServlet = new DispatcherServlet(context);

        ServletRegistration.Dynamic registration =
            servletContext.addServlet("dispatcher", dispatcherServlet);

        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```

중요한 점은 웹 요청의 중심에 `DispatcherServlet`이 있다는 것이다.

## 10. HTTP 요청이 Controller로 전달된다

사용자가 다음 요청을 보낸다고 하자.

```http
GET /posts/1
```

요청은 먼저 WAS로 들어온다.

그리고 WAS는 URL 매핑에 따라 요청을 `DispatcherServlet`으로 전달한다.

```text
Client
 -> Tomcat
 -> DispatcherServlet
```

`DispatcherServlet`은 요청을 처리할 Controller 메서드를 찾는다.

이때 `HandlerMapping`이 사용된다.

```java
@Controller
public class PostController {
    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }

    @GetMapping("/posts/{id}")
    @ResponseBody
    public PostResponse findPost(@PathVariable Long id) {
        return postService.findPost(id);
    }
}
```

`GET /posts/1` 요청은 다음 메서드와 매핑된다.

```java
@GetMapping("/posts/{id}")
public PostResponse findPost(@PathVariable Long id)
```

그리고 `{id}` 값은 메서드 파라미터로 바인딩된다.

```text
/posts/1
       └─ id = 1
```

## 11. HandlerAdapter가 메서드를 실행한다

Controller 메서드를 찾은 뒤에는 `HandlerAdapter`가 실제 메서드를 호출한다.

`DispatcherServlet`이 직접 모든 종류의 Controller를 실행하는 것이 아니라, 실행 방법을 아는 어댑터에게 맡기는 구조다.

흐름은 다음과 같다.

```text
DispatcherServlet
 -> HandlerMapping으로 실행할 핸들러 찾기
 -> HandlerAdapter로 핸들러 실행
 -> Controller 메서드 호출
```

Controller는 요청 처리에 필요한 값을 파라미터로 받고, Service를 호출한다.

```java
@ResponseBody
@GetMapping("/posts/{id}")
public PostResponse findPost(@PathVariable Long id) {
    return postService.findPost(id);
}
```

## 12. Service와 Repository로 흐른다

Controller는 HTTP 요청과 응답에 가까운 역할을 맡고, 실제 비즈니스 로직은 보통 Service에서 처리한다.

```java
@Service
public class PostService {
    private final PostRepository postRepository;

    public PostResponse findPost(Long id) {
        Post post = postRepository.findById(id);
        return new PostResponse(post.getId(), post.getTitle(), post.getContent());
    }
}
```

Repository는 데이터 접근을 담당한다.

```java
@Repository
public class PostRepository {
    public Post findById(Long id) {
        // JDBC, MyBatis, JPA 등을 사용해 DB 조회
        return SomePost;
    }
}
```

전체 흐름은 다음처럼 이어진다.

```text
Controller
 -> Service
 -> Repository
 -> Database
```

Spring은 이 계층들을 강제하지는 않는다.

하지만 `Controller`, `Service`, `Repository`로 역할을 나누면 요청 처리, 비즈니스 로직, 데이터 접근의 책임이 분리된다.

## 13. 트랜잭션이 적용된다

데이터를 변경하는 작업에서는 트랜잭션이 필요하다.

Spring에서는 보통 `@Transactional`을 사용한다.

```java
@Service
public class PostService {
    private final PostRepository postRepository;

    @Transactional
    public void updatePost(Long id, String title) {
        Post post = postRepository.findById(id);
        post.changeTitle(title);
    }
}
```

`@Transactional`은 AOP 기반으로 동작한다.

Spring은 `PostService`를 프록시로 감싸고, 메서드 호출 전후에 트랜잭션 처리를 넣는다.

```text
트랜잭션 시작
 -> updatePost() 실행
 -> 정상 종료
 -> 커밋
```

예외가 발생하면 다음처럼 처리된다.

```text
트랜잭션 시작
 -> updatePost() 실행
 -> 예외 발생
 -> 롤백
```

중요한 점은 `@Transactional`이 단순한 표시가 아니라, Spring 컨테이너가 만든 프록시를 통해 실행된다는 것이다.

그래서 같은 클래스 내부에서 자기 자신의 `@Transactional` 메서드를 직접 호출하면 프록시를 거치지 않아 트랜잭션이 적용되지 않을 수 있다.

## 14. 응답을 만든다

Controller 메서드가 값을 반환하면 Spring MVC는 그 값을 HTTP 응답으로 바꾼다.

`@ResponseBody`가 붙어 있으면 반환 객체가 응답 본문에 들어간다.

```java
@ResponseBody
@GetMapping("/posts/{id}")
public PostResponse findPost(@PathVariable Long id) {
    return postService.findPost(id);
}
```

반환 객체가 다음과 같다면,

```java
public record PostResponse(
    Long id,
    String title,
    String content
) {
}
```

Spring MVC는 `HttpMessageConverter`를 사용해 객체를 JSON 같은 형식으로 변환한다.

```json
{
  "id": 1,
  "title": "Spring은 어떻게 작동할까",
  "content": "Spring의 흐름을 따라가 보자."
}
```

만약 `@ResponseBody`가 없다면 보통 반환값은 View 이름으로 해석된다.

```java
@GetMapping("/posts")
public String posts() {
    return "posts/list";
}
```

이 경우 Spring MVC는 `ViewResolver`를 통해 실제 View를 찾고 렌더링한다.

## 전체 흐름 다시 보기

순수 Spring 애플리케이션의 시작 흐름은 다음과 같다.

```text
1. ApplicationContext 생성
2. 설정 클래스 또는 XML 읽기
3. BeanDefinition 생성
4. BeanFactoryPostProcessor 실행
5. Bean 생성
6. 의존성 주입
7. Aware 인터페이스 처리
8. BeanPostProcessor 전처리
9. 초기화 메서드 실행
10. BeanPostProcessor 후처리
11. 프록시 적용
12. 애플리케이션 실행 준비 완료
```

Spring MVC 요청 흐름은 다음과 같다.

```text
Client
 -> WAS
 -> DispatcherServlet
 -> HandlerMapping
 -> HandlerAdapter
 -> Controller
 -> Service
 -> Repository
 -> Database
 -> Repository
 -> Service
 -> Controller
 -> HttpMessageConverter 또는 ViewResolver
 -> Response
 -> Client
```

## Spring이 해결하는 문제

Spring의 구조는 처음에는 복잡해 보인다.

하지만 Spring이 해결하려는 문제를 보면 방향이 분명하다.

객체 생성과 의존성 연결을 컨테이너가 맡으면, 개발자는 객체 사이의 관계를 직접 조립하는 코드에서 벗어날 수 있다.

Bean 생명주기를 컨테이너가 관리하면, 초기화와 종료 시점의 작업을 일관되게 처리할 수 있다.

AOP와 프록시를 사용하면 트랜잭션, 로깅, 보안 같은 공통 기능을 비즈니스 로직에서 분리할 수 있다.

Spring MVC를 사용하면 HTTP 요청이 Controller까지 도달하고 응답으로 바뀌는 흐름을 표준화할 수 있다.

결국 Spring은 객체를 잘 만들고, 잘 연결하고, 필요한 시점에 필요한 기능을 끼워 넣는 프레임워크다.

## 정리

Spring은 단순히 어노테이션을 붙이는 도구가 아니다.

컨테이너가 설정 정보를 읽고, BeanDefinition을 만들고, Bean을 생성하고, 의존성을 주입하고, 생명주기를 관리한다.

실행 중에는 프록시를 통해 트랜잭션 같은 공통 기능을 적용하고, Spring MVC에서는 `DispatcherServlet`을 중심으로 HTTP 요청을 Controller까지 전달한다.

이 흐름을 이해하면 `@Component`, `@Autowired`, `@Transactional`, `@Controller` 같은 어노테이션이 단순한 암기 대상이 아니라 Spring 실행 흐름 안에서 어떤 역할을 하는지 보이기 시작한다.

Spring은 마법이 아니라 컨테이너, Bean, 프록시, MVC 흐름이 맞물려 돌아가는 구조다.
