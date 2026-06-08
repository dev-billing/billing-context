# 서비스 맵

> 각 서비스의 상세 정보는 해당 ai-context 디렉토리를 직접 참조할 것.
> 이 파일은 전체 조망용 요약이다.
> **이 목록은 `ai-context/` 디렉토리에 실제 컨텍스트가 있는 서비스만 포함한다.**

## 서비스 목록

| 서비스명 | 한 줄 역할 | 기술 스택 | ai-context 경로 |
|----------|-----------|-----------|----------------|
| movie-service | 영화·상영관·좌석 관리 및 좌석 예약 처리 | Java 17, Spring Boot 3.5.9, MySQL, Redis, Kafka | `./movie-service/ai-context/` |
| todo-service | Todo 항목 CRUD REST API (공개/내부/외부 3그룹) | Java 21, Spring Boot 4.0.3, MySQL | `./todo-service/ai-context/` |

> ⚠️ **payment-service**, **user-service**는 `ai-context/`가 비어 있어 루트 컨텍스트 분석 범위에서 제외됩니다.
> 등록하려면 해당 서비스에서 `/generate-ai-context`를 먼저 실행하세요.

## 서비스별 핵심 도메인 엔티티

| 서비스명 | 핵심 엔티티 |
|----------|------------|
| movie-service | Movie, Theater, TheaterSeat, TheaterSeatGrade, Screen, Reservation, ReservationSeat |
| todo-service | Todo |

## 서비스별 기술 스택 요약

| 서비스명 | DB | 메시지 브로커 | 캐시/인프라 |
|----------|-----|--------------|-----------|
| movie-service | MySQL 8.x | Kafka | Redis (Spring Cache + Redisson 분산 락) |
| todo-service | MySQL 8.x | - | - |

## API 엔드포인트 요약

| 서비스명 | 엔드포인트 수 | 주요 경로 |
|----------|------------|----------|
| movie-service | 2개 | `POST /api/reservations`, `PATCH /api/reservations/{id}` |
| todo-service | 10개+ | `/api/todo-list`, `/internal/api/todo-list`, `/external/api/todo-list`, `/api/reports/todos` |

## 서비스별 ai-context 파일 목록

| 서비스명 | 파일 |
|----------|------|
| movie-service | `domain-overview.md`, `api-spec.json`, `kafka-spec.json`, `external-integration.md`, `data-model.md` |
| todo-service | `domain-overview.md`, `api-spec.json`, `external-integration.md`, `data-model.md` |

## 배포 URL 현황

| 서비스명 | Alpha | Real |
|----------|-------|------|
| todo-service | https://alpha-todo-service.com | https://todo-service.com |
| movie-service | 미확인 | 미확인 |

## CI/CD 자동화 현황

| 서비스명 | AI Context 동기화 |
|----------|-----------------|
| todo-service | GitHub Actions (`develop` 브랜치 push 시 자동 실행 → `billing-context` 레포 동기화) |
| movie-service | 미확인 |
