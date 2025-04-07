# API 문서

본 문서는 Flask 기반 서버에서 제공하는 **입금/주문/내역** 관련 API들을 정리한 것입니다.  
외부 사용자(클라이언트, 프론트엔드, 기타 서비스 등) 혹은 내부 운영팀에서 참고하여 원하는 기능을 연동할 수 있습니다.

> 주의: 본 예시 코드에는 인증/인가 로직이 없습니다. 실제 운영 환경에서는 반드시 인증/인가(Access Control), 민감 정보 보안(x-api-key, x-mall-id), 입력값 검증 등이 필요합니다.

-------------------------------------------------------------------------------

## 1. /deposit-request (POST)

주문(입금 요청) 생성  
사용자가 결제를 요청할 때 호출하며, 내부적으로 DB에 주문 정보를 저장하고 Payaction 결제 주문을 생성합니다.

### 요청 (Request)

- URL: /deposit-request
- Method: POST

- Headers:

  Key            | Value                | Description
  -------------- | -------------------- | -----------------------------
  Content-Type   | application/json     | 요청 본문이 JSON 형식임을 지정

- Body: JSON

  Field          | Type   | Required | Example       | Description
  -------------- | ------ | -------- | ------------- | ---------------------------------------
  username       | string | 예       | "test_user"   | 사용자 아이디
  billing_name   | string | 예       | "홍길동"      | 입금자명 (계좌 이체 시 실제 입금인)
  amount         | number | 예       | 50000         | 주문 금액(결제 요청 금액)

예시 요청:
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "username": "test_user",
    "billing_name": "홍길동",
    "amount": 50000
  }' \
  http://<SERVER_DOMAIN_OR_IP>/deposit-request
```

### 응답 (Response)

성공 (HTTP 200)

```json
{
  "status": "success",
  "bank": "국민은행",
  "account": "11400204283485",
  "order_number": "123456"
}
```

필드       | 타입   | 설명
---------- | ------ | ---------------------------------------------
status     | string | 성공 여부. 항상 "success"
bank       | string | 입금 은행명
account    | string | 입금 은행 계좌번호
order_number | string | 서버가 생성한 주문 번호(랜덤 6자리 등)

실패 (HTTP 400 or 500)

```json
{
  "status": "fail",
  "msg": "에러 메시지"
}
```

필드   | 타입   | 설명
------ | ------ | --------------------------------------
status | string | 실패 여부. 항상 "fail"
msg    | string | 오류 혹은 실패 상세 원인 안내

-------------------------------------------------------------------------------

## 2. /webhook (POST)

Payaction으로부터의 Webhook 처리  
Payaction에서 입금 매칭이 완료되었을 때(주문 상태가 매칭완료), 이 엔드포인트를 호출해 DB의 주문 상태를 업데이트하고 사용자 잔액을 증가시킵니다.

주의: 실제 운영 시에는 Webhook 인증/검증(서명 등)이 필요할 수 있습니다.

### 요청 (Request)

- URL: /webhook
- Method: POST

- Headers:

  Key            | Value              | Description
  -------------- | ------------------ | --------------------------------
  Content-Type   | application/json   | 요청 본문이 JSON 형식임을 지정

- Body: JSON (Payaction에서 전송하는 예시)

```json
{
  "order_number": "123456",
  "order_status": "매칭완료"
  // ... 기타 필요한 필드들
}
```

### 동작 프로세스

1) order_status가 "매칭완료"인지 확인.
2) order_number로 DB에서 주문 정보를 조회.
3) 주문이 존재하면 상태를 "매칭완료"로 업데이트 후, username 사용자의 잔액(amount)을 주문 금액만큼 증가.
4) 처리 후 JSON 응답 반환.

### 응답 (Response)

항상 HTTP 200
```json
{
  "status": "success"
}
```

-------------------------------------------------------------------------------

## 3. /cancel-order (POST)

주문 취소  
이미 생성된 주문(order_number)을 Payaction API를 통해 취소하고, DB에서도 주문 상태를 "취소됨"으로 업데이트합니다.

### 요청 (Request)

- URL: /cancel-order
- Method: POST

- Headers:

  Key            | Value              | Description
  -------------- | ------------------ | -----------------------------
  Content-Type   | application/json   | 요청 본문이 JSON 형식임을 지정

- Body: JSON

  Field         | Type   | Required | Example   | Description
  ------------- | ------ | -------- | --------- | ----------------------
  order_number  | string | 예       | "123456"  | 취소할 주문 번호

예시 요청:
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "order_number": "123456"
  }' \
  http://<SERVER_DOMAIN_OR_IP>/cancel-order
```

### 응답 (Response)

성공 (HTTP 200)

```json
{
  "status": "success",
  "msg": "주문이 취소되었습니다",
  "payaction_response": { ... }
}
```

필드                | 타입   | 설명
------------------- | ------ | -------------------------------------------------
status             | string | "success"
msg                | string | 취소 성공 메시지
payaction_response | object | Payaction 취소 API 응답 본문

실패 (HTTP 4xx or 5xx)

```json
{
  "status": "fail",
  "msg": "PayAction API 오류",
  "payaction_response": "...오류 상세..."
}
```

필드                | 타입   | 설명
------------------- | ------ | -----------------------------------------
status             | string | "fail"
msg                | string | 실패 원인(예: API 오류 등)
payaction_response | string | Payaction API에서 온 상세 오류 내용

-------------------------------------------------------------------------------

