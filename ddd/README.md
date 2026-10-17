# DDD 정리

업무 규칙을 프레임워크·DB·HTTP와 분리된 **도메인 모델**에 모으고, 의존 방향을 **안쪽(도메인)으로만** 향하게 하는 설계 방법.

## 키워드

| 키워드 | 정의 |
|---|---|
| **Value Object**<br>값 객체 | 값이 같으면 같은 것으로 보는 불변 객체. 식별자가 없다. |
| **Entity**<br>엔티티 | ID로 구분되는 객체. 속성이 바뀌어도 같은 것이며 생명주기가 있다. |
| **Invariant**<br>불변식 | 애그리거트가 항상 참으로 유지해야 하는 규칙. 루트가 지킨다. |
| **Aggregate Root**<br>애그리거트 루트 | 애그리거트의 대표 엔티티. 바깥은 루트를 통해서만 접근하고, 다른 애그리거트는 루트 ID로만 참조한다. |
| **Aggregate**<br>애그리거트 | 함께 바뀌어야 일관성이 유지되는 엔티티·값 객체의 묶음. 트랜잭션 경계. |
| **Factory**<br>팩토리 | 복잡한 애그리거트·값 객체를 유효한 상태로 생성하는 책임. 정적 메서드나 별도 클래스. |
| **Repository**<br>리포지토리 | 애그리거트를 컬렉션처럼 저장·조회하는 인터페이스. 루트마다 하나. |
| **Domain Service**<br>도메인 서비스 | 특정 엔티티에 속하지 않는, 여러 객체를 놓고 판단하는 상태 없는 업무 규칙. |
| **Domain Event**<br>도메인 이벤트 | 도메인에서 일어난 의미 있는 사건. 애그리거트 간·컨텍스트 간 느슨한 연결에 쓴다. |
| **Application Service**<br>애플리케이션 서비스 | 유스케이스 하나를 조립하는 층. 트랜잭션·순서·포트 호출을 맡고 규칙 판단은 도메인에 맡긴다. |
| **Module**<br>모듈 | 관련된 도메인 객체를 묶는 패키지 단위. 유비쿼터스 언어의 이름을 따른다. |
| **Bounded Context**<br>바운디드 컨텍스트 | 하나의 모델과 용어가 일관되게 통하는 경계. 코드에서는 최상위 모듈 하나. |
| **Anti-Corruption Layer**<br>부패 방지 계층 | 다른 컨텍스트·외부 시스템의 모델이 내 모델을 오염시키지 않도록 번역하는 층. |
| **Context Map**<br>컨텍스트 맵 | 컨텍스트들이 서로 어떤 관계(상류/하류, 공유 커널, 부패 방지 계층 등)로 연결되는지 그린 그림. |
| **Ubiquitous Language**<br>유비쿼터스 언어 | 도메인 전문가·문서·코드가 같은 단어를 쓰는 것. 컨텍스트마다 하나. |
| **Subdomain**<br>서브도메인 | 도메인을 쪼갠 단위. 핵심(Core), 지원(Supporting), 범용(Generic)으로 나눈다. |
| **Domain**<br>도메인 | 소프트웨어가 해결하려는 업무 영역과 그 지식. |

## 계층과 의존 방향

```
presentation → application → domain ← infrastructure
```

| 층 | 들어 있는 것 | 금지 |
|---|---|---|
| domain | 애그리거트, 값 객체, 도메인 서비스, 도메인 이벤트, 리포지토리 인터페이스 | 프레임워크, ORM, HTTP |
| application | 애플리케이션 서비스(유스케이스), 조회 서비스, Command/Result 타입 | ORM, 컨트롤러, 요청·응답 타입 |
| infrastructure | 리포지토리 구현, ORM 매핑, 외부 시스템 어댑터, 마이그레이션 | 유스케이스, 컨트롤러 |
| presentation | 컨트롤러, 요청·응답 타입, 스케줄러·컨슈머 같은 진입점 | 리포지토리 직접 호출, 인프라 |

- 데이터 흐름: `Request → Command → [Application Service] → Result → Response`. 도메인 객체는 층 밖으로 내보내지 않는다.
- 의존 규칙은 린터(dependency-cruiser 등)로 자동 검사한다.

## 디렉토리 샘플

```
src/
├── shared/                      컨텍스트 무관 기반
│   ├── domain/domain-error.ts
│   ├── application/transaction-runner.ts
│   ├── infrastructure/persistence/   ORM 설정, 트랜잭션 구현, migrations/
│   └── presentation/                 오류 필터, 검증 파이프, 공통 응답
├── customer/                    다른 컨텍스트. application의 인터페이스만 공개
│   ├── customer.module.ts
│   ├── domain/
│   ├── application/customer-directory.ts
│   └── infrastructure/
└── order/                       주문 컨텍스트
    ├── order.module.ts          인터페이스 ↔ 구현 바인딩 (유일한 곳)
    ├── domain/
    │   ├── order/               애그리거트 하나 = 폴더 하나
    │   │   ├── order.ts                 애그리거트 루트
    │   │   ├── order-line.ts            자식 엔티티
    │   │   ├── order.repository.ts      리포지토리 인터페이스
    │   │   ├── order.errors.ts
    │   │   ├── order-placed.event.ts    도메인 이벤트
    │   │   └── order.spec.ts
    │   └── pricing/
    │       ├── money.ts                 값 객체
    │       └── discount-policy.ts       도메인 서비스
    ├── application/order/
    │   ├── place-order.use-case.ts      애플리케이션 서비스. Command 타입은 이 안에
    │   ├── order-query.service.ts
    │   └── order.result.ts
    ├── infrastructure/persistence/
    │   ├── order.schema.ts              ORM 매핑
    │   └── orm-order.repository.ts      리포지토리 구현
    └── presentation/
        ├── http/
        │   ├── order.controller.ts
        │   ├── order.request.ts
        │   └── order.response.ts
        └── scheduler/
test/
└── support/fake-ports.ts        인터페이스의 가짜 구현
.dependency-cruiser.ts
```

접미사: 없음(애그리거트·값 객체·인터페이스·도메인 서비스) / `.use-case` / `.service` / `.result` / `.repository` / `.schema` / `.event` / `.controller` / `.request` / `.response` / `.module` / `.spec`
