---
name: generate-ai-context
description: 프로젝트 소스코드를 분석하여 .claude/ai-context/ 디렉토리에 AI가 프로젝트를 이해하는 데 필요한 컨텍스트 문서들을 자동 생성하는 스킬.
   "ai-context 생성해줘", "프로젝트 컨텍스트 만들어줘", "이 프로젝트 분석해서 문서화해줘", "CLAUDE.md 세팅해줘", "ai-context 업데이트해줘" 등의 요청 시 사용한다.
   새 프로젝트 온보딩, 기존 프로젝트의 컨텍스트 초기 구축, 코드 변경 후 컨텍스트 동기화 등 모든 상황에 적용된다.
---

# Generate AI Context

프로젝트의 소스코드를 분석하여 `.claude/ai-context/` 아래에 구조화된 컨텍스트 문서들을 생성한다.
이 문서들은 Claude가 프로젝트를 빠르게 파악하고 정확하게 작업하기 위한 지식 기반이다.

---

## 실행 위치 결정

**모든 Step에 앞서 가장 먼저 수행한다.**

이 커맨드는 인자(디렉토리명) 유무에 따라 작업 기준 경로가 달라진다.

```bash
echo "$ARGUMENTS
```

**인자가 있는 경우** — `/generate-ai-context pay-api` 처럼 디렉토리명을 지정한 경우

1. 해당 디렉토리가 존재하는지 확인한다
   ```bash
   ls -d "$ARGUMENTS" 2>/dev/null && echo "EXISTS" || echo "NOT_FOUND"
   ```
2. 디렉토리가 없으면 즉시 중단하고 사용자에게 알린다
   ```
   ❌ 디렉토리를 찾을 수 없습니다: pay-api
      현재 위치에서 접근 가능한 디렉토리 목록:
      (find . -maxdepth 1 -mindepth 1 -type d 결과 출력)
   ```
3. 디렉토리가 있으면 해당 경로를 **작업 기준 경로($TARGET_DIR)**로 설정한다
   ```
   📁 작업 디렉토리: ./pay-api
      이 경로를 기준으로 ai-context를 생성합니다.
   ```

**인자가 없는 경우** — `/generate-ai-context` 만 입력한 경우

현재 디렉토리(`./`)를 **작업 기준 경로($TARGET_DIR)**로 설정하고 진행한다.

> 이하 모든 Step의 파일 탐색, 코드 분석, 파일 생성은 **$TARGET_DIR 기준**으로 수행한다.

---

## 프로젝트 유형 감지

Step 1(규모 판단)과 동시에 프로젝트 유형을 판별한다.

```bash
# API 프로젝트 감지
grep -rl "@RestController\|@Controller" $TARGET_DIR/src/main --include="*.java" --include="*.kt" 2>/dev/null | wc -l

# Batch 프로젝트 감지
grep -rl "spring-batch\|@EnableBatchProcessing\|JobBuilderFactory\|StepBuilderFactory" $TARGET_DIR/src/main --include="*.java" --include="*.kt" --include="*.xml" 2>/dev/null | wc -l
# XML 기반 배치 Job 감지 (레거시)
find $TARGET_DIR/src/main -path "*/batch/jobs/*.xml" -o -path "*/batch/*.xml" 2>/dev/null | wc -l
```

| Controller 존재 | Batch 설정 존재 | 유형 | 생성 파일 |
|----------------|----------------|------|-----------|
| O | X | **API** | api-spec.json |
| X | O | **Batch** | job-spec.json |
| O | O | **Hybrid** | api-spec.json + job-spec.json |
| X | X | **기타** | 둘 다 생성하지 않음 |

> 유형에 따라 api-spec.json과 job-spec.json 중 해당하는 것만 생성한다.

---

## 생성할 파일 목록

