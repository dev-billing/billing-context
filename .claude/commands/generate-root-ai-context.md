---
name: generate-root-ai-context
description: |
  모노레포 또는 멀티레포 환경에서 각 서비스(레포)의 .claude/ai-context/를 읽어
  루트 레벨의 .claude/ai-context/를 자동 생성하는 스킬.
  "루트 ai-context 만들어줘", "전체 시스템 컨텍스트 생성해줘", "루트 CLAUDE.md 세팅해줘",
  "마스터 컨텍스트 생성해줘" 등의 요청 시 사용한다.
  각 서비스에 generate-ai-context 스킬로 생성된 ai-context가 이미 존재하는 상황을 전제로 한다.
---

# Generate Root AI Context

각 서비스 레포의 `.claude/ai-context/`를 읽어, 루트 레벨의 `.claude/ai-context/`를 생성한다.
루트 ai-context는 Master Agent가 업무 요청을 받았을 때 **어느 서비스를 봐야 하는지 즉시 판단**하고,
**해당 서비스의 ai-context로 깊이 진입**할 수 있는 네비게이션 허브 역할을 한다.

## 생성할 파일 목록

```
.claude/                              ← 루트
└── ai-context/
    ├── service-map.md                # 전체 서비스 목록, 역할, ai-context 경로
    ├── dependency-graph.md           # 서비스 간 호출/이벤트 의존관계
    ├── interface-contracts.json      # 서비스 간 공유 API·이벤트 인터페이스 집약
    └── routing-guide.md             # 업무 유형별 → 관련 서비스 매핑 가이드
CLAUDE.md                            ← 루트 CLAUDE.md (Master Agent 진입점)
```

---

## 실행 절차

### Step 1: 서비스 목록 탐색

루트 디렉토리 바로 아래의 서브 디렉토리를 전수 조사하여, `.claude/ai-context/`가 존재하는/ 서비스만 수집한다.

```bash
# 루트 바로 아래 디렉토리 목록 전체 확인
find . -maxdepth 1 -mindepth 1 -type d | sort

# ai-context 디렉토리를 가진 서비스만 필터링
find . -maxdepth 1 -mindepth 1 -type d \
  | while read dir; do
      if [ -d "$dir/.claude/ai-context" ]; then
        echo "✅ $dir"
      else
        echo "⏭️  $dir"
      fi
    done
```

**등록 기준 — 반드시 준수할 것:**
- `.claude/ai-context/` 디렉토리가 존재하는 서비스만 루트 ai-context에 등록한다
- `.claude/ai-context/`가 없는 서비스는 존재 자체를 루트 ai-context의 어떤 파일에도 기재하지 않는다
- 단, 탐색 결과 보고에서는 제외된 서비스 목록을 사용자에게 한 번 보여준다

탐색 결과를 아래 형식으로 사용자에게 보여준다. 확인 후 바로 다음 단계로 진행한다.

```
## 서비스 탐색 결과

등록 대상 (ai-context 있음):
  ✅ user-service      → ./user-service/.claude/ai-context/
  ✅ order-service     → ./order-service/.claude/ai-context/
  ✅ payment-service   → ./payment-service/.claude/ai-context/

제외 (ai-context 없음 — 루트 ai-context에 등록하지 않음):
  ⏭️  gateway          → .claude/ai-context/ 없음
  ⏭️  frontend         → .claude/ai-context/ 없음

제외된 서비스를 등록하려면 해당 레포에서 먼저 /generate-ai-context를 실행해주세요.
```

---

### Step 2: 각 서비스 ai-context 수집

발견된 서비스마다 아래 파일을 순서대로 읽는다.
파일이 없으면 스킵하고 기록만 남긴다.

읽는 순서:
1. `domain-overview.md` — 서비스 역할, 기술 스택, 패키지 구조
2. `api-spec.json` — 노출하는 엔드포인트 목록
3. `kafka-spec.json` — 발행/구독 이벤트 목록
4. `external-integration.md` — 외부 서비스 호출 정보
5. `data-model.md` — 핵심 엔티티명만 추출 (상세는 각 서비스 파일 참조)

