# Prography API 명세서 (종합)

> 최종 수정: 2026-02-09
> Base URL: `https://api.prography.org/api/v1`
> 인증: `Authorization: Bearer {accessToken}`
> 총 엔드포인트: 94개 | 도메인: 18개

---

## 공통 사항

### 인증 (JWT)

- 로그인 시 발급되는 `accessToken`을 `Authorization: Bearer {token}` 헤더에 포함
- Access Token 만료 시 `/auth/refresh`로 갱신
- 일부 API는 인증 불필요 (로그인, 토큰 갱신, 권한 조회, 지원서 제출)

### 응답 형식

**성공**
```json
{
  "success": true,
  "data": { ... },
  "error": null
}
```

**에러**
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "ERROR_CODE",
    "message": "사람이 읽을 수 있는 메시지",
    "details": null
  }
}
```

### 페이지네이션

**Spring Data Page** (대부분의 목록 API)
```json
{
  "success": true,
  "data": {
    "content": [ ... ],
    "pageable": { ... },
    "totalElements": 150,
    "totalPages": 8,
    "size": 20,
    "number": 0,
    "first": true,
    "last": false,
    "empty": false
  }
}
```

**커스텀 PageResponse** (Audit Log API)
```json
{
  "success": true,
  "data": {
    "content": [ ... ],
    "page": 0,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8
  }
}
```

### 타임스탬프

모든 리소스 응답에 `createdAt`, `updatedAt` (ISO 8601) 포함. DB 레벨 6개 감사 컬럼 중 API에는 이 2개만 노출.

### API 버전

Spring Boot 4 내장 API Versioning 사용. 현재 모든 엔드포인트 v1.

### 공통 에러 코드

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `INVALID_INPUT` | 입력값 검증 실패 |
| 401 | `UNAUTHORIZED` | 인증 필요 |
| 401 | `AUTH_TOKEN_EXPIRED` | Access Token 만료 |
| 403 | `FORBIDDEN` | 권한 부족 |
| 404 | (도메인별 코드) | 리소스 없음 |
| 409 | (도메인별 코드) | 리소스 충돌 |
| 500 | `INTERNAL_SERVER_ERROR` | 서버 내부 오류 |

### Enum 값 참조

| Enum | 값 |
|------|------|
| `Part` | `ANDROID`, `iOS`, `WEB`, `SERVER`, `DESIGN`, `PO` |
| `MemberRole` | `MEMBER`, `ADMIN`, `SUPER_ADMIN` |
| `MemberStatus` | `INACTIVE`, `ACTIVE`, `ON_LEAVE`, `GRADUATED`, `BLACKLISTED`, `WITHDRAWN` |
| `GatheringStatus` | `SCHEDULED`, `OPEN`, `CLOSED` |
| `AttendanceStatus` | `PRESENT`, `LATE`, `ABSENT`, `EXCUSED` |
| `PenaltyType` | `ABSENCE`, `LATE`, `MANUAL_ADD`, `MANUAL_DEDUCT`, `RESET` |
| `PolicyKey` | `BLACKLIST_THRESHOLD`, `LATE_THRESHOLD_MINUTES`, `CLOSE_THRESHOLD_MINUTES`, `VERIFICATION_EXPIRY_SECONDS` |
| `CohortStatus` | `PLANNED`, `RECRUITING`, `ACTIVE`, `COMPLETED` |
| `TeamStatus` | `ACTIVE`, `DISBANDED` |
| `TeamRole` | `LEADER`, `MEMBER` |
| `ApplicationStatus` | `SUBMITTED`, `REVIEWING`, `ACCEPTED`, `REJECTED` |
| `AnnouncementTargetType` | `ALL`, `COHORT`, `TEAM` |
| `ProjectStatus` | `PLANNING`, `IN_PROGRESS`, `COMPLETED`, `ARCHIVED` |
| `MilestoneStatus` | `PENDING`, `COMPLETED` |
| `NotificationType` | `GATHERING_REMINDER`, `PENALTY_IMPOSED`, `PENALTY_THRESHOLD`, `ANNOUNCEMENT_NEW`, `APPLICATION_ACCEPTED`, `APPLICATION_REJECTED`, `COHORT_COMPLETED`, `FEE_REMINDER`, `SYSTEM` |
| `ExpenseCategory` | `VENUE`, `FOOD`, `EQUIPMENT`, `EVENT`, `MARKETING`, `OTHER` |

---

## 1. Auth API

**Base Path**: `/api/v1/auth`

### 1.1 로그인

```
POST /auth/login
```

| 항목 | 값 |
|------|------|
| 인증 | 불필요 |

**Request Body**
```json
{
  "email": "member@example.com",
  "password": "initialPassword123"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| email | String | O | 이메일 |
| password | String | O | 비밀번호 |

**Response 200**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzUxMiJ9...",
    "refreshToken": "550e8400-e29b-41d4-a716-446655440000",
    "tokenType": "Bearer",
    "expiresIn": 1800,
    "passwordChanged": false,
    "member": {
      "id": "0192e4a1-7b5a-7d3e-8f1a-2b3c4d5e6f70",
      "name": "홍길동",
      "email": "member@example.com",
      "role": "MEMBER",
      "generation": 11,
      "part": "SERVER"
    },
    "permissions": ["ATTENDANCE_CHECK_IN", "ATTENDANCE_EXCUSE_REQUEST", "ATTENDANCE_READ", "PENALTY_READ"]
  }
}
```

> `passwordChanged: false`이면 클라이언트에서 비밀번호 변경 화면으로 유도.

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 401 | `AUTH_INVALID_CREDENTIALS` | 이메일 또는 비밀번호 불일치 |
| 403 | `AUTH_ACCOUNT_BLOCKED` | 블랙리스트/탈퇴 계정 |

```bash
curl -X POST "https://api.prography.org/api/v1/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email": "member@example.com", "password": "initialPassword123"}'
```

---

### 1.2 로그아웃

```
POST /auth/logout
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{
  "refreshToken": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Response 200**
```json
{ "success": true, "data": null }
```

```bash
curl -X POST "https://api.prography.org/api/v1/auth/logout" \
  -H "Authorization: Bearer {accessToken}" \
  -H "Content-Type: application/json" \
  -d '{"refreshToken": "550e8400-e29b-41d4-a716-446655440000"}'
```

---

### 1.3 토큰 갱신

```
POST /auth/refresh
```

| 항목 | 값 |
|------|------|
| 인증 | 불필요 |

**Request Body**
```json
{
  "refreshToken": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Response 200**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzUxMiJ9...",
    "refreshToken": "new-refresh-token-uuid",
    "tokenType": "Bearer",
    "expiresIn": 1800
  }
}
```

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 401 | `AUTH_REFRESH_TOKEN_INVALID` | Refresh Token 무효 또는 만료 |

```bash
curl -X POST "https://api.prography.org/api/v1/auth/refresh" \
  -H "Content-Type: application/json" \
  -d '{"refreshToken": "550e8400-e29b-41d4-a716-446655440000"}'
```

---

### 1.4 비밀번호 변경

```
PATCH /auth/password
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{
  "currentPassword": "oldPassword123",
  "newPassword": "newSecurePassword456"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| currentPassword | String | O | 현재 비밀번호 |
| newPassword | String | O | 새 비밀번호 |

**Response 200**
```json
{ "success": true, "data": null }
```

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 401 | `AUTH_INVALID_CREDENTIALS` | 현재 비밀번호 불일치 |

```bash
curl -X PATCH "https://api.prography.org/api/v1/auth/password" \
  -H "Authorization: Bearer {accessToken}" \
  -H "Content-Type: application/json" \
  -d '{"currentPassword": "oldPassword123", "newPassword": "newSecurePassword456"}'
```

---

## 2. Member API

**Base Path**: `/api/v1/members`

### 2.1 회원 목록 조회 (검색)

```
GET /members
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| page | Int | X | 페이지 번호 (기본 0) |
| size | Int | X | 페이지 크기 (기본 20) |
| sort | String | X | 정렬 (예: `name,asc`) |
| generation | Int | X | 기수 필터 |
| status | MemberStatus | X | 상태 필터 |

**Response 200** — `Page<MemberResponse>`
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "0192e4a1-7b5a-7d3e-8f1a-2b3c4d5e6f70",
        "email": "member@example.com",
        "name": "홍길동",
        "phone": "010-1234-5678",
        "generation": 11,
        "part": "SERVER",
        "role": "MEMBER",
        "status": "ACTIVE",
        "profileImageUrl": null,
        "penaltyScore": 1.5,
        "passwordChanged": true,
        "joinedAt": "2026-01-15",
        "createdAt": "2026-01-15T09:00:00Z",
        "updatedAt": "2026-02-01T10:30:00Z"
      }
    ],
    "totalElements": 45,
    "totalPages": 3,
    "size": 20,
    "number": 0
  }
}
```

```bash
curl -X GET "https://api.prography.org/api/v1/members?generation=11&status=ACTIVE&page=0&size=20" \
  -H "Authorization: Bearer {accessToken}"