```
.claude/
└── ai-context/
    ├── domain-overview.md          # 서비스 역할, 도메인 개념, 패키지 구조
    ├── data-model.md               # 엔티티 관계 및 필드 정의
    ├── api-spec.json               # API 엔드포인트 명세 (API/Hybrid 유형)
    ├── job-spec.json               # Batch Job 명세 (Batch/Hybrid 유형)
    ├── kafka-spec.json             # 이벤트 발행/구독 명세 (해당 시)
    └── external-integration.md     # 외부 API 및 인프라 연동 정보
```

## CLAUDE.md 생성 또는 업데이트

1. 기존 CLAUDE.md가 있는지 확인한다
2. 있으면: 기존 내용을 유지하고, AI Context 섹션만 추가하거나 갱신한다
3. 없으면: 새로 생성한다

---

## 실행 절차

### Step 1: 프로젝트 규모 판단

가장 먼저 프로젝트의 규모를 측정한다.

```bash
# 소스 파일 수 측정 (테스트 제외)
find $TARGET_DIR/src/main -name "*.java" -o -name "*.kt" -o -name "*.ts" -o -name "*.py" | wc -l

# 패키지/디렉토리 수 측정
find $TARGET_DIR/src/main -type d | wc -l
```

**소규모** (소스 파일 50개 미만, 패키지 10개 미만): Step 2로 진행하여 한 번에 분석한다.
**대규모** (소스 파일 50개 이상 또는 패키지 10개 이상): Step 1-1의 분할 처리 절차를 따른다.

#### Step 1-1: 대규모 프로젝트 분할 처리

대규모 프로젝트는 파일 단위로 나눠서 분석한다. 각 파일을 독립적으로 생성하되, 사용자에게 진행 상황을 보고하면서 하나씩 처리한다.

**처리 순서:**

먼저 디렉토리 구조 전체를 탐색하여 패키지 목록과 역할을 파악한다. 그 다음 아래 순서로 파일을 하나씩 생성한다.

**① data-model.md 생성**
- entity, domain, model 패키지의 클래스만 읽는다
- `@Entity`, `@Document`, `@Table` 등 영속성 어노테이션이 있는 클래스를 찾는다
- 완료 후 사용자에게 결과를 보여주고, 누락 여부를 확인받는다

**② api-spec.json 또는 job-spec.json 생성** (프로젝트 유형에 따라)

**API 유형인 경우 — api-spec.json 생성:**
- controller, web, api, router 패키지의 클래스만 읽는다
- `@RestController`, `@Controller`, `@RequestMapping` 등이 있는 클래스를 찾는다
- 완료 후 사용자에게 결과를 보여주고, 누락 여부를 확인받는다

**Batch 유형인 경우 — job-spec.json 생성:**
- batch/jobs/ 하위 XML 파일 또는 `@Configuration` + `@Bean Job` 클래스를 읽는다
- Job ID, Step 구성, 스케줄(Quartz cron 또는 `@Scheduled`), 담당 BO/Service 클래스를 추출한다
- Quartz 설정 파일(quartz-manager.xml, quartz.properties 등)에서 스케줄 정보를 읽는다
- 완료 후 사용자에게 결과를 보여주고, 누락 여부를 확인받는다

**Hybrid 유형인 경우 — 둘 다 생성:**
- api-spec.json과 job-spec.json을 각각 위의 절차로 생성한다
  O
**③ kafka-spec.json 생성** (메시지 브로커 사용 시)
- consumer, producer, event, listener, messaging 패키지를 읽는다
- `@KafkaListener`, `KafkaTemplate`, `@RabbitListener` 등이 있는 클래스를 찾는다
- 완료 후 사용자에게 결과를 보여주고, 누락 여부를 확인받는다

**④ external-integration.md 생성**
- client, infra, adapter, gateway, external 패키지를 읽는다
- `RestTemplate`, `WebClient`, `FeignClient`, SDK 클래스 등을 사용하는 클래스를 찾는다
- 설정 파일(application.yml 등)에서 외부 URL, Redis, S3 등 인프라 설정을 읽는다
- 완료 후 사용자에게 결과를 보여주고, 누락 여부를 확인받는다

