# 02. 각 레포 설정

> **대상**: 관리자 (1회 수행)
> **선행 조건**: [01-ai-context-repo-setup.md](./01-ai-context-repo-setup.md) 완료

각 레포에 `CLAUDE.md`와 GitHub Actions workflow 파일을 추가한다.
팀원이 직접 할 필요 없이 관리자가 PR로 추가하면 된다.

---

## Step 1. 각 레포에 CLAUDE.md 추가

아래 템플릿을 각 레포에 맞게 `{레포명}` 두 곳만 바꿔서 추가한다.

CLAUDE.md는 두 가지를 fetch한다:
- **루트 ai-context** — 전체 시스템 구조 (서비스 간 의존관계, 인터페이스 등)
- **레포 전용 ai-context** — 이 레포의 도메인, API, 엔티티 상세

덕분에 팀원이 담당 레포에서 claude를 실행해도 전체 아키텍처 맥락을 알고 답해준다.

```markdown
# {레포명}

## 세션 시작 시 필수 작업

claude 시작 즉시 아래를 수행한다. 사용자 입력 전에 반드시 먼저 실행한다.

```bash
mkdir -p .claude/ai-context

# git 로그인 정보로 인증 (별도 토큰 불필요)
GH_TOKEN=$(git credential fill <<< "protocol=https
host=github.nhnent.com" | grep password | cut -d= -f2)

BASE_ROOT="https://github.nhnent.com/bill/bill-context/raw/main/.claude/ai-context"
BASE_REPO="https://github.nhnent.com/bill/bill-context/raw/main/{레포명}/ai-context"

# 루트 ai-context (전체 시스템 구조)
for file in service-map.md dependency-graph.md interface-contracts.json routing-guide.md; do
  curl -fsSL \
    -H "Authorization: token ${GH_TOKEN}" \
    "${BASE_ROOT}/${file}" \
    -o ".claude/ai-context/root-${file}" 2>/dev/null
done

# 레포 전용 ai-context
for file in domain-overview.md api-spec.json data-model.md external-integration.md; do
  curl -fsSL \
    -H "Authorization: token ${GH_TOKEN}" \
    "${BASE_REPO}/${file}" \
    -o ".claude/ai-context/${file}" 2>/dev/null
done
```

성공 시: ✅ ai-context 최신화 완료 (루트 + {레포명})
실패 시: ⚠️ fetch 실패 — 로컬 캐시로 진행합니다.

## AI Context 읽는 순서

업무 요청을 받으면 반드시 아래 순서로 읽는다.

1. `.claude/ai-context/root-routing-guide.md`         → 관련 서비스 판단
2. `.claude/ai-context/root-service-map.md`           → 전체 서비스 목록 및 역할
3. `.claude/ai-context/root-dependency-graph.md`      → 서비스 간 의존관계 및 영향 범위
4. `.claude/ai-context/root-interface-contracts.json` → 서비스 간 API·이벤트 인터페이스
5. `.claude/ai-context/domain-overview.md`            → 이 레포의 도메인 및 패키지 구조
6. `.claude/ai-context/api-spec.json`                 → 이 레포의 API 스펙
7. `.claude/ai-context/data-model.md`                 → 이 레포의 엔티티 정의
8. `.claude/ai-context/external-integration.md`       → 외부 연동 정보

> 이 레포만 관련된 단순 작업이라면 5번부터 읽어도 된다.
> 다른 서비스와 연관된 작업이라면 반드시 1번부터 읽는다.

## 코딩 컨벤션

<!-- 빌드 파일, 기존 코드 스타일에서 파악한 컨벤션 기재 -->

## 주의사항

<!-- 이 섹션은 개발자가 직접 관리합니다 -->
```

### 레포별 적용 목록

| 레포 | CLAUDE.md 추가 | {레포명} 값 |
|------|--------------|------------|
| pay-api | ☐ | `pay-api` |
| pay-batch | ☐ | `pay-batch` |
| pay-admin-api | ☐ | `pay-admin-api` |
| pay-admin-web | ☐ | `pay-admin-web` |
| nebill | ☐ | `nebill` |
| ne-settle | ☐ | `ne-settle` |
| settle-admin | ☐ | `settle-admin` |
| settle-batch | ☐ | `settle-batch` |
| taurus-pay-admin | ☐ | `taurus-pay-admin` |

---

## Step 2. 각 레포 .gitignore에 추가

각 레포의 `.gitignore`에 아래를 추가한다.

```gitignore
# AI Context — bill-context repo에서 중앙 관리됨 (로컬 캐시)
.claude/ai-context/
```

---

## Step 3. 각 레포에 GitHub Actions workflow 추가

각 레포의 `.github/workflows/ai-context.yml`을 생성한다.
`{레포명}` 값만 레포마다 다르고 나머지는 동일하다.

```yaml
name: Update AI Context

on:
  push:
    branches: [develop]

jobs:
  update:
    uses: bill/bill-shared-workflows/.github/workflows/update-ai-context.yml@main
    with:
      repo-name: {레포명}   # ← 레포마다 이 값만 변경
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      AI_CONTEXT_TOKEN: ${{ secrets.AI_CONTEXT_TOKEN }}
```

### 레포별 `repo-name` 값

| 레포 | repo-name |
|------|-----------|
| pay-api | `pay-api` |
| pay-batch | `pay-batch` |
| pay-admin-api | `pay-admin-api` |
| pay-admin-web | `pay-admin-web` |
| nebill | `nebill` |
| ne-settle | `ne-settle` |
| settle-admin | `settle-admin` |
| settle-batch | `settle-batch` |
| taurus-pay-admin | `taurus-pay-admin` |

---

## Step 4. PR로 각 레포에 반영

```bash
# 예시: pay-api 레포
cd pay-api
git checkout -b chore/add-claude-context
# CLAUDE.md, .gitignore, .github/workflows/ai-context.yml 추가
git add .
git commit -m "chore: Claude AI Context 자동화 설정 추가"
git push origin chore/add-claude-context
# PR 생성 → develop에 머지
```

---

## 완료 확인

각 레포에 아래 파일이 있으면 설정 완료다.

```
{레포}/
├── CLAUDE.md                              ✅
├── .gitignore (.claude/ai-context/ 제외)  ✅
└── .github/
    └── workflows/
        └── ai-context.yml                 ✅
```

다음 단계: [03-github-actions-automation.md](./03-github-actions-automation.md)
