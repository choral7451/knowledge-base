# DDD 정리

업무 규칙을 프레임워크·DB·HTTP와 분리된 **도메인 모델**에 모으고, 의존 방향을 **안쪽(도메인)으로만** 향하게 하는 설계 방법.

## 키워드

<table>
<thead><tr><th>키워드</th><th>정의</th><th>가까운 개념과의 차이</th></tr></thead>
<tbody>
<tr><td><b>Value Object</b><br>값 객체</td><td>값이 같으면 같은 것으로 보는 불변 객체. 식별자가 없다.</td><td>Entity와 달리 ID가 없다. 값이 같으면 같은 것이고, 바꾸는 대신 새로 만든다.</td></tr>
<tr><td colspan="3">

```ts
// 돈. 금액과 통화가 같으면 같은 돈이다. 바꾸지 않고 새로 만든다.
export class Money {
  private constructor(
    readonly amount: number,
    readonly currency: 'KRW' | 'USD',
  ) {}

  static of(amount: number, currency: 'KRW' | 'USD'): Money {
    if (!Number.isInteger(amount) || amount < 0) throw new Error(`금액은 0 이상의 정수여야 합니다: ${amount}`);
    return new Money(amount, currency);
  }

  static zero(currency: 'KRW' | 'USD'): Money {
    return new Money(0, currency);
  }

  plus(other: Money): Money {
    this.assertSameCurrency(other);
    return Money.of(this.amount + other.amount, this.currency);    // 새 객체. 검증도 다시
  }

  minus(other: Money): Money {
    this.assertSameCurrency(other);
    return Money.of(this.amount - other.amount, this.currency);    // 음수가 되면 of()가 거부
  }

  times(quantity: number): Money {
    return Money.of(this.amount * quantity, this.currency);
  }

  equals(other: Money): boolean {                                   // 값으로 비교
    return this.amount === other.amount && this.currency === other.currency;
  }

  private assertSameCurrency(other: Money): void {
    if (this.currency !== other.currency) throw new Error('통화가 다른 돈은 더할 수 없습니다.');
  }
}

// 사용
const price = Money.of(12_000, 'KRW');
const total = price.times(3).plus(Money.of(3_000, 'KRW'));   // 39,000원. price는 그대로
total.equals(Money.of(39_000, 'KRW'));                        // true
```

<code>private</code> 생성자 + 정적 팩토리 <code>of()</code>에서 검증<br>필드는 <code>readonly</code>, 연산은 새 객체를 돌려줌<br><code>equals()</code>로 값 비교. ID 없음

</td></tr>
<tr><td><b>Entity</b><br>엔티티</td><td>ID로 구분되는 객체. 속성이 바뀌어도 같은 것이며 생명주기가 있다.</td><td>Value Object와 달리 ID로 구분한다. 속성이 전부 같아도 ID가 다르면 다른 것.</td></tr>
<tr><td colspan="3">

```ts
// 상품. 이름·가격이 바뀌어도 id가 같으면 같은 상품이다.
export class Product {
  private constructor(
    readonly id: string,
    private _name: string,
    private _price: Money,
    private _status: 'ON_SALE' | 'DISCONTINUED',
  ) {}

  static register(id: string, name: string, price: Money): Product {
    if (name.trim() === '') throw new Error('상품 이름은 비울 수 없습니다.');
    return new Product(id, name, price, 'ON_SALE');
  }

  get name() { return this._name; }
  get price() { return this._price; }
  get status() { return this._status; }

  changePrice(price: Money): void {                 // 상태 변경은 메서드로만
    if (this._status === 'DISCONTINUED') throw new Error('판매 종료된 상품의 가격은 바꿀 수 없습니다.');
    this._price = price;
  }

  discontinue(): boolean {                          // 이미 그 상태면 false (멱등)
    if (this._status === 'DISCONTINUED') return false;
    this._status = 'DISCONTINUED';
    return true;
  }

  equals(other: Product): boolean {                 // id로만 비교
    return this.id === other.id;
  }
}

// 사용
const a = Product.register('P-1', '티셔츠', Money.of(12_000, 'KRW'));
a.changePrice(Money.of(9_900, 'KRW'));            // 가격이 바뀌어도
a.equals(Product.register('P-1', '티셔츠', Money.of(12_000, 'KRW')));   // true. 같은 id
```

<code>id</code>로 동일성 판단. 속성이 바뀌어도 같은 상품<br>상태 변경은 업무 동사 메서드로만(<code>changePrice</code>, <code>discontinue</code>). 필드 직접 대입 불가<br>생명주기가 있음: 등록 → 가격 변경 → 판매 종료