**FeignClient URL 형태 판별 — 반드시 수행**

`@FeignClient`가 발견되면 application.yml에서 해당 url 값을 읽어 아래 기준으로 분류한다.
분류 결과에 따라 external-integration.md 내 섹션을 분리해서 작성한다.

| url 값 형태 | 분류 | 예시 |
|------------|------|------|
| `http://` 또는 `https://` 로 시작 | 하드코딩 URL | `https://api.pgservice.com` |
| `lb://` 로 시작 | 서비스 디스커버리 (로드밸런서) | `lb://user-service` |
| `${...}` 환경변수 | 환경별 URL — 실제값은 application-{env}.yml 확인 | `${USER_SERVICE_URL}` |
| 프로토콜 없는 문자열 | **서비스 레지스트리 등록명** | `bill-repoA` |

> 프로토콜 없는 문자열(`bill-repoA` 등)은 실제 URL이 아니라 서비스 레지스트리에 등록된 이름이다.
> Spring Cloud가 런타임에 레지스트리를 조회하여 실제 주소로 resolve한다.
> 코드에서 이 값을 URL처럼 사용하려 하면 안 된다.

**⑤ domain-overview.md 생성**
- ①~④에서 수집한 정보를 종합한다
- 빌드 파일, 설정 파일, 패키지 구조를 기반으로 전체 그림을 작성한다
- 이 파일은 반드시 마지막에 생성한다 (앞선 분석 결과를 종합해야 하므로)

**⑥ CLAUDE.md 생성**
- 모든 ai-context 파일이 완성된 후 진입점 문서를 생성한다

각 단계를 완료할 때마다 사용자에게 보고한다:
```
✅ data-model.md 생성 완료 (엔티티 12개 추출)
➡️ 다음: api-spec.json 생성을 진행합니다
```

Step 1-1을 따른 경우 Step 2, 3은 건너뛰고 Step 4(검증)로 이동한다.

---

### Step 2: 프로젝트 탐색 (소규모)

아래 순서로 프로젝트 구조를 파악한다.

1. 프로젝트 루트의 빌드 파일을 읽는다 (`$TARGET_DIR/build.gradle`, `$TARGET_DIR/pom.xml`, `$TARGET_DIR/package.json` 등)
   - 프로젝트명, 기술 스택, 주요 의존성 파악
2. 소스 디렉토리 구조를 탐색한다
   - `$TARGET_DIR/src/main/java` (또는 해당 언어의 소스 루트) 아래 패키지 구조 파악
3. 설정 파일을 읽는다
   - `$TARGET_DIR/src/main/resources/application.yml`, `application.properties`, `.env` 등
   - DB, 메시지 큐, 외부 서비스 연결 정보 파악

### Step 3: 코드 분석 및 정보 추출 (소규모)

다음 항목들을 소스코드에서 추출한다.

**도메인 정보**
- 패키지 구조와 각 패키지의 역할
- 핵심 비즈니스 로직이 있는 클래스 식별
- 사용 중인 아키텍처 패턴 (Layered, Hexagonal, DDD 등)

**엔티티/모델**
- Entity, Document 등 영속성 객체 클래스
- 필드명, 타입, 제약조건, 관계(OneToMany, ManyToOne 등)
- Enum 정의

**API 엔드포인트** (API/Hybrid 유형)
- Controller/Router 클래스에서 모든 엔드포인트 추출
- HTTP Method, Path, Request/Response 타입
- 인증/인가 설정

**배치 Job 스펙** (Batch/Hybrid 유형)
- Job 정의 (XML 또는 Java Config)에서 모든 Job과 Step 추출
- Job ID, Step 구성, Reader/Writer/Tasklet 클래스
- Quartz/Scheduler 설정에서 cron 스케줄 추출
- 카테고리별 그룹핑 (Job XML 디렉토리 구조 기반)

