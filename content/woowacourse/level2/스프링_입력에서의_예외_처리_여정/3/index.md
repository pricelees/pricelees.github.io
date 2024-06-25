---
emoji: '🌱'
title: 스프링 입력에서의 예외 처리 여정 3 - Custom Deserializer
date: '2024-06-20 19:00:00'
author: 이상진
tags: 안녕하세요!
categories: level2 Spring Exception
---

## 서론

드디어 마지막이네요. 처음 이 글을 작성하고자 했을 때는 3편까지 쓸 것이라는 생각을 전혀 안했는데..ㅎㅎ 최대한 간단하게 쓴다고 노력해도 역시 욕심은 끝이 없는 것 같습니다.

이번에는 지난번에 학습한 JsonNode를 활용해서 Custom Deserializer를 직접 만들어 볼텐데요, JsonNode가 헷갈리신다면 [이전 편](http://pricelees.github.io/woowacourse/level2/%EC%8A%A4%ED%94%84%EB%A7%81_%EC%9E%85%EB%A0%A5%EC%97%90%EC%84%9C%EC%9D%98_%EC%98%88%EC%99%B8_%EC%B2%98%EB%A6%AC_%EC%97%AC%EC%A0%95/2/)를 확인해주시면 감사하겠습니다.🙇

## 해결해야 할 것

이번 Custom Deserializer를 통해 해결해야 하는 문제는, 1편의 맨 마지막에 나온 두 가지 문제입니다. 본문을 시작하기 전에 간단하게 리마인드 하고 가겠습니다 ㅎㅎ

1. 지금의 예외 핸들링은, 테마, 시간, 멤버를 선택하지 않았을 때 기본 입력값인 `“테마 선택”` 등에 의존합니다. **즉 클라이언트 코드에 완전히 의존하는 구조입니다.**
2. 예약을 추가할 때 멤버, 시간, 테마 중 2개 이상의 값이 입력되지 않아도 하나의 값만 표시됩니다. 즉 **입력되지 않은 모든 값을 예외 메시지에 담을 수 없습니다.**
    - 추가적으로, 날짜를 입력하지 않았을 때 발생하는 MethodArgumentNotValidException와 같이 묶어서 처리할 수도 없습니다!

이 문제들은 `1번 문제, 2번 문제` 로 뒤에서 언급할 예정이니 참고 부탁드립니다😄

## Custom Deserializer - 정상 입력 처리

예외는 나중에 생각하고, 우선 **값이 정상적으로 입력되었다고 가정했을 때의 Custom Deserializer**를 만들어 보겠습니다.

### 배경: 요청 JSON 및 DTO

모든 값(회원, 테마, 날짜, 시간)을 선택했을 때의 JSON과 모든 값을 선택하지 않았을 때의 JSON, 그리고 요청 DTO를 다시 한번 작성해 보겠습니다.

```json
// 모든 값 선택
{
    "date": "2024-06-20",
    // 숫자 값은 예시입니다. 
    "memberId": "1", 
    "themeId": "2",
    "timeId": "3"
}

// 모든 값을 선택하지 않음.
{
    "date": "",
    "memberId": "멤버 선택",
    "themeId": "테마 선택",
    "timeId": "시간 선택"
}
```

지난 1편에서 파악한 문제는 **모든 값을 선택하지 않았을 때**, date의 경우 `@NotNull` 에 의해 `MethodArgumentException`이, 나머지는 InvalidFormat에서 비롯된 **HttpMessageNotReadableException**이 발생된다는 것이었습니다.

```java
public record AdminReservationCreateRequest(
        @NotNull(message = "날짜를 입력해 주세요.") LocalDate date,
        @NotNull(message = "시간을 입력해 주세요.") Long timeId,
        @NotNull(message = "테마를 입력해 주세요.") Long themeId,
        @NotNull(message = "회원을 입력해 주세요.") Long memberId
) {
}
```

요청 DTO는 위와 같습니다 ㅎㅎ

### 우선 모든 값 노드를 불러오겠습니다.

2편의 맨 앞에서 작성한 코드에,  값 노드들을 불러오는 코드를 추가해 보겠습니다 ㅎㅎ

```java
public class AdminReservationCreateRequestDeserializer extends StdDeserializer<AdminReservationCreateRequest> {

	// 생성자는 Baeldung에 있는 것과 동일하게 작성
	
	@Override
	public AdminReservationCreateRequest deserialize(JsonParser jsonParser, DeserializationContext deserializationContext)
            throws IOException, JacksonException {
            
     JsonNode rootNode = jsonParser.getCodec().readTree(jsonParser);
     
     JsonNode dateNode = rootNode.path("date");     
     JsonNode memberIdNode = rootNode.path("memberId");
     JsonNode themeIdNode = rootNode.path("themeId");
     JsonNode timeIdNode = rootNode.path("timeId");
		
		 ..
  }
}
```

2편의 내용에 따르면,

1. rootNode는 `ContainerNode(ObjectNode)` 이다.
2. 나머지 date, memberId 노드는 `ValueNode(TextNode)` 이다.

### 정상 입력에 대한 Custom Deserializer 마무리

```java
public class AdminReservationCreateRequestDeserializer extends StdDeserializer<AdminReservationCreateRequest> {

	// 생성자는 Baeldung에 있는 것과 동일하게 작성
	
	@Override
	public AdminReservationCreateRequest deserialize(JsonParser jsonParser, DeserializationContext deserializationContext)
            throws IOException, JacksonException {
            
     JsonNode rootNode = jsonParser.getCodec().readTree(jsonParser);
     
     JsonNode dateNode = rootNode.path("date");     
     JsonNode memberIdNode = rootNode.path("memberId");
     JsonNode themeIdNode = rootNode.path("themeId");
     JsonNode timeIdNode = rootNode.path("timeId");
		
		 LocalDate date = LocalDate.parse(dateNode.asText(), DateTimeFormatter.ISO_LOCAL_DATE);
		 Long memberId = memberIdNode.asLong();
		 Long themeId = themeIdNode.asLong();
		 Long timeId = timeIdNode.asLong();
		 
		 return new AdminReservationCreateRequest(date, timeId, themeId, memberId);
  }
}
```

값은 문자열, 숫자, 불리언 형태로만 불러올 수 있기에 날짜의 경우 문자열로 불러온 뒤 `LocalDate.parse` 를 이용했고, 나머지는 `asLong()`을 이용했습니다. 물론 처음부터 `rootNode.path(”date”).asText()` 와 같이 불러와도 됩니다.

> DateTimeFormatter.ISO_LOCAL_DATE는 “yyyy-MM-dd” 형태로 변환하는 상수입니다.
>

```java
@JsonDeserialize(using = AdminReservationCreateRequestDeserializer.class)
public record AdminReservationCreateRequest(
        LocalDate date,
        Long timeId,
        Long themeId,
        Long memberId
) {
}
```

최종적으로, 해당 DTO 클래스에 `@JsonDeserialize`  어노테이션과 Deserializer 타입을 넣어주면 됩니다!

### 값을 입력하지 않는다면?

눈썰미가 훌륭하신 분들은 이미 파악하셨겠지만, 바로 위에 있는 AdminReservationCreateRequest 클래스 코드에서 `@NotNull`이 사라졌습니다!

만약 위와 같이 구현한 Custom Deserializer를 사용할 때, 값을 입력하지 않으면 어떻게 될까요? 2편의 내용과 같이 보면 이해가 쉽습니다😄

**date**

- date의 경우 선택하지 않으면 빈 문자열(`“”`)로 요청되는데, 그러면 위의 Custom Deserializer에서 `rootNode.path(”date”).asText()` 를 했을 때의 값은 `“”`가 되어 `LocalDate.parse` 에서 `DateTimeParseException`이 발생하게 됩니다.
- 만약 요청 JSON에 “date” 필드 자체가 없다면, `path()`를 통해 값을 불러오므로, dateNode는 `MissingNode` 가 됩니다. 따라서 이 노드에 `asText()`를 호출한 값이 기본값인 빈 문자열(`“”` )이 되어 위와 동일한 `DateTimeParseException`이 발생합니다.

**timeId**

- `themeId, memberId`도 동일합니다. timeId를 기준으로 설명하겠습니다.
- 선택하지 않으면 `“시간 선택”` 으로 요청됩니다. 따라서 timeIdNode는 “시간 선택”이라는 값을 가지는 `ValueNode(TextNode)` 가 됩니다.
- 이 노드에 `asLong()`을 호출하면, 숫자로 변환이 안되는 경우 기본값인 0을 반환합니다.
- 필드 자체가 없는 경우 역시 위의 date와 동일한 과정으로 기본값인 0이 됩니다.
- ID(DB의 PK이자 AUTO_INCREMENT)는 모두 1 이상이므로, **이후의 과정에서 ID 0에 해당되는 시간을 찾을 수 없다는 예외가 발생**하게 됩니다.

따라서, `@NotNull` 어노테이션 자체가 작동되지 않습니다.

## 첫 번째 방법 - @Valid 어노테이션 활용

우선, Custom Deserializer에 의해 **1번 문제**는 해결이 되었습니다. (”테마 선택”과 같은 **클라이언트 코드 의존이 없습니다**) 그러면 2번 문제만 해결하면 1편에서의 문제점을 해소할 수 있겠네요 ㅎㅎ

첫 번째 방법은, **Deserializer는 그대로 두고**, `@Valid`를 활용하여 모든 예외를 `MethodArgumentNotValidException` 으로 처리하는 방법입니다.

- MethodArgumentNotValidException에 대한 처리는 1편에 있습니다.

위에 작성한 Deserializer 코드를 다시 작성하는데, 조금 더 간결하게 작성하겠습니다.

```java
@Override
public AdminReservationCreateRequest deserialize(JsonParser jsonParser, DeserializationContext deserializationContext)
          throws IOException, JacksonException {
          
   JsonNode rootNode = jsonParser.getCodec().readTree(jsonParser);
   
   String date = rootNode.path("date").asText();     
   Long memberId = rootNode.path("memberId").asLong();
   Long themeId = rootNode.path("themeId").asLong();
   Long timeId = rootNode.path("timeId").asLong();
	 
	 return new AdminReservationCreateRequest(date, timeId, themeId, memberId);
}
```

가장 중요한 변경 사항은, **date의 타입이 String으로 변경된 것입니다**. 마찬가지로 DTO의 필드 타입도 바꾸겠습니다.

```java
@JsonDeserialize(using = AdminReservationCreateRequestDeserializer.class)
public record AdminReservationCreateRequest(
    @Pattern(regexp = "\\d{4}-\\d{2}-\\d{2}", message = "날짜가 입력되지 않았거나, 입력 형식이 올바르지 않아요(yyyy-MM-dd)")
    String date,
    @Min(value = 1, message = "회원을 입력해 주세요")
    Long memberId,
    @Min(value = 1, message = "시간을 입력해 주세요")
    Long timeId,
    @Min(value = 1, message = "테마를 입력해 주세요")
    Long themeId
) {

    public LocalDate getDate() {
        return LocalDate.parse(date);
    }
}
```

이전에 확인했던 대로 `값이 입력되지 않으면 ID 필드의 값은 0`이 되기에 `@Min(1)` 으로 입력 여부를 검증할 수 있습니다.

날짜는 `@Pattern`  어노테이션으로 형식을 검증합니다. 날짜가 선택되지 않거나 필드가 없어 빈 문자열이 들어오면 예외가 발생하게 됩니다. 이전에 만들어둔 코드는 `date()` 라는 **record의 getter**를 사용하는데, `getDate()`라는 LocalDate를 반환하는 별도의 getter를 만든 뒤 이를 사용하게 하도록 수정하였습니다.

```java
@JsonIgnore
@Override
public String date() {
	return date
}
```

조금 더 꼼꼼하게 하려면, 위와 같이 `@JsonIgnore` 도 적용할 수 있겠네요. 하지만 요청 객체이기 때문에 크게 의미가 없어 저는 적용하지 않았습니다!

### 확인

```json
// 요청
{
    "date": "",
    "memberId": "멤버 선택",
    "themeId": "테마 선택",
    "timeId": "시간 선택"
}

// 응답
{
    "error": "BAD_REQUEST",
    "message": "테마를 선택해주세요, 회원을 선택해주세요, 시간을 선택해주세요, 날짜가 입력되지 않았거나, 입력 형식이 올바르지 않아요(yyyy-MM-dd)"
}
```

모든 값을 선택하지 않고 요청을 보냈고, 응답은 위와 같이 입력되지 않은 모든 필드에 대한 메시지를 출력하는 것을 확인할 수 있습니다. 이렇게 해서 2번 문제도 해결했습니다.

## 두 번째 방법 - Custom Deserializer 내부에서 해결

### 이전 첫 번째 방법에서의 문제

이 방법으로 간단하게 해결할 수 있었지만, **Deserializer와 요청 DTO간의 의존이 커지기도 했고,**(AdminReservationCreateRequest에 있는 어노테이션 자체가 Deserializer의 반환값에 의존하기 때문)

문제라고 생각하진 않지만 **날짜를 입력하지 않은 경우와 형식이 잘못된 경우를 하나로 묶어서 처리해야** 합니다.

따라서 코드가 복잡해지더라도 **Deserializer 내부에서 모든 예외를 처리하는 것이 더 올바른 방법**이라고 생각하고, 이번에는 이 방법으로 문제를 해결해 보겠습니다.

### 시작하기 전에

커스텀 예외를 만들어서 별도로 처리할 수 있지만, 이번에는 예시이므로 `IllegalArgumentException` 으로 예외를 처리하겠습니다.

### Custom Deserializer 구현

```java
@Override
public AdminReservationCreateRequest deserialize(JsonParser jsonParser, DeserializationContext deserializationContext)
          throws IOException, JacksonException {
          
   JsonNode rootNode = jsonParser.getCodec().readTree(jsonParser);
   
   String date = rootNode.path("date").asText();     
   Long memberId = rootNode.path("memberId").asLong();
   Long themeId = rootNode.path("themeId").asLong();
   Long timeId = rootNode.path("timeId").asLong();
	 
	 List<String> errorMessages = new ArrayList<>();
	 validateAllFields(date, memberId, themeId, timeId, errorMessages);
	 
	 ..
}
```

우선, 이전과 같이 값을 불러오는 코드는 동일합니다. 여기에 추가적으로 예외 메시지를 담을 리스트를 생성하여 `validateAllFields`에 모든 필드와 함께 넣습니다.

(더 좋은 방법이 지금은 떠오르지 않네요 ㅎㅎ.. 혹시라도 다른 방법이 있다면 의견 주시면 너무 감사하겠습니다)

```java
private void validateAllFields(String date, long memberId, long themeId, long timeId, List<String> errorMessages) {
    validateDate(date, errorMessages);
    if (memberId <= 0L) {
        errorMessage.add("회원을 선택해 주세요");
    }
    if (themeId <= 0L) {
        errorMessage.add("테마를 선택해 주세요");
    }
    if (timeId <= 0L) {
        errorMessage.add("시간을 선택해 주세요");
    }
}
```

`validateAllFields`는 `validateDate` 에 날짜 문자열과 예외 메시지를 담을 리스트를 넣어 날짜에 대해 먼저 확인하고, ID 값이 올바른지 확인합니다. `memberId == 0L` 과 같이 검증해도 충분하지만 유효한 ID값은 1 이상이기에, **0 이하이면 값이 선택되지 않은 것이라고 판단합니다**.

```java
private void validateDate(String date, List<String> errorMessages) {
    if (date.isBlank()) {
        errorMessages.add("날짜를 선택해 주세요");
        return;
    }
    try {
        LocalDate.parse(date, DateTimeFormatter.ISO_LOCAL_DATE);
    } catch (Exception e) {
        errorMessages.add("날짜는 yyyy-MM-dd 형식으로 입력해 주세요");
    }
}
```

`validateDate` 에서는 두 가지를 검증합니다. 이전의 첫 번째 방법과 달리, **입력되지 않은 경우와 형식이 올바르지 않은 경우를 따로 처리**합니다.

### 정상 입력의 경우

validateAllFields가 호출되면, 값이 올바르지 않은 경우 errorMessages 리스트에 메시지를 추가합니다. 따라서 이 리스트가 비어있다면 모든 값이 정상적으로 입력되었다고 생각할 수 있습니다.

- 입력된 ID에 대한 시간,테마,회원 등이 존재하는지는 Service에서 검증하기에, 1 이상의 숫자이면 정상적으로 입력된 것이라 판단합니다.
- 날짜의 경우, “yyyy-MM-dd” 형식인 LocalDate 타입으로 파싱할 수 있으면 정상적으로 입력된 것이라 판단합니다.

```java
@Override
public AdminReservationCreateRequest deserialize(JsonParser jsonParser, DeserializationContext deserializationContext)
          throws IOException, JacksonException {
          
	 JsonNode rootNode = jsonParser.getCodec().readTree(jsonParser);
   
   String date = rootNode.path("date").asText();     
   Long memberId = rootNode.path("memberId").asLong();
   Long themeId = rootNode.path("themeId").asLong();
   Long timeId = rootNode.path("timeId").asLong();
   
	 List<String> errorMessages = new ArrayList<>();
	 validateAllFields(date, memberId, themeId, timeId, errorMessages);
	 
	 
	 if (errorMessage.isEmpty()) {
			return new AdminReservationCreateRequest(LocalDate.parse(date), memberId, timeId, themeId);
	 }
	 ..
}
```

따라서, 이 경우에는 `AdminReservationCreateRequest` 객체를 생성하여 바로 반환합니다.

### 정상 입력이 아닌 경우

```java
@Override
public AdminReservationCreateRequest deserialize(JsonParser jsonParser, DeserializationContext deserializationContext)
          throws IOException, JacksonException {
          
	 JsonNode rootNode = jsonParser.getCodec().readTree(jsonParser);
   
   String date = rootNode.path("date").asText();     
   Long memberId = rootNode.path("memberId").asLong();
   Long themeId = rootNode.path("themeId").asLong();
   Long timeId = rootNode.path("timeId").asLong();
   
	 List<String> errorMessages = new ArrayList<>();
	 validateAllFields(date, memberId, themeId, timeId, errorMessages);
	 
	 if (errorMessage.isEmpty()) {
			return new AdminReservationCreateRequest(LocalDate.parse(date), memberId, timeId, themeId);
	 }
	 
	 String combinedMessage = String.join(", ", errorMessages);
	 throw new IllegalArgumentException(combinedMessage);
}
```

String.join을 이용해 리스트 안의 모든 메시지를 합친 뒤 예외를 던집니다.

### 요청 DTO 수정

```java
@JsonDeserialize(using = AdminReservationCreateRequestDeserializer.class)
public record AdminReservationCreateRequest(
    String date,
    Long memberId,
    Long timeId,
    Long themeId
) {
}
```

위와 같이 Deserializer에서 검증을 하게 되면, DTO에는 별도의 어노테이션을 넣지 않아도 됩니다.

### 확인

```json
// 모든 필드를 입력하지 않고 요청
{
    "date": "",
    "memberId": "멤버 선택", 
    "themeId": "테마 선택",
    "timeId": "시간 선택"
}

// 모든 필드를 입력하지 않을 때의 응답
{
    "error": "BAD_REQUEST",
    "message": "날짜가 선택되지 않았습니다, 회원이 선택되지 않았습니다, 테마가 선택되지 않았습니다, 시간이 선택되지 않았습니다"
}

// 다른 필드는 입력하지 않고, 날짜는 형식을 다르게 입력하여 요청
{
    "date": "20240620",
    "memberId": "멤버 선택", 
    "themeId": "테마 선택",
    "timeId": "시간 선택"
}

// 날짜 형식이 올바르지 않을 때의 응답
{
    "error": "BAD_REQUEST",
    "message": "날짜 형식이 올바르지 않습니다, 회원이 선택되지 않았습니다, 테마가 선택되지 않았습니다, 시간이 선택되지 않았습니다"
}
```

응답 결과가 예상한 것과 동일하게 나온 것을 확인할 수 있습니다😄

## 결론

### 요약

1. 해결해야 할 문제는 **클라이언트 코드에서 반환하는 값에 의존하는 것과, 여러 필드에서 예외가 발생했을 때 예외가 발생한 필드에 대한 메시지를 한 번에 응답하지 못하는 것**이었습니다.
2. 첫 번째 방법에선, **Custom Deserializer와 @Valid**를 함께 이용하여 해결합니다. 이 방법은 **코드가 훨씬 이해하기 쉽고 간결하다는 장점이 있지만, DTO와 역직렬화 객체 간의 의존이 커지는 문제**가 있었습니다.
3. 그래서 두 번째 방법에선 **Custom Deserializer 내부에서 확인**합니다. 구현 방식이 크게 마음에 드는 것은 아니지만, 의존을 제거할 수 있다는 큰 장점이 있었습니다.

드디어 길고긴 예외 처리가 마무리 되었네요. 1편부터 지금까지의 전체 흐름을 다시 정리해 보겠습니다.

1. 현재 요청은 예약 추가, 테마 추가, 시간 추가로 구성되어 있습니다. 1편에서는 발생되는 예외 타입을 확인하고 테마 추가, 시간 추가에서 발생하는 @Valid 어노테이션에 의한 MethodArgumentNotValidException을 처리합니다.
2. 하지만, 예약 추가의 경우 날짜와 ID에서 발생하는 예외 타입이 달라서 처리할 수 없었습니다.
3. 2편에서는 Custom Deserializer의 사용에 필요한 JsonNode의 대략적인 구성, 사용법을 다룹니다.
4. 3편에서는 2편에 있는 내용을 바탕으로 1편에서 해결하지 못한 문제점을 해결합니다.

최대한 맥락을 살려서 작성했다고 생각하지만, 글이 길어지다 보니 중간에 이해하기 힘든 내용이 있을 것이라고 생각합니다. 부족한 부분은 댓글에 남겨주시면 정말 감사하겠습니다.🙇

부족한 글이지만 읽어주셔서 감사합니다. 즐거운 하루 보내세요😄

```toc
```