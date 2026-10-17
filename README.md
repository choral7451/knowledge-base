# knowledge-base

공부한 내용을 주제별로 정리하는 저장소입니다.

## 구조

```
knowledge-base/
├── <주제>/
│   └── <개념>.md     # 한 파일에 한 개념
└── README.md
```

## 작성 규칙

- 폴더는 주제(언어·프레임워크·분야) 단위로 나눕니다.
- 파일 하나에 개념 하나를 담고, 파일명은 소문자 kebab-case로 씁니다.
- 새 글을 추가하면 아래 목록에 링크를 더합니다.

## 목록

### DDD (도메인 주도 설계)

인아웃 백엔드 과제(NestJS + MikroORM)를 DDD로 구현하며 정리한 내용. [개요와 읽는 순서](ddd/overview.md)부터.

- [값 객체](ddd/value-object.md) — 불변, 값으로 동일성, 생성 시 검증
- [엔티티](ddd/entity.md) — 식별자, 상태 전이는 메서드로만, `now`는 인자로
- [애그리거트](ddd/aggregate.md) — 일관성 경계, ID로만 참조, 불변식은 루트가 지킴
- [도메인 서비스와 애플리케이션 서비스](ddd/domain-service-vs-application-service.md) — 로직을 어느 층에 둘지 판단 기준
- [리포지토리와 포트](ddd/repository-and-ports.md) — 포트는 도메인에, 구현은 인프라에, 매핑도 인프라에
- [바운디드 컨텍스트와 유비쿼터스 언어](ddd/bounded-context.md) — 컨텍스트 나누기, 포트로 통신, 용어표
- [계층 구조와 의존 방향](ddd/layered-architecture.md) — 네 층, 데이터 타입 네 가지, dependency-cruiser로 검사
- [디렉토리 구조 샘플](ddd/directory-structure-sample.md) — 새 프로젝트용 뼈대, 접미사·네이밍 규칙, 체크리스트