```

---

### 2.2 회원 추가

```
POST /members
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{
  "email": "newmember@example.com",
  "password": "initialPass123",
  "name": "김철수",
  "phone": "010-9876-5432",
  "generation": 11,
  "part": "WEB",
  "role": "MEMBER",
  "profileImageUrl": null,
  "joinedAt": "2026-02-08"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| email | String | O | 이메일 (중복 불가) |
| password | String | O | 초기 비밀번호 |
| name | String | O | 이름 |
| phone | String | X | 전화번호 |
| generation | Int | O | 기수 |
| part | Part | O | 파트 |
| role | MemberRole | O | 역할 |
| profileImageUrl | String | X | 프로필 이미지 URL |
| joinedAt | LocalDate | O | 가입일 |

**Response 201** — `MemberResponse` (status는 항상 `INACTIVE`)

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 409 | `MEMBER_EMAIL_DUPLICATE` | 이메일 중복 |

```bash
curl -X POST "https://api.prography.org/api/v1/members" \
  -H "Authorization: Bearer {accessToken}" \
  -H "Content-Type: application/json" \
  -d '{"email":"newmember@example.com","password":"initialPass123","name":"김철수","generation":11,"part":"WEB","role":"MEMBER","joinedAt":"2026-02-08"}'
```

---

### 2.3 회원 상세 조회

```
GET /members/{memberId}
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Response 200** — `MemberResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `MEMBER_NOT_FOUND` | 회원 없음 |

---

### 2.4 이메일로 회원 조회

```
GET /members/email/{email}
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Response 200** — `MemberResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `MEMBER_NOT_FOUND` | 회원 없음 |

---

### 2.5 회원 정보 수정

```
PUT /members/{memberId}
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{
  "name": "홍길동",
  "phone": "010-1111-2222",
  "part": "WEB",
  "profileImageUrl": "https://example.com/profile.jpg"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| name | String | X | 이름 (null이면 기존 유지) |
| phone | String | X | 전화번호 |
| part | Part | X | 파트 |
| profileImageUrl | String | X | 프로필 이미지 URL |

**Response 200** — `MemberResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `MEMBER_NOT_FOUND` | 회원 없음 |

---

### 2.6 회원 상태 변경

```
PATCH /members/{memberId}/status
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{ "newStatus": "GRADUATED" }
```

**Response 200** — `MemberResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `MEMBER_INVALID_STATUS_TRANSITION` | 허용되지 않는 상태 전환 |
| 404 | `MEMBER_NOT_FOUND` | 회원 없음 |

---

### 2.7 회원 역할 변경

```
PATCH /members/{memberId}/role
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{ "newRole": "ADMIN" }
```

**Response 200** — `MemberResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `MEMBER_NOT_FOUND` | 회원 없음 |

---

### 2.8 블랙리스트 처리

```
POST /members/{memberId}/blacklist
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

> Request Body 없음. `@AuthenticationPrincipal`에서 현재 사용자 ID를 추출.

**Response 200** — `MemberResponse` (status가 `BLACKLISTED`로 변경)

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `MEMBER_NOT_FOUND` | 회원 없음 |

---

### 2.9 회원별 팀 조회

```
GET /members/{memberId}/teams
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| cohortId | UUID | X | 특정 기수의 팀만 조회 |

**Response 200** — `List<MemberTeamResponse>`
```json
{
  "success": true,
  "data": [
    {
      "cohortId": "0192e4a1-aaaa-...",
      "cohortNumber": 11,
      "cohortName": "프로그라피 11기",
      "teamId": "0192e4a1-cccc-...",
      "teamName": "팀 A",
      "teamRole": "LEADER",
      "assignedAt": "2026-02-09T10:00:00Z"
    }
  ]
}
```

---

## 3. Cohort API (기수 관리)

**Base Path**: `/api/v1/cohorts`

### 3.1 기수 목록 조회

```
GET /cohorts
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| page | Int | X | 기본 0 |
| size | Int | X | 기본 20 |
| status | CohortStatus | X | 상태 필터 |

**Response 200** — `Page<CohortResponse>`
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "0192e4a1-aaaa-7d3e-8f1a-2b3c4d5e6f70",
        "number": 11,
        "name": "프로그라피 11기",
        "description": "2026년 상반기 활동",
        "status": "PLANNED",
        "startDate": "2026-03-01",
        "endDate": null,
        "createdAt": "2026-02-09T10:00:00Z",
        "updatedAt": "2026-02-09T10:00:00Z"
      }
    ],
    "totalElements": 11,
    "totalPages": 1,
    "size": 20,
    "number": 0
  }
}
```

```bash
curl -X GET "https://api.prography.org/api/v1/cohorts?status=ACTIVE" \
  -H "Authorization: Bearer {accessToken}"
```

---

### 3.2 기수 생성

```
POST /cohorts
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{
  "number": 12,
  "name": "프로그라피 12기",
  "description": "2026년 하반기 활동",
  "startDate": "2026-09-01"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| number | Int | O | 기수 번호 (>= 1, 유일) |
| name | String | O | 기수 이름 |
| description | String | X | 기수 설명 |
| startDate | LocalDate | O | 활동 시작일 |

**Response 201** — `CohortResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 409 | `COHORT_NUMBER_DUPLICATE` | 기수 번호 중복 |

---

### 3.3 기수 상세 조회

```
GET /cohorts/{cohortId}
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Response 200** — `CohortResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `COHORT_NOT_FOUND` | 기수 없음 |

---

### 3.4 기수 정보 수정

```
PATCH /cohorts/{cohortId}
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body** (부분 수정 — null이면 기존 유지)
```json
{
  "name": "프로그라피 11기 (수정)",
  "description": "수정된 설명",
  "startDate": "2026-03-15",
  "endDate": "2026-08-31"
}
```

**Response 200** — `CohortResponse`

---

### 3.5 기수 상태 변경

```
PATCH /cohorts/{cohortId}/status
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{ "newStatus": "ACTIVE" }
```

**Response 200** — `CohortResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `COHORT_INVALID_STATUS_TRANSITION` | 허용되지 않는 상태 전환 |
| 404 | `COHORT_NOT_FOUND` | 기수 없음 |

---

### 3.6 기수별 멤버 조회

```
GET /cohorts/{cohortId}/members
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| page | Int | X | 기본 0 |
| size | Int | X | 기본 20 |

**Response 200** — `Page<TeamMemberResponse>`

---

### 3.7 기수 종료 (Orchestrator)

```
POST /cohorts/{cohortId}/complete
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Response 200** — `CompleteCohortResponse`
```json
{
  "success": true,
  "data": {
    "cohortId": "0192e4a1-bbbb-7d3e-8f1a-2b3c4d5e6f70",
    "cohortNumber": 11,
    "status": "COMPLETED",
    "disbandedTeamCount": 8,
    "graduatedMemberCount": 40,
    "completedAt": "2026-08-31T18:00:00Z"
  }
}
```

> **기수 종료 시 자동 처리** (CompleteCohortOrchestrator):
> - Cohort 상태 COMPLETED로 변경
> - 해당 기수 ACTIVE 팀 모두 DISBANDED
> - 해당 기수 ACTIVE 회원 모두 GRADUATED

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `COHORT_INVALID_STATUS_TRANSITION` | ACTIVE 상태가 아님 |
| 404 | `COHORT_NOT_FOUND` | 기수 없음 |

---

## 4. Team API (팀 관리)

**Base Path**: `/api/v1/cohorts/{cohortId}/teams`

> 팀은 기수에 종속되므로 기수 하위 리소스로 설계.

### 4.1 팀 목록 조회

```
GET /cohorts/{cohortId}/teams
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| page | Int | X | 기본 0 |
| size | Int | X | 기본 20 |
| status | TeamStatus | X | 상태 필터 |

**Response 200** — `Page<TeamResponse>`
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "0192e4a1-cccc-7d3e-8f1a-2b3c4d5e6f70",
        "cohortId": "0192e4a1-aaaa-7d3e-8f1a-2b3c4d5e6f70",
        "cohortNumber": 11,
        "name": "팀 A",
        "description": "서버 + 웹 팀",
        "status": "ACTIVE",
        "memberCount": 6,
        "createdAt": "2026-02-09T10:00:00Z",
        "updatedAt": "2026-02-09T10:00:00Z"
      }
    ],
    "totalElements": 8,
    "totalPages": 1,
    "size": 20,
    "number": 0
  }
}
```

---

### 4.2 팀 생성

```
POST /cohorts/{cohortId}/teams
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{
  "name": "팀 A",
  "description": "서버 + 웹 팀"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| name | String | O | 팀 이름 |
| description | String | X | 팀 설명 |

**Response 201** — `TeamResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `COHORT_NOT_FOUND` | 기수 없음 |
| 409 | `TEAM_NAME_DUPLICATE` | 같은 기수 내 팀명 중복 |

---

### 4.3 팀 상세 조회

```
GET /cohorts/{cohortId}/teams/{teamId}
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Response 200** — `TeamDetailResponse` (members 목록 포함)

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `TEAM_NOT_FOUND` | 팀 없음 |

---

### 4.4 팀 정보 수정

```
PATCH /cohorts/{cohortId}/teams/{teamId}
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Request Body**
```json
{ "name": "팀 A (수정)", "description": "수정된 설명" }
```

**Response 200** — `TeamResponse`

---

### 4.5 팀 해체

```
POST /cohorts/{cohortId}/teams/{teamId}/disband
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Response 200** — `TeamResponse` (status → `DISBANDED`)

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `TEAM_ALREADY_DISBANDED` | 이미 해체된 팀 |
| 404 | `TEAM_NOT_FOUND` | 팀 없음 |

---

### 4.6 팀 멤버 목록 조회

```
GET /cohorts/{cohortId}/teams/{teamId}/members
```

**Response 200** — `List<TeamMemberResponse>`

---

### 4.7 팀 멤버 추가 (단건)

```
POST /cohorts/{cohortId}/teams/{teamId}/members
```

**Request Body**
```json
{ "memberId": "0192e4a1-7b5a-...", "teamRole": "MEMBER" }
```

| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|-------|------|
| memberId | UUID | O | | 배정할 회원 ID |
| teamRole | TeamRole | X | MEMBER | 팀 역할 |

**Response 201** — `TeamMemberResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `MEMBER_NOT_FOUND` | 회원 없음 |
| 400 | `TEAM_MEMBER_NOT_ACTIVE` | 비활성 회원 배정 시도 |
| 409 | `TEAM_MEMBER_ALREADY_ASSIGNED` | 해당 기수에 이미 팀 배정됨 |
| 400 | `TEAM_DISBANDED` | 해체된 팀에 추가 시도 |

---

### 4.8 팀 멤버 일괄 추가

```
POST /cohorts/{cohortId}/teams/{teamId}/members/batch
```

**Request Body**
```json
{
  "members": [
    { "memberId": "...", "teamRole": "LEADER" },
    { "memberId": "...", "teamRole": "MEMBER" }
  ]
}
```

**Response 201** — `List<TeamMemberResponse>` (첫 번째 오류 시 전체 롤백)

---

### 4.9 팀 멤버 역할 변경

```
PATCH /cohorts/{cohortId}/teams/{teamId}/members/{teamMemberId}
```

**Request Body**
```json
{ "teamRole": "LEADER" }
```

**Response 200** — `TeamMemberResponse`

---

### 4.10 팀 멤버 제거

```
DELETE /cohorts/{cohortId}/teams/{teamId}/members/{teamMemberId}
```

**Response 200** — `{ "success": true, "data": null }`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `TEAM_MEMBER_NOT_FOUND` | 팀 멤버 없음 |

---

## 5. Application API (지원 관리)

**Base Path**: `/api/v1/cohorts/{cohortId}/applications`

### 5.1 지원서 제출

```
POST /cohorts/{cohortId}/applications
```

| 항목 | 값 |
|------|------|
| 인증 | 불필요 (외부 지원자) |

**Request Body**
```json
{
  "applicantName": "김철수",
  "applicantEmail": "chulsoo@example.com",
  "applicantPhone": "010-1234-5678",
  "part": "SERVER",
  "portfolioUrl": "https://github.com/chulsoo",
  "motivation": "프로그라피에서 성장하고 싶습니다."
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| applicantName | String | O | 지원자 이름 |
| applicantEmail | String | O | 지원자 이메일 |
| applicantPhone | String | X | 전화번호 |
| part | Part | O | 지원 파트 |
| portfolioUrl | String | X | 포트폴리오 URL |
| motivation | String | O | 지원 동기 |

**Response 201** — `ApplicationResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `COHORT_NOT_FOUND` | 기수 없음 |
| 400 | `COHORT_NOT_RECRUITING` | 모집 중이 아닌 기수 |
| 409 | `APPLICATION_EMAIL_DUPLICATE` | 해당 기수에 이미 지원 |

---

### 5.2 지원서 목록 조회

```
GET /cohorts/{cohortId}/applications
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| page | Int | X | 기본 0 |
| size | Int | X | 기본 20 |
| status | ApplicationStatus | X | 상태 필터 |
| part | Part | X | 파트 필터 |

**Response 200** — `Page<ApplicationResponse>`

---

### 5.3 지원서 상세 조회

```
GET /cohorts/{cohortId}/applications/{applicationId}
```

**Response 200** — `ApplicationResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `APPLICATION_NOT_FOUND` | 지원서 없음 |

---

### 5.4 심사 시작

```
PATCH /cohorts/{cohortId}/applications/{applicationId}/review
```

**Response 200** — `ApplicationResponse` (status → `REVIEWING`)

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `APPLICATION_INVALID_STATUS_TRANSITION` | 허용되지 않는 상태 전환 |
| 404 | `APPLICATION_NOT_FOUND` | 지원서 없음 |

---

### 5.5 합격 처리

```
POST /cohorts/{cohortId}/applications/{applicationId}/accept
```

**Request Body**
```json
{ "reviewNote": "기술 역량과 협업 경험이 우수합니다." }
```

**Response 200** — `AcceptApplicationResponse`
```json
{
  "success": true,
  "data": {
    "application": { "id": "...", "status": "ACCEPTED", "reviewNote": "...", "reviewedBy": "...", "reviewedAt": "..." },
    "createdMemberId": "0192e4b0-cccc-..."
  }
}
```

> **합격 시 자동 처리** (AcceptApplicationOrchestrator): Application 상태 ACCEPTED, Member 자동 생성 (INACTIVE, 랜덤 초기 비밀번호)

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `APPLICATION_INVALID_STATUS_TRANSITION` | REVIEWING 상태가 아님 |
| 404 | `APPLICATION_NOT_FOUND` | 지원서 없음 |
| 409 | `MEMBER_EMAIL_DUPLICATE` | 이미 해당 이메일의 회원 존재 |

---

### 5.6 불합격 처리

```
POST /cohorts/{cohortId}/applications/{applicationId}/reject
```

**Request Body**
```json
{ "reviewNote": "포트폴리오 부족" }
```

**Response 200** — `ApplicationResponse` (status → `REJECTED`)

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `APPLICATION_INVALID_STATUS_TRANSITION` | REVIEWING 상태가 아님 |
| 404 | `APPLICATION_NOT_FOUND` | 지원서 없음 |

---

## 6. Announcement API (공지사항)

**Base Path**: `/api/v1/announcements`

### 6.1 공지사항 목록 조회

```
GET /announcements
```

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| page | Int | X | 기본 0 |
| size | Int | X | 기본 20 |
| targetType | AnnouncementTargetType | X | 대상 필터 |
| targetId | UUID | X | 대상 기수/팀 ID |
| pinned | Boolean | X | 고정 여부 필터 |

**Response 200** — `Page<AnnouncementResponse>`

---

### 6.2 공지사항 생성

```
POST /announcements
```

**Request Body**
```json
{
  "title": "11기 활동 안내",
  "content": "매주 토요일 14:00 정기 모임입니다.",
  "targetType": "COHORT",
  "targetId": "0192e4a1-bbbb-...",
  "pinned": false
}
```

| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|-------|------|
| title | String | O | | 공지 제목 |
| content | String | O | | 공지 내용 (Markdown) |
| targetType | AnnouncementTargetType | O | | 대상 유형 |
| targetId | UUID | 조건부 | null | ALL이 아니면 필수 |
| pinned | Boolean | X | false | 상단 고정 여부 |

**Response 201** — `AnnouncementResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `COHORT_NOT_FOUND` | targetType=COHORT인데 기수 없음 |
| 404 | `TEAM_NOT_FOUND` | targetType=TEAM인데 팀 없음 |
| 400 | `ANNOUNCEMENT_TARGET_ID_REQUIRED` | targetId 누락 |

---

### 6.3 공지사항 상세 조회

```
GET /announcements/{announcementId}
```

**Response 200** — `AnnouncementResponse`

**에러**: `404 ANNOUNCEMENT_NOT_FOUND`

---

### 6.4 공지사항 수정

```
PATCH /announcements/{announcementId}
```

**Request Body** (부분 수정)
```json
{ "title": "수정된 제목", "content": "수정된 내용" }
```

**Response 200** — `AnnouncementResponse`

---

### 6.5 공지사항 삭제

```
DELETE /announcements/{announcementId}
```

**Response 200** — `{ "success": true, "data": null }`

**에러**: `404 ANNOUNCEMENT_NOT_FOUND`

---

### 6.6 공지사항 고정/해제

```
PATCH /announcements/{announcementId}/pin
```

**Request Body**
```json
{ "pinned": true }
```

**Response 200** — `AnnouncementResponse`

---

## 7. Gathering API (모임)

**Base Path**: `/api/v1/gatherings`

### 7.1 모임 목록 조회

```
GET /gatherings
```

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| page | Int | X | 기본 0 |
| size | Int | X | 기본 20 |
| cohortId | UUID | X | 기수 ID 필터 |
| generation | Int | X | 기수 번호 필터 (deprecated, cohortId 우선) |
| status | GatheringStatus | X | 상태 필터 |

**Response 200** — `Page<GatheringResponse>`
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "0192e4a3-9d7c-...",
        "title": "11기 3차 정기 모임",
        "cohortId": "0192e4a1-cccc-...",
        "cohortNumber": 11,
        "generation": 11,
        "gatheringDate": "2026-02-15",
        "startTime": "14:00:00",
        "lateThresholdMinutes": 10,
        "closeThresholdMinutes": 30,
        "status": "SCHEDULED",
        "createdAt": "2026-02-08T10:00:00Z",
        "updatedAt": "2026-02-08T10:00:00Z"
      }
    ]
  }
}
```

---

### 7.2 모임 생성

```
POST /gatherings
```

**Request Body**
```json
{
  "title": "11기 3차 정기 모임",
  "description": "Spring Boot 실습",
  "cohortId": "0192e4a1-cccc-...",
  "gatheringDate": "2026-02-15",
  "startTime": "14:00",
  "lateThresholdMinutes": 10,
  "closeThresholdMinutes": 30
}
```

| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|-------|------|
| title | String | O | | 모임 제목 |
| description | String | X | null | 설명 |
| cohortId | UUID | O | | 기수 ID |
| gatheringDate | LocalDate | O | | 모임 날짜 |
| startTime | LocalTime | O | | 시작 시간 |
| lateThresholdMinutes | Int | X | 10 | 지각 기준 (분) |
| closeThresholdMinutes | Int | X | 30 | 마감 기준 (분) |

**Response 201** — `GatheringResponse`

**에러**: `400 COHORT_NOT_ACTIVE` (ACTIVE 상태가 아닌 기수)

---

### 7.3 모임 상세 조회

```
GET /gatherings/{id}
```

**Response 200** — `GatheringResponse`

**에러**: `404 GATHERING_NOT_FOUND`

---

### 7.4 모임 수정

```
PATCH /gatherings/{id}
```

**Request Body** (부분 수정)
```json
{
  "title": "수정된 제목",
  "gatheringDate": "2026-02-20",
  "startTime": "15:00",
  "lateThresholdMinutes": 15,
  "closeThresholdMinutes": 45
}
```

**Response 200** — `GatheringResponse`

---

### 7.5 출석 인증 코드 생성

```
POST /gatherings/{id}/verification
```

**Query Parameters**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|-------|------|
| expirySeconds | Int | X | 30 | 코드 유효 시간 (초) |

**Response 200** — `VerificationResponse`
```json
{
  "success": true,
  "data": {
    "gatheringId": "...",
    "code": "550e8400-e29b-41d4-a716-446655440000",
    "expiresAt": "2026-02-15T14:00:30Z",
    "expiresInSeconds": 30,
    "qrPayload": "{...}"
  }
}
```

---

### 7.6 모임 수동 마감

```
POST /gatherings/{id}/close
```

**Response 200** — `CloseGatheringResponse`
```json
{
  "success": true,
  "data": {
    "gatheringId": "...",
    "status": "CLOSED",
    "absentCount": 5,
    "penaltiesApplied": 7,
    "closedBy": "...",
    "closedDateTime": "2026-02-15T14:30:00Z"
  }
}
```

> **마감 시 자동 처리** (Orchestrator): 미출석자 ABSENT 생성 → 패널티 자동 부과 → 상태 CLOSED

---

## 8. Attendance API (출석)

**Base Path**: `/api/v1/attendances`

### 8.1 출석 체크인

```
POST /attendances
```

**Request Body**
```json
{
  "gatheringId": "0192e4a3-9d7c-...",
  "code": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Response 201** — `AttendanceRecordResponse`

> 출석 상태 자동 판정: `startTime` 이내 → PRESENT, + `lateThresholdMinutes` 이내 → LATE

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `VERIFICATION_EXPIRED` | 인증 코드 만료 |
| 400 | `VERIFICATION_INVALID` | 인증 코드 무효 |
| 400 | `GATHERING_NOT_OPEN` | 모임 미오픈/마감 |
| 403 | `ATTENDANCE_MEMBER_NOT_ACTIVE` | 비활성 회원 |
| 409 | `ATTENDANCE_ALREADY_CHECKED` | 중복 출석 |

---

### 8.2 사전 불참 신청

```
POST /attendances/excuse
```

**Request Body**
```json
{ "gatheringId": "...", "reason": "가족 행사로 인한 불참" }
```

**Response 201** — `AttendanceRecordResponse` (status: `EXCUSED`)

**에러**: `400 EXCUSE_DEADLINE_PASSED` (모임 이미 시작됨)

---

### 8.3 불참 신청 승인/거부

```
PATCH /attendances/{id}/excuse
```

**Request Body**
```json
{ "excuseApproved": true }
```

**Response 200** — `AttendanceRecordResponse`

---

### 8.4 출석 기록 조회 (검색)

```
GET /attendances
```

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| gatheringId | UUID | X | 모임 ID 필터 |
| memberId | UUID | X | 회원 ID 필터 |
| status | AttendanceStatus | X | 출석 상태 필터 |
| page | Int | X | 기본 0 |
| size | Int | X | 기본 20 |

**Response 200** — `Page<AttendanceRecordResponse>`

---

## 9. Penalty API (패널티)

**Base Path**: `/api/v1`

### 9.1 회원 패널티 이력 조회

```
GET /members/{memberId}/penalties
```

**Query Parameters**: `page` (기본 0), `size` (기본 20)

**Response 200** — `Page<PenaltyLedgerResponse>`
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "...",
        "memberId": "...",
        "penaltyId": "...",
        "gatheringId": "...",
        "type": "ABSENCE",
        "score": 1.0,
        "reason": null,
        "createdAt": "2026-02-01T15:30:00Z"
      }
    ]
  }
}
```

> `score`는 항상 양수. 타입으로 부호 판단 (`MANUAL_DEDUCT`, `RESET`은 차감).

---

### 9.2 수동 패널티 부과

```
POST /penalties/manual
```

**Request Body**
```json
{
  "memberId": "...",
  "type": "MANUAL_ADD",
  "score": 1.0,
  "reason": "무단 이탈"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| memberId | UUID | O | 대상 회원 ID |
| type | PenaltyType | O | `MANUAL_ADD` 또는 `MANUAL_DEDUCT` |
| score | BigDecimal | O | 점수 (양수) |
| reason | String | X | 사유 (max 500자) |

**Response 201** — `PenaltyLedgerResponse`

---

### 9.3 패널티 초기화

```
POST /penalties/reset
```

**Request Body**
```json
{
  "memberId": "...",
  "reason": "신규 기수 시작으로 패널티 초기화"
}
```

**Response 200** — `{ "success": true, "data": null }`

---

## 10. Policy API (정책)

**Base Path**: `/api/v1/policies`

### 10.1 전체 정책 조회

```
GET /policies
```

**Response 200** — `List<PolicyResponse>`
```json
{
  "success": true,
  "data": [
    { "key": "BLACKLIST_THRESHOLD", "value": "3.0", "description": "블랙리스트 자동 전환 패널티 점수", "updatedBy": null },
    { "key": "LATE_THRESHOLD_MINUTES", "value": "10", "description": "지각 기준 (분)", "updatedBy": null },
    { "key": "CLOSE_THRESHOLD_MINUTES", "value": "30", "description": "마감 기준 (분)", "updatedBy": null },
    { "key": "VERIFICATION_EXPIRY_SECONDS", "value": "30", "description": "인증 코드 유효 시간 (초)", "updatedBy": null }
  ]
}
```

---

### 10.2 정책 상세 조회

```
GET /policies/{key}
```

**Response 200** — `PolicyResponse`

**에러**: `404` Not Found

---

### 10.3 정책 값 변경

```
PUT /policies/{key}
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token (`@AuthenticationPrincipal`) |
| 감사 | `@Auditable(action = "POLICY_UPDATE", targetType = "POLICY")` |

**Request Body**
```json
{ "value": "5.0" }
```

**Response 200** — `PolicyResponse`

**에러**: `404 POLICY_NOT_FOUND`

---

## 11. Project API (프로젝트)

**Base Path**: `/api/v1/cohorts/{cohortId}/teams/{teamId}/projects`

### 11.1 프로젝트 생성

```
POST /cohorts/{cohortId}/teams/{teamId}/projects
```

**Request Body**
```json
{
  "name": "프로그라피 앱 v2",
  "description": "프로그라피 앱 리뉴얼 프로젝트",
  "repositoryUrl": "https://github.com/prography/app-v2"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| name | String | O | 프로젝트명 |
| description | String | X | 프로젝트 설명 |
| repositoryUrl | String | X | 코드 저장소 URL |

**Response 201** — `ProjectResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `TEAM_NOT_FOUND` | 팀 없음 |
| 400 | `TEAM_ALREADY_DISBANDED` | 해산된 팀 |
| 409 | `PROJECT_NAME_DUPLICATE` | 팀 내 프로젝트명 중복 |

---

### 11.2 프로젝트 목록 조회

```
GET /cohorts/{cohortId}/teams/{teamId}/projects
```

**Query Parameters**: `page`, `size`, `status` (ProjectStatus)

**Response 200** — `Page<ProjectResponse>`

---

### 11.3 프로젝트 상세 조회

```
GET /cohorts/{cohortId}/teams/{teamId}/projects/{projectId}
```

**Response 200** — `ProjectResponse` (마일스톤 포함)

**에러**: `404 PROJECT_NOT_FOUND`

---

### 11.4 프로젝트 수정

```
PATCH /cohorts/{cohortId}/teams/{teamId}/projects/{projectId}
```

**Request Body** (부분 수정)
```json
{ "name": "수정된 프로젝트명", "description": "수정된 설명", "repositoryUrl": "..." }
```

**Response 200** — `ProjectResponse`

---

### 11.5 프로젝트 상태 변경

```
PATCH /cohorts/{cohortId}/teams/{teamId}/projects/{projectId}/status
```

**Request Body**
```json
{ "status": "IN_PROGRESS", "startDate": "2026-03-01" }
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| status | ProjectStatus | O | 변경할 상태 |
| startDate | LocalDate | 조건부 | IN_PROGRESS 전환 시 필수 |

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `PROJECT_INVALID_STATUS_TRANSITION` | 불가능한 상태 전환 |
| 400 | `PROJECT_START_DATE_REQUIRED` | startDate 누락 |

---

### 11.6 마일스톤 추가

```
POST /cohorts/{cohortId}/teams/{teamId}/projects/{projectId}/milestones
```

**Request Body**
```json
{ "name": "MVP 완성", "description": "핵심 기능 구현 완료", "dueDate": "2026-04-15" }
```

**Response 201** — `ProjectResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `PROJECT_NOT_ACTIVE` | 프로젝트가 COMPLETED/ARCHIVED |
| 409 | `MILESTONE_NAME_DUPLICATE` | 마일스톤명 중복 |

---

### 11.7 마일스톤 완료

```
POST /cohorts/{cohortId}/teams/{teamId}/projects/{projectId}/milestones/{milestoneId}/complete
```

**Response 200** — `ProjectResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `MILESTONE_NOT_FOUND` | 마일스톤 없음 |
| 400 | `MILESTONE_ALREADY_COMPLETED` | 이미 완료 |

---

## 12. ActivityReport API (활동 보고서)

**Base Path**: `/api/v1/gatherings/{gatheringId}/activity-reports`

### 12.1 활동 보고서 생성

```
POST /gatherings/{gatheringId}/activity-reports
```

**Request Body**
```json
{
  "teamId": "...",
  "title": "11기 3차 모임 팀 A 활동 보고서",
  "content": "## 금주 진행 사항\n\n- API 설계 완료",
  "attendeeSummary": "참석: 홍길동, 김철수 / 불참: 박민수"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| teamId | UUID | O | 작성 팀 ID |
| title | String | O | 보고서 제목 |
| content | String | O | 보고서 내용 (Markdown) |
| attendeeSummary | String | X | 참석자 요약 |

**Response 201** — `ActivityReportResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `GATHERING_NOT_FOUND` | 모임 없음 |
| 404 | `TEAM_NOT_FOUND` | 팀 없음 |
| 403 | `ACTIVITY_REPORT_NOT_TEAM_LEADER` | 팀 리더가 아님 |
| 400 | `ACTIVITY_REPORT_COHORT_MISMATCH` | 모임과 팀의 기수 불일치 |
| 409 | `ACTIVITY_REPORT_ALREADY_EXISTS` | 해당 모임에 팀 보고서 이미 존재 |

---

### 12.2 활동 보고서 목록 조회 (모임별)

```
GET /gatherings/{gatheringId}/activity-reports
```

**Query Parameters**: `page`, `size`

**Response 200** — `Page<ActivityReportResponse>`

---

### 12.3 활동 보고서 상세 조회

```
GET /gatherings/{gatheringId}/activity-reports/{reportId}
```

**Response 200** — `ActivityReportResponse`

**에러**: `404 ACTIVITY_REPORT_NOT_FOUND`

---

### 12.4 활동 보고서 수정

```
PATCH /gatherings/{gatheringId}/activity-reports/{reportId}
```

**Request Body** (부분 수정)
```json
{ "title": "수정된 제목", "content": "수정된 내용", "attendeeSummary": "..." }
```

**Response 200** — `ActivityReportResponse`

**에러**: `403 ACTIVITY_REPORT_NOT_TEAM_LEADER`

---

### 12.5 팀별 활동 보고서 목록

```
GET /cohorts/{cohortId}/teams/{teamId}/activity-reports
```

**Query Parameters**: `page`, `size`

**Response 200** — `Page<ActivityReportResponse>`

---

## 13. Notification API (알림)

**Base Path**: `/api/v1/notifications`

### 13.1 내 알림 목록 조회

```
GET /notifications
```

| 항목 | 값 |
|------|------|
| 인증 | Bearer Token |
| 주체 | `@AuthenticationPrincipal UUID` (본인 알림만) |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| page | Int | X | 기본 0 |
| size | Int | X | 기본 20 |
| read | Boolean | X | 읽음 여부 필터 |
| type | NotificationType | X | 알림 유형 필터 |

**Response 200** — `Page<NotificationResponse>`
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "...",
        "type": "GATHERING_REMINDER",
        "title": "모임 알림",
        "message": "내일 14:00에 11기 3차 정기 모임이 예정되어 있습니다.",
        "referenceType": "GATHERING",
        "referenceId": "...",
        "read": false,
        "readAt": null,
        "createdAt": "2026-02-14T10:00:00Z"
      }
    ]
  }
}
```

---

### 13.2 안읽은 알림 수 조회

```
GET /notifications/unread-count
```

**Response 200**
```json
{ "success": true, "data": { "unreadCount": 5 } }
```

---

### 13.3 알림 읽음 처리

```
PATCH /notifications/{notificationId}/read
```

**Response 200** — `NotificationResponse` (read=true, readAt 설정)

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `NOTIFICATION_NOT_FOUND` | 알림 없음 |
| 403 | `NOTIFICATION_NOT_RECIPIENT` | 본인 알림이 아님 |

---

### 13.4 전체 읽음 처리

```
PATCH /notifications/read-all
```

**Response 200**
```json
{ "success": true, "data": { "updatedCount": 5 } }
```

---

## 14. Dashboard API (통계)

**Base Path**: `/api/v1/dashboard`

> ADMIN 이상만 접근 가능.

### 14.1 기수별 대시보드

```
GET /dashboard/cohorts/{cohortId}
```

**Response 200** — `CohortDashboardResponse`
```json
{
  "success": true,
  "data": {
    "cohortId": "...",
    "cohortNumber": 11,
    "cohortName": "프로그라피 11기",
    "cohortStatus": "ACTIVE",
    "memberStats": { "total": 50, "active": 45, "onLeave": 2, "graduated": 0, "blacklisted": 1, "withdrawn": 2 },
    "teamStats": { "total": 8, "active": 8, "disbanded": 0 },
    "gatheringStats": { "total": 5, "scheduled": 1, "closed": 4 },
    "attendanceStats": { "averageAttendanceRate": 87.5, "totalPresent": 180, "totalLate": 15, "totalAbsent": 5, "totalExcused": 10 },
    "partDistribution": { "ANDROID": 8, "iOS": 7, "WEB": 10, "SERVER": 12, "DESIGN": 8, "PO": 5 },
    "feeStats": { "totalPolicies": 1, "totalCollected": 2250000, "totalExpenses": 1800000, "balance": 450000 }
  }
}
```

**에러**: `404 COHORT_NOT_FOUND`

---

### 14.2 팀별 대시보드

```
GET /dashboard/cohorts/{cohortId}/teams/{teamId}
```

**Response 200** — `TeamDashboardResponse`
```json
{
  "success": true,
  "data": {
    "teamId": "...",
    "teamName": "팀 A",
    "cohortId": "...",
    "cohortNumber": 11,
    "totalMembers": 6,
    "attendanceRate": 91.7,
    "totalActivityReports": 4,
    "projectCount": 1,
    "memberSummaries": [
      { "memberId": "...", "memberName": "홍길동", "part": "SERVER", "presentCount": 4, "lateCount": 1, "absentCount": 0, "excusedCount": 0, "attendanceRate": 100.0, "penaltyScore": 0.5 }
    ]
  }
}
```

**에러**: `404 COHORT_NOT_FOUND`, `404 TEAM_NOT_FOUND`

---

## 15. FeePolicy API (회비 관리)

**Base Path**: `/api/v1/cohorts/{cohortId}/fees`

### 15.1 회비 정책 생성

```
POST /cohorts/{cohortId}/fees
```

**Request Body**
```json
{
  "name": "11기 정기 회비",
  "amount": 50000,
  "dueDate": "2026-03-31",
  "description": "11기 활동 기간 정기 회비"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| name | String | O | 회비 정책명 |
| amount | BigDecimal | O | 납부 금액 (0 초과) |
| dueDate | LocalDate | O | 납부 기한 |
| description | String | X | 설명 |

**Response 201** — `FeePolicyResponse`

> 생성 시 기수 전체 회원에게 `FEE_REMINDER` 알림 자동 발송.

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `COHORT_NOT_FOUND` | 기수 없음 |
| 409 | `FEE_POLICY_NAME_DUPLICATE` | 기수 내 정책명 중복 |

---

### 15.2 회비 정책 목록 조회

```
GET /cohorts/{cohortId}/fees
```

**Response 200** — `List<FeePolicyResponse>`

---

### 15.3 회비 정책 상세 조회

```
GET /cohorts/{cohortId}/fees/{feeId}
```

**Response 200** — `FeePolicyResponse` (전체 납부 목록 포함)

**에러**: `404 FEE_POLICY_NOT_FOUND`

---

### 15.4 납부 기록 등록

```
POST /cohorts/{cohortId}/fees/{feeId}/payments
```

**Request Body**
```json
{
  "memberId": "...",
  "paidAt": "2026-02-15T10:00:00Z",
  "note": "계좌이체 확인"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| memberId | UUID | O | 납부 회원 ID |
| paidAt | Instant | O | 납부 일시 (ISO 8601) |
| note | String | X | 비고 |

**Response 201** — `FeePolicyResponse`

**에러**

| HTTP | 코드 | 설명 |
|------|------|------|
| 404 | `FEE_POLICY_NOT_FOUND` | 회비 정책 없음 |
| 404 | `MEMBER_NOT_FOUND` | 회원 없음 |
| 409 | `PAYMENT_ALREADY_EXISTS` | 이미 납부 완료 |

---

### 15.5 미납자 목록 조회

```
GET /cohorts/{cohortId}/fees/{feeId}/unpaid
```

**Response 200** — `List<UnpaidMemberResponse>`
```json
{
  "success": true,
  "data": [
    { "memberId": "...", "memberName": "박민수", "memberEmail": "minsu@example.com", "part": "WEB", "isOverdue": true }
  ]
}
```

---

### 15.6 기수별 회비 수지 요약

```
GET /cohorts/{cohortId}/fees/summary
```

**Response 200** — `FeeSummaryResponse`
```json
{
  "success": true,
  "data": {
    "cohortId": "...",
    "cohortNumber": 11,
    "totalCollected": 2250000,
    "totalExpenses": 1800000,
    "balance": 450000,
    "policies": [{ "id": "...", "name": "11기 정기 회비", "amount": 50000, "paidCount": 45, "totalMembers": 50 }],
    "expensesByCategory": { "VENUE": 960000, "FOOD": 540000, "EQUIPMENT": 200000, "EVENT": 100000, "MARKETING": 0, "OTHER": 0 }
  }
}
```

**에러**: `404 COHORT_NOT_FOUND`

---

## 16. Expense API (지출 관리)

**Base Path**: `/api/v1/cohorts/{cohortId}/expenses`

### 16.1 지출 내역 등록

```
POST /cohorts/{cohortId}/expenses
```

**Request Body**
```json
{
  "category": "VENUE",
  "description": "스터디카페 대여 (2월 3주차)",
  "amount": 120000,
  "spentAt": "2026-02-15",
  "receiptUrl": "https://storage.example.com/receipts/2026-02-15.jpg"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| category | ExpenseCategory | O | 지출 카테고리 |
| description | String | O | 지출 설명 |
| amount | BigDecimal | O | 지출 금액 (0 초과) |
| spentAt | LocalDate | O | 지출일 |
| receiptUrl | String | X | 영수증 URL |

**Response 201** — `ExpenseResponse`

---

### 16.2 지출 내역 목록 조회

```
GET /cohorts/{cohortId}/expenses
```

**Query Parameters**: `page`, `size`, `category` (ExpenseCategory)

**Response 200** — `Page<ExpenseResponse>`

---

### 16.3 지출 내역 수정

```
PATCH /cohorts/{cohortId}/expenses/{expenseId}
```

**Request Body** (부분 수정)
```json
{ "description": "수정된 설명", "amount": 130000 }
```

**Response 200** — `ExpenseResponse`

**에러**: `404 EXPENSE_NOT_FOUND`

---

### 16.4 지출 내역 삭제

```
DELETE /cohorts/{cohortId}/expenses/{expenseId}
```

**Response 200** — `{ "success": true, "data": null }`

**에러**: `404 EXPENSE_NOT_FOUND`

---

## 17. Permission API (권한)

**Base Path**: `/api/v1/permissions`

### 17.1 역할별 권한 목록 조회

```
GET /permissions/role/{role}
```

| 항목 | 값 |
|------|------|
| 인증 | 불필요 |

**Response 200** — `List<PermissionResponse>`
```json
{
  "success": true,
  "data": [
    { "id": "...", "code": "ATTENDANCE_CHECK_IN", "name": "출석 체크인", "description": "모임 출석 체크인 권한", "category": "ATTENDANCE", "createdAt": "...", "updatedAt": "..." }
  ]
}
```

---

### 17.2 권한 확인

```
GET /permissions/check
```

| 항목 | 값 |
|------|------|
| 인증 | 불필요 |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| role | MemberRole | O | 확인할 역할 |
| code | String | O | 권한 코드 |

**Response 200** — `Boolean`
```json
{ "success": true, "data": true }
```

---

## 18. Audit Log API (감사 로그)

**Base Path**: `/api/v1/audit-logs`

> 모든 Audit Log API는 커스텀 `PageResponse` 형식 사용.

### 18.1 전체 감사 로그 조회

```
GET /audit-logs
```

**Query Parameters**: `page` (기본 0), `size` (기본 20), `sort` (기본 `createdAt,desc`)

**Response 200** — `PageResponse<AuditLogResponse>`
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "...",
        "actorId": "...",
        "action": "MEMBER_STATUS_CHANGE",
        "targetType": "MEMBER",
        "targetId": "...",
        "beforeState": { "status": "ACTIVE" },
        "afterState": { "status": "BLACKLISTED" },
        "diff": { "status": { "first": "ACTIVE", "second": "BLACKLISTED" } },
        "ipAddress": "192.168.1.1",
        "userAgent": "Mozilla/5.0...",
        "createdAt": "2026-02-08T10:00:00Z"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 100,
    "totalPages": 5
  }
}
```

---

### 18.2 감사 로그 상세 조회

```
GET /audit-logs/{id}
```

**Response 200** — `AuditLogResponse`

---

### 18.3 행위자별 감사 로그 조회

```
GET /audit-logs/actor/{actorId}
```

**Query Parameters**: `page`, `size`, `sort`

**Response 200** — `PageResponse<AuditLogResponse>`

---

### 18.4 액션별 감사 로그 조회

```
GET /audit-logs/action/{action}
```

**주요 action 값**: `MEMBER_STATUS_CHANGE`, `MEMBER_ROLE_CHANGE`, `MEMBER_BLACKLIST_ADD`, `ATTENDANCE_STATUS_MODIFY`, `PENALTY_MANUAL_ADD`, `PENALTY_MANUAL_DEDUCT`, `PENALTY_RESET`, `POLICY_UPDATE`, `GATHERING_CLOSE`

**Response 200** — `PageResponse<AuditLogResponse>`

---

### 18.5 대상 타입별 감사 로그 조회

```
GET /audit-logs/target-type/{targetType}
```

**targetType 값**: `MEMBER`, `GATHERING`, `ATTENDANCE`, `PENALTY`, `POLICY`

**Response 200** — `PageResponse<AuditLogResponse>`

---

### 18.6 기간별 감사 로그 조회

```
GET /audit-logs/date-range
```

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| startDate | Instant | O | 시작 일시 (ISO 8601) |
| endDate | Instant | O | 종료 일시 (ISO 8601) |
| page | Int | X | 기본 0 |
| size | Int | X | 기본 20 |

**Response 200** — `PageResponse<AuditLogResponse>`

---

## 부록 A: 전체 엔드포인트 요약표

| # | Method | Path | 설명 | 인증 |
|---|--------|------|------|------|
| **Auth** | | | | |
| 1 | POST | `/auth/login` | 로그인 | X |
| 2 | POST | `/auth/logout` | 로그아웃 | O |
| 3 | POST | `/auth/refresh` | 토큰 갱신 | X |
| 4 | PATCH | `/auth/password` | 비밀번호 변경 | O |
| **Member** | | | | |
| 5 | GET | `/members` | 회원 목록 (검색) | O |
| 6 | POST | `/members` | 회원 추가 | O |
| 7 | GET | `/members/{memberId}` | 회원 상세 | O |
| 8 | GET | `/members/email/{email}` | 이메일로 조회 | O |
| 9 | PUT | `/members/{memberId}` | 회원 수정 | O |
| 10 | PATCH | `/members/{memberId}/status` | 상태 변경 | O |
| 11 | PATCH | `/members/{memberId}/role` | 역할 변경 | O |
| 12 | POST | `/members/{memberId}/blacklist` | 블랙리스트 | O |
| 13 | GET | `/members/{memberId}/teams` | 회원별 팀 조회 | O |
| **Cohort** | | | | |
| 14 | GET | `/cohorts` | 기수 목록 | O |
| 15 | POST | `/cohorts` | 기수 생성 | O |
| 16 | GET | `/cohorts/{cohortId}` | 기수 상세 | O |
| 17 | PATCH | `/cohorts/{cohortId}` | 기수 수정 | O |
| 18 | PATCH | `/cohorts/{cohortId}/status` | 기수 상태 변경 | O |
| 19 | GET | `/cohorts/{cohortId}/members` | 기수별 멤버 | O |
| 20 | POST | `/cohorts/{cohortId}/complete` | 기수 종료 | O |
| **Team** | | | | |
| 21 | GET | `/cohorts/{cohortId}/teams` | 팀 목록 | O |
| 22 | POST | `/cohorts/{cohortId}/teams` | 팀 생성 | O |
| 23 | GET | `/cohorts/{cohortId}/teams/{teamId}` | 팀 상세 | O |
| 24 | PATCH | `/cohorts/{cohortId}/teams/{teamId}` | 팀 수정 | O |
| 25 | POST | `/cohorts/{cohortId}/teams/{teamId}/disband` | 팀 해체 | O |
| 26 | GET | `/cohorts/{cohortId}/teams/{teamId}/members` | 팀 멤버 목록 | O |
| 27 | POST | `/cohorts/{cohortId}/teams/{teamId}/members` | 멤버 추가 | O |
| 28 | POST | `/cohorts/{cohortId}/teams/{teamId}/members/batch` | 멤버 일괄 추가 | O |
| 29 | PATCH | `/cohorts/{cohortId}/teams/{teamId}/members/{id}` | 멤버 역할 변경 | O |
| 30 | DELETE | `/cohorts/{cohortId}/teams/{teamId}/members/{id}` | 멤버 제거 | O |
| **Application** | | | | |
| 31 | POST | `/cohorts/{cohortId}/applications` | 지원서 제출 | X |
| 32 | GET | `/cohorts/{cohortId}/applications` | 지원서 목록 | O |
| 33 | GET | `/cohorts/{cohortId}/applications/{id}` | 지원서 상세 | O |
| 34 | PATCH | `/cohorts/{cohortId}/applications/{id}/review` | 심사 시작 | O |
| 35 | POST | `/cohorts/{cohortId}/applications/{id}/accept` | 합격 처리 | O |
| 36 | POST | `/cohorts/{cohortId}/applications/{id}/reject` | 불합격 처리 | O |
| **Announcement** | | | | |
| 37 | GET | `/announcements` | 공지 목록 | O |
| 38 | POST | `/announcements` | 공지 생성 | O |
| 39 | GET | `/announcements/{id}` | 공지 상세 | O |
| 40 | PATCH | `/announcements/{id}` | 공지 수정 | O |
| 41 | DELETE | `/announcements/{id}` | 공지 삭제 | O |
| 42 | PATCH | `/announcements/{id}/pin` | 공지 고정/해제 | O |
| **Gathering** | | | | |
| 43 | GET | `/gatherings` | 모임 목록 | O |
| 44 | POST | `/gatherings` | 모임 생성 | O |
| 45 | GET | `/gatherings/{id}` | 모임 상세 | O |
| 46 | PATCH | `/gatherings/{id}` | 모임 수정 | O |
| 47 | POST | `/gatherings/{id}/verification` | 인증 코드 생성 | O |
| 48 | POST | `/gatherings/{id}/close` | 모임 마감 | O |
| **Attendance** | | | | |
| 49 | POST | `/attendances` | 출석 체크인 | O |
| 50 | POST | `/attendances/excuse` | 불참 신청 | O |
| 51 | PATCH | `/attendances/{id}/excuse` | 불참 승인/거부 | O |
| 52 | GET | `/attendances` | 출석 기록 조회 | O |
| **Penalty** | | | | |
| 53 | GET | `/members/{memberId}/penalties` | 패널티 이력 | O |
| 54 | POST | `/penalties/manual` | 수동 패널티 | O |
| 55 | POST | `/penalties/reset` | 패널티 초기화 | O |
| **Policy** | | | | |
| 56 | GET | `/policies` | 전체 정책 | O |
| 57 | GET | `/policies/{key}` | 정책 상세 | O |
| 58 | PUT | `/policies/{key}` | 정책 변경 | O |
| **Project** | | | | |
| 59 | POST | `/cohorts/{cohortId}/teams/{teamId}/projects` | 프로젝트 생성 | O |
| 60 | GET | `/cohorts/{cohortId}/teams/{teamId}/projects` | 프로젝트 목록 | O |
| 61 | GET | `/cohorts/{cohortId}/teams/{teamId}/projects/{id}` | 프로젝트 상세 | O |
| 62 | PATCH | `/cohorts/{cohortId}/teams/{teamId}/projects/{id}` | 프로젝트 수정 | O |
| 63 | PATCH | `/cohorts/{cohortId}/teams/{teamId}/projects/{id}/status` | 프로젝트 상태 변경 | O |
| 64 | POST | `/cohorts/{cohortId}/teams/{teamId}/projects/{id}/milestones` | 마일스톤 추가 | O |
| 65 | POST | `.../{id}/milestones/{milestoneId}/complete` | 마일스톤 완료 | O |
| **ActivityReport** | | | | |
| 66 | POST | `/gatherings/{gatheringId}/activity-reports` | 보고서 생성 | O |
| 67 | GET | `/gatherings/{gatheringId}/activity-reports` | 모임별 보고서 목록 | O |
| 68 | GET | `/gatherings/{gatheringId}/activity-reports/{id}` | 보고서 상세 | O |
| 69 | PATCH | `/gatherings/{gatheringId}/activity-reports/{id}` | 보고서 수정 | O |
| 70 | GET | `/cohorts/{cohortId}/teams/{teamId}/activity-reports` | 팀별 보고서 목록 | O |
| **Notification** | | | | |
| 71 | GET | `/notifications` | 내 알림 목록 | O |
| 72 | GET | `/notifications/unread-count` | 안읽은 알림 수 | O |
| 73 | PATCH | `/notifications/{id}/read` | 알림 읽음 처리 | O |
| 74 | PATCH | `/notifications/read-all` | 전체 읽음 처리 | O |
| **Dashboard** | | | | |
| 75 | GET | `/dashboard/cohorts/{cohortId}` | 기수별 대시보드 | O |
| 76 | GET | `/dashboard/cohorts/{cohortId}/teams/{teamId}` | 팀별 대시보드 | O |
| **FeePolicy** | | | | |
| 77 | POST | `/cohorts/{cohortId}/fees` | 회비 정책 생성 | O |
| 78 | GET | `/cohorts/{cohortId}/fees` | 회비 정책 목록 | O |
| 79 | GET | `/cohorts/{cohortId}/fees/{feeId}` | 회비 정책 상세 | O |
| 80 | POST | `/cohorts/{cohortId}/fees/{feeId}/payments` | 납부 기록 등록 | O |
| 81 | GET | `/cohorts/{cohortId}/fees/{feeId}/unpaid` | 미납자 목록 | O |
| 82 | GET | `/cohorts/{cohortId}/fees/summary` | 수지 요약 | O |
| **Expense** | | | | |
| 83 | POST | `/cohorts/{cohortId}/expenses` | 지출 등록 | O |
| 84 | GET | `/cohorts/{cohortId}/expenses` | 지출 목록 | O |
| 85 | PATCH | `/cohorts/{cohortId}/expenses/{id}` | 지출 수정 | O |
| 86 | DELETE | `/cohorts/{cohortId}/expenses/{id}` | 지출 삭제 | O |
| **Permission** | | | | |
| 87 | GET | `/permissions/role/{role}` | 역할별 권한 | X |
| 88 | GET | `/permissions/check` | 권한 확인 | X |
| **Audit Log** | | | | |
| 89 | GET | `/audit-logs` | 전체 로그 | O |
| 90 | GET | `/audit-logs/{id}` | 로그 상세 | O |
| 91 | GET | `/audit-logs/actor/{actorId}` | 행위자별 | O |
| 92 | GET | `/audit-logs/action/{action}` | 액션별 | O |
| 93 | GET | `/audit-logs/target-type/{targetType}` | 대상 타입별 | O |
| 94 | GET | `/audit-logs/date-range` | 기간별 | O |

> **총 94개 엔드포인트** (21개 컨트롤러, 18개 도메인)

---

## 부록 B: 전체 에러코드 참조표

| HTTP | 코드 | 설명 |
|------|------|------|
| **공통** | | |
| 400 | `INVALID_INPUT` | 입력값 검증 실패 |
| 401 | `UNAUTHORIZED` | 인증 필요 |
| 401 | `AUTH_TOKEN_EXPIRED` | Access Token 만료 |
| 403 | `FORBIDDEN` | 권한 부족 |
| 500 | `INTERNAL_SERVER_ERROR` | 서버 내부 오류 |
| **Auth** | | |
| 401 | `AUTH_INVALID_CREDENTIALS` | 이메일/비밀번호 불일치 |
| 403 | `AUTH_ACCOUNT_BLOCKED` | 차단된 계정 |
| 401 | `AUTH_REFRESH_TOKEN_INVALID` | 유효하지 않은 Refresh Token |
| 403 | `AUTH_PASSWORD_CHANGE_REQUIRED` | 초기 비밀번호 변경 필요 |
| **Member** | | |
| 404 | `MEMBER_NOT_FOUND` | 회원 없음 |
| 409 | `MEMBER_EMAIL_DUPLICATE` | 이메일 중복 |
| 400 | `MEMBER_INVALID_STATUS_TRANSITION` | 불가능한 상태 전환 |
| **Cohort** | | |
| 404 | `COHORT_NOT_FOUND` | 기수 없음 |
| 409 | `COHORT_NUMBER_DUPLICATE` | 기수 번호 중복 |
| 400 | `COHORT_INVALID_STATUS_TRANSITION` | 불가능한 상태 전환 |
| 400 | `COHORT_NOT_RECRUITING` | 모집 중이 아닌 기수 |
| 400 | `COHORT_NOT_ACTIVE` | ACTIVE 상태가 아닌 기수 |
| **Team** | | |
| 404 | `TEAM_NOT_FOUND` | 팀 없음 |
| 409 | `TEAM_NAME_DUPLICATE` | 같은 기수 내 팀명 중복 |
| 400 | `TEAM_ALREADY_DISBANDED` | 이미 해체된 팀 |
| 400 | `TEAM_DISBANDED` | 해체된 팀에 대한 작업 시도 |
| **TeamMember** | | |
| 404 | `TEAM_MEMBER_NOT_FOUND` | 팀 멤버 없음 |
| 409 | `TEAM_MEMBER_ALREADY_ASSIGNED` | 해당 기수에 이미 팀 배정됨 |
| 400 | `TEAM_MEMBER_NOT_ACTIVE` | 비활성 회원 배정 시도 |
| **Application** | | |
| 404 | `APPLICATION_NOT_FOUND` | 지원서 없음 |
| 409 | `APPLICATION_EMAIL_DUPLICATE` | 해당 기수에 이미 지원 |
| 400 | `APPLICATION_INVALID_STATUS_TRANSITION` | 불가능한 상태 전환 |
| **Announcement** | | |
| 404 | `ANNOUNCEMENT_NOT_FOUND` | 공지 없음 |
| 400 | `ANNOUNCEMENT_TARGET_ID_REQUIRED` | targetId 누락 |
| **Gathering** | | |
| 404 | `GATHERING_NOT_FOUND` | 모임 없음 |
| 400 | `GATHERING_NOT_OPEN` | 모임 미오픈/마감 |
| **Attendance** | | |
| 400 | `VERIFICATION_INVALID` | 인증 코드 무효 |
| 400 | `VERIFICATION_EXPIRED` | 인증 코드 만료 |
| 409 | `ATTENDANCE_ALREADY_CHECKED` | 중복 출석 |
| 404 | `ATTENDANCE_RECORD_NOT_FOUND` | 출석 기록 없음 |
| 403 | `ATTENDANCE_MEMBER_NOT_ACTIVE` | 비활성 회원 |
| 400 | `EXCUSE_DEADLINE_PASSED` | 불참 신청 기한 초과 |
| **Penalty** | | |
| 404 | `PENALTY_NOT_FOUND` | 패널티 정의 없음 |
| 409 | `PENALTY_ALREADY_BLACKLISTED` | 이미 블랙리스트 |
| **Policy** | | |
| 404 | `POLICY_NOT_FOUND` | 정책 없음 |
| **Project** | | |
| 404 | `PROJECT_NOT_FOUND` | 프로젝트 없음 |
| 409 | `PROJECT_NAME_DUPLICATE` | 팀 내 프로젝트명 중복 |
| 400 | `PROJECT_INVALID_STATUS_TRANSITION` | 불가능한 상태 전환 |
| 400 | `PROJECT_START_DATE_REQUIRED` | startDate 누락 |
| 400 | `PROJECT_NOT_ACTIVE` | COMPLETED/ARCHIVED 상태 |
| **Milestone** | | |
| 404 | `MILESTONE_NOT_FOUND` | 마일스톤 없음 |
| 409 | `MILESTONE_NAME_DUPLICATE` | 마일스톤명 중복 |
| 400 | `MILESTONE_ALREADY_COMPLETED` | 이미 완료된 마일스톤 |
| **ActivityReport** | | |
| 404 | `ACTIVITY_REPORT_NOT_FOUND` | 보고서 없음 |
| 403 | `ACTIVITY_REPORT_NOT_TEAM_LEADER` | 팀 리더가 아님 |
| 400 | `ACTIVITY_REPORT_COHORT_MISMATCH` | 모임과 팀의 기수 불일치 |
| 409 | `ACTIVITY_REPORT_ALREADY_EXISTS` | 모임+팀 보고서 중복 |
| **Notification** | | |
| 404 | `NOTIFICATION_NOT_FOUND` | 알림 없음 |
| 403 | `NOTIFICATION_NOT_RECIPIENT` | 본인 알림이 아님 |
| **FeePolicy** | | |
| 404 | `FEE_POLICY_NOT_FOUND` | 회비 정책 없음 |
| 409 | `FEE_POLICY_NAME_DUPLICATE` | 기수 내 정책명 중복 |
| 409 | `PAYMENT_ALREADY_EXISTS` | 이미 납부 완료 |
| **Expense** | | |
| 404 | `EXPENSE_NOT_FOUND` | 지출 내역 없음 |
| **Permission** | | |
| 403 | `PERMISSION_DENIED` | 권한 없음 |
| **Audit** | | |
| 404 | `AUDIT_LOG_NOT_FOUND` | 감사 로그 없음 |

---
