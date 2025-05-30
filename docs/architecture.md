# 콘서트 예약 서비스 아키텍쳐

## ERD
![erd.png](erd.png)
```sql
-- 유저 계정 정보
CREATE TABLE ACCOUNT
(
   id         VARCHAR(64) PRIMARY KEY COMMENT '유저 고유 ID',
   email      VARCHAR(128) NOT NULL UNIQUE COMMENT '유저 이메일 (로그인용)',
   name       VARCHAR(64)  NOT NULL COMMENT '유저 이름',
   amount     INT          NOT NULL COMMENT '현재 잔액',
   created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '계정 생성 시각',
   updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '계정 수정 시각',
   unique (email)
) COMMENT='유저 계정';

-- 잔액 변동 내역
CREATE TABLE BALANCE_HISTORY
(
   id         BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '기록 ID',
   account_id VARCHAR(64) NOT NULL COMMENT '해당 유저 ID',
   amount     INT         NOT NULL COMMENT '변동 금액 (양수: 충전, 음수: 사용)',
   created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '변동 시각'
) COMMENT='잔액 변경 이력';

-- 대기열 토큰 관리 테이블
CREATE TABLE QUEUE_TOKEN
(
   token      VARCHAR(128) PRIMARY KEY COMMENT '랜덤 토큰 (UUID 등)',
   account_id VARCHAR(64) NOT NULL COMMENT '유저 ID',
   position   INT         NOT NULL COMMENT '대기열 내 순번',
   status     VARCHAR(32) NOT NULL COMMENT '대기열 상태',
   issued_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '토큰 발급 시각',
   ready_at   TIMESTAMP COMMENT 'READY 상태로 전환된 시각',
   expires_at TIMESTAMP COMMENT '토큰 만료 시각'
) COMMENT='대기열 토큰 정보';

-- 공연/행사 정보
CREATE TABLE EVENT
(
   id          BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '공연 ID',
   title       VARCHAR(255) NOT NULL COMMENT '공연 제목',
   description TEXT COMMENT '공연 설명',
   venue_id    BIGINT       NOT NULL COMMENT '공연 장소 ID',
   start_date  DATE         NOT NULL COMMENT '공연 시작일',
   end_date    DATE         NOT NULL COMMENT '공연 종료일',
   created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시각',
   updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정 시각'
) COMMENT='공연 정보';

-- 공연장(장소) 정보
CREATE TABLE VENUE
(
   id         BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '공연장 ID',
   name       VARCHAR(255) NOT NULL COMMENT '공연장 이름',
   address    VARCHAR(255) COMMENT '공연장 주소',
   capacity   INT          NOT NULL COMMENT '총 수용 가능한 좌석 수',
   created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시각',
   updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정 시각'
) COMMENT='공연장 정보';

-- 공연장의 좌석 정보
CREATE TABLE VENUE_SEAT
(
   id            BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '좌석 ID',
   venue_id      BIGINT      NOT NULL COMMENT '공연장 ID',
   seat_number   VARCHAR(16) NOT NULL COMMENT '좌석 번호 (예: A12, B5)',
   area         VARCHAR(64)  NOT NULL COMMENT '좌석 구역 (예: A구역, B구역)',
   seat_type     VARCHAR(32) NOT NULL COMMENT '좌석 유형',
   created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시각',
   UNIQUE (venue_id, seat_number)
) COMMENT='공연장 좌석 정보';

-- 공연 회차 정보
CREATE TABLE EVENT_SCHEDULE
(
   id         BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '스케줄 ID',
   event_id   BIGINT NOT NULL COMMENT '공연 ID',
   date       DATE   NOT NULL COMMENT '공연 날짜',
   start_time TIME   NOT NULL COMMENT '공연 시작 시간',
   end_time   TIME COMMENT '공연 종료 시간',
   created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시각',
   UNIQUE (event_id, date, start_time)
) COMMENT='공연 회차 스케줄';

-- 특정 회차의 좌석 예약 정보
CREATE TABLE RESERVATION
(
   id          BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '예약 ID',
   schedule_id BIGINT      NOT NULL COMMENT '공연 스케줄 ID',
   seat_id     BIGINT      NOT NULL COMMENT '좌석 ID',
   account_id  VARCHAR(64) NOT NULL COMMENT '예약한 유저 ID',
   reserved_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '예약 시각',
   UNIQUE (schedule_id, seat_id)
) COMMENT='좌석 예약 정보';
```

## 필수 기능 및 API
1. **유저 토큰 발급 API**
    - 유저의 UUID와 대기열 정보(순서, 잔여 시간 등)를 포함한 토큰 발급
    - 모든 API는 해당 토큰을 통해 대기열 검증 후 이용 가능
2. **예약 가능 날짜/좌석 API**
    - 예약 가능한 날짜 목록 조회
    - 날짜별 예약 가능한 좌석(1~50번) 조회
