# 05. 일상적인 사용 방법

> **대상**: 팀 전체

---

## 팀원 — 일반 개발 작업

### 기본 패턴

```bash
cd pay-api
claude
# → 자동으로 최신 ai-context fetch (루트 + pay-api 전용)
# → 작업 요청 입력
```

### 작업 요청 예시

```
# 영향 범위 분석
"이 기획서를 보고 어떤 서비스와 파일을 수정해야 하는지 분석해줘"

# 아키텍처 질문 (담당 레포에서도 가능)
"pay-api가 어떤 서비스를 호출하고 어디에서 호출받아?"
"이 API 변경하면 다른 서비스에 영향이 가?"

# 코드 수정 요청
"OrderController에 주문 상태 변경 API 추가해줘"

# 코드 리뷰
"방금 수정한 OrderService의 문제점을 찾아줘"
```

### PR merge 후 ai-context 업데이트 확인

develop에 PR merge 후 약 3~5분 뒤 GitHub Actions가 완료된다.
다음 `claude` 실행 시 자동으로 최신 내용이 반영된다.

---

## 관리자 — ai-context 수동 업데이트

자동화 외에 수동으로 업데이트가 필요한 경우에만 사용한다.

### 특정 레포 ai-context 부분 업데이트

GitHub Actions 자동화가 실패한 경우:

```bash
cd pay-api
claude
# Claude에 입력:
/update-ai-context
```

### 특정 레포 ai-context 전체 재생성

대규모 리팩토링 등으로 전체 재분석이 필요한 경우:

```bash
cd pay-api
claude
# Claude에 입력:
/generate-ai-context
```

### 루트 ai-context 업데이트

새 레포가 추가되거나 레포 간 의존관계가 크게 바뀐 경우:

```bash
cd {임의의 레포}
claude
# Claude에 입력:
/generate-root-ai-context
```

---

## bill-context repo 직접 관리

### 로컬에서 bill-context 확인

```bash
git clone git@github.nhnent.com:bill/bill-context.git
cd bill-context
git pull origin main
ls pay-api/ai-context/
```

### ai-context 내용 직접 수정

자동 생성이 잘못되었거나 개발자 주석을 추가할 때:

```bash
cd bill-context
vim pay-api/ai-context/domain-overview.md

git add .
git commit -m "docs(pay-api): 주의사항 섹션 보강"
git push origin main
```

> `<!-- 이 섹션은 개발자가 직접 관리합니다 -->` 주석이 있는 섹션은
> 자동 업데이트 시 덮어쓰지 않는다.

---

## ai-context 동기화 상태 확인

```bash
# 로컬 ai-context 최종 수정 시각
ls -la .claude/ai-context/

# bill-context 최신 커밋
cd bill-context && git log --oneline -5
```
