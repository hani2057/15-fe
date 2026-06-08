# 기술 스택 설계

---

## 1. MVP 스택 (확정)

| 영역             | 선택                        | 비고                         |
| ---------------- | --------------------------- | ---------------------------- |
| 프레임워크       | Next.js 14 App Router       | 프론트 + 백엔드 통합         |
| 언어             | TypeScript                  |                              |
| DB               | PostgreSQL                  |                              |
| DB 호스팅        | Neon                        | 서버리스, Vercel 공식 파트너 |
| ORM              | Prisma                      | 타입 자동 생성, 마이그레이션 |
| 배포             | Vercel                      |                              |
| 스타일링         | Montage (wds) + CSS Modules | Emotion 기반 디자인 시스템   |
| 날짜             | dayjs                       |                              |
| 폼 / 유효성 검사 | React Hook Form + Zod       |                              |
| 패키지 매니저    | pnpm                        |                              |
| 인증 (MVP)       | 쿠키 기반 커스텀            | OAuth 미포함                 |
| 인증 (post-MVP)  | NextAuth.js                 | Kakao 등 OAuth 추가 시       |

---

## 2. 백엔드 구조 방침

MVP에서는 Next.js Route Handlers(`app/api/`)를 백엔드로 사용한다.  
단, **나중에 NestJS로 이전하기 쉽도록** 진입점과 로직을 레이어로 분리한다.

```
app/
  api/
    schedules/
      route.ts          ← 진입점만 (요청 파싱, 응답 반환)

services/
  scheduleService.ts    ← 비즈니스 로직

repositories/
  scheduleRepository.ts ← DB 접근 (Prisma 쿼리)
```

**규칙:**

- Route Handler는 요청 파싱과 응답 반환만 담당. 비즈니스 로직 포함 금지.
- 비즈니스 로직은 `services/`에 작성.
- Prisma 쿼리는 `repositories/`에 작성. `services/`에서 직접 Prisma Client 호출 금지.

이 구조를 유지하면 추후 NestJS 이전 시 `services/`와 `repositories/` 코드를 그대로 옮길 수 있다.

---

## 3. CI/CD

GitHub Actions로 PR 및 main 브랜치 푸시 시 자동 검사.

```
lint → type-check → build
```

Vercel은 main 브랜치 머지 시 자동 배포. PR 생성 시 preview 배포 자동 생성.

---

## 4. 향후 전환 계획 (post-MVP)

### 단계 1 — NestJS 이전

Next.js Route Handlers → NestJS로 API 이전.  
`services/`, `repositories/` 레이어는 그대로 이전 가능.

NestJS를 선택하는 이유: 의존성 주입, 모듈/컨트롤러/서비스 구조, Guard/Interceptor/Pipe 패턴이 현업 백엔드 표준에 가깝고 Spring과 유사한 구조라 확장성과 학습 효과가 높음.

### 단계 2 — 모노레포 전환

```
apps/
  web/    ← Next.js (프론트엔드만)
  api/    ← NestJS (백엔드)

packages/
  types/  ← 공통 타입 (Request/Response DTO 등)
  utils/  ← 공통 유틸
```

패키지 매니저 pnpm workspace 기반으로 구성.

---

## 관련 문서

- [DB 스키마](./schedule/db-schema.md)
