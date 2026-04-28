# 06. Claude Code 커맨드 레퍼런스

> **위치**: `.claude/commands/` (관리자 로컬 루트 기준)
> **사용**: 해당 레포에서 `claude` 실행 후 슬래시 커맨드로 호출

---

## 커맨드 목록

| 커맨드 | 용도 | 실행 빈도 |
|--------|------|----------|
| `/generate-ai-context` | 현재 레포 ai-context 전체 생성 | 최초 온보딩, 대규모 리팩토링 시 |
| `/generate-root-ai-context` | 루트 ai-context 생성 | 레포 추가/삭제 시 |
| `/update-ai-context` | git diff 기반 부분 업데이트 | 자동화 실패 시 수동 실행 |

---

## /generate-ai-context

### 설명
레포의 소스코드를 전체 분석하여 ai-context를 처음부터 생성한다.

### 사용법
```bash
# 해당 레포에서 claude 실행 후
/generate-ai-context
```

### 생성되는 파일
```
.claude/ai-context/
├── domain-overview.md      # 서비스 역할, 패키지 구조, 비즈니스 규칙
├── data-model.md           # 엔티티 관계 및 필드 정의
├── api-spec.json           # API 엔드포인트 명세 (API/Hybrid 유형)
├── job-spec.json           # Batch Job 명세 (Batch/Hybrid 유형)
├── kafka-spec.json         # 이벤트 발행/구독 명세 (해당 시)
└── external-integration.md # 외부 API 및 인프라 연동 정보
```

### 언제 사용하나
- 레포를 처음 온보딩할 때
- 패키지 구조가 전면 변경된 경우
- ai-context 파일이 손상되거나 삭제된 경우

---

## /generate-root-ai-context

### 설명
각 레포의 ai-context를 읽어 전체 시스템을 조망하는 루트 ai-context를 생성한다.
bill-context repo의 `.claude/ai-context/`가 있는 레포만 대상으로 한다.

### 사용법
```bash
# 임의의 레포에서 claude 실행 후
/generate-root-ai-context
```

### 생성되는 파일 (로컬에 root- 접두사로 저장)
```
.claude/ai-context/
├── root-service-map.md             # 전체 서비스 목록, 역할, ai-context 경로
├── root-dependency-graph.md        # 서비스 간 호출/이벤트 의존관계
├── root-interface-contracts.json   # 서비스 간 공유 API·이벤트 인터페이스
└── root-routing-guide.md           # 업무 유형별 서비스 매핑 가이드
```

> bill-context에 저장할 때는 `root-` 접두사를 제거한다.
> 로컬에서는 레포 전용 파일과 구분을 위해 접두사를 유지한다.

### 언제 사용하나
- 새 레포가 추가되고 해당 레포의 ai-context가 생성된 후
- 레포가 삭제되거나 이름이 바뀐 경우
- 서비스 간 의존관계가 크게 변경된 경우

---

## /update-ai-context

### 설명
git diff를 기반으로 변경된 파일만 감지하여 관련 ai-context만 부분 업데이트한다.
전체 재분석 없이 변경 최소 범위만 처리하므로 토큰 소비가 적다.

### 사용법
```bash
# 해당 레포에서 claude 실행 후
/update-ai-context
```

### 변경 유형별 업데이트 대상

| 변경된 파일 | 업데이트되는 ai-context |
|------------|----------------------|
| `**/controller/**` | `api-spec.json` |
| `**/entity/**`, `**/domain/**` | `data-model.md` |
| `**/consumer/**`, `**/producer/**` | `kafka-spec.json` |
| `**/client/**`, `**/infra/**` | `external-integration.md` |
| `application.yml` (feign 섹션) | `external-integration.md` |
| 패키지 구조 변경 또는 2개 이상 변경 | `domain-overview.md` 도 포함 |

### 언제 사용하나
- GitHub Actions 자동화가 실패한 경우 수동으로 실행

---

## FeignClient URL 판별 기준

`/generate-ai-context`, `/update-ai-context` 실행 시
`feign.xxx.url` 값을 아래 기준으로 자동 분류한다.

| url 값 형태 | 분류 | 예시 |
|------------|------|------|
| `http://` 또는 `https://` 시작 | 하드코딩 URL | `https://api.pgservice.com` |
| `lb://` 시작 | 서비스 디스커버리 (로드밸런서) | `lb://user-service` |
| `${...}` 환경변수 | 환경별 URL | `${USER_SERVICE_URL}` |
| 프로토콜 없는 문자열 | **서비스 레지스트리 등록명** | `bill-pay-api` |

> 프로토콜 없는 문자열은 실제 URL이 아니다.
> Spring Cloud가 런타임에 레지스트리에서 실제 주소로 resolve한다.

---

## ai-context 업데이트 필요 여부 판단

코드 수정 작업 완료 후 아래 변경이 있으면 ai-context 업데이트가 필요하다.
GitHub Actions가 자동으로 처리하지만, 수동 확인이 필요한 경우 참고한다.

**업데이트 필요:**
- Entity 필드 추가 / 삭제 / 타입 변경
- Controller 엔드포인트 추가 / 삭제 / 경로 변경
- FeignClient 추가 / 삭제 / 메서드 변경
- application.yml의 `feign.xxx.url` 값 변경
- Kafka 토픽 추가 / 삭제 / 메시지 필드 변경
- 패키지 구조 변경 / 신규 모듈 추가

**업데이트 불필요:**
- 비즈니스 로직 내부 변경 (API 스펙·엔티티 변화 없는 경우)
- 테스트 코드 변경
- 리팩토링 (인터페이스 변화 없는 경우)
- application.yml에서 URL 외의 설정값만 변경
