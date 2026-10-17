# DDD 개요와 읽는 순서

인아웃 백엔드 과제(단식 루틴·예약 알림, NestJS + MikroORM + MySQL)를 DDD로 구현하면서 익힌 내용입니다.
교과서 정의보다 "실제 코드에서 어떤 모양이었고, 왜 그렇게 나눴는지"를 중심으로 적었습니다.

## DDD를 한 문장으로

**업무 규칙을 프레임워크·DB·HTTP와 분리된 도메인 모델에 모으고, 그 모델을 중심으로 코드를 배치하는 방법.**

과제에서 체감한 핵심은 두 가지였습니다.

1. **규칙은 한 곳에서만 지킨다.** "지속 시간은 1~24", "같은 유저의 단식 구간은 겹칠 수 없다" 같은 규칙이 컨트롤러·서비스·SQL에 흩어지지 않고 값 객체·애그리거트·정책 클래스 안에 있다.
2. **의존 방향은 안쪽으로만.** 도메인은 NestJS도 MikroORM도 모른다. 바깥 층이 도메인을 가져다 쓰지, 도메인이 바깥을 부르지 않는다.

## 전술적 패턴 지도

```
Bounded Context (fasting, user)
└── Aggregate (Routine, FastingSession, SessionNotification)
    ├── Entity          식별자로 구분. 애그리거트 루트가 대표적인 엔티티
    ├── Value Object    값으로 구분. Weekdays, LocalTime, FastingInterval ...
    ├── Domain Error    규칙 위반을 뜻하는 예외. RoutineOverlapError ...
    └── Repository(Port) 애그리거트를 저장·조회하는 추상 인터페이스
Domain Service / Policy   한 애그리거트에 넣기 어색한 규칙. OverlapPolicy
Application Service / Use Case  트랜잭션·순서·포트 호출을 조립. CreateRoutineUseCase
```

## 읽는 순서

1. [값 객체](value-object.md) — 가장 작고, 가장 많이 만들게 되는 것
2. [엔티티](entity.md) — 식별자와 생명주기
3. [애그리거트](aggregate.md) — 일관성 경계, ID로만 참조
4. [도메인 서비스와 애플리케이션 서비스](domain-service-vs-application-service.md) — "어디에 둘까"의 기준
5. [리포지토리와 포트](repository-and-ports.md) — 도메인이 인프라를 모르는 방법
6. [바운디드 컨텍스트와 유비쿼터스 언어](bounded-context.md) — 컨텍스트 나누기, 용어 맞추기
7. [계층 구조와 의존 방향](layered-architecture.md) — 네 층, 검사 자동화
8. [디렉토리 구조 샘플](directory-structure-sample.md) — 새 프로젝트에 바로 쓸 뼈대

## 과제에서 의도적으로 뺀 것

DDD 책에 나오지만 규모에 비해 과했던 것들. "안 쓰기로 한 결정"도 기록해 둡니다.

| 뺀 것 | 이유 |
|---|---|
| 도메인 이벤트, 이벤트 버스 | 컨텍스트 간 통신이 `UserDirectory.exists` 하나뿐 |
| CQRS(명령/조회 모델 분리) | 조회 서비스가 같은 애그리거트를 읽어 결과 타입으로 바꾸는 것으로 충분 |
| 도메인 모델 ↔ 영속 모델 이중 매핑 | MikroORM `EntitySchema`로 도메인 클래스를 데코레이터 없이 그대로 매핑 가능 |
| 팩토리 클래스 | 정적 팩토리 메서드(`Routine.register`, `FastingSession.start`)로 충분 |
