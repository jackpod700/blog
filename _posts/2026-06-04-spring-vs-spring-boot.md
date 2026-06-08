---
title: 기존 Spring과 비교하면 Spring Boot는 무엇이 다를까
date: 2026-06-04 00:00:00 +0900
categories: [Spring]
tags: [spring, spring-boot, java, backend, auto-configuration, embedded-server]
---

## Spring Boot는 Spring을 대체하는 것이 아니다

Spring Boot를 처음 보면 Spring과 완전히 다른 프레임워크처럼 느껴질 수 있다.

하지만 Spring Boot는 Spring을 대체하는 기술이 아니다.

Spring Boot는 기존 Spring을 더 쉽게 시작하고 운영할 수 있게 만들어 주는 도구에 가깝다.

핵심 기능은 여전히 Spring이 제공한다.

- IoC 컨테이너
- DI
- Bean 생명주기 관리
- AOP
- 트랜잭션
- Spring MVC
- 데이터 접근

Spring Boot는 이 기능들을 더 적은 설정으로 사용할 수 있게 해 준다.

즉, 관계를 한 문장으로 정리하면 다음과 같다.

```text
Spring = 애플리케이션의 핵심 프레임워크
Spring Boot = Spring 애플리케이션을 빠르게 만들고 간편하게 실행할 수 있게 해 주는 도구
```

## 기존 Spring의 불편함

기존 Spring은 강력하지만, 프로젝트를 시작하려면 직접 설정해야 할 것이 많았다.

예를 들어 Spring MVC 웹 애플리케이션을 만들려면 다음과 같은 작업이 필요했다.

- 필요한 라이브러리 의존성 추가
- 라이브러리 버전 맞추기
- `DispatcherServlet` 등록
- Spring 설정 파일 작성
- 컴포넌트 스캔 설정
- ViewResolver 설정
- JSON 변환기 설정
- 외부 Tomcat 배포 설정

Java 설정을 사용하면 다음처럼 직접 컨테이너 설정을 작성해야 한다.

```java
@Configuration
@ComponentScan(basePackages = "com.example.blog")
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
}
```

그리고 웹 환경에서는 `DispatcherServlet`도 등록해야 한다.

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

이 설정들은 한 번 이해하면 명확하지만, 매 프로젝트마다 반복하기에는 번거롭다.

Spring Boot는 바로 이 반복 설정을 줄이는 데서 출발한다.

## Spring Boot의 시작 코드

Spring Boot 애플리케이션은 보통 다음 코드로 시작한다.

```java
@SpringBootApplication
public class BlogApplication {
    public static void main(String[] args) {
        SpringApplication.run(BlogApplication.class, args);
    }
}
```

겉으로 보면 매우 짧다.

하지만 이 안에는 많은 설정이 숨어 있다.

`@SpringBootApplication`은 대략 다음 세 어노테이션을 합친 것이다.

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

각 역할은 다음과 같다.

- `@SpringBootConfiguration`: 이 클래스가 Spring 설정 클래스임을 의미한다.
- `@EnableAutoConfiguration`: 현재 의존성과 설정을 보고 필요한 Bean을 자동으로 등록한다.
- `@ComponentScan`: 현재 패키지와 하위 패키지에서 컴포넌트를 찾는다.

결국 Spring Boot의 시작 코드는 짧지만, 내부적으로는 기존 Spring 설정을 자동으로 구성한다.

## 차이 1. 설정 방식

기존 Spring에서는 필요한 설정을 개발자가 직접 명시하는 경우가 많았다.

```java
@Configuration
@EnableWebMvc
@ComponentScan(basePackages = "com.example.blog")
public class WebConfig {
}
```

Spring Boot에서는 기본 설정을 자동으로 제공한다.

```java
@SpringBootApplication
public class BlogApplication {
}
```

예를 들어 `spring-boot-starter-web` 의존성을 추가하면 Spring Boot는 웹 애플리케이션이라고 판단하고 필요한 설정을 자동으로 적용한다.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-web'
```

이 의존성 하나로 다음 기능들이 함께 준비된다.

- Spring MVC
- 내장 Tomcat
- Jackson JSON 변환
- 기본 에러 처리
- 웹 관련 자동 설정

기존 Spring에서는 각각의 설정을 직접 맞춰야 했다면, Spring Boot에서는 대부분 기본값으로 시작할 수 있다.

## 차이 2. 의존성 관리

기존 Spring에서는 라이브러리 버전 충돌을 직접 신경 써야 하는 일이 많았다.

예를 들어 Spring MVC, Jackson, Validation, Tomcat, Logging 라이브러리를 함께 사용할 때 서로 호환되는 버전을 골라야 했다.

Spring Boot는 `starter`와 의존성 관리 기능을 제공한다.

대표적인 starter는 다음과 같다.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
implementation 'org.springframework.boot:spring-boot-starter-security'
implementation 'org.springframework.boot:spring-boot-starter-test'
```