3. **좌석 예약 및 결제 API**
    - 날짜와 좌석 정보를 입력받아 예약 및 결제 처리
    - 유효한 토큰과 충분한 잔액 검증 후 좌석 소유권 배정
    - 결제 완료 시 대기열 토큰 만료 처리
4. **잔액 충전/조회 API**
    - 사용자 식별자 및 금액을 받아 잔액 충전
    - 사용자 식별자로 잔액 조회

## 시퀀스 다이어그램

### 1. 유저 토큰 발급 API

```mermaid
sequenceDiagram
    actor 사용자
    participant API as API 서버
    participant Queue as 대기열 관리
    participant DB as 데이터베이스

    사용자->>API: 1. 토큰 발급 요청(account_id)
    API->>Queue: 2. 대기열 위치 확인
    Queue->>Queue: 3. 대기 순번(position) 결정
    Queue->>DB: 4. 토큰 정보 저장
    DB-->>Queue: 5. 저장 완료
    Queue-->>API: 6. 토큰 및 대기 정보 반환
    API-->>사용자: 7. 토큰 및 대기 정보 응답
```

### 2. 예약 가능 날짜/좌석 API

```mermaid
sequenceDiagram
    actor 사용자
    participant API as API 서버
    participant Auth as 토큰 검증
    participant DB as 데이터베이스

    사용자->>API: 1. 날짜/좌석 조회 요청(token)
    API->>Auth: 2. 토큰 유효성 검증
    Auth-->>API: 3. 토큰 유효(또는 실패)
    alt 토큰 유효
        API->>DB: 4a. 가능한 날짜/좌석 조회
        DB-->>API: 5a. 조회 결과 반환
        API-->>사용자: 6a. 조회 결과 응답
    else 토큰 무효
        API-->>사용자: 4b. 에러: 유효하지 않은 토큰
    end
```

### 3. 좌석 예약 및 결제 API

```mermaid
sequenceDiagram
    actor 사용자
    participant API as API 서버
    participant Auth as 토큰 검증
    participant Lock as 분산 락 관리자
    participant DB as 데이터베이스

    사용자->>API: 1. 좌석 예약 및 결제 요청(token, 날짜, 좌석번호)
    API->>Auth: 2. 토큰 유효성 검증
    Auth-->>API: 3. 토큰 유효(또는 실패)
    
    alt 토큰 유효
        API->>Lock: 4a. 좌석에 대한 락 획득 시도
        alt 락 획득 성공
            Lock-->>API: 5a. 락 획득 성공
            API->>DB: 6a. 예약 가능 여부 확인
            DB-->>API: 7a. 예약 가능 여부 응답
            
            alt 예약 가능
                API->>DB: 8a-1. 계정 잔액 조회
                DB-->>API: 9a-1. 계정 잔액 응답
                
                alt 잔액 충분
                    API->>DB: 10a-1-1. 트랜잭션 시작
                    API->>DB: 11a-1-1. 좌석 예약 저장
                    API->>DB: 12a-1-1. 잔액 차감
                    API->>DB: 13a-1-1. 결제 내역 저장
                    API->>DB: 14a-1-1. 대기열 토큰 만료 처리
                    API->>DB: 15a-1-1. 트랜잭션 커밋
                    DB-->>API: 16a-1-1. 커밋 완료
                    API-->>사용자: 17a-1-1. 예약 및 결제 성공 응답
                else 잔액 부족
                    API-->>사용자: 10a-1-2. 에러: 잔액 부족
                end
            else 예약 불가
                API-->>사용자: 8a-2. 에러: 이미 예약된 좌석
            end
            
            API->>Lock: 18a. 락 해제
        else 락 획득 실패
            Lock-->>API: 5b. 락 획득 실패
            API-->>사용자: 6b. 에러: 다른 사용자가 예약 중
        end
    else 토큰 무효
        API-->>사용자: 4b. 에러: 유효하지 않은 토큰
    end
```

### 4. 잔액 충전/조회 API

```mermaid
sequenceDiagram
    actor 사용자
    participant API as API 서버
    participant Auth as 토큰 검증
    participant DB as 데이터베이스
    
    alt 잔액 충전
        사용자->>API: 1a. 잔액 충전 요청(account_id, 금액)
        API->>DB: 2a. 계정 정보 조회
        DB-->>API: 3a. 계정 정보 응답
        API->>DB: 4a. 잔액 업데이트 및 히스토리 저장
        DB-->>API: 5a. 업데이트 완료
        API-->>사용자: 6a. 충전 완료 응답
    else 잔액 조회
        사용자->>API: 1b. 잔액 조회 요청(token, account_id)
        API->>Auth: 2b. 토큰 유효성 검증
        Auth-->>API: 3b. 토큰 유효(또는 실패)
        alt 토큰 유효
            API->>DB: 4b-1. 계정 잔액 조회
            DB-->>API: 5b-1. 잔액 정보
            API-->>사용자: 6b-1. 잔액 정보 응답
        else 토큰 무효
            API-->>사용자: 4b-2. 에러: 유효하지 않은 토큰
        end
    end
```

## 인프라 구성도
![infra.png](infra.png)