각 서비스 수집 완료 시 진행 상황을 보고한다.

```
✅ user-service 수집 완료
   - API 엔드포인트: 8개
   - 이벤트 발행: 1개 (user.registered)
   - 이벤트 구독: 없음
   - 외부 연동: SMS API, S3

✅ order-service 수집 완료
   - API 엔드포인트: 12개
   - 이벤트 발행: 2개 (order.completed, order.cancelled)
   - 이벤트 구독: 1개 (payment.confirmed)
   - 외부 연동: 없음
```

---

### Step 3: 파일 생성

수집된 정보를 기반으로 파일을 아래 순서로 생성한다.

**① dependency-graph.md 먼저 생성**
서비스 간 관계를 먼저 정리해야 이후 파일 작성이 정확해진다.
- 각 서비스의 `api-spec.json`에서 다른 서비스를 호출하는 패턴 파악
- `kafka-spec.json`의 produces/consumes 토픽을 매핑하여 이벤트 흐름 구성

**② interface-contracts.json 생성**
서비스 간 직접 연결되는 인터페이스만 추출한다.
- A 서비스가 발행하고 B 서비스가 구독하는 이벤트
- A 서비스가 B 서비스의 API를 직접 호출하는 경우

**③ service-map.md 생성**
전체 서비스 요약표 및 ai-context 파일 경로를 정리한다.

**④ routing-guide.md 생성**
업무 요청 유형별로 어느 서비스를 봐야 하는지 매핑한다.
domain-overview.md의 핵심 비즈니스 규칙과 api-spec.json의 엔드포인트를 분석하여 작성한다.

**⑤ 루트 CLAUDE.md 생성 또는 덮어쓰기**
모든 파일이 완성된 후 아래 절차로 처리한다.

```bash
# 루트 CLAUDE.md 존재 여부 확인
ls CLAUDE.md 2>/dev/null && echo "EXISTS" || echo "NOT_FOUND"
```

**CLAUDE.md가 없는 경우 — 신규 생성**
아래 `루트 CLAUDE.md 포맷`에 따라 새로 작성한다.

**CLAUDE.md가 이미 있는 경우 — 덮어쓰기**
기존 파일을 읽어 아래 두 가지를 확인한 후 덮어쓴다.

1. `<!-- 이 섹션은 개발자가 직접 관리합니다 -->` 주석이 있는 섹션을 찾아 내용을 메모해둔다
2. 포맷에 따라 새로 작성하되, 메모해둔 개발자 관리 섹션의 내용은 그대로 이식한다
3. 개발자 관리 섹션이 없었다면 포맷의 placeholder를 그대로 둔다

처리 결과를 사용자에게 보고한다.
```
✅ CLAUDE.md 신규 생성 완료
또는
♻️  CLAUDE.md 덮어쓰기 완료 (기존 개발자 관리 섹션 내용 보존)
```

---

### Step 4: 검증

생성 완료 후 아래를 자동으로 검증한다.

**이벤트 정합성 검증**
- `interface-contracts.json`에 기록된 모든 토픽이 producer와 consumer 양쪽에 모두 존재하는지 확인
- 한쪽만 있는 토픽(발행만 하고 아무도 안 받는 이벤트, 반대의 경우)을 찾아 사용자에게 알림

**API 호출 정합성 검증**
- A 서비스가 B 서비스를 호출한다고 기록되어 있을 때, B 서비스의 api-spec.json에 해당 엔드포인트가 실제로 존재하는지 확인

검증 결과 보고:

```
⚠️ 정합성 이슈 발견:
   - 토픽 'inventory.decreased': order-service가 구독하지만 발행하는 서비스 없음
   - 토픽 'user.deleted': user-service가 발행하지만 구독하는 서비스 없음
   → interface-contracts.json에 WARNING 플래그로 기록했습니다
   → 실제 코드를 확인하거나 누락된 서비스의 ai-context를 먼저 생성해주세요
```

---

## 파일별 포맷

### 1. service-map.md