</td></tr>
<tr><td><b>Invariant</b><br>불변식</td><td>객체가 항상 지켜야 하는 조건. 예: 재고 ≥ 0, 주문 금액 = 항목 합계.</td><td>입력 검증은 들어올 때 한 번, 불변식은 바뀔 때마다 매번.</td></tr>
<tr><td colspan="3">

**"재고는 0 미만이 될 수 없다"**

```ts
decrease(quantity: number): void {
  if (this.stock - quantity < 0) throw new Error('재고가 부족합니다.');
  this.stock -= quantity;
}
```

**"주문 금액은 항상 항목 금액의 합이다"**

```ts
addLine(line: OrderLine): void {
  this.lines.push(line);
  this.totalAmount = this.lines.reduce((sum, l) => sum.plus(l.amount()), Money.zero('KRW'));  // 항목이 바뀔 때마다 다시 계산
}
```

**"결제가 끝난 주문은 항목을 바꿀 수 없다"**

```ts
addLine(line: OrderLine): void {
  if (this.status !== 'PENDING') throw new Error('결제된 주문은 변경할 수 없습니다.');
  ...
}
```

**"쿠폰은 한 번만 쓸 수 있다"**

```ts
use(now: Date): boolean {
  if (this.usedAt !== null) return false;   // 이미 썼으면 아무것도 안 함
  this.usedAt = now;
  return true;
}
```

공통점: 조건을 깨뜨릴 수 있는 **모든 메서드**가 같은 검사를 한다. 필드를 밖에서 직접 바꾸지 못하게 막아야 우회가 안 된다.

</td></tr>
<tr><td><b>Aggregate</b><br>애그리거트</td><td>함께 바뀌어야 일관성이 유지되는 엔티티·값 객체의 묶음. 트랜잭션 경계.</td><td>Module과 달리 패키지가 아니라 트랜잭션·일관성 단위. 다른 애그리거트는 ID로만 참조.</td></tr>
<tr><td><b>Aggregate Root</b><br>애그리거트 루트</td><td>애그리거트의 대표 엔티티. 바깥은 루트를 통해서만 접근하고, 다른 애그리거트는 루트 ID로만 참조한다.</td><td>일반 Entity와 달리 바깥에서 직접 접근할 수 있는 유일한 엔티티. 리포지토리도 루트에만 둔다.</td></tr>
<tr><td colspan="3">

```
Order 애그리거트
├── Order                 ★ Aggregate Root (엔티티). 바깥은 이것만 잡는다
│   ├── id, status
│   ├── totalAmount       Money (Value Object)
│   └── customerId        → Customer 애그리거트를 ID로만 참조
├── OrderLine ×N          Entity (자식). 루트를 거쳐서만 접근
│   ├── id, quantity
│   ├── unitPrice         Money (Value Object). 주문 당시 가격 복사
│   └── productId         → Product 애그리거트를 ID로만 참조
└── AppliedDiscount ×N    Entity (자식)
    ├── id, amount        Money (Value Object)
    └── couponId          → Coupon 애그리거트를 ID로만 참조
```

**안에 들어가는 것**: 루트 엔티티, 자식 엔티티, 값 객체<br>
**밖에 두는 것**: 다른 애그리거트(`Product`, `Customer`, `Coupon`). 객체 대신 **ID만** 가진다<br>
**저장·조회·트랜잭션** 단위는 이 묶음 전체

같은 애그리거트에 넣을지 판단 기준 (위일수록 중요)

1. **같은 불변식을 지켜야 하는가** — 둘 사이에 항상 참이어야 하는 규칙이 있으면 같은 애그리거트. 예: 주문 총액 = 항목 합계
2. **한 트랜잭션에서 함께 바뀌어야 하는가** — 따로 커밋해도 괜찮으면 분리
3. **생명주기를 같이하는가** — 루트가 사라지면 같이 사라져야 하면 같은 애그리거트. 예: 주문 없는 주문 항목은 없다
4. **바깥에서 그것만 따로 조회하는가** — 단독으로 찾을 일이 있으면 분리. 예: 리뷰는 상품 없이도 조회된다
5. **묶으면 너무 커지거나 동시 수정이 잦은가** — 그렇다면 분리. 예: 상품에 리뷰 수천 개를 넣으면 리뷰마다 상품을 잠근다

판단이 갈리면 기본값은 **분리**. 1번이 확실할 때만 합친다.

