# 05. 포트원 결제 시스템과 PG 추상화

## 1. 포트원의 문제를 한 문장으로 정의하기

> 고객사에는 일관된 결제 API와 운영 경험을 제공하되, 내부에서는 PG사와 결제수단마다 다른 상태, 파라미터, 비동기 결과, 오류, 취소 정책을 정보 손실 없이 다루는 문제입니다.

공통 모델을 깔끔하게 만드는 것만큼 원본 provider 정보와 예외를 보존하는 것이 중요하다.

## 2. 결제 플로우 설계 질문

### 예상 질문

`결제 완료 API를 설계해 보세요.`

### 모범 답변

> 브라우저가 보낸 성공 여부와 금액을 신뢰하지 않습니다. 고객사 서버가 payment ID를 받고 포트원 서버 API로 결제 상태를 조회한 뒤, 내부 주문의 order ID, expected amount, currency, store와 비교합니다. 검증이 끝난 뒤 주문 상태를 조건부로 갱신합니다. client callback이 유실될 수 있으므로 검증된 webhook도 같은 상태 머신으로 들어오게 합니다. callback과 webhook이 동시에 와도 payment ID와 event ID를 기준으로 멱등 처리합니다. 가상계좌 발급은 결제 완료가 아니므로 `VIRTUAL_ACCOUNT_ISSUED`와 `PAID`를 분리합니다.

이는 포트원 공식 V2 가이드의 서버 측 결제 조회와 금액 검증 원칙에 맞는다.

### 상태 예시

```text
READY
  -> PENDING_AUTH
  -> VIRTUAL_ACCOUNT_ISSUED
  -> PAID
  -> PARTIALLY_CANCELLED
  -> CANCELLED

어느 단계에서든 provider 결과에 따라 FAILED 가능
비동기 취소는 CANCEL_REQUESTED -> CANCELLED 또는 CANCEL_FAILED
```

실제 상태명은 포트원 모델을 따른다. 면접에서는 상태 수보다 허용 전이와 late event 처리 원칙을 설명한다.

## 3. PG Adapter 추상화

### 예상 질문

`여러 PG사를 어떻게 추상화하겠습니까?`

### 모범 답변

> core domain에는 `authorize`, `capture`, `cancel`, `query`, `issueBillingKey` 같은 capability를 정의하고 provider adapter가 각 PG request와 response를 변환하게 합니다. 하지만 모든 PG가 같은 기능을 지원한다고 가장하면 안 됩니다. channel별 capability matrix를 별도로 두고, 지원하지 않는 기능은 명시적인 domain error로 반환합니다. normalized status와 error category를 제공하면서도 원본 provider code, message, transaction ID, raw event reference를 보존해 고객 지원과 장애 분석에 사용합니다. provider별 특수 옵션은 core request를 오염시키지 않도록 namespaced extension 또는 typed provider option으로 격리합니다.

### 계층 예시

| 계층 | 책임 |
|---|---|
| Public API | 일관된 request, validation, idempotency contract |
| Payment orchestration | 상태 머신, invariant, routing, retry decision |
| Provider capability | channel과 payment method별 지원 기능 |
| PG adapter | provider protocol, auth, mapping, timeout |
| Raw event store | 원본 webhook과 request reference 보존 |
| Normalized model | 공통 payment, transaction, cancellation 모델 |

### 추상화에서 버리면 안 되는 것

- provider transaction ID와 channel
- original status와 error code
- sync response인지 webhook인지 polling인지
- 금액, currency, tax-free amount
- 부분 취소 history와 remaining cancellable amount
- provider event occurred time과 received time
- test/live mode
- provider-specific receipt와 settlement reference

## 4. 오류 모델

### 예상 질문

`PG 오류 코드를 어떻게 공통화하나요?`

### 모범 답변

> 고객이 행동을 바꿀 수 있는 공통 category와 운영 분석용 원본 code를 함께 제공합니다. 예를 들어 `INVALID_REQUEST`, `AUTHENTICATION_FAILED`, `PAYMENT_DECLINED`, `RATE_LIMITED`, `PROVIDER_UNAVAILABLE`, `UNKNOWN_RESULT`로 normalize하되 `pgCode`와 `pgMessage`는 별도 필드로 보존합니다. retry 가능 여부를 HTTP status 하나로 정하지 않고 operation과 결과 확실성까지 포함합니다. 특히 timeout은 실패가 아니라 결과 불명일 수 있어 query 또는 reconciliation으로 확인합니다.

### 절대 하면 안 되는 것

- 모든 provider 오류를 500으로 변환
- 모든 5xx를 동일 request ID 없이 자동 재시도
- decline을 장애로 집계
- 내부 secret이나 provider raw payload를 고객에게 그대로 노출
- 새로운 provider code를 enum parse 실패로 버림

## 5. Webhook 수신과 발신

### 예상 질문

`포트원 웹훅을 고객사가 안전하게 처리하도록 어떻게 설계할까요?`

### 모범 답변

> endpoint는 빠르게 signature와 timestamp를 검증하고 raw event를 durable하게 저장한 뒤 2xx를 반환하며 실제 처리는 비동기로 넘깁니다. Standard Webhooks 규격의 message ID와 timestamp, signature를 검증하고 replay window와 event ID dedup을 적용합니다. 고객사가 signature 검증을 쓰지 않는 경우에는 webhook body 자체를 신뢰하지 않고 포트원 API로 payment를 재조회할 수 있어야 합니다. 동일 event 재전송과 서로 다른 event의 상태 역전을 모두 고려합니다.

### 포트원 공식 문서와 연결되는 포인트

