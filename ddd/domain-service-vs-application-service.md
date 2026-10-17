# 도메인 서비스와 애플리케이션 서비스

"이 로직을 어디에 둘까"에서 가장 많이 헷갈렸던 부분이다. 과제에서 정리된 기준을 적는다.

## 세 가지 자리

| 자리 | 무엇을 두나 | 아는 것 | 과제의 예 |
|---|---|---|---|
| **애그리거트/값 객체** | 자기 상태만으로 판단할 수 있는 규칙 | 자기 자신 | `Routine.changeSchedule`, `FastingInterval.overlaps` |
| **도메인 서비스 (정책)** | 여러 애그리거트·값을 놓고 판단하는 규칙. 상태 없음 | 도메인 객체들 | `OverlapPolicy.findConflict` |
| **애플리케이션 서비스 (유스케이스)** | 순서, 트랜잭션, 포트 호출, 입출력 변환 | 도메인 + 포트 | `CreateRoutineUseCase`, `RoutineOverlapService` |

판단 순서는 **위에서 아래로**. 애그리거트에 넣을 수 있으면 거기, 안 되면 도메인 서비스, 그래도 안 되면(DB를 읽어야 하면) 유스케이스.

## 도메인 서비스: 상태 없는 순수 규칙

```ts
// domain/scheduling/overlap-policy.ts
export class OverlapPolicy {
  static readonly HORIZON_DAYS = 400;

  static findConflict(candidates: readonly FastingInterval[], others: readonly FastingInterval[]): OverlapConflict | null {
    const sorted = [...candidates].sort((a, b) => a.startAt.getTime() - b.startAt.getTime());
    for (let i = 1; i < sorted.length; i += 1) {          // 새 일정끼리의 겹침
      if (sorted[i - 1].overlaps(sorted[i])) return { candidate: sorted[i], other: sorted[i - 1] };
    }
    for (const candidate of sorted) {                       // 기존 구간과의 겹침
      const other = others.find((o) => candidate.overlaps(o));
      if (other) return { candidate, other };
    }
    return null;
  }
}
```

특징:

- **입력을 받아 결과를 돌려줄 뿐** 저장소를 모른다. 그래서 `domain/`에 있고 DB 없이 테스트된다.
- 인스턴스 상태가 없어 정적 메서드로 두었다. DI가 필요 없다.
- 이름을 `~Service` 대신 `~Policy`로 지었다. "규칙"임이 더 잘 드러난다.

## 애플리케이션 서비스: 조립

```ts
// application/routine/routine-overlap.service.ts
@Injectable()
export class RoutineOverlapService {
  constructor(private readonly routines: RoutineRepository, private readonly sessions: FastingSessionRepository) {}

  async ensureNoOverlap(routine: Routine, now: Date, startedLocalDates?: ReadonlySet<LocalDate>): Promise<void> {
    const until = OverlapPolicy.horizonEnd(now);
    const others = (await this.routines.findByUser(routine.userId)).filter((o) => o.id !== routine.id);   // 1. 읽고
    const inProgress = await this.sessions.findInProgressByUser(routine.userId);
    const existing = [...inProgress.map((s) => s.interval()), ...others.flatMap((o) => o.upcomingIntervals(until))];
    const conflict = OverlapPolicy.findConflict(routine.upcomingIntervals(until, startedLocalDates), existing); // 2. 도메인에 판단 맡기고
    if (conflict) throw new RoutineOverlapError(conflict.candidate, conflict.other);                           // 3. 결과를 오류로
  }
}
```

같은 이름 "겹침 검사"가 **두 층에 나뉘어** 있다. 애플리케이션 서비스는 데이터를 모아 오고, 도메인 정책이 판정한다. 이 분리 덕에 판정 로직 테스트는 구간 배열만 넘기면 된다.

## 유스케이스의 모양

```ts
@Injectable()
export class CreateRoutineUseCase {
  constructor(routines, overlap, users, clock, transactions) {}

  execute(command: CreateRoutineCommand): Promise<RoutineResult> {
    return this.transactions.run(async () => {            // 트랜잭션 경계
      const now = this.clock.now();                         // 시각은 여기서 한 번 읽어 넘김
      const schedule = RoutineSchedule.of({...});           // 값 객체 생성 = 입력 검증
      await this.users.ensureExists(command.userId);        // 다른 컨텍스트 포트
      const existing = await this.routines.findByUser(command.userId);
      if (existing.length >= Routine.MAX_PER_USER) throw new RoutineLimitExceededError(Routine.MAX_PER_USER);
      const routine = Routine.register({...});              // 애그리거트 생성
      await this.overlap.ensureNoOverlap(routine, now);     // 애플리케이션 서비스
      await this.routines.add(routine);                     // 저장
      return toRoutineResult(routine);                      // 도메인 객체를 밖으로 내보내지 않음
    });
  }
}
```

유스케이스가 하는 일과 하지 않는 일:

- **한다**: 트랜잭션 열기, 시각 읽기, 포트 호출 순서 정하기, 도메인 객체 → 결과 타입 변환
- **하지 않는다**: `if (durationHours > 24)` 같은 규칙 판단, HTTP 상태 코드 결정, SQL

## "서비스" 접미사의 구분

과제에서 쓴 네이밍:

| 접미사 | 뜻 | 위치 |
|---|---|---|
| `~UseCase` | 쓰기 한 건. `execute(command)` 하나 | application |
| `~QueryService` | 조회 전용. 트랜잭션 없이 읽어서 결과 타입으로 | application |
| `~Service` | 여러 유스케이스가 공유하는 애플리케이션 로직 | application |
| `~Policy` | 상태 없는 도메인 규칙 | domain |

`SessionStartService`가 `~Service`인 이유: 작업자(시작 기록)와 변경 요청(밀린 시작 반영) **둘 다** "회차 하나 시작 + 알림 대기 + 저장"을 써야 해서 유스케이스 밖으로 뺐다.

## 배운 것

- **빈약한 도메인 모델(anemic model) 경고 신호**: 유스케이스에 `if`가 늘어나고 애그리거트가 getter/setter만 가지면 규칙이 잘못된 층에 있는 것이다. 과제 초반 `Routine`이 그랬다가 `scheduleNextAfter`를 안으로 옮기면서 유스케이스가 짧아졌다.
- **`now`는 인자로.** 도메인 서비스도 유스케이스도 `Clock`을 직접 읽지 않고 받는다. 작업자의 세 단계가 같은 `now`를 써야 한다는 요구가 이 규칙을 강제했다.
- 도메인 서비스가 리포지토리를 받는 설계(책에 종종 나옴)는 **쓰지 않았다.** 리포지토리를 받는 순간 DB 없이 테스트가 안 되고, 애플리케이션 서비스와 구분이 흐려진다.
