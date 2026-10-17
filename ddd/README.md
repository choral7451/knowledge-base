# DDD 정리

업무 규칙을 프레임워크·DB·HTTP와 분리된 **도메인 모델**에 모으고, 의존 방향을 **안쪽(도메인)으로만** 향하게 하는 설계 방법.

## 키워드

| 키워드 | 정의 |
|---|---|
| **Value Object**<br>값 객체 | 값이 같으면 같은 것. 불변. |
| **Entity**<br>엔티티 | ID로 구분. 상태가 변하는 생명주기가 있다. |
| **Aggregate**<br>애그리거트 | 함께 바뀌어야 하는 객체 묶음. 루트로만 접근. |
| **Invariant**<br>불변식 | 애그리거트가 항상 참으로 유지할 조건. |
| **Domain**<br>도메인 층 | 업무 규칙만 있는 가장 안쪽 층. |
| **Domain Service**<br>도메인 서비스 | 여러 객체를 놓고 판단하는 상태 없는 규칙. |
| **Application Service**<br>유스케이스 | 순서·트랜잭션·포트 호출을 조립하는 층. |
| **Repository**<br>리포지토리 | 애그리거트를 컬렉션처럼 다루는 인터페이스. |
| **Port / Adapter**<br>포트·어댑터 | 포트는 안쪽이 요구하는 계약, 어댑터는 그 구현. |
| **Bounded Context**<br>바운디드 컨텍스트 | 하나의 모델과 용어가 통하는 범위. |
| **Ubiquitous Language**<br>유비쿼터스 언어 | 명세·문서·코드가 같은 단어를 쓰는 것. |
| **Domain Error**<br>도메인 오류 | 업무 규칙 위반을 뜻하는 예외. |

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
