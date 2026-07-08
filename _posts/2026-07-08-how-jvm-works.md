---
title: JVM은 자바 코드를 어떻게 실행할까
date: 2026-07-08 00:00:00 +0900
categories: [Java]
tags: [java, jvm, bytecode, classloader, memory, gc, jit]
---

## JVM을 한 문장으로 보면

JVM(Java Virtual Machine)은 `.class` 파일에 들어 있는 바이트코드를 읽고, 검증하고, 메모리에 올리고, 실제 운영체제 위에서 실행해 주는 자바의 런타임 환경이다.

자바 코드는 운영체제가 바로 실행하는 코드가 아니다.

먼저 `javac`가 `.java` 파일을 `.class` 파일로 컴파일하고, JVM이 그 바이트코드를 실행한다.

```text
Java source code
 -> javac
 -> Java bytecode
 -> JVM
 -> OS / CPU
```

그래서 자바는 같은 소스 코드를 운영체제마다 다시 작성하지 않아도 된다.

운영체제마다 JVM 구현체는 다르지만, JVM이 이해하는 바이트코드 형식은 같기 때문이다.

```text
Hello.java
 -> Hello.class
 -> Windows JVM
 -> Windows에서 실행

Hello.java
 -> Hello.class
 -> Linux JVM
 -> Linux에서 실행
```

흔히 말하는 "Write Once, Run Anywhere"는 이 구조에서 나온다.

## JDK, JRE, JVM의 관계

JVM을 볼 때 자주 같이 나오는 단어가 JDK와 JRE다.

개념적으로 정리하면 다음과 같다.

```text
JDK = 개발 도구 + 실행 환경
JRE = 실행 환경
JVM = 바이트코드를 실행하는 가상 머신
```

JDK에는 `javac`, `java`, 디버깅 도구, 표준 라이브러리, JVM 등이 들어 있다.

개발자는 보통 JDK를 설치한다.

`javac`로 코드를 컴파일하고, `java` 명령으로 JVM을 실행하기 때문이다.

```bash
javac Hello.java
java Hello
```

여기서 `java Hello`를 실행하면 JVM이 시작되고, JVM이 `Hello.class`를 찾아 실행한다.

## JVM 전체 구조 먼저 보기

이 글에서 다룰 JVM의 큰 흐름을 그림으로 보면 다음과 같다.

![JVM 전체 구조](/assets/img/posts/2026-07-08-jvm-architecture.svg)

왼쪽에서 `.java` 파일이 `.class` 바이트코드로 컴파일되고, JVM 안에서는 ClassLoader, Runtime Data Area, Execution Engine, Garbage Collector가 함께 동작한다.

이후 섹션에서는 이 그림의 각 영역을 하나씩 따라가며 살펴본다.

## 1. 자바 코드는 바이트코드로 컴파일된다

다음과 같은 코드가 있다고 하자.

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello JVM");
    }
}
```

이 코드는 바로 CPU가 이해하는 기계어로 바뀌지 않는다.

`javac`는 이 코드를 JVM이 이해할 수 있는 바이트코드로 바꾼다.

```bash
javac Hello.java
```

그러면 `Hello.class` 파일이 생긴다.

`.class` 파일 안에는 JVM 명령어가 들어 있다.

예를 들어 `javap -c Hello`로 바이트코드를 확인하면 다음과 비슷한 흐름을 볼 수 있다.

```text
0: getstatic
3: ldc
5: invokevirtual
8: return
```

자바 소스 코드의 `System.out.println("Hello JVM")`이 JVM이 실행할 수 있는 명령어들로 바뀐 것이다.

즉, JVM은 `.java` 파일을 직접 실행하지 않는다.

JVM이 실행하는 대상은 컴파일된 `.class` 파일이다.

## 2. ClassLoader가 클래스를 메모리에 올린다

JVM이 시작되면 모든 클래스를 한 번에 전부 메모리에 올리지 않는다.

필요한 클래스가 사용되는 시점에 JVM은 ClassLoader를 통해 클래스를 찾고 읽는다.

```text
JVM 실행
 -> main class 필요
 -> ClassLoader가 class 파일 탐색
 -> class 파일을 읽어서 JVM 메모리에 로드
