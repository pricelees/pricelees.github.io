---
emoji: '🌱'
title: 회원 역할에 따른 API 접근 - Spring Interceptor
date: '2024-06-24 19:00:00'
author: 이상진
tags: 안녕하세요!
categories: level2 Spring Interceptor
---

## 배경

[우아한테크코스의 두 번째 미션](https://github.com/pricelees/spring-roomescape-member/tree/step2)에서의 새로운 요구사항은 JWT를 이용해 로그인 기능을 구현하고, 역할에 따라 접근 권한을 다르게 설정하는 것이었습니다. 요구사항에 명시된 것은 **로그인 된 회원이 직접 예약을 추가하는 기능 구현**과 **관리자 페이지접근을 제한**하는 것이었는데요, 이 두 가지와 더불어 **관리자와 회원의 역할을 조금 더 명확하게 나누기 위해** Spring Interceptor를 이용했던 경험을 기록하고자 합니다.

## API 분류

우선, 현재 있는 API를 역할별로 구분해볼 필요가 있을 것 같습니다.

### 1. 로그인을 하지 않아도 접근 가능

| Endpoint | HTTP Method | 기능 |
| --- | --- | --- |
| /login | POST | 로그인 요청 |
| /themes/weekly | GET | 접속일 기준 지난 7일간 가장 많이 예약된 테마 10개 조회 |

`/themes/weekly` 는 인덱스 페이지에서 사용하기에 로그인이 되어있지 않아도 접근이 가능해야 합니다.

(일단 간단하게 작성하기 위해 `/weekly` 를 사용했고, `/themes/most-reserved-last-week?count=10` 와 같이 더 구체화 해서 사용할 수도 있겠습니다 ㅎㅎ)

### 2. 로그인 상태인 모든 회원(관리자 포함)

| Endpoint | HTTP Method | 기능 |
| --- | --- | --- |
| /reservations | POST | 회원이 예약을 추가 |
| /times/{date}/{themeId} | GET | 입력된 날짜와 테마에 대한 모든 예약 시간 조회 |
| /themes | GET | 모든 테마 조회 |
| /logout | POST | 로그아웃 |

날짜와 테마에 대한 모든 예약 시간 조회 기능은 이 시간이 예약된 시간인지에 대한 정보도 포함하고 있습니다. 예약된 시간이라면 예약 창에서 해당 시간을 선택할 수 없습니다. 또한, 회원은 테마를 선택할 수 있어야 하니 전체 테마를 조회하는 API에 역시 접근이 가능해야 합니다.

### 3. 관리자 전용

| Endpoint | HTTP Method | 기능 |
| --- | --- | --- |
| /reservations | GET | 모든 예약 조회 |
| /admin/reservastions | POST | 관리자가 예약을 추가 |
| /reservations/{id} | DELETE | 예약 취소(삭제) |
| /times | GET | 모든 예약 시간 조회 |
| /times | POST | 예약 시간 추가 |
| /times/{id} | DELETE | 예약 시간 삭제 |
| /themes | POST | 테마 추가 |
| /themes/{id} | DELETE | 테마 삭제 |
| /members | GET | 모든 회원 조회 |

다음 문단에서는, 인터셉터를 적용해보기 전에 **인터셉터 메서드와 호출 순서**에 대해 간단하게 확인해보겠습니다😄

## (참고) 인터셉터 등록 순서와 호출 순서

스프링 인터셉터를 처음 학습하며, 여러 인터셉터가 적용된 경우 어떻게 호출될지 궁금했습니다. 이 글에서 이용한 `preHandle()` 이외에도 인터셉터 적용 시점을 결정하는 여러 메서드가 있는데, 메서드 별로 호출 순서가 다르다는 것을 확인할 수 있었습니다. 우선 간략하게 소개하자면 다음과 같습니다.

- **preHandle()** : **핸들러가 실행되기 전에 호출**된다. 반환된 boolean 값이 `true`이면 핸들러를 실행합니다.
- **posthandle()** : **핸들러가 실행된 후, 즉 View를 생성하기 전**에 호출됩니다.
- **afterCompletion()** : **요청이 완료되고, View가 생성된 후** 호출됩니다.

같은 경로에 대한 인터셉터를 여러개 등록하면 어떤 순서로 호출이 되는지 직접 확인해 봤습니다.

우선 인터셉터와 컨트롤러 메서드는 다음과 같습니다. TestInterceptor2인 경우 출력을 “testInterceptor2: ..”로 하게 됩니다.

```java
public class TestInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        System.out.println("testInterceptor: preHandle");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
                           ModelAndView modelAndView) throws Exception {
        System.out.println("testInterceptor: postHandle");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
            throws Exception {
        System.out.println("testInterceptor: afterCompletion");
    }
}
```

```java
@GetMapping("/interceptor/test")
public void interceptorTest() {
	System.out.println("컨트롤러 메서드 호출");
}	
```

이제, 인터셉터를 다음과 같이 등록하고 테스트를 해보겠습니다. 같은 경로에 TestInterceptor, 2, 3 순서로 등록하였습니다.

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new TestInterceptor())
            .addPathPatterns("/interceptor/test");

    registry.addInterceptor(new TestInterceptor2())
            .addPathPatterns("/interceptor/test");

    registry.addInterceptor(new TestInterceptor3())
            .addPathPatterns("/interceptor/test");
}
```

```java
@Test
void 인터셉터_호출_순서_확인() {
    RestAssured.given().log().all()
            .when().get("/interceptor/test")
            .then().log().all()
            .statusCode(200);
}
```

RestAssured를 이용한 테스트에서의 출력값을 확인해보면 다음과 같습니다.

```java
testInterceptor1: preHandle
testInterceptor2: preHandle
testInterceptor3: preHandle

