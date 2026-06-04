---
title: Spring Boot에서 공통 API 응답 형식 적용하기
date: 2026-06-04 00:00:00 +0900
categories: [Spring Boot]
tags: [spring-boot, api, response, responsebodyadvice, rest-api, backend]
---
## 시작점

API를 만들다 보면 성공 응답과 에러 응답의 모양이 조금씩 달라지기 쉽다.

예를 들어 로그인 API는 DTO를 그대로 반환하고,

```json
{
  "accessToken": "...",
  "userId": 1
}
```

에러 응답은 별도 형태로 반환할 수 있다.

```json
{
  "code": "INVALID_TOKEN",
  "message": "올바르지 않은 JWT 토큰입니다."
}
```

이렇게 되면 프론트엔드에서는 API마다 응답을 해석하는 방식이 달라진다. 그래서 모든 API 응답을 다음처럼 통일하기로 했다.

성공 응답:

```json
{
  "success": true,
  "data": {
    "accessToken": "...",
    "userId": 1
  }
}
```

실패 응답:

```json
{
  "success": false,
  "error": {
    "code": "INVALID_TOKEN",
    "message": "올바르지 않은 JWT 토큰입니다."
  }
}
```

## ApiResponse 만들기

공통 응답 클래스는 `record`로 만들었다.

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public record ApiResponse<T>(
        boolean success,
        T data,
        ApiError error
) {

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, data, null);
    }

    public static ApiResponse<Void> noContent() {
        return new ApiResponse<>(true, null, null);
    }

    public static ApiResponse<Void> error(String code, String message) {
        return new ApiResponse<>(false, null, new ApiError(code, message));
    }

    public record ApiError(String code, String message) {
    }
}
```

`@JsonInclude(JsonInclude.Include.NON_NULL)`은 `null` 필드를 JSON에서 제외하기 위한 설정이다. 성공 응답에는 `error`가 필요 없고, 실패 응답에는 `data`가 필요 없기 때문에 불필요한 `null` 필드를 숨긴다.

`noContent()`라는 이름을 사용한 이유도 있다. `record`의 필드명이 `success`이면 Java가 자동으로 `success()` 접근자를 만든다. 그래서 파라미터 없는 정적 팩토리 메서드를 `success()`로 만들 수 없다. 데이터 없는 성공 응답이라는 의미를 살려 `noContent()`로 이름을 정했다.

## 컨트롤러에서 직접 감싸지 않기

가장 단순한 방법은 모든 컨트롤러에서 직접 `ApiResponse.success(...)`를 반환하는 것이다.

```java
@GetMapping("/profile")
public ResponseEntity<ApiResponse<UserProfileResponse>> findProfile(Authentication authentication) {
    Long userId = (Long) authentication.getPrincipal();
    return ResponseEntity.ok(ApiResponse.success(userService.findProfile(userId)));
}
```

하지만 이 방식은 컨트롤러마다 반복 코드가 생긴다. 컨트롤러는 가능한 한 요청을 받고 서비스를 호출하는 흐름만 보여주는 편이 좋다.

그래서 컨트롤러는 기존처럼 DTO나 `ResponseEntity<DTO>`를 반환하고, 응답 직전에 자동으로 감싸는 방식을 선택했다.

## ResponseBodyAdvice로 성공 응답 자동 래핑

Spring MVC의 `ResponseBodyAdvice`를 사용하면 컨트롤러가 반환한 body를 HTTP 응답으로 쓰기 직전에 가로챌 수 있다.

```java
@RestControllerAdvice
public class ApiResponseAdvice implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return !GlobalExceptionHandler.class.equals(returnType.getContainingClass());
    }

    @Override
    public Object beforeBodyWrite(
            Object body,
            MethodParameter returnType,
            MediaType selectedContentType,
            Class<? extends HttpMessageConverter<?>> selectedConverterType,
            ServerHttpRequest request,
            ServerHttpResponse response
    ) {
        if (isSpringDocRequest(request)) {
            return body;
        }

        if (body == null || body instanceof ApiResponse<?> || body instanceof String) {
            return body;
        }

        return ApiResponse.success(body);
    }

    private boolean isSpringDocRequest(ServerHttpRequest request) {
        String path = request.getURI().getPath();
        return path.startsWith("/v3/api-docs")
                || path.startsWith("/swagger-ui")
                || path.equals("/swagger-ui.html");
    }
}
```

컨트롤러가 DTO만 반환하면 기본 상태 코드는 `200 OK`다.

```java
@GetMapping("/health")
public HealthResponse health() {
    return new HealthResponse("UP", "AI Health Coach Backend", Instant.now());
}
```

실제 응답은 다음처럼 감싸진다.

```json
{
  "success": true,
  "data": {
    "status": "UP",
    "service": "AI Health Coach Backend",
    "timestamp": "2026-06-04T02:10:41Z"
  }
}
```

`ResponseEntity<DTO>`를 반환하면 `ResponseEntity`의 상태 코드는 유지하고 body만 감싼다. 예를 들어 `201 Created`를 반환하면 상태 코드는 그대로 `201`이고, body만 `data` 안에 들어간다.

삭제 API처럼 `ResponseEntity.noContent().build()`를 반환하는 경우에는 body가 `null`이므로 감싸지 않는다. 따라서 `204 No Content`는 그대로 유지된다.

## Swagger UI가 Petstore로 뜬 문제

처음에는 `ResponseBodyAdvice`가 `/v3/api-docs` 응답까지 감싸버렸다.

Swagger UI는 `/v3/api-docs`에서 OpenAPI 원본 JSON을 기대한다.

```json
{
  "openapi": "3.1.0",
  "info": {
    "title": "AI Health Coach API"
  },
  "paths": {}
}
```

그런데 이 응답이 다음처럼 감싸지면 Swagger UI가 OpenAPI 문서로 인식하지 못한다.

```json
{
  "success": true,
  "data": {
    "openapi": "3.1.0",
    "info": {
      "title": "AI Health Coach API"
    },
    "paths": {}
  }
}
```

그 결과 Swagger UI가 기본 예제인 Petstore를 보여줄 수 있다. 그래서 `/v3/api-docs`, `/swagger-ui`, `/swagger-ui.html` 경로는 래핑 대상에서 제외했다.

## 에러 응답은 GlobalExceptionHandler에서 처리

성공 응답은 `ResponseBodyAdvice`에서 자동으로 감싸지만, 에러 응답은 `GlobalExceptionHandler`에서 직접 `ApiResponse.error(...)`를 반환한다.

```java
@ExceptionHandler(UserException.class)
public ResponseEntity<ApiResponse<Void>> handleUserException(UserException exception) {
    UserErrorCode errorCode = exception.getErrorCode();

    return ResponseEntity
            .status(errorCode.getStatus())
            .body(ApiResponse.error(errorCode.name(), errorCode.getMessage()));
}
```

`ApiResponseAdvice`에서 `GlobalExceptionHandler`를 제외한 이유는 에러 응답이 다시 성공 응답처럼 감싸지는 것을 막기 위해서다.

원하는 응답:

```json
{
  "success": false,
  "error": {
    "code": "INVALID_TOKEN",
    "message": "올바르지 않은 JWT 토큰입니다."
  }
}
```

잘못 감싸진 응답:

```json
{
  "success": true,
  "data": {
    "success": false,
    "error": {
      "code": "INVALID_TOKEN",
      "message": "올바르지 않은 JWT 토큰입니다."
    }
  }
}
```

## Validation 예외 처리

요청 값 검증 실패도 공통 응답 형식으로 맞춰야 한다.

`@Valid @RequestBody` 검증 실패는 `MethodArgumentNotValidException`으로 들어온다.

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ApiResponse<Void>> handleMethodArgumentNotValidException(
        MethodArgumentNotValidException exception
) {
    String message = exception.getBindingResult()
            .getFieldErrors()
            .stream()
            .findFirst()
            .map(FieldError::getDefaultMessage)
            .orElse("요청 값이 올바르지 않습니다.");

    return ResponseEntity
            .badRequest()
            .body(ApiResponse.error("VALIDATION_ERROR", message));
}
```

