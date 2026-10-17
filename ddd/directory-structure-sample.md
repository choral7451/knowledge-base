# 디렉토리 구조 샘플

새 프로젝트를 DDD 네 층으로 시작할 때 복사해 쓸 뼈대. 과제에서 검증된 구조를 일반화했다.
예시 컨텍스트는 `order`(주문), 다른 컨텍스트와의 관계를 보이기 위해 `customer`를 둔다.

## 뼈대

```
src/
├── main.ts
├── app.module.ts
│
├── shared/                                    컨텍스트와 무관한 기반. 특정 컨텍스트를 import하지 않음
│   ├── domain/
│   │   └── domain-error.ts                    DomainError 기반 클래스 (kind, code, field)
│   ├── application/
│   │   └── transaction-runner.ts              트랜잭션 포트 (추상 클래스)
│   ├── infrastructure/
│   │   └── persistence/
│   │       ├── orm.config.ts
│   │       ├── persistence.module.ts
│   │       ├── orm-transaction-runner.ts      TransactionRunner 구현
│   │       └── migrations/
│   │           └── 0001-create-order-tables.ts
│   └── presentation/
│       ├── http.module.ts                     필터·파이프를 전역 적용
│       ├── error-response.filter.ts           DomainError.kind → HTTP 상태
│       ├── request-validation.pipe.ts
│       ├── id-param.pipe.ts
│       ├── current-user-id.decorator.ts
│       ├── list.response.ts
│       └── error.response.ts
│
├── customer/                                  다른 컨텍스트. order가 포트로만 접근
│   ├── customer.module.ts                     포트 ↔ 구현 바인딩, exports: [CustomerDirectory]
│   ├── domain/
│   │   ├── customer.ts
│   │   └── customer.errors.ts
│   ├── application/
│   │   └── customer-directory.ts              다른 컨텍스트에 열어 주는 포트
│   └── infrastructure/
│       ├── customer.schema.ts
│       └── orm-customer-directory.ts
│
└── order/                                     주문 컨텍스트
    ├── order.module.ts                        이 컨텍스트의 포트 ↔ 구현을 묶는 유일한 곳
    │
    ├── domain/
    │   ├── order/                             애그리거트 하나 = 폴더 하나
    │   │   ├── order.ts                       애그리거트 루트. 정적 팩토리 + 상태 전이 메서드
    │   │   ├── order-line.ts                  루트에 속한 엔티티 (있다면)
    │   │   ├── order-status.ts                값 객체 / 유니온 타입
    │   │   ├── order.repository.ts            리포지토리 포트 (추상 클래스)
    │   │   ├── order.errors.ts                이 애그리거트의 도메인 오류
    │   │   └── order.spec.ts                  DB 없는 단위 테스트는 파일 옆에
    │   ├── payment/
    │   │   ├── payment.ts
    │   │   ├── payment.repository.ts
    │   │   └── payment.errors.ts
    │   └── pricing/                           애그리거트에 속하지 않는 값 객체·정책
    │       ├── money.ts                       값 객체
    │       ├── discount-policy.ts             상태 없는 도메인 정책 (정적 메서드)
    │       └── discount-policy.spec.ts
    │
    ├── application/
    │   ├── order/
    │   │   ├── place-order.use-case.ts        쓰기 한 건. Command 타입은 이 파일 안에
    │   │   ├── cancel-order.use-case.ts
    │   │   ├── order-query.service.ts         조회 전용. 트랜잭션 없이 읽어 Result로
    │   │   ├── order-stock.service.ts         여러 유스케이스가 공유하는 애플리케이션 로직
    │   │   ├── order.result.ts                유스케이스·조회 서비스의 출력 타입 + toOrderResult()
    │   │   └── place-order.use-case.spec.ts   가짜 포트로 흐름만 검증
    │   └── payment/
    │       ├── confirm-payment.use-case.ts
    │       └── payment.result.ts
    │
    ├── infrastructure/
    │   └── persistence/
    │       ├── order.schema.ts                EntitySchema 매핑. 도메인 클래스에 데코레이터 없음
    │       ├── payment.schema.ts
    │       ├── orm-order.repository.ts        OrderRepository 구현
    │       └── orm-payment.repository.ts
    │
    └── presentation/
        ├── http/
        │   ├── order.controller.ts            받고 → toCommand → execute → Response.from
        │   ├── order.request.ts               검증 데코레이터 + toCommand()
        │   ├── order.response.ts              JSON 모양 + static from(result)
        │   └── order.docs.ts                  Swagger 데코레이터 묶음
        └── scheduler/                         타이머·컨슈머 같은 비HTTP 진입점도 여기
            └── payment-expiry.scheduler.ts    tick() 하나. 유스케이스를 주기마다 부를 뿐

test/                                          DB가 필요한 통합 테스트
├── support/
│   ├── test-app.ts                            createTestApp()
│   ├── database.ts                            clearDomainTables()
│   └── fake-ports.ts                          모든 포트의 가짜 구현을 한 곳에
├── order/
│   ├── place-order.spec.ts
│   └── query-orders.spec.ts
└── payment/
    └── confirm-payment.spec.ts

.dependency-cruiser.ts                         의존 방향 검사 규칙
```