컨트롤러 메서드 호출

testInterceptor3: postHandle
testInterceptor2: postHandle
testInterceptor1: postHandle

testInterceptor3: afterCompletion
testInterceptor2: afterCompletion
testInterceptor1: afterCompletion
```

결론은 다음과 같습니다.

1. `preHandle → 컨트롤러 메서드 → postHandle → afterCompletion` 순으로 호출된다.
2. `preHandle()`의 경우 **등록한 인터셉터 순서대로 호출**된다.
3. `postHandle()`와 `afterCompletion()`의 경우 **등록한 인터셉터의 역순**으로 호출된다.

## 인터셉터 경로 직접 지정

이제 인터셉터를 적용 해 볼텐데요, 저는 두 가지 방법을 시도했고 그중 첫 번째 방법인 경로 지정 방법으로 먼저 구현해 보겠습니다. **로그인 된 회원(관리자 포함)이 이용 가능한 API**와 **관리자가 이용 가능한 API**를 구분하기 위해 우선 두 개의 인터셉터를 만들고, API를 호출하기 전에 인터셉터가 적용되어야 하므로, `preHandle()` 만 재정의 하였습니다.

```java
@Component
public class LoginInterceptor implements HandlerInterceptor {
		
		..
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        
        // 로그인 상태 확인 로직        
    }
}
```

```java
@Component
public class AdminInterceptor implements HandlerInterceptor {
		
		..
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        
        // 관리자 여부 확인 로직
        
	  }
}
```

인터셉터 적용의 편리를 위해, 위의 API 중 관리자 전용 API의 경우는 `/admin`을 추가로 붙였습니다.

| 수정 전  | 수정 후  |
| --- | --- |
| /reservations | /admin/reservations |
| /admin/reservstions | /admin/reservstions |
| /reservations/{id} | /admin/reservations/{id} |
| /times | /admin/times |
| /times | /admin/times |
| /times/{id} | /admin/times/{id} |
| /themes | /admin/themes |
| /themes/{id} | /admin/themes/{id} |
| /members | /admin/members |

인터셉터의 등록은, `WebMvcConfigurer` 를 구현한 객체를 만들고, `addInterceptors()` 를 재정의하면 됩니다.

```java
@Configuration
public class WebMvcConfiguration implements WebMvcConfigurer {

    private final LoginInterceptor loginInterceptor;
    private final AdminInterceptor adminInterceptor;

