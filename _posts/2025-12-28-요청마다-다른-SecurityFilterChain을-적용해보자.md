---
title: 요청마다 다른 SecurityFilterChain을 적용해보자
description: Spring Security의 Multiple SecurityFilterChain 적용 중 JWT 인증 필터가 계속 실행되던 원인과 해결 방법 정리
author: kancho
date: 2025-12-28 20:11:00
categories: [Frameworks, Spring Security]
tags: [Filter]
pin: false
# math: true
# mermaid: true
---



## 개요
현재 진행중인 사이드 프로젝트에서는 Spring Security 기반의 인증 시스템을 구축하였고, 서버에서 발급한 access token을 검증하기 위해 `JwtAuthenticationFilter` 를 직접 구현하여 사용하고 있다. 대부분의 요청은 인증이 필요하기 때문에 JWT 인증 필터를 거쳐야 하지만, 일부 요청은 access token의 상태와 관계 없이 처리되어야 한다.

대표적으로 다음과 같은 요청들이 있다.

- `/refresh` 는 access token이 없거나 유효하지 않더라도, 유효한 refresh token이 존재한다면 access token을 재발급해야 한다.
- `/logout` 은 token의 상태와 무관하게 항상 로그아웃 처리가 이루어져야 한다.

하지만 실제로는 위 두 요청 모두 `JwtAuthenticationFilter` 를 거치면서 `EXPIRED_TOKEN` 예외가 발생하고 있었다.

어떻게 하면 특정 요청에 대해서는 JWT 인증 필터를 거치지 않도록 만들 수 있을까?

<br/>

## 단순하게 분기 처리해보자
가장 먼저 떠올릴 수 있는 방법은 Filter 내부에서 URL을 기준으로 직접 분기 처리하는 것이다.

```java
if (request.getRequestURI().equals("/refresh")) {
	filterChain.doFilter(request, response);
	return;
}
```

간단하지만 명확한 단점이 있다.

새로운 예외 URL이 추가될 때마다, Security 설정을 수정하고, Filter 내부 로직을 수정해야 한다는 것이다. 해당 요청이 인증이 필요한가라는 판단이 Security 설정에 있지 않고 Filter 내부에 숨어버리므로, 확장성과 가독성 측면에서 좋은 설계라고 보기 어려웠다.

URL에 따른 인증 정책은 Security 설정에서 결정하고, Filter는 그 결정에 따라 실행되도록 할 수 없을까?

이 때 필요한 것이 Spring Security에서 제공하는 Multiple SecurityFilterChain이다.

<br/>

## 요청마다 다른 SecurityFilterChain을 적용해보자
Spring Security는 하나의 애플리케이션 안에서 여러 개의 `SecurityFilterChain` 을 정의할 수 있도록 지원해준다.
요청 URL이나 조건에 따라 서로 다른 Filter Chain을 적용할 수 있는 구조다.

이를 활용한다면 인증이 필요하지 않는 `/refresh`, `/logout` 과 같은 요청은 JWT 인증 필터가 없는 체인으로, 그 외 인증이 필요한 요청은 JWT 인증 필터가 포함된 체인으로 분리할 수 있다.

설정은 다음과 같이 구성했다:

```java
@Configuration  
@EnableWebSecurity  
@RequiredArgsConstructor  
public class SecurityConfig {  
    private final JwtAuthenticationFilter jwtAuthenticationFilter;  
    // ...
  
    @Bean  
    @Order(1)  
    public SecurityFilterChain publicSecurityFilterChain(HttpSecurity http) throws Exception {  
        http  
                .securityMatcher( 
                        "/api/auth/refresh",  
                        "/api/auth/logout",
                        // (중략)
                )  
                .csrf(AbstractHttpConfigurer::disable)  
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))  
                .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())  
                .headers(headers -> headers  
                        .frameOptions(HeadersConfigurer.FrameOptionsConfig::disable));  
  
        return http.build();  
    }  
    @Bean  
    @Order(2)  
    public SecurityFilterChain privateSecurityFilterChain(HttpSecurity http) throws Exception {  
        http  
                .csrf(AbstractHttpConfigurer::disable)  
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))  
                .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())  
                .exceptionHandling(exception -> exception  
                        .authenticationEntryPoint(customAuthenticationEntryPoint))  
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);  
  
        return http.build();  
    }
}
```

