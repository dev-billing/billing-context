# 외부 연동

## 내부 서비스 호출 (FeignClient)

없음

## 외부 서비스 호출

없음

## 인프라 연동

### MySQL
- 용도: Todo 데이터 영구 저장
- JDBC URL: `jdbc:mysql://localhost:3306/review_db` (로컬 기본값)
- 환경 변수: `DB_USERNAME`, `DB_PASSWORD`
- DDL 전략: `ddl-auto: update` (스키마 자동 변경 — 운영 환경에서는 `validate` 또는 `none` 권장)

## CI/CD 연동

### API 문서 자동화 (GitHub Actions)
- 워크플로우: `.github/workflows/api-doc-pr.yml`, `.github/workflows/api-doc-trigger.yml`
- PR open/reopen → `dev-billing/shared-workflows`의 `reusable-generate-pr-drafts.yml` (mode: draft) 호출하여 변경 API 문서 초안 생성
- PR merge → `reusable-generate-pr-drafts.yml` (mode: all) 호출하여 최종 코드 기준 초안 최신화 + 발행 + deprecated 처리
- PR close (미merge) → `reusable-generate-pr-drafts.yml` (mode: delete_draft) 호출하여 초안 삭제
- 대상 브랜치: `master`, `develop`

### Dooray Wiki
- 용도: REST API 문서 자동 발행
- API 문서 레지스트리: `.shared-config/rest-api-docs/todo-service/api-docs-registry.json`
- 환경 변수: `DOORAY_API_KEY`, `DOORAY_WIKI_ID`, `DOORAY_PROJECT_ID`, `DOORAY_DRAFT_PARENT_PAGE_ID`, `DOORAY_EXTERNAL_PARENT_PAGE_ID`, `DOORAY_INTERNAL_PARENT_PAGE_ID`, `DOORAY_DEFAULT_PARENT_PAGE_ID`
- 발행 대상: external/internal/default URL 분류 기준으로 Wiki 페이지 구성
- 수동 발행: `api-doc-publish.yml` (workflow_dispatch) — `api_key`와 `branch` 지정
