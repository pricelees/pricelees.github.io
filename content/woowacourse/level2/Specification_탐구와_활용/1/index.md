---
emoji: '🌱'
title: Spring Data JPA 에서의 동적 쿼리 1 - Specification 탐구
date: '2024-06-27 19:00:00'
author: 이상진
tags: 안녕하세요!
categories: level2 Spring Specification
---

## 배경

[우아한테크코스의 세 번째 스프링 미션](https://github.com/woowacourse/spring-roomescape-waiting/pull/30)의 첫 번째 요구사항은, 기존의 JDBC 구현을 Spring Data JPA로 바꾸는 것이었습니다. 다른 부분은 크게 어렵지 않았지만, 관리자가 특정 조건에 해당하는 예약을 조회하는 기능이 그나마(?) 어려웠던 것 같습니다.

예약 검색은 `특정 회원 / 특정 테마 / 시작 날짜 / 종료 날짜` 를 이용하는 것이었는데요, 저는 이전 미션에서 이 네 가지 값이 모두 입력되지 않으면 예외를 발생시켰는데 페어는 동적 쿼리를 이용해서 네가지 값이 모두 입력되지 않으면 전체 예약을 조회하고, 하나의 값만 입력되도 그 값으로 조회하도록 구현을 했었습니다.

결과적으로는 동적 쿼리로 조회하는 것이 사용자에게 더 편하겠다고 생각했고, Spring Data JPA의 Specification을 이용하여 이를 구현할 수 있었습니다. 그래서 **이번 글에서는 Specification에 대해 다루고**, 다음 글에서는 이를 활용하여 문제를 해결해 보겠습니다😄

## Specification 알아보기

### 개요

> JPA 2 introduces a criteria API that you can use to build queries programmatically. By writing a `criteria`, you define the where clause of a query for a domain class.
>

[공식 문서](https://docs.spring.io/spring-data/jpa/reference/jpa/specifications.html)에 따르면, Specification은 프로그래밍으로 쿼리를 작성하는 API 이고, 쿼리의 `Where` 절을 정의할 수 있다고 합니다. 사용 방법은 기존의 Repository 인터페이스에 `JpaSpecificationExecutor` 의 상속을 추가하면 됩니다.

따라서 이번에 사용할 Reservation에 대한 Repository는 다음과 같이 구현할 수 있습니다.

```java
public interface ReservationRepository extends JpaRepository<Reservation, Long>, JpaSpecificationExecutor<Reservation> {
}
```

JpaSpecificationExecutor 인터페이스의 코드는 다음과 같습니다.

```java
public interface JpaSpecificationExecutor<T> {
    Optional<T> findOne(Specification<T> spec);

    List<T> findAll(Specification<T> spec);

    Page<T> findAll(Specification<T> spec, Pageable pageable);

    List<T> findAll(Specification<T> spec, Sort sort);

    long count(Specification<T> spec);

    boolean exists(Specification<T> spec);

    long delete(Specification<T> spec);

    <S extends T, R> R findBy(Specification<T> spec, Function<FluentQuery.FetchableFluentQuery<S>, R> queryFunction);
}
```

대략적으로 보면 `검색 /  갯수 확인 /  존재 여부 / 삭제` 에 `Specification` 객체를 넣어서 사용할 수 있다는 것을 확인할 수 있습니다. 따라서, 기존의 JpaRepository가 지원하는 기능에 더불어 `Specification` 을 이용한 기능까지 사용할 수 있겠네요!

### 소스 코드 확인

우선, Specification의 소스코드를 먼저 확인해 보겠습니다.

```java
public interface Specification<T> extends Serializable {

    static <T> Specification<T> not(@Nullable Specification<T> spec) {
        return spec == null ? (root, query, builder) -> {
            return null;
        } : (root, query, builder) -> {
            return builder.not(spec.toPredicate(root, query, builder));
        };
    }

    static <T> Specification<T> where(@Nullable Specification<T> spec)

    default Specification<T> and(@Nullable Specification<T> other)
    
    default Specification<T> or(@Nullable Specification<T> other

    @Nullable
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder);

    static <T> Specification<T> allOf(Iterable<Specification<T>> specifications)

    @SafeVarargs
    static <T> Specification<T> allOf(Specification<T>... specifications)

    static <T> Specification<T> anyOf(Iterable<Specification<T>> specifications)
    
    @SafeVarargs
    static <T> Specification<T> anyOf(Specification<T>... specifications)
}
```

`not`을 제외한 나머지 세부 코드는 분량상 제외했고 어떤 메서드가 있는지만 작성하였습니다. 코드를 보면 다음을 알 수 있겠네요.

1. Predicate를 반환하는 **함수형 인터페이스**이고, Null 반환이 가능하다.
    - 여기서 Predicate는 우리가 일반적으로 알고 있는 자바 함수형 인터페이스가 아닌 `Expression<Boolean>` 을 상속받는  `jakarta.persistence.criteria` 패키지의 클래스입니다.
2. not, where, and, or 등 다른 static 및 default 메서드가 모두 `Specification` 을 반환하는 것을 보니 체이닝을 이용할 수 있다.
3. not의 코드를 보니, CriteriaBuilder의 not, and, or 등을 사용하는데 이 메서드에는 Predicate가 들어간다.

공식 문서에 있는 예시 코드를 확인해 보겠습니다.

```java
public class CustomerSpecs {
	
	..
	
  public static Specification<Customer> isLongTermCustomer() {
    return (root, query, builder) -> {
      LocalDate date = LocalDate.now().minusYears(2);
      return builder.lessThan(root.get(Customer_.createdAt), date);
    };
  }
  ..
}
```

예시 코드를 바탕으로 생각해보면, 다음과 같이 대략적으로 방향을 정할 수 있겠네요.

1. CriteriaBuilder의 and, or, where등 조건 메서드를 이용한다.
2. 이때, `root.get()` 을 통해 특정 컬럼의 값과, 비교할 값을 파라미터에 넣으면 된다.

마지막으로 실제로 테스트 해보기 전에 CriteriaBuilder와 Root의 코드를 확인해 보겠습니다.

```java
public interface CriteriaBuilder {    
		..
		Predicate and(Predicate... var1);
		Predicate or(Predicate... var1);
		..
    Predicate equal(Expression<?> var1, Object var2);
    ..
    <Y extends Comparable<? super Y>> Predicate lessThan(Expression<? extends Y> var1, Y var2);

		..
    <Y extends Comparable<? super Y>> Predicate between(Expression<? extends Y> var1, Y var2, Y var3);
}
```

마찬가지로, CriteriaBuilder의 소스 코드를 통해 다음과 같은 추측을 할 수 있겠네요.

1. equal, less, between 등 값을 비교하는 경우 어떤 타입에 대한 Expression 객체와 그 타입의 객체를 받는다.
    - 그러면, 위의 예시 코드에 있던 `root.get()` 이 Expression 및 이를 상속받는 타입을 반환한다고 생각할 수 있겠네요!
2. and, or 등을 이용해서 여러 조건식을 동시에 확인할 수 있다.
    - 1의 equal, less, between을 통해 얻은 여러 조건식을 and, or에 넣어서 한번에 정렬할 수 있겠네요.

```java
public interface Path<X> extends Expression<X> {
		..
    <Y> Path<Y> get(String var1);
}
```

`Root`의 부모 클래스 중 하나인 Path의 소스코드를 보면, 이전에 추측했던 `root.get()` 이 Expression 타입을 반환하는 것을 확인할 수 있고, “**get이 문자열을 받는데, 이 문자열은 컬럼 이름이고 get을 연속으로 사용하는 것을 보니 연결된 다른 테이블에 있는 값 역시 확인할 수 있다”**를 추측할 수 있겠네요.

다음 문단에서는 위에서 추측한 내용들을 바탕으로, 간단한 구현을 먼저 해보겠습니다.

## 간단하게 구현해보기

우선, 구현을 시작하기 전에 공통적으로 쓰이는 테스트용 데이터와 Entity 클래스 코드를 먼저 작성하겠습니다.

### 테스트용 데이터

```sql
-- 예약 검색 테스트용 데이터
insert into reservation_time(start_at) values('15:00');

insert into theme(name) values('1번 테마');
insert into theme(name) values('2번 테마');

insert into member(name) values('1번 회원');
insert into member(name) values ('2번 회원');

--- 예약 정보 : 모든 시간은 동일(15시)
-- 1번 회원은 오늘 1번 테마, 1일 전 2번 테마, 2일 전 1번 테마, 3일 전 2번 테마 예약
insert into reservation(date, member_id, theme_id, reservation_time_id) values(DATEADD('DAY', 0, CURRENT_DATE()), 1, 1, 1);
insert into reservation(date, member_id, theme_id, reservation_time_id) values(DATEADD('DAY', -1, CURRENT_DATE()), 1, 2, 1);
insert into reservation(date, member_id, theme_id, reservation_time_id) values(DATEADD('DAY', -2, CURRENT_DATE()), 1, 1, 1);
insert into reservation(date, member_id, theme_id, reservation_time_id) values(DATEADD('DAY', -3, CURRENT_DATE()), 1, 2, 1);

-- 2번 회원은 4일 전 1번 테마, 5일 전 2번 테마, 6일 전 1번 테마, 7일 전 1번 테마 예약
insert into reservation(date, member_id, theme_id, reservation_time_id) values(DATEADD('DAY', -4, CURRENT_DATE()), 2, 1, 1);
insert into reservation(date, member_id, theme_id, reservation_time_id) values(DATEADD('DAY', -5, CURRENT_DATE()), 2, 2, 1);
insert into reservation(date, member_id, theme_id, reservation_time_id) values(DATEADD('DAY', -6, CURRENT_DATE()), 2, 1, 1);
insert into reservation(date, member_id, theme_id, reservation_time_id) values(DATEADD('DAY', -7, CURRENT_DATE()), 2, 1, 1);
```

테스트를 위한 모든 데이터를 일일이 넣기 힘들기 때문에, `sql` 파일을 불러와서 사용하도록 구성하였습니다. 이후 과정에서 `테스트용 데이터` 라고 언급하는 부분은 모두 위의 데이터를 사용합니다.

### Entity

예약에 대한 Entity 클래스 코드는 다음과 같습니다.

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class Reservation {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private LocalDate date;
    @ManyToOne
    private Member member;
    @ManyToOne
    private Theme theme;
    @ManyToOne
    private ReservationTime reservationTime;
		
		..
}
```

Member, Theme, ReservationTIme는 다음과 같이 구성되어 있습니다.

- Member: `ID` 및 `회원 이름`에 해당되는 `String name` 을 필드로 가집니다.
- Theme: ID 및 `테마 이름`에 해당되는 `String name` 을 필드로 가집니다.
- ReservationTime: ID 및 `시작 시간`에 해당되는 `LocalTime startAt` 을 필드로 가집니다.

### 단일 조건 테스트

우선, 공식 문서에 있는 코드와 비슷하게, **같은 테마 이름으로 조회**하는 Specification을 만들어 보겠습니다.

```java
public class ReservationFilterSpecs {

	public static Specification<Reservation> hasThemeName(String themeName) {
		return (root, query, criteriaBuilder) -> criteriaBuilder.equal(
						root.get("theme").get("name"), themeName
	  );
	}
}
```

`root.get(”theme”).get(”name”)` 은, 이전에 추측한 대로라면 `Reservation.getTheme().getName()` 과 동일한 값을 불러올 것이라 기대할 수 있겠네요. 테스트를 통해 정확히 조회되는지 확인해 보겠습니다.

데이터는 `기본 코드` 문단에 정의한 테스트용 데이터를 사용할 것이고, 1번 테마는 5개의 예약이, 2번 테마는 3개의 예약이 나오는지 확인해 보겠습니다.

```java
@ParameterizedTest(name = "테마 이름이 {0}인 예약은 {1}개가 조회된다.")
@CsvSource(value = {"1번 테마/5", "2번 테마/3"}, delimiter = '/')
@Sql({"/truncate.sql", "/spec_test_data.sql"})
void hasThemeName(String themeName, int expectedCount) {
    // given
    Specification<Reservation> spec = ReservationFilterSpecs.hasThemeName(themeName);

    // when
    List<Reservation> found = reservationRepository.findAll(spec);

    // then
    assertThat(found.size()).isEqualTo(expectedCount);
}
```

위에 정의한 테스트용 데이터 파일 이름을 `spec_test_data.sql` , 그리고 모든 테이블을 초기화하는 `truncate.sql` 을 이용합니다. 테스트는 정상적으로 통과되며, 쿼리가 어떻게 호출되는지 확인해 보겠습니다.

```sql
Hibernate: 
    select
        r1_0.id,
        r1_0.date,
        r1_0.member_id,
        r1_0.reservation_time_id,
        r1_0.theme_id 
    from
        reservation r1_0 
    join
        theme t1_0 
            on t1_0.id=r1_0.theme_id 
    where
        t1_0.name=?
```

쿼리를 출력해 보니 위와 같이 테마 이름에 대한 where절이 추가되었네요!

### 여러 조건 테스트

이번에는 여러개의 조건식을 넣어 확인해 보겠습니다. CriteriaBuilder의 and는 간단한 것 같아, or을 이용하여 테스트 해보겠습니다. 입력된 테마 이름과 회원 이름 중, 적어도 한 개의 값과 같은 값을 가지는 예약을 조회하는 Specification을 만들겠습니다.

```java
public class ReservationFilterSpecs {

		..
    public static Specification<Reservation> hasThemeNameOrMemberName(String themeName, String memberName) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.or(
                criteriaBuilder.equal(root.get("theme").get("name"), themeName),
                criteriaBuilder.equal(root.get("member").get("name"), memberName)
        );
    }
}
```

테스트용 데이터를 이용하여 마찬가지로 테스트 해볼 것이고,

1. 1번 테마와 1번 회원 모두에 해당되는 예약은 2개
2. 1번 테마이지만, 1번 회원은 아닌 예약은 3개
3. 1번 회원이지만, 1번 테마는 아닌 예약은 2개

따라서 총 7개의 예약이 조회되는 것을 기대할 수 있고 이를 테스트 해보겠습니다.

```java
@Test
@DisplayName("테마가 1번 테마이거나, 회원이 1번 회원인 예약은 7개 존재한다.")
@Sql({"/truncate.sql", "/spec_test_data.sql"})
void hasThemeNameOrMemberName() {
    // given
    Specification<Reservation> spec = ReservationFilterSpecs.hasThemeNameOrMemberName("1번 테마", "1번 회원");

    // when
    List<Reservation> found = reservationRepository.findAll(spec);

    // then
    assertThat(found.size()).isEqualTo(7);
}
```

테스트가 잘 통과하고, 쿼리는 다음과 같이 where절에 테마 이름과 회원 이름이 추가되었네요!

```sql
Hibernate: 
    select
        r1_0.id,
        r1_0.date,
        r1_0.member_id,
        r1_0.reservation_time_id,
        r1_0.theme_id 
    from
        reservation r1_0 
    join
        member m1_0 
            on m1_0.id=r1_0.member_id 
    join
        theme t1_0 
            on t1_0.id=r1_0.theme_id 
    where
        t1_0.name=? 
        and m1_0.name=?
```

### 값이 입력되지 않는다면?

우선, 이전에 **단일 조건 테스트** 문단에서 구현했던 같은 테마 이름을 찾는 Specification을 활용할건데, 여기에 null을 입력해 보겠습니다. 실제 테스트 값이 어떻게 나올지 지금은 모르기 때문에, 일단 출력만 해보겠습니다.

```java
@Test
@DisplayName("테마 이름에 null을 입력")
@Sql({"/truncate.sql", "/spec_test_data.sql"})
void nullThemeName() {
    Specification<Reservation> spec = ReservationFilterSpecs.hasThemeName(null);
    List<Reservation> found = reservationRepository.findAll(spec);
    System.out.println("specification = " + spec);
    System.out.println("found = " + found);
    System.out.println("count = " + found.size());
}
```

에러가 발생할 수도 있겠다고 생각했지만, 에러가 발생하진 않았고 출력된 값은 다음과 같습니다.

```java
specification = com.blog.test.specification.repository.ReservationFilterSpecs$$Lambda$1865/0x000000f801accae0@77f2807f
found = []
count = 0
```

Specification 객체가 null이 아니네요. 실제로 null인 값이랑 비교를 하는 것 같고 그래서 해당되는 예약이 없어 결과가 빈 리스트가 나오는 것 같습니다.

그러면 이번에는 Specification 객체를 null로 지정하여 다시 한번 조회해 보겠습니다. 이때, findAll()에 null을 넣는 것이 아닌 Specification 객체를 null로 만든 다음에 입력해야 합니다.

> 위의 Specification 소스코드를 보면, Specification은 `Nullable` 임을 알 수 있습니다. 코드에 Nullable이 명시된 것이 힌트가 될 수 있겠네요!
>

```java
@Test
@DisplayName("테마 이름에 null을 입력")
@Sql({"/truncate.sql", "/spec_test_data.sql"})
void nullThemeName() {
    Specification<Reservation> spec = null;
    List<Reservation> found = reservationRepository.findAll(spec);
    System.out.println("specification = " + spec);
    System.out.println("found = " + found);
    System.out.println("count = " + found.size());
}
```

예외가 발생할 줄 알았는데, 다음의 결과를 보면 모든 데이터가 조회되네요.

```java
specification = null
found = [너무 길어서 생략...]
count = 8
```

왜 그런지 확인해 보기 위해,  `List<T> findAll(Specification<T> spec);` 을 계속 타고 올라가보면 다음과 같은 코드를 확인할 수 있습니다.

```java
private <S, U extends T> Root<U> applySpecificationToCriteria(@Nullable Specification<U> spec, Class<U> domainClass, CriteriaQuery<S> query) {
    Assert.notNull(domainClass, "Domain class must not be null");
    Assert.notNull(query, "CriteriaQuery must not be null");
    Root<U> root = query.from(domainClass);
    if (spec == null) {
        return root;
    } else {
        ..
    }
}
```

`query.from(domainClass)` 가 무엇인지 까지 파악할 순 없지만, 코드의 네이밍만 보고 유추할 수 있는 것은 입력된 Specification이 null이면 모든 데이터를 그대로 반환하는 것 같습니다.

```sql
Hibernate: 
    select
        r1_0.id,
        r1_0.date,
        r1_0.member_id,
        r1_0.reservation_time_id,
        r1_0.theme_id 
    from
        reservation r1_0
```

실제로 쿼리를 확인해보면, 어떠한 where절도 없이 전체를 조회하고 있네요!

### 체이닝

이전에 Specification 소스코드를 볼 때 체이닝을 사용할 수 있다는 것을 알 수 있었는데요, 앞의 **여러 조건 테스트** 문단에서는 CriteriaBuilder의 or을 통해 여러 조건식을 한번에 넣었었고, 이번에는 Specification의 체이닝을 활용해서 같은 조건식을 만들어 확인해 보겠습니다.

전에는 회원 이름, 테마 이름을 둘다 입력받아 조회를 했었는데, 우선 이것을 각각을 입력받도록 분리하겠습니다.

```java
public class ReservationFilterSpecs {

	public static Specification<Reservation> hasThemeName(String themeName) {
		return (root, query, criteriaBuilder) -> criteriaBuilder.equal(
						root.get("theme").get("name"), themeName
	  );
	}
	
	public static Specification<Reservation> hasMemberName(String memberName) {
		return (root, query, criteriaBuilder) -> criteriaBuilder.equal(
						root.get("member").get("name"), memberName
	  );
}
```

Specification의 or을 이용하면, 여러 개의 Specification을 연결할 수 있습니다.

```java
@Test
@DisplayName("테마가 1번 테마이거나 회원이 1번 회원인 예약은 7개 존재한다.")
@Sql({"/truncate.sql", "/spec_test_data.sql"})
void hasMemberNameOrThemeName() {
    // given
    Specification<Reservation> spec = ReservationFilterSpecs.hasThemeName("1번 테마")
            .or(ReservationFilterSpecs.hasThemeName("1번 회원"));

    // when
    List<Reservation> found = reservationRepository.findAll(spec);

    // then
    assertThat(found.size()).isEqualTo(7);
}
```

테스트는 정상적으로 통과하고, 쿼리도 전과 동일하게 출력됨을 확인할 수 있습니다.

```sql
Hibernate: 
    select
        r1_0.id,
        r1_0.date,
        r1_0.member_id,
        r1_0.reservation_time_id,
        r1_0.theme_id 
    from
        reservation r1_0 
    join
        member m1_0 
            on m1_0.id=r1_0.member_id 
    join
        theme t1_0 
            on t1_0.id=r1_0.theme_id 
    where
        t1_0.name=? 
        or m1_0.name=?
```

## 다음 편에서 계속됩니다..

### 요약

1. Specification을 통해 **쿼리가 아닌 자바 코드로 where절을 추가**할 수 있다.
2. Specification은 **Nullable**인 함수형 인터페이스이고, **null인 경우 모든 조건을 조회**한다.
3. 값을 비교하는 조건식은 CriteriaBuilder의 equal, between 등을 이용하여 만들 수 있다.
4. 값을 꺼낼 때는 `root.get(”컬럼명”)` 을 이용하고, get을 여러번 사용하여 연관된 다른 테이블의 값을 꺼낼 수 있다.
5. Specification의 and, or 등을 이용한 **체이닝이 가능**하다.

사실 구현하고 사용할 땐 정말 간단하게 했었는데, 이게 글로 작성하다보니 분량이 참 길어지네요. 다음 편에서는 지금까지 다룬 내용을 바탕으로 실제 문제를 해결해 보겠습니다.

읽어주셔서 감사합니다. 즐거운 하루 보내세요🙇

```toc
```