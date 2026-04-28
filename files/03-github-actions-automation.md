# 03. GitHub Actions 자동화 구성

> **대상**: 관리자 (1회 수행)
> **선행 조건**: [02-each-repo-setup.md](./02-each-repo-setup.md) 완료

---

## Step 1. GitHub Secrets 등록

bill org 레벨로 등록하면 모든 레포에서 자동으로 사용 가능하다.

```
github.nhnent.com → bill org Settings
→ Secrets and variables → Actions → New organization secret
```

| Secret 이름 | 값 | 용도 |
|------------|-----|------|
| `ANTHROPIC_API_KEY` | `sk-ant-...` | Claude Code 실행 |
| `AI_CONTEXT_TOKEN` | GitHub PAT | bill-context repo에 push 권한 |

### AI_CONTEXT_TOKEN 발급 방법

```
github.nhnent.com → 본인 Settings → Developer settings
→ Personal access tokens → Fine-grained tokens → Generate new token

권한 설정:
  Repository access: bill/bill-context (Only this repo)
  Permissions:
    Contents: Read and Write
    Metadata: Read
```

---

## Step 2. bill-shared-workflows에 공통 workflow 추가

아래 파일을 `bill-shared-workflows/.github/workflows/update-ai-context.yml`로 추가한다.

```yaml
name: Update AI Context (Shared)

on:
  workflow_call:
    inputs:
      repo-name:
        required: true
        type: string
    secrets:
      ANTHROPIC_API_KEY:
        required: true
      AI_CONTEXT_TOKEN:
        required: true

jobs:
  update-context:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout 소스 레포 (develop)
        uses: actions/checkout@v4
        with:
          ref: develop
          fetch-depth: 0

      - name: Checkout bill-context repo
        uses: actions/checkout@v4
        with:
          repository: bill/bill-context
          token: ${{ secrets.AI_CONTEXT_TOKEN }}
          path: _ai_context_repo

      - name: bill-context에 레포 디렉토리 없으면 자동 생성
        working-directory: _ai_context_repo
        run: |
          if [ ! -d "${{ inputs.repo-name }}/ai-context" ]; then
            echo "📁 ${{ inputs.repo-name }}/ai-context 디렉토리가 없습니다. 자동 생성합니다."
            mkdir -p "${{ inputs.repo-name }}/ai-context"
            git config user.name "github-actions[bot]"
            git config user.email "github-actions[bot]@users.noreply.github.com"
            touch "${{ inputs.repo-name }}/ai-context/.gitkeep"
            git add "${{ inputs.repo-name }}/"
            git commit -m "feat(${{ inputs.repo-name }}): ai-context 디렉토리 자동 생성 [skip ci]

            최초 PR merge 감지: ${{ github.repository }}@${{ github.sha }}"
            git push origin main
          else
            echo "✅ ${{ inputs.repo-name }}/ai-context 디렉토리 확인됨"
          fi

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: ai-context 존재 여부 확인
        id: check_context
        run: |
          if [ -f "_ai_context_repo/${{ inputs.repo-name }}/ai-context/domain-overview.md" ]; then
            echo "exists=true" >> $GITHUB_OUTPUT
          else
            echo "exists=false" >> $GITHUB_OUTPUT
          fi

      - name: ai-context 최초 생성 (없는 경우)
        if: steps.check_context.outputs.exists == 'false'
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude --non-interactive "
            현재 디렉토리는 ${{ inputs.repo-name }} 레포의 develop 브랜치다.
            .claude/commands/generate-ai-context.md 절차에 따라
            이 레포의 ai-context를 처음부터 생성해줘.
            생성된 파일은 ./_ai_context_repo/${{ inputs.repo-name }}/ai-context/ 에 저장해줘.
            완료 후 생성된 파일 목록을 출력해줘.
          "

      - name: ai-context 부분 업데이트 (이미 있는 경우)
        if: steps.check_context.outputs.exists == 'true'
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude --non-interactive "
            현재 디렉토리는 ${{ inputs.repo-name }} 레포의 develop 브랜치다.
            git diff origin/develop~1 --name-only 결과를 분석해서
            변경된 파일에 해당하는 ai-context만 부분 업데이트해줘.
            기존 ai-context는 ./_ai_context_repo/${{ inputs.repo-name }}/ai-context/ 에 있다.
            업데이트된 파일은 동일 경로에 저장해줘.
            .claude/commands/update-ai-context.md 의 절차를 따를 것.
            완료 후 업데이트된 파일 목록을 출력해줘.
          "

      - name: 레포 ai-context 커밋
        working-directory: _ai_context_repo
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add ${{ inputs.repo-name }}/ai-context/
          git diff --staged --quiet && echo "변경 없음 — 스킵" && exit 0
          git commit -m "feat(${{ inputs.repo-name }}): ai-context 자동 업데이트 [skip ci]

          트리거: ${{ github.repository }}@${{ github.sha }}"
          git push origin main

      - name: 루트 ai-context 업데이트
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude --non-interactive "
            _ai_context_repo/${{ inputs.repo-name }}/ai-context/ 내용이 업데이트됐어.
            _ai_context_repo/.claude/ai-context/ 의 루트 컨텍스트에서
            ${{ inputs.repo-name }} 관련 항목만 갱신해줘.
            대상 파일: service-map.md, dependency-graph.md, interface-contracts.json, routing-guide.md
            변경이 없으면 파일을 수정하지 말 것.
          "

      - name: 루트 ai-context 커밋
        working-directory: _ai_context_repo
        run: |
          git add .claude/ai-context/
          git diff --staged --quiet && echo "루트 변경 없음 — 스킵" && exit 0
          git commit -m "feat(root): 루트 ai-context 업데이트 (${{ inputs.repo-name }}) [skip ci]"
          git push origin main

      - name: 결과 요약
        run: |
          echo "================================"
          echo "✅ AI Context 업데이트 완료"
          echo "   레포     : ${{ inputs.repo-name }}"
          echo "   최초생성 : ${{ steps.check_context.outputs.exists == 'false' && 'YES' || 'NO' }}"
          echo "   커밋     : ${{ github.sha }}"
          echo "================================"
```

---

## Step 3. 동작 확인

```bash
# 테스트용 커밋을 develop에 push
cd pay-api
git checkout develop
echo "# test" >> README.md
git add . && git commit -m "test: ai-context 자동화 동작 확인"
git push origin develop
```

### 예상 동작 순서

```
1. pay-api develop push 감지
2. ai-context.yml 트리거
3. bill-shared-workflows/update-ai-context.yml 호출
4. bill-context에 pay-api/ 디렉토리 없으면 자동 생성
5. Claude Code 실행 → ai-context 생성 또는 업데이트
6. bill-context repo에 자동 커밋
7. 루트 ai-context 갱신 후 bill-context repo에 커밋
```

### 확인 포인트

```
✅ pay-api Actions 탭 → workflow 성공
✅ bill-context repo → pay-api/ai-context/ 파일 생성 확인
✅ bill-context repo → .claude/ai-context/ 루트 업데이트 확인
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| workflow가 트리거 안 됨 | ai-context.yml이 develop 브랜치에 없음 | develop에 파일 존재 확인 |
| Claude Code 실행 실패 | ANTHROPIC_API_KEY 누락/만료 | Secrets 값 확인 |
| bill-context repo push 실패 | AI_CONTEXT_TOKEN 만료 또는 권한 부족 | PAT 재발급, bill/bill-context write 권한 확인 |

다음 단계: [04-team-onboarding.md](./04-team-onboarding.md)
