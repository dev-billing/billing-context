# todo-service

## 서비스 역할
Todo 항목의 CRUD를 제공하는 단순 REST API 서비스다. 외부 호출(external), 내부 호출(internal), 일반(api) 세 가지 경로를 별도 컨트롤러로 분리하여 제공한다.

## 기술 스택
- Language: Java 21
- Framework: Spring Boot 4.0.3 (spring-boot-starter-webmvc, spring-boot-starter-data-jpa)
- DB: MySQL (mysql-connector-j)
- Lombok

## 아키텍처 패턴
전통적인 Layered Architecture (Controller → Service → Repository). 단일 도메인(Todo) 중심의 소규모 서비스.

## 패키지 구조

```
com.example.review
├── controller/         # REST API 엔드포인트 (3개 컨트롤러)
│   ├── TodoController          ← /api/todo-list (일반 용도)
│   ├── ExternalTodoController  ← /external/api/todo-list (외부 호출용)
│   └── InternalTodoController  ← /internal/api/todo-list (내부 호출용)
├── service/            # 비즈니스 로직 (TodoService)
├── entity/             # JPA 엔티티 (Todo, TodoStatus enum)
├── repository/         # 데이터 접근 계층 (TodoRepository extends JpaRepository)
├── dto/
│   ├── request/        # TodoCreateRequest, TodoUpdateRequest
│   └── response/       # TodoResponse
└── exception/          # GlobalExceptionHandler (@RestControllerAdvice)
```

## 핵심 비즈니스 규칙
- Todo 생성 시 status 기본값은 `TODO`
- 수정 시 null 필드는 변경하지 않음 (Partial Update)
- 존재하지 않는 ID 조회/수정/삭제 시 `IllegalArgumentException` → 404 응답
- `TodoCreateRequest.dueDate` 필드가 있으나 엔티티에 컬럼이 없어 **저장되지 않음** (버그 또는 미완성 기능)

## API 경로 설계 특이사항
세 컨트롤러가 동일한 `TodoService`를 공유하며 제공하는 기능이 겹침.

| 기능 | TodoController | ExternalTodoController | InternalTodoController |
|------|---------------|----------------------|----------------------|
| 전체 조회 | 없음 | GET /find | GET / |
| 단건 조회 | GET /get/{id} | GET /{id} | GET /get/{id} |
| 생성 | POST /insert | POST / | POST /insert |
| 수정 | PUT /update/{id} | PUT /{id} | PUT /update/{id} |
| 삭제 | 없음 | DELETE /{id} | DELETE /delete/{id} |

`ExternalTodoController`가 REST 네이밍 규칙에 가장 잘 부합하며, `TodoController`와 `InternalTodoController`는 동사를 URL에 포함하는 규칙 위반이 있음.

## 주의사항
<!-- 이 섹션은 개발자가 직접 보강해야 합니다 -->
- `TodoCreateRequest.dueDate` 필드와 `Todo` 엔티티 간 불일치 해소 필요
- 세 컨트롤러의 역할 분리 기준(external/internal/api)이 코드상 명확하지 않음 — 인증/인가 차이인지 확인 필요
- `ddl-auto: update`가 application.yaml에 설정되어 있어 운영 적용 전 확인 필요