`@RequestParam`, `@PathVariable` 검증 실패는 `ConstraintViolationException`으로 처리한다.

```java
@ExceptionHandler(ConstraintViolationException.class)
public ResponseEntity<ApiResponse<Void>> handleConstraintViolationException(
        ConstraintViolationException exception
) {
    String message = exception.getConstraintViolations()
            .stream()
            .findFirst()
            .map(ConstraintViolation::getMessage)
            .orElse("요청 값이 올바르지 않습니다.");

    return ResponseEntity
            .badRequest()
            .body(ApiResponse.error("VALIDATION_ERROR", message));
}
```

이제 validation 실패도 다음처럼 내려간다.

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "요청 값이 올바르지 않습니다."
  }
}
```

## 예상하지 못한 서버 에러 처리

마지막 fallback으로 `Exception.class` 핸들러를 추가했다.

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ApiResponse<Void>> handleException(Exception exception) {
    log.error("Unhandled exception", exception);

    return ResponseEntity
            .internalServerError()
            .body(ApiResponse.error("INTERNAL_SERVER_ERROR", "서버 오류가 발생했습니다."));
}
```

여기서 중요한 점은 실제 exception message를 클라이언트에 그대로 노출하지 않는 것이다. 예외 메시지에는 내부 클래스명, SQL, 환경 정보 같은 민감한 정보가 섞일 수 있다. 그래서 클라이언트에는 일반 메시지를 내려주고, 서버 로그에만 실제 예외를 남긴다.