```

ClassLoader는 보통 다음 계층으로 이해할 수 있다.

```text
Bootstrap ClassLoader
 -> Platform ClassLoader
 -> Application ClassLoader
 -> Custom ClassLoader
```

각 역할은 다음과 같다.

- Bootstrap ClassLoader: Java의 핵심 클래스들을 로드한다.
- Platform ClassLoader: Java 플랫폼 관련 클래스들을 로드한다.
- Application ClassLoader: 애플리케이션의 classpath 또는 module path에 있는 클래스들을 로드한다.
- Custom ClassLoader: 프레임워크나 서버가 특별한 방식으로 클래스를 로드할 때 사용한다.

클래스를 찾을 때는 보통 부모 ClassLoader에게 먼저 위임한다.

```text
Application ClassLoader
 -> Platform ClassLoader에게 먼저 요청
 -> Bootstrap ClassLoader에게 먼저 요청
 -> 부모가 못 찾으면 자신이 탐색
```

이 방식을 부모 위임 모델이라고 한다.

덕분에 애플리케이션에서 `java.lang.String` 같은 핵심 클래스를 마음대로 바꿔치기하기 어렵다.

## 3. 클래스는 로딩, 링킹, 초기화를 거친다

ClassLoader가 클래스 파일을 읽었다고 해서 바로 사용할 수 있는 것은 아니다.

JVM은 클래스를 다음 흐름으로 준비한다.

```text
Loading
 -> Linking
 -> Initialization
```

### Loading

Loading은 `.class` 파일을 읽고 JVM 내부에서 클래스 정보로 만드는 단계다.

예를 들어 `PostService.class` 파일을 읽으면 JVM은 이 클래스의 이름, 부모 클래스, 메서드, 필드 같은 정보를 파악한다.

그리고 Java 코드에서 사용할 수 있는 `Class` 객체도 준비한다.

```java
Class<PostService> clazz = PostService.class;
```

### Linking

Linking은 읽어 온 클래스가 JVM 안에서 실행될 수 있도록 연결하는 단계다.

세부적으로는 다음 과정이 있다.

- Verification: 바이트코드가 JVM 규칙에 맞고 안전한지 검사한다.
- Preparation: static 필드에 필요한 메모리를 준비하고 기본값을 넣는다.
- Resolution: 심볼릭 참조를 실제 참조로 바꾼다.

예를 들어 다음 코드가 있다고 하자.

```java
public class Counter {
    static int count = 10;
}
```

Preparation 단계에서는 `count`에 먼저 기본값인 `0`이 들어간다.

그 다음 Initialization 단계에서 코드에 적힌 값인 `10`이 들어간다.

### Initialization

Initialization은 클래스의 static 초기화 코드가 실행되는 단계다.

```java
public class AppConfig {
    static String profile = loadProfile();

    static {
        System.out.println("AppConfig initialized");
    }
}
```

이 단계에서 `loadProfile()`이 실행되고, static block도 실행된다.

즉, 클래스가 JVM에 로드되는 것과 static 초기화 코드가 실행되는 것은 같은 말이 아니다.

클래스는 로드된 뒤 검증과 준비 과정을 거치고, 실제 사용 시점에 초기화된다.

## 4. JVM은 Runtime Data Area를 사용한다

클래스가 준비되면 JVM은 코드를 실행하기 위해 여러 메모리 영역을 사용한다.

이를 Runtime Data Area라고 부른다.

큰 관점에서 보면 모든 스레드가 공유하는 영역과 스레드마다 따로 가지는 영역으로 나눌 수 있다.

```text
Runtime Data Area

shared by all threads:
  - Heap
  - Method Area

created per thread:
  - Java Stack
  - PC Register
  - Native Method Stack
