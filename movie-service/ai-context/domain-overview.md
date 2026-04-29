# movie-service

## 서비스 역할
영화 예약 도메인을 담당하는 서비스다. 상영관(Theater), 영화(Movie), 상영 일정(Screen) 데이터를 관리하고,
사용자의 좌석 예약(Reservation) 생성 및 취소 흐름을 처리한다.
결제는 별도의 결제 서비스에서 처리하며, Kafka 이벤트를 통해 예약 상태를 동기화한다.

## 기술 스택
- Language: Java 17
- Framework: Spring Boot 3.5.9
- DB: MySQL 8.x (JPA/Hibernate)
- Cache: Redis (Spring Cache + Lettuce)
- Distributed Lock: Redisson (Redis 기반 MultiLock)
- Message Broker: Kafka (Spring Kafka)
- Build: Gradle

## 아키텍처 패턴
DDD 기반 레이어드 아키텍처를 사용한다.

- **domain**: 순수 비즈니스 로직. JPA 의존 없음. 모델, 리포지토리 인터페이스, 서비스, 이벤트 정의
- **infra**: 기술 구현체. JPA Entity, Repository 구현체, Kafka 설정/리스너, Redis 설정
- **presentation**: HTTP 요청/응답 처리. Controller, DTO, ExceptionHandler

도메인 모델(Reservation 등)과 JPA 엔티티(ReservationEntity 등)를 분리하여
영속성 계층이 도메인 로직에 영향을 주지 않도록 설계되어 있다.

## 패키지 구조
```
com.movieservice
├── domain/
│   ├── common/model/       # Money (Value Object)
│   ├── event/              # ReservationCreatedEvent (Kafka 발행용)
│   ├── movie/
│   │   ├── enums/          # MovieAgeRateType
│   │   └── model/          # Movie 도메인 객체
│   ├── reservation/
│   │   ├── enums/          # ReservationStatus
│   │   ├── event/          # ReservationCancelEvent
│   │   ├── model/          # Reservation, ReservationSeat 도메인 객체
│   │   ├── repository/     # ReservationRepository 인터페이스
│   │   └── service/        # ReservationService (핵심 비즈니스 로직)
│   ├── screen/
│   │   ├── model/          # Screen 도메인 객체
│   │   └── repository/     # ScreenRepository 인터페이스
│   └── theater/
│       ├── model/          # Theater, TheaterSeat, TheaterSeatGrade
│       └── repository/     # TheaterRepository 인터페이스
├── infra/
│   ├── config/             # JpaConfig, KafkaConfig, RedisCacheConfig, RedissonConfig
│   ├── event/
│   │   ├── DomainEvent     # 이벤트 마커 인터페이스
│   │   ├── EventPublisher  # Kafka 이벤트 발행
│   │   └── payment/
│   │       ├── dto/        # PaymentCompletedEvent, PaymentCancelledEvent
│   │       └── listener/   # PaymentEventListener (Kafka 수신)
│   ├── initializer/        # DataInitializer (초기 데이터 세팅)
│   ├── lock/               # DistributedLockHelper (Redisson MultiLock)
│   └── persistence/        # JPA Entity, JpaRepository, Repository 구현체
│       ├── common/entity/  # BaseEntity (createDateTime, modifyDateTime)
│       ├── movie/
│       ├── reservation/
│       ├── screen/
│       └── theater/
└── presentation/
    ├── common/
    │   ├── advice/         # ApiControllerAdvice (IllegalArgumentException → 500)
    │   └── response/       # ApiResponse<T> 공통 응답 래퍼
    └── reservation/
        ├── controller/     # ReservationController
        └── dto/            # 요청/응답 DTO (record)
```

## 핵심 비즈니스 규칙
- 예약 생성 시 동일 screenId + theaterSeatId 조합에 PENDING 또는 CONFIRMED 상태 예약이 있으면 중복 예약 불가
- 분산 락을 사용해 동시 예약 요청으로 인한 경쟁 조건 방지 (락 키: `lock:screenId:{id}:seatId:{id}`)
- 예약 상태 전환 규칙:
  - PENDING → CONFIRMED: 결제 완료(payment-completed 이벤트) 수신 시만 가능
  - PENDING/CONFIRMED → CANCELLING: 취소 요청 API 호출 시
  - CANCELLING → CANCELLED: 결제 취소 완료(payment-cancelled 이벤트) 수신 시
  - PENDING → CREATED_FAIL: 결제 생성 실패(payment-created-fail 이벤트) 수신 시
- Money 값 객체: 0 미만이면 IllegalArgumentException 발생

## 주의사항
<!-- 이 섹션은 개발자가 직접 보강해야 합니다 -->
- JPA ddl-auto가 `create`로 설정되어 있어 로컬 외 환경에서는 반드시 변경 필요
- DataInitializer가 있으나 어떤 초기 데이터를 생성하는지 확인 필요
- Kafka ErrorHandler가 `IllegalArgumentException`, `NullPointerException`은 재시도 없이 DLQ로 전송
- 알려진 기술 부채나 특이한 설계 결정 사항은 이 섹션에 직접 추가
