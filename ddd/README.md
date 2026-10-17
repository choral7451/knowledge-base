# DDD 정리

인아웃 백엔드 과제(단식 루틴·알림, NestJS + MikroORM)를 DDD로 구현하며 익힌 것.

**한 문장**: 업무 규칙을 프레임워크·DB·HTTP와 분리된 **도메인 모델**에 모으고, 의존 방향을 **안쪽(도메인)으로만** 향하게 하는 설계 방법.

## 키워드

| 키워드 | 정의 | 핵심 규칙 | 과제 예 |
|---|---|---|---|
| **Value Object**<br>값 객체 | 식별자가 없고 **값이 같으면 같은 것.** 불변. | `private` 생성자 + 정적 팩토리에서 검증 + `equals()`. 바꾸려면 새로 만든다. 한 번 만들어지면 유효하다고 믿는다. | `Weekdays`(요일 비트마스크), `LocalTime`(HH:mm), `FastingInterval`(시작<끝, 겹침 판정), `RoutineSchedule`(조합) |
| **Entity**<br>엔티티 | **ID로 구분.** 속성이 다 바뀌어도 같은 것. 상태가 변하는 생명주기가 있다. | 필드 직접 대입 ✗, 업무 동사 메서드로만(`completeIfDue`, `markDeleted`). 이미 그 상태면 `false` 반환(멱등). `now`는 인자로 받는다. | `Routine`, `FastingSession`(IN_PROGRESS→COMPLETED), `SessionNotification`(PENDING→SENT/SKIPPED) |
| **Aggregate**<br>애그리거트 | **함께 바뀌어야 일관성이 유지되는 묶음.** 루트를 통해서만 접근, 저장·조회도 루트 단위. | 애그리거트끼리 **ID로만 참조**(`routineId: string`). 불변식은 루트의 메서드 하나로 모은다. 트랜잭션 경계 = 애그리거트 경계. 작을수록 좋다. 수정은 ORM 변경 감지, 새 것만 `add()`. | 루틴·회차·알림 세 애그리거트. "활성·미삭제일 때만 다음 예정"을 `scheduleNextAfter` 하나가 지킴 |
| **Domain**<br>도메인 층 | 업무 규칙만 있는 **가장 안쪽 층.** 아무것도 의존하지 않는다. | NestJS·ORM·HTTP·Clock 전부 모름. ORM 데코레이터 안 붙임(매핑은 인프라). 오류는 HTTP 상태가 아니라 종류(`INVALID`/`NOT_FOUND`/`CONFLICT`)만. | `fasting/domain/` 아래 애그리거트, 값 객체, 오류, 포트, 정책. DB 없이 테스트 |
| **Domain Service**<br>도메인 서비스 | 여러 애그리거트·값을 놓고 **판단**하는 상태 없는 규칙. | 도메인 객체만 안다. 리포지토리 모름. 정적 메서드로 충분. 애그리거트에 넣을 수 있으면 거기가 우선. | `OverlapPolicy.findConflict(candidates, others)` |
| **Application Service**<br>유스케이스 | 순서·트랜잭션·포트 호출·입출력 변환을 **조립**하는 층. 규칙 판단은 안 한다. | `execute(command)` 하나. 트랜잭션 열고, `now` 읽고, 도메인에 판단 맡기고, Result로 변환해 반환. 유스케이스에 `if`가 늘면 빈약한 모델 신호. | `CreateRoutineUseCase`, `RoutineOverlapService`(DB에서 구간 모아 정책에 넘김), `SessionStartService`(공유 로직) |
| **Repository**<br>리포지토리 | 애그리거트를 **컬렉션처럼** 다루는 인터페이스. 루트마다 하나. | 메서드는 업무 질문(`findDueIds`, `findOwned`, `findForUpdate`). 계약은 주석에. `update()` 없음. | `RoutineRepository`, `FastingSessionRepository`, `SessionNotificationRepository` |
| **Port / Adapter**<br>포트와 어댑터 | 포트 = 안쪽이 바깥에 요구하는 **계약.** 어댑터 = 그 구현. | 포트는 domain/application에, 구현은 infrastructure에 → 의존 역전. 테스트에서 바꿔 끼울 것은 전부 포트. 바인딩은 모듈 파일 한 곳. DI 토큰용으로 추상 클래스. | `TransactionRunner`, `Clock`, `NotificationSender`, `UserDirectory` ↔ `MikroOrm~` 구현 |
| **Bounded Context**<br>바운디드 컨텍스트 | 하나의 모델과 용어가 **일관되게 통하는 범위.** | 최상위 폴더 하나. 컨텍스트 간 통신은 포트로, 객체 대신 질문만(`exists(id)`). 방향은 한쪽. `shared`는 컨텍스트 이름을 모르는 코드만. import 규칙으로 강제. | `fasting`, `user`, `shared`. `fasting → user`는 `UserDirectory` 포트만 |
| **Ubiquitous Language**<br>유비쿼터스 언어 | 명세·문서·코드가 **같은 단어**를 쓰는 것. | 설계 문서 맨 위에 용어표. 메서드는 업무 동사. 시간 접미사 구분: `~At`(UTC 순간), `~LocalDate`(현지 날짜), `~Minute`(현지 시각). | 루틴=`Routine`, 회차=`FastingSession`, 작업자=`FastingScheduler`, 알림=`SessionNotification` |
| **Domain Error**<br>도메인 오류 | 업무 규칙 위반을 뜻하는 예외. | `kind`(종류) + `code` + `field`(틀린 입력)만. HTTP 상태 변환은 프레젠테이션 필터. 남의 것은 403 대신 404(존재를 숨김). | `RoutineOverlapError`, `RoutineLimitExceededError`, `RoutineNotFoundError` |
| **Invariant**<br>불변식 | 애그리거트가 **항상 참**으로 유지해야 하는 조건. | 루트가 1차 방어, DB CHECK 제약은 마지막 안전망. 모든 상태 전이가 같은 private 메서드를 거치게 한다. | "다음 예정은 활성·미삭제 루틴에만", "완료일 때만 `endedAt`" |