    public WebMvcConfiguration(LoginInterceptor loginInterceptor,
                               AdminInterceptor adminInterceptor) {
        this.loginInterceptor = loginInterceptor;
        this.adminInterceptor = adminInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/**")
                .excludePathPatterns("/", "/themes/weekly", "/login")
                .excludePathPatterns("/css/**", "/js/**", "/image/**", "/favicon.ico");

        registry.addInterceptor(adminInterceptor)
                .addPathPatterns("/admin/**");
    }
}
```

> addPathPatterns는 인터셉터를 적용할 경로를,
excludePathPatterns는 인터셉터를 적용하지 않을 경로를 지정할 때 사용합니다.
>

우선 로그인 인터셉터를 전체 경로에 대해 적용하고 css, js 등과 로그인 하지 않아도 접근이 가능해야 하는 경로를 지정합니다. (사실 처음 구현할 때는 직접 하나씩 지정했는데, 코드 리뷰를 받을 때 리뷰어께서 **전체를 등록하고 뺄 부분을 빼는 방식**을 추천해주셨고 이 방식이 더 직관적이고 간결하게 느껴져서 이렇게 사용하고 있습니다.)

관리자 전용 API의 Endpoint는 `/admin` 이 접두사가 되도록 수정했으니, /admin으로 시작하는 모든 경로에 관리자 인터셉터를 적용하였습니다.

```java
@ParameterizedTest
@ValueSource(strings = {"/", "/themes/ranking", "/login", "/css/flatpickr.css", "/js/flatpickr.js"})
void 로그인_하지_않아도_접근_가능하다(String path) {
    RestAssured.given().log().all()
            .when().get(path)
            .then().log().all()
            .statusCode(200);
}
```

테스트는 위와 같이 RestAssured를 이용해 진행할 수 있습니다😄

### 문제점

전체적으로 보면 간단한 방법인데, 지금의 방법에서 발생하는 몇몇 문제점이 있습니다.

1. 인터셉터는 요청 URL에 적용됩니다. 즉 HTTP 메서드에 따라 분리할 수 없습니다. 만약 `/reservations` 라는 API에 로그인 된 회원은 POST, 관리자는 GET 요청을 한다면 이를 구분할 수 없습니다.
2. 1의 문제점 때문에, 관리자 API를 `/admin` 으로 시작하도록 수정한 것인데 이렇게 하면 API를 바로 파악하기 쉽지 않습니다. 예약과 관련된 API는 모두 `/reservations` 로 통일하는게 가장 직관적인데, 지금은 관리자 예약의 경우 `/admin/reservations` 를 사용하기 때문입니다.
3. 개인적인 생각이지만, 코드를 보고 이해하기 힘들 것 같다는 생각입니다. 컨트롤러 메서드를 보고 바로 관리자 전용인지, 회원 전용인지 판단할 수 있으면 좋을 것 같습니다.

다음 문단에선, 커스텀 어노테이션을 이용한 인터셉터로 이 문제를 해결해 보겠습니다.

## 커스텀 어노테이션과 인터셉터

우선 컨트롤러 메서드에 붙일 로그인 / 관리자 어노테이션을 정의하겠습니다.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LoginRequired {
}
```

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AdminOnly {
}
```

목표는, **@LoginRequired**가 붙은 컨트롤러 메서드는 로그인 여부를 확인하고 **@AdminOnly**가 붙은 컨트롤러 메서드는 관리자 여부를 확인하는 것입니다!

다음으로, 이전에 정의한 `LoginInterceptor`와 `AdminInterceptor`에서 이 어노테이션을 확인하도록 하겠습니다.

```java
@Component
public class LoginInterceptor implements HandlerInterceptor {
		
		..
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        
        if (isLoginNotRequired(handler)) {
	        return true;
	      }
	      
	      // 나머지 로직(로그인 상태 확인)은 동일
    }
    
    private boolean isLoginNotRequired(Object handler) {
	    if (handler instanceof HandlerMethod handlerMethod) {
		    return !handlerMethod.hasMethodAnnotation(LoginRequired.class);
		  }
			return true;
		}
}
```

`isLoginNotRequired()` 는 **로그인 확인 인터셉터를 적용할 메서드를 구분하기 위해 사용**합니다. 다음의 경우에는 `preHandle()`이 `true`를 반환하도록 하여 로그인 인터셉터를 적용하지 않습니다.

1. 핸들러 객체가 HandlerMethod 인스턴스가 아닌 경우
2. 핸들러 객체가 HandlerMethod 인스턴스이지만 LoginRequired 어노테이션이 없는 경우

즉, `isLoginNotRequired()` 이 false를 반환했다는 것은 이 메서드에 `@LoginRequired` 어노테이션이 있다는 것이므로 이 경우에는 로그인 상태를 확인합니다.

마찬가지로, 관리자 인터셉터도 다음과 같이 수정할 수 있습니다.

```java
@Component
public class AdminInterceptor implements HandlerInterceptor {
		
		..
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        
        if (isAdminNotRequired(handler)) {
	        return true;
	      }
	      
	      // 나머지 로직(관리자 여부 확인)은 동일
    }
    
    private boolean isAdminNotRequired(Object handler) {
	    if (handler instanceof HandlerMethod handlerMethod) {
		    return !handlerMethod.hasMethodAnnotation(AdminOnly.class);
		  }
			return true;
		}
}
```

이렇게 인터셉터를 구현했다면, 이전의 `WebMvcConfiguration` 객체를 다음과 같이 수정합니다. 이전과 같이 구체적인 경로를 지정하지 않고, 인터셉터 자체만 등록하면 됩니다!

> 지금은 Interceptor 내부에서 사용하는 객체가 있기 때문에 Autowired를 이용했으나, 만약 인터셉터가 별도의 필드를 가지고 있지 않다면 `new LoginInterceptor()` 와 같이 대체할 수 있습니다. ([참고](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/interceptors.html))
>

```java
@Configuration
public class WebMvcConfiguration implements WebMvcConfigurer {

    private final LoginInterceptor loginInterceptor;
    private final AdminInterceptor adminInterceptor;

    public WebMvcConfiguration(LoginInterceptor loginInterceptor,
                               AdminInterceptor adminInterceptor) {
        this.memberIdResolver = memberIdResolver;
        this.loginInterceptor = loginInterceptor;
        this.adminInterceptor = adminInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor);
        registry.addInterceptor(adminInterceptor);    
    }
    
    ..
}

```

이제 컨트롤러 메서드에 커스텀 어노테이션을 붙여 이전 문단에서 언급한 문제점을 해결해 보겠습니다.

```java
@Controller
public class ReservationController {
	..
	
	@LoginRequired
	@PostMapping("/reservations")
	public .. createMemberReservation(@Authenticated Long memberId ..) {
		..
	}
	
	@AdminOnly
	@GetMapping("/reservations")
	public .. getAllReservations() {
		..
	}
	
	..
}
```

이전 문단에 작성한 문제점을 다시 한번 작성해 보겠습니다.

1. 인터셉터는 요청 URL에 적용됩니다. 즉 HTTP 메서드에 따라 분리할 수 없습니다. 만약 `/reservations` 라는 API에 로그인 된 회원은 POST, 관리자는 GET 요청을 한다면 이를 구분할 수 없습니다.
    - 같은 Endpoint에 접근 권한을 나눌 수 있습니다.
2. 1의 문제점 때문에, 관리자 API를 `/admin` 으로 시작하도록 수정한 것인데 이렇게 하면 **API를 바로 파악하기 쉽지 않습니다.** 예약과 관련된 API는 모두 `/reservations` 로 통일하는게 가장 직관적인데, 지금은 관리자 예약의 경우 `/admin/reservations` 를 사용하기 때문입니다.
    - 1의 문제가 해결되었기 때문에 관리자 전용 엔드포인트를 수정할 필요가 없어졌습니다.
3. 개인적인 생각이지만, 코드를 보고 이해하기 힘들 것 같다는 생각입니다. 컨트롤러 메서드를 보고 바로 관리자 전용인지, 회원 전용인지 판단할 수 있으면 좋을 것 같습니다.
    - 이전에는 `WebMvcConfiguration` 에 등록된 인터셉터 적용 경로와 컨트롤러 메서드를 둘다 확인해야 했지만, 이제 컨트롤러 메서드 안에서 모두 파악할 수 있게 되었습니다.

## 🧐 ArgumentResolver 사용?

Spring Interceptor가 아닌 ArgumentResolver를 이용하여 비슷한 방식으로 해결할 수도 있습니다. 아래의 코드처럼 `PARAMETER` 에 적용되는 어노테이션을 만들고, 이를 이용한 ArgumentResolver를 구현합니다. Admin은 말 그대로 관리자, Authenticated는 로그인 된 회원을 조회할 때 사용하겠습니다.

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Admin {
}

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Authenticated {
}
```

모든 테마를 조회하는, 로그인이 필요한 `GET /themes` 를 예시로 들어보면 컨트롤러 메서드를 아래와 같이 구현할 수 있습니다. 인터셉터 코드는 위에서 구현한 커스텀 어노테이션 방식을 이용했습니다.

```java
// Spring Interceptor 사용
@GetMapping("/themes")
@LoginRequired
public .. getAllThemes() {
	..
}

// ArgumentResolver 사용
@GetMapping("/themes")
public .. getAllThemes(@Authenticated Long memberId) {
	..
}
```

만약 모든 접근 권한이 필요한 컨트롤러 메서드가 `memberId` 라는 값을 필요로 한다면 `ArgumentResolver`를 사용하는게 더 간단할 것이라고 생각합니다. 하지만, `GET /themes` 와 같은 요청에는 예약 추가와 같이 회원 ID를 직접적으로 사용할 일이 없고, 단순히 권한 체크에만 사용하기 때문에 Interceptor를 사용하는 것이 더 낫다고 판단했습니다.

## 결론

### 요약

1. 특정 API들에 대한 접근 권한을 분류하기 위해 Spring Interceptor를 사용할 수 있다.
2. 인터셉터를 만들고, WebMvcConfigurer에서 경로를 직접 지정할 수 있으나, 이 방법은 **동일한 경로에 다른 HTTP 메서드를 사용하는 경우를 구분할 수 없다**는 큰 단점이 존재한다.
3. 커스텀 어노테이션을 통해 인터셉터를 구현하는 방법으로 이 문제를 해결할 수 있다.
4. 모든 경우에 메서드 파라미터를 사용한다면 ArgumentResolver의 사용을 고려해 볼 수 있다.

미션을 진행할 때는 저도 간단한 사용법만 익히고 뚜다다닥(?) 사용했었는데, 글을 작성하는 과정에서 다시 하나하나 확인해보는 즐거움이 있네요 ㅎㅎ

부족하거나 잘못된 내용이 있다면 댓글에 남겨주시면 정말 감사드리겠습니다. 즐거운 하루 보내세요🙇