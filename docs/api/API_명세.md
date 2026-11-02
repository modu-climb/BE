# API 명세

> 🔌 기능 명세서 7장의 엔드포인트에 인증·회원 API와 요청·응답 형식, 오류 코드를 더한 명세다. 실제 OpenAPI 문서는 M1에서 springdoc로 코드에서 생성하고, 이 문서는 구현 전 합의용이다. WBS 0.6의 산출물이다.

관련 문서: [기능 명세서](../02_기능_명세서.md) · [애그리거트](../domain/03_애그리거트.md) · [ADR-0003 인증](../adr/0003-인증-소셜-로그인-JWT.md) · [ADR-0004 메시지 계약](../adr/0004-분석-단계와-메시지-계약.md)

## 공통 규칙

### 요청

- 경로는 `/v1`로 시작한다.
- 인증이 필요한 요청은 `Authorization: Bearer {accessToken}` 헤더를 보낸다. 인증이 필요 없는 것은 `POST /v1/auth/kakao`, `POST /v1/auth/refresh`뿐이다.
- **모든 쓰기 요청(POST, PUT, DELETE)은 `Idempotency-Key` 헤더를 받는다.**
  - 값은 클라이언트가 만든 UUID다. 같은 회원이 같은 키로 다시 보내면 처음 응답을 그대로 돌려준다(24시간 보관).
  - 같은 키로 다른 본문을 보내면 `422 IDEMPOTENCY_KEY_REUSED`다.
  - MVP에서는 헤더가 없어도 처리하되, 업로드 완료·시도 생성·홀드 확정처럼 중복되면 안 되는 요청은 필수로 한다.
- 시각은 ISO-8601 UTC 문자열(`2026-10-13T09:30:00Z`)이다.
- ID는 UUIDv7 문자열이다.

### 응답

모든 응답은 같은 형식으로 감싼다(`global/common/BaseResponse`).

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공했습니다.",
  "result": { }
}
```

```json
{
  "isSuccess": false,
  "code": "JOB_INVALID_STATE",
  "message": "현재 작업 상태에서는 홀드를 선택할 수 없습니다.",
  "result": null
}
```

### 목록

- 커서 기반이다. 요청은 `?cursor=&size=`(기본 20, 최대 50), 응답은 `{ "items": [...], "nextCursor": "..." }`이며 마지막 페이지면 `nextCursor`가 `null`이다.

### 스토리지 URL

- 이미지·마스크·오버레이처럼 스토리지에 있는 파일은 **짧은 만료(5분)의 presigned GET URL**로 준다. 스토리지 키는 응답에 노출하지 않는다.

## 인증·회원

| 메서드 | 경로 | 용도 | 관련 |
| --- | --- | --- | --- |
| POST | /v1/auth/kakao | 카카오 인가 코드로 로그인(처음이면 가입) | ADR-0003 |
| POST | /v1/auth/refresh | access token 재발급(refresh 쿠키 사용) | ADR-0003 |
| POST | /v1/auth/logout | 로그아웃(refresh token 폐기) | ADR-0003 |
| GET | /v1/me | 내 정보 | |
| DELETE | /v1/me | 회원 탈퇴 | NFR-08 |
| PUT | /v1/me/consents | 분석·학습 활용 동의 저장 | NFR-08 |
| GET | /v1/me/body-profile | 현재 신체 프로필 | FR-01 |
| PUT | /v1/me/body-profile | 신체 프로필 등록·수정(새 기록 추가) | FR-01 |

```http
POST /v1/auth/kakao
{ "authorizationCode": "abc...", "redirectUri": "https://moduclimb.app/oauth/kakao" }

200 OK
Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict; Path=/v1/auth; Max-Age=1209600
{ "result": {
    "accessToken": "eyJ...", "expiresIn": 1800,
    "climber": { "id": "0192...", "nickname": "클라이머", "role": "CLIMBER", "isNewClimber": true,
                 "hasAnalysisConsent": false, "hasBodyProfile": false } } }
```

```http
PUT /v1/me/consents
{ "analysis": true, "training": false }

PUT /v1/me/body-profile
{ "heightCm": 172, "wingspanCm": 176 }
→ { "result": { "heightCm": 172, "wingspanCm": 176, "measuredAt": "2026-10-13T09:30:00Z" } }
```

## 영상·시도

| 메서드 | 경로 | 용도 | 관련 |
| --- | --- | --- | --- |
| POST | /v1/videos/upload-url | 업로드 URL 발급 | FR-02 |
| POST | /v1/videos/{videoId}/complete | 업로드 완료 알림 | FR-02 |
| POST | /v1/attempts | 시도 생성 + 분석 요청 | FR-02 |
| GET | /v1/attempts | 분석 이력(문제별·기간별) | FR-15 |
| GET | /v1/attempts/{attemptId} | 시도 상세(작업 상태 요약 포함) | FR-15 |

```http
POST /v1/videos/upload-url
{ "contentType": "video/mp4", "sizeBytes": 48211234 }

