# DDD 정리

인아웃 백엔드 과제(단식 루틴·알림, NestJS + MikroORM)를 DDD로 구현하며 익힌 것. 키워드별 핵심만.

## 한 문장

업무 규칙을 프레임워크·DB·HTTP와 분리된 **도메인 모델**에 모으고, 의존 방향을 **안쪽(도메인)으로만** 향하게 하는 설계 방법.

## Value Object (값 객체)

- 식별자 없음. **값이 같으면 같은 것.** 불변.
- 생성 시점에 검증하고, 그 뒤로는 "이 타입이면 유효하다"를 믿는다.
- 모양: `private` 생성자 + 정적 팩토리(`of`, `parse`) + `equals()` + `readonly` 필드.
- 과제: `Weekdays`(요일 비트마스크, 1개 이상·중복 불가), `LocalTime`(HH:mm), `FastingInterval`(시작 < 끝, 겹침 판정), `RoutineSchedule`(이들의 조합).

## Entity (엔티티)

- **ID로 구분.** 속성이 다 바뀌어도 같은 것. 상태가 변하는 생명주기가 있다.
- 필드를 직접 대입하지 않고 **업무 동사 메서드**로만 바꾼다. `setStatus` ✗ → `completeIfDue`, `activate`, `markDeleted` ✓
- 이미 그 상태면 예외 대신 `false` 반환(멱등). 중복 처리에 안전.
- 현재 시각은 `new Date()` 대신 **`now`를 인자로** 받는다. 테스트에서 시간을 옮길 수 있다.
- 과제: `Routine`, `FastingSession`(IN_PROGRESS → COMPLETED), `SessionNotification`(PENDING → SENT | SKIPPED).

## Aggregate (애그리거트)

- **함께 바뀌어야 일관성이 유지되는 묶음.** 대표인 루트를 통해서만 접근하고, 저장·조회도 루트 단위.
- **애그리거트끼리는 ID로만 참조.** `session.routineId: string`. ORM 관계 매핑(`@ManyToOne`) 안 씀.
- 불변식(invariant)은 루트가 지킨다. "활성·미삭제일 때만 다음 예정이 있다"를 private 메서드 하나(`scheduleNextAfter`)로 모아 모든 상태 전이가 호출. DB CHECK는 마지막 안전망.
- 트랜잭션 경계 = 애그리거트 경계. 작을수록 좋다. "같이 안 바뀌면 같은 애그리거트가 아니다."
- 수정은 리포지토리 `update()`가 아니라 ORM 변경 감지로 커밋 때 저장. 새로 만든 것만 `add()`.

## Domain (도메인 층)

- 가장 안쪽. **아무것도 의존하지 않는다.** NestJS, ORM, HTTP, Clock 전부 모름.
- 들어 있는 것: 애그리거트, 값 객체, 도메인 오류, 리포지토리 **포트**(인터페이스), 정책.
- 도메인 오류는 HTTP 상태가 아니라 **위반 종류**(`INVALID` / `NOT_FOUND` / `CONFLICT`)만 안다. 상태 코드 변환은 프레젠테이션 필터가.
- 도메인 클래스에 ORM 데코레이터 안 붙임. 매핑은 인프라의 `EntitySchema`에. 덕분에 도메인 테스트가 DB 없이 돈다.

## Domain Service vs Application Service

| | 도메인 서비스(정책) | 애플리케이션 서비스(유스케이스) |
|---|---|---|
| 하는 일 | 여러 애그리거트·값을 놓고 **판단** | 순서, 트랜잭션, 포트 호출, 입출력 변환 |
| 아는 것 | 도메인 객체만. 상태 없음 | 도메인 + 포트(리포지토리, Clock 등) |
| 과제 | `OverlapPolicy.findConflict(candidates, others)` | `RoutineOverlapService`가 DB에서 구간을 모아 정책에 넘김 |

- 배치 기준: 애그리거트에 넣을 수 있으면 거기 → 안 되면 도메인 서비스 → DB를 읽어야 하면 유스케이스.
- 유스케이스에 `if`가 늘고 애그리거트가 getter/setter만 가지면 **빈약한 모델** 신호.
- 접미사: `~UseCase`(쓰기 1건), `~QueryService`(조회), `~Service`(공유 애플리케이션 로직), `~Policy`(도메인 규칙).

## Repository / Port (리포지토리와 포트)

- 리포지토리 = 애그리거트를 컬렉션처럼 다루는 인터페이스. **루트마다 하나.**
- 포트(계약)는 **domain**에, 구현(어댑터)은 **infrastructure**에. 인프라가 도메인을 구현하므로 의존이 역전된다.
- 리포지토리만 포트가 아니다. `TransactionRunner`, `Clock`, `NotificationSender`, 다른 컨텍스트의 `UserDirectory`도 포트. **테스트에서 바꿔 끼울 것은 전부 포트.**
- 메서드는 범용(`findWhere`)이 아니라 **업무 질문**(`findDueIds`, `findOwned`, `findForUpdate`). 계약(정렬, null 조건)은 주석에.
- NestJS DI 토큰으로 쓰려고 `interface` 대신 **추상 클래스**. 포트 ↔ 구현 바인딩은 모듈 파일 한 곳에서만.

## Bounded Context (바운디드 컨텍스트)

- **하나의 모델과 용어가 일관되게 통하는 범위.** 코드에서는 최상위 폴더 하나.
- 과제: `fasting`(루틴·회차·알림), `user`(유저 존재 확인), `shared`(컨텍스트를 모르는 기반 코드).
- 컨텍스트 간 통신은 **포트로.** `fasting`은 `user`의 `UserDirectory.exists(id)`만 부른다. 유저 객체를 넘기지 않고 질문만. 방향은 한쪽(`fasting → user`).
- `shared` 판단 기준: "이 코드가 fasting이라는 단어를 아는가?" 알면 shared 아님.
- 경계는 폴더 + **import 규칙(dependency-cruiser)**으로 박아야 유지된다.

## Ubiquitous Language (유비쿼터스 언어)

- 명세·문서·코드가 **같은 단어.** 설계 문서 맨 위에 용어표부터.
- 과제: 루틴=`Routine`, 회차=`FastingSession`, 작업자=`FastingScheduler`, 알림=`SessionNotification`.
- 시간 이름은 접미사로 종류 구분: `~At`(UTC 순간), `~LocalDate`(현지 날짜), `~Minute`/`~LocalTime`(현지 시각). 이게 없으면 시간대 버그.

## Layers (계층과 의존 방향)

```
presentation → application → domain ← infrastructure
```

| 층 | 역할 | 금지 |
|---|---|---|
| domain | 업무 규칙 | 프레임워크, ORM, Clock |
| application | 유스케이스, 조회, 트랜잭션 경계 | ORM, 컨트롤러, 요청·응답 타입 |
| infrastructure | 포트 구현, 테이블 매핑 | 유스케이스, 컨트롤러 |
| presentation | 컨트롤러, 요청·응답, **타이머 같은 비HTTP 진입점도** | 리포지토리 직접 호출, 인프라 |

- 층을 건너는 데이터: `Request → Command → [UseCase] → Result → Response`. 유스케이스는 **도메인 객체를 반환하지 않는다.**
- 규칙은 `.dependency-cruiser.ts`에 옮겨 `verify`에서 자동 검사. 문서만으로는 깨진다.

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