이제 `/refresh` 와 `logout` 요청은 당연히 JwtAuthenticationFilter를 거치지 않을 것이라 기대했다.

하지만 예상과 달리 두 요청 모두 여전히 JwtAuthenticationFilter를 지나가고 있었다.

<br/>

## ❓ 왜 아직도 JwtAuthenticationFilter를 지나가는 것일까
이 문제를 이해하기 위해서는 Spring Security의 Filter 구조를 조금 더 자세히 들여다볼 필요가 있다.

### Servlet Container가 관리하는 FilterChain과 Spring Container가 관리하는 SecurityFilterChain
Spring Security에서는 이름은 비슷하지만 다른 역할을 가진 두 가지 FilterChain이 존재한다.

#### FilterChain
`FilterChain` 은 Tomcat과 같은 Embedded <u>Servlet Container가 관리</u>하는 FilterChain으로 `javax.servlet.Filter` 을 기반으로 동작한다.

Servlet Container 관점에서 Filter는 HTTP 요청이 들어올 때 가장 앞단에서 실행되는 전처리 로직이며, 요청이 들어오면 등록된 순서대로 FilterChain을 따라 실행된다.

#### SecurityFilterChain
반면 `SecurityFilterChain` 은 <u>Spring Container가 관리</u>하는 논리적인 FilterChain으로, 요청 URL이나 RequestMatcher 조건에 따라 서로 다른 보안 규칙을 적용할 수 있다.

Spring Security는 인증 방식, 인가 정책, 예외 처리 전략 등을 Bean으로 구성하고, 요청 조건에 따라 이를 다르게 조합하기를 원한다. 이런 요구를 충족하기 위해 등장한 개념이 `SecurityFilterChain` 이다. 

즉 SecurityFilterChain은 어떤 요청에 어떤 보안 Filter들을 적용할 것인가를 정의한 보안 규칙의 집합이라 볼 수 있다.


#### DelegatingFilterProxy와 FilterChainProxy
Spring Security는 인증과 인가 같은 보안 로직이 Spring MVC의 DispatcherServlet에 도달하기 이전에 수행되기를 원한다. 이를 위해 Servlet Spec에서 제공하는 Filter 레벨에서 동작해야 한다.

하지만 Servlet Container는 Spring Bean들을 직접 관리할 수 없다. 인증을 수행하는 `AuthenticationManager`, 사용자 정보를 조회하는 `UserDetailsService` 와 같은 핵심 보안 컴포넌트들은 모두 Spring Container에서 관리되는 Bean이기 때문이다.

이 두 개의 독립적인 Container를 연결하기 위해 등장한 것이 **`DelegatingFilterProxy`** 다.

`DelegatingFilterProxy` 는 Servlet Container의 FilterChain에 Servlet Filter로 등록된다. 그리고 **실제 보안 처리는** Spring Container 안에 있는 **Spring Security로 위임**하는데, 이 때 위임 대상이 되는 Bean이 바로 `FilterChainProxy` 다. 

`FilterChainProxy` 는 여러 개의 `SecurityFilterChain` 을 관리한다. 요청이 들어오면 URL이나 Matcher 조건을 기준으로 어떤 SecurityFilterChain을 적용할지 결정하고, 그에 해당하는 보안 Filter들을 실행한다.

내가 설정한 Multiple SecurityFilterChain 역시 바로 이 `FilterChainProxy` 안에서 요청을 URL 별로 분기하고 있는 구조다.

