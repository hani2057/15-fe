# DB 스키마 설계

---

## 엔티티 관계 요약

```
Team 1 ─── N Member
Team 1 ─── 1 Schedule
Member N ─── 1 Schedule  (Member.team_id → Team.id → Schedule.team_id)
Member 1 ─── N Availability

(post-MVP)
Team 1 ─── N TeamMember ←── N User
Member.user_id → User.id 로 연결
```

---

## Team

팀 정보. 모든 팀 기능의 최상위 엔티티.

| 컬럼         | 타입            | 제약                    | 설명    |
| ------------ | --------------- | ----------------------- | ------- |
| `id`           | bigint unsigned | PK, AUTO_INCREMENT      |                        |
| `name`         | varchar(20)     | NOT NULL                | 팀 이름                |
| `member_count` | smallint        | NOT NULL                | 팀 정원 (고정값, 2~25) |
| `created_at`   | timestamptz     | NOT NULL, DEFAULT now() |                        |

**인덱스**
- PK: `id`

---

## Member

팀에 등록된 참여자. MVP에서는 쿠키 기반 익명 참여자.

| 컬럼         | 타입                    | 제약                       | 설명                                      |
| ------------ | ----------------------- | -------------------------- | ----------------------------------------- |
| `id`         | bigint unsigned         | PK, AUTO_INCREMENT         |                                           |
| `token`      | uuid                    | NOT NULL, UNIQUE           | 쿠키 인증용 식별자. 생성 시 자동 발급.    |
| `team_id`    | bigint unsigned         | NOT NULL, FK→Team.id       |                                           |
| `user_id`    | bigint unsigned         | NULLABLE, FK→User.id       | MVP에서는 null. 로그인 기능 추가 시 연결. |
| `nickname`   | varchar(5)              | NOT NULL                   | 팀 내 고유 닉네임 (최대 5자)              |
| `role`       | enum('leader','member') | NOT NULL, DEFAULT 'member' | 팀장은 항상 1명                           |
| `submitted`  | boolean                 | NOT NULL, DEFAULT false    | 슬롯 제출 완료 여부                       |
| `created_at` | timestamptz             | NOT NULL, DEFAULT now()    | 입장 순 정렬 기준                         |

**인덱스**
- PK: `id`
- UNIQUE: `token`
- UNIQUE: `(team_id, nickname)` — 팀 내 닉네임 중복 방지
- INDEX: `team_id`

**비즈니스 규칙**
- `role = 'leader'`인 멤버는 팀당 정확히 1명 유지
- 팀 생성 시 생성자 멤버의 `role`은 `'leader'`로 생성
- 팀장 위임 시: 기존 팀장 `role → 'member'`, 새 팀장 `role → 'leader'` (트랜잭션)
- [일정 등록] 클릭 시 `submitted → true`
- 쿠키에는 `token` 값(UUID)을 저장. 서버는 쿠키의 token으로 Member 조회 (integer PK 미노출)

---

## Schedule

팀의 일정 조율 세션.

| 컬럼               | 타입                    | 제약                       | 설명                      |
| ------------------ | ----------------------- | -------------------------- | ------------------------- |
| `id`               | varchar(10)             | PK                         | 랜덤 문자열 ID            |
| `team_id`          | bigint unsigned         | NOT NULL, FK→Team.id       |                           |
| `creator_id`       | bigint unsigned         | NOT NULL, FK→Member.id     | 생성자 member_id (참조용) |
| `date_list`        | date[]                  | NOT NULL                   | 조율 대상 날짜 목록       |
| `time_range_start` | smallint                | NOT NULL                   | 시작 시각 (0~23) |
| `time_range_end`   | smallint                | NOT NULL                   | 종료 시각 (1~24) |
| `status`           | enum('active','closed') | NOT NULL, DEFAULT 'active' | 방 상태          |
| `created_at`       | timestamptz             | NOT NULL, DEFAULT now()    |                           |

**인덱스**
- PK: `id`
- UNIQUE: `team_id` — Team당 Schedule 1개 강제
- INDEX: `team_id`

**비즈니스 규칙**
- Team과 Schedule은 1:1. 방 생성 시 두 엔티티를 함께 생성 (트랜잭션).
- `status`가 `closed`로 변경되는 조건: 팀장이 직접 종료하거나, `submitted_count === Team.member_count`
- `creator_id`는 역할 표시에 사용하지 않음 (역할은 `Member.role`로 판별). 데이터 정합성 참조용.

---

## Availability

멤버별 슬롯(날짜 × 시간) 가용 여부.

| 컬럼          | 타입            | 제약                       | 설명                      |
| ------------- | --------------- | -------------------------- | ------------------------- |
| `id`          | bigint unsigned | PK, AUTO_INCREMENT         |                           |
| `member_id`   | bigint unsigned | NOT NULL, FK→Member.id     |                           |
| `schedule_id` | varchar(10)     | NOT NULL, FK→Schedule.id   | 쿼리 편의를 위해 비정규화 |
| `date`        | date            | NOT NULL                   | 슬롯 날짜                 |
| `hour`        | smallint        | NOT NULL                   | 슬롯 시작 시각 (0~23)     |
| `available`   | boolean         | NOT NULL                   | 가능 여부                 |

**인덱스**
- PK: `id`
- UNIQUE: `(member_id, date, hour)` — 멤버별 슬롯 중복 방지
- INDEX: `(schedule_id, date, hour)` — 공통 가능 시간 집계 쿼리용

**비즈니스 규칙**
- 슬롯 제출 시 `date_list × time_range` 전체 슬롯을 upsert (부분 전송 없음)
- 범위 확장 시: 새 슬롯은 `available = false`로 초기화, 기존 멤버 `Member.submitted → false`로 리셋
- 범위 축소 시: 제거된 날짜/시간의 슬롯 데이터 삭제

---

## 결과 집계 로직 (서버)

`GET /api/schedules/[id]/result` 응답 생성 시:

```
submitted = true인 Member 집합 S (|S| = submitted_count, 전체 = Team.member_count)

commonSlot 조건:
  해당 슬롯에서 S의 모든 멤버가 available = true
  → COUNT(available=true) = submitted_count

nearMissSlot(unavailable_count=N) 조건:
  해당 슬롯에서 S 중 정확히 N명만 available = false
  → COUNT(available=false) = N
```