## 계층과 의존 방향

```
presentation → application → domain ← infrastructure
```

| 층 | 들어 있는 것 | 금지 |
|---|---|---|
| domain | 애그리거트, 값 객체, 오류, 포트, 정책 | 프레임워크, ORM, Clock |
| application | 유스케이스, 조회 서비스, Command/Result 타입 | ORM, 컨트롤러, 요청·응답 타입 |
| infrastructure | 포트 구현, `EntitySchema` 매핑, 마이그레이션 | 유스케이스, 컨트롤러 |
| presentation | 컨트롤러, 요청·응답, 타이머 같은 비HTTP 진입점 | 리포지토리 직접 호출, 인프라 |

- 데이터 흐름: `Request → Command → [UseCase] → Result → Response`. 유스케이스는 도메인 객체를 반환하지 않는다.
- 규칙은 `.dependency-cruiser.ts`로 자동 검사. 문서만으로는 깨진다.

## 디렉토리 샘플

```
src/
├── shared/                      컨텍스트 무관 기반
│   ├── domain/domain-error.ts
│   ├── application/transaction-runner.ts
│   ├── infrastructure/persistence/   ORM 설정, 트랜잭션 구현, migrations/
│   └── presentation/                 오류 필터, 검증 파이프, 공통 응답
├── customer/                    다른 컨텍스트. 포트(application)만 공개
│   ├── customer.module.ts
│   ├── domain/
│   ├── application/customer-directory.ts
│   └── infrastructure/
└── order/                       주문 컨텍스트
    ├── order.module.ts          포트 ↔ 구현 바인딩 (유일한 곳)
    ├── domain/
    │   ├── order/               애그리거트 하나 = 폴더 하나
    │   │   ├── order.ts                 루트
    │   │   ├── order.repository.ts      포트
    │   │   ├── order.errors.ts
    │   │   └── order.spec.ts            DB 없는 테스트는 파일 옆
    │   └── pricing/
    │       ├── money.ts                 값 객체
    │       └── discount-policy.ts       도메인 정책
    ├── application/order/
    │   ├── place-order.use-case.ts      Command 타입은 이 안에
    │   ├── order-query.service.ts
    │   └── order.result.ts
    ├── infrastructure/persistence/
    │   ├── order.schema.ts              EntitySchema 매핑
    │   └── orm-order.repository.ts      포트 구현
    └── presentation/
        ├── http/
        │   ├── order.controller.ts      받고 → toCommand → execute → Response.from
        │   ├── order.request.ts
        │   └── order.response.ts
        └── scheduler/                   타이머·컨슈머
test/
└── support/fake-ports.ts        모든 포트의 가짜 구현 한 곳에
.dependency-cruiser.ts
```

접미사: 없음(애그리거트·값 객체·포트·정책) / `.use-case` / `.service` / `.result` / `.repository` / `.schema` / `.controller` / `.request` / `.response` / `.module` / `.spec`

## 과제에서 안 쓴 것

이벤트 버스, CQRS, 도메인↔영속 모델 이중 매핑, 팩토리 클래스. 컨텍스트 둘·애그리거트 셋 규모에는 과했다.
