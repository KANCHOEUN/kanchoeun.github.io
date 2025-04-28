---
title: Servlet Application의 Architecture
description: Servlet 기반 애플리케이션에서 Spring Security의 전반적인 개념 정리
author: kancho
date: 2024-11-01 17:25:00 +1500
categories: [Frameworks, Spring Security]
tags: [Spring Security]
pin: false
# math: true
# mermaid: true
---

해당 페이지는 스프링 시큐리티의 공식 문서를 기반으로 작성하였습니다.


## Filter란
스프링 시큐리티의 서블릿은 Servlet Filter를 기반으로, Filter에 대한 역할을 아는 것이 먼저다.
다음은 하나의 HTTP 요청을 처리하는 방법이다.

![filter](/assets/img/posts/filter.png)

클라이언트가 애플리케이션에 요청을 보내면, 컨테이너는 요청 URI 경로에 따라 `HttpServletRequest` 를 처리할, `Filter` 인스턴스와 `Servlet` 을 포함하는 `FilterChain` 을 생성한다.

> 여기서 이야기하는 컨테이너는 Bean을 관리하는 Spring Container와는 다른, 요청을 처리하는 Servlet Container(ex. Tomcat)이다.

Spring MVC 애플리케이션에서는 `Servlet` 은 `DispatcherServlet` 의 인스턴스이다.
→ `Servlet` 은 단 하나의 `HttpServletRequest` 혹은 `HttpServletResponse` 만 처리 가능하다.

#### Filter의 기능
- 요청을 처리하는 과정에서 이후에 실행될 `Filter` 나 `Servlet` 이 호출되지 않도록 막을 수 있다.
	- 예를 들어 인증이 필요하지만 인증되지 않은 사용자인 경우, 필터가 요청을 차단하고 `HttpServletResponse` 를 작성하여 접근 거부 메세지를 반환함으로써 요청이 더 이상 진행되지 않고, 다음 필터나 서블릿이 호출되지 않도록 할 수 있다.
- 이후에 실행될 `Filter` 나 `Servlet` 에서 사용되는 `HttpServletRequest` 혹은 `HttpServletResponse` 를 수정할 수 있다.

<br/>

```java
public void doFilter(ServletRequest request, ServletResponse response) {
	chain.doFilter(request, response);
}
```

필터는 하위 필터와 서블릿에만 영향을 미치기 때문에, 각 필터가 호출되는 순서가 매우 중요하다.

<br/>

## Filter 구현체

### DelegatingFilterProxy

Spring은 <u>Servlet Container의 생명주기</u>와 <u>Spring의 ApplicationContext 간 연결</u>을 허용하는 Filter의 구현체인 `DelegatingFilterProxy` 를 제공한다.

Servlet Container는 자체 표준을 사용하여 `Filter` 인스턴스들을 등록할 수 있지만, Spring에서 정의한 Bean에 대해서는 인식하지 못한다.

> 기존의 Servlet Container와 Spring Container는 독립적으로 동작하여, 서로 직접적인 상호 작용을 하지 않는다. 

`DelegatingFilterProxy` 를 표준 Servlet 컨테이너 메커니즘을 통해 등록할 수 있지만, 실제 작업은 Filter를 구현한 Spring Bean에 위임한다.

![delegating-filter-proxy](/assets/img/posts/delegating-filter-proxy.png)

<br/>


> 지연 로딩(Lazy Loading)이란
> JPA에서는 <u>성능 향상</u>을 위해 Proxy 엔티티를 통해 지연 로딩을 하는데, Servlet Container에서는 <u>Spring Container의 Bean으로 등록된 Filter를 사용하고 싶을 때</u>, DelegatingFilterProxy 객체를 활용하여 지연 로딩 기법을 사용한다.
> 
> 지연 로딩이란 객체를 바로 호출하지 않고, 객체가 실제로 필요할 때까지 미뤄두었다가, 해당 객체가 호출되는 순간에 로드하는 기법이다. 반대로 객체가 초기화될 때 관련된 모든 데이터를 함께 로드하는 것은 즉시 로딩(Eager Loading)이라 한다. 지연 로딩은 초기화 시점에 불필요한 데이터를 미리 로딩하지 않기 때문에 메모리와 CPU 자원을 절약할 수 있는 방식이다.

### FilterChainProxy

![filter-chain-proxy](/assets/img/posts/filter-chain-proxy.png)

`FilterChainProxy` 는 `SecurityFilterChain` 을 통해 많은 `Filter` 인스턴스로 작업을 위임할 수 있도록 해준다. 즉 `FilterChainProxy` 는 여러 가지 보안 필터들을 하나의 필터처럼 관리할 수 있도록 해주는 객체이다. `FilterChainProxy` 는 Bean이기 때문에 `DelegatingFilterProxy` 에 감싸서 사용된다.

