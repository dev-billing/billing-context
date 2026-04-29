# 외부 연동

## 내부 서비스 호출 (FeignClient)

없음. movie-service는 다른 서비스를 직접 HTTP 호출하지 않는다.
서비스 간 통신은 Kafka 이벤트를 통해서만 이루어진다.

---

## 인프라 연동

### MySQL
- 용도: 주요 도메인 데이터 영속화 (영화, 상영관, 예약)
- DB 이름: `movie`
- Host: `localhost:3306` (application.yml)
- JPA ddl-auto: `create` (로컬 기준 — 운영 환경 설정 별도 확인 필요)

### Redis
- 용도 1: 캐시 (Spring Cache, Lettuce 클라이언트)
  - 기본 TTL: 1시간
  - 캐시별 개별 설정 없음 (기본값만 사용 중)
- 용도 2: 분산 락 (Redisson MultiLock)
  - 락 키 패턴: `lock:screenId:{screenId}:seatId:{theaterSeatId}`
  - 락 대기 시간: 3초, 점유 시간: 5초
  - 좌석 중복 예약 방지에 사용
- Host: `localhost:6379` (application.yml)
- Command Timeout: 3초

### Kafka
- 용도: 서비스 간 비동기 이벤트 통신 (결제 서비스 ↔ 영화 서비스)
- Broker: `localhost:9092` (application.yml)
- 상세 내용: `kafka-spec.json` 참조