```

## 5. Heap에는 객체가 저장된다

`new`로 만든 객체와 배열은 Heap에 저장된다.

```java
Post post = new Post("JVM");
```

위 코드에서 `new Post("JVM")`로 만들어진 `Post` 객체는 Heap에 저장된다.

Heap은 모든 스레드가 공유하는 영역이다.

그래서 여러 스레드가 같은 객체를 참조할 수 있다.

Spring 애플리케이션에서 Controller, Service, Repository 같은 Bean 객체들도 대부분 Heap에 만들어진다.

```text
Heap
  - PostController object
  - PostService object
  - PostRepository object
  - request DTO object
  - response DTO object
```

Heap은 GC의 주요 대상이다.

더 이상 참조되지 않는 객체는 나중에 Garbage Collector가 정리할 수 있다.

## 6. Method Area에는 클래스 정보가 저장된다

Method Area에는 클래스 단위의 정보가 저장된다.

예를 들면 다음과 같은 정보다.

- 클래스 이름
- 부모 클래스 정보
- 메서드 정보
- 필드 정보
- 런타임 상수 풀
- static 정보

HotSpot JVM에서는 Java 8 이후 클래스 메타데이터를 주로 Metaspace라는 네이티브 메모리 영역에 저장한다.

즉, JVM 명세에서는 Method Area라는 개념으로 설명하고, 실제 구현체에서는 Metaspace 같은 방식으로 구현될 수 있다.

개발자가 기억해야 할 핵심은 이것이다.

```text
객체 인스턴스는 Heap에 저장된다.
클래스 자체에 대한 정보는 Method Area 계열의 영역에서 관리된다.
```

예를 들어 `PostService` 객체는 Heap에 있지만, `PostService`라는 클래스의 구조 정보는 별도로 관리된다.

## 7. Java Stack에는 메서드 호출 정보가 쌓인다

Java Stack은 스레드마다 하나씩 만들어진다.

그리고 메서드가 호출될 때마다 Stack Frame이 하나씩 쌓인다.

```java
public void createPost(String title) {
    Post post = new Post(title);
    postRepository.save(post);
}
```

위 메서드가 실행되면 `createPost()`를 위한 Stack Frame이 생긴다.

그 안에는 지역 변수, 매개변수, 연산 중간값 등이 저장된다.

```text
Thread Stack
  createPost stack frame
    - title
    - post reference
```

여기서 중요한 점은 `post` 변수 자체가 객체를 직접 들고 있는 것이 아니라는 점이다.

`post` 변수는 Heap에 있는 `Post` 객체를 가리키는 참조값을 들고 있다.

```text
Java Stack
  post reference
       |
       v
Heap
  Post object
```

메서드 실행이 끝나면 Stack Frame은 제거된다.

하지만 Heap에 있는 객체는 바로 사라지는 것이 아니다.

그 객체를 더 이상 아무도 참조하지 않을 때 GC의 정리 대상이 된다.

## 8. PC Register와 Native Method Stack

PC Register는 현재 스레드가 실행 중인 JVM 명령어 위치를 기억한다.

JVM은 여러 스레드를 번갈아 실행할 수 있기 때문에, 각 스레드가 어디까지 실행했는지 따로 알아야 한다.

```text
Thread A -> 현재 실행 중인 bytecode 위치
Thread B -> 현재 실행 중인 bytecode 위치
```

Native Method Stack은 Java가 아닌 네이티브 코드를 실행할 때 사용된다.

예를 들어 JVM 내부나 JNI를 통해 C, C++ 같은 코드와 연결될 때 필요하다.

일반적인 Java, Spring 개발을 할 때 직접 다룰 일은 많지 않지만, JVM 메모리 구조를 이해할 때 함께 등장한다.

## 9. Execution Engine이 바이트코드를 실행한다

JVM 메모리에 클래스가 올라가고 실행 준비가 끝나면, Execution Engine이 바이트코드를 실행한다.

Execution Engine은 크게 두 방식으로 바이트코드를 실행한다.

```text
Interpreter
JIT Compiler
```

Interpreter는 바이트코드를 한 줄씩 읽고 실행한다.

처음 실행되는 코드는 보통 Interpreter를 통해 실행된다.

```text
bytecode instruction
 -> interpret
 -> execute