### SecurityFilterChain
![security-filter-chain](/assets/img/posts/security-filter-chain.png)

`SecurityFilterChain` 은 `FilterChainProxy` 가 이용하는데, 어떤 보안 필터 인스턴스가 호출되어야 하는지를 결정해준다.

체인 안에 있는 Security Filter들은 주로 Bean이지만, `DelegatingFilterProxy` 가 아닌 `FilterChainProxy` 에 등록된다. `FilterChainProxy` 에 등록하는 것은 Servlet 컨테이너나 `DelegatingFilterProxy` 에 직접 등록하는 것보다 여러 가지 장점이 있다.

1. 스프링 시큐리티의 서블릿 지원을 위한 진입점으로, 트러블 슈팅 때 디버깅의 시작점으로 좋다.
2. 메무리 누수 방지를 위해 `SecurityContext` 를 초기화하고, `HttpFirewall` 을 적용하여 특정 유형의 공격으로부터 애플리케이션을 보호한다.
3. 서블릿 컨테이너에서는 URL만을 기반으로 필터가 호출되는데, `FilterChainProxy` 는 `RequestMatcher` 를 통해 `HttpServletRequest` 내 다양한 조건을 기반으로 호출을 결정할 수 있다.

> `SecurityContext` 와 메모리 누수와의 관계는 [[FilterChainProxy와 SecurityContext (feat. ThreadLocal)]] 글에 정리해놓았다.

<br/>

![multiple-security-filter-chain](/assets/img/posts/multiple-security-filter-chain.png)

위 그림을 살펴보면, `FilterChainProxy` 는 어떤 `SecurityFilterChain` 을 사용할지 결정한다. 가장 처음 일치하는 `SecurityFilterChain` 만 호출된다.
- `/api/messages/` 요청이 들어오면, `/api/**` 패턴을 가진 `SecurityFilterChain0` 과 먼저 일치하기 때문에 `SecurityFilterChain0` 만 호출된다.
- `/**` 패턴을 가진 `SecurityFilterChainN` 과도 일치하지만, 일단 첫 번째로 일치한 것만 실행된다.

0번째 FilterChain은 3개의 보안 필터 인스턴스로, N번째 FilterChain은 4개의 보안 필터 인스턴스로 구성되어 있다. 각 FilterChain마다 고유하게 설정할 수 있으며, 독립적으로 구성될 수 있다.

### Security Filters
보안 필터들은 `SecurityFilterChain` API를 통해 `FilterChainProxy` 에 삽입된다. 해당 필터들은 인증, 권한 부여, 공격 방지 등 다양한 목적으로 사용될 수 있다.

- 필터들은 특정 순서로 실행되는 것을 보장한다.
	- ex) 인증을 수행하는 필터가 권한 부여를 수행하는 필터보다 먼저 호출된다.
	- Filter 순서를 알고 싶다면, `FilterOrderRegistration` 코드를 확인하면 된다.

<br/>

```java
@Configuration // 빈을 등록하거나 의존성 설정
@EnableWebSecurity // Spring Security 활성화
public class SecurityConfig {
	
	// HttpSecurity 객체는 애플리케이션의 보안 설정 정의하는 데 사용되며,
	// 특정 보안 규칙들을 쉽게 설정할 수 있다.
	
	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http)
												throws Exception {
		http.csrf(Customizer.withDefaults()) // 1) CSRF 보호 기본값 설정
			// 3) authorizeHttpRequests: HTTP 요청에 대한 권한 부여 규칙 정의
			// 2) authorize.anyRequest().authenticated()
			//	: 모든 요청에 대한 인증이 필요, 즉 인증된 사용자만 요청에 접근 가능
			.authorizeHttpRequests(authorize
				-> authorize.anyRequest().authenticated())
			// 4) HTTP 기본 인증 활성화를 통해 클라이언트의 인증을 수행
			//  : 브라우저의 팝업창을 통해 아이디, 비밀번호 입력하여 인증 수행
			.httpBasic(Customizer.withDefaults())
			// 5) 폼 로그인 활성화
			//  : 사용자가 로그인 페이지에 입력한 자격 증명을 기반으로 인증 수행
			.formLogin(Customizer.withDefaults());

		// 최종 보안 설정을 SecurityFilterChain 객체로 빌드하고 반환함
		return http.build();
	}
}
```

1. 첫 번째로 `CsrfFilter` 가 호출되어, CSRF 공격으로부터 보호한다.
2. 두 번째로 인증 필터들이 호출되어, 요청을 인증한다.
3. 세 번째로 `AuthorizationFilter` 가 호출되어, 요청에 대한 권한을 부여한다.