```markdown
# 서비스 맵

> 각 서비스의 상세 정보는 해당 ai-context 디렉토리를 직접 참조할 것.
> 이 파일은 전체 조망용 요약이다.
> **이 목록은 `.claude/ai-context/`가 존재하는 서비스만 포함한다.**
> 목록에 없는 레포는 아직 ai-context가 생성되지 않은 것이며, 루트 컨텍스트의 분석 범위에서 제외된다.

## 서비스 목록

| 서비스명 | 한 줄 역할 | 기술 스택 | ai-context 경로 |
|----------|-----------|-----------|----------------|
| user-service | 회원 가입·로그인·프로필 관리 | Spring Boot, PostgreSQL | `./user-service/.claude/ai-context/` |
| order-service | 주문 생성·조회·취소 | Spring Boot, PostgreSQL, Kafka | `./order-service/.claude/ai-context/` |
| payment-service | 결제 처리·환불 | Spring Boot, PostgreSQL, Kafka | `./payment-service/.claude/ai-context/` |
| notification-service | 이메일·SMS 알림 발송 | Spring Boot, Redis | `./notification-service/.claude/ai-context/` |

## 서비스별 핵심 도메인 엔티티

| 서비스명 | 핵심 엔티티 |
|----------|------------|
| user-service | User, UserGrade |
| order-service | Order, OrderItem |
| payment-service | Payment, Refund |
| notification-service | NotificationLog |

## 서비스별 기술 스택 요약

| 서비스명 | DB | 메시지 브로커 | 외부 연동 |
|----------|-----|--------------|----------|
| user-service | PostgreSQL | - | SMS API, S3 |
| order-service | PostgreSQL | Kafka | - |
| payment-service | PostgreSQL | Kafka | PG사 API |
| notification-service | - | Kafka | SendGrid, Kakao |
```

---

### 2. dependency-graph.md

```markdown
# 서비스 간 의존관계

## 호출 흐름도

```
[클라이언트]
│
▼
[API Gateway]
│
├──▶ user-service      (회원 관련 요청)
│
├──▶ order-service     (주문 관련 요청)
│         │
│         ├──REST──▶ user-service    (주문 시 회원 검증)
│         └──REST──▶ payment-service (결제 요청)
│
└──▶ payment-service   (결제 관련 요청)
```

## 이벤트 흐름도

```
user-service
└─[발행] user.registered ──▶ notification-service (환영 메일 발송)

order-service
└─[발행] order.completed  ──▶ payment-service      (결제 처리 시작)
└─[발행] order.completed  ──▶ notification-service (주문 완료 알림)
└─[발행] order.cancelled  ──▶ notification-service (취소 알림)
└─[구독] payment.confirmed ◀── payment-service    (주문 상태 → PAID 변경)

payment-service
└─[발행] payment.confirmed ──▶ order-service       (결제 완료 통보)
```

## 서비스 간 영향 매트릭스

변경 시 영향을 주는 방향을 나타낸다 (행 → 열 방향으로 영향).

|                    | user-service | order-service | payment-service | notification-service |
|--------------------|:---:|:---:|:---:|:---:|
| user-service       | -   | ●   | -   | ●   |
| order-service      | -   | -   | ●   | ●   |
| payment-service    | -   | ●   | -   | -   |
| notification-service | - | -   | -   | -   |

● = 직접 영향 (API 호출 또는 이벤트 연결)
```

---

### 3. interface-contracts.json

서비스 간 직접 연결된 인터페이스만 집약한다.
상세 스펙은 각 서비스의 `api-spec.json`, `kafka-spec.json`을 참조.