200 OK
{ "result": { "videoId": "0192...", "uploadUrl": "https://...", "method": "PUT",
              "requiredHeaders": { "Content-Type": "video/mp4" }, "expiresAt": "2026-10-13T09:45:00Z" } }
```

```http
POST /v1/attempts
{ "videoId": "0192...", "problemId": null }

202 Accepted
{ "result": { "attemptId": "0192...", "jobId": "0192...", "state": "REQUESTED" } }
```

- `problemId`를 보내면 같은 문제의 재시도로 처리한다. 촬영 위치가 다르다고 판정되면 일반 흐름(홀드 선택)으로 간다.
- 분석 동의가 없거나 신체 프로필이 없으면 `403 CONSENT_REQUIRED`, `409 BODY_PROFILE_REQUIRED`다.

```http
GET /v1/attempts?problemId=0192...&from=2026-10-01&to=2026-10-31&cursor=&size=20
→ { "result": { "items": [
      { "attemptId": "0192...", "problemId": "0192...", "problemName": "초록 V3", "attemptNo": 2,
        "createdAt": "2026-10-13T09:30:00Z", "jobState": "COMPLETED", "failureFrame": 842 } ],
    "nextCursor": null } }
```

## 분석 작업

| 메서드 | 경로 | 용도 | 관련 |
| --- | --- | --- | --- |
| GET | /v1/jobs/{jobId} | 작업 상태와 다음 행동 | FR-12 |
| GET | /v1/jobs/{jobId}/events | 진행 상황 SSE | FR-12 |
| GET | /v1/jobs/{jobId}/holds | 탐지된 홀드와 벽 기준 이미지 | FR-04 |
| POST | /v1/jobs/{jobId}/hold-selection | 기준 홀드 선택 | FR-05 |
| POST | /v1/jobs/{jobId}/holds | 누락 홀드 추가(탭 좌표) | FR-07 |
| POST | /v1/jobs/{jobId}/hold-confirmation | 홀드 추가·제외, 시작·종료 지정, 문제 확정 | FR-06, FR-07 |

```http
GET /v1/jobs/{jobId}
→ { "result": {
      "jobId": "0192...", "attemptId": "0192...", "state": "AWAITING_HOLD_SELECTION",
      "phase": "PREPROCESS", "progress": 100,
      "nextAction": "SELECT_REFERENCE_HOLD",
      "waitingUntil": "2026-10-20T09:30:00Z",
      "rejectReason": null, "errorCode": null } }
```

`nextAction`: `NONE`, `SELECT_REFERENCE_HOLD`, `CONFIRM_HOLDS`, `RESHOOT`(반려됨).

**SSE (`text/event-stream`)**

| event | data | 언제 |
| --- | --- | --- |
| `progress` | `{ "phase": "ANALYZE", "progress": 42, "step": "P4" }` | 워커 진행률 보고 |
| `state-changed` | `{ "state": "AWAITING_CONFIRMATION", "nextAction": "CONFIRM_HOLDS" }` | 상태 전이 |
| `hold-added` | `{ "requestId": "...", "hold": { ... } }` | 누락 홀드 추가 결과 |
| `completed` | `{ "attemptId": "0192..." }` | 리포트 준비 완료 |

- 연결 직후 현재 상태를 `state-changed`로 한 번 보낸다. 끊기면 클라이언트가 다시 연결하고, 놓친 이벤트는 `GET /v1/jobs/{jobId}`로 맞춘다.

```http
GET /v1/jobs/{jobId}/holds
→ { "result": {
      "wallSceneId": "0192...", "cleanPlateUrl": "https://...",
      "holds": [ { "holdId": "0192...", "label": "h03", "maskUrl": "https://...",
                   "bbox": { "x": 120, "y": 340, "w": 48, "h": 52 }, "source": "AUTO",
                   "membershipProb": 0.42, "status": "PENDING", "role": null } ] } }
```

- `membershipProb`, `status`, `role`은 문제 소속 추정 후에만 채워진다.

```http
POST /v1/jobs/{jobId}/hold-selection
{ "holdId": "0192..." }
→ 200 { "result": { "state": "INFERRING_PROBLEM" } }

POST /v1/jobs/{jobId}/holds
{ "x": 412, "y": 233 }
→ 202 { "result": { "requestId": "0192..." } }   // 결과는 SSE hold-added

POST /v1/jobs/{jobId}/hold-confirmation
{ "problemName": "초록 V3",
  "holds": [ { "holdId": "0192...", "status": "CONFIRMED", "role": "START" },
             { "holdId": "0192...", "status": "EXCLUDED", "role": null } ] }