</td></tr>
<tr><td><b>Factory</b><br>팩토리</td><td>복잡한 애그리거트·값 객체를 유효한 상태로 생성하는 책임. 정적 메서드나 별도 클래스.</td><td>생성자와 달리 복잡한 조립·검증을 맡는다. 유효하지 않은 객체가 태어나지 않게 한다.</td></tr>
<tr><td><b>Repository</b><br>리포지토리</td><td>애그리거트를 컬렉션처럼 저장·조회하는 인터페이스. 루트마다 하나.</td><td>DAO와 달리 테이블이 아니라 애그리거트 단위로 다룬다. 인터페이스는 도메인에, 구현은 인프라에.</td></tr>
<tr><td><b>Domain Service</b><br>도메인 서비스</td><td>특정 엔티티에 속하지 않는, 여러 객체를 놓고 판단하는 상태 없는 업무 규칙.</td><td>Application Service와 달리 업무 규칙 자체를 판단한다. 상태·트랜잭션·DB 접근이 없다.</td></tr>
<tr><td><b>Domain Event</b><br>도메인 이벤트</td><td>도메인에서 일어난 의미 있는 사건. 애그리거트 간·컨텍스트 간 느슨한 연결에 쓴다.</td><td>Command와 달리 이미 일어난 사실. 과거형 이름(<code>OrderPlaced</code>)이고 거부할 수 없다.</td></tr>
<tr><td><b>Application Service</b><br>애플리케이션 서비스</td><td>유스케이스 하나를 조립하는 층. 트랜잭션·순서·포트 호출을 맡고 규칙 판단은 도메인에 맡긴다.</td><td>Domain Service와 달리 규칙을 판단하지 않는다. 순서·트랜잭션·포트 호출만 조립.</td></tr>
<tr><td><b>Module</b><br>모듈</td><td>관련된 도메인 객체를 묶는 패키지 단위. 유비쿼터스 언어의 이름을 따른다.</td><td>Bounded Context와 달리 모델 경계가 아니라 같은 모델 안의 패키지 묶음.</td></tr>
<tr><td><b>Bounded Context</b><br>바운디드 컨텍스트</td><td>하나의 모델과 용어가 일관되게 통하는 경계. 코드에서는 최상위 모듈 하나.</td><td>Subdomain과 달리 업무가 아니라 <b>모델과 코드</b>의 경계. 보통 서브도메인 하나에 컨텍스트 하나.</td></tr>
<tr><td><b>Anti-Corruption Layer</b><br>부패 방지 계층</td><td>다른 컨텍스트·외부 시스템의 모델이 내 모델을 오염시키지 않도록 번역하는 층.</td><td>단순 어댑터와 달리 상대 모델을 내 모델의 언어로 <b>번역</b>한다. 상대 용어가 안으로 새지 않게.</td></tr>
<tr><td><b>Context Map</b><br>컨텍스트 맵</td><td>컨텍스트들이 서로 어떤 관계(상류/하류, 공유 커널, 부패 방지 계층 등)로 연결되는지 그린 그림.</td><td>Bounded Context가 경계 하나라면, 컨텍스트 맵은 경계들 <b>사이의 관계</b>를 그린다.</td></tr>
<tr><td><b>Ubiquitous Language</b><br>유비쿼터스 언어</td><td>도메인 전문가·문서·코드가 같은 단어를 쓰는 것. 컨텍스트마다 하나.</td><td>용어집과 달리 코드 이름까지 포함한다. 컨텍스트가 다르면 같은 단어도 다른 언어.</td></tr>
<tr><td><b>Subdomain</b><br>서브도메인</td><td>도메인을 쪼갠 단위. 핵심(Core), 지원(Supporting), 범용(Generic)으로 나눈다.</td><td>Bounded Context와 달리 업무(문제 공간)의 구분. 코드가 아니라 비즈니스를 나눈다.</td></tr>
<tr><td><b>Domain</b><br>도메인</td><td>소프트웨어가 해결하려는 업무 영역과 그 지식.</td><td>Subdomain과 달리 전체 업무 영역. 서브도메인은 이것을 쪼갠 조각.</td></tr>
</tbody>
</table>

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
    │   │   ├── order.ts                 ★ Aggregate Root
    │   │   ├── order-line.ts            Entity (자식). 루트를 통해서만 접근
    │   │   ├── order.repository.ts      Repository (인터페이스)
    │   │   ├── order.errors.ts
    │   │   ├── order-placed.event.ts    Domain Event
    │   │   └── order.spec.ts
    │   └── pricing/
    │       ├── money.ts                 Value Object
    │       └── discount-policy.ts       Domain Service
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
