# 계층 구조와 의존 방향

## 네 층

컨텍스트 하나를 네 층으로 나눴다. 헥사고날(포트와 어댑터)과 전통적 레이어드의 절충이다.

```
presentation → application → domain ← infrastructure
```

| 층 | 역할 | 들어 있는 것 | 의존해도 되는 것 | 금지 |
|---|---|---|---|---|
| `domain` | 업무 규칙 | 애그리거트, 값 객체, 도메인 오류, 리포지토리 **포트**, 정책 | 같은 컨텍스트의 도메인, `shared/domain` | NestJS, ORM, HTTP, `Clock` |
| `application` | 흐름 조립 | 유스케이스, 조회 서비스, 애플리케이션 서비스, 커맨드·결과 타입 | 도메인, 포트들(`TransactionRunner`, `Clock`, `UserDirectory`) | ORM, 컨트롤러, 요청·응답 타입 |
| `infrastructure` | 포트 구현 | 테이블 매핑(`EntitySchema`), 리포지토리 구현, 트랜잭션 구현, 마이그레이션 | 도메인의 포트와 엔티티, ORM | 유스케이스, 컨트롤러 |
| `presentation` | 바깥 세계와의 접점 | 컨트롤러, 요청·응답 타입, API 문서, **작업자 타이머** | 유스케이스, 조회 서비스 | 리포지토리 직접 호출, 인프라 |

**도메인이 가장 안쪽**이고 아무것도 의존하지 않는다. 인프라는 도메인의 포트를 구현하니 화살표가 도메인을 **향한다.**

## 자주 틀리는 배치

| 이것은 | 여기에 | 이유 |
|---|---|---|
| 리포지토리 **인터페이스** | domain | 도메인이 요구하는 계약이므로 |
| 리포지토리 **구현** | infrastructure | ORM을 아니까 |
| 테이블 매핑 | infrastructure | 도메인 클래스에 데코레이터 안 붙임 |
| 트랜잭션 포트 | shared/application | 유스케이스가 경계를 잡는 도구 |
| 스케줄러 타이머 | presentation | HTTP와 같은 "진입점". `tick()` 하나로 유스케이스를 부를 뿐 |
| 환경 변수 읽기 | 모듈 파일 | 유스케이스는 `process.env`를 모른다. 설정 객체로 주입 |
| 요청 모양 검증 (`class-validator`) | presentation | 모양만. 값의 규칙은 도메인 값 객체가 |
| HTTP 상태 코드 결정 | presentation (오류 필터) | 도메인 오류는 `kind`만 갖는다 |

## 층을 건너는 데이터: 네 가지 타입

```
요청(Request) → 커맨드(Command) → [유스케이스] → 결과(Result) → 응답(Response)
```

| 타입 | 위치 | 특징 |
|---|---|---|
| `CreateRoutineRequest` | presentation | 검증 데코레이터 붙은 클래스. `toCommand(userId)` 메서드 |
| `CreateRoutineCommand` | application (유스케이스 파일 안) | 데코레이터 없는 순수 타입 |
| `RoutineResult` | application | 순수 데이터. 순간은 `Date` |
| `RoutineResponse` | presentation | JSON 모양. `from(result)`로 변환. 순간을 문자열로 |

**유스케이스는 도메인 객체를 반환하지 않는다.** 결과 타입으로 바꿔서 돌려준다. 컨트롤러가 애그리거트 메서드를 부르는 길을 원천 차단하는 효과.

컨트롤러는 "받고, 커맨드로 바꾸고, 유스케이스 부르고, 응답으로 바꿔 돌려준다" 네 줄이다.

## 오류의 흐름

```ts
// shared/domain/domain-error.ts
export type DomainErrorKind = 'INVALID' | 'NOT_FOUND' | 'CONFLICT';   // HTTP 상태가 아니라 위반의 종류

export abstract class DomainError extends Error {
  protected constructor(readonly kind: DomainErrorKind, readonly code: string, message: string, readonly field?: string) {...}
}
```

