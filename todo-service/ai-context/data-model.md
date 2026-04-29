# 데이터 모델

## 엔티티 관계도 (텍스트)

```
Todo (단일 엔티티, 연관 관계 없음)
```

## 엔티티 상세

### Todo
테이블명: `todos`

| 필드 | 타입 | 제약조건 | 설명 |
|------|------|----------|------|
| id | Long | PK, Auto Increment | Todo ID |
| title | String | NotNull | 제목 |
| content | String | TEXT, Nullable | 내용 |
| status | TodoStatus | NotNull, Default=TODO | 진행 상태 |
| createdAt | LocalDateTime | Auto(@CreationTimestamp) | 생성일시 |
| updatedAt | LocalDateTime | Auto(@UpdateTimestamp) | 수정일시 |

## Enum 정의

### TodoStatus
| 값 | 설명 |
|----|------|
| TODO | 할 일 (기본값) |
| IN_PROGRESS | 진행 중 |
| DONE | 완료 |

## 주의사항

- `TodoCreateRequest`에 `dueDate(LocalDate)` 필드가 있지만 `Todo` 엔티티에 해당 컬럼이 없어 실제로 저장되지 않음
- `ddl-auto: update` 설정이므로 스키마가 자동 변경될 수 있음 (운영 환경 주의)
