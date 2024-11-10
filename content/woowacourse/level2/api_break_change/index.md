---
emoji: '🌱'
title: API Break Change 방지를 위한 리스트 형태의 JSON 응답 수정
date: '2024-07-08 19:00:00'
author: 이상진
tags: 안녕하세요!
categories: level2 Spring API
---

## 배경

[우아한테크코스의 세 번째 미션](https://github.com/woowacourse/spring-roomescape-waiting/pull/30)을 진행하며, 리뷰어께서 좋은 의견을 주셨습니다. 웹 구현을 처음
해보는 입장에서 생각지도 못했던 `API Break Change` 에 대한 것인데요, API JSON 응답은 **List가 아닌 Object 형식**으로 하는 것을 권하셨고 이번 글에서는 이 내용에 대해 기록해보고자
합니다.

<br/>

## API Break Change란?

[출처](https://nordicapis.com/how-to-manage-breaking-changes-throughout-an-apis-lifecycle/)에 있는 글에서는 Breaking Change를 다음과
같이 설명합니다.

> A[breaking change](https://nordicapis.com/what-are-breaking-changes-and-how-do-you-avoid-them/)is when one such change
> causes a client application to break somehow. While some changes make a minimal impact, breaking changes are those
> fundamental changes that cause the system to cease functioning. This could be a change in a field name, the removal of
> an unused resource type, a new feature that deprecates older features, and so forth.
>

이 글을 요약해보면,

1. **클라이언트 애플리케이션이 중지되도록 하는 변경**을 Breaking Change라고 한다.
2. 변경 사항이 최소한의 영향을 미칠수도 있지만, 시스템 작동을 중단시킬 수도 있다.
3. 필드 이름 변경, 사용하지 않는 리소스 제거, 기존의 기능을 deprecate하는 새로운 기능 등이 있다.

그렇다면, **현재 API 코드의 변화가 기존의 클라이언트 코드에 영향을 미치지 않도록 설계하는것이 중요**하겠네요! 다음으로는 기존에 구현했던 방법을
작성해 보겠습니다.

<br/>

## 기존 구현

예시를 위한 코드는 기존과 같은 형식으로, 내용만 조금 간단하게 작성하겠습니다.

```java
@Entity
@NoArgsConstructor(access = lombok.AccessLevel.PROTECTED)
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private Long age;

    @Builder
    public Member(String name, Long age) {
        this.name = name;
        this.age = age;
    }
}
```

엔티티는 위와 같이 이름, 나이로 구성되어 있습니다. 사실 코드 자체가 중요한게 아니고 이름(name), 나이(age) 필드가 있다는 것만 알아도 충분합니다.

<br/>

```java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberRepository memberRepository;

    @GetMapping("/members")
    public List<Member> getAllMembers() {
        return memberRepository.findAll();
    }
}
```

테스트를 위해 간단하게 작성한 코드라, 별도의 Layer 구분 없이 Repositroy를 컨트롤러에서 바로 사용하도록 작성하였습니다. 기존 구현은 위와 같이 `GET /members` 요청을 보내면 전체 회원을
반환하고 있습니다.

<br/>

```json
[
  {
    "id": 1,
    "name": "sangdol",
    "age": 10
  },
  {
    "id": 2,
    "name": "sangdol1",
    "age": 20
  }
]
```

응답은 위와 같은 JSON 배열로 나갈 것이고, **이 JSON만 보고 어떤 데이터인지 바로 파악하기 쉽지 않다는 문제**가 있긴 하지만 클라이언트 코드에서 이를 처리하는 것 자체는 크게 문제가 없는 것 같습니다.

<br/>

```jsx
function fetchMembers() {
  const data = fetch('/members').json();

  data.forEach(member => ..)
}
```

클라이언트 코드는 대략 이런 형태로 구성될 것 같고, 웹을 처음 구현하는 입장에선 클라이언트 코드 조금 고치면 되니깐 크게 문제가 있다는 생각을 하진 못했습니다.

<br/>

## 리뷰어 피드백

리뷰어께서 [아티클](https://blog.stackademic.com/creating-a-high-quality-rest-api-201819325356)을 바탕으로 피드백을 주셨는데, 해당 링크에 들어가면 다음과
같은 내용이 나옵니다.

> • Don’t return arrays as top-level responses : The top-level response from an endpoint should be an Object and not
> an array. For instance GET /books returns: {“data”:[{…book1…}], [{…book2…}]} is good and GET /books returns:[{…book1…}]
> GET is bad. Returning arrays is not recommendable as it makes it hard to make changes to the output that are backward
> compatible. For example, adding a pagination field like totalCount. This will be a breaking change for the client.
>

기존에 구현한 방법과 같이 List 형태의 응답보단, 확장성을 고려해 이를 감싼 Object 형식으로 반환하라는 내용인데요, 사실 이 부분이 크게 와닿진 않았지만 리뷰어께서 남겨주신 다음 의견을 보고 어떠한
의도셨는지 바로 이해할 수 있었습니다.

> 서버 코드 변경에 맞추어 프론트 코드를 항상 바꾸어달라고 요구하는 건 현업에서 매우 어려운 일입니다 (단순히 공수를 넘어 배포 일정, 정치적인 관계 등 다양한 변수가 존재합니다 ㅋㅋ 누가 상돌 미션 처리할 거
> 4개 쌓였는데 '죄송한데 이거 조금만 수정해서 내일 새벽 6시에 배포좀 같이 해주시면 안 될까요? ㅠㅠ' 라고 부탁했다고 상상해보세요)
>

이번 미션에서는 간단한 클라이언트 코드 수정은 직접 했지만, 프로젝트를 하거나 회사에 가면 이 변경을 프론트쪽에서도 반영을 해주어야 하는데.. 리뷰어께서 말씀하신 대로 쉬운 작업은 아니겠다는 생각을 할 수 있었습니다
ㅎㅎ 이제 마지막으로 이 피드백을 바탕으로 수정한 방법을 작성해보겠습니다.

<br/>

## 구현 방법 수정

```java
public record MembersResponse(
    List<Member> members
) {
}
```

```java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberRepository memberRepository;

    @GetMapping("/members")
    public MembersResponse getAllMembers() {
        List<Member> members = memberRepository.findAll();
        return new MembersResponse(members);
    }
}
```

위와 같이, `List<Member>` 를 감싸는 별도의 응답 DTO를 만들어 반환하도록 수정하였습니다.

<br/>

```json
{
  "members": [
    {
      "id": 1,
      "name": "sangdol",
      "age": 10
    },
    {
      "id": 2,
      "name": "sangdol1",
      "age": 20
    }
  ]
}
```

그러면 응답 JSON 형태가 위와 같이 바뀌게 되며, 기존 구현과 달리 JSON 만을 보고도 회원 정보 데이터라는 것을 바로 파악할 수 있고 회원 정보 필드 이외의 다른 값(전체 데이터 개수)을 
**새로운 필드로 추가해도 기존의 클라이언트 코드는 정상적으로 작동**하기에 Breaking Change의 발생 가능성을 줄일 수 있겠네요!

<br/>

## 결론

1. 응답을 JSON Arrray 형식으로 반환하는 것은 확장하기 힘든 구조이며, 따라서 API Breaking Change의 가능성이 크다.
2. 따라서 처음부터 Array가 아닌 Object 형식으로 이를 감싸서 응답하는 방법으로 Breaking Change의 발생 가능성을 줄일 수 있다.

리뷰어께서 좋은 의견을 주신 덕분에 생각하지도 못한 것들에 대해 새로 알아갈 수 있어 유익한 시간이었습니다 ㅎㅎ

읽어주셔서 감사합니다. 즐거운 하루 보내세요🙇