## JWT 인증 실패는 GlobalExceptionHandler로 가지 않는다

JWT 검증은 컨트롤러에 도달하기 전인 Spring Security 필터 단계에서 일어난다.

요청 흐름은 대략 다음과 같다.

```txt
요청
 -> JwtAuthenticationFilter
 -> Spring Security 인증/인가 검사
 -> 컨트롤러
```

토큰이 없거나 잘못된 경우 컨트롤러까지 가지 못한다. 따라서 `GlobalExceptionHandler`가 아니라 Spring Security의 인증 실패 처리 지점에서 직접 응답을 작성해야 한다.

401 인증 실패는 `AuthenticationEntryPoint`로 처리했다.

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException
    ) throws IOException {
        UserErrorCode errorCode = UserErrorCode.INVALID_TOKEN;

        response.setStatus(errorCode.getStatus().value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding(StandardCharsets.UTF_8.name());

        objectMapper.writeValue(
                response.getWriter(),
                ApiResponse.error(errorCode.name(), errorCode.getMessage())
        );
    }
}
```

`Authorization` 헤더가 없어서 보호 API 접근에 실패하는 경우 Spring Security가 이 EntryPoint를 호출한다.

잘못된 Bearer 토큰은 `JwtAuthenticationFilter`에서 먼저 예외가 발생한다. 이때도 응답 형식을 한 곳에서 관리하기 위해 필터에서 직접 JSON을 쓰지 않고 EntryPoint로 위임했다.

```java
try {
    userId = jwtTokenProvider.getUserId(token);
} catch (UserException exception) {
    SecurityContextHolder.clearContext();
    jwtAuthenticationEntryPoint.commence(
            request,
            response,
            new BadCredentialsException(exception.getMessage(), exception)
    );
    return;
}
```

이렇게 하면 토큰 없음과 토큰 오류 모두 같은 응답 형식으로 내려간다.

```json
{
  "success": false,
  "error": {
    "code": "INVALID_TOKEN",
    "message": "올바르지 않은 JWT 토큰입니다."
  }
}
```

## 403 권한 실패 처리

401은 인증 실패다. 즉 사용자가 누구인지 확인되지 않은 상태다.

반면 403은 인증은 되었지만 접근 권한이 부족한 상태다. 이 경우 Spring Security는 `AuthenticationEntryPoint`가 아니라 `AccessDeniedHandler`를 사용한다.

```java
@Component
@RequiredArgsConstructor
public class JwtAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException
    ) throws IOException {
        UserErrorCode errorCode = UserErrorCode.PROFILE_ACCESS_DENIED;

        response.setStatus(errorCode.getStatus().value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding(StandardCharsets.UTF_8.name());

        objectMapper.writeValue(
                response.getWriter(),
                ApiResponse.error(errorCode.name(), errorCode.getMessage())
        );
    }
}
```

그리고 `SecurityConfig`에서 401과 403 처리기를 함께 등록했다.

```java
return http
        .csrf(AbstractHttpConfigurer::disable)
        .exceptionHandling(exception -> exception
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler))
        .authorizeHttpRequests(auth -> auth
                .requestMatchers(SecurityPaths.PUBLIC_PATHS).permitAll()
                .anyRequest().authenticated())
        .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
        .build();
```

이제 인증 실패와 권한 실패 모두 공통 API 응답 형식을 따른다.

## 정리

이번 작업으로 다음 응답 경로를 모두 공통 형식으로 맞췄다.

- 일반 성공 응답: `ResponseBodyAdvice`에서 `{ success: true, data: ... }`로 래핑
- 일반 비즈니스 예외: `GlobalExceptionHandler`에서 `{ success: false, error: ... }` 반환
- validation 실패: `VALIDATION_ERROR` 반환
- 예상하지 못한 서버 에러: `INTERNAL_SERVER_ERROR` 반환
- JWT 인증 실패 401: `JwtAuthenticationEntryPoint`에서 처리
- 권한 실패 403: `JwtAccessDeniedHandler`에서 처리
- Swagger `/v3/api-docs`: OpenAPI 원본 JSON을 유지하기 위해 래핑 제외

`ResponseBodyAdvice`를 쓰면 컨트롤러 코드를 반복적으로 수정하지 않고 성공 응답을 통일할 수 있다. 하지만 모든 응답을 가로채는 만큼 Swagger, 문자열 응답, 파일 다운로드, 빈 응답처럼 원본 body가 중요한 경우는 반드시 제외 조건을 고려해야 한다.
