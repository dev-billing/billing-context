# 외부 연동

## 배포 환경

| 환경 | URL |
|------|-----|
| Alpha | https://alpha-todo-service.com |
| Real | https://todo-service.com |

- API Gateway 미사용

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

### AI Context 동기화 (GitHub Actions)
- 워크플로우: `.github/workflows/sync-ai-context.yml`
- 트리거: `develop` 브랜치 `push` 시 자동 실행
- 실행 위임: `dev-billing/shared-workflows/.github/workflows/sync-ai-context.yml@main` 호출
- `repo-name`: `${{ github.event.repository.name }}` (현재 레포명 자동 전달)
- `context-repo`: `billing-context`
- `secrets: inherit`로 시크릿 전달