```json
{
  "system": "시스템명",
  "generatedAt": "2025-01-01",
  "restContracts": [
    {
      "caller": "order-service",
      "callee": "user-service",
      "endpoint": "GET /api/v1/users/{id}/verify",
      "purpose": "주문 시 회원 유효성 검증",
      "callerClass": "UserServiceClient",
      "calleeClass": "UserController",
      "warning": null
    },
    {
      "caller": "order-service",
      "callee": "payment-service",
      "endpoint": "POST /api/v1/payments",
      "purpose": "주문 완료 시 결제 요청",
      "callerClass": "PaymentServiceClient",
      "calleeClass": "PaymentController",
      "warning": null
    }
  ],
  "eventContracts": [
    {
      "topic": "order.completed",
      "producer": "order-service",
      "consumers": ["payment-service", "notification-service"],
      "messageFields": {
        "orderId": "Long",
        "userId": "Long",
        "totalAmount": "BigDecimal",
        "completedAt": "LocalDateTime"
      },
      "warning": null
    },
    {
      "topic": "payment.confirmed",
      "producer": "payment-service",
      "consumers": ["order-service"],
      "messageFields": {
        "paymentId": "Long",
        "orderId": "Long",
        "paidAmount": "BigDecimal"
      },
      "warning": null
    },
    {
      "topic": "inventory.decreased",
      "producer": null,
      "consumers": ["order-service"],
      "messageFields": {},
      "warning": "producer 서비스 없음 — ai-context가 없는 서비스가 발행하거나 누락 가능"
    }
  ]
}
```

---

### 4. routing-guide.md

업무 요청이 들어왔을 때 Master Agent가 어느 서비스의 ai-context를 읽어야 하는지 판단하는 가이드.
단순 키워드 매핑이 아니라, 실제 도메인 분석을 기반으로 작성한다.

```markdown
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
| 회원, 사용자, 로그인, 인증, 비밀번호, 프로필 | user-service | - |
| 주문, 장바구니, 배송, 주문 상태 | order-service | payment-service, notification-service |
| 결제, 환불, PG, 카드 | payment-service | order-service |
| 알림, 이메일, SMS, 푸시 | notification-service | - |
| 회원 탈퇴 | user-service | order-service, notification-service |

## 업무 유형별 주의사항

### 신규 기능 추가
- 새 API가 생기면 → 해당 서비스의 api-spec.json 확인 후 중복 엔드포인트 여부 체크
- 새 이벤트가 생기면 → interface-contracts.json에 반드시 추가 필요

### 기존 기능 수정
- API 응답 필드 변경 → 해당 API를 호출하는 서비스(restContracts 기준)도 함께 확인
- 이벤트 메시지 필드 변경 → 해당 토픽의 모든 consumer 서비스도 함께 수정

### 데이터 모델 변경
- 엔티티 필드 추가/삭제 → DB 마이그레이션 스크립트 필요 여부 사용자에게 확인
- 외래키 관계 변경 → 연관 서비스 API 스펙에도 영향 가능

### 인프라/공통 변경
- 인증 방식 변경 → 전 서비스 영향. 모든 서비스의 ai-context 확인
- 공통 라이브러리 변경 → service-map.md의 전체 서비스 목록 대상 영향 분석
```

---

### 5. 루트 CLAUDE.md

```markdown
# [시스템명] — Master Agent

## 이 파일의 목적
업무 요청이 들어오면 이 파일을 진입점으로 삼아
루트 ai-context → 관련 서비스 ai-context 순서로 읽고 작업을 수행한다.

---

## 루트 AI Context 읽는 순서

업무 요청을 받으면 반드시 아래 순서로 읽는다.

```
1. .claude/ai-context/routing-guide.md     → 어느 서비스가 관련됐는지 판단
2. .claude/ai-context/service-map.md       → 관련 서비스의 ai-context 경로 확인
3. .claude/ai-context/dependency-graph.md  → 연쇄 영향 서비스 파악
4. .claude/ai-context/interface-contracts.json → 서비스 간 인터페이스 확인
5. [관련 서비스]/.claude/ai-context/       → 실제 도메인·API·데이터 상세 파악
```

> 4번까지 읽은 것만으로 작업을 시작하지 말 것.
> 반드시 5번의 해당 서비스 ai-context까지 읽은 후 작업 계획을 수립한다.

---

## 업무 처리 워크플로우

### Step 1 — 라우팅
routing-guide.md를 읽고 관련 서비스를 특정한다.

### Step 2 — 상세 파악
특정된 서비스들의 ai-context를 읽는다.
읽어야 할 파일:
- `domain-overview.md` — 서비스 역할과 비즈니스 규칙
- `api-spec.json` — 수정 대상 엔드포인트 확인
- `data-model.md` — 관련 엔티티 및 필드 확인
- `kafka-spec.json` — 이벤트 연동 확인 (해당 시)

### Step 3 — 계획 수립 및 보고
아래 형식으로 작업 계획을 사용자에게 먼저 보여준다. 승인 전에 코드를 수정하지 않는다.

```
## 영향 분석 결과

