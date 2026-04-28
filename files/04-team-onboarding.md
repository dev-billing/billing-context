# 04. 팀원 온보딩 가이드

> **대상**: 팀원 (최초 1회 수행)
> **소요 시간**: 5분 이내

---

## 사전 확인

- [ ] GitHub 계정으로 PC에 git 로그인되어 있는가
- [ ] `bill/pay-api` (본인 담당 레포) 접근 권한이 있는가

---

## Step 1. 레포 clone

```bash
# 담당 레포를 원하는 위치에 clone — 어디에 clone해도 상관없다
git clone git@github.nhnent.com:bill/pay-api.git
cd pay-api
```

> **로컬 폴더 구조는 자유다.**
> ai-context는 GitHub(bill/bill-context)에서 직접 fetch하므로
> 어느 경로에 clone하든 동작에 영향이 없다.
>
> ```
> ~/projects/pay-api/        → ✅
> ~/work/backend/pay-api/    → ✅
> /workspace/pay-api/        → ✅
> ```

> **bill-context repo는 clone하지 않아도 된다.**
> claude 실행 시 자동으로 필요한 파일을 받아온다.

---

## Step 2. git 로그인 확인

claude가 ai-context를 받아올 때 git 로그인 정보를 사용한다.
이미 로그인되어 있으면 추가 작업이 없다.

```bash
git config --global user.name
git config --global user.email
```

값이 출력되면 완료다.

### git 로그인이 안 되어 있는 경우

**macOS (키체인 사용 중)**
```bash
# 별도 작업 불필요 — 키체인에서 자동으로 인증
```

**SSH 키 방식 사용 중**
```bash
ssh -T git@github.nhnent.com  # 연결 확인
```

**HTTPS 방식, 인증 저장이 안 된 경우**
```bash
git config --global credential.helper store
git pull  # 한 번 실행하면 인증 정보 저장됨
```

---

## Step 3. 완료

설정이 끝났다. 이제 `claude`를 실행하면 된다.

```bash
cd pay-api
claude
```

실행하면 자동으로:

```
✅ ai-context 최신화 완료 (루트 + pay-api)

이제 작업 요청을 말씀해주세요.
```

담당 레포에서 실행해도 전체 시스템 아키텍처를 알고 답해준다.

---

## 이후 매일 하는 것

```bash
cd pay-api
claude
```

**이게 전부다.**

- git pull 불필요
- ai-context 수동 업데이트 불필요
- bill-context repo clone 불필요

PR이 develop에 머지될 때마다 ai-context가 자동으로 최신화되고,
`claude`를 실행하는 순간 그 최신 내용을 받아서 작업을 시작한다.

---

## FAQ

**Q. ai-context가 잘 받아지는지 확인하고 싶어요.**

```bash
ls .claude/ai-context/
# root-service-map.md  root-dependency-graph.md
# root-interface-contracts.json  root-routing-guide.md
# domain-overview.md  api-spec.json  data-model.md  external-integration.md
```

claude 실행 후 위 파일들이 생성되면 정상이다.

**Q. .claude/ai-context/ 폴더가 git에 올라가나요?**

올라가지 않는다. `.gitignore`에 제외 처리되어 있다.
로컬에만 존재하는 캐시 파일이다.

**Q. 네트워크가 없는 환경에서도 claude를 쓸 수 있나요?**

fetch 실패 시 로컬에 캐시된 파일로 진행한다.
이전에 한 번이라도 claude를 실행했다면 `.claude/ai-context/` 파일이 남아있어 동작한다.

**Q. "이 기능 어느 서비스 건드려야 해?" 같은 질문도 할 수 있나요?**

된다. 루트 ai-context(전체 서비스 구조)를 함께 받아오기 때문에
담당 레포에서 실행해도 전체 아키텍처를 알고 답해준다.

**Q. 여러 레포의 코드를 동시에 수정해야 할 때는요?**

분석·계획 수립은 어느 레포에서든 가능하다.
실제로 여러 레포 코드를 동시에 수정하려면 여러 레포를 로컬에 clone한 후
관리자에게 Master Agent 셋업 방법을 문의한다.
