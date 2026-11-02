# ADR (Architecture Decision Record)

되돌리기 어렵거나 여러 곳에 영향을 주는 결정을 하나씩 기록한다. 결정을 바꿀 때는 기존 ADR을 고치지 않고 새 ADR을 쓴 뒤 기존 ADR의 상태를 "대체됨"으로 바꾼다.

| 번호 | 제목 | 상태 |
| --- | --- | --- |
| [0001](0001-BE-AI-분리.md) | BE(Spring Boot)와 AI(Python 워커) 분리 | 승인 |
| [0002](0002-Clean-Architecture-레이어-구조.md) | Clean Architecture(UseCase) 레이어 구조 | 승인 |
| [0003](0003-인증-소셜-로그인-JWT.md) | 인증: 카카오 소셜 로그인 + JWT | 제안 |
| [0004](0004-분석-단계와-메시지-계약.md) | 분석 실행 단계와 워커 메시지 계약 보완 | 제안 |
| [0005](0005-ID-UUIDv7.md) | ID는 UUIDv7 | 제안 |

## 형식

```markdown
# ADR-NNNN 제목

- 상태: 제안 | 승인 | 대체됨(ADR-XXXX)
- 날짜: YYYY-MM-DD

## 배경
## 결정
## 결과 (좋아지는 점, 감수할 점)
```
