# 서비스 맵

> 각 서비스의 상세 정보는 해당 ai-context 디렉토리를 직접 참조할 것.
> 이 파일은 전체 조망용 요약이다.
> **이 목록은 `ai-context/` 디렉토리가 존재하는 서비스만 포함한다.**

## 서비스 목록

| 서비스명 | 한 줄 역할 | 기술 스택 | ai-context 경로 |
|----------|-----------|-----------|----------------|
| movie-service | 영화·상영관·좌석 관리 및 예약 처리 | Java 17, Spring Boot 3.5.9, MySQL, Redis, Kafka | `./movie-service/ai-context/` |
| todo-service | Todo 항목 CRUD REST API | Java 21, Spring Boot 4.0.3, MySQL | `./todo-service/ai-context/` |
| payment-service | 결제 처리 ⚠️ ai-context 미생성 | 미확인 | `./payment-service/ai-context/` |
| user-service | 사용자 관리 ⚠️ ai-context 미생성 | 미확인 | `./user-service/ai-context/` |

> ⚠️ payment-service, user-service는 `ai-context/` 디렉토리만 존재하며 내용이 생성되지 않았습니다.
> 해당 서비스에서 `/generate-ai-context`를 먼저 실행해주세요.

## 서비스별 핵심 도메인 엔티티

| 서비스명 | 핵심 엔티티 |
|----------|------------|
| movie-service | Movie, Theater, TheaterSeat, TheaterSeatGrade, Screen, Reservation, ReservationSeat |
| todo-service | Todo |
| payment-service | 미확인 (ai-context 없음) |
| user-service | 미확인 (ai-context 없음) |

## 서비스별 기술 스택 요약

| 서비스명 | DB | 메시지 브로커 | 캐시/인프라 |
|----------|-----|--------------|-----------|
| movie-service | MySQL | Kafka | Redis (Spring Cache + Redisson 분산 락) |
| todo-service | MySQL | - | - |
| payment-service | 미확인 | Kafka (추정) | 미확인 |
| user-service | 미확인 | 미확인 | 미확인 |
