# Todo Service — Domain Overview

## 서비스 역할

Todo 항목의 생성·조회·수정·삭제 및 통계를 제공하는 단일 도메인 REST API 서비스.
접근 주체별로 세 가지 API 그룹(공개/내부/외부)을 분리하여 노출한다.

## 기술 스택

| 분류 | 기술 |
|------|------|
| Language | Java 21 |
| Framework | Spring Boot 4.0.3 |
| Web | Spring Web MVC |
| Persistence | Spring Data JPA + Hibernate |
| Database | MySQL 8.x |
| Boilerplate | Lombok |
| Test | JUnit 5 |

## 아키텍처 패턴

- **표준 3-레이어 MVC**: Controller → Service → Repository
- **DTO 변환 레이어**: 엔티티를 직접 노출하지 않고 Request/Response DTO로 경계 분리
- **읽기 전용 트랜잭션 최적화**: `@Transactional(readOnly = true)` 기본, 변경 연산만 `@Transactional`
- **글로벌 예외 핸들러**: `@RestControllerAdvice`로 일관된 에러 응답
- **선택적 필드 업데이트**: `fields` 파라미터 기반 PATCH 지원 (ExternalTodoController)

## 패키지 구조

```
com.example.review
├── ReviewApplication.java          # 진입점
├── controller/
│   ├── TodoController.java         # /api/todo-list — 공개 API
│   ├── InternalTodoController.java # /internal/api/todo-list — 내부 API
│   └── ExternalTodoController.java # /external/api/todo-list — 외부 API + 통계
├── report/
│   └── controller/
│       └── TodoReportController.java  # /api/reports/todos — 리포트 API (별도 도메인)
├── service/
│   └── TodoService.java            # 비즈니스 로직
├── repository/
│   └── TodoRepository.java         # JpaRepository 확장
├── entity/
│   ├── Todo.java                   # 핵심 엔티티 + TodoStatus Enum
│   └── TodoCategory.java           # 카테고리 Enum (PERSONAL/WORK/STUDY/OTHER)
├── dto/
│   ├── request/
│   │   ├── TodoCreateRequest.java
│   │   ├── TodoCategorizedCreateRequest.java
│   │   └── TodoUpdateRequest.java
│   └── response/
│       └── TodoResponse.java
└── exception/
    └── GlobalExceptionHandler.java
```

## 핵심 비즈니스 규칙

1. **Todo 상태 흐름**: `TODO` → `IN_PROGRESS` → `DONE` (강제 순서 없음, 자유 전환)
2. **부분 업데이트**: `PATCH /external/api/todo-list/{id}?fields=title,status` 형태로 특정 필드만 업데이트 가능. `fields` 미지정 시 전체 업데이트
3. **null-safe 업데이트**: `Todo.update()`는 null 값을 무시하므로 요청에 없는 필드는 변경되지 않음
4. **통계 집계**: `total`, `done`, `pending(= total - done)` 카운트를 `status`, `minPriority` 필터 조합으로 계산 (DB가 아닌 메모리 내 집계). 삭제된 항목은 집계에서 제외됩니다.
5. **존재하지 않는 Todo 접근**: `IllegalArgumentException` → GlobalExceptionHandler가 HTTP 404로 변환
6. **스키마 관리**: `ddl-auto: update` — 엔티티 변경 시 자동 DDL 반영 (프로덕션 배포 시 주의 필요)
7. **카테고리 분류**: `POST /external/api/todo-list/categorized`로 생성 시 `TodoCategory`(PERSONAL/WORK/STUDY/OTHER) 지정 가능. 현재 요청 필드(title/content/dueDate/priority/category) 모두 service 레이어에 미전달(향후 확장 예정)
8. **키워드 검색**: `GET /external/api/todo-list/find?keyword=...&status=...` — 제목·내용 대소문자 무시 포함 검색. `status` 필터와 조합 가능하며 메모리 내 스트림 필터링으로 처리
9. **외부 API 감사 파라미터**: `GET /external/api/todo-list/{id}` 요청 시 `X-Caller-Id` 헤더(필수)와 `includeDeleted` 쿼리 파라미터를 수신하나 현재 service 레이어에는 미전달 (향후 감사 로그·soft delete 확장 예정)
10. **일일 리포트**: `GET /api/reports/todos/daily?status=...` — 특정 상태의 Todo 건수를 집계해 `{"total": N}` 형태로 반환. `report` 패키지에 속하며 별도 도메인(`report.example.com`) 사용
11. **우선순위 분포 리포트**: `GET /api/reports/todos/priority-distribution` — 전체 Todo의 우선순위별 건수를 `Map<Integer, Long>` 형태로 반환. `priority` null 항목은 집계 제외. 별도 도메인(`report.example.com`) 사용
