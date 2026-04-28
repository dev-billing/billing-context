---
name: update-ai-context
description: |
  git diff를 기반으로 실제 변경된 파일만 감지하여 관련 ai-context만 부분 업데이트하는 커맨드.
  전체 재분석 없이 변경 최소 범위만 처리하므로 토큰 소비가 적다.
  "ai-context 업데이트해줘", "컨텍스트 동기화해줘", "변경사항 반영해줘" 등의 요청 시 사용.
  generate-ai-context로 최초 생성 이후의 유지보수 상황에서 사용한다.
---

# Update AI Context (경량 업데이트)

git diff를 기준으로 변경된 파일을 감지하고,
영향받는 ai-context 파일만 선택적으로 업데이트한다.
변경이 없는 파일은 읽지도 않는다.

---

## 실행 위치 결정

`$ARGUMENTS` 값을 먼저 확인한다.

```bash
echo "$ARGUMENTS"
```

- 인자가 있는 경우 (`update-ai-context pay-api`) → `$TARGET_DIR = ./$ARGUMENTS`
- 인자가 없는 경우 (`update-ai-context`) → `$TARGET_DIR = ./`

디렉토리 존재 여부 확인:
```bash
ls -d "$TARGET_DIR" 2>/dev/null && echo "EXISTS" || echo "NOT_FOUND"
```

없으면 즉시 중단하고 사용자에게 알린다.

---

## Step 1: 변경 파일 감지

git을 이용해 마지막 ai-context 생성 이후 변경된 소스 파일만 추출한다.

```bash
# 방법 A — 마지막 커밋 기준 (기본)
git -C $TARGET_DIR diff HEAD~1 --name-only

# 방법 B — 스테이징 포함 uncommitted 변경까지
git -C $TARGET_DIR diff HEAD --name-only

# 방법 C — 특정 커밋 이후 전체 (ai-context 생성 시점 커밋 해시를 아는 경우)
# git -C $TARGET_DIR diff {커밋해시}..HEAD --name-only
```

> 기본은 방법 A를 사용한다.
> 사용자가 "마지막 배포 이후", "오늘 변경분" 등으로 범위를 지정하면 그에 맞게 조정한다.

변경 파일 목록을 사용자에게 보여준다:

```
📋 감지된 변경 파일 (HEAD~1 기준):
   src/main/java/.../OrderController.java
   src/main/java/.../OrderService.java
   src/main/java/.../Order.java
   src/main/resources/application.yml
```

변경 파일이 없으면 즉시 종료한다:
```
✅ 변경된 소스 파일이 없습니다. ai-context는 최신 상태입니다.
```

---

## Step 2: 영향 범위 매핑

변경된 파일 경로를 아래 규칙으로 분류하여 업데이트할 ai-context 파일을 결정한다.
**분류된 대상 외의 ai-context 파일은 읽지도, 수정하지도 않는다.**

| 변경된 파일 패턴 | 업데이트할 ai-context |
|----------------|----------------------|
| `**/controller/**`, `**/web/**`, `**/api/**` | `api-spec.json` |
| `**/entity/**`, `**/domain/**`, `**/model/**` + `@Entity` 포함 | `data-model.md` |
| `**/consumer/**`, `**/producer/**`, `**/listener/**`, `**/event/**` | `kafka-spec.json` |
| `**/client/**`, `**/infra/**`, `**/adapter/**`, `**/external/**` | `external-integration.md` |
| `application.yml`, `application-*.yml` 내 `feign.` 키 변경 | `external-integration.md` |
| 위 패턴 중 2개 이상 변경, 또는 패키지 구조 변경 | `domain-overview.md` 도 포함 |

매핑 결과를 사용자에게 보고한다:

```
🎯 업데이트 대상 ai-context:
   ✅ api-spec.json       ← OrderController.java 변경
   ✅ data-model.md       ← Order.java 변경
   ⏭️  kafka-spec.json    ← 변경 없음 (스킵)
   ⏭️  external-integration.md ← 변경 없음 (스킵)
   ⏭️  domain-overview.md ← 변경 없음 (스킵)
```

---

## Step 3: 부분 업데이트 실행

