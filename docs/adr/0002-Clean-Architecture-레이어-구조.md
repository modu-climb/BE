# ADR-0002 Clean Architecture(UseCase) 레이어 구조

- 상태: 승인
- 날짜: 2026-10-13

## 배경

Notion 아키텍처 문서에는 헥사고날 구조로 적혀 있었다. 팀에는 이미 다른 프로젝트에서 검증한 Clean Architecture(UseCase 중심) 레이어 구조가 있다.

## 결정

- 도메인별로 `application`(usecase, dto, port) / `domain`(entity, repository, service) / `ui` / `infra` 레이어를 둔다.
- 의존 방향은 `ui → application → domain ← infra`이다.
- 영속성이 아닌 외부 연동(작업 큐, 스토리지)의 인터페이스는 `application/port/`에, 구현체는 `infra/`에 둔다.
- 엔티티의 JPA 매핑 어노테이션만 도메인 레이어의 프레임워크 의존으로 허용한다.
- 세부 규칙은 [코드 컨벤션](../CODE_CONVENTION.md)을 따른다.

## 결과

- 좋아지는 점
  - 팀이 익숙한 구조라 학습 비용이 없다.
  - 저장소 인터페이스는 domain에, 구현은 infra에 두므로 "도메인은 기술을 모른다"는 원칙은 헥사고날과 같게 지켜진다.
- 감수할 점
  - 헥사고날보다 규칙이 덜 엄격해 경계가 흐려질 수 있다 → M1에서 ArchUnit으로 의존 방향을 CI에서 검사한다.