```

하지만 모든 코드를 계속 한 줄씩 해석하면 느릴 수 있다.

그래서 JVM은 자주 실행되는 코드를 찾아 JIT Compiler로 네이티브 코드로 컴파일한다.

JIT는 Just-In-Time Compiler의 줄임말이다.

```text
처음 실행
 -> Interpreter로 실행

반복해서 많이 실행되는 코드
 -> Hot Code로 판단
 -> JIT Compiler가 네이티브 코드로 컴파일
 -> 이후 더 빠르게 실행
```

예를 들어 게시글 조회 API가 자주 호출된다면, 그 요청을 처리하는 메서드들 중 일부가 JIT 컴파일 대상이 될 수 있다.

```text
PostController.findPost()
 -> PostService.findPost()
 -> PostRepository.findById()
```

JVM은 실행 중인 애플리케이션의 실제 사용 패턴을 보면서 최적화한다.

그래서 Java 애플리케이션은 시작 직후보다 어느 정도 실행된 뒤 더 안정적인 성능을 보이기도 한다.

## 10. GC는 참조되지 않는 객체를 정리한다

Java에서는 개발자가 직접 객체 메모리를 해제하지 않는다.

대신 JVM의 Garbage Collector가 Heap을 관리한다.

객체가 GC 대상인지 판단할 때 중요한 기준은 "도달 가능한가"이다.

JVM은 GC Roots에서 시작해서 참조를 따라간다.

대표적인 GC Roots는 다음과 같다.

- 실행 중인 메서드의 지역 변수
- static 필드가 참조하는 객체
- JNI가 참조하는 객체
- JVM 내부에서 사용하는 참조

GC Roots에서 참조를 따라가서 도달할 수 있는 객체는 살아 있는 객체로 본다.

반대로 어떤 경로로도 도달할 수 없는 객체는 정리 대상이 된다.

```text
GC Roots
 -> reachable object
 -> reachable object

unreachable object
 -> GC 대상
```

예를 들어 다음 코드를 보자.

```java
public void handle() {
    Post post = new Post("JVM");
}
```

`handle()`이 실행되는 동안에는 Stack Frame 안의 `post` 변수가 Heap의 `Post` 객체를 참조한다.

하지만 메서드가 끝나면 Stack Frame이 사라진다.

그리고 다른 곳에서 그 `Post` 객체를 참조하지 않는다면, 그 객체는 더 이상 도달할 수 없는 객체가 된다.

이후 GC가 실행되면 정리될 수 있다.

## 11. GC는 세대별로 보는 경우가 많다

많은 JVM 구현체는 Heap을 세대별로 나누어 관리한다.

대표적으로 Young 영역과 Old 영역으로 이해할 수 있다.

```text
Heap
  Young Generation
    - 새로 만들어진 객체
  Old Generation
    - 오래 살아남은 객체
```

대부분의 객체는 오래 살지 않는다.

예를 들어 HTTP 요청을 처리하기 위해 만들어진 request DTO, response DTO, 임시 문자열 같은 객체는 요청 처리가 끝나면 금방 필요 없어지는 경우가 많다.

그래서 JVM은 새 객체들을 Young 영역에 두고, 여기서 자주 정리한다.

Young 영역에서 여러 번 살아남은 객체는 Old 영역으로 이동할 수 있다.

```text
new object
 -> Young
 -> 여러 번 GC 후에도 살아남음
 -> Old
```

다만 GC 알고리즘마다 실제 동작 방식은 다르다.

G1 GC, ZGC, Shenandoah GC처럼 구현 방식이 다른 GC들이 있고, JVM 옵션에 따라 선택할 수 있다.

여기서 중요한 것은 GC가 단순히 "메모리를 알아서 치워 주는 기능"만은 아니라는 점이다.

객체를 계속 참조하고 있으면 GC도 정리할 수 없다.

```java
public class BadCache {
    private final List<byte[]> values = new ArrayList<>();

