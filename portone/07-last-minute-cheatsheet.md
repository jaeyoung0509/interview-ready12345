# 07. 면접 직전 압축 복습

## 1. 내 핵심 메시지 세 문장

1. 결제와 한도처럼 함께 바뀌어야 하는 상태는 조건부 쓰기와 트랜잭션으로 불변식을 지켰다.
2. 외부 기관과 후속 작업은 실패와 중복을 전제로 상태를 영속화하고 멱등성, 재시도, DLQ, 대사 경로를 만들었다.
3. 포트원에서도 PG 차이를 공통 모델로 추상화하되 원본 상태와 오류를 잃지 않고 운영 가능한 시스템을 만드는 데 기여할 수 있다.

## 2. 10초 정의

| 개념 | 답변 |
|---|---|
| DynamoDB transaction | 여러 item 쓰기를 한 리전에서 all-or-nothing으로 묶는 ACID operation |
| Conditional write | 현재 version이나 상태가 예상과 같을 때만 쓰게 하는 optimistic concurrency 도구 |
| Distributed lock | 흐름을 직렬화하는 lease 기반 조정 수단. 최종 DB 불변식의 대체물이 아님 |
| Outbox | 상태 변경과 발행할 event를 같은 DB transaction에 저장해 dual write를 피하는 패턴 |
| Idempotency | 같은 논리 요청이 반복돼도 효과가 한 번 반영된 것과 같게 만드는 성질 |
| SQS FIFO | 같은 message group 안의 순서와 제한된 dedup 기능. exactly-once side effect는 아님 |
| DLQ | 반복 실패 격리 수단. alarm, triage, safe redrive가 함께 있어야 함 |
| HMAC | shared secret 기반 message integrity와 authenticity. encryption이나 replay 방지는 아님 |
| AES-CBC | 기밀성은 주지만 인증이 없는 block cipher mode. 신규 설계는 AEAD 선호 |
| KMS | key material과 암복호화 권한을 중앙 통제하는 서비스. IAM과 encryption scope 설명 필요 |
| CORS | browser cross-origin response access policy. authentication이 아님 |
| ULID | 시간 정렬 가능한 unique ID. authorization 수단이 아님 |

## 3. 숫자와 제한

- DynamoDB `TransactWriteItems`: 최대 100 actions, 4MB, 같은 account와 Region.
- SQS FIFO dedup: 제한된 deduplication interval만 제공. 영구 dedup으로 사용하지 않는다.
- Lambda SQS: batch가 중복 전달될 수 있다. partial batch failure를 고려한다.
- DynamoDB TTL: 만료 시각에 즉시 삭제된다고 보장하지 않는다.
- 금액: float 금지, integer minor unit + currency.

## 4. 무조건 먼저 말할 실패 시나리오

| 주제 | 실패 시나리오 |
|---|---|
| 결제 승인 | 동시 요청, 응답 유실, 중복 요청 |
| 취소 | 부분 취소 경쟁, PG timeout, late webhook |
| Outbox | relay 중복, poison event, stream lag |
| Queue | visibility timeout, batch 일부 실패, hot message group |
| 외부 API | 429, 5xx, 점검, 결과 불명 timeout |
| Webhook | 위조, replay, 중복, 역순, 고객 endpoint 장애 |
| Projection | 중복, 역순, lag, source와 불일치 |
| Lock | owner crash, lease expiry, stale owner |

## 5. 위험 문장 금지

- `완벽하게 보장했습니다`
- `정확히 한 번 처리됩니다`
- `CORS로 보안을 막았습니다`
- `ULID라 해킹할 수 없습니다`
- `HMAC이 replay를 막습니다`
- `암호화했으니 개인정보가 아닙니다`
- `MSA라 확장성이 좋습니다`
- `NoSQL이 RDB보다 빠릅니다`

대신 보장 범위와 남은 실패를 정확히 말한다.

## 6. 모범 답변 만능 골격

> 당시 문제의 핵심 불변식은 ___였습니다. ___ 경쟁 또는 실패가 생기면 ___가 깨질 수 있었습니다. 그래서 ___를 적용했고, 구체적으로 ___ 조건에서만 상태를 변경했습니다. 전달과 외부 side effect는 중복될 수 있어 ___로 멱등하게 처리했습니다. 운영에서는 ___ metric과 ___ 복구 경로를 뒀습니다. 이 선택의 한계는 ___이고, 현재 다시 설계한다면 규모와 요구에 따라 ___도 비교하겠습니다.

## 7. 마지막으로 확인할 개인 사실

- 분산 락의 실제 key, owner, lease, release 구현
- transaction에 들어간 정확한 item 목록
- idempotency record 또는 상태 필드의 실제 이름
- Message Group ID가 customer인지 payment인지 event별 구분
- DLQ redrive를 실제로 누가 어떻게 했는지
- P99 측정 기간과 표본, warm-up ping의 스케줄
- KCB/KCS 암호화 규격 중 provider 요구와 직접 선택한 부분
- QueryCriteria 대표 API 하나와 before/after 코드 구조
- 장애 사례 하나와 본인의 직접 기여
- Moonberg 60~80% 절감의 계산 근거

기억나지 않으면 추정해서 채우지 않는다. 면접에서는 정확한 경계를 말하는 것이 깊이로 보인다.

