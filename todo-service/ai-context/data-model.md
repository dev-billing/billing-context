# Todo Service — Data Model

## 엔티티 관계

현재 단일 엔티티 구조이며 연관관계 없음.

```
┌────────────────────────────┐
│           todos            │
├────────────────────────────┤
│ id          BIGINT (PK AI) │
│ title       VARCHAR NOT NULL│
│ content     TEXT           │
│ status      VARCHAR NOT NULL│
│ due_date    DATE           │
│ priority    INT            │
│ created_at  DATETIME       │
│ updated_at  DATETIME       │
└────────────────────────────┘
```

## 엔티티 필드 정의

### Todo (`todos` 테이블)

| 필드 | 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|------|
| id | id | BIGINT | PK, AUTO_INCREMENT | 식별자 |
| title | title | VARCHAR | NOT NULL | 제목 |
| content | content | TEXT | NULLABLE | 상세 내용 |
| status | status | VARCHAR | NOT NULL, DEFAULT 'TODO' | 진행 상태 (Enum 문자열 저장) |
| dueDate | due_date | DATE | NULLABLE | 마감일 |
| priority | priority | INT | NULLABLE | 우선순위 (1~5 권장, 숫자가 높을수록 중요) |
| createdAt | created_at | DATETIME | 자동 생성 | 생성 시각 (`@CreationTimestamp`) |
| updatedAt | updated_at | DATETIME | 자동 갱신 | 수정 시각 (`@UpdateTimestamp`) |

## Enum

### TodoStatus

| 값 | 의미 |
|----|------|
| `TODO` | 미시작 (기본값) |
| `IN_PROGRESS` | 진행 중 |
| `DONE` | 완료 |

- DB 저장 방식: `EnumType.STRING` (값 그대로 문자열 저장)
- 기본값: `TODO`

## DTO 구조

### TodoCreateRequest

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| title | String | 사실상 필수 (엔티티 NOT NULL) | 제목 (최대 100자) |
| content | String | 선택 | 내용 |
| dueDate | LocalDate | 선택 | 마감일 (yyyy-MM-dd) |
| priority | Integer | 선택 | 우선순위 (1~5 권장, 숫자가 높을수록 중요) |
| tags | List\<String\> | 선택 | 태그 목록 (현재 미사용) |

### TodoUpdateRequest

| 필드 | 타입 | 설명 |
|------|------|------|
| title | String | 제목 |
| content | String | 내용 |
| status | TodoStatus | 상태 |
| dueDate | LocalDate | 마감일 |
| priority | Integer | 우선순위 (1~5 권장) |

### TodoResponse

| 필드 | 타입 | 설명 |
|------|------|------|
| id | Long | 식별자 |
| title | String | 제목 |
| content | String | 내용 |
| status | TodoStatus | 상태 |
| dueDate | LocalDate | 마감일 |
| priority | Integer | 우선순위 (1~5) |
| createdAt | LocalDateTime | 생성 시각 |
| updatedAt | LocalDateTime | 수정 시각 |
