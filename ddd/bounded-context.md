# 바운디드 컨텍스트와 유비쿼터스 언어

## 바운디드 컨텍스트 정의

**하나의 도메인 모델과 용어가 일관되게 통하는 범위.** 같은 단어라도 컨텍스트가 다르면 다른 모델일 수 있다. (예: 쇼핑몰에서 "상품"은 카탈로그 컨텍스트와 재고 컨텍스트에서 다른 속성을 가진다)

전략적 설계의 단위이고, 코드에서는 보통 **최상위 폴더 하나 = 컨텍스트 하나**로 나타난다.

## 과제의 컨텍스트

```
src/
├── fasting/     단식 컨텍스트. 루틴, 회차, 알림, 작업자
├── user/        유저 컨텍스트. 유저 존재 확인만
└── shared/      컨텍스트와 무관한 기반 코드. 특정 컨텍스트를 import하지 않음
```

컨텍스트가 둘뿐인 작은 과제였지만, 나누면서 배운 점이 있다.

### user를 따로 뗀 이유

`users` 테이블은 과제가 제공한 것이고 읽기 전용이었다. `fasting` 안에 넣으면 "루틴 컨텍스트가 유저를 소유한다"는 잘못된 모델이 된다. 실제 서비스라면 유저는 인증·프로필 컨텍스트의 것이다.

### 컨텍스트 간 통신은 포트로

```ts
// user/application/user-directory.ts — fasting이 user에게 묻는 유일한 창구
export abstract class UserDirectory {
  abstract exists(userId: number): Promise<boolean>;
  async ensureExists(userId: number): Promise<void> {
    if (!(await this.exists(userId))) throw new UserNotFoundError(userId);
  }
}
```

- `fasting`은 `user`의 **포트(application)만** import한다. `user/domain`이나 `user/infrastructure`는 모듈 파일 외에는 못 본다. 구조 검사 규칙(`other-context-through-port-only`)으로 강제.
- 방향은 `fasting → user` 한쪽. `user`는 `fasting`을 모른다.
- 유저 **객체**를 넘기지 않고 `exists(userId)`로 **질문**만 한다. `fasting`에는 `User` 클래스가 없다. 필요한 건 "있는 유저인가"뿐이었다.

이것이 DDD의 **고객-공급자(Customer-Supplier)** 관계의 가장 단순한 형태다. 이벤트 버스·메시지 큐 없이 포트 하나로 충분했다.

### shared의 규칙

| 둘 수 있는 것 | 둘 수 없는 것 |
|---|---|
| `DomainError` 기반 클래스 | 특정 컨텍스트의 오류 |
| `TransactionRunner` 포트 | 특정 애그리거트의 리포지토리 |
| 오류 응답 필터, 검증 파이프 | 루틴 요청 타입 |

기준은 "이 코드가 fasting이라는 단어를 아는가". 알면 shared가 아니다.

## 유비쿼터스 언어 (Ubiquitous Language)

**도메인 전문가, 문서, 코드가 같은 단어를 쓰는 것.** 과제에서는 과제 명세의 용어를 코드 이름과 일대일로 맞췄다.

| 명세 용어 | 코드 | 설명 |
|---|---|---|
| 루틴 | `Routine` | 반복 단식 규칙 |
| 회차 | `FastingSession` | 루틴에 따라 실제 시작한 단식 한 번 |
| 작업자 | `FastingScheduler` | 예정 시각에 회차를 시작·종료하고 알림을 보내는 서버 안의 코드 |
| 알림 | `SessionNotification` | 회차의 시작·종료 때 발송기를 부르는 일 |
| 예정 | `Occurrence` | 루틴 규칙을 특정 날짜에 적용한 결과 |
| 다음 시작 | `nextStartAt` | 루틴마다 하나뿐인 다음 예정 |

### 지켰던 규칙

- **설계 문서에 용어표를 먼저 쓴다.** `docs/design.md` 맨 위가 용어표였다. 코드보다 문서를 먼저 고친다는 원칙과 함께.
- **시간 이름은 접미사로 종류를 구분.** `~At`은 UTC 순간, `~LocalDate`는 현지 날짜, `~Minute`/`~LocalTime`은 현지 시각. 같은 "시작"이라도 `scheduledStartAt`과 `startLocalDate`, `startMinute`은 다른 것이다. 이 구분이 없으면 시간대 버그가 난다.
- **메서드 이름은 업무 동사.** `markDeleted`(삭제 표시), `completeIfDue`(때가 되면 완료), `startDueSession`(예정이 된 회차 시작). `delete`, `setStatus`, `create`를 피했다.
- **오류 이름도 상황 이름.** `RoutineOverlapError`, `RoutineLimitExceededError`. HTTP 상태 코드는 이름에 안 들어간다.

### 용어가 흔들렸던 곳

- "시간대"의 코드 이름이 `timeZone`(도메인)과 `timezone`(API 필드, 오류 field)으로 갈렸다. 외부 계약(제공된 `/users` 응답)이 `timezone`이라 경계에서만 그 표기를 쓰기로 결정. **경계에서 이름이 바뀌는 지점은 문서에 적어 둔다.**
- "서비스"라는 단어가 너무 넓어서 `UseCase`, `QueryService`, `Service`, `Policy`로 쪼갰다. ([도메인 서비스와 애플리케이션 서비스](domain-service-vs-application-service.md))

## 배운 것

- 컨텍스트 경계는 **폴더 + import 규칙**으로 코드에 박아야 유지된다. 문서만으로는 금방 무너진다.
- 작은 프로젝트에서도 컨텍스트를 나누는 연습은 가치가 있었다. "fasting이 User를 몰라도 되나?"를 묻는 것만으로 모델이 단순해졌다.
- 다음에 컨텍스트가 셋 이상이면 **컨텍스트 맵**(어느 컨텍스트가 어느 컨텍스트에 어떤 관계로 의존하는지 그림)을 설계 문서에 넣을 것.