**이벤트 스펙** (Kafka, RabbitMQ 등 사용 시)
- Producer: 발행하는 토픽, 메시지 구조
- Consumer: 구독하는 토픽, 처리 로직 요약

**외부 연동**
- 외부 API 호출 (RestTemplate, WebClient, Feign 등)
- 연동 대상 서비스명, 엔드포인트, 용도
- 사용 중인 인프라 (Redis, S3, ElasticSearch 등)

### Step 4: 파일 생성 (소규모)

아래 포맷에 맞춰 각 파일을 생성한다.

### Step 5: 검증

모든 파일 생성이 완료된 후 (소규모/대규모 모두) 아래를 자동으로 검증한다.

```bash
# 실제 엔티티 수와 data-model.md에 기재된 엔티티 수 비교
grep -rl "@Entity\|@Document\|@Table" $TARGET_DIR/src/main --include="*.java" --include="*.kt" | wc -l

# API 유형: 실제 컨트롤러 엔드포인트 수와 api-spec.json에 기재된 수 비교
grep -rl "@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping\|@PatchMapping\|@RequestMapping" $TARGET_DIR/src/main --include="*.java" --include="*.kt" | wc -l

# Batch 유형: 실제 배치 Job XML 수와 job-spec.json에 기재된 Job 수 비교
find $TARGET_DIR/src/main -path "*/batch/jobs/*.xml" -o -path "*/batch/jobs/**/*.xml" | wc -l
# 또는 Java Config 기반인 경우
grep -rl "JobBuilderFactory\|@Bean.*Job\b" $TARGET_DIR/src/main --include="*.java" --include="*.kt" | wc -l
```

불일치가 발견되면:
1. 누락된 항목을 구체적으로 식별한다
2. 해당 소스 파일을 읽어 정보를 추출한다
3. 해당 ai-context 파일에 보충한다
4. 사용자에게 보충 내역을 보고한다

```
⚠️ 검증 결과: @Entity 클래스 15개 중 data-model.md에 12개만 기재됨
   누락: PaymentHistory, RefundLog, AuditTrail
   → 보충 완료
```

---

## 파일별 포맷

### 1. domain-overview.md

```markdown
# [서비스명]

## 서비스 역할
이 서비스가 전체 시스템에서 담당하는 역할을 2~3문장으로 설명.

## 기술 스택
- Language: Java 17
- Framework: Spring Boot 3.x
- DB: PostgreSQL
- Message Broker: Kafka
- (기타)

## 아키텍처 패턴
사용 중인 아키텍처 패턴과 그 구조를 간략히 설명.

## 패키지 구조
각 패키지가 어떤 역할인지 설명.
```
com.example.service
├── controller/    # REST API 엔드포인트
├── service/       # 비즈니스 로직
├── domain/        # 엔티티 및 도메인 객체
├── repository/    # 데이터 접근 계층
├── config/        # 설정 클래스
├── dto/           # 요청/응답 DTO
└── infra/         # 외부 연동 어댑터
```

## 핵심 비즈니스 규칙
코드에서 파악 가능한 주요 비즈니스 규칙이나 검증 로직을 나열.
예: "주문 금액이 0 이하이면 예외 발생", "회원 등급에 따라 할인율 차등 적용"

## 주의사항
<!-- 이 섹션은 개발자가 직접 보강해야 합니다 -->
- 레거시 코드나 건드리면 안 되는 부분
- 알려진 기술 부채
- 특이한 설계 결정의 이유
```

### 2. data-model.md