## 4. /transaction-history/<username> (GET)

사용자별 거래(주문) 내역 조회  
특정 사용자(username)가 발행한 주문 내역을 날짜 역순으로 조회.

### 요청 (Request)

- URL: /transaction-history/<username>
- Method: GET

예시:
```bash
curl -X GET \
  http://<SERVER_DOMAIN_OR_IP>/transaction-history/test_user
```

### 응답 (Response)

성공 (HTTP 200)

```json
{
  "status": "success",
  "transactions": [
    {
      "order_number": "123456",
      "amount": 50000,
      "date": "2025-04-06 14:30:25",
      "status": "대기중"
    }
  ]
}
```

필드          | 타입    | 설명
------------- | ------- | -------------------------------------------------
status        | string  | "success"
transactions  | array   | 주문 내역 리스트
└ order_number | string | 주문 번호
└ amount       | number | 금액
└ date         | string | 주문 날짜(한국 시간, YYYY-MM-DD HH:mm:ss)
└ status       | string | 주문 상태 (대기중, 매칭완료, 취소됨 등)

실패 (HTTP 500)

```json
{
  "status": "fail",
  "msg": "에러 설명"
}
```

필드   | 타입   | 설명
------ | ------ | ---------------------------------
status | string | "fail"
msg    | string | 오류 혹은 실패 상세 원인 안내

-------------------------------------------------------------------------------

## 5. /admin/transactions (GET)

관리자용 전체 거래 내역 조회  
상태가 매칭완료(또는 완료)인 모든 주문을 조회.

### 요청 (Request)

- URL: /admin/transactions
- Method: GET

예시:
```bash
curl -X GET \
  http://<SERVER_DOMAIN_OR_IP>/admin/transactions
```

### 응답 (Response)

성공 (HTTP 200)

```json
{
  "status": "success",
  "deposits": [
    {
      "order_number": "123456",
      "username": "test_user",
      "amount": 50000,
      "date": "2025-04-06 14:30:25",
      "status": "매칭완료",
      "billing_name": "홍길동",
      "wallet_address": "Txx... (랜덤)"
    }
  ]
}
```

필드                | 타입    | 설명
------------------- | ------ | -------------------------------------------
status             | string | "success"
deposits           | array  | 매칭(완료)된 거래 내역 리스트
└ order_number     | string | 주문 번호
└ username         | string | 사용자 아이디
└ amount           | number | 금액
└ date             | string | 주문 날짜(한국 시간, YYYY-MM-DD HH:mm:ss)
└ status           | string | 주문 상태 (매칭완료, 완료)
└ billing_name     | string | 입금자명
└ wallet_address   | string | 임의 지갑 주소(예시)

실패 (HTTP 500)

```json
{
  "status": "fail",
  "msg": "에러 설명"
}
```

필드   | 타입   | 설명
------ | ------ | -------------------------------
status | string | "fail"
msg    | string | 오류 혹은 실패 상세 원인 안내

-------------------------------------------------------------------------------

## 동작 순서 요약

1) 사용자 입금/주문 요청 (POST /deposit-request)
   - 요청 받은 후 DB에 주문 정보 저장 → Payaction API로 주문 생성 → 결과 반환

2) Payaction Webhook (POST /webhook)
   - 입금 매칭 완료 시 (매칭완료) 웹훅을 서버에 전송 → DB 주문 상태 업데이트 및 사용자 잔액 증가

3) 주문 취소 (POST /cancel-order)
   - Payaction API에 취소 요청 → DB 상태 "취소됨"으로 업데이트

4) 입금(주문) 내역 조회 (GET /transaction-history/<username>)
   - 특정 사용자의 주문 이력을 반환

5) 관리자 전체 거래 내역 조회 (GET /admin/transactions)
   - 매칭완료(완료)된 모든 거래를 리스트로 조회

-------------------------------------------------------------------------------

## 환경 및 보안 주의 사항

1) DB 연결 정보(db_config)
   - 운영 시, DB 호스트/유저/비밀번호가 코드에 노출되지 않도록 환경변수나 설정 파일로 분리

2) 민감 정보(x-api-key, x-mall-id)
   - 예시에서는 Payaction API 키 등이 하드코딩되어 있음
   - 실제 운영에서는 환경변수 또는 안전한 비밀 저장소를 사용

3) 인증/인가(Access Control)
   - 예시 코드에는 외부 접근 제한이 없음
   - 반드시 토큰, 세션, 방화벽 등을 활용해 무단 접근을 방지

4) 입력값 검증
   - amount가 음수인지, username이 존재하는지, billing_name에 이상한 문자가 있는지 등에 대한 서버 측 검증 필요

5) 예외 처리
   - Payaction API 호출 실패, DB 오류 등을 위한 재시도 로직과 명확한 에러 핸들링 권장

-------------------------------------------------------------------------------

## 요구 라이브러리

pip install flask pymysql requests

- Flask : 웹 서버 프레임워크
- PyMySQL : MySQL 연결
- requests : Payaction API 호출
- (optional) logging : 서버 로그 기록

-------------------------------------------------------------------------------

## 라이선스

- 본 코드는 예시이므로, 그대로 사용 시 발생하는 문제(보안 이슈, 데이터 손실, API 키 도용 등)에 대한 책임은 사용자에게 있습니다.
- 상용 환경에서는 반드시 보안 강화, 오류 처리, 로그 관리, 인증 등을 추가한 뒤 사용하세요.
