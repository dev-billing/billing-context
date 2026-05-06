# 서비스 간 의존관계

> 분석 기준: movie-service, todo-service ai-context 기반.
> payment-service · user-service는 ai-context 미생성 — 아래 관계는 movie-service kafka-spec.json 기준으로 역추론.

## 호출 흐름도

```
[클라이언트]
│
├──▶ movie-service      (영화·예약 관련 요청)
│         │  REST 직접 호출 없음 (모든 서비스 간 통신은 Kafka)
│         └──[Kafka]──▶ payment-service
│
├──▶ todo-service       (Todo CRUD 요청)
│         standalone — 타 서비스 의존 없음
│
├──▶ payment-service    (결제 관련 요청, 상세 미확인)
│         └──[Kafka]──▶ movie-service
│
└──▶ user-service       (사용자 관련, 상세 미확인)
```

## 이벤트 흐름도

```
movie-service
├─[발행] reservation-created   ──▶ payment-service  (예약 생성 → 결제 처리 시작)
└─[발행] reservation-cancelled ──▶ payment-service  (예약 취소 → 환불 처리 시작)

payment-service
├─[발행] payment-completed     ──▶ movie-service     (결제 완료 → 예약 상태 CONFIRMED)
├─[발행] payment-cancelled     ──▶ movie-service     (결제 취소 → 예약 상태 CANCELLED)
└─[발행] payment-created-fail  ──▶ movie-service     (결제 실패 → 예약 상태 CREATED_FAIL)

todo-service   : Kafka 연동 없음 (standalone)
user-service   : 미확인 (ai-context 없음)
```

## 서비스 간 영향 매트릭스

변경 시 영향을 주는 방향 (행 → 열 방향).

|                   | movie-service | todo-service | payment-service | user-service |
|-------------------|:---:|:---:|:---:|:---:|
| movie-service     | -   | -   | ●   | -   |
| todo-service      | -   | -   | -   | -   |
| payment-service   | ●   | -   | -   | -   |
| user-service      | ?   | -   | ?   | -   |

● = 직접 영향 (Kafka 이벤트 연결 확인됨)
? = ai-context 없어 미확인

## 배포 순서 주의사항

- movie-service ↔ payment-service: Kafka 토픽 기반 느슨한 결합 — 독립 배포 가능
- todo-service: 완전 독립 배포 가능
- Kafka 이벤트 메시지 포맷 변경 시 producer·consumer 동시 배포 계획 필요
