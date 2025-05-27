---
title: Filter에서 발생한 예외는 어떻게 처리할까
description: GlobalExceptionHandler가 잡지 못하는 예외 처리 방법 (feat. AuthenticationEntryPoint)
author: kancho
date: 2025-05-26 22:06:00
categories: [Project, Pigrest]
tags: [Java, Spring, Filter, Exception]
pin: true
# math: true
# mermaid: true
---

## 개요

Expired된 Access Token을 넘겨서, JwtAuthenticationFilter에서 `ExpiredJwtException` 이 발생했다.
그런데 `@GlobalExceptionHandler` 에서 반환해주는 공통 응답 형식처럼 반환해주지를 않았다.

당연하다...

## ⚠️ 문제 원인

<img src="../assets/img/posts/jwt-filter-dispatcher-servlet.svg" />

Spring의 `@ControllerAdvice` 또는 `@ExceptionHandler`는 Controller 내부에서 발생한 예외만 처리한다. 반면 `JwtAuthenticationFilter` 는 Spring MVC DispatcherServlet 이전 단계, 즉 서블릿 필터 체인 단계에서 발생하기 때문에 GlobalExceptionHandler 에서는 잡힐 수 없다.
따라서 클라이언트는 응답 형식 없이 `401` 혹은 `403` Http Status code만 받게 된다.

이렇게 되면 클라이언트의 Axios Interceptor에서 예외 응답 파싱할 때 문제가 발생하게 될 것이다.

<br/>

## ✏️ 해결 방안

```java
catch (JwtException e) {
    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    response.setContentType("application/json");
    response.getWriter().write(new ObjectMapper().writeValueAsString(
        ApiResponse.error(ApiStatusCode.INVALID_TOKEN, e.getMessage())
    ));
    return;
}
```

<br/>

## ✅ 코드 개선

예외 응답 포맷을 통일하고, 필터에서는 인증 로직에만 집중하도록 하기 위해 다음과 같이 공통 유틸 클래스를 만들었다.

```java
@Component
@RequiredArgsConstructor
public class FilterExceptionHandler {
    private record ErrorInfo(ApiStatusCode statusCode, String message) {}

    private final ObjectMapper objectMapper;

    public void handle(HttpServletResponse response, Exception e) throws IOException {
        ErrorInfo error = getErrorInfo(e);

        response.setStatus(error.statusCode().getStatus());
        response.setContentType("application/json");
        objectMapper.writeValue(response.getWriter(), ApiResponse.error(error.statusCode(), error.message()));
    }

    private ErrorInfo getErrorInfo(Exception e) {
        if (e instanceof JwtException) {
            return new ErrorInfo(ApiStatusCode.INVALID_TOKEN, "Invalid token. Please log in again.");
        } else if (e instanceof AuthenticationException) {
            return new ErrorInfo(ApiStatusCode.UNAUTHORIZED, "Authentication failed. Please check your credentials.");
        }
        return new ErrorInfo(ApiStatusCode.INTERNAL_SERVER_ERROR, "Internal Server Error");
    }
}
```