    public void add(byte[] value) {
        values.add(value);
    }
}
```

위 코드에서 `values`가 계속 객체를 참조하고 있다면, 그 객체들은 사용하지 않더라도 GC 대상이 되기 어렵다.

이런 경우 Java에서도 메모리 누수가 생길 수 있다.

## 12. JVM 관점에서 Spring Boot 실행 흐름 보기

Spring Boot 애플리케이션도 결국 JVM 위에서 실행된다.

```bash
java -jar app.jar
```

이 명령을 실행하면 대략 다음 흐름이 진행된다.

```text
JVM 시작
 -> main class 로드
 -> main() 실행
 -> SpringApplication.run() 실행
 -> 필요한 클래스 로드
 -> Spring Container 생성
 -> Bean 객체 생성
 -> 내장 서버 시작
 -> HTTP 요청 처리
```

Spring 코드로 보면 시작점은 보통 다음과 같다.

```java
@SpringBootApplication
public class BlogApplication {
    public static void main(String[] args) {
        SpringApplication.run(BlogApplication.class, args);
    }
}
```

JVM은 먼저 `BlogApplication` 클래스를 로드하고 `main()` 메서드를 실행한다.

그 다음 Spring Boot가 필요한 설정 클래스, Controller, Service, Repository 등을 찾고 Bean으로 등록한다.

이 Bean 객체들은 대부분 Heap에 만들어진다.

```text
Heap
  - BlogApplication
  - PostController
  - PostService
  - PostRepository
  - DataSource
  - TransactionManager
```

HTTP 요청이 들어오면 서버의 요청 처리 스레드가 동작한다.

각 요청 스레드는 자기 Java Stack을 가지고 있고, Controller에서 Service, Repository로 메서드를 호출하면서 Stack Frame을 쌓는다.

```text
Request Thread Stack
  PostRepository.findById()
  PostService.findPost()
  PostController.findPost()
