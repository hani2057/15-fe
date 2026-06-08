# API 에러 코드 표준

---

## 1. 응답 형태

### 에러 응답

모든 에러는 동일한 구조로 반환한다.

```json
{
  "error": {
    "code": "NICKNAME_TAKEN",
    "message": "이미 사용 중인 닉네임이에요."
  }
}
```

유효성 검사 실패 시 `details`로 필드별 오류를 함께 반환한다.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "입력값을 확인해주세요.",
    "details": [
      { "field": "nickname", "message": "닉네임을 입력해주세요." },
      { "field": "memberCount", "message": "인원수를 입력해주세요." }
    ]
  }
}
```

### 성공 응답

리소스 이름을 키로 사용한다. 제네릭 `data` 래퍼는 사용하지 않는다.

```json
{ "schedule": { ... } }
{ "member": { ... } }
{ "success": true }   ← 반환할 리소스가 없을 때 (DELETE 등)
```

---

## 2. HTTP 상태 코드 사용 규칙

| 코드 | 의미                  | 사용 상황                                   |
| ---- | --------------------- | ------------------------------------------- |
| 200  | OK                    | 조회, 수정 성공                             |
| 201  | Created               | 리소스 생성 성공                            |
| 400  | Bad Request           | 유효성 검사 실패                            |
| 401  | Unauthorized          | 쿠키 없음 또는 유효하지 않은 token          |
| 403  | Forbidden             | 인증은 됐으나 권한 없음 (팀장 전용 기능 등) |
| 404  | Not Found             | 존재하지 않는 리소스                        |
| 409  | Conflict              | 비즈니스 규칙 충돌 (중복, 정원 초과 등)     |
| 500  | Internal Server Error | 예상치 못한 서버 오류                       |

> 401 vs 403 구분: 401은 "누구인지 모름", 403은 "누구인지 알지만 허용 안 됨".

---

## 3. 에러 코드 목록

### 인증 / 권한

| code           | status | 상황                                                      |
| -------------- | ------ | --------------------------------------------------------- |
| `UNAUTHORIZED` | 401    | 쿠키 없음 또는 token이 DB의 어떤 Member와도 일치하지 않음 |
| `FORBIDDEN`    | 403    | 팀장이 아닌 사용자가 설정 변경 시도                       |

### Schedule

| code                 | status | 상황                       |
| -------------------- | ------ | -------------------------- |
| `SCHEDULE_NOT_FOUND` | 404    | 존재하지 않는 scheduleId   |
| `SCHEDULE_CLOSED`    | 409    | 종료된 방에 일정 제출 시도 |

### Member

| code               | status | 상황                            |
| ------------------ | ------ | ------------------------------- |
| `MEMBER_NOT_FOUND` | 404    | 존재하지 않는 memberId          |
| `NICKNAME_TAKEN`   | 409    | 같은 팀 내 닉네임 중복          |
| `ROOM_FULL`        | 409    | 현재 멤버 수 ≥ team.memberCount |

### 입력 검증

| code               | status | 상황                                               |
| ------------------ | ------ | -------------------------------------------------- |
| `VALIDATION_ERROR` | 400    | Zod 스키마 검사 실패. `details`에 필드별 오류 포함 |

### 서버

| code             | status | 상황                  |
| ---------------- | ------ | --------------------- |
| `INTERNAL_ERROR` | 500    | 예상치 못한 서버 오류 |

---

## 4. 구현 패턴

### 서버 (Route Handler)

```ts
// lib/errors.ts
export class AppError extends Error {
  constructor(
    public code: string,
    public message: string,
    public status: number,
    public details?: { field: string; message: string }[],
  ) {
    super(message);
  }
}

export const Errors = {
  UNAUTHORIZED: () => new AppError("UNAUTHORIZED", "인증이 필요해요.", 401),
  FORBIDDEN: () => new AppError("FORBIDDEN", "권한이 없어요.", 403),
  SCHEDULE_NOT_FOUND: () =>
    new AppError("SCHEDULE_NOT_FOUND", "일정을 찾을 수 없어요.", 404),
  SCHEDULE_CLOSED: () =>
    new AppError("SCHEDULE_CLOSED", "종료된 방이에요.", 409),
  MEMBER_NOT_FOUND: () =>
    new AppError("MEMBER_NOT_FOUND", "멤버를 찾을 수 없어요.", 404),
  NICKNAME_TAKEN: () =>
    new AppError("NICKNAME_TAKEN", "이미 사용 중인 닉네임이에요.", 409),
  ROOM_FULL: () => new AppError("ROOM_FULL", "정원이 모두 찼어요.", 409),
  INTERNAL_ERROR: () =>
    new AppError("INTERNAL_ERROR", "서버 오류가 발생했어요.", 500),
} as const;
```

```ts
// app/api/schedules/[id]/members/route.ts
import { Errors } from "@/lib/errors";

export async function POST(req: Request) {
  try {
    // ...
    throw Errors.NICKNAME_TAKEN();
  } catch (e) {
    if (e instanceof AppError) {
      return Response.json(
        { error: { code: e.code, message: e.message, details: e.details } },
        { status: e.status },
      );
    }
    const err = Errors.INTERNAL_ERROR();
    return Response.json(
      { error: { code: err.code, message: err.message } },
      { status: 500 },
    );
  }
}
```

### 클라이언트

```ts
// lib/api.ts — fetch 래퍼
const res = await fetch('/api/schedules/.../members', { method: 'POST', ... });

if (!res.ok) {
  const { error } = await res.json();

  if (error.code === 'NICKNAME_TAKEN') {
    setError('nickname', { message: error.message }); // RHF
    return;
  }
  // 그 외 공통 처리
  throw new Error(error.message);
}
```

---

## 관련 문서

- [기술 스택](../tech-stack.md)
- [DB 스키마](./schedule/db-schema.md)
