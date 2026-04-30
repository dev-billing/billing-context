# 업무 요청 라우팅 가이드

Master Agent는 업무 요청을 받으면 이 가이드를 먼저 읽고,
관련 서비스의 ai-context를 깊이 읽은 후 작업 계획을 수립한다.

## 라우팅 원칙

1. 요청에서 핵심 명사(도메인 개념)를 추출한다
2. 아래 도메인 → 서비스 매핑으로 1차 서비스를 특정한다
3. dependency-graph.md에서 연쇄 영향 서비스를 확인한다
4. 해당 서비스들의 ai-context를 읽고 실제 수정 범위를 판단한다

## 도메인 개념 → 서비스 매핑

| 도메인 개념 | 주담당 서비스 | 연쇄 확인 서비스 |
|------------|-------------|----------------|
| 영화, 영화 목록, 개봉일, 연령등급 | movie-service | - |
| 상영관, 상영 일정, 좌석, 좌석 등급, 좌석 가격 | movie-service | - |
| 예약, 예약 생성, 예약 취소, 예약 상태, 중복 예약 | movie-service | payment-service |
| 결제, 환불, 결제 완료, 결제 실패, 결제 취소 | payment-service | movie-service |
| 분산 락, 동시 예약, 경쟁 조건 | movie-service | - |
| Todo, 할 일, 작업 목록, 작업 상태, IN_PROGRESS | todo-service | - |
| 사용자, 회원, 유저, 로그인 | user-service | movie-service (userId 외래 참조) |

## 업무 유형별 주의사항

### 예약 흐름 변경
- 예약 생성/취소 로직 수정 시 Kafka 이벤트(`reservation-created`, `reservation-cancelled`) 스펙 변경 가능
- payment-service consumer와 정합성 필수 확인 (payment-service ai-context 생성 후)
- 분산 락 키 패턴(`lock:screenId:{id}:seatId:{id}`) 변경 시 동시성 제어 전체 검토

### 결제 이벤트 연동 변경
- `payment-completed`, `payment-cancelled`, `payment-created-fail` 이벤트 스펙 변경 시
  movie-service `PaymentEventListener`도 반드시 함께 수정
- interface-contracts.json의 messageFields 업데이트 필요

### Todo 기능 변경
- todo-service는 standalone — 타 서비스 영향 없음
- 세 컨트롤러(TodoController, ExternalTodoController, InternalTodoController)가 동일 서비스를 공유하므로 수정 시 세 경로 모두 확인
- `dueDate` 필드: `TodoCreateRequest`에만 존재, 엔티티에 없어 저장 안 됨 (알려진 버그)

### 신규 기능 추가
- 새 API 엔드포인트 추가 → 해당 서비스의 `api-spec.json` 업데이트
- 새 Kafka 이벤트 추가 → `interface-contracts.json` 업데이트
- DLQ 패턴: `<original-topic>.dlq` (movie-service 기준)

### 데이터 모델 변경
- `Reservation.userId` 변경 → user-service와 직접 연동 없음. 외래 참조 의미 변경만 확인
- Movie/Screen/Theater 스키마 변경 → movie-service 단독 영향

## 서비스별 진입 파일 경로

| 요청 유형 | 읽어야 할 파일 (우선순위 순) |
|----------|---------------------------|
| 영화·예약·상영관 | `movie-service/ai-context/domain-overview.md` → `api-spec.json` → `kafka-spec.json` → `data-model.md` |
| 결제·환불 | `interface-contracts.json` → `payment-service/ai-context/` (미생성, 생성 필요) |
| Todo | `todo-service/ai-context/domain-overview.md` → `api-spec.json` |
| 사용자 | `user-service/ai-context/` (미생성, 생성 필요) |
| 서비스 간 이벤트 | `interface-contracts.json` → `dependency-graph.md` |
