# 리포지토리와 포트

## 리포지토리 정의

**애그리거트를 컬렉션처럼 다루게 해 주는 인터페이스.** 호출하는 쪽은 "루틴 저장소에서 유저의 루틴을 꺼낸다"고만 생각하고, SQL·ORM·테이블은 모른다.

DDD에서 리포지토리는 **애그리거트 루트마다 하나**다. 값 객체나 자식 엔티티에는 리포지토리를 두지 않는다.

## 포트와 어댑터

리포지토리는 **포트(port)**의 한 종류다. 포트는 "안쪽(도메인·애플리케이션)이 바깥쪽에 요구하는 계약"이고, 어댑터는 그 계약의 구현이다.

```
domain/routine/routine.repository.ts          ← 포트 (추상 클래스). 도메인 층에 있음
infrastructure/persistence/mikro-orm-routine.repository.ts   ← 어댑터. 인프라 층에 있음
fasting.module.ts                              ← 둘을 묶는 유일한 곳
```

의존 방향이 뒤집힌다. 인프라가 도메인의 인터페이스를 **구현**하므로 `infrastructure → domain`이고, 도메인은 인프라를 전혀 모른다. 이것이 의존성 역전(DIP)이다.

## 과제의 포트 목록

| 포트 | 위치 | 구현 | 역할 |
|---|---|---|---|
| `RoutineRepository` | fasting/domain | `MikroOrmRoutineRepository` | 루틴 저장·조회 |
| `FastingSessionRepository` | fasting/domain | `MikroOrm...` | 회차 저장·조회 |
| `SessionNotificationRepository` | fasting/domain | `MikroOrm...` | 알림 저장·조회 |
| `TransactionRunner` | shared/application | `MikroOrmTransactionRunner` | 트랜잭션 경계 |
| `UserDirectory` | user/application | `MikroOrmUserDirectory` | 다른 컨텍스트가 유저 존재 확인 |
| `Clock` | support (제공 코드) | `SystemClock`, 테스트용 `ManualClock` | 현재 시각 |
| `NotificationSender` | support (제공 코드) | 제공된 가짜 발송기 | 외부 발송 |

리포지토리만 포트가 아니다. **시간, 트랜잭션, 외부 호출**도 전부 포트다. 테스트에서 바꿔 끼울 것은 모두 포트로 뺀다.

## 코드 모양: 포트

```ts
// domain/routine/routine.repository.ts
export abstract class RoutineRepository {
  /** 새 루틴을 저장하고 id를 채웁니다. */
  abstract add(routine: Routine): Promise<void>;
  /** 그 유저의 삭제하지 않은 루틴을 만든 순서로. 비활성도 포함. */
  abstract findByUser(userId: number): Promise<Routine[]>;
  /** 다른 유저의 루틴과 삭제한 루틴은 null. */
  abstract findOwned(userId: number, routineId: string): Promise<Routine | null>;
  /** 작업자용. 예정이 된 루틴 id를 오래된 순으로 limit개. */
  abstract findDueIds(now: Date, limit: number): Promise<string[]>;
  /** 행을 잠그고 읽는다. */
  abstract findForUpdate(routineId: string): Promise<Routine | null>;
}
```

- TS `interface`가 아니라 **추상 클래스**인 이유: NestJS DI 토큰으로 쓰려면 런타임에 남는 값이 필요하다. 인터페이스는 컴파일 후 사라진다.
- 메서드 이름은 **업무 질문**이다. `findWhere(criteria)` 같은 범용 메서드 대신 `findDueIds`, `findOwned`처럼 쓰임새마다 하나. 구현이 인덱스를 제대로 타게 하기도 쉽다.
- 주석에 **계약**(정렬, 포함 범위, null 조건)을 적는다. 가짜 구현도 이 계약을 지켜야 한다.
- `update`가 없다. 변경은 ORM 변경 감지가 커밋 때 저장한다. ([애그리거트](aggregate.md) 참고)

## 코드 모양: 어댑터

```ts
// infrastructure/persistence/mikro-orm-routine.repository.ts
@Injectable()
export class MikroOrmRoutineRepository extends RoutineRepository {
  constructor(private readonly em: EntityManager) { super(); }

  async add(routine: Routine): Promise<void> { await this.em.persist(routine).flush(); }

  findByUser(userId: number): Promise<Routine[]> {
    return this.em.find(Routine, { userId, deletedAt: null }, { orderBy: { id: 'asc' } });
  }

  findForUpdate(routineId: string): Promise<Routine | null> {
    return this.em.findOne(Routine, { id: routineId }, { lockMode: LockMode.PESSIMISTIC_WRITE });
  }
}
```

ORM 특유의 것(`EntityManager`, `LockMode`, `flush`)은 **전부 이 파일 안**에 있다.

## 매핑도 인프라에

도메인 클래스에 `@Entity()`, `@Property()`를 붙이지 않았다. 대신 인프라의 `EntitySchema`가 "이 클래스를 이 테이블에 이렇게 매핑한다"를 적는다.

```ts
// infrastructure/persistence/routine.schema.ts
export const RoutineSchema = new EntitySchema<Routine>({
  class: Routine,
  tableName: 'routines',
  properties: {
    id: { type: new BigIntType('string'), primary: true, autoincrement: true },
    timeZone: { type: 'string', fieldName: 'timezone', length: 64 },
    nextLocalDate: { type: 'date', nullable: true },
    ...
  },
});
```

이렇게 하면 도메인 모델과 영속 모델을 **따로 두지 않으면서도** 도메인이 ORM을 import하지 않는다. 과제 규모에서 가장 비용 대비 효과가 좋았던 선택.

## 포트와 구현을 묶는 곳

```ts
// fasting.module.ts — 이 컨텍스트에서 포트와 구현을 묶는 유일한 곳
providers: [
  { provide: RoutineRepository, useClass: MikroOrmRoutineRepository },
  { provide: FastingSessionRepository, useClass: MikroOrmFastingSessionRepository },
  ...
]
```

테스트는 같은 자리에 가짜를 꽂는다 (`test/support/fake-ports.ts`). 유스케이스 코드는 한 글자도 안 바뀐다.

## 배운 것

- **트랜잭션도 포트다.** `TransactionRunner.run(work)` 하나로 유스케이스가 경계를 잡는다. MikroORM의 `em.transactional`은 어댑터 안에만 있다.
- **요청 밖(작업자)에서 읽기만 해도 트랜잭션 안에서.** ORM이 요청/트랜잭션 컨텍스트 밖의 조회를 거부했다. 포트를 넓히지 않고 호출 규칙으로 해결.
- **ID만 돌려주는 메서드**(`findDueIds`)와 **잠그고 읽는 메서드**(`findForUpdate`)를 나눈 것: 대상 목록은 가볍게 뽑고, 실제 처리 때 한 건씩 잠가 "아직 대상인지" 다시 확인한다. 동시성 처리 패턴으로 기억해 둘 것.
- 리포지토리 포트에 메서드를 더하면 **가짜 구현도 함께** 고쳐야 한다. 포트가 비대해지는 비용이 여기서 보인다.