→ 200 { "result": { "problemId": "0192...", "state": "ANALYZING" } }
```

- 보내지 않은 홀드는 추정 결과를 그대로 쓴다.
- `START`·`FINISH`가 없거나 `PENDING` 홀드에 역할을 주면 `400 PROBLEM_INVALID_HOLDS`다.

## 결과·비교·코치

| 메서드 | 경로 | 용도 | 관련 |
| --- | --- | --- | --- |
| GET | /v1/attempts/{attemptId}/report | 피드백 리포트 | FR-10, FR-11 |
| GET | /v1/attempts/{attemptId}/overlay | 오버레이 재생 데이터 | FR-08 |
| GET | /v1/problems | 내 문제 목록 | FR-15 |
| GET | /v1/problems/{problemId}/comparison?a=&b= | 두 시도 비교 | FR-13 |
| GET | /v1/attempts/{attemptId}/coach-reviews | 코치 검토 목록 | FR-14 |
| POST | /v1/attempts/{attemptId}/coach-reviews | 코치 검토 등록 | FR-14 |
| PUT | /v1/attempts/{attemptId}/coach-reviews/{reviewId} | 코치 검토 수정 | FR-14 |

```http
GET /v1/attempts/{attemptId}/report
→ { "result": {
      "attemptId": "0192...", "pipelineVersion": "0.3.0",
      "failurePoint": { "frame": 842, "timeSec": 28.1, "confidence": 0.71 },
      "metrics": [ { "id": "hip_wall_distance", "label": "골반-벽 거리", "unit": "cm",
                     "estimate": { "min": 18, "max": 26 }, "confidence": 0.64 } ],
      "items": [ { "type": "CAUSE", "rank": 1, "text": "...", "evidenceMetricIds": ["hip_wall_distance"], "confidence": 0.6 },
                 { "type": "NEXT_MOVE", "rank": 1, "text": "...", "evidenceMetricIds": ["reach_ratio"], "confidence": 0.55 } ] } }
```

- 추정 수치는 범위(`min`, `max`)로 준다(NFR-06).
- 오버레이는 포즈 시계열·접촉 이벤트 파일의 presigned URL과 영상 URL을 준다. 큰 시계열을 JSON 응답에 직접 담지 않는다.
- 비교는 같은 Problem·같은 WallScene의 시도만 허용하고, 아니면 `400 COMPARISON_NOT_ALLOWED`다.

```http
POST /v1/attempts/{attemptId}/coach-reviews
{ "verdict": "PARTIAL", "comment": "발 위치 제안은 맞지만..." }
→ 201 { "result": { "reviewId": "0192..." } }
```

## 오류 코드

| 코드 | HTTP | 뜻 |
| --- | --- | --- |
| `COMMON_400` | 400 | 요청 형식 오류(검증 실패 필드 목록을 `result`에 담는다) |
| `COMMON_404` | 404 | 자원이 없거나 **본인 것이 아님**(존재 여부를 숨기려고 403 대신 404) |
| `COMMON_500` | 500 | 서버 오류 |
| `AUTH_UNAUTHORIZED` | 401 | access token 없음·만료·위조 |
| `AUTH_REFRESH_INVALID` | 401 | refresh token 없음·만료·재사용 감지 |
| `AUTH_KAKAO_FAILED` | 502 | 카카오 토큰 교환·사용자 조회 실패 |
| `AUTH_FORBIDDEN` | 403 | 역할 부족(예: 코치 전용 API) |
| `CONSENT_REQUIRED` | 403 | 분석 동의 없음 |
| `BODY_PROFILE_REQUIRED` | 409 | 신체 프로필 없음 |
| `BODY_PROFILE_OUT_OF_RANGE` | 400 | 키·윙스팬 범위 밖 |
| `VIDEO_INVALID_FORMAT` | 400 | 허용하지 않는 형식·크기 |
| `VIDEO_NOT_UPLOADED` | 409 | 스토리지에 파일이 없음 |
| `VIDEO_ALREADY_USED` | 409 | 이미 시도에 쓰인 영상 |
| `JOB_INVALID_STATE` | 409 | 현재 작업 상태에서 할 수 없는 요청 |
| `HOLD_NOT_IN_SCENE` | 400 | 작업의 벽 장면에 없는 홀드 |
| `PROBLEM_INVALID_HOLDS` | 400 | 시작·종료 없음, 보류 홀드에 역할 지정 등 |
| `REPORT_NOT_READY` | 409 | 분석이 끝나지 않음 |
| `COMPARISON_NOT_ALLOWED` | 400 | 다른 문제·다른 촬영 위치의 시도 비교 |
| `COACH_REVIEW_DUPLICATED` | 409 | 같은 코치가 이미 검토함 |
| `IDEMPOTENCY_KEY_REUSED` | 422 | 같은 키로 다른 요청 |