`starter`는 특정 기능에 필요한 라이브러리 묶음이다.

예를 들어 `spring-boot-starter-data-jpa`를 추가하면 JPA 개발에 필요한 주요 라이브러리가 함께 들어온다.

```text
spring-boot-starter-data-jpa
 ├─ Spring Data JPA
 ├─ Hibernate
 ├─ JDBC 관련 라이브러리
 └─ 트랜잭션 관련 설정
```

개발자는 필요한 기능의 starter를 고르면 되고, 세부 라이브러리 버전은 Spring Boot가 관리한다.

## 차이 3. 자동 설정

Spring Boot의 가장 중요한 특징은 자동 설정이다.

자동 설정은 현재 프로젝트에 어떤 라이브러리가 있는지, 어떤 설정값이 있는지, 사용자가 직접 등록한 Bean이 있는지를 보고 필요한 설정을 자동으로 적용하는 기능이다.

예를 들어 클래스패스에 Spring MVC가 있으면 Spring Boot는 웹 MVC 설정을 준비한다.

데이터베이스 드라이버와 datasource 설정이 있으면 `DataSource`를 만든다.

JPA가 있으면 `EntityManagerFactory`와 `TransactionManager`를 구성한다.

대략 이런 방식으로 판단한다.

```text
웹 라이브러리가 있네?
 -> Spring MVC 설정
 -> DispatcherServlet 등록
 -> JSON 변환기 등록

JPA 라이브러리가 있네?
 -> EntityManagerFactory 설정
 -> TransactionManager 설정
 -> Repository 스캔 설정
```

하지만 자동 설정은 무조건 밀어붙이는 방식이 아니다.

사용자가 직접 Bean을 등록하면 Spring Boot는 많은 경우 그 설정을 우선한다.

예를 들어 사용자가 직접 `ObjectMapper` Bean을 만들면, Spring Boot의 기본 `ObjectMapper` 설정 대신 사용자의 Bean이 사용될 수 있다.

```java
@Bean
public ObjectMapper objectMapper() {
    return new ObjectMapper()
        .registerModule(new JavaTimeModule());
}
```

Spring Boot의 철학은 다음에 가깝다.

```text
기본값은 제공한다.
하지만 사용자가 직접 정하면 그 선택을 존중한다.
```

## 차이 4. 내장 서버

기존 Spring 웹 애플리케이션은 보통 WAR 파일로 패키징해서 외부 WAS에 배포했다.

```text
애플리케이션 WAR 생성
 -> 외부 Tomcat 설치
 -> Tomcat webapps에 WAR 배포
 -> 서버 실행
```

Spring Boot는 내장 서버 방식을 기본으로 사용한다.

`spring-boot-starter-web`을 사용하면 기본적으로 내장 Tomcat이 포함된다.

그래서 애플리케이션을 JAR 파일로 실행할 수 있다.

```bash
java -jar blog.jar
```

흐름은 다음처럼 바뀐다.

```text
애플리케이션 JAR 실행
 -> 내장 Tomcat 실행
 -> Spring 컨테이너 시작
 -> 요청 처리
```

이 방식은 배포를 단순하게 만든다.

서버에 Tomcat을 따로 설치하고 애플리케이션을 올리는 대신, 애플리케이션 자체가 서버를 포함해서 실행된다.

물론 Spring Boot도 WAR 배포가 가능하다.

하지만 기본 방향은 독립 실행 가능한 애플리케이션이다.

## 차이 5. 설정 파일

기존 Spring에서는 XML이나 Java Config에 많은 설정을 작성했다.

Spring Boot에서는 환경 설정을 주로 `application.properties`나 `application.yml`에 작성한다.

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/blog
    username: root
    password: password
