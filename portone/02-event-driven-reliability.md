# 02. 이벤트 순서, 멱등성, 재시도, DLQ

## 1. SQS FIFO가 보장하는 것과 보장하지 않는 것

### 예상 질문

`SQS FIFO를 썼으니 메시지가 정확히 한 번, 순서대로 처리되나요?`

### 1분 모범 답변

> 아닙니다. FIFO는 같은 Message Group ID 안에서 메시지 전달 순서를 관리하고 deduplication ID를 기준으로 제한된 시간 동안 중복 전송을 억제하지만, 애플리케이션의 side effect까지 exactly-once로 만들지는 않습니다. consumer가 처리한 뒤 delete 전에 죽으면 같은 메시지를 다시 받을 수 있습니다. 그래서 고객 또는 결제 단위로만 순서를 좁혀 Message Group ID를 정했고, 각 처리 단계는 event ID나 business key와 현재 상태를 이용해 멱등하게 만들었습니다. 서로 무관한 고객은 다른 group으로 병렬 처리했습니다.

### 정확히 설명할 항목

- 순서의 범위는 queue 전체가 아니라 같은 `MessageGroupId`다.
- 한 group은 사실상 직렬 처리되므로 모든 메시지를 같은 group에 넣으면 처리량 병목이 된다.
- content-based deduplication은 message body를 기반으로 하고 message attributes는 포함하지 않는다.
- deduplication window에 의존해 영구 중복 방지를 구현하면 안 된다.
- visibility timeout은 처리 완료 시간이 아니며 연장이 없으면 실행 중 재전달될 수 있다.
- Lambda batch에서 일부만 실패했을 때 partial batch response가 없으면 성공한 record도 다시 처리될 수 있다.

## 2. 멱등 처리는 상태를 먼저 읽는 것으로 충분한가

### 예상 질문

`같은 메시지가 오면 상태를 확인하고 건너뛰었다고 했는데 race condition은 없나요?`

### 모범 답변

> 단순히 상태를 읽고 나중에 쓰는 것만으로는 충분하지 않습니다. 두 consumer가 동시에 `아직 처리 안 됨`을 읽을 수 있기 때문입니다. 내부 DB 변경은 event ID를 key로 한 idempotency record를 조건부 생성하거나, 상태 version을 조건부 갱신하는 방식으로 처리 권한을 원자적으로 획득해야 합니다. 처리 결과도 가능하면 같은 트랜잭션에 저장합니다. 외부 API 호출은 더 어렵습니다. provider가 idempotency key를 지원하면 안정적인 business key를 전달하고, 지원하지 않으면 호출 전 `IN_PROGRESS`를 기록한 뒤 timeout 시 무조건 재호출하지 않고 조회 API나 reconciliation으로 결과를 판별합니다.

### 권장 상태 모델

| 상태 | 의미 | 다음 동작 |
|---|---|---|
| 없음 | 처음 본 event | conditional put으로 IN_PROGRESS 선점 |
| IN_PROGRESS, lease 유효 | 다른 실행이 처리 중 | retry 또는 visibility 연장 |
| IN_PROGRESS, lease 만료 | 이전 실행 중단 가능 | owner/version 조건으로 takeover |
| COMPLETED | side effect와 결과 확정 | 저장된 결과를 반환하거나 skip |
| FAILED_RETRYABLE | 일시 실패 | backoff 후 재시도 |
| FAILED_FINAL | 자동 재시도 불가 | DLQ 또는 운영 판단 |

### 중요한 경계

idempotency record를 `COMPLETED`로 바꾸는 것과 외부 API 성공은 하나의 로컬 transaction으로 묶을 수 없다. 이 불확실성을 숨기지 않고 provider idempotency key, status query, webhook, reconciliation job을 조합한다.

## 3. 순서는 언제 필요한가

### 예상 질문

`모든 결제 이벤트의 순서를 보장해야 하나요?`

### 모범 답변

> 전역 순서는 비용이 크고 대부분 필요하지 않습니다. 먼저 어떤 aggregate에서 어떤 전이가 순서에 민감한지 봅니다. 같은 payment의 `PAID`와 `CANCELLED`, 같은 customer의 한도 변경은 순서가 중요할 수 있지만 서로 다른 고객의 알림은 독립적으로 처리할 수 있습니다. 저는 Message Group ID를 고객 또는 결제 단위로 선택했습니다. 다만 늦게 도착한 이벤트 자체는 FIFO만으로 해결되지 않기 때문에 aggregate version과 상태 전이 규칙을 확인해 오래된 이벤트가 최신 상태를 덮지 못하게 해야 합니다.

### customer ID와 payment ID 중 무엇을 선택하나

| 기준 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| customer ID | 고객 한도와 여러 결제를 직렬화 | heavy customer가 hot group | 고객 단위 한도 불변식 |
| payment ID | 결제별 병렬성 높음 | 고객 공통 한도 경쟁은 별도 해결 | 결제 상태, 웹훅 처리 |
| partner ID | 파트너 rate limit 제어 가능 | 대형 파트너 병목 | provider별 호출 제한 |

하나의 queue에 모든 목적을 억지로 넣기보다 workload와 ordering key가 다르면 queue를 분리할 수 있다.

## 4. 재시도 정책

### 예상 질문

`어떤 오류를 재시도했나요?`

### 모범 답변

