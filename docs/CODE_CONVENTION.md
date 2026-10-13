# 코드 컨벤션 (Clean Architecture)

기존 프로젝트에서 검증된 Clean Architecture 레이어 구조(UseCase 중심)를 이 프로젝트에 적용한다. 이슈·브랜치·커밋·PR 규칙은 [개발 컨벤션](CONVENTION.md)을 따른다.

## 공통 원칙

- **의존 방향:** `ui → application → domain ← infra`. 바깥 레이어만 안쪽을 안다. `infra`는 `domain`(과 `application`)에 정의된 인터페이스를 구현할 뿐, 안쪽 레이어가 `infra`를 import하지 않는다.
- **도메인은 기술을 모른다.** `domain/service`와 엔티티의 비즈니스 메서드에는 Spring, Redis, S3 같은 기술이 들어가지 않는다. 엔티티의 JPA 매핑 어노테이션(`@Entity`, `@Column` 등)만 예외로 허용한다.
- **도메인은 AI 모델을 모른다.** 도메인에는 "깊이 맵", "홀드 마스크" 같은 결과 개념만 두고, 모델 이름은 워커에만 둔다(NFR-04).
- **트랜잭션 경계는 UseCase 단위다.**

## 패키지 구조

루트 패키지는 `com.moduclimb`이다.

### 도메인 1개당

```
domain/<feature>/
├── application/
│   ├── dto/
│   │   ├── request/            # XXXRequest
│   │   └── response/           # XXXResponse
│   ├── port/                   # 외부 연동 인터페이스 (작업 큐, 스토리지 등)
│   └── usecase/                # XXXUseCase
├── domain/
│   ├── entity/                 # 엔티티, 값 객체, enum
│   ├── repository/             # XXXRepository (인터페이스)
│   └── service/                # XXXService (도메인 서비스)
├── ui/                         # XXXController
└── infra/                      # 리포지토리 구현체, 외부 연동 어댑터
```

`application/port/`는 이 프로젝트에서 추가한 폴더다. 영속성이 아닌 외부 연동(Redis Streams 작업 큐, 오브젝트 스토리지 presigned URL 등)은 비즈니스 규칙이 아니라 시나리오의 일부라서 `domain/repository`가 아닌 여기에 인터페이스를 두고, 구현체는 `infra/`에 둔다.

### 도메인 목록 (초안)

ERD 기준 초안이며 M0 도메인 설계에서 확정한다.

| feature | 담당 테이블 | 주요 책임 |
| --- | --- | --- |
| `member` | climber, body_profile | 회원, 인증, 신체 프로필, 동의 |
| `video` | video | 업로드 URL, 영상 메타데이터, 보관 기한 |
| `wallscene` | wall_scene, hold | 벽 기준 이미지, 홀드 |
| `problem` | problem, problem_hold | 문제 확정, 문제-홀드 소속 |
| `attempt` | attempt | 시도, 신체 스냅샷, 시도 비교 |
| `analysis` | analysis_job, analysis_artifact | 분석 작업 상태 머신, 워커 메시지 송수신, 진행 상황 |
| `feedback` | feedback_report, feedback_item | 리포트, 피드백 문장 |
| `coachreview` | coach_review | 코치 검토 |

### 전역 공통

```
global/
├── annotation/          # CurrentUser 등 커스텀 어노테이션
├── common/              # BaseEntity(생성·수정 시각), BaseResponse
├── config/              # SecurityConfig, RedisConfig, SwaggerConfig ...
├── exception/           # BaseCode, ExceptionAdvice, RestApiException
├── idempotency/         # Idempotency-Key 처리
├── interceptor/
├── resolver/            # CurrentUserArgumentResolver 등
├── security/            # JwtAuthenticationFilter, TokenProvider ...
├── swagger/
└── util/
```

## 레이어 책임

### UI (`ui/`)

- HTTP 입출력 어댑터. 파싱, 검증, 상태 코드 매핑만 하고 얇게 유지한다.
- SSE 진행 상황 스트림도 여기서 연다.
- 네이밍: `XXXController`

### Application (`application/`)

- 시나리오 오케스트레이션(여러 저장소·외부 연동을 넘나드는 흐름), 트랜잭션 경계, 권한 확인, 비즈니스 규칙 위임.
- 네이밍: `XXXUseCase`, `XXXRequest`, `XXXResponse`, 포트는 역할 이름(`AnalysisJobQueue`, `ObjectStorage`)

### Domain (`domain/`)

- 비즈니스 규칙의 중심. 상태 전이 규칙(예: AnalysisJob 상태 머신)은 엔티티 메서드 안에 두고, 허용되지 않은 전이는 예외로 막는다.
- 네이밍: `XXXRepository`(인터페이스), `XXXService`
- 엔티티는 `global/common/BaseEntity`를 상속해 생성·수정 시각을 JPA Auditing으로 기록한다.
- **시각은 전부 `Instant`(UTC)로 저장한다.** 보관 기한, 7일 무응답 만료, presigned URL 만료처럼 기한 계산이 많아서 서버·클라이언트 타임존에 의존하는 `LocalDateTime`을 쓰지 않는다. 표시용 타임존 변환은 응답 직전에만 한다.

### Infra (`infra/`)

- 리포지토리 구현체, 외부 연동(Redis Streams, 오브젝트 스토리지, 결제 등).
- 네이밍: `XXXRepository`(구현체), `XXXAdapter`, `XXXPublisher`, `XXXConsumer`

## 도메인 간 의존

- 한 도메인의 `application`(UseCase)이 다른 도메인의 `domain/service` 인터페이스를 쓰는 것은 허용한다. 예: `attempt`의 UseCase가 `problem`의 홀드 확정 여부를 확인한다.
- 의존 방향은 항상 **쓰는 도메인 → 제공하는 도메인** 단방향이고, 순환 의존은 금지한다.
- 다른 도메인의 엔티티·리포지토리를 직접 쓰지 않는다.
- 이 규칙은 M1에서 아키텍처 테스트(ArchUnit)로 CI에서 검사한다.

## 코드 스타일

- **`var` 금지, 타입은 항상 명시한다.** 로컬 변수든 스트림 체인 중간값이든 `var x = ...`로 두지 않고 `String x = ...`처럼 타입을 적는다. 리뷰·디버깅 때 타입을 추론하는 비용을 없애고, 오버로드가 많은 API(Spring Security, Redis 등)에서 의도와 다른 오버로드가 선택돼도 드러나지 않는 실수를 막는다.
- **import는 파일 상단에 모으고, 코드 본문에 완전정규화 클래스명(FQCN)을 쓰지 않는다.** `java.time.Duration validity` 대신 `import java.time.Duration;` 후 `Duration validity`로 쓴다. Javadoc의 `{@link}`·`{@code}` 안에서 다른 패키지 클래스를 가리킬 때만 예외다.

## 테스트 컨벤션

- **단위 테스트:** `domain`(엔티티 상태 전이, 도메인 서비스)이 1순위, `application/usecase`는 시나리오 테스트.
- **계약 테스트:** `infra`의 리포지토리 구현체, 워커 메시지 직렬화(JSON Schema 기준).
- **통합 테스트:** 핵심 흐름만 Testcontainers(PostgreSQL, Redis, MinIO)로.