```

이 설정값을 보고 Spring Boot의 자동 설정이 동작한다.

예를 들어 `server.port`를 바꾸면 내장 서버 포트가 바뀐다.

`spring.datasource.url`을 지정하면 DataSource 설정에 사용된다.

기존 Spring에서는 Bean과 설정을 직접 조립하는 느낌이 강했다면, Spring Boot에서는 설정값을 제공하면 자동 설정이 그 값을 읽어 필요한 구성을 완성하는 느낌에 가깝다.

## 차이 6. 운영 기능

Spring Boot는 애플리케이션 운영에 필요한 기능도 쉽게 붙일 수 있다.

대표적인 것이 Actuator다.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

Actuator를 사용하면 애플리케이션 상태를 확인할 수 있는 엔드포인트를 제공한다.

```text
/actuator/health
/actuator/metrics
/actuator/info
```

예를 들어 `/actuator/health`는 애플리케이션이 정상 상태인지 확인하는 데 사용할 수 있다.

```json
{
  "status": "UP"
}
```

기존 Spring에서도 이런 기능을 직접 구성할 수는 있다.

하지만 Spring Boot는 운영에 자주 필요한 기능을 표준화해서 쉽게 사용할 수 있게 해 준다.

## 비교 표

기존 Spring과 Spring Boot의 차이를 표로 정리하면 다음과 같다.

| 구분 | 기존 Spring | Spring Boot |
| --- | --- | --- |
| 목적 | Spring 기반 애플리케이션 개발 | Spring 애플리케이션을 빠르게 시작하고 실행 |
| 설정 | 직접 설정이 많음 | 자동 설정 제공 |
| 의존성 | 버전 관리 직접 처리 | starter와 BOM으로 관리 |
| 서버 | 외부 WAS 배포가 일반적 | 내장 서버로 독립 실행 가능 |
| 실행 | WAR 배포 중심 | JAR 실행 중심 |
| 설정 파일 | XML, Java Config 중심 | `application.yml`, `application.properties` 중심 |
| 운영 기능 | 직접 구성 필요 | Actuator 등 기본 지원 |
| 핵심 기술 | IoC, DI, AOP, MVC | Spring의 핵심 기술 위에 편의 기능 추가 |

## Spring Boot가 해 주는 일

Spring Boot가 해 주는 일을 더 직접적으로 표현하면 다음과 같다.

```text
1. 필요한 라이브러리 묶음을 starter로 제공한다.
2. 라이브러리 버전을 맞춰 준다.
3. 클래스패스와 설정값을 보고 자동 설정을 적용한다.
4. 내장 서버를 실행한다.
5. 기본 로깅과 에러 처리를 제공한다.
6. 운영용 엔드포인트를 쉽게 붙일 수 있게 한다.
```

하지만 중요한 점이 있다.

Spring Boot가 Spring의 동작 원리를 없애는 것은 아니다.

Spring Boot 애플리케이션 안에서도 여전히 Spring 컨테이너가 Bean을 만들고, DI를 수행하고, AOP 프록시를 만들고, Spring MVC가 요청을 처리한다.

Spring Boot는 그 과정을 더 쉽게 시작하게 해 줄 뿐이다.

## Spring Boot를 쓰면 Spring을 몰라도 될까

Spring Boot는 많은 설정을 자동으로 처리해 준다.

그래서 처음에는 Spring의 내부 원리를 몰라도 애플리케이션을 만들 수 있다.

하지만 문제가 생기면 결국 Spring의 원리를 알아야 한다.

예를 들어 다음과 같은 상황이 있다.

- Bean이 등록되지 않는다.
- 의존성 주입이 실패한다.
- `@Transactional`이 적용되지 않는다.
- 자동 설정이 예상과 다르게 동작한다.
- 어떤 Bean이 기본 설정을 덮어썼는지 모르겠다.
- 요청이 Controller까지 도달하지 않는다.

이런 문제를 해결하려면 Spring Boot의 편의 기능뿐 아니라 Spring 컨테이너, Bean, DI, AOP, MVC 흐름을 이해해야 한다.

즉, Spring Boot는 Spring을 몰라도 되게 만드는 도구가 아니라, Spring을 더 빠르게 사용할 수 있게 해 주는 도구다.

## 정리

Spring은 애플리케이션의 핵심 구조를 제공한다.

객체를 만들고, 의존성을 연결하고, 생명주기를 관리하고, 트랜잭션과 MVC 흐름을 처리한다.

Spring Boot는 그 Spring을 더 쉽게 시작하고 배포하고 운영할 수 있게 해 준다.

기존 Spring에서 직접 설정하던 많은 부분을 자동 설정과 starter, 내장 서버, 설정 파일, Actuator 같은 기능으로 단순화한다.

그래서 둘의 관계는 경쟁 관계가 아니다.

```text
Spring 위에서 Spring Boot가 동작한다.
```

Spring을 이해하면 Spring Boot가 자동으로 해 주는 일이 보이고, Spring Boot를 사용하면 Spring 애플리케이션을 훨씬 빠르게 만들 수 있다.