> timeout, connection reset, 429, 일부 5xx처럼 일시적일 가능성이 높은 오류만 재시도하고 validation error, 인증 실패, 허용되지 않은 상태 전이 같은 영구 오류는 즉시 실패로 분류합니다. exponential backoff와 jitter를 쓰고 최대 시도 횟수와 전체 시간 budget을 둡니다. 결제처럼 결과가 불명확한 timeout은 일반 오류와 다르게 취급해 provider 조회로 먼저 확인합니다. 재시도가 downstream 장애를 증폭하지 않도록 concurrency limit, rate limit, circuit breaker도 함께 봅니다.

### 예시 정책

| 오류 | 정책 |
|---|---|
| connect timeout | 같은 idempotency key로 backoff retry |
| response timeout | 상태 조회 후 미처리 확인 시 retry |
| HTTP 429 | Retry-After 존중, jitter, 동시성 축소 |
| HTTP 500/502/503/504 | 제한된 retry, circuit breaker 고려 |
| HTTP 400 validation | retry 금지, 입력과 mapping 확인 |
| HTTP 401/403 | 자동 반복 금지, credential 또는 권한 점검 |
| business decline | 기술 retry 금지, 최종 비즈니스 결과로 저장 |

## 5. DLQ는 실패 메시지 창고가 아니다

### 예상 질문

`DLQ에 간 메시지는 어떻게 운영했나요?`

### 모범 답변

> DLQ는 유실 방지 수단일 뿐 복구 프로세스가 없으면 실패 메시지 창고가 됩니다. 원본 event ID, aggregate ID, 실패 단계, error category, 첫 실패와 마지막 실패 시각, attempt count를 확인할 수 있게 하고 DLQ depth와 oldest message age에 alarm을 둡니다. 재처리 전에는 코드나 데이터 원인이 해결됐는지 확인하고, 같은 idempotency key를 유지한 채 격리된 redrive 경로로 소량부터 재처리합니다. 결제 데이터는 무조건 재실행하지 않고 현재 내부 상태와 PG 상태를 먼저 대사합니다.

### EventBridge target DLQ와 consumer DLQ의 차이

- target DLQ: EventBridge가 target queue나 Lambda에 전달하지 못한 실패다. 권한, KMS policy, target availability를 본다.
- consumer DLQ: queue에는 들어왔지만 handler가 반복 실패한 경우다. 코드, payload, downstream, business validation을 본다.
- 둘을 구분해야 장애 위치와 재처리 방법이 명확하다.

## 6. 외부 기관 점검 시간을 상태 기반으로 복구한 방식

### 예상 질문

`READY, PROGRESS, SUCCEED, FAILED와 next execution time 설계를 설명해 주세요.`

### 모범 답변

> 외부 신용평가는 수분 이상 걸리고 정부24나 홈택스 정기 점검도 있어 HTTP 요청 하나에 묶을 수 없었습니다. 작업 상태와 다음 실행 시각을 영속화하고, scheduler가 실행 가능한 작업만 조회해 조건부 쓰기로 선점한 뒤 queue에 넣었습니다. 점검 오류는 일반 exponential retry로 소모하지 않고 알려진 점검 종료 이후로 next execution time을 옮겼습니다. worker가 죽어도 lease나 progress timeout이 지나면 다시 선점할 수 있게 했고, 이미 성공한 단계는 재호출하지 않도록 상태를 확인했습니다.

### 베스트 프랙티스로 보완할 점

- `PROGRESS`보다 `IN_PROGRESS`처럼 용어를 명확히 한다.
- 기술 실패와 비즈니스 실패, 재시도 가능 여부를 한 `FAILED`에 섞지 않는다.
- state, attempt, next_run_at, owner, lease_until, last_error_category, provider_reference를 둔다.
- GSI를 `status + time bucket` 식으로 설계해 scan을 피하고 hot partition을 점검한다.
- scheduler 중복 실행을 정상으로 보고 conditional claim을 사용한다.
- maximum elapsed time과 수동 검토 상태를 둔다.
- 상태 전이와 event 저장을 transaction으로 묶는다.

## 7. 이벤트 스키마와 배포

### 예상 질문

`producer와 consumer를 독립 배포할 때 이벤트 스키마를 어떻게 바꾸나요?`

### 모범 답변

> event envelope에 event type과 schema version을 명시하고, 기본은 optional field를 추가하는 backward-compatible 변경으로 합니다. consumer가 새 필드를 모르는 동안에도 무시할 수 있어야 합니다. field 삭제나 의미 변경은 새 event type 또는 version을 만들고 양쪽이 공존하는 전환 기간을 둡니다. 계약 테스트와 실제 과거 event fixture replay로 호환성을 확인합니다. 민감정보는 편의를 위해 event 전체에 복제하지 않고 최소화합니다.

## 8. 관측해야 할 메트릭

- queue age, depth, in-flight count
- Lambda error, throttle, duration, iterator age
- retry count by error category
- idempotency hit, conflict, stale takeover count
- DLQ ingress와 oldest age
- 처리 end-to-end latency by event type
- provider latency, timeout, 429, 5xx, decline 비율
- 상태별 task 체류 시간
- reconciliation mismatch count

### 공식 참고 자료

- [AWS SQS FIFO delivery logic](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-understanding-logic.html)
- [AWS SQS exactly-once processing](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html)
- [AWS Lambda with SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [AWS Lambda partial batch responses](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html)

