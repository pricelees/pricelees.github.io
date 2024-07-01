---
emoji: '🌱'
title: 날짜와 시간의 직렬화 & 역직렬화 시도 - JsonFormat, ObjectMapper
date: '2024-06-26 19:00:00'
author: 이상진
tags: 안녕하세요!
categories: level2 Spring JsonFormat ObjectMapper
---

## 배경

[우아한테크코스의 세 번째 스프링 미션](https://github.com/woowacourse/spring-roomescape-waiting/pull/30)을 진행하며 코드 리뷰를 받던 도중 리뷰어께서 좋은 의견을 공유해 주셨습니다. 의견은 기존의 `@JsonFormat`을 중복해서 사용하는 코드를 ObjectMapper를 만들어 해결하는 것이었는데요, 여기에 더해서, 미션을 하며 그냥 대략적으로만 알고 사용했던 @JsonFormat에 대해서도 알아보는 과정을 기록하고자 합니다.

## @JsonFormat 이란?

[공식 문서](https://www.javadoc.io/doc/com.fasterxml.jackson.core/jackson-annotations/2.9.8/com/fasterxml/jackson/annotation/JsonFormat.html)에 따르면, JsonFormat을 다음과 같이 설명합니다.

> General-purpose annotation used for configuring details of how values of properties are to be serialized. Unlike most other Jackson annotations, annotation does not have specific universal interpretation: instead, effect depends on datatype of property being annotated (or more specifically, deserializer and serializer being used)
>

JsonFormat이라는 이름과 맨 앞부분의 설명만 보면, `직렬화 할 때 형식을 지정하는데 사용하는 구나~` 정도는 알 수 있겠네요. 미션을 진행하는 도중에는 그냥 이정도 까지만 알고 사용했는데, 그러다 보니 사용할 때 마다 찝찝함이 남더라구요. 다음 문단부터는 JsonFormat이 어떻게 적용되는지 확인해 보겠습니다.

## 사용할 코드

**Entity & Repository**

DB는 Spring Data JPA와 h2를 사용했습니다. 이번 테스트에 사용할 Entity와 Repository 코드는 다음과 같습니다.

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private LocalDate manufactureDate;
    private LocalTime manufactureTime;

    public Product(String name, LocalDate manufactureDate, LocalTime manufactureTime) {
        this.name = name;
        this.manufactureDate = manufactureDate;
        this.manufactureTime = manufactureTime;
    }
}
```

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
}
```

**요청 DTO**

```java
public record ProductSaveRequest(
        String name,
        LocalDate manufactureDate,
        LocalTime manufactureTime
) {
    public Product toEntity() {
        return new Product(name, manufactureDate, manufactureTime);
    }
}
```

데이터 저장 요청시 사용하는 DTO 객체입니다.

**응답 DTO**

```java
public record ProductResponse(
        Long id,
        LocalDate manufactureDate,
        LocalTime manufactureTime
) {
    public static ProductResponse from(Product product) {
        return new ProductResponse(product.getId(), product.getManufactureDate(), product.getManufactureTime());
    }
}
```

데이터 저장 / 조회 등 상품 데이터 반환 시 사용하는 DTO 객체입니다.

**컨트롤러**

```java
@RestController
@RequestMapping("/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductRepository productRepository; 

    @GetMapping("/{id}")
    public ProductResponse findById(@PathVariable Long id) {
        Product product = productRepository.findById(id).orElseThrow();
        return ProductResponse.from(product);
    }

    @PostMapping
    public ProductResponse save(@RequestBody ProductSaveRequest productSaveRequest) {
        Product product = productSaveRequest.toEntity();
        return ProductResponse.from(productRepository.save(product));
    }

    @ExceptionHandler
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public void handle(Exception e) {
        e.printStackTrace();
    }
}
```

간단한 `저장 / 조회 기능`과 Stacktrace 출력을 위한 ExceptionHandler로 구성했습니다.

다음 문단부터는 이 코드들을 이용하여 `@JsonFormat`에 대해 확인해 보겠습니다.

## JsonFormat - 역직렬화

우선 요청 데이터를 객체로 역직렬화 하는 방법부터 확인해 보겠습니다. 형식은 가장 일반적으로 쓰이는 yyyy-MM-dd와 HH:mm:ss를 이용하겠습니다.

```java
public record ProductSaveRequest(
        String name,
        @JsonFormat(pattern = "yyyy-MM-dd")
        LocalDate manufactureDate,
        @JsonFormat(pattern = "HH:mm:ss")
        LocalTime manufactureTime
) {
    ..
}
```

### 정상 테스트

위에서 지정한 JsonFormat에 맞춰, 정상적인 요청을 보내는 테스트를 진행해 보겠습니다. 응답 상태코드는 컨트롤러에서 별도로 지정하지 않았기에 200으로 확인합니다.

```java
@Test
void 정확한_형식으로_요청() {
    Map<String, Object> params = Map.of(
            "name", "상품1",
            "manufactureDate", "2024-06-25",
            "manufactureTime", "12:34:56"
    );

    RestAssured.given().log().all()
            .contentType(MediaType.APPLICATION_JSON_VALUE)
            .body(params)
            .when().post("/products")
            .then().log().all()
            .statusCode(200);
}
```

### 날짜만 다른 형식으로 테스트

정상적인 요청에 대한 테스트는 성공합니다. 그러면 이번에는 날짜만 JsonFormat으로 지정한 값과 다른 형식으로 테스트를 해보겠습니다.

```java
@Test
void 날짜만_다른_형식으로_요청() {
    Map<String, Object> params = Map.of(
            "name", "상품1",
            "manufactureDate", "2024/06/25",
            "manufactureTime", "12:34:56"
    );
    
    // RestAssured는 전과 동일
	  ..
}
```

위의 테스트는 실패하였습니다. 이전에 ExceptionHandler에서 Stacktrace를 출력하도록 해놨으니, 한번 확인해 보겠습니다.

```java
org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Cannot deserialize value of type `java.time.LocalDate` from String "2024/06/25": Failed to deserialize java.time.LocalDate: (java.time.format.DateTimeParseException) Text '2024/06/25' could not be parsed at index 4
..
Caused by: java.time.format.DateTimeParseException: Text '2024/06/25' could not be parsed at index 4
	at java.base/java.time.format.DateTimeFormatter.parseResolved0(DateTimeFormatter.java:2052)
	at java.base/java.time.format.DateTimeFormatter.parse(DateTimeFormatter.java:1954)
	at java.base/java.time.LocalDate.parse(LocalDate.java:430)
	at com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer._fromString(LocalDateDeserializer.java:176)
	... 63 more
..
```

이전에 예외 처리에서 알아본 대로, HttpMessageNotReadable 에러가 발생하고 그 원인으로 DateTimeParseException이 발생하네요. 더 자세히 보면 `LocalDateDeserializer`의 `_fromString` , 즉 문자열을 LocalDate 형식으로 변환하는 과정에서 발생하는 것을 알 수 있었습니다.

### 시간만 다른 형식으로 테스트

전에 했던 날짜와 동일하게, 날짜는 yyyy-MM-dd의 형식으로, 시간은 JsonFormat에서 지정한 형식과 다르게 지정하여 테스트를 진행하였습니다.

```java
Caused by: java.time.format.DateTimeParseException: Text '12/34/56' could not be parsed at index 2
	at java.base/java.time.format.DateTimeFormatter.parseResolved0(DateTimeFormatter.java:2052)
	at java.base/java.time.format.DateTimeFormatter.parse(DateTimeFormatter.java:1954)
	at java.base/java.time.LocalTime.parse(LocalTime.java:465)
	at com.fasterxml.jackson.datatype.jsr310.deser.LocalTimeDeserializer._fromString(LocalTimeDeserializer.java:193)
	... 63 more
```

예외는 동일하게 발생했는데, 이번에는 LocalTimeDeserializer에서 발생했습니다.

### 결론

역직렬화시 `@JsonFormat` 으로 지정한 형식과 다르면 날짜 / 시간을 변환하는 과정에서 예외가 발생합니다.

## @JsonFormat을 안 붙인다면?

```java
public class LocalDateDeserializer extends JSR310DateTimeDeserializerBase<LocalDate> {
	..
	private static final DateTimeFormatter DEFAULT_FORMATTER;
	
  ..
  
  static {
    DEFAULT_FORMATTER = DateTimeFormatter.ISO_LOCAL_DATE;
		..
  }
}
```

```java
public class LocalTimeDeserializer extends JSR310DateTimeDeserializerBase<LocalTime> {
	..
	private static final DateTimeFormatter DEFAULT_FORMATTER;
	
	..
	
  static {
    DEFAULT_FORMATTER = DateTimeFormatter.ISO_LOCAL_TIME;
		..
  }
}
```

이전에 예외를 확인하는 과정에서 LocalDateDeserializer와 LocalTimeDeserializer 소스코드를 확인해 봤는데, 소스코드를 보면 DEFAULT_FORMATTER라는 값이 있습니다.

- `LocalDate`의 경우 `ISO_LOCAL_DATE`이고, 이는 `“yyyy-MM-dd”` 형식을 나타냅니다.
- `LocalTime`의 경우 `ISO_LOCAL_TIME`이고, 이는 `“HH:mm:ss”` 또는 `“HH:mm”` 형식입니다.

```java
public abstract class JSR310DateTimeDeserializerBase<T> extends JSR310DeserializerBase<T> implements ContextualDeserializer {
    protected final DateTimeFormatter _formatter;
    ..
}
```

추가적으로, LocalDateDeserializer, LocalTimeDeserializer의  부모 클래스인 JSR310..을 보면, **_formatter** 필드가 있네요.

```java
public class LocalDateDeserializer extends JSR310DateTimeDeserializerBase<LocalDate> {
		..
    private static final DateTimeFormatter DEFAULT_FORMATTER;
		..

    protected LocalDateDeserializer() {
        this(DEFAULT_FORMATTER);
    }

    public LocalDateDeserializer(DateTimeFormatter dtf) {
        super(LocalDate.class, dtf);
    }
    ..
}
```

그러면, LocalDateDeserializer의 생성자를 보니 다음을 유추할 수 있을 것 같습니다. (**LocalTime도 동일합니다.**)

1. @JsonFormat을 지정하지 않으면 **DEFAULT_FORMATTER** 값을 사용한다.
2. @JsonFormat을 지정하면 그 값을 사용한다.

### 테스트

그러면 DTO에서 JsonFormat을 지정하지 않아도, 날짜는 “yyyy-MM-dd”, 시간은 “HH:mm:ss” 또는 “HH:mm” 형식을 넣으면 정상적으로 작동할 것을 기대할 수 있겠네요.

```java
public record ProductSaveRequest(
        String name,
        LocalDate manufactureDate,
        LocalTime manufactureTime
) {
    ..
}
```

위 코드와 같이 @JsonFormat을 없애고, 이전에 했던 테스트를 다시 실행해 보겠습니다.

```java
@Test
void 정확한_형식으로_요청() {
    Map<String, Object> params = Map.of(
            "name", "상품1",
            "manufactureDate", "2024-06-25",
            "manufactureTime", "12:34:56"
    );

    RestAssured.given().log().all()
            .contentType(MediaType.APPLICATION_JSON_VALUE)
            .body(params)
            .when().post("/products")
            .then().log().all()
            .statusCode(200);
}
```

테스트는 정상적으로 통과하고, 여기서 시간은 “12:34”로 입력해도 마찬가지로 잘 통과하는 것을 확인할 수 있습니다.

### 결론

@JsonFormat을 붙이지 않아도, **날짜의 경우 “yyyy-MM-dd”, 시간의 경우 “HH:mm:ss” 또는 “HH:mm” 형식**의 값을 입력하면 정상적으로 변환된다는 것입니다.

더 추가적으로 확인해볼 수 있는 것은 `@JsonFormat`을 붙이지 않고 `"2011-12-03T10:15:30”`  형식으로 JSON 요청을 보내도 정상적으로 값이 파싱되는 것이었는데요, 이 부분은 지금 단계에서는 크게 의미가 없는 것 같아 작성하지 않았습니다.(_fromString() 소스코드에서 확인할 수 있습니다)

## JsonFormat - 직렬화

그러면 `@ResponseBody` 혹은 `@RestController` 를 통해 응답 JSON을 보낼 때의 경우도 확인해 보겠습니다.

### JsonFormat을 지정하지 않을 때

이전 문단에서 `@JsonFormat` 을 지정하지 않았을 때의 기본값이 있던 것 처럼, 직렬화를 할 때도 기본값이 있을 것이라고 생각할 수 있겠네요. 그래서 우선 @JsonFormat을 지정하지 않고 테스트를 해보겠습니다.

```java
public record ProductResponse(
        Long id,
        LocalDate manufactureDate,
        LocalTime manufactureTime
) {
    ..
}
```

조회 테스트를 하기 전에 저장을 해야 하는데, 이렇게 되면 코드 자체가 길어져서 `data.sql` 을 이용하여 하나의 초기 데이터를 지정하였습니다.

```sql
INSERT INTO product(name, manufacture_date, manufacture_time) VALUES ('name1', '2024-06-25', '10:00');
```

테스트 코드는 위와 같이 RestAssured를 이용해 작성하였습니다.

```java
@Test
void JsonFormat_지정_없이_응답() {
    RestAssured.given().log().all()
            .when().get("/products/1")
            .then().log().all()
            .statusCode(200);
}
```

테스트는 정상적으로 완료되었고, 응답 로그를 통해 응답 JSON을 확인해 보겠습니다.

```json
{
    "id": 1,
    "manufactureDate": "2024-06-25",
    "manufactureTime": "10:00:00"
}
```

위와 같이 이전에 역직렬화를 할 때의 기본 형식과 같은 형식으로 값이 반환되었습니다. 데이터를 넣을 때 시간은 “10:00”으로 입력했는데, 값을 조회할 때는 기본 형식인 “10:00:00”으로 반환이 되네요.

### JsonFormat 지정

아래와 같이 JsonFormat을 지정하면, 응답 JSON의 값을 해당 형식으로 표현할 것이라고 생각할 수 있겠네요.

```java
public record ProductResponse(
        Long id,
        @JsonFormat(pattern = "yyyy / MM / dd")
        LocalDate manufactureDate,
        @JsonFormat(pattern = "HH / mm")
        LocalTime manufactureTime
) {
    ..
}
```

이전에 JsonFormat을 지정하지 않았을 때와 같은 테스트를 돌려보면, 아래와 같이 원하는 형식으로 날짜가 출력됨을 확인할 수 있었습니다😄

```java
{
    "id": 1,
    "manufactureDate": "2024 / 06 / 25",
    "manufactureTime": "10 / 00"
}
```

## ObjectMapper Bean 등록

### 배경

현재 프로그램 요구사항에선 요청 / 응답 모두 날짜는 “yyyy-MM-dd”, 시간은 “HH:mm” 형식을 사용하고 있습니다. 코드를 처음 제출했을 때는 모든 LocalDate와 LocalTime에 JsonFormat을 지정했는데요, 지금까지 알아본 내용 대로라면 다음과 같이 지정해도 이전과 같이 동작할 것을 기대할 수 있겠네요.

1. `LocalDate`는 JsonFormat을 지정하지 않아도 된다.
2. `LocalTime`은 요청의 경우 지정하지 않아도 되고, **응답의 경우만 JsonFormat을 “HH:mm”으로 지정**한다.

하지만 이 방법이 좋다고 생각하는가? 하면 아닌 것 같습니다. 코드량은 줄었지만 **JsonFormat을 지정하지 않았을 때의 기본값을 모를 수도 있기 때문**에 이전과 같이 **하나하나 다 지정**하는게 가장 좋은 방법이라고 생각하는데요, 코드의 중복을 없애면서 형식을 통일하는 방법이 있으면 참 좋을 것 같습니다.

### ObjectMapper 등록

리뷰어께서 주신 의견을 바탕으로 구글링을 통해 [Baeldung의 글](https://www.baeldung.com/spring-boot-customize-jackson-objectmapper)을 찾았고, 이 글의 내용을 바탕으로 모든 JsonFormat을 통일해 보겠습니다. 코드는 일부 수정하였습니다.

> @ConfigurationProperties 또는 @Value를 사용하면 yaml을 통해 더 쉽게 관리할 수 있지만, 이번에는 사용하지 않았습니다.
>

1. `JavaTimeModule` 빈 등록

```java
@Configuration
public class JacksonConfig {
	
	..
	
  @Bean
  public JavaTimeModule javaTimeModule() {
      return (JavaTimeModule) new JavaTimeModule()
              .addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ISO_LOCAL_DATE))
              .addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ISO_LOCAL_DATE))
              .addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern("HH:mm")))
              .addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern("HH:mm")));
  }
}
```

우선, 날짜와 시간은 `JavaTimeModule` 에 serializer와 deserializer를 추가하는 방식으로 형식을 지정할 수 있습니다. 지금은 LocalDate와 LocalTime만 적용했지만, LocalDateTime 등의 타입에서도 지정할 수 있습니다.

```java
@Bean
public JavaTimeModule javaTimeModule() {
    JavaTimeModule javaTimeModule =  new JavaTimeModule();
    
    javaTimeModule.addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ISO_LOCAL_DATE));
    javaTimeModule.addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ISO_LOCAL_DATE));
    javaTimeModule.addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern("HH:mm")));
    javaTimeModule.addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern("HH:mm")));
    
    return javaTimeModule;
}
```

캐스팅이 불편하다면 위와 같이 등록해도 됩니다!

2. `ObjectMapper` 빈 등록

```java
@Configuration
public class JacksonConfig {
	
	@Bean
	public ObjectMapper objectMapper() {
		return new ObjectMapper().registerModule(javaTimeModule());
	}
	
  @Bean
  public JavaTimeModule javaTimeModule() {
      // 이전 코드와 동일
  }
}
```

ObjectMapper의 `registerModule` 에 이전에 구현한 `JavaTimeModule`을 입력하면 됩니다.

### 테스트

위와 같이 ObjectMapper를 등록했다면, 기존에 LocalDate및 LocalTime에 붙은 모든 JsonFormat을 지울 수 있습니다. 모든 JsonFormat을 지운 뒤, 테스트를 통해 정상적으로 변환되는지 확인해 보겠습니다.

1. 날짜에 대한 역직렬화 테스트

```java
@ParameterizedTest(name = "{0} 형식의 날짜에 대한 역직렬화 요청시 {1} 코드를 반환한다.")
@CsvSource(value = {"2024-06-25 , 200", "2024/06/25 , 400"}, delimiter = ',')
void date_deserialize(String date, int expectedStatusCode) {
    Map<String, Object> body = Map.of(
            "name", "name1",
            "manufactureDate", date,
            "manufactureTime", "12:30"
    );

    RestAssured.given().log().all()
            .contentType(MediaType.APPLICATION_JSON_VALUE)
            .body(body)
            .when().post("/products")
            .then().statusCode(expectedStatusCode);
}
```

이전에 컨트롤러 코드에서, 예외 발생시 400 응답을 하도록 ExceptionHandler를 정의하였기에 올바르지 않은 형식의 날짜로 요청을 하면 400 응답이 발생해야 합니다.

2. 시간에 대한 역직렬화 테스트

```java
@ParameterizedTest(name = "{0} 형식의 시간에 대한 역직렬화 요청시 {1} 코드를 반환한다.")
@CsvSource(value = {"12:30 , 200", "12:30:00 , 400", "12/30, 400"}, delimiter = ',')
void time_deserialize(String time, int expectedStatusCode) {
    Map<String, Object> body = Map.of(
            "name", "name1",
            "manufactureDate", "2024-06-25",
            "manufactureTime", time
    );

    RestAssured.given().log().all()
            .contentType(MediaType.APPLICATION_JSON_VALUE)
            .body(body)
            .when().post("/products")
            .then().statusCode(expectedStatusCode);
}
```

날짜의 경우와 거의 동일하고, @JsonFormat을 지정하지 않았다면 “12:30:00”도 정상적으로 변환이 되어야 하지만 형식을 “HH:mm”으로 지정하였기에 예외가 발생하는 것을 확인할 수 있습니다.

3. 직렬화 테스트

편의를 위해 data.sql에 다음과 같은 데이터를 추가하였습니다.

```sql
INSERT INTO product(name, manufacture_date, manufacture_time) VALUES ('name1', '2024-06-25', '10:00');
```

날짜의 경우 기본값과 같기 때문에 크게 의미가 없으나, 시간은 기본값이 `“HH:mm:ss”` 이기 때문에 `“HH:mm”` 으로 직렬화 되는지 확인하는 것이 중요합니다.

```java
@DisplayName("날짜는 yyyy-MM-dd 형식으로 직렬화된다.")
@Test
void date_serialize() {
    RestAssured.given().log().all()
            .when().get("/products/1")
            .then().log().all()
            .statusCode(200)
            .body("manufactureDate", is("2024-06-25"));
}

@DisplayName("시간은 HH:mm 형식으로 직렬화된다.")
@Test
void time_serialize() {
    RestAssured.given().log().all()
            .when().get("/products/1")
            .then().log().all()
            .statusCode(200)
            .body("manufactureTime", is("10:00"));
}
```

## 결론

### 요약

1. JsonFormat을 지정하지 않는 경우
    - LocalDate는 “yyyy-MM-dd” 형식이 기본값이다.
    - LocalTime은 역직렬화시 “HH:mm:ss” 또는 “HH:mm”, 직렬화시 “HH:mm:ss” 가 기본값이다.
2. 역직렬화시 JsonFormat을 지정하면, 요청 JSON의 값 형식은 지정한 형식과 동일해야 한다.
3. 직렬화시 JsonFormat을 지정하면, 응답 JSON의 값이 지정한 형식으로 표현된다.
4. Custom ObjectMapper를 통해 JsonFormat의 코드 중복을 없앨 수 있다.

분명 구현하고 테스트할때는 간단했는데, 이게 글로 작성하자니 참 오래 걸리고 길어졌네요. 그래도 드디어 항상 의문을 가지던 JsonFormat에 대해 이해한 것 같아 즐거운 과정이었다고 생각합니다. 잘못된 정보나 의견이 있으시면 편하게 부탁드립니다.

읽어주셔서 감사합니다. 즐거운 하루 보내세요🙇

```toc
```