```markdown
# 데이터 모델

## 엔티티 관계도 (텍스트)
```
User (1) ──── (N) Order
│
Order (1) ──── (N) OrderItem
│
OrderItem (N) ──── (1) Product
```

## 엔티티 상세

### User
| 필드 | 타입 | 제약조건 | 설명 |
|------|------|----------|------|
| id | Long | PK, Auto | 사용자 ID |
| email | String | Unique, NotNull | 이메일 |
| name | String | NotNull | 이름 |
| grade | UserGrade | NotNull | 회원 등급 |
| createdAt | LocalDateTime | NotNull | 생성일시 |

### (다음 엔티티...)

## Enum 정의

### UserGrade
| 값 | 설명 |
|----|------|
| BASIC | 일반 회원 |
| PREMIUM | 프리미엄 회원 |
| VIP | VIP 회원 |
```

### 3. api-spec.json

```json
{
   "service": "서비스명",
   "basePath": "/api/v1",
   "endpoints": [
      {
         "method": "GET",
         "path": "/users/{id}",
         "summary": "사용자 단건 조회",
         "auth": "ROLE_USER",
         "requestParams": [],
         "requestBody": null,
         "responseBody": "UserResponse",
         "notes": ""
      },
      {
         "method": "POST",
         "path": "/orders",
         "summary": "주문 생성",
         "auth": "ROLE_USER",
         "requestParams": [],
         "requestBody": "CreateOrderRequest",
         "responseBody": "OrderResponse",
         "notes": "재고 차감이 동시에 발생"
      }
   ]
}
```

### 4. job-spec.json

Batch 유형이 아닌 프로젝트라면 이 파일은 생성하지 않는다.

```json
{
   "service": "서비스명",
   "type": "batch",
   "scheduler": "Quartz 1.5 / Spring Scheduler / etc.",
   "batchJobs": [
      {
         "category": "카테고리명",
         "description": "이 카테고리의 Job들이 하는 일 요약",
         "jobs": [
            {
               "id": "sampleJob",
               "summary": "Job이 하는 일 한 줄 요약",
               "steps": ["step1 설명", "step2 설명"],
               "schedule": "매일 05:30 / 5분 간격 / 수동 실행 등",
               "boClass": "SampleBO",
               "notes": "특이사항이 있으면 기재"
            }
         ]
      }
   ]
}
```

**작성 규칙:**
- `category`는 Job XML 디렉토리 구조 또는 `@Configuration` 클래스 그룹 기준으로 분류
- `schedule`은 Quartz cron 표현식을 사람이 읽을 수 있는 형태로 변환하여 기재 (예: `0 30 5 * * ?` → `매일 05:30`)
- 스케줄이 없는 Job(수동 실행)은 `schedule` 필드 생략
- `steps`는 주요 Step만 기재. 단일 Step이면 생략 가능
- 비활성(disabled/주석처리) Job은 `"notes": "비활성"` 표시
- `boClass`는 해당 Job의 핵심 비즈니스 로직을 담당하는 BO/Service 클래스명

### 5. kafka-spec.json

Kafka, RabbitMQ 등 메시지 브로커를 사용하지 않는 프로젝트라면 이 파일은 생성하지 않는다.

```json
{
   "service": "서비스명",
   "produces": [
      {
         "topic": "order.completed",
         "trigger": "주문 완료 시",
         "messageFormat": {
            "orderId": "Long",
            "userId": "Long",
            "totalAmount": "BigDecimal",
            "completedAt": "LocalDateTime"
         },
         "producerClass": "OrderEventPublisher"
      }
   ],
   "consumes": [
      {
         "topic": "payment.confirmed",
         "action": "주문 상태를 PAID로 변경",
         "messageFormat": {
            "paymentId": "Long",
            "orderId": "Long",
            "paidAmount": "BigDecimal"
         },
         "consumerClass": "PaymentEventConsumer",
         "groupId": "order-service"
      }
   ]
}
```

### 5. external-integration.md

