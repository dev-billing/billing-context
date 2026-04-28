# AI Context 자동화 셋업 — 해야 할 일

> 순서대로 진행한다. 각 단계 완료 후 체크.

---

## Phase 1. GitHub 준비 (관리자, 1회)

### 1-1. bill/bill-context repo 생성

- [ ] `github.nhnent.com`에서 `bill/bill-context` private repo 생성
- [ ] 로컬에 clone

```bash
git clone git@github.nhnent.com:bill/bill-context.git
cd bill-context
```

- [ ] 루트 ai-context 디렉토리만 생성 후 push
  > 각 레포별 디렉토리는 첫 번째 PR merge 시 workflow가 자동으로 생성한다.

```bash
mkdir -p .claude/ai-context
touch .claude/ai-context/.gitkeep

git add .
git commit -m "feat: bill-context repo 초기 구성"
git push origin main
```

---

### 1-2. GitHub Secrets 등록

- [ ] Anthropic API 키 발급 확인 (https://console.anthropic.com)
- [ ] GitHub PAT 발급

```
github.nhnent.com → 본인 Settings → Developer settings
→ Personal access tokens → Fine-grained tokens → Generate new token

설정:
  Repository access: bill/bill-context (Only this repo)
  Permissions:
    Contents: Read and Write
    Metadata: Read
```

- [ ] bill org 레벨 Secrets 등록

```
github.nhnent.com → bill org Settings
→ Secrets and variables → Actions → New organization secret

등록 항목:
  이름: ANTHROPIC_API_KEY    값: sk-ant-...
  이름: AI_CONTEXT_TOKEN     값: github_pat_...
```

---

### 1-3. bill-shared-workflows에 공통 workflow 추가

- [ ] 아래 경로에 `update-ai-context.yml` 파일 추가
  → 첨부된 `update-ai-context.yml` 파일을 그대로 사용

```
bill-shared-workflows/.github/workflows/update-ai-context.yml
```

- [ ] develop 브랜치에 PR 올려서 merge

---

### 1-4. 각 레포에 CLAUDE.md + workflow 추가

아래 레포 각각에 대해 반복:

```
pay-api / pay-batch / pay-admin-api / pay-admin-web /
nebill / ne-settle / settle-admin / settle-batch / taurus-pay-admin
```

각 레포마다:

- [ ] `CLAUDE.md` 추가 → 첨부 파일에서 `{레포명}` 두 곳을 실제 레포명으로 변경
- [ ] `.gitignore`에 아래 추가

```gitignore
# AI Context — bill-context repo에서 중앙 관리됨 (로컬 캐시)
.claude/ai-context/
```

- [ ] `.github/workflows/ai-context.yml` 추가 → 첨부 파일에서 `{레포명}` 값만 변경
- [ ] develop 브랜치에 PR 올려서 merge

> **PR merge 시 자동으로 처리되는 것:**
> - bill-context에 `{레포명}/ai-context/` 디렉토리가 없으면 자동 생성
> - ai-context 최초 생성 (domain-overview, api-spec, data-model 등)
> - 루트 ai-context 자동 갱신

---

### 1-5. 루트 ai-context 초기 생성

모든 레포의 PR merge 후 workflow가 각 레포 ai-context를 자동 생성한다.
완료 확인 후 루트 ai-context를 한 번 수동으로 생성한다.

- [ ] bill-context repo에 각 레포 디렉토리가 생겼는지 확인

```bash
cd bill-context && git pull
ls
# pay-api/  pay-batch/  pay-admin-api/ ...
```

- [ ] 임의의 레포에서 claude 실행 후 입력

```
/generate-root-ai-context
```

- [ ] 생성된 루트 파일을 bill-context에 저장

```bash
# root- 접두사 파일들을 bill-context로 이동
cp .claude/ai-context/root-*.md   /path/to/bill-context/.claude/ai-context/
cp .claude/ai-context/root-*.json /path/to/bill-context/.claude/ai-context/

# root- 접두사 제거
cd /path/to/bill-context/.claude/ai-context/
for f in root-*; do mv "$f" "${f#root-}"; done

cd /path/to/bill-context
git add .
git commit -m "feat(root): 루트 ai-context 최초 생성"
git push origin main
```

---

## Phase 2. 동작 확인

- [ ] 아무 레포에서 테스트 커밋을 develop에 push

```bash
cd pay-api
git checkout develop
echo "# test" >> README.md
git add . && git commit -m "test: ai-context 자동화 동작 확인"
git push origin develop
```

- [ ] `github.nhnent.com → bill/pay-api → Actions` 탭에서 workflow 실행 확인
- [ ] `bill/bill-context` repo에 자동 커밋 확인
- [ ] 로컬에서 claude 실행 후 fetch 정상 확인

```bash
cd pay-api && claude
# ✅ ai-context 최신화 완료 (루트 + pay-api) 메시지 확인

ls .claude/ai-context/
# root-service-map.md  root-dependency-graph.md
# root-interface-contracts.json  root-routing-guide.md
# domain-overview.md  api-spec.json  data-model.md  external-integration.md
```

---

## Phase 3. 팀원 공유

- [ ] 팀원에게 `04-team-onboarding.md` 공유
- [ ] 팀원 각자 git 로그인 확인 후 claude 실행 테스트

---

## 참고 파일 목록

| 파일 | 용도 | 넣을 위치 |
|------|------|-----------|
| `CLAUDE.md` | 각 레포 진입점 | `{레포}/CLAUDE.md` |
| `ai-context.yml` | 각 레포 workflow | `{레포}/.github/workflows/ai-context.yml` |
| `update-ai-context.yml` | 공통 workflow | `bill-shared-workflows/.github/workflows/update-ai-context.yml` |