### ⚠️ Filter는 이미 Servlet Container에 등록되어 있다

문제의 핵심은 여기에 있었다.

공식 문서에 다음과 같이 명시되어 있다.

> [Add a Servlet, Filter, or Listener to an application](https://docs.spring.io/spring-boot/docs/1.5.4.RELEASE/reference/html/howto-embedded-servlet-containers.html#howto-add-a-servlet-filter-or-listener)
> 
> To add a `Servlet`, `Filter`, or Servlet `*Listener` provide a `@Bean` definition for it.
> As [described above](https://docs.spring.io/spring-boot/docs/1.5.4.RELEASE/reference/html/howto-embedded-servlet-containers.html#howto-add-a-servlet-filter-or-listener-as-spring-bean "73.1.1 Add a Servlet, Filter or Listener using a Spring bean") any `Servlet` or `Filter` beans will be registered with the servlet container automatically.

Spring Boot에서는 Filter를 Spring Bean으로 등록하면 Servlet Container에 자동으로 등록된다는 것이다.

즉 `JwtAuthenticationFilter` 를 Spring Bean으로 등록한 순간, 해당 Filter는 이미 Servlet Container의 FilterChain에 포함된 상태였다.

이 상태에서 다시 SecurityFilterChain에 `addFilterBefore()` 로 추가했기 때문에,
- Servlet Container의 FilterChain에서 한 번
- Spring Container의 SecurityFilterChain에서 한 번

중복으로 실행되는 구조가 만들어진 것이다.

<br/>

## Servlet Container로의 자동 등록을 막아보자
해결 방법은 Servlet Container로의 자동 등록을 비활성화하는 것이었다.

```java
@Bean  
public FilterRegistrationBean<JwtAuthenticationFilter> jwtFilterDisable(JwtAuthenticationFilter filter) {  
    FilterRegistrationBean<JwtAuthenticationFilter> bean = new FilterRegistrationBean<>(filter);  
    bean.setEnabled(false);  
    return bean;  
}
```

[[참고] Disable registration of a Servlet or Filter](https://docs.spring.io/spring-boot/docs/1.5.4.RELEASE/reference/html/howto-embedded-servlet-containers.html#howto-disable-registration-of-a-servlet-or-filter)

이렇게 설정하면 `JwtAuthenticationFilter` 는 Servlet Container의 FilterChain에는 등록되지 않고, Spring Container의 SecurityFilterChain 에만 등록하게 된다.

> 한 가지 대안으로 `new` 를 통해 Filter를 직접 생성하는 방식도 있지만, 해당 Filter가 의존하는 Bean들이 많았기 때문에 이번 경우에는 적용하지 않았다.

그 결과 요청별로 의도한 SecurityFilterChain이 정상적으로 적용되었다.

<br/>

## ✅ 정리

- Spring Security를 사용하면 요청은 Servlet Container가 관리하는 FilterChain과 Spring Container가 관리하는 SecurityFilterChain을 지나간다.
- Spring Boot 환경에서는 Filter를 Spring Bean으로 등록하면 Servlet Container에 자동으로 등록된다.

이번 이슈는 해당 사실을 인지하지 못한 채 `JwtAuthenticationFilter` 를 Spring DI로 주입하고 Multiple SecurityFilterChain에 다시 추가하면서, 동일한 Filter가 Servlet Container와 Spring Security 양쪽에 모두 등록되어 발생한 문제이다. 즉 Filter는 요청 흐름의 가장 앞단에서 이미 한 번 실행되고 있었고, 그 이후 SecurityFilterChain에서 다시 실행되면서 의도와 다르게 모든 요청이 Jwt 인증 필터를 거치게 되었다. 따라서 Servlet Container로의 자동 등록을 비활성화함으로써 의도한대로 요청별로 서로 다른 SecurityFilterChain이 적용되는 구조를 만들 수 있었다.