```

요청 처리 중 만들어진 DTO나 엔티티 객체는 Heap에 생성된다.

요청이 끝나고 더 이상 참조되지 않는 객체는 나중에 GC 대상이 된다.

자주 호출되는 Controller, Service, Repository 메서드는 JVM의 JIT 최적화 대상이 될 수도 있다.

즉, Spring Boot를 이해할 때도 JVM 관점은 중요하다.

Spring Container가 Bean을 관리하더라도, 그 Bean은 결국 JVM의 Heap 위에 존재하고 JVM의 실행 엔진에 의해 실행된다.

## 13. JVM 관련 에러를 이해하기

JVM 구조를 알면 자주 보는 에러도 더 잘 이해할 수 있다.

### StackOverflowError

StackOverflowError는 Java Stack이 너무 깊게 쌓였을 때 발생한다.

대표적인 원인은 끝나지 않는 재귀 호출이다.

```java
public void call() {
    call();
}
```

`call()`이 끝나지 않고 계속 자기 자신을 호출하면 Stack Frame이 계속 쌓인다.

결국 스택 공간이 부족해지고 StackOverflowError가 발생한다.

### OutOfMemoryError

OutOfMemoryError는 JVM이 필요한 메모리를 확보하지 못할 때 발생한다.

가장 흔하게는 Heap 공간이 부족할 때 볼 수 있다.

```text
java.lang.OutOfMemoryError: Java heap space
```

하지만 Metaspace, native memory, thread 생성 문제 등 다양한 원인으로도 발생할 수 있다.

```text
java.lang.OutOfMemoryError: Metaspace
java.lang.OutOfMemoryError: unable to create native thread
```

따라서 OutOfMemoryError를 보면 단순히 "Heap이 부족하다"로만 단정하지 말고 어떤 메모리 영역에서 문제가 발생했는지 확인해야 한다.

### ClassNotFoundException

ClassNotFoundException은 실행 중에 특정 클래스를 찾으려고 했지만 classpath나 module path에서 찾지 못했을 때 발생한다.

예를 들어 리플렉션으로 클래스 이름을 문자열로 넘겼는데 실제 런타임에 해당 클래스가 없으면 발생할 수 있다.

```java
Class.forName("com.example.MissingClass");
```

### NoClassDefFoundError

NoClassDefFoundError는 컴파일할 때는 있었거나 이미 참조된 적 있는 클래스가 런타임에 정상적으로 로드되지 못할 때 발생하는 경우가 많다.

의존성 버전이 맞지 않거나, 배포 파일에 필요한 라이브러리가 빠졌을 때 볼 수 있다.

ClassNotFoundException과 비슷해 보이지만, NoClassDefFoundError는 JVM이 클래스를 로드하거나 초기화하는 과정에서 실패했다는 쪽에 더 가깝다.

### UnsupportedClassVersionError

UnsupportedClassVersionError는 더 높은 버전의 Java로 컴파일된 class 파일을 낮은 버전의 JVM에서 실행하려고 할 때 발생한다.

예를 들어 Java 21로 컴파일한 애플리케이션을 Java 17 JVM에서 실행하면 이런 문제가 생길 수 있다.

```text
class file version이 현재 JVM에서 지원하는 버전보다 높음
```

이 에러는 JVM이 `.class` 파일을 읽는 단계에서 버전이 맞지 않는다는 뜻이다.

## 전체 흐름 다시 보기

JVM이 자바 코드를 실행하는 흐름을 다시 정리하면 다음과 같다.

```text
1. 개발자가 .java 파일을 작성한다.
2. javac가 .java 파일을 .class 바이트코드로 컴파일한다.
3. java 명령으로 JVM이 시작된다.
4. ClassLoader가 필요한 클래스를 찾고 로드한다.
5. JVM이 클래스를 검증, 준비, 해석, 초기화한다.
6. Runtime Data Area에 클래스 정보, 객체, 스택 프레임 등이 관리된다.
7. Execution Engine이 바이트코드를 실행한다.
8. 자주 실행되는 코드는 JIT Compiler가 최적화한다.
9. Heap에 만들어진 객체 중 더 이상 도달할 수 없는 객체는 GC 대상이 된다.
10. 애플리케이션이 종료되면 JVM 프로세스도 종료된다.
```

더 짧게 보면 다음과 같다.

```text
.java
 -> .class
 -> ClassLoader
 -> Runtime Data Area
 -> Execution Engine
 -> GC
```

## 정리

JVM은 자바 코드를 실행하는 단순한 프로그램이 아니다.

클래스를 로드하고, 바이트코드를 검증하고, 메모리를 나누어 관리하고, 코드를 해석하거나 JIT 컴파일하고, 필요 없어진 객체를 GC로 정리하는 런타임 환경이다.

자바 애플리케이션에서 객체는 Heap에 만들어지고, 메서드 호출은 각 스레드의 Java Stack에 쌓인다.

클래스 정보는 Method Area 계열의 영역에서 관리되고, 실행 엔진은 바이트코드를 실제 CPU에서 실행 가능한 형태로 처리한다.

Spring Boot 애플리케이션도 이 흐름 위에서 동작한다.

Spring이 Bean을 만들고 의존성을 주입하더라도, 그 객체들은 JVM 메모리 위에 존재한다.

Controller, Service, Repository 메서드가 호출될 때마다 Stack Frame이 생기고, 요청 처리 중 만들어진 객체들은 Heap에 쌓였다가 더 이상 참조되지 않으면 GC 대상이 된다.

그래서 JVM을 이해하면 단순히 자바 문법만 아는 것보다 더 깊게 문제를 볼 수 있다.

Java 버전이 맞지 않는 이유, GC 로그를 보는 이유, StackOverflowError와 OutOfMemoryError가 발생하는 이유, Spring 애플리케이션이 실행 중에 어떤 방식으로 메모리를 쓰는지까지 연결해서 이해할 수 있다.

결국 JVM은 Java와 운영체제 사이에서 바이트코드 실행, 메모리 관리, 최적화, 안정성을 책임지는 핵심 런타임이다.
