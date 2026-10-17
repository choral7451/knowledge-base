# 엔티티 (Entity)

## 정의

**식별자(ID)로 구분되는 객체.** 속성이 전부 바뀌어도 ID가 같으면 같은 것이고, 속성이 전부 같아도 ID가 다르면 다른 것이다. 시간에 따라 상태가 변하는 **생명주기**를 가진다.

값 객체와의 대비가 핵심이다.

| | 엔티티 | 값 객체 |
|---|---|---|
| 동일성 | ID | 값 전체 |
| 변경 | 상태가 바뀜 (메서드로만) | 불변, 새로 만듦 |
| 예 | `Routine`, `FastingSession` | `Weekdays`, `LocalTime` |

## 과제에서의 엔티티

세 애그리거트 루트가 전부 엔티티였다. 각각 상태 전이가 있다.

```
Routine:            등록 → (설정 변경)* → 비활성 ⇄ 활성 → 삭제 표시
FastingSession:     IN_PROGRESS → COMPLETED
SessionNotification: PENDING → SENT | SKIPPED
```

## 코드 모양: 상태는 메서드로만 바꾼다

```ts
export class FastingSession {
  id!: string;                       // DB가 부여. 저장 전에는 비어 있음
  status!: SessionStatus;
  endedAt!: Date | null;
  ...

  static start(params: {...}): FastingSession {   // 생성 = 첫 상태 전이
    const session = new FastingSession();
    session.status = 'IN_PROGRESS';
    session.endedAt = null;
    ...
    return session;
  }

  completeIfDue(now: Date): boolean {              // 상태 전이는 의도를 드러내는 이름으로
    if (this.status !== 'IN_PROGRESS') return false;
    if (this.scheduledEndAt.getTime() > now.getTime()) return false;
    this.status = 'COMPLETED';
    this.endedAt = this.scheduledEndAt;            // now가 아니라 목표 시각. 늦게 처리해도 원래 시각이 남음
    return true;
  }
}
```

규칙으로 적으면:

- 필드는 public이지만 **직접 대입하지 않는다.** 바꾸는 길은 메서드뿐. (TS에서 private setter를 강제하기 어려워 주석과 리뷰로 지킴)
- 메서드 이름은 CRUD(`setStatus`)가 아니라 **업무 동작**(`completeIfDue`, `activate`, `markDeleted`)
- 이미 그 상태면 예외 대신 `false`를 돌려주는 **멱등** 설계. 작업자가 같은 건을 두 번 잡아도 안전
- 현재 시각은 `now`를 **인자로 받는다.** 엔티티가 `new Date()`를 부르지 않아야 테스트에서 시간을 옮길 수 있다

## 생성은 정적 팩토리로

`new Routine()` 뒤에 필드를 하나씩 채우는 코드는 유스케이스마다 반복되고, 누락이 생긴다.
`Routine.register(...)`, `FastingSession.start(...)`, `SessionNotification.pendingStart(session)`처럼 **생성 의도가 이름에 드러나는 팩토리**를 두면 "유효한 상태로만 태어난다"를 보장한다.

ORM이 복원할 때는 빈 생성자를 쓰므로 생성자를 private으로 막지는 않았다. 대신 팩토리를 쓰는 것이 관례다.

## ID에 대해

- 과제에서는 DB auto-increment를 그대로 썼다. 그래서 `add(routine)` 전에는 `id`가 비어 있고, 알림이 회차 ID를 참조하려면 **회차를 먼저 저장**해야 했다 (`SessionStartService.startDue` 참고).
- 이 순서 의존이 거슬리면 애플리케이션에서 UUID/ULID를 미리 발급하는 방식이 대안이다. 다음 프로젝트에서는 이쪽을 고려할 것.
- BIGINT는 JS `number` 정밀도를 넘을 수 있어 ID 타입을 `string`으로 두었다.

## 배운 것

- 엔티티에 **ORM 데코레이터를 붙이지 않았다.** 매핑은 인프라의 `EntitySchema`에 두고 도메인 클래스는 순수 TS로 유지. 덕분에 도메인 단위 테스트가 DB 없이 돈다.
- 계산 값은 getter 대신 **메서드**로 두었다. MikroORM이 getter를 컬럼으로 매핑하려 들기 때문. (`schedule()`, `interval()`, `startTime()`)
- "진행 중인가?"처럼 시각에 따라 답이 다른 질문은 `isInProgressAt(now)`처럼 **시각을 받는 질의 메서드**로.
