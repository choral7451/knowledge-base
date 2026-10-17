# 값 객체 (Value Object)

## 정의

**식별자가 없고, 담고 있는 값이 같으면 같은 것으로 보는 객체.** 불변이며, 자신의 유효성을 스스로 지킨다.

`Date`나 `Money`가 전형적인 예다. "2026-10-17"이라는 날짜 두 개는 서로 다른 인스턴스여도 같은 날짜다.

## 왜 쓰나

원시 타입(`number`, `string`)을 그대로 쓰면 규칙이 사용하는 쪽마다 흩어진다.

```ts
// 값 객체 없이: 검증이 컨트롤러, 서비스, 테스트에 반복됨
if (durationHours < 1 || durationHours > 24) throw ...
```

값 객체로 감싸면 **생성 시점에 한 번만 검증**하고, 그 뒤로는 "이 타입이면 유효하다"를 믿을 수 있다.

## 과제에서 만든 값 객체

| 값 객체 | 감싼 원시 값 | 지키는 규칙 |
|---|---|---|
| `Weekdays` | 요일 비트마스크 (1~127) | 1개 이상, 중복 불가 |
| `LocalTime` | 하루 중 분 (0~1439) | `HH:mm` 형식, 범위 |
| `LocalDate` | `YYYY-MM-DD` 문자열 | 날짜 연산(`addDays`, 요일 계산) |
| `FastingInterval` | 시작·끝 `Date` | 끝이 시작보다 늦어야 함, 겹침 판정 |
| `Occurrence` | 현지 날짜 + 구간 | 루틴 규칙을 특정 날짜에 적용한 결과 |
| `RoutineSchedule` | 요일·시각·지속시간·시간대 묶음 | 지속 시간 1~24, IANA 시간대 |

## 코드 모양

```ts
export class Weekdays {
  private constructor(private readonly mask: number) {}   // 1. 생성자는 private

  static of(codes: readonly string[]): Weekdays {          // 2. 정적 팩토리에서 검증
    if (codes.length === 0) throw new InvalidRoutineError('days', '반복 요일은 하나 이상이어야 합니다.');
    let mask = 0;
    for (const code of codes) {
      const index = DAY_CODES.indexOf(code as DayCode);
      if (index < 0) throw new InvalidRoutineError('days', `알 수 없는 요일입니다: ${code}`);
      const bit = 1 << index;
      if (mask & bit) throw new InvalidRoutineError('days', `같은 요일을 두 번 지정할 수 없습니다: ${code}`);
      mask |= bit;
    }
    return new Weekdays(mask);
  }

  static fromMask(mask: number): Weekdays { ... }          // 3. DB 값에서 복원하는 팩토리도 따로

  includes(weekdayIndex: number): boolean { ... }          // 4. 도메인 질문에 답하는 메서드
  equals(other: Weekdays): boolean { return this.mask === other.mask; }  // 5. 값 비교
}
```

체크리스트로 쓰면:

1. 생성자는 `private`, 들어오는 길은 정적 팩토리뿐
2. 팩토리에서 검증하고 실패하면 **도메인 오류**를 던진다
3. 입력용(`of`, `parse`)과 복원용(`fromMask`, `ofMinute`) 팩토리를 구분
4. 필드는 `readonly`, setter 없음. 바꾸려면 새 객체
5. `equals`를 직접 구현. 참조 비교에 기대지 않음

## 값 객체끼리의 조합

`RoutineSchedule`은 `Weekdays`, `LocalTime`을 품는 **복합 값 객체**다. 이 조합 덕분에 "반복 규칙이 바뀌었나?"를 한 줄로 묻는다.

```ts
changeSchedule(schedule: RoutineSchedule, ...): boolean {
  if (this.schedule().equals(schedule)) return false;  // 같은 설정이면 아무것도 안 함
  ...
}
```

## 배운 것·주의할 점

- **값 객체는 ORM이 몰라도 된다.** 과제에서는 `Routine` 애그리거트의 필드는 테이블 컬럼과 같은 원시 값(`daysMask: number`)으로 두고, `schedule()` 메서드가 호출 시점에 값 객체를 조립했다. 이중 매핑 없이 값 객체의 이점만 가져가는 절충이다.
- **복원 실패는 입력 오류가 아니다.** DB에 저장된 값으로 값 객체를 만들다 실패하면 400이 아니라 500이어야 한다. `Routine.schedule()`이 값 객체의 `InvalidRoutineError`를 일반 `Error`로 바꿔 던진 이유다.
- **외부 라이브러리는 한 파일에 가둔다.** 시간대 계산용 luxon은 `zoned-time.ts` 한 파일만 import하도록 구조 검사 규칙으로 막았다. 값 객체가 라이브러리 타입을 노출하지 않는다.