## 파일 접미사 규칙

| 접미사 | 뜻 | 층 |
|---|---|---|
| (없음) | 애그리거트, 엔티티, 값 객체, 포트, 정책, 기반 클래스 | domain, shared |
| `.repository.ts` | 리포지토리 포트 또는 구현 | domain / infrastructure |
| `.errors.ts` | 도메인 오류 모음 | domain |
| `.use-case.ts` | 쓰기 유스케이스. 메서드는 `execute(command)` | application |
| `.service.ts` | 조회 서비스, 공유 애플리케이션 서비스 | application |
| `.result.ts` | 유스케이스 출력 타입 | application |
| `.schema.ts` | 테이블 매핑 | infrastructure |
| `.controller.ts` | HTTP 컨트롤러 | presentation |
| `.request.ts` / `.response.ts` | 요청·응답 타입 | presentation |
| `.docs.ts` | API 문서 데코레이터 | presentation |
| `.filter.ts` / `.pipe.ts` / `.decorator.ts` | 전역 공통 처리 | shared/presentation |
| `.module.ts` | NestJS 모듈 (포트 바인딩) | 컨텍스트 루트 |
| `.spec.ts` | 테스트 | 어디든 |

## 네이밍 규칙

| 대상 | 규칙 | 예 |
|---|---|---|
| 포트 | 접두사 없는 추상 클래스 | `OrderRepository`, `TransactionRunner` |
| 포트 구현 | 기술 이름 접두사 | `OrmOrderRepository`, `MikroOrmOrderRepository` |
| 유스케이스 | 동사 + 대상 + `UseCase` | `PlaceOrderUseCase` |
| 커맨드 / 결과 | 동사 + `Command`, 대상 + `Result` | `PlaceOrderCommand`, `OrderResult` |
| 조회 | 대상 + `QueryService` | `OrderQueryService` |
| 값 객체 | 개념 이름 그대로 | `Money`, `Weekdays` |
| 정책 | 규칙 이름 + `Policy` | `DiscountPolicy`, `OverlapPolicy` |
| 도메인 오류 | 상황 + `Error` | `OrderAlreadyPaidError` |
| 애그리거트 메서드 | 업무 동사 | `place`, `cancel`, `markPaid`, `completeIfDue` |
| 시각 필드 | `~At` 순간, `~LocalDate` 날짜, `~LocalTime` 시각 | `paidAt`, `deliveryLocalDate` |

## 시작할 때 체크리스트

1. `shared/domain/domain-error.ts`와 오류 필터부터. 오류 흐름이 먼저 서야 도메인 오류를 마음 놓고 던진다.
2. `TransactionRunner` 포트와 구현. 유스케이스 첫 줄이 `this.transactions.run(...)`이 되도록.
3. 컨텍스트 폴더를 만들고 **용어표**를 설계 문서에 쓴다. 코드 이름은 그 표를 따른다.
4. 애그리거트 하나당 `domain/<이름>/` 폴더. 값 객체 → 애그리거트 → 리포지토리 포트 → 오류 순으로.
5. `.dependency-cruiser.ts`에 층 규칙을 넣고 `verify` 스크립트에 연결. **첫 커밋 전에.**
6. `test/support/fake-ports.ts`를 만들고 포트가 늘 때마다 같이 고친다.

## 규모별 조정

| 상황 | 조정 |
|---|---|
| 컨텍스트가 하나뿐 | `src/<context>/` 대신 `src/` 바로 아래 네 층. 나중에 쪼개기 쉽게 애그리거트 폴더는 유지 |
| 애그리거트가 하나뿐 | `domain/<이름>/` 없이 `domain/` 바로 아래 |
| 조회가 복잡해짐 | `application/<이름>/queries/`로 조회 서비스를 모으고, 필요하면 읽기 전용 매핑을 인프라에 추가 (CQRS 입구) |
| 컨텍스트 간 통신이 늘어남 | 포트 호출 → 도메인 이벤트 + 핸들러로. `domain/<이름>/events/`, `application/<이름>/handlers/` |
| 외부 API 호출 | `domain/`에 포트(`PaymentGateway`), `infrastructure/external/`에 구현 |
