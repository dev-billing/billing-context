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
