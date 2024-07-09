---
emoji: '🌱'
title: 스프링 입력에서의 예외 처리 여정 1 - 문제 인식 및 일부 해결
date: '2024-06-18 19:00:00'
author: 이상진
tags: 안녕하세요!
categories: level2 Spring Exception
---

## 배경

[우아한테크코스의 두 번째 스프링 미션](https://github.com/pricelees/spring-roomescape-member/tree/step2)을 통해 처음으로 스프링의 예외 처리를 경험하게 되었습니다. 

미션의 요구 사항은 **예약, 테마, 시간 생성시 발생하는 예외를 적절하게 처리**하는 것이었는데요, `지나간 날짜에 대한 예약은 불가능하다` 와 `입력된 ID에 대한 값이 DB에 없는` 것과 같은 예외 처리는 크게 어렵지 않았으나 입력에서의
예외, 즉 **입력되지 않은 값 등 값 자체가 잘못되었을 때**에 대한 처리가 가장 어려웠습니다.

가장 많이 헤맸던 이유는 **요청 상황마다 발생하는 예외 타입이 달랐기 때문**인데요, 이번 글에서는 해당 문제와 이 문제를 해결해가는 과정에 대해 기록해보려 합니다.

<U>**주의**</U>

스프링을 이번에 처음 사용하게 되어, 내용이 매우 부실할 수 있습니다. 이 글은 지식을 전달하는 글이 아닌, 개인의 시행착오 과정을 기록하는 글임을 감안해주시면 감사하겠습니다.🙇

<br/>

## API 구성

### API - 관리자가 직접 예약을 추가

```json
POST /reservations HTTP/1.1
content-type: application/json

{
   "date": "2024-05-10",
   "memberId": "1",
   "themeId": "1",
   "timeId": "1"
}
```

관리자가 직접 예약을 추가할 땐, 예약 페이지에서 이미 등록된 회원, 테마, 시간과 날짜를 선택합니다. 이때 **회원, 테마, 시간은** **DB에 저장된 ID값으로 요청**됩니다. (DB에 회원1의 ID가 1로
저장되어있다면, 관리자는 회원1을 선택하는데 요청은 이 회원의 ID인 1로 보내집니다)

<br/>

### API - 테마 추가

```json
POST /themes HTTP/1.1
content-type: application/json

{
   "description": "테마 설명",
   "name": "테마 이름",
   "thumbnail": "테마 썸네일 사진 URL"
}
```

테마를 추가할땐, 이름, 설명, 썸네일 주소를 입력받습니다. 이 값은 모두 문자열입니다.

<br/>

### API - 시간 추가

```json
POST /times HTTP/1.1
content-type: application/json

{
    "startAt": "HH:mm"  
}
```

시간을 추가할 땐, HH:mm 형태의 시간값을 입력받습니다.

<br/>

## 문제 상황

### 테마와 시간을 추가할 때 값을 입력하지 않는다면?- MethodArgumentNotValidException

```java
public record ThemeCreateRequest(
        @NotBlank(message = "테마 이름을 입력해 주세요.")
        String name,
        @NotBlank(message = "테마 설명을 입력해 주세요.")
        String description,
        @NotBlank(message = "썸네일 주소를 입력해 주세요.")
        String thumbnail
) {
}
```

위의 코드는, 테마를 추가할 때 사용하는 요청 DTO의 코드입니다. 위의 테마 추가 API 명세에 맞게 구성되어 있으며, `@NotBlank` 를 통해 빈 값을 입력받지 않도록 구성하였습니다.

> `@NotBlank` 어노테이션은 해당 필드가 null이거나 빈 문자열("")이 아닌지를 확인합니다.

<br/>

**만약 값을 입력하지 않으면** 어떤 형태로 요청이 될까요? 테마를 추가할 때 이 세 가지의 값을 입력하지 않으면 요청 JSON은 다음과 같은 형태로 전송됩니다.

```java
POST /themes HTTP/1.1
content-type:application/json 

{
    "description": "",
    "name": "",
    "thumbnail": ""
}
```
<br/>

빈 값이 입력되었을 때 발생되는 에러의 원인을 확인해보기 위해, 아래와 같이 간단한 ExceptionHandler를 만들어 스택 트레이스를 출력해 보겠습니다 ㅎㅎ

```java
@ExceptionHandler(value = Exception.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ErrorResponse handleException(Exception e) {
    e.printStackTrace();
}
```

<br/>

스택 트레이스를 통해 발생되는 예외를 확인해보니, 다음과 같이 `MethodArgumentNotValidException`이 발생하는 것을 확인할 수 있었습니다. **시간의 경우**도 `startAt` 값이 빈
문자열로 입력되어 같은 타입의 예외가 발생합니다.

```java
org.springframework.web.bind.MethodArgumentNotValidException:
Validation failed for...
```

<br/>

### ⭐️ 예약을 추가할 때 값을 입력하지 않는다면? - InvalidFormatException

테마와 시간을 추가할 때, 값을 입력하지 않으면 `빈 문자열` 형태로 요청되는 것을 확인하였기에, **예약을 추가할 때 역시 값을 입력하지 않으면 빈 값으로 처리될 것이라고 생각**했습니다. 그래서 예약 페이지에서
모든 값을 선택하지 않고 요청을 보냈는데…

```java
POST /reservations HTTP/1.1
content-type:application/json 

{
    "date":"",
    "memberId":"멤버 선택",
    "themeId":"테마 선택",
    "timeId":"시간 선택"
}
```

위와 같이 <U>**날짜는 빈 값으로 입력되었으나 회원, 테마, 시간의 경우 각각 “멤버 선택”, “테마 선택”, “시간 선택”이라는 값으로 요청이 되고 있었습니다.**</U>

<br/>

```java
public record AdminReservationCreateRequest(
        @NotNull(message = "날짜를 입력해 주세요.") LocalDate date,
        @NotNull(message = "시간을 입력해 주세요.") Long timeId,
        @NotNull(message = "테마를 입력해 주세요.") Long themeId,
        @NotNull(message = "회원을 입력해 주세요.") Long memberId
) {
}
```

그래서 이 Request DTO 코드에서 date값을 제외하고 `@NotNull` 이 작동하지 않는 상황이었습니다. (date의 경우 값을 입력하지 않으면 빈 문자열로
요청되어 `MethodArgumentNotValidException`이 발생합니다.)

<br/>

마찬가지로, 이전에 작성한 ExceptionHandler를 이용하여 스택 트레이스를 출력해 보겠습니다. 

```java
org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Cannot deserialize value of type `java.lang.Long` from String "멤버 선택": not a valid `java.lang.Long`value
....
Caused by:com.fasterxml.jackson.databind.exc.InvalidFormatException: Cannot deserialize value of type `java.lang.Long` from String "멤버 선택": not a valid `java.lang.Long`value
..
```

출력된 결과를 보고 아래의 두 가지를 유추할 수 있었습니다. 

1. `“멤버 선택”`이라는 **문자열을 Long 타입으로 변환할 수 없기에** `InvalidFormatException`이 발생한다.
2. `InvalidFormatException`이 발생해도, 발생되는 예외 타입은 `HttpMessageNotReadableException` 이다.

<br/>

**(번외)**

실제로 스택 트레이스를 보면 `AbstractJackson2HttpMessageConverter.readJavaType` 에서 예외가 발생했다고 하는데, 이 소스코드를 보면 **try - catch**로 특정 예외가
발생하면 `HttpMessageNotReadableException`을 던지는 것을 확인할 수 있습니다.

<br/>

### 요약

결론적으로 현재 상황에서는 `MethodArgumentNotValidException` 과 `InvalidFormatException` 으로 예외 처리를 나눌 수 있겠다고 판단했습니다.

- **MethodArgumentNotValidException** 은 날짜, 시간, 테마 이름, 테마 설명, 테마 썸네일
- **InvalidFormatException(HttpMessageNotReadableException)** 은 회원 ID, 테마 ID, 시간 ID

<br/>

## 예외 응답 DTO

본격적으로 예외 처리를 해보기 전에, 다음과 같은 간단한 **예외 응답용 DTO**를 구성해 보겠습니다. 

```java
public record ErrorResponse(String url, String method, String message) {
}
```

이 DTO는 **요청 url, 요청 HTTP 메서드, 예외 메시지**로 구성하였습니다.(**이번 단계에서는 실제 응답 값으로 디버깅을 했기 때문에** 위와 같이 구성했고, 이후에 로깅을 시도함에 따라 예외 응답
객체의 구성을 바꾸게 되었습니다 ㅎㅎ)

<br/>

## MethodArgumentNotValidException 처리

우선, `@Valid`  어노테이션 검증으로 인해 발생하는 `MethodArgumentNotValidException` 을 먼저 처리하고자 했습니다. 방법을 모르니 열심히 구글링한
결과 [빛과 같은 baeldung의 글(빛덩..)](https://www.baeldung.com/global-error-handler-in-a-spring-rest-api#1-handling-the-exceptions)
을 찾게 되었고 이곳에 작성된 방법을 참고하여 진행하였습니다.

예외 메시지를 작성하기에 앞서, 만약 **여러 개의 값에서 예외가 발생하면** 어떻게 할까요? **모든 예외 메시지를 합쳐서 응답**할수도 있고, **그 중 하나의 값에 대한 메시지만 응답**할 수도 있습니다. 당연히 처음 하는 만큼 두
방법 모두를 시도하였습니다.

<br/>

### 1. 여러 값에서 예외가 발생되도 하나만 출력하자

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ErrorResponse handleMethodArgumentNotValidException(HttpServletRequest request,
                                                           MethodArgumentNotValidException e) {
    String errorMessage = e.getBindingResult().getFieldError().getDefaultMessage();

    return new ErrorResponse(request.getRequestURI(), request.getMethod(), errorMessage);
}                
```

Baeldung의 글에서는 `getFieldErrors()` 를 이용하여 모든 예외를 리스트로 받지만, `getFieldError()` 라는 하나의 예외만 받아오는 메서드가 있었고 이를 활용하였습니다.

하지만, 이 방법은 좋은 방법이 아니었습니다. 만약 테마에서 이름, 설명, 썸네일 모두를 입력하지 않게 되면 **어떨 때는 “설명을 입력해 주세요” 가 반환되고, 어떨 때는 “썸네일을 입력해 주세요”가 반환되는 등
매번 다른 결과가 나왔기에** 특정 상황에 대한 정확한 예외 메시지를 예측할 수 없는 문제가 있었습니다.

- 테스트를 할 때, 단순히 HTTP 코드로 확인하는 경우 문제가 없겠지만 구체적인 예외 메시지에 대한 테스트는 불가능해지게 됩니다.

<br/>

### 2. 여러 값에서 예외가 발생하면, 모든 예외 메시지를 합쳐서 출력하자.

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ErrorResponse handleMethodArgumentNotValidException(HttpServletRequest request,
                                                           MethodArgumentNotValidException e) {
    String errorMessage = e.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.joining(", "));

    return new ErrorResponse(request.getRequestURI(), request.getMethod(), errorMessage);
}        
```

Baeldung에서는 For문을 이용하여 처리하지만, Stream을 이용하여 모든 예외 메시지를 `“, “`로 구분하여 합치는 방식으로 구현하였습니다.

<br/>

### 결과

```json
// 모든 값을 입력하지 않는 경우
{
  "url": "/themes",
  "method": "POST",
  "message": "테마 설명을 입력해 주세요, 썸네일 주소를 입력해 주세요, 테마 이름을 입력해 주세요"
}

// 테마 설명만 입력한 경우
{
  "url": "/themes",
  "method": "POST",
  "message": "썸네일 주소를 입력해 주세요, 테마 이름을 입력해 주세요"
}
```

지금은 값을 입력하지 않은(빈 값인) 경우만 고려하지만, `@Valid` 어노테이션으로 지정한 모든 예외는 이 방법으로 처리할 수 있었습니다. 

<br/>

## InvalidFormatException 처리

### 들어가기 전에..

```java
@ExceptionHandler(Exception.class)
@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
public ErrorResponse handleException(HttpServletRequest request, Exception e) {
    e.printStackTrace();
    return new ErrorResponse(request.getRequestURI(), request.getMethod(), e.getMessage());
}
```

**ControllerAdvice**에는 핸들러가 없는 예외 처리를 위해, 위와 같은 Exception 타입을 처리하는 핸들러가 정의되어 있습니다.

<br/>

```java
@ExceptionHandler(InvalidFormatException e)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ErrorResponse handleInvalidFormatException(HttpServletRequest request, InvalidFormatException e) {
    ..
}
```

이 상황에서, 위와 같이 InvalidFormatException 타입을 처리하도록 구현하면, 예외 발생시 Exception 타입 핸들러가 예외를 처리하게 됩니다.

- **실제 발생한 예외는 InvalidFormatException 이지만, Jackson은 HttpMessageNotReadableException 타입으로 예외를 발생시키기 때문입니다!**

따라서, 이후의 과정에서는 InvalidFormatException이 아닌 `HttpMessageNotReadableException`으로 해당 예외를 처리하겠습니다.

<br/>

### 우선 다 뽑아보겠습니다.

```java
@ExceptionHandler(HttpMessageNotReadableException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ErrorResponse handleHttpMessageNotReadableException(HttpServletRequest request,
                                                           HttpMessageNotReadableException e) {
    System.out.println("e.getMessage() = " + e.getMessage());

    return null;
}
```

글이 길어 다시 간략하게 작성하자면, HttpMessageNotReadableException은 **관리자가 예약을 추가할 때 시간, 테마, 회원을 입력하지 않는 경우에 발생**합니다. 이 예외 타입은
이전처럼 `getBindingResult()` 와 같은 기능도 없기에.. 무작정 가능한 모든 것을 출력해 보았습니다.. 라고 하지만 사실 `getMessage()` 밖에 없더라구요 ㅎㅎ..

```java
e.getMessage() = JSON parse error: Cannot deserialize value of type `java.lang.Long` from String "테마 선택": not a valid `java.lang.Long`value
```

출력 결과는 위와 같이 되는데, 앞 부분은 크게 의미 없고 뒤에 있는 **“테마 선택”** 부분은 활용할 수 있을 것 같습니다.

<br/>

### 1차 시도

```java
@ExceptionHandler(HttpMessageNotReadableException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ErrorResponse handleHttpMessageNotReadableException(HttpServletRequest request,
                                                           HttpMessageNotReadableException e) {
    String message = e.getMessage();
    String errorField = "";

    if (message.contains("멤버")) {
        errorField = "멤버";
    }
    if (message.contains("시간")) {
        errorField = "시간";
    }
    if (message.contains("테마")) {
        errorField = "테마";
    }

    String responseMessage = errorField + "을(를) 입력해 주세요.";
    return new ErrorResponse(request.getRequestURI(), request.getMethod(), responseMessage);
}
```

단순하게, 이전에 발생한 예외 메시지를 참고하여 String의 contains를 이용하여 처리하는 방식입니다.

<br/>

### 문제점

일단 처리 자체는 되었지만, 문제점이 정~말 많은 방식이라고 느껴졌습니다.

1. 지금의 예외 핸들링은, 테마, 시간, 멤버를 선택하지 않았을 때 기본 입력값인 `“테마 선택”` 등에 의존합니다. **즉 클라이언트 코드에 완전히 의존하는 구조입니다.**
2. 멤버, 시간, 테마 중 2개 이상의 값이 입력되지 않아도 하나의 값만 표시됩니다. 즉 **입력되지 않은 모든 값을 예외 메시지에 담을 수 없습니다.**
   - 추가적으로, 날짜를 입력하지 않았을 때 발생하는 MethodArgumentNotValidException와 같이 묶어서 처리할 수도 없습니다!

<br/>

## 다음 편에 계속됩니다..

여러 방법을 찾아보던 도중 `Custom Deserializer`를 이용하여 이 문제들을 해결할 수 있었는데요, `Custom Deserializer`와 `JsonNode`와 같은 내용에 대한 고민 과정까지
작성하기엔 글이 너무 길어지겠다는 생각에 이 부분은 다음 편에 별도로 작성하겠습니다 🙇

여기까지 읽어주셔서 감사합니다. 즐거운 하루 보내세요😄

```toc
```