매핑된 ai-context 파일별로 아래를 수행한다.
**변경된 소스 파일만 읽는다. 변경 없는 파일은 읽지 않는다.**

### api-spec.json 업데이트
1. 변경된 Controller 파일만 읽는다
2. 기존 `api-spec.json`을 읽는다
3. 변경된 Controller의 엔드포인트만 추출하여 기존 내용과 병합한다
   - 경로가 같으면 덮어쓴다
   - 삭제된 메서드는 제거한다
   - 신규 메서드는 추가한다

### data-model.md 업데이트
1. 변경된 Entity 파일만 읽는다
2. 기존 `data-model.md`를 읽는다
3. 해당 엔티티 섹션만 교체한다. 다른 엔티티 섹션은 그대로 둔다

### kafka-spec.json 업데이트
1. 변경된 Consumer/Producer 파일만 읽는다
2. 기존 `kafka-spec.json`을 읽는다
3. 변경된 토픽 항목만 교체한다

### external-integration.md 업데이트
1. 변경된 Client 파일과 application.yml을 읽는다
2. application.yml의 `feign.` 섹션에서 변경된 항목을 확인한다
3. **FeignClient url 값 형태 판별:**
   - 프로토콜 없는 문자열 → 서비스 레지스트리 등록명 (실제 URL 아님)
   - `http(s)://` → 하드코딩 URL
   - `lb://` → 로드밸런서 서비스 디스커버리
   - `${...}` → 환경변수 (application-{env}.yml 확인 필요)
4. 해당 서비스 섹션만 교체한다

### domain-overview.md 업데이트
패키지 구조가 변경되었거나 2개 이상의 ai-context가 업데이트된 경우에만 실행한다.
1. 빌드 파일과 패키지 구조만 읽는다 (전체 소스 재분석 하지 않음)
2. 변경된 섹션(기술 스택, 패키지 구조)만 교체한다
3. `<!-- 이 섹션은 개발자가 직접 관리합니다 -->` 섹션은 보존한다

---

## Step 4: 완료 보고

```
✅ ai-context 업데이트 완료

업데이트된 파일:
  api-spec.json    — POST /orders 엔드포인트 수정, GET /orders/{id}/status 신규 추가
  data-model.md    — Order 엔티티 status 필드 타입 변경 (String → OrderStatus)

스킵된 파일 (변경 없음):
  kafka-spec.json, external-integration.md, domain-overview.md

읽은 소스 파일: 3개 (전체 분석 대비 약 90% 절감)
```

---

## 팀 환경에서의 권장 사용 패턴

여러 명이 동시에 개발하는 환경에서는 아래 패턴을 권장한다.

### PR 머지 후 1회 실행 (권장)
```
# PR 머지 후 변경된 레포에서 한 번만 실행
/update-ai-context pay-api
```
개인마다 실행하지 않고, **PR이 main에 머지된 후 한 번만** 실행한다.
이렇게 하면 팀 전체가 동일한 ai-context를 공유하게 된다.

### git hook으로 자동화 (선택)
`.git/hooks/post-merge`에 아래를 추가하면 머지할 때 자동으로 알려준다:

```bash
#!/bin/sh
echo ""
echo "💡 ai-context 업데이트가 필요할 수 있습니다."
echo "   변경된 레포에서 /update-ai-context {레포명} 을 실행해주세요."
```

### 전체 재생성이 필요한 경우
아래 상황에서는 경량 업데이트 대신 `/generate-ai-context {레포명}`을 실행한다:
- 레포를 처음 온보딩할 때
- 대규모 리팩토링으로 패키지 구조가 전면 변경된 경우
- ai-context 파일이 손상되거나 삭제된 경우

---

## 품질 기준

- 변경되지 않은 ai-context 파일은 절대 수정하지 않는다
- `<!-- 이 섹션은 개발자가 직접 관리합니다 -->` 섹션은 항상 보존한다
- 읽은 소스 파일 수를 완료 보고에 항상 표시한다 (토큰 절감 효과 가시화)
- 불확실한 변경(패턴 매핑 애매한 경우)은 추측하지 않고 사용자에게 확인을 요청한다