```markdown
# 외부 연동

## 내부 서비스 호출 (FeignClient + 서비스 레지스트리)

> url 값이 프로토콜 없는 문자열인 경우 서비스 레지스트리 등록명이다.
> 실제 URL이 아니므로 코드에서 직접 사용 불가. Spring Cloud가 런타임에 resolve한다.
> 등록명 변경 시 호출 서비스와 레지스트리 설정을 동시에 수정해야 한다.

### [서비스명]
- 호출 클래스: `XxxClient` (`@FeignClient(name = "xxx", url = "${feign.xxx.url}")`)
- 레지스트리 등록명: `org-xxx`  ← 실제 URL 아님
- YAML 키: `feign.xxx.url`
- 주요 엔드포인트:
  - `GET /api/v1/...` - 용도 설명
  - `POST /api/v1/...` - 용도 설명

## 외부 서비스 호출 (하드코딩 URL)

> 아래는 서비스 레지스트리를 거치지 않고 URL을 직접 지정하는 호출이다.

### [연동 대상 서비스명]
- 용도: 왜 호출하는지
- Base URL: `https://api.example.com` (고정 URL)
- 호출 클래스: 실제 호출하는 클래스명
- 주요 엔드포인트:
  - `GET /api/something` - 용도 설명
  - `POST /api/something` - 용도 설명
- 인증: API Key, OAuth 등
- 타임아웃/재시도: 설정이 있다면 명시

## 인프라 연동

### Redis
- 용도: 캐시 / 세션 / 분산락 등
- 주요 키 패턴: `user:{id}:profile`, `order:{id}:lock`
- TTL: 설정이 있다면 명시

### S3 (또는 다른 저장소)
- 용도: 파일 업로드 등
- 버킷/경로 패턴

### (기타 인프라...)
```

---

## CLAUDE.md 생성

`$TARGET_DIR/CLAUDE.md`를 생성한다. 이 파일은 Claude Code 실행 시 자동으로 로딩되는 진입점이다.

```markdown
# [서비스명]

## 서비스 위치
- 이 서비스의 역할 한 줄 요약
- 의존하는 서비스: [서비스A], [서비스B]
- 이 서비스에 의존하는 서비스: [서비스C], [서비스D]

## AI Context
이 프로젝트의 상세 컨텍스트는 아래 파일들을 참고:
- `.claude/ai-context/domain-overview.md` - 도메인 설명 및 정책 지식
- `.claude/ai-context/data-model.md` - 엔티티 정의
- `.claude/ai-context/api-spec.json` - API 스펙 (API/Hybrid 유형)
- `.claude/ai-context/job-spec.json` - 배치 Job 스펙 (Batch/Hybrid 유형)
- `.claude/ai-context/kafka-spec.json` - 이벤트 발행/수신 스펙
- `.claude/ai-context/external-integration.md` - 외부 API 호출 정보

## 코딩 컨벤션
(빌드 파일, 기존 코드 스타일에서 파악한 컨벤션 기재)

