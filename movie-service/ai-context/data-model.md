# 데이터 모델

## 엔티티 관계도

```
Theater (1) ──── (N) TheaterSeat
Theater (1) ──── (N) Screen
TheaterSeat (N) ──── (1) TheaterSeatGrade
Movie (1) ──── (N) Screen

Screen (1) ──── (N) Reservation
Reservation (1) ──── (N) ReservationSeat
TheaterSeat (1) ──── (N) ReservationSeat
```

## 엔티티 상세

### MovieEntity (`movies`)
| 필드 | 타입 | 제약조건 | 설명 |
|------|------|----------|------|
| movieId | Long | PK, Auto | 영화 ID |
| title | String | NotNull, length=100 | 영화 제목 |
| releaseDate | LocalDate | NotNull | 개봉일 |
| ageRate | MovieAgeRateType | NotNull, Enum | 관람 연령 등급 |
| createDateTime | LocalDateTime | Auto (BaseEntity) | 생성일시 |
| modifyDateTime | LocalDateTime | Auto (BaseEntity) | 수정일시 |

### ReservationEntity (`reservations`)
| 필드 | 타입 | 제약조건 | 설명 |
|------|------|----------|------|
| reservationId | Long | PK, Auto | 예약 ID |
| userId | Long | NotNull | 사용자 ID (외부 참조) |
| screenId | Long | NotNull | 상영 ID |
| status | ReservationStatus | NotNull, Enum | 예약 상태 |
| amount | int | - | 총 결제 금액 |
| reservationSeats | List\<ReservationSeatEntity\> | OneToMany (LAZY) | 예약 좌석 목록 |
| createDateTime | LocalDateTime | Auto (BaseEntity) | 생성일시 |
| modifyDateTime | LocalDateTime | Auto (BaseEntity) | 수정일시 |

### ReservationSeatEntity (`reservation_seats`)
| 필드 | 타입 | 제약조건 | 설명 |
|------|------|----------|------|
| reservationSeatId | Long | PK, Auto | 예약 좌석 ID |
| reservationId | Long | - | 예약 ID |
| theaterSeatId | Long | NotNull | 상영관 좌석 ID |
| createDateTime | LocalDateTime | Auto (BaseEntity) | 생성일시 |
| modifyDateTime | LocalDateTime | Auto (BaseEntity) | 수정일시 |

### ScreenEntity (`screens`)
| 필드 | 타입 | 제약조건 | 설명 |
|------|------|----------|------|
| screenId | Long | PK, Auto | 상영 ID |
| theaterId | Long | NotNull | 상영관 ID |
| movieId | Long | NotNull | 영화 ID |
| screenDateTime | LocalDateTime | NotNull | 상영 일시 |

### TheaterEntity (`theaters`)
| 필드 | 타입 | 제약조건 | 설명 |
|------|------|----------|------|
| theaterId | Long | PK, Auto | 상영관 ID |
| theaterName | String | NotNull, length=100 | 상영관 이름 |

### TheaterSeatEntity (`theater_seats`)
| 필드 | 타입 | 제약조건 | 설명 |
|------|------|----------|------|
| theaterSeatId | Long | PK, Auto | 좌석 ID |
| theaterId | Long | NotNull | 상영관 ID |
| theaterSeatGrade | TheaterSeatGradeEntity | ManyToOne (LAZY) | 좌석 등급 |
| rowName | String | NotNull, length=10 | 열 이름 (예: A, B) |
| seatNum | Integer | NotNull | 좌석 번호 |
| createDateTime | LocalDateTime | Auto (BaseEntity) | 생성일시 |
| modifyDateTime | LocalDateTime | Auto (BaseEntity) | 수정일시 |

### TheaterSeatGradeEntity (`theater_seat_grades`)
| 필드 | 타입 | 제약조건 | 설명 |
|------|------|----------|------|
| theaterSeatGradeId | Long | PK, Auto | 좌석 등급 ID |
| gradeName | String | NotNull, length=50 | 등급 이름 |
| price | int | NotNull | 좌석 가격 (원) |
| createDateTime | LocalDateTime | Auto (BaseEntity) | 생성일시 |
| modifyDateTime | LocalDateTime | Auto (BaseEntity) | 수정일시 |

---

## Enum 정의

### MovieAgeRateType
| 값 | 설명 |
|----|------|
| ALL | 전체관람가 |
| TWELVE | 12세 이상 관람가 |
| FIFTEEN | 15세 이상 관람가 |
| NINETEEN | 19세 이상 관람가 |

### ReservationStatus
| 값 | 표시명 | 설명 |
|----|--------|------|
| PENDING | 대기중 | 결제 대기 중인 예약 |
| CREATED_FAIL | 생성 실패 | 예약 생성 실패 (결제 실패 시) |
| CONFIRMED | 확정 | 결제가 완료된 예약 |
| CANCELLING | 취소 처리중 | 환불 진행 중인 예약 |
| CANCELLED | 취소 완료 | 사용자가 취소한 예약 |
| COMPLETED | 완료 | 관람이 완료된 예약 |

> 활성 예약 상태: `PENDING`, `CONFIRMED` — 이 상태의 예약이 있는 좌석은 중복 예약 불가

---

## 도메인 객체 (JPA Entity 아님)

### Money (Value Object)
| 필드 | 타입 | 설명 |
|------|------|------|
| value | int | 금액 (0 이상, 0 미만이면 예외) |