### 변경 대상 서비스 및 파일
- user-service
  - UserController.java — POST /api/v1/users/verify 엔드포인트 추가
  - UserService.java — 검증 비즈니스 로직 추가

- order-service
  - OrderService.java — 회원 검증 호출 로직 추가

### 서비스 간 인터페이스 변경 여부
- 없음 / 있음 (구체적으로 명시)

### 수정하지 않는 서비스
- payment-service, notification-service — 영향 없음

### 사용자 확인이 필요한 사항
- DB 마이그레이션 스크립트 필요 여부
- 배포 순서 제약 여부
```

### Step 4 — Sub-agent 위임 (승인 후)
승인 후 서비스별로 Task를 생성하여 병렬 실행한다.
각 Task에 포함할 내용:
- 작업 디렉토리 (해당 서비스 루트 경로)
- 수정할 파일과 구체적인 변경 내용
- 인터페이스 변경이 있을 경우 연관 서비스와의 정합성 주의사항

### Step 5 — 완료 보고
```
## 작업 완료 보고

### 수정된 파일 목록
- user-service/src/.../UserController.java
- order-service/src/.../OrderService.java

### 추가 조치 필요사항
- [ ] DB 마이그레이션 스크립트 작성 (담당자 직접)
- [ ] interface-contracts.json 업데이트 (신규 인터페이스 추가 시)
```

---

## 공통 컨벤션
> 서비스별 상세 컨벤션은 각 서비스의 CLAUDE.md 참조.

- API 경로: `/api/v1/` 접두사 공통 사용
- 인증: Gateway 발급 JWT 공통 사용
- 이벤트 토픽 네이밍: `{도메인}.{동사과거형}` (예: `order.completed`)
- 에러 응답 포맷: `{ "code": "...", "message": "..." }`

<!-- 이 섹션은 개발자가 직접 관리합니다 -->
```

---

## 업데이트 모드

"루트 ai-context 업데이트해줘" 요청 시 전체를 새로 만들지 않는다.

1. 루트 바로 아래 디렉토리를 전수 조사하여 `.claude/ai-context/`가 생긴 서비스가 있는지 확인한다
2. 각 서비스의 ai-context 파일의 최종 수정 시각을 확인한다
3. 마지막 루트 ai-context 생성 시각(`generatedAt`)과 비교한다
4. 아래 세 가지 경우를 처리한다:
  - **신규 등록** — 이전엔 ai-context가 없었다가 새로 생긴 서비스 → 모든 루트 ai-context 파일에 추가
  - **내용 갱신** — 기존 등록 서비스 중 ai-context가 변경된 서비스 → 변경된 부분만 반영
  - **등록 유지** — ai-context가 여전히 없는 서비스 → 루트 ai-context에 추가하지 않음 (이전과 동일)
5. `<!-- 이 섹션은 개발자가 직접 관리합니다 -->` 주석이 있는 섹션은 보존한다

변경 내역을 사용자에게 보고한다.

```
## 업데이트 완료

변경 감지된 서비스: order-service, payment-service
  - interface-contracts.json: settlement.requested 이벤트 신규 추가
  - dependency-graph.md: payment-service → settlement-service 연결 추가
  - routing-guide.md: '정산' 키워드 → settlement-service 매핑 추가

변경 없음: user-service, notification-service
```

---

## 품질 기준

- service-map.md는 새 팀원이 3분 안에 전체 서비스 구조를 파악할 수 있는 수준으로 작성한다
- interface-contracts.json은 실제 코드의 서비스 간 연결과 1:1 대응해야 한다
- routing-guide.md의 도메인 매핑은 단순 단어 매핑이 아닌 실제 비즈니스 개념 기반으로 작성한다
- 코드에서 확인되지 않는 내용은 추측하지 않고 `warning` 또는 placeholder로 표시한다
- 루트 ai-context 전체 분량은 CLAUDE.md 포함 400줄 이내를 목표로 한다