## 최근 주요 변경사항
<!-- 이 섹션은 개발자가 직접 관리합니다 -->
1. (변경사항 기재)
```

---

## 업데이트 모드

"ai-context 업데이트해줘"라는 요청을 받으면 전체를 새로 만들지 않는다.

1. 기존 ai-context 파일들을 읽는다
2. 현재 소스코드와 비교하여 변경된 부분만 식별한다
3. 변경된 부분만 업데이트한다
4. 개발자가 직접 작성한 섹션(주의사항, 비즈니스 맥락 등)은 보존한다

`<!-- 이 섹션은 개발자가 직접 관리합니다 -->` 주석이 있는 섹션은 자동 업데이트 대상에서 제외한다.

---

## 품질 기준

생성된 컨텍스트는 아래 기준을 만족해야 한다.

- 각 파일은 해당 영역의 정보만 담는다 (관심사 분리)
- 코드에서 직접 확인할 수 없는 내용은 추측하지 않고, 개발자가 보강할 수 있도록 placeholder를 남긴다
- domain-overview.md는 이 프로젝트에 처음 투입된 개발자가 5분 안에 전체 구조를 파악할 수 있는 수준으로 작성한다
- api-spec.json, job-spec.json, kafka-spec.json은 실제 코드와 1:1 대응해야 한다. 누락이 있으면 안 된다
- 파일 전체 분량은 CLAUDE.md 포함 500줄 이내를 목표로 한다

---

## ai-context 업데이트 규칙

### 코드 작업 완료 후 반드시 확인한다

코드 수정 작업이 완료되면 아래 표를 기준으로 ai-context 업데이트 필요 여부를 판단하고
사용자에게 알린다. 사용자가 원하면 즉시 업데이트를 수행한다.

| 변경 유형 | 업데이트 대상 파일 |
|----------|-----------------|
| Entity 필드 추가 / 삭제 / 타입 변경 | `data-model.md` |
| Controller 엔드포인트 추가 / 삭제 / 경로 변경 | `api-spec.json` |
| Batch Job 추가 / 삭제 / Step 변경 / 스케줄 변경 | `job-spec.json` |
| FeignClient 추가 / 삭제 / 메서드 변경 | `external-integration.md` |
| application.yml의 feign.xxx.url 값 변경 | `external-integration.md` |
| Kafka 토픽 추가 / 삭제 / 메시지 필드 변경 | `kafka-spec.json` |
| 패키지 구조 변경 / 신규 모듈 추가 | `domain-overview.md` |
| 새 레포 추가 후 ai-context 생성 완료 시 | 루트에서 `/generate-root-ai-context` 실행 |

### 업데이트하지 않아도 되는 변경

- 비즈니스 로직 내부 변경 (API 스펙 · 엔티티 변화 없는 경우)
- 테스트 코드 변경
- 리팩토링 (인터페이스 변화 없는 경우)
- application.yml에서 URL 값 외의 설정값만 바뀐 경우

### ai-context가 오래된 경우

작업 요청을 받았을 때 ai-context가 현재 코드와 불일치한다고 판단되면
작업을 시작하기 전에 사용자에게 먼저 알린다.

```
⚠️ ai-context가 현재 코드와 다를 수 있습니다.
   마지막 업데이트 이후 아래 파일이 변경된 것으로 보입니다:
   - OrderController.java (api-spec.json 업데이트 필요 가능성)

   먼저 업데이트할까요? (Y: 업데이트 후 작업 / N: 현재 상태로 작업)
```

---

## Step 6: 루트 ai-context 자동 갱신

서비스 ai-context 생성/업데이트가 완료되면 **자동으로** `/generate-root-ai-context`를 실행한다.
사용자에게 별도 확인을 받지 않고 바로 진행한다.

```
✅ [서비스명] ai-context 생성 완료
➡️ 루트 ai-context 자동 갱신을 시작합니다 (/generate-root-ai-context)
```

이 단계는 아래 조건을 모두 만족할 때 실행한다:
1. 현재 작업 디렉토리($TARGET_DIR)가 루트가 아닌 **개별 서비스 디렉토리**일 것
2. 루트 디렉토리에 `.claude/ai-context/`가 이미 존재할 것 (최초 생성이 아닌 갱신 대상이 있을 것)

조건 확인:
```bash
# 루트 ai-context 존재 여부 확인 (루트 = $TARGET_DIR의 상위 또는 프로젝트 루트)
ls -d .claude/ai-context 2>/dev/null && echo "ROOT_CONTEXT_EXISTS" || echo "NO_ROOT_CONTEXT"
```

- `ROOT_CONTEXT_EXISTS` → `/generate-root-ai-context`를 자동 실행한다
- `NO_ROOT_CONTEXT` → 루트 ai-context가 없으므로 사용자에게 안내만 한다:
  ```
  ℹ️ 루트 ai-context가 아직 없습니다.
     전체 시스템 컨텍스트를 생성하려면 루트 디렉토리에서 /generate-root-ai-context를 실행해주세요.
  ```