- V2 웹훅은 공개 endpoint이므로 본문을 그대로 신뢰하면 안 된다.
- Standard Webhooks 기반 signature 검증 또는 API 재조회 전략이 있다.
- `Payment.Paid`, `Payment.Cancelled`, `BillingKey.Issued`처럼 event type이 나뉜다.
- 가상계좌 입금과 비동기 취소 승인처럼 결과가 나중에 확정되는 trigger가 있다.

### 웹훅 발송 시스템 설계

- 고객 endpoint별 delivery ID와 event ID를 분리한다.
- 상태: PENDING, DELIVERING, SUCCEEDED, RETRY_WAIT, EXHAUSTED.
- customer endpoint 단위 rate limit과 circuit breaker를 둔다.
- 2xx만 성공으로 볼지 contract를 문서화한다.
- 3xx redirect는 SSRF와 secret 유출 위험 때문에 자동 추적하지 않는 편이 안전하다.
- private IP, link-local, metadata endpoint를 막아 SSRF를 방어한다.
- DNS rebinding과 redirect 후 재검증을 고려한다.
- request timeout은 짧게 두고 exponential backoff와 jitter를 쓴다.
- payload에는 version을 넣고 재발송해도 같은 event ID와 timestamp 의미를 유지한다.
- 고객이 delivery history, response code, 다음 retry, manual redrive를 볼 수 있게 한다.

## 6. 부분 취소

### 예상 질문

`부분 취소 API의 동시성을 어떻게 처리합니까?`

### 모범 답변

> `sum(successful cancellation amounts) <= paid amount`가 불변식입니다. cancel request마다 idempotency key를 받고 payment의 version과 remaining cancellable amount를 조건부로 예약합니다. provider 호출 전에 내부 상태를 `CANCEL_REQUESTED`로 저장하고, 성공하면 cancellation transaction을 확정합니다. 두 요청이 동시에 같은 잔액을 취소하지 못하게 version 또는 reservation을 사용합니다. timeout은 provider transaction 조회로 확정하고, webhook이 먼저 와도 같은 cancellation ID와 상태 전이를 사용합니다.

### 가상계좌 환불의 차이

포트원 공식 문서처럼 가상계좌는 환불 수령 계좌가 추가로 필요할 수 있고 PG별 특약, 수수료, 처리 시간이 다르다. 공통 cancel API에 무리하게 숨기기보다 payment method capability와 conditional required fields로 명확하게 표현한다.

## 7. 직접 PG에서 취소했을 때

### 예상 질문

`고객이 PG 콘솔에서 직접 취소해 포트원 상태와 달라지면요?`

### 모범 답변

> 공식 경로를 포트원 API와 콘솔로 제한하는 것이 우선이지만 외부 현실을 완전히 통제할 수는 없습니다. provider webhook 또는 주기적 inquiry로 차이를 감지하고 raw provider transaction과 normalized payment를 대사합니다. 자동 보정 가능한 전이는 idempotent state machine으로 반영하고, 금액이나 ownership이 불명확하면 manual review로 보냅니다. 수정은 audit log와 adjustment reason을 남깁니다.

## 8. Smart routing

### 예상 질문

`PG 라우팅을 어떻게 설계할까요?`

### 모범 답변

> 먼저 merchant contract와 payment method capability, 국가와 currency 같은 hard constraint로 후보를 좁힙니다. 그다음 성공률, latency, cost, 장애 상태를 점수화하되 짧은 구간의 noise에 과민 반응하지 않게 window와 minimum sample을 둡니다. 같은 payment의 retry가 다른 provider로 넘어가 중복 승인되지 않도록 routing decision을 attempt에 고정하고, 결과 불명 상태에서는 failover보다 기존 provider 조회를 우선합니다. 정책 변경은 version과 audit을 남기고 shadow evaluation과 제한된 rollout으로 검증합니다.

## 9. 글로벌 결제에서 추가되는 문제

- currency와 minor unit 차이, zero-decimal currency
- FX rate 시점과 환율 source
- timezone, settlement date, business day
- 3DS와 SCA, 지역별 인증 흐름
- async payment method와 긴 pending 상태
- chargeback, dispute, refund 기간
- data residency와 PCI scope
- provider별 idempotency와 webhook semantics
- local payment method의 redirect와 mobile app return

### 답변 연결

> 저는 KCB와 KCS 연동에서 외부 기관의 redirect, callback, 점검 시간, provider-specific error를 내부 상태로 흡수해 본 경험이 있습니다. 포트원에서는 그 원칙을 결제수단과 PG별 capability, raw data 보존, normalized state machine으로 확장할 수 있습니다.

## 10. 간단한 시스템 디자인 답변 구조

1. 기능: 결제 요청, 조회, 취소, webhook, billing key.
2. 비기능: correctness 우선, availability, latency, audit, security.
3. ID: merchant payment ID, internal payment ID, provider transaction ID.
4. 상태 머신과 invariant.
5. sync API와 async event 경계.
6. idempotency와 unknown outcome.
7. PG adapter와 capability matrix.
8. storage와 Outbox.
9. reconciliation과 운영 도구.
10. metric, rollout, failure drill.

### 공식 참고 자료

- [포트원 V2 인증 결제 연동](https://developers.portone.io/opi/ko/integration/start/v2/checkout?v=v2)
- [포트원 V2 웹훅 연동](https://developers.portone.io/opi/ko/integration/webhook/readme-v2?v=v2)
- [포트원 V2 결제 취소](https://developers.portone.io/opi/ko/integration/cancel/v2/readme?v=v2)
- [포트원 V2 REST API](https://developers.portone.io/api/rest-v2/payment?v=v2)

