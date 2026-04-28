# Claude AI Context 자동화 시스템

## 개요

여러 레포를 가진 팀에서 Claude Code를 효과적으로 활용하기 위한 AI Context 자동화 시스템이다.
각 레포의 코드 변경이 PR merge될 때마다 AI Context가 자동으로 업데이트되고,
팀원은 `claude`를 실행하는 것만으로 항상 최신 컨텍스트를 기반으로 작업할 수 있다.

---

## 핵심 원칙

- **팀원은 아무것도 하지 않는다** — `claude` 실행 한 번이 전부
- **로컬 폴더 구조는 자유다** — 어디에 clone해도 동작한다
- **ai-context는 GitHub에서 직접 fetch한다** — 서버 불필요
- **담당 레포에서도 전체 아키텍처를 안다** — 루트 + 레포 전용 컨텍스트를 동시에 fetch
- **develop 브랜치 기준으로만 관리한다** — master/main은 관리 대상 아님
- **변경된 파일만 업데이트한다** — 전체 재분석 없이 토큰 절감

---

## GitHub 레포 구조

```
github.nhnent.com
├── bill/bill-context              ← AI Context 중앙 저장소 (신규 생성)
│   ├── .claude/
│   │   └── ai-context/           ← 루트 컨텍스트 (전체 시스템 조망)
│   │       ├── service-map.md
│   │       ├── dependency-graph.md
│   │       ├── interface-contracts.json
│   │       └── routing-guide.md
│   ├── pay-api/
│   │   └── ai-context/           ← pay-api 전용 컨텍스트
│   ├── pay-batch/
│   │   └── ai-context/
│   └── ...각 레포별 동일 (PR merge 시 자동 생성)
│
├── bill/bill-shared-workflows     ← 공통 GitHub Actions (기존)
│   └── .github/workflows/
│       └── update-ai-context.yml ← 신규 추가
│
├── bill/pay-api                   ← 기존 레포 (CLAUDE.md + workflow 추가)
├── bill/pay-batch
└── ...
```

---

## 동작 흐름

```
① 팀원이 develop 브랜치에 PR merge
        ↓
② GitHub Actions 트리거 (bill-shared-workflows 호출)
        ↓
③ bill-context에 해당 레포 디렉토리 없으면 자동 생성
        ↓
④ ai-context 존재 여부에 따라 분기
   ├── 없음(최초) → 소스 전체 분석 → ai-context 생성
   └── 있음(이후) → git diff → 변경분만 업데이트
        ↓
⑤ bill-context repo에 자동 커밋
        ↓
⑥ 루트 ai-context 자동 갱신
        ↓
⑦ 팀원이 claude 실행하면 자동으로 최신 ai-context fetch
        ↓
⑧ 항상 최신 컨텍스트로 작업 시작
```

---

## 로컬 구조 — 팀원마다 달라도 된다

ai-context는 GitHub에서 직접 fetch하므로 로컬 폴더 구조와 무관하게 동작한다.

```bash
~/projects/pay-api/        → claude 실행 → ✅
~/work/backend/pay-api/    → claude 실행 → ✅
/workspace/pay-api/        → claude 실행 → ✅
```

각 레포에서 claude를 실행하면 루트 ai-context(전체 시스템 구조)와
레포 전용 ai-context(도메인·API 상세)를 동시에 받아온다.
담당 레포에서도 전체 아키텍처 맥락을 알고 답해준다.

---

## 문서 목록

| 문서 | 대상 | 내용 |
|------|------|------|
| [TODO.md](./TODO.md) | 관리자 | 셋업 순서 체크리스트 |
| [01-ai-context-repo-setup.md](./01-ai-context-repo-setup.md) | 관리자 | bill-context repo 초기 구성 |
| [02-each-repo-setup.md](./02-each-repo-setup.md) | 관리자 | 각 레포 CLAUDE.md + workflow 추가 |
| [03-github-actions-automation.md](./03-github-actions-automation.md) | 관리자 | GitHub Actions 자동화 구성 |
| [04-team-onboarding.md](./04-team-onboarding.md) | 팀원 | 팀원 최초 1회 셋업 가이드 |
| [05-daily-usage.md](./05-daily-usage.md) | 팀 전체 | 일상적인 사용 방법 |
| [06-commands-reference.md](./06-commands-reference.md) | 전체 | Claude Code 커맨드 파일 레퍼런스 |

---

## 실제 파일 위치

```
{각 레포}/
├── CLAUDE.md                              ← claude 실행 시 자동 로딩
└── .github/
    └── workflows/
        └── ai-context.yml                 ← develop push 시 트리거

bill-shared-workflows/
└── .github/
    └── workflows/
        └── update-ai-context.yml          ← 핵심 자동화 워크플로우

{커맨드 파일 — 관리자 로컬 루트}/
└── .claude/
    └── commands/
        ├── generate-ai-context.md
        ├── generate-root-ai-context.md
        └── update-ai-context.md
```
