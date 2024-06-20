---
emoji: '🌱'
title: Layered Architecture 적용 과정에서의 DTO 사용에 대한 고민
date: '2024-06-17 19:00:00'
author: 이상진
tags: 안녕하세요!
categories: level2 Spring 
---

## 배경

[우아한테크코스의 첫 스프링 미션](https://github.com/pricelees/spring-roomescape-admin/tree/step2)을 진행하며, 처음으로 **레이어드 아키텍쳐**를 적용해보는
경험을 하게 되었습니다.


Controller에서 **JdbcTemplate**을 사용하여 DB 처리를 하는 기존 코드에 Service - Repository 계층을 추가하는 과정에서 자연스레 들었던 고민은 **도메인과 DTO의 처리를 어떻게
해야할지**였고, 이때 가졌던 생각의 흐름을 정리해보고자 합니다.


아래의 예시는 **예약을 추가(저장) 하는 기능**을 바탕으로 작성하였습니다😄



<br/>

## 개요

### API의 구성

```html
POST /reservations HTTP/1.1
content-type: application/json

{
  "date": "2024-04-20",
  "name": "상돌",
  "timeId": 1
}
```


예약을 추가할 때는 `이름, 날짜, 시간 ID값` 을 JSON 형태로 요청합니다. 예약 시간은 별도의 데이터베이스 테이블에 저장을 해둔 뒤, 시간의 ID값을 이용하여 조회합니다.


```html
HTTP/1.1 200
Content-Type: application/json

{
  "id": 1,
  "name": "상돌",
  "date": "2024-04-20",
  "time" : {
    "id": 1,
    "startAt" : "10:00"
  }
}
```


요청이 들어오면 **ID값은 자동으로 채운 뒤**(AUTO_INCREMENT), 입력된 시간 ID에 해당되는 예약 시간까지 불러와서 저장된 예약 내역을 응답하는 구조입니다.

<br/>

### 도메인


```java
public class ReservationTime {

    private final Long id;
    private final String startAt;

    // 생성자 및 Getter..
}
```


```java
public class Reservation {

    private final Long id;
    private final String name;
    private final String date;
    private final ReservationTime time;

    // 생성자 및 Getter..
}
```


여기서 날짜와 시간은 각각 LocalDate와 LocalTime을 사용할 수 있으나, 이번 단계에선 고려하지 않았습니다😄


<br/>

### 요청 DTO

```java
public record ReservationRequest(String name, String date, Long timeId) {
}
```


- 요청 DTO는 API 명세에 따라 이름, 날짜, 시간 ID값으로 구성되어 있습니다.

<br/>

### 응답 DTO


```java
public record TimeResponse(Long id, String startAt) {

    public TimeResponse(ReservationTime reservationTime) {
        this(reservationTime.getId(), reservationTime.getStartAt());
    }
}
```


- 예약 시간에 대한 응답 DTO입니다. ID와 시간으로 이루어져 있습니다.


```java
public record ReservationResponse(Long id, String name, String date, TimeResponse time) {

    public ReservationResponse(Reservation reservation) {
        this(reservation.getId(), reservation.getName(), reservation.getDate(),
                new TimeResponse(reservation.getTime()));
    }
}
```


- 예약에 대한 응답 DTO입니다. API 명세와 동일하게 ID, 이름, 날짜, 시간으로 구성되어 있습니다.

<br/>

## 첫 번째 시도

### 구현 방법

DTO 처리와 관련하여 열심히 구글링을 해본 결과, 크게 다음의 두 가지 의견으로 나뉘는 것을 확인하였습니다.


1. **Controller에서 요청 DTO를 도메인으로 변환**하고 Service에 전달한다.

2. **Controller에서 요청 DTO를 Service로 전달하고, Service에서 도메인으로 변환**한 뒤 Repository로 전달한다.


저는 아래의 **(지극히 주관적인)이유**로 **1번 방법**을 선택하였습니다.


1. DTO는 도메인에 비해 변경 가능성이 크다.

2. 서비스에서 DTO를 처리하면, DTO가 변하면 서비스의 로직도 수정해야 한다.

3. 이전 레벨에서, 로직은 가급적 수정하지 않는 것이 좋다는 것을 크게 느낄 수 있었기에 가급적 수정하지 않는 방향으로 구현하고 싶었다.


<br/>
코드는 다음과 같습니다.
<br/>

<U>**Controller**</U>

```java
@PostMapping
public ReservationResponse addReservation(@RequestBody ReservationRequest request) {
    ReservationTime time = new ReservationTime(null, request.getTimeId());
    Reservation reservation = new Reservation(null, request.getName(), request.getDate(), time);

    Reservation created = reservationService.addReservation(reservation);

    return new ReservationResponse(created);
}
```


<U>**Service**</U>

```java
@Service
public class ReservationService {

    private final ReservationDao reservationDao;
    private final ReservationTimeDao reservationTimeDao;

    // 생성자..

    public Reservation addReservation(Reservation reservation) {
        ReservationTime reservationTime = reservationTimeDao.findById(reservation.getTime().getId());
        Reservation reservationForAdd = new Reservation(
                reservation.getId(), reservation.getName(), reservation.getDate(), reservationTime
        );

        return reservationDao.add(reservationForAdd);
    }
}
```

<br/>

### 문제점


일단 구현 자체는 완료되었으나, **불편함이 느껴지는 몇몇 문제**가 있었습니다.


<U>**문제1. 컨트롤러에서 객체를 생성할 때 null 값을 입력하는 것**</U>


위의 컨트롤러 코드를 보면, `ReservationTime, Reservation`  객체를 생성할 때 null을 입력합니다. 보기 안 좋은 것도 있고, 전체 맥락을 모르는 사람이 이 코드를 보고 null값을 바로
AUTO_INCREMENT 되는 ID로 생각할 수 있을까? 에 대한 의문이 들었습니다.


하지만, 이 부분은 다음과 같이 **생성자를 추가하여 해결**할 수도 있기에 큰 문제라고 생각하진 않았습니다.


```java
public class ReservationTime {

    private final Long id;
    private final String startAt;

    public ReservationTime(String startAt) {
        this(null, startAt);
    }

    // 다른 생성자 및 Getter..
}
```
<br/>

<U>**문제2. 객체를 여러번 생성해야 하고, getter의 호출이 많은 것**</U>


컨트롤러에서도 ReservationTime과 Reservation 객체를 생성하는데, **서비스에서도 똑같이 생성합니다.** 특히 시간 ID로 ReservationTime 객체를 조회할 때, **그냥 ID로 한번에
조회**하면 되는 것을 `컨트롤러에서 ReservationTime 객체 생성 → 이 객체로 컨트롤러에서 Reservation 객체 생성 → Service에서 Reservation 객체에 Getter를 두 번 적용하여 시간 ID값을 꺼냄`
이라는 불필요하게 복잡한 방법을 사용한다는 생각을 지울 수 없었습니다.

<br/>

<U>**⭐️문제3. 컨트롤러가 도메인을 알고 있어야 한다.**</U>


문제1에서 느낀 것인데, 컨트롤러에서 null값을 넣는 것을 떠나서 **ID 필드가 있다는 것을 알아야 하나?** 라는 의문이 들었습니다. 컨트롤러에 도메인 객체가 있게 되면 실제 요청 / 응답에 사용되는 값 이외의
다른 값들이 노출될 여지가 있다고 생각했습니다.

<br/>

## 두 번째 시도

### 구현 방법

이전의 문제2,3이 생각보다 크게 느껴져서 이번에는 **컨트롤러에서 DTO를 서비스로 그대로 전달하고, 서비스에서 도메인으로 변환하는** 방법을 시도하였습니다. 응답할 때도 마찬가지로 **서비스에서 도메인을 DTO로
변환하여 컨트롤러에 전달**하도록 수정하였습니다.


<U>**Controller**</U>

```java
@PostMapping
public ReservationResponse addReservation(@RequestBody ReservationRequest request) {
    return reservationService.addReservation(request);
}
```


<U>**Service**</U>


```java
public ReservationResponse addReservation(ReservationRequest request) {
    ReservationTime reservationTime = reservationTimeDao.findById(request.timeId());
    Reservation reservationForAdd = new Reservation(
            request.name(), request.date(), reservationTime
    );

    Reservation created = reservationDao.add(reservationForAdd);
    return new ReservationResponse(created);
}
```


**이전 문단의 문제1에서 언급한 해결책인 생성자 추가**와 더불어 getter도 이천처럼 과하게 사용하지 않고, 코드가 훨씬 간결해졌습니다. 지금 단계에서는 DTO가 변경될 가능성도 크게 없고, 별도의 로직이라고
할 법한 것도 없기에 간결하다는 장점이 명확한 이 방법을 최종적으로 사용하였습니다.

<br/>

### 문제점


코드가 간결해진다는 장점이 정말 크게 다가오기 때문에 이 방법을 사용하였으나, 예상되는 문제점이 없는 것은 아니었습니다.


<U>**문제1. DTO의 변화가 컨트롤러와 서비스 레이어에 모두 영향을 주게 된다.**</U>


같은 DTO를 사용하기에, 이 DTO의 변화가 두 레이어에 영향을 주게 되며, 추가적으로 **요청이라는 View의 정보가 Service까지 영향을 미친다**는 생각 역시 할 수 있습니다.


가장 간단하게 떠오르는 해결 방법은 **서비스용 DTO를 별도로 생성**하고, 컨트롤러에서 변환한 뒤 넘기는 것인데 지금의 구조에선 **두 DTO가 어차피 같은 값을 가져야 하기에** 큰 의미가 없다는
생각이었습니다.


<br/>

<U>**문제2. (서비스에서 응답 DTO를 반환하는 경우) 서비스간에 의존한다면?**</U>


서비스에서 도메인이 아닌 DTO를 반환한다면 서비스간의 의존이 존재하는, 다음의 예시와 같은 문제가 발생할 수 있습니다. 하지만 지금의 구조에선 서비스간의 의존이 없고, 컨트롤러에 도메인 객체를 노출하는 것이 더
좋지 않은 방법으로 느껴졌기에 서비스에서 응답 DTO를 반환하는 지금의 구조를 유지하였습니다.


**예시**


```java
@Service
public class ReservationTimeService {
	..

    public ReservationTimeResponse findById(Long id) {
		..
    }
}
```


- ReservationTimeService에서 findById()는 ReservationTimeResponse라는 DTO를 반환합니다.


```java
@Service
public class ReservationService {

    private final ReservationDao reservationDao;
    private final ReservationTimeService timeService;

    public ReservationResponse addReservation(ReservationRequest request) {
        ReservationTimeResponse timeResponse = timeService.findById(request.timeId());

        // ReservationTimeResponse를 ReservationTime으로 변환하는 로직 필요
        ReservationTime reservationTime = ...

        Reservation reservationForAdd = new Reservation(
                request.name(), request.date(), reservationTime
        );

        Reservation created = reservationDao.add(reservationForAdd);
        return new ReservationResponse(created);
    }
}
```

- 예약을 추가하려면 Reservation 객체가 필요한데, 이 객체를 생성하려면 ReservationTime이 필요합니다.

- 만약 ReservationTimeService에서 DTO를 반환하는 경우, 이 DTO를 도메인으로 변환하는 추가적인 연산이 필요해지게 됩니다.

<br/>

## 결론 및 요약


이번 단계의 미션을 진행하며 했던 여러 시행착오와 고민 끝에 다음의 결론을 내렸습니다.

- 요청 시 컨트롤러에서 DTO를 도메인으로 변환하지 않는다.
    - 컨트롤러에서 받는 요청값과 서비스에서 사용하는 값이 다른 경우 서비스 전용 DTO를 만든다.
  
    - 컨트롤러에서 받는 요청값을 서비스에서 그대로 사용한다면, 요청 DTO를 서비스로 그대로 전달한다.


- 서비스간의 의존이 없도록 구현하고, 서비스에서 응답 DTO를 반환하도록 구현한다.


지금 내린 결론이 당연히 정답은 아니기에 이후의 과정에서 방법이 변할 수도 있겠지만, 새로운 구현을 해야 할 때 고민하지 않고 바로 나아갈 방향을 정했고, 상황에 맞는 방법을 선택할 수 있게 되었다는 점에서
정말 의미있는 경험을 했다고 생각합니다😄

읽어주셔서 감사합니다! 즐거운 하루 보내세요🙇


```toc
```