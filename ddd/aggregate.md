# 애그리거트 (Aggregate)

## 정의

**함께 바뀌어야 일관성이 유지되는 엔티티·값 객체의 묶음.** 바깥에서는 묶음의 대표인 **애그리거트 루트**를 통해서만 접근하고, 저장·조회도 루트 단위로 한다.

"트랜잭션 하나로 보호해야 하는 범위가 어디까지인가"를 정하는 도구다.

## 과제의 애그리거트 셋

| 애그리거트 루트 | 품고 있는 것 | 불변식(invariant) |
|---|---|---|
| `Routine` | 반복 규칙(`RoutineSchedule`), 다음 예정(`nextStartAt`, `nextLocalDate`) | 활성·미삭제일 때만 다음 예정이 있다. 설정이 바뀌면 다음 예정도 같이 바뀐다 |
| `FastingSession` | 시작 당시 설정 복사본, 구간, 상태 | 완료일 때만 `endedAt`이 있다. 구간의 끝이 시작보다 늦다 |
| `SessionNotification` | 종류, 예정 순간, 처리 상태 | `PENDING`일 때만 `processedAt`이 null |

세 개 모두 **루트 하나 = 테이블 하나**로 작은 애그리거트였다. 자식 엔티티 컬렉션을 품는 큰 애그리거트는 없었다.

## 핵심 규칙 1: 애그리거트끼리는 ID로만 참조

```ts
export class FastingSession {
  routineId!: string;   // Routine 객체가 아니라 id
  userId!: number;
}

export class SessionNotification {
  sessionId!: string;   // FastingSession 객체가 아니라 id
}
```

ORM의 관계 매핑(`@ManyToOne`)을 쓰지 않았다. 이유는:

- 루틴을 불러올 때 회차 수백 건이 딸려 오는 것을 막는다
- 한 트랜잭션이 건드리는 범위가 루트 하나로 명확해진다
- 회차는 **시작 당시 루틴 설정을 복사**해 갖는다. 루틴이 나중에 바뀌어도 회차는 안 바뀐다는 규칙이 참조가 아닌 복사로 자연스럽게 표현된다

다른 애그리거트의 데이터가 필요하면 유스케이스가 리포지토리로 따로 불러 **인자로 넘긴다**:

```ts
// 알림이 "회차가 아직 진행 중인가"를 알아야 할 때
decide(now: Date, session: FastingSession): NotificationDecision {
  if (session.id !== this.sessionId) throw new Error(...);   // 맞는 회차인지 방어
  ...
}
```

## 핵심 규칙 2: 불변식은 루트가 지킨다

`Routine`에서 "다음 예정"은 네 가지 상태 전이에서 모두 다시 계산돼야 한다. 이걸 호출하는 쪽에 맡기면 반드시 한 군데 빠진다. 그래서 **private 메서드 하나**로 모으고 모든 전이가 그걸 부른다.

```ts
register(...)        { ...; routine.scheduleNextAfter(now); }
changeSchedule(...)  { ...; this.scheduleNextAfter(now, startedLocalDates); }
activate(...)        { ...; this.scheduleNextAfter(now, startedLocalDates); }
deactivate(...)      { ...; this.scheduleNextAfter(now); }   // 비활성이라 null이 됨
markDeleted(...)     { ...; this.scheduleNextAfter(now); }
startDueSession(...) { ...; this.scheduleNextAfter(session.scheduledStartAt, new Set([date])); }

private scheduleNextAfter(instant: Date, startedLocalDates?: ReadonlySet<LocalDate>): void {
  const next = this.isActive && !this.deletedAt
    ? this.schedule().nextOccurrenceAfter(instant, startedLocalDates)
    : null;
  this.nextStartAt = next?.startAt ?? null;
  this.nextLocalDate = next?.startLocalDate ?? null;
}
```

DB CHECK 제약(`next_start_at`은 활성·미삭제일 때만 값)은 **마지막 안전망**이고, 1차 방어는 애그리거트다.

## 핵심 규칙 3: 애그리거트가 다른 애그리거트를 "만들" 수는 있다

`Routine.startDueSession(now)`는 `FastingSession`을 **반환**한다. 루틴이 회차를 소유하지는 않지만, "예정 시각이 됐으니 회차를 시작한다"는 판단과 다음 예정 전진이 **한 메서드 안에서 원자적으로** 일어나야 하기 때문이다. 저장은 호출한 유스케이스가 한다.

```ts
startDueSession(now: Date): FastingSession | null {
  if (!this.nextStartAt || !this.nextLocalDate) return null;
  if (this.nextStartAt.getTime() > now.getTime()) return null;
  const session = FastingSession.start({ ..., scheduledStartAt: this.nextStartAt, now });
  this.scheduleNextAfter(session.scheduledStartAt, new Set([session.startLocalDate]));
  return session;
}
```

## 트랜잭션 경계 = 애그리거트 경계

작업자의 "시작 기록"은 루틴 1건 + 회차 1건 + 알림 1건을 **건별 트랜잭션**으로 커밋한다. 묶음 처리를 하지 않은 이유는 한 건의 실패가 다른 건을 막지 않게 하기 위해서다. 애그리거트가 작으니 트랜잭션도 짧고, 행 잠금(`findForUpdate`)도 루트 한 행만 잡는다.

## 배운 것·판단 기준

- **애그리거트는 작을수록 좋다.** "같이 안 바뀌면 같은 애그리거트가 아니다"가 기준. 회차를 루틴의 자식으로 넣고 싶은 유혹이 있었지만, 회차는 루틴이 삭제돼도 남아야 하므로 분리가 맞았다.
- **수정은 리포지토리 메서드가 아니라 변경 감지로.** 불러온 애그리거트를 메서드로 바꾸면 트랜잭션 커밋 때 ORM이 알아서 저장한다. `update(routine)` 같은 메서드를 두지 않았다. 새로 만든 것만 `add`.
- **정적 상수도 루트에.** `Routine.MAX_PER_USER = 10`, `SessionNotification.MAX_DELAY_MS`처럼 애그리거트에 관한 규칙 수치는 그 클래스에 둔다.