- 도메인은 `kind`(종류)와 `code`(식별 코드)만 안다. `INVALID → 400`, `NOT_FOUND → 404`, `CONFLICT → 409`로 바꾸는 것은 presentation의 오류 필터.
- 값이 틀린 오류는 `field`를 담는다. 필터가 응답 `details`로 내보내 요청 모양 오류와 같은 모양이 된다.
- 다른 유저의 루틴은 403이 아니라 `ROUTINE_NOT_FOUND`(404). "남의 것이 존재한다"를 알려 주지 않는 도메인 결정이 오류 타입에 들어 있다.

## 의존 방향을 자동으로 검사

문서에 적은 규칙은 깨진다. **dependency-cruiser**로 `.dependency-cruiser.ts`에 규칙을 옮기고 `pnpm run deps:check`를 `verify`에 넣었다.

```ts
{
  name: 'domain-knows-no-framework',
  severity: 'error',
  from: { path: '^src/[^/]+/domain/' },
  to: { path: 'node_modules/(@nestjs|@mikro-orm|class-validator|class-transformer|mysql2|rxjs)/' },
},
{
  name: 'presentation-calls-no-repository',
  from: { path: '^src/[^/]+/presentation/' },
  to: { path: '\\.repository\\.ts$|node_modules/(@mikro-orm|mysql2)/' },
},
{
  name: 'luxon-only-in-zoned-time',
  from: { path: '^(src|support)/', pathNot: '^src/fasting/domain/scheduling/zoned-time\\.ts$' },
  to: { path: 'node_modules/(luxon|@types/luxon)/' },
},
```

규칙으로 잡은 것들:

- 층 간 금지 방향 전부 (도메인→바깥, 애플리케이션→인프라/프레젠테이션, 인프라→유스케이스, 프레젠테이션→인프라)
- 도메인·애플리케이션의 프레임워크 import 금지
- 특정 라이브러리를 **한 파일**에만 허용 (luxon)
- 검증·문서 라이브러리는 프레젠테이션에만
- 컨텍스트 간 방향 (`user → fasting` 금지, `fasting`은 `user`의 포트만)
- `shared`는 어떤 컨텍스트도 import 금지
- 순환 의존 금지
- 운영 코드가 테스트·스크립트 import 금지

`tsPreCompilationDeps: true`로 **타입만 가져오는 import도** 의존으로 센다.

## 테스트가 층을 따라간다

| 대상 | DB | 위치 | 방법 |
|---|---|---|---|
| 값 객체, 애그리거트, 정책 | 없음 | 파일 옆 `.spec.ts` | 그냥 `new`해서 호출 |
| 유스케이스 흐름 | 없음 | 파일 옆 `.spec.ts` | `test/support/fake-ports.ts`의 가짜 포트 주입 |
| 리포지토리 구현, API, 작업자 전체 | 있음 | `test/` | 실제 앱을 띄우고 요청. 시간은 `ManualClock` |

도메인이 프레임워크를 모르니 **도메인 테스트가 가장 많고 가장 빠르다.** 서머타임 경계 같은 까다로운 케이스를 전부 여기서 잡았다.

## 배운 것

- **구조 규칙은 CI에 넣어야 산다.** 문서(`CLAUDE.md`)와 검사 설정을 같은 표현으로 유지하고, 둘 중 하나를 고치면 다른 쪽도 고치는 규칙을 두었다.
- **타이머도 프레젠테이션.** "진입점은 전부 프레젠테이션"이라고 생각하면 스케줄러, 메시지 컨슈머, CLI가 어디 가야 할지 헷갈리지 않는다.
- **`now`를 인자로 넘기는 규칙**이 층 분리의 리트머스였다. 도메인이 `Clock`을 import하고 싶어지는 순간이 곧 층을 잘못 긋고 있는 순간이었다.
- 유스케이스가 트랜잭션을 잡으면서도 ORM을 모르려면 **트랜잭션 포트**가 필요하다. 처음엔 과해 보였지만 작업자(요청 밖)에서 조회가 거부되는 문제를 포트 호출 규칙만으로 풀 수 있었다.
