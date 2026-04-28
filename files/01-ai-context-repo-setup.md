# 01. bill-context 중앙 저장소 초기 구성

> **대상**: 관리자 (1회 수행)
> **선행 조건**: 없음

---

## Step 1. bill-context repo 생성

`github.nhnent.com`에서 새 repo를 생성한다.

```
repo명: bill-context
org: bill
visibility: private
```

로컬에 clone:

```bash
git clone git@github.nhnent.com:bill/bill-context.git
cd bill-context
```

---

## Step 2. 루트 디렉토리 구조 생성

루트 ai-context 디렉토리만 생성한다.
각 레포별 디렉토리는 해당 레포에 첫 PR이 merge될 때 workflow가 자동으로 생성한다.

```bash
mkdir -p .claude/ai-context
touch .claude/ai-context/.gitkeep
```

---

## Step 3. .gitignore 생성

```bash
cat > .gitignore << 'EOF'
.DS_Store
EOF
```

---

## Step 4. push

```bash
git add .
git commit -m "feat: bill-context repo 초기 구성"
git push origin main
```

---

## Step 5. 각 레포 ai-context 최초 생성

각 레포에 CLAUDE.md와 workflow가 추가된 후(02-each-repo-setup.md 완료 후)
develop에 PR이 merge되면 workflow가 자동으로 ai-context를 생성한다.

수동으로 생성하고 싶은 경우:

```bash
cd pay-api
claude
# Claude에 입력:
/generate-ai-context
```

생성된 파일을 bill-context에 저장:

```bash
git clone git@github.nhnent.com:bill/bill-context.git /tmp/bill-context

mkdir -p /tmp/bill-context/pay-api/ai-context
cp .claude/ai-context/*.md .claude/ai-context/*.json \
   /tmp/bill-context/pay-api/ai-context/ 2>/dev/null

cd /tmp/bill-context
git add .
git commit -m "feat(pay-api): ai-context 최초 생성"
git push origin main
```

---

## Step 6. 루트 ai-context 생성

각 레포 ai-context가 모두 생성된 후 루트 컨텍스트를 생성한다.

임의의 레포에서 claude 실행 후 입력:

```
/generate-root-ai-context
```

생성되는 파일:

```
.claude/ai-context/
├── root-service-map.md           # 전체 서비스 목록 및 경로
├── root-dependency-graph.md      # 서비스 간 의존관계
├── root-interface-contracts.json # 서비스 간 API·이벤트 인터페이스
└── root-routing-guide.md         # 업무 유형별 서비스 매핑 가이드
```

bill-context에 저장 (`root-` 접두사 제거):

```bash
cp .claude/ai-context/root-*.md   /tmp/bill-context/.claude/ai-context/
cp .claude/ai-context/root-*.json /tmp/bill-context/.claude/ai-context/

cd /tmp/bill-context/.claude/ai-context/
for f in root-*; do mv "$f" "${f#root-}"; done

cd /tmp/bill-context
git add .
git commit -m "feat(root): 루트 ai-context 최초 생성"
git push origin main
```

---

## 완료 확인

```bash
cd /tmp/bill-context && git pull
find . -type f | grep -v ".git" | sort
```

예상 출력:

```
./.claude/ai-context/dependency-graph.md
./.claude/ai-context/interface-contracts.json
./.claude/ai-context/routing-guide.md
./.claude/ai-context/service-map.md
./pay-api/ai-context/api-spec.json
./pay-api/ai-context/data-model.md
./pay-api/ai-context/domain-overview.md
./pay-api/ai-context/external-integration.md
...
```

다음 단계: [02-each-repo-setup.md](./02-each-repo-setup.md)
