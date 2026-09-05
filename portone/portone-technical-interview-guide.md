# 코리아포트원 기술 면접 완전 대비

이재영님의 이력서와 과거 보안 구현을 기준으로 예상 질문, 1분 모범 답변, 심화 답변, 꼬리 질문, 베스트 프랙티스와 포트원 결제 시스템 디자인을 한 문서에 정리했습니다.

## 목차

1. 이력서 기반 기술 면접 지도
2. 결제 정합성, DynamoDB 트랜잭션, 분산 락, Outbox
3. 이벤트 순서, 멱등성, 재시도, DLQ
4. 보안 심층 면접 대비
5. 데이터 모델, API 추상화, Lambda, 관측성
6. 포트원 결제 시스템과 PG 추상화
7. 프로젝트, 오픈소스, 30문항 모의 면접
8. 면접 직전 압축 복습

---

## 00. 이력서 기반 기술 면접 지도

### 1. 면접관이 가장 깊게 볼 주장

| 우선순위 | 이력서의 주장 | 확인하려는 것 | 준비 문서 |
|---|---|---|---|
| S | 결제 승인, 한도 변경, Outbox를 트랜잭션으로 커밋 | 불변식, 동시성 제어, 실패 경계 | 01 |
| S | 고객 단위 분산 락 | 락이 정말 필요한지, lease와 장애 복구 | 01 |
| S | Streams, EventBridge, SQS FIFO | 전달 보장과 순서 보장의 정확한 범위 | 02 |
| S | 상태 확인 기반 멱등 처리 | 원자성, 외부 API 중복 호출, idempotency key | 02 |
| S | KCB HMAC, AES 복호화, CI 해시 | 암호화와 무결성의 차이, replay, 키 관리 | 03 |
| A | JWT claims로 customerId 강제 주입 | 인증과 인가, IDOR, tenant boundary | 03 |
| A | Lambda P99 2~3초에서 200ms대 | 측정 방법, 워머 한계, Provisioned Concurrency | 04 |
| A | DynamoDB와 PostgreSQL 분리 | source of truth, projection lag, reconciliation | 04 |
| A | QueryCriteria로 2~3일을 3~4시간으로 단축 | 추상화 경계, 쿼리 성능과 보안 | 04 |
| A | PG사와 파트너 정책 추상화 | 포트원 도메인으로의 전이 가능성 | 05 |
| B | Moonberg, Temporal, Genkit, Chalice | 개인 기여 범위와 기술적 호기심 | 06 |

### 2. 90초 경력 요약 모범 답변

> 파이노버스랩에서 3년 7개월 동안 B2B BNPL의 결제, 신용한도, 전자계약, 정산 백엔드를 개발하고 운영했습니다. 제가 주로 해결한 문제는 여러 외부 기관과 비동기 처리가 연결된 환경에서 결제 상태와 한도가 어긋나지 않게 만들고, 중간 실패가 나도 운영자가 복구할 수 있게 하는 일이었습니다. 결제 승인과 한도 차감처럼 하나의 불변식으로 묶이는 변경은 DynamoDB 트랜잭션과 조건부 쓰기로 처리했고, 전자계약과 알림처럼 별도로 재시도할 수 있는 일은 Outbox, Streams, EventBridge, SQS로 분리했습니다. 이 과정에서 순서는 고객 또는 결제 단위로만 보장하고, 소비자는 중복 전달을 전제로 멱등하게 만들었습니다. KCB 본인인증과 KCS 신용평가 연동에서는 서명 검증, 민감정보 암호화, JWT claim 기반 인가와 로그 마스킹을 다뤘습니다. 포트원에서도 여러 PG의 서로 다른 상태와 실패 의미를 공통 모델로 추상화하면서도 원본 정보를 잃지 않는 문제가 제 경험과 가장 직접적으로 연결된다고 생각합니다.

### 3. 답변 깊이를 보여주는 구조

#### 첫 문장

기술 이름보다 불변식을 먼저 말한다.

나쁜 답변:

> DynamoDB 트랜잭션과 분산 락을 썼습니다.

좋은 답변:

> 승인된 결제 금액의 합이 고객 한도를 넘지 않고, 승인 상태와 잔여 한도가 항상 같이 바뀌어야 했습니다.

#### 본문

정상 흐름보다 경쟁과 실패를 설명한다.

> 같은 고객의 승인 두 건이 동시에 들어오면 둘 다 이전 잔여 한도를 읽을 수 있습니다. 그래서 조건부 갱신 또는 트랜잭션 안의 조건 검사로 한도 버전을 확인했고, 한도 차감, 결제 생성, Outbox 기록을 한 번에 커밋했습니다. 충돌한 요청은 최신 상태를 다시 읽고 제한된 횟수만 재시도했습니다.

#### 마지막 문장

대안과 한계를 스스로 말한다.

> 지금 다시 설계한다면 트랜잭션의 조건부 갱신만으로 불변식을 보장할 수 있는지 먼저 검토하고, 별도의 분산 락은 외부 호출까지 직렬화해야 하는 명확한 이유가 있을 때만 쓰겠습니다.

### 4. 반드시 정정해서 말할 표현

| 위험한 표현 | 정확한 표현 |
|---|---|
| SQS FIFO라 exactly-once 처리된다 | FIFO의 중복 제거는 제한된 범위의 전달 기능이다. 소비자는 at-least-once를 전제로 멱등해야 한다 |
| 분산 락으로 정합성을 완벽히 보장했다 | 최종 불변식은 조건부 쓰기와 트랜잭션이 보장한다. 락은 경쟁을 줄이거나 흐름을 직렬화한다 |
| HMAC으로 replay까지 막았다 | HMAC은 무결성과 인증을 제공한다. replay는 만료, nonce, 1회성 상태 소비로 별도 방어한다 |
| CORS로 비인가 호출을 막았다 | CORS는 브라우저의 cross-origin 읽기를 제한할 뿐 인증이나 서버 간 호출을 막지 않는다 |
| 내부 API 키로 상호 인증했다 | 공유 API 키 기반 호출자 인증이다. 진정한 상호 인증은 mTLS나 workload identity가 필요하다 |
| AES-CBC E2EE를 구현했다 | 자사 서버에서 평문을 받아 암호화했다면 외부 기관 프로토콜용 payload 암호화다. 종단간 암호화라고 단정하지 않는다 |
| CI를 해시해 비식별화했다 | 결정적 가명처리다. 원본과 연결 가능성이 남으므로 개인정보 보호 대상이다 |
| ULID라 열거 공격이 불가능하다 | ULID의 random 부분은 추측이 어렵지만 timestamp가 노출된다. 인가는 별도로 필요하다 |
| 웜업 핑으로 콜드스타트를 해결했다 | 관측 구간에서 지연을 줄였지만 보장 수단은 아니다. 확실한 용량 보장은 Provisioned Concurrency가 적합하다 |
| DynamoDB Streams가 정확히 한 번 전달한다 | Streams와 Lambda 소비는 중복 가능성이 있다. checkpoint와 멱등 소비가 필요하다 |

### 5. 사실 확인이 필요한 수치

면접 전 본인이 확인할 수 있으면 아래를 채운다. 기억나지 않으면 범위를 꾸미지 말고 측정 조건을 설명한다.

| 주장 | 확인할 자료 | 답변 시 주의 |
|---|---|---|
| P99 2~3초에서 200ms대 | CloudWatch 기간, 표본 수, 함수 이름 | warm invocation과 전체 API latency를 구분 |
| 신용평가 10분 이내 | 정상 시간대 기준 percentile | 점검 시간과 외부 기관 장애 제외 조건 명시 |
| 수작업 주 15~20시간 제거 | 담당 인원, 기존 절차, 계산 근거 | 개발 전후 운영 프로세스 비교 |
| 조회 API 2~3일에서 3~4시간 | 대표 API 1개와 공통화 범위 | 모든 API가 그랬다고 일반화하지 않기 |
| Moonberg 60~80% 절감 | 작업 유형과 샘플 기간 | 추정인지 계측인지 밝히기 |

### 6. 모르는 질문을 받았을 때

> 그 부분은 당시 제가 직접 소유한 범위 밖이라 실제 구현을 했다고 말씀드리기는 어렵습니다. 다만 제가 설계한다면 먼저 깨지면 안 되는 불변식을 정의하고, 이 경우에는 A와 B를 비교하겠습니다. 현재 정보만 보면 A를 택하되, X 메트릭으로 가정을 검증하겠습니다.

이 답변은 회피가 아니라 소유 범위와 추론을 분리한다. 모르는 내부 구현을 지어내는 것보다 훨씬 안전하다.


---

## 01. 결제 정합성, DynamoDB 트랜잭션, 분산 락, Outbox

### 1. 어떤 정합성 문제였나

#### 예상 질문

`결제 승인과 한도 변경이 어긋난다는 게 구체적으로 어떤 상황인가요?`

#### 1분 모범 답변

> 고객의 잔여 한도가 10만 원일 때 8만 원 승인 요청 두 건이 동시에 들어오는 상황이 대표적입니다. 둘 다 10만 원을 읽은 뒤 각각 승인하면 총 16만 원이 승인될 수 있습니다. 저희의 핵심 불변식은 승인된 결제 총액이 가용 한도를 넘지 않는 것, 그리고 결제 승인 생성과 한도 차감이 함께 성공하거나 함께 실패하는 것이었습니다. 이를 위해 한도 레코드에 조건부 갱신을 걸고, 결제 승인, 한도 차감, Outbox 이벤트 생성을 `TransactWriteItems` 하나로 커밋했습니다. 조건 충돌은 정상적인 동시성 결과로 보고 제한적으로 재시도하거나 한도 부족으로 반환했습니다.

#### 3분 심화 포인트

- 읽고 계산한 뒤 쓰는 흐름만으로는 lost update가 발생한다.
- 가능한 경우 `remaining_limit >= amount` 조건과 차감 연산을 한 item의 atomic update로 표현한다.
- 결제 item 생성에는 `attribute_not_exists(pk)`를 넣어 같은 결제 ID의 재생성을 막는다.
- Outbox item도 같은 트랜잭션에 넣어 DB 커밋과 이벤트 발행 의도의 dual write를 없앤다.
- 트랜잭션 응답을 잃어 결과가 불명확할 수 있으므로 안정적인 `ClientRequestToken` 또는 비즈니스 idempotency key가 필요하다.
- 충돌 재시도에는 작은 횟수 제한과 jitter를 둔다. 한 고객이 hot key가 되면 무작정 재시도하지 않는다.

#### 면접관의 꼬리 질문

`왜 락까지 필요했나요? 트랜잭션 조건부 쓰기면 충분하지 않나요?`

> 맞습니다. DB 내부 불변식만 보면 트랜잭션과 조건부 쓰기가 최종 안전장치이고, 별도 락이 항상 필요한 것은 아닙니다. 당시 락은 같은 고객의 긴 승인 흐름이 겹치는 것을 줄이고 충돌을 앞단에서 제어하려는 목적이었습니다. 다만 락이 트랜잭션보다 안전한 것은 아니고, lease 만료나 소유권 상실 문제가 있습니다. 지금 다시 설계한다면 외부 API 호출을 포함한 흐름을 꼭 직렬화해야 하는지 먼저 확인하고, DB 변경은 조건부 트랜잭션만으로 처리하는 단순한 설계를 우선 검토하겠습니다.

### 2. DynamoDB 트랜잭션은 무엇을 보장하나

#### 예상 질문

`TransactWriteItems의 isolation 수준과 제한은 무엇인가요?`

#### 모범 답변

> 한 리전과 한 계정 안에서 최대 100개의 서로 다른 item에 대한 쓰기를 all-or-nothing으로 수행하고, 전체 item 크기는 4MB 제한이 있습니다. 트랜잭션 쓰기끼리와 일반적인 단일 item 쓰기 사이에는 serializable isolation이 적용되는 범위가 있지만, GSI와 Streams에서 즉시 하나의 원자적 스냅샷으로 관찰된다고 생각하면 안 됩니다. 트랜잭션은 각 item마다 prepare와 commit에 해당하는 용량을 사용해 일반 쓰기보다 비용이 높고, 같은 item에 경쟁이 많으면 transaction conflict가 납니다. 따라서 작은 aggregate의 핵심 불변식에만 쓰고, 리포트성 데이터나 후속 처리는 비동기로 분리하는 게 좋습니다.

#### 반드시 기억할 제한

- 같은 item에 `ConditionCheck`와 `Update`를 동시에 넣을 수 없다. 조건은 Update의 condition expression으로 합친다.
- GSI를 대상으로 직접 transaction action을 수행할 수 없다.
- Streams record가 트랜잭션 단위로 묶여 한 번에 보인다고 가정하지 않는다.
- global table의 일반적인 multi-region eventual consistency 모드에서는 다른 리전까지 하나의 글로벌 ACID 트랜잭션이 아니다.
- 취소 사유에는 조건 실패, 충돌, 처리량, validation 등이 섞일 수 있어 구분과 metric이 필요하다.

#### 베스트 프랙티스

1. transaction item 수를 작고 고정되게 유지한다.
2. user input 전체가 아니라 안정적인 business request ID로 idempotency를 건다.
3. `CancellationReasons`를 관측해 한도 부족과 시스템 충돌을 구분한다.
4. hot partition과 특정 고객의 경합률을 metric으로 본다.
5. transaction 바깥 side effect는 절대 같은 성공으로 간주하지 않고 Outbox로 연결한다.

### 3. 분산 락을 구현했다면 어디까지 설명해야 하나

#### 예상 질문

`DynamoDB 조건부 쓰기 락의 구조를 설명해 주세요.`

#### 안전한 모범 답변

> 락 item은 customer ID를 key로 하고 owner token과 lease expiry를 저장합니다. 획득은 item이 없거나 lease가 만료된 경우에만 성공하는 조건부 쓰기로 하고, 갱신과 해제는 owner token이 현재 실행자와 같을 때만 허용해야 합니다. 프로세스가 죽어도 lease가 지나면 다른 실행자가 획득할 수 있습니다. 다만 오래 멈춘 실행자가 lease 만료 후 다시 살아나 쓰기를 수행하는 stale owner 문제가 남기 때문에, 중요한 쓰기는 락만 믿지 않고 DB version 조건이나 fencing token으로 거부해야 합니다. DynamoDB TTL은 삭제 시각을 보장하지 않으므로 lock correctness에 TTL 삭제 자체를 사용하지 않고, 애플리케이션이 expiry 값을 조건식으로 판단해야 합니다.

#### 면접 전에 실제 코드에서 확인할 것

- lock item의 PK
- owner token이 매 요청마다 고유했는지
- lease 기간과 연장 방식
- release가 소유자 조건부 delete였는지
- Lambda timeout보다 lease가 길었는지
- stale owner를 막는 version 또는 fencing token이 있었는지
- 락 획득 후 외부 API를 호출했는지

확인하지 못한 항목은 구현했다고 단정하지 않는다.

### 4. Outbox는 어떤 문제를 풀었나

#### 예상 질문

`결제 저장 후 EventBridge에 바로 publish하면 안 되나요?`

#### 모범 답변

> DB 저장과 메시지 publish는 서로 다른 시스템이라 원자적으로 묶이지 않습니다. DB는 성공했는데 publish가 실패하면 후속 계약이나 정산이 영원히 시작되지 않고, publish 후 DB가 실패하면 존재하지 않는 결제를 소비자가 보게 됩니다. 그래서 결제 상태와 발행할 Outbox record를 같은 DynamoDB 트랜잭션에 저장했습니다. Streams relay는 Outbox record를 읽어 EventBridge로 전달합니다. 다만 이것은 exactly-once가 아니라 at-least-once relay이므로 event ID를 고정하고 소비자가 중복을 제거해야 합니다.

#### Relay 설계 베스트 프랙티스

- Outbox event에는 `event_id`, `aggregate_id`, `aggregate_version`, `event_type`, `occurred_at`, `schema_version`, 최소 payload를 둔다.
- 소비자가 DB를 다시 읽어야 한다면 현재 상태와 event 시점 상태가 다를 수 있음을 고려한다.
- publish 성공 표시를 별도 update할 경우 그 update가 다시 Stream을 만들지 filter한다.
- event source mapping의 partial batch response와 실패 record 보존 전략을 정한다.
- 오래된 stream record가 retention을 넘기기 전에 장애를 알리는 iterator age alarm을 둔다.
- event schema는 additive change를 기본으로 하고 producer와 consumer의 독립 배포를 테스트한다.
- 삭제 여부와 보존 기간은 감사 요구와 재처리 전략으로 결정한다.

#### 꼬리 질문

`Streams에서 같은 트랜잭션의 여러 변경 순서가 보장되나요?`

> 같은 item의 변경 순서는 shard 내에서 보존되지만, 서로 다른 item이나 shard 전체에 대한 전역 순서는 가정하면 안 됩니다. 따라서 결제와 한도 record가 보이는 순서에 의존하지 않고, 하나의 aggregate version이나 소비자의 상태 검증으로 처리해야 합니다.

### 5. 취소와 환불의 동시성

#### 예상 질문

`승인과 취소가 동시에 들어오면 어떻게 합니까?`

#### 모범 답변

> 먼저 허용 가능한 상태 전이를 정의합니다. 예를 들어 `APPROVED -> CANCEL_PENDING -> CANCELLED`를 두고, 현재 상태와 version이 기대값일 때만 다음 상태로 바꾸는 conditional update를 사용합니다. PG 취소처럼 외부 side effect가 있으면 내부 트랜잭션만으로 원자성을 만들 수 없습니다. 그래서 취소 요청 idempotency key를 저장하고 `CANCEL_PENDING`을 먼저 커밋한 뒤 외부 PG를 호출합니다. 타임아웃으로 성공 여부가 불명확하면 같은 키로 재시도하거나 PG 조회 API로 결과를 확인합니다. 최종 webhook과 능동 조회 결과가 경쟁할 수 있으므로 동일한 상태 머신과 version 조건을 통과하게 합니다. 한도 복원은 최종 취소 확인과 연결하고 중복 복원이 불가능하도록 취소 transaction ID를 dedup key로 둡니다.

#### 결제 도메인의 핵심 불변식 예시

- 승인 합계는 고객 한도를 넘지 않는다.
- 한 결제의 누적 취소 금액은 승인 금액을 넘지 않는다.
- 동일 취소 요청은 한 번만 금액에 반영된다.
- 최종 상태가 된 거래가 이전 상태로 회귀하지 않는다. 단, 명시적 조정 이벤트는 별도 모델링한다.
- 금액은 정수 최소 화폐 단위와 currency로 표현한다. float를 쓰지 않는다.
- 외부 PG 원본 transaction ID는 provider namespace와 함께 unique해야 한다.

### 6. 이 주제의 압박 질문

#### `락을 잡은 Lambda가 죽으면요?`

> lease expiry로 회수하되, TTL 삭제 시각은 믿지 않습니다. 새 owner가 생긴 뒤 이전 실행이 돌아오는 상황은 owner token, version 조건 또는 fencing token으로 차단합니다.

#### `트랜잭션 응답 직전에 네트워크가 끊기면 성공인지 어떻게 아나요?`

> 동일한 business request ID와 `ClientRequestToken`으로 재시도하고, 결제 ID의 conditional put과 조회로 결과를 판별합니다. 새 ID를 발급해 다시 실행하면 중복 승인이 생길 수 있습니다.

#### `DynamoDB 대신 PostgreSQL이면 더 쉽지 않나요?`

> 관계와 ad hoc query가 핵심이고 단일 DB transaction으로 충분하다면 PostgreSQL이 더 단순할 수 있습니다. 당시에는 서버리스 scale, key-value access pattern, Streams 통합이 강점이었지만 트랜잭션과 락을 과도하게 조립하고 있었다면 RDB 선택을 재검토해야 합니다. 데이터베이스 이름보다 access pattern, transaction boundary, 운영 복잡도로 판단하겠습니다.

#### 공식 참고 자료

- [AWS DynamoDB transactions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)
- [AWS DynamoDB read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [AWS transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [AWS DynamoDB Lock Client](https://github.com/awslabs/amazon-dynamodb-lock-client)


---

## 02. 이벤트 순서, 멱등성, 재시도, DLQ

### 1. SQS FIFO가 보장하는 것과 보장하지 않는 것

#### 예상 질문

`SQS FIFO를 썼으니 메시지가 정확히 한 번, 순서대로 처리되나요?`

#### 1분 모범 답변

> 아닙니다. FIFO는 같은 Message Group ID 안에서 메시지 전달 순서를 관리하고 deduplication ID를 기준으로 제한된 시간 동안 중복 전송을 억제하지만, 애플리케이션의 side effect까지 exactly-once로 만들지는 않습니다. consumer가 처리한 뒤 delete 전에 죽으면 같은 메시지를 다시 받을 수 있습니다. 그래서 고객 또는 결제 단위로만 순서를 좁혀 Message Group ID를 정했고, 각 처리 단계는 event ID나 business key와 현재 상태를 이용해 멱등하게 만들었습니다. 서로 무관한 고객은 다른 group으로 병렬 처리했습니다.

#### 정확히 설명할 항목

- 순서의 범위는 queue 전체가 아니라 같은 `MessageGroupId`다.
- 한 group은 사실상 직렬 처리되므로 모든 메시지를 같은 group에 넣으면 처리량 병목이 된다.
- content-based deduplication은 message body를 기반으로 하고 message attributes는 포함하지 않는다.
- deduplication window에 의존해 영구 중복 방지를 구현하면 안 된다.
- visibility timeout은 처리 완료 시간이 아니며 연장이 없으면 실행 중 재전달될 수 있다.
- Lambda batch에서 일부만 실패했을 때 partial batch response가 없으면 성공한 record도 다시 처리될 수 있다.

### 2. 멱등 처리는 상태를 먼저 읽는 것으로 충분한가

#### 예상 질문

`같은 메시지가 오면 상태를 확인하고 건너뛰었다고 했는데 race condition은 없나요?`

#### 모범 답변

> 단순히 상태를 읽고 나중에 쓰는 것만으로는 충분하지 않습니다. 두 consumer가 동시에 `아직 처리 안 됨`을 읽을 수 있기 때문입니다. 내부 DB 변경은 event ID를 key로 한 idempotency record를 조건부 생성하거나, 상태 version을 조건부 갱신하는 방식으로 처리 권한을 원자적으로 획득해야 합니다. 처리 결과도 가능하면 같은 트랜잭션에 저장합니다. 외부 API 호출은 더 어렵습니다. provider가 idempotency key를 지원하면 안정적인 business key를 전달하고, 지원하지 않으면 호출 전 `IN_PROGRESS`를 기록한 뒤 timeout 시 무조건 재호출하지 않고 조회 API나 reconciliation으로 결과를 판별합니다.

#### 권장 상태 모델

| 상태 | 의미 | 다음 동작 |
|---|---|---|
| 없음 | 처음 본 event | conditional put으로 IN_PROGRESS 선점 |
| IN_PROGRESS, lease 유효 | 다른 실행이 처리 중 | retry 또는 visibility 연장 |
| IN_PROGRESS, lease 만료 | 이전 실행 중단 가능 | owner/version 조건으로 takeover |
| COMPLETED | side effect와 결과 확정 | 저장된 결과를 반환하거나 skip |
| FAILED_RETRYABLE | 일시 실패 | backoff 후 재시도 |
| FAILED_FINAL | 자동 재시도 불가 | DLQ 또는 운영 판단 |

#### 중요한 경계

idempotency record를 `COMPLETED`로 바꾸는 것과 외부 API 성공은 하나의 로컬 transaction으로 묶을 수 없다. 이 불확실성을 숨기지 않고 provider idempotency key, status query, webhook, reconciliation job을 조합한다.

### 3. 순서는 언제 필요한가

#### 예상 질문

`모든 결제 이벤트의 순서를 보장해야 하나요?`

#### 모범 답변

> 전역 순서는 비용이 크고 대부분 필요하지 않습니다. 먼저 어떤 aggregate에서 어떤 전이가 순서에 민감한지 봅니다. 같은 payment의 `PAID`와 `CANCELLED`, 같은 customer의 한도 변경은 순서가 중요할 수 있지만 서로 다른 고객의 알림은 독립적으로 처리할 수 있습니다. 저는 Message Group ID를 고객 또는 결제 단위로 선택했습니다. 다만 늦게 도착한 이벤트 자체는 FIFO만으로 해결되지 않기 때문에 aggregate version과 상태 전이 규칙을 확인해 오래된 이벤트가 최신 상태를 덮지 못하게 해야 합니다.

#### customer ID와 payment ID 중 무엇을 선택하나

| 기준 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| customer ID | 고객 한도와 여러 결제를 직렬화 | heavy customer가 hot group | 고객 단위 한도 불변식 |
| payment ID | 결제별 병렬성 높음 | 고객 공통 한도 경쟁은 별도 해결 | 결제 상태, 웹훅 처리 |
| partner ID | 파트너 rate limit 제어 가능 | 대형 파트너 병목 | provider별 호출 제한 |

하나의 queue에 모든 목적을 억지로 넣기보다 workload와 ordering key가 다르면 queue를 분리할 수 있다.

### 4. 재시도 정책

#### 예상 질문

`어떤 오류를 재시도했나요?`

#### 모범 답변

> timeout, connection reset, 429, 일부 5xx처럼 일시적일 가능성이 높은 오류만 재시도하고 validation error, 인증 실패, 허용되지 않은 상태 전이 같은 영구 오류는 즉시 실패로 분류합니다. exponential backoff와 jitter를 쓰고 최대 시도 횟수와 전체 시간 budget을 둡니다. 결제처럼 결과가 불명확한 timeout은 일반 오류와 다르게 취급해 provider 조회로 먼저 확인합니다. 재시도가 downstream 장애를 증폭하지 않도록 concurrency limit, rate limit, circuit breaker도 함께 봅니다.

#### 예시 정책

| 오류 | 정책 |
|---|---|
| connect timeout | 같은 idempotency key로 backoff retry |
| response timeout | 상태 조회 후 미처리 확인 시 retry |
| HTTP 429 | Retry-After 존중, jitter, 동시성 축소 |
| HTTP 500/502/503/504 | 제한된 retry, circuit breaker 고려 |
| HTTP 400 validation | retry 금지, 입력과 mapping 확인 |
| HTTP 401/403 | 자동 반복 금지, credential 또는 권한 점검 |
| business decline | 기술 retry 금지, 최종 비즈니스 결과로 저장 |

### 5. DLQ는 실패 메시지 창고가 아니다

#### 예상 질문

`DLQ에 간 메시지는 어떻게 운영했나요?`

#### 모범 답변

> DLQ는 유실 방지 수단일 뿐 복구 프로세스가 없으면 실패 메시지 창고가 됩니다. 원본 event ID, aggregate ID, 실패 단계, error category, 첫 실패와 마지막 실패 시각, attempt count를 확인할 수 있게 하고 DLQ depth와 oldest message age에 alarm을 둡니다. 재처리 전에는 코드나 데이터 원인이 해결됐는지 확인하고, 같은 idempotency key를 유지한 채 격리된 redrive 경로로 소량부터 재처리합니다. 결제 데이터는 무조건 재실행하지 않고 현재 내부 상태와 PG 상태를 먼저 대사합니다.

#### EventBridge target DLQ와 consumer DLQ의 차이

- target DLQ: EventBridge가 target queue나 Lambda에 전달하지 못한 실패다. 권한, KMS policy, target availability를 본다.
- consumer DLQ: queue에는 들어왔지만 handler가 반복 실패한 경우다. 코드, payload, downstream, business validation을 본다.
- 둘을 구분해야 장애 위치와 재처리 방법이 명확하다.

### 6. 외부 기관 점검 시간을 상태 기반으로 복구한 방식

#### 예상 질문

`READY, PROGRESS, SUCCEED, FAILED와 next execution time 설계를 설명해 주세요.`

#### 모범 답변

> 외부 신용평가는 수분 이상 걸리고 정부24나 홈택스 정기 점검도 있어 HTTP 요청 하나에 묶을 수 없었습니다. 작업 상태와 다음 실행 시각을 영속화하고, scheduler가 실행 가능한 작업만 조회해 조건부 쓰기로 선점한 뒤 queue에 넣었습니다. 점검 오류는 일반 exponential retry로 소모하지 않고 알려진 점검 종료 이후로 next execution time을 옮겼습니다. worker가 죽어도 lease나 progress timeout이 지나면 다시 선점할 수 있게 했고, 이미 성공한 단계는 재호출하지 않도록 상태를 확인했습니다.

#### 베스트 프랙티스로 보완할 점

- `PROGRESS`보다 `IN_PROGRESS`처럼 용어를 명확히 한다.
- 기술 실패와 비즈니스 실패, 재시도 가능 여부를 한 `FAILED`에 섞지 않는다.
- state, attempt, next_run_at, owner, lease_until, last_error_category, provider_reference를 둔다.
- GSI를 `status + time bucket` 식으로 설계해 scan을 피하고 hot partition을 점검한다.
- scheduler 중복 실행을 정상으로 보고 conditional claim을 사용한다.
- maximum elapsed time과 수동 검토 상태를 둔다.
- 상태 전이와 event 저장을 transaction으로 묶는다.

### 7. 이벤트 스키마와 배포

#### 예상 질문

`producer와 consumer를 독립 배포할 때 이벤트 스키마를 어떻게 바꾸나요?`

#### 모범 답변

> event envelope에 event type과 schema version을 명시하고, 기본은 optional field를 추가하는 backward-compatible 변경으로 합니다. consumer가 새 필드를 모르는 동안에도 무시할 수 있어야 합니다. field 삭제나 의미 변경은 새 event type 또는 version을 만들고 양쪽이 공존하는 전환 기간을 둡니다. 계약 테스트와 실제 과거 event fixture replay로 호환성을 확인합니다. 민감정보는 편의를 위해 event 전체에 복제하지 않고 최소화합니다.

### 8. 관측해야 할 메트릭

- queue age, depth, in-flight count
- Lambda error, throttle, duration, iterator age
- retry count by error category
- idempotency hit, conflict, stale takeover count
- DLQ ingress와 oldest age
- 처리 end-to-end latency by event type
- provider latency, timeout, 429, 5xx, decline 비율
- 상태별 task 체류 시간
- reconciliation mismatch count

#### 공식 참고 자료

- [AWS SQS FIFO delivery logic](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-understanding-logic.html)
- [AWS SQS exactly-once processing](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html)
- [AWS Lambda with SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [AWS Lambda partial batch responses](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html)


---

## 03. 보안 심층 면접 대비

### 0. 가장 먼저 말할 원칙

> 보안 기능의 이름을 나열하기보다 위협 모델, 신뢰 경계, 보호할 데이터, 키와 자격증명의 수명, 실패 시 동작을 먼저 설명하겠습니다. 당시 외부 기관이 요구한 프로토콜을 구현한 부분과 제가 새로 선택한 설계를 구분해서 말씀드리겠습니다.

이 문장이 중요한 이유는 첨부 보안 문서에 강한 표현이 일부 있기 때문이다. 아래 모범 답변은 그 표현을 정확하게 교정한다.

### 1. 전체 신뢰 경계

#### 예상 질문

`보안 아키텍처를 2분 안에 설명해 주세요.`

#### 모범 답변

> 외부 클라이언트, 공개 진입점, 내부 서비스, 데이터 저장소, KCB와 KCS 같은 외부 기관을 서로 다른 신뢰 경계로 봤습니다. 공개 요청은 JWT를 검증하고 토큰의 customer claim으로 대상 리소스를 결정해 client가 보낸 customer ID를 신뢰하지 않았습니다. 내부 서비스 호출은 당시 공유 API key로 인증했고, 민감정보와 외부 기관 secret은 Secrets Manager와 KMS 기반 저장 암호화를 사용했습니다. KCB callback은 HMAC으로 파라미터 무결성을 검증하고, 외부 규격에 따라 암호화된 본인인증 결과를 복호화했습니다. 로그에는 주민번호, CI, token, secret이 남지 않게 마스킹했습니다. 다만 공유 API key는 mTLS 수준의 상호 인증은 아니고, CORS도 인가 경계가 아닙니다. 현재 설계라면 workload identity, 짧은 수명의 자격증명, callback replay 방지, 민감정보 allowlist logging까지 보완하겠습니다.

### 2. JWT와 IDOR

#### 예상 질문

`body의 customerId를 JWT cid로 덮어쓰면 IDOR이 완전히 해결되나요?`

#### 모범 답변

> 중요한 방어지만 그것만으로 완전하다고 말하지는 않겠습니다. 먼저 JWT의 signature와 허용 algorithm, issuer, audience, expiration, not-before를 검증하고 key rotation을 고려해야 합니다. 그 뒤 subject와 customer claim의 관계를 서버가 신뢰할 수 있어야 합니다. endpoint는 client가 보낸 owner ID를 무시하고 검증된 principal에서 tenant scope를 만들며, repository query에도 같은 scope를 강제해야 합니다. 하위 리소스 ID만 받는 endpoint는 그 리소스가 해당 customer 소유인지 다시 확인합니다. 관리자나 파트너 권한은 별도의 role과 permission으로 분리하고 audit log를 남깁니다.

#### 베스트 프랙티스 체크리스트

- `alg` allowlist를 고정하고 token header가 알고리즘을 선택하게 두지 않는다.
- `iss`, `aud`, `exp`, `nbf`를 검증한다.
- key ID와 rotation, 폐기 시나리오를 둔다.
- customer ID를 request에서 덮는 것에 그치지 않고 data access layer의 tenant scope를 강제한다.
- opaque resource ID라도 authorization check를 생략하지 않는다.
- 401은 인증 실패, 403은 인증됐지만 권한 부족으로 구분하되 과도한 존재 정보는 노출하지 않는다.
- raw token과 claims 전체를 로그에 남기지 않는다.

#### 꼬리 질문

`JWT audience를 Origin으로 검증하는 게 맞나요?`

> 일반적으로 audience는 token이 의도된 수신 서비스나 API를 식별하고, browser Origin과 동일한 개념은 아닙니다. Origin은 CORS 입력이고 spoof 가능한 non-browser client도 있습니다. 실제 구현이 Origin 값을 aud로 사용했다면 당시 token 발급 계약을 먼저 확인해야 하고, 일반적인 베스트 프랙티스로는 issuer가 발급한 고정 API audience를 검증하고 Origin allowlist는 별도로 처리하겠습니다.

### 3. CORS의 정확한 의미

#### 예상 질문

`CORS 화이트리스트가 어떤 공격을 막나요?`

#### 모범 답변

> CORS는 브라우저가 다른 origin의 response를 script에서 읽는 것을 제한하는 정책입니다. curl이나 서버 간 요청을 막지 않으므로 인증이나 네트워크 접근 제어가 아닙니다. credential을 쓰는 경우 정확한 origin allowlist와 `Vary: Origin`을 설정하고 필요한 method와 header만 허용합니다. CSRF는 cookie 인증을 쓴다면 SameSite, CSRF token, Origin 검증 등으로 별도 방어합니다.

### 4. 내부 API key

#### 예상 질문

`x-pymt-api-key는 충분히 안전한 서비스 간 인증인가요?`

#### 모범 답변

> 당시에는 외부에 노출되지 않은 내부 endpoint에서 공유 secret을 검증해 무인증 호출을 막는 실용적인 통제였습니다. 하지만 장기 수명의 공용 key는 한 서비스가 침해되면 다른 서비스로 가장할 수 있고 호출 주체 구분과 rotation이 어렵습니다. 그래서 상호 인증이라고 부르지 않겠습니다. 개선한다면 서비스별 IAM role과 SigV4, private API resource policy, mTLS 또는 service mesh workload identity 중 환경에 맞는 방식을 사용하고, 서비스별 최소 권한과 짧은 수명, rotation, auditability를 확보하겠습니다.

#### 헤더 strip에 대한 꼬리 질문

> hop-by-hop header와 infrastructure header를 제거하는 것은 좋지만, 임의의 forwarded header를 신뢰하면 spoofing이 생깁니다. 허용할 header를 allowlist로 새로 구성하는 편이 denylist보다 안전합니다. 내부 인증 header도 외부 요청에서 받은 값을 그대로 전달하지 않고 gateway가 제거한 뒤 새로 주입해야 합니다.

### 5. HMAC callback 서명

#### 예상 질문

`HMAC으로 무엇을 보장했고 무엇은 못 보장하나요?`

#### 모범 답변

> 공유 secret을 아는 당사와 외부 기관 사이에서 callback parameter가 변조되지 않았고 예상한 발신 규격에서 왔다는 무결성과 인증을 제공합니다. 암호화는 아니어서 payload를 숨기지 않고, 서명만으로 freshness나 replay는 막지 못합니다. 검증할 때는 수신한 raw bytes 또는 명확히 canonicalize한 representation을 사용하고 constant-time comparison을 합니다. timestamp 허용 범위, nonce나 transaction ID의 1회성 소비, 완료 상태의 conditional transition을 함께 적용해 replay를 막습니다.

#### URL과 JSON canonicalization

- URL은 query parameter 순서, percent encoding, 공백, duplicate key 처리 규칙이 양쪽에서 같아야 한다.
- JSON을 임의로 parse 후 dump하면 key order, 숫자, Unicode 표현이 바뀔 수 있다.
- 가장 안전한 방식은 provider가 정의한 raw request body와 지정 header를 정확히 연결해 서명하는 것이다.
- 자체 webhook 발신 시 문서화된 canonical format 또는 raw bytes 서명을 제공하고 version을 붙인다.
- signature header에 key ID와 timestamp를 포함하면 rotation과 replay 방어가 쉬워진다.

#### `같은 payload면 같은 서명`에 대한 정정

> 결정적 직렬화는 수신자 검증을 안정적으로 만들지만 replay를 쉽게 식별해 주지는 않습니다. event ID, timestamp, nonce와 delivery attempt를 포함하고 receiver가 event ID를 멱등 처리해야 합니다.

### 6. AES-CBC와 외부 기관 프로토콜

#### 예상 질문

`AES-CBC 구현을 설명하고 보안상 주의점을 말해 주세요.`

#### 모범 답변

> KCB와 KCS가 정한 연동 규격에 맞춰 AES-CBC와 PKCS#7 padding을 사용했습니다. CBC는 기밀성만 제공하고 자체 무결성은 없기 때문에 일반적인 신규 설계라면 AES-GCM 같은 AEAD를 우선합니다. 부득이하게 CBC 규격을 따라야 하면 무작위이며 예측 불가능하고 재사용되지 않는 IV가 필요하고, ciphertext와 IV 전체에 별도 MAC을 적용하는 encrypt-then-MAC 구조가 필요합니다. 복호화 오류도 padding 여부를 외부에 구분해 노출하지 않아야 합니다. KCS 규격처럼 key material의 일부를 고정 IV로 쓰는 구조였다면 제가 선택한 베스트 프랙티스가 아니라 provider compatibility 요구였다고 구분해서 설명하겠습니다.

#### 왜 E2EE라고 부르면 위험한가

자사 backend가 주민번호 평문을 받고 암호화한다면 자사 backend는 한 endpoint다. 이는 TLS 위에 적용한 application-layer payload encryption이지, client에서 최종 KCS만 복호화할 수 있게 암호화한 진정한 end-to-end encryption이라고 단정하기 어렵다.

#### 2단계 KCB 복호화 설명

> 본인인증 시작 시 transaction별 IV와 provider token, encrypted key material을 임시 저장했습니다. 완료 후 규격상 첫 단계에서 data encryption key를 얻고, 두 번째 단계에서 CI와 사용자 정보를 복호화했습니다. 복호화 후에는 필요한 최소 데이터만 남기고 임시 key material을 삭제했습니다. 여기서 중요한 것은 algorithm 이름보다 transaction binding, key 수명, 실패 시 삭제, 로그 비노출, 재처리 시 one-time token의 의미입니다.

### 7. CI 해시와 개인정보

#### 예상 질문

`CI + salt를 SHA-256한 값은 안전한 익명정보인가요?`

#### 모범 답변

> 익명정보라기보다 결정적 가명 식별자입니다. 같은 CI를 같은 값으로 연결해야 해서 linkability가 남고, secret salt와 원본 시스템이 있으면 다시 개인과 연결될 수 있습니다. 따라서 여전히 개인정보로 취급하고 접근 통제, 보존 기간, 삭제, audit을 적용해야 합니다. 여러 환경이나 파트너 사이의 상관관계를 줄이려면 목적별 secret key를 사용한 HMAC 기반 pseudonymous ID도 고려할 수 있습니다. raw CI가 실제로 저장되지 않았다는 점과, 어떤 업무에서 복호화 또는 조회가 가능한지는 별도로 설명해야 합니다.

#### Salt와 pepper의 구분

- user별 공개 salt는 password hash의 rainbow table을 어렵게 한다.
- 시스템 공통 secret을 붙였다면 성격상 pepper에 가깝고 Secrets Manager 보호와 rotation 문제가 생긴다.
- deterministic ID의 key를 rotation하면 기존 lookup이 깨질 수 있어 versioned key와 migration 계획이 필요하다.

### 8. ULID

#### 예상 질문

`ULID가 보안 식별자로 좋은 이유는 무엇인가요?`

#### 모범 답변

> 정렬 가능성과 분산 생성 편의가 주목적입니다. random component 때문에 단순 순차 정수보다 추측은 어렵지만 timestamp가 포함돼 생성 시점이 드러나고, 식별자의 불투명성이 authorization을 대신하지는 않습니다. 모든 object access는 소유권 검증을 해야 합니다. 높은 쓰기량에서 시간순 key가 partition hot spot을 만드는지도 저장소별로 확인해야 합니다.

### 9. Secrets Manager와 KMS

#### 예상 질문

`KMS로 암호화했다고 했는데 실제로 어떤 방식인가요?`

#### 모범 답변

> 먼저 서비스 관리형 저장 암호화와 애플리케이션 필드 암호화를 구분하겠습니다. DynamoDB의 AWS managed key 암호화는 disk와 backup의 at-rest protection을 제공하지만, 해당 table을 읽을 IAM 권한이 있는 application에는 평문이 반환됩니다. 더 강한 분리가 필요하면 customer managed key 정책과 application-level envelope encryption을 고려합니다. 외부 API secret은 Secrets Manager에 두고 Lambda execution role별로 필요한 secret ARN과 KMS decrypt 권한만 허용합니다. secret 값은 invocation마다 원격 조회하지 않고 짧게 cache하되 rotation과 invalidation을 고려하고 절대 로그에 남기지 않습니다.

#### Rotation 깊이 질문

- DB password rotation은 RDS Proxy와 client connection behavior까지 검증해야 한다.
- webhook HMAC key는 발신자와 수신자가 동시에 바뀌지 않으므로 old/new key 동시 검증 기간과 key ID가 필요하다.
- CI deterministic key는 단순 rotation이 ID 변경을 일으킨다.
- compromised key rotation과 정기 rotation은 절차가 다르다.
- secret을 환경 변수로 주입하지 않았다는 표현은 실제 runtime load 방식을 확인한 뒤 말한다.

### 10. 로깅과 마스킹

#### 예상 질문

`DEBUG=false일 때 body를 마스킹하면 충분한가요?`

#### 모범 답변

> 운영 플래그에 의존한 전체 body logging은 설정 실수 한 번으로 유출될 수 있습니다. 기본은 request body와 authorization header를 기록하지 않고, 필요한 field만 allowlist로 구조화해 남기는 방식이 안전합니다. customer ID도 가능하면 내부 correlation ID로 대체하고, error object나 third-party response가 PII를 포함하는지 확인합니다. 로그 접근 권한, 보존 기간, encryption, export 경로와 감사 기록도 함께 관리해야 합니다.

#### 관측성과 개인정보의 균형

남겨야 할 것:

- request ID, event ID, transaction ID
- provider와 operation 이름
- normalized error category와 provider error code
- latency, attempt, 상태 전이
- 개인정보가 아닌 tenant 또는 customer pseudonymous reference

남기지 않을 것:

- 주민번호, raw CI/DI, 전화번호, 이름
- access token, refresh token, API key, HMAC secret
- 전체 request와 response body
- 암호화 key material과 복호화 중간값

### 11. 보안 문서에서 검증이 필요한 주장

| 문서 표현 | 면접 답변 |
|---|---|
| 전 구간 TLS 1.3 강제 | 실제 API Gateway/ALB security policy 확인 전 단정 금지. 최소 TLS version과 cipher policy로 설명 |
| 내부 서비스 VPC 완전 은닉 | Lambda의 VPC 연결, API endpoint type, resource policy와 route를 실제 IaC에서 확인 |
| HMAC이 CSRF와 replay 차단 | 무결성 외에 state/nonce/expiry/one-time consume이 있었는지 확인 |
| 임시 transaction 즉시 영구 삭제 | DynamoDB delete 후 backup/PITR와 로그 사본까지 포함한 법적 삭제 의미는 아님 |
| PBKDF2 100,000회가 현재 OWASP 권고 충족 | 현재 권고와 비용은 바뀔 수 있음. 측정 후 상향 또는 Argon2id 검토 |
| AWS managed KMS | AWS owned key, AWS managed key, customer managed key를 정확히 구분 |
| RDS Proxy가 DB 직접 연결 차단 | network SG와 IAM 정책이 차단한다. Proxy 사용 자체가 직접 연결을 자동 차단하지 않음 |

### 12. 보안 사고 시나리오

#### `웹훅 secret이 유출됐다면?`

> key ID로 영향 범위를 식별하고 해당 key를 폐기 또는 rotate합니다. old/new 병행 기간을 최소화하고, 유출 구간의 callback을 event ID와 provider 조회로 재검증합니다. secret access audit와 로그 유출 여부를 조사하고, replay된 상태 전이가 version 조건과 idempotency에서 차단됐는지 확인합니다.

#### `주민번호가 로그에 남았다면?`

> 추가 기록을 즉시 막고 접근 범위를 제한한 뒤, 어느 log group과 export sink, backup에 복제됐는지 확인합니다. 보존 정책에 따라 삭제하고 접근 기록과 영향 대상, 노출 기간을 파악합니다. 법무와 개인정보 담당 절차에 따라 신고와 통지를 판단합니다. 이후 schema-aware redaction, logging allowlist, CI test로 재발을 막습니다.

#### 공식 참고 자료

- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [AWS Secrets Manager rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [AWS DynamoDB encryption at rest](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/EncryptionAtRest.html)


---

## 04. 데이터 모델, API 추상화, Lambda, 관측성

### 1. DynamoDB와 PostgreSQL을 왜 함께 썼나

#### 예상 질문

`결제는 DynamoDB, 정산은 PostgreSQL로 나눈 이유가 무엇인가요?`

#### 모범 답변

> 결제와 고객의 온라인 상태 변경은 key 기반 접근, 조건부 쓰기, 트랜잭션, Streams 연동이 중요해 DynamoDB를 사용했습니다. 반면 정산과 운영 리포트는 기간, 파트너, 상태, 금액을 조합한 조회와 집계가 많아 PostgreSQL이 적합했습니다. 중요한 점은 두 DB를 동시에 source of truth로 두지 않는 것입니다. 결제 상태는 DynamoDB를 기준으로 하고 Streams event로 PostgreSQL projection을 갱신했습니다. projection은 지연되거나 누락될 수 있으므로 event ID와 aggregate version으로 중복과 역전을 막고, lag metric과 정기 reconciliation으로 원본과 대사해야 합니다.

#### 꼬리 질문

`Streams event가 누락되거나 순서가 바뀌면요?`

> consumer checkpoint와 DLQ만으로 끝내지 않고 iterator age, failed batch, projection lag를 감시합니다. 각 projection row에 마지막 적용 version을 저장해 오래된 event를 무시하고, 정기적으로 source와 projection의 count와 금액 checksum을 비교합니다. 차이가 나면 원본에서 해당 기간 또는 aggregate를 다시 build할 수 있어야 합니다.

#### 베스트 프랙티스

- 하나의 데이터에 대한 authoritative source를 명시한다.
- projection은 rebuild 가능하게 만든다.
- event ID unique constraint와 aggregate version 조건을 둔다.
- 금액과 건수 reconciliation을 자동화한다.
- 정산 확정 시점에는 projection lag를 고려한 cutoff와 close 절차를 둔다.
- 개인정보 삭제와 retention이 두 저장소에 모두 적용되게 한다.

### 2. 상환 알림 데이터 모델

#### 예상 질문

`sending_date를 partition key로 쓰면 hot partition 문제가 없나요?`

#### 모범 답변

> 해당 날짜 대상만 읽는 access pattern에는 맞지만 모든 고객이 같은 날짜 key에 몰리면 write와 read가 한 partition key에 집중될 수 있습니다. 실제 규모에서 throttling이 없었는지 metric으로 확인해야 합니다. 규모가 커지면 `sending_date#shard`로 write sharding하고 당일 모든 shard를 병렬 조회하거나, 날짜 bucket과 partner를 함께 key로 구성할 수 있습니다. shard 수는 peak 건수와 처리량을 기반으로 정하고, 무작정 늘리면 query fan-out 비용이 커집니다.

#### 합산과 취소 불변식

> 고객과 발송일 단위의 예정 금액을 transaction으로 갱신하되, 각 payment event ID를 중복 반영하지 않도록 해야 합니다. 단순히 합계만 저장하면 재처리 시 중복 합산될 수 있으므로 event ledger나 applied-event record를 같이 저장하는 방법이 있습니다. 합계가 0이면 soft delete를 했다고 되어 있는데, 이후 재승인이나 복구가 가능한 상태 전이라면 tombstone과 version을 유지해 오래된 event가 되살리지 못하게 해야 합니다. 발송 직전에는 최신 CMS 동의와 실제 미납 금액을 다시 확인해 projection 지연으로 잘못 발송하는 것을 막습니다.

### 3. 정산 모델

#### 예상 질문

`결제와 정산을 어떻게 분리해서 모델링합니까?`

#### 모범 답변

> 결제는 고객이 지불했는지에 대한 거래 상태이고, 정산은 그 거래를 기준으로 누구에게 얼마를 언제 지급할지 계산하고 확정하는 별도 도메인입니다. 하나의 payment가 부분 취소, 수수료, 세금, 프로모션, 여러 하위 상점 때문에 여러 settlement line으로 나뉠 수 있습니다. 계산 결과는 versioned snapshot 또는 immutable ledger entry로 남기고, 확정 후 변경은 기존 row 수정이 아니라 adjustment entry로 추적하는 편이 감사에 유리합니다. PG 원본 금액, 포트원 normalized 값, 고객사 주문 값 사이 대사를 별도로 둡니다.

#### 핵심 테이블 또는 aggregate 개념

- payment, payment_attempt, provider_transaction
- cancellation 또는 refund
- settlement_batch, settlement_line
- fee, tax, promotion allocation
- payout와 payout failure
- adjustment와 reconciliation_result
- raw_provider_event와 normalized_event

### 4. QueryCriteria 추상화

#### 예상 질문

`SQLAlchemy QueryCriteria는 어떤 문제를 어떻게 해결했나요?`

#### 모범 답변

> 운영 API마다 기간, 상태, 파트너, 정렬, pagination 조건을 반복 구현하면서 validation과 join 방식이 달라지는 문제가 있었습니다. 허용된 field와 operator를 typed criteria로 제한하고, domain filter를 SQLAlchemy expression으로 변환하는 공통 계층을 만들었습니다. endpoint는 filter 조합만 선언하고 tenant scope와 기본 정렬, pagination은 공통 적용했습니다. 그래서 대표적인 조회 API 개발 시간을 줄였습니다. 다만 범용 query language로 키우면 임의 join, 느린 sort, SQL injection, count query 폭증 위험이 있어 field와 operator를 allowlist하고 query plan과 index를 access pattern별로 검증했습니다.

#### 구조 예시

```text
HTTP query params
  -> validation DTO
  -> typed QueryCriteria
  -> repository-specific compiler
  -> SQLAlchemy expression
  -> stable sort + cursor
```

#### 꼬리 질문

`offset pagination과 cursor pagination 중 무엇을 썼나요?`

> 실제 구현을 기준으로 답해야 합니다. 베스트 프랙티스로는 깊은 페이지와 변경이 잦은 결제 목록에는 stable unique ordering을 가진 keyset pagination을 선호합니다. 예를 들어 `created_at DESC, id DESC`를 cursor에 넣습니다. offset은 단순하지만 깊어질수록 비용이 늘고 중간 insert 때문에 중복이나 누락이 생길 수 있습니다.

`공통화가 잘못된 사례는요?`

> filter DSL이 domain의 특수한 의미를 숨기거나, 모든 관계를 dynamic join으로 풀게 되면 성능 예측이 어렵습니다. 공통화는 validation과 반복되는 operator까지만 두고, 복잡한 정산 query는 명시적인 repository method나 read model로 빼겠습니다.

### 5. 파트너별 정책과 공통 규칙

#### 예상 질문

`파트너별 if문을 어떻게 줄였나요?`

#### 모범 답변

> 먼저 모든 파트너에 동일한 domain invariant와 실제로 달라지는 policy를 분리했습니다. 예를 들어 승인 금액이 가용 한도를 넘지 않는 것은 공통 규칙이고, 한도 계산 방식, 계약 필요 여부, 정산 주기는 파트너 policy가 될 수 있습니다. use case는 공통 흐름을 소유하고 policy interface를 호출하며, partner ID에 따른 구현 선택은 composition root에서 합니다. 단, 차이가 단순 parameter라면 별도 class를 늘리지 않고 configuration이나 decision table로 표현합니다. 규칙 변경 이력을 남기고 당시 payment가 어느 policy version으로 계산됐는지도 보존하는 게 중요합니다.

#### 위험 신호

- strategy class가 사실상 if문을 파일로 옮긴 것뿐이다.
- partner마다 aggregate와 상태 이름이 달라 공통 domain model이 무너진다.
- 설정 변경이 과거 거래 계산을 바꾼다.
- 운영자가 규칙을 바꿨는데 audit과 approval이 없다.
- partner custom field가 core table에 무한히 늘어난다.

### 6. DDD와 Hexagonal Architecture

#### 예상 질문

`DDD를 실제로 어디에 적용했나요?`

#### 모범 답변

> 폴더 구조보다 결제, 한도, 계약, 정산의 용어와 불변식을 코드 모델에 반영한 것이 핵심이었습니다. use case가 HTTP나 DynamoDB SDK에 직접 의존하지 않도록 repository와 external provider port를 두고 adapter에서 Chalice, PynamoDB, KCB/KCS API를 연결했습니다. 그 덕분에 외부 기관 response를 domain result로 변환하고 retry나 mapping을 adapter 경계에 둘 수 있었습니다. 다만 10개 이상 작은 서비스에 동일한 계층을 반복하면 ceremony가 커졌고, 팀 규모와 변경 빈도에 비해 MSA가 과한 부분도 있었습니다. 지금은 bounded context와 독립 배포 필요가 명확한 곳에만 경계를 두겠습니다.

#### `MSA 15개가 정말 필요했나요?`

> 결과적으로 작은 팀에는 배포, 로컬 재현, observability, event contract 관리 비용이 컸습니다. 외부 장애 격리와 scaling 단위 분리에는 장점이 있었지만, 서비스 수 자체를 성과로 보지는 않습니다. 처음부터 다시 한다면 modular monolith 또는 더 큰 서비스 단위로 시작하고 팀 소유권, 독립 scale, 보안 경계, 변경 주기가 실제로 달라질 때 분리하겠습니다.

### 7. Lambda cold start

#### 예상 질문

`P99를 2~3초에서 200ms대로 줄였다는 것을 설명해 주세요.`

#### 모범 답변

> 먼저 CloudWatch에서 어떤 함수와 기간, memory, runtime 기준인지 분리해 cold invocation의 init duration과 전체 duration을 봤습니다. Python import graph에서 사용하지 않는 무거운 module을 handler import 시점에 모두 불러오던 부분을 필요한 경로에서만 lazy load하도록 바꿨고, 사용자 traffic이 있는 시간대에는 scheduler warm-up도 적용했습니다. 관측한 구간에서 P99가 2~3초에서 200ms대로 내려갔습니다. 다만 warmer는 scale-out이나 AWS의 environment recycle 때 보장되지 않고 비용과 noise가 있습니다. 엄격한 latency SLO라면 Provisioned Concurrency, package 축소, memory 조정, runtime 개선을 비용과 함께 비교하겠습니다.

#### 반드시 준비할 측정 상세

- 전체 API latency인지 Lambda duration인지
- cold sample을 어떻게 식별했는지
- P99 집계 window와 invocation 수
- warm-up 호출을 percentile에서 제외했는지
- memory size와 deployment package 변화
- lazy import가 첫 해당 기능 호출로 지연을 옮긴 것은 아닌지
- 오류율과 비용 변화

### 8. 관측성

#### 예상 질문

`이벤트 ID 하나만 전달하면 distributed tracing이 되나요?`

#### 모범 답변

> correlation에는 도움이 되지만 완전한 distributed tracing은 아닙니다. trace ID와 span context를 표준 header와 message attribute로 전파하고, 각 hop에서 span과 구조화 로그에 payment ID, event ID를 연결해야 합니다. 비동기 queue에서는 producer trace와 consumer processing 사이를 link로 표현할지 child span으로 표현할지 도구 특성을 봅니다. 중요한 것은 한 요청을 찾는 것뿐 아니라 queue 대기 시간과 provider 호출 시간, retry를 분리해 병목 위치를 알 수 있게 하는 것입니다.

#### Golden signals와 business metrics

- latency: API, queue wait, handler, provider별
- traffic: 결제 요청, 승인, 취소, webhook량
- errors: 기술 오류와 business decline 분리
- saturation: Lambda concurrency, queue age, DB connection, throttling
- business: 승인율, 상태별 체류, 취소율, 대사 불일치, 정산 지연

#### 공식 참고 자료

- [AWS Lambda cold starts](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)
- [AWS Lambda Provisioned Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)
- [PostgreSQL multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
- [OpenTelemetry traces](https://opentelemetry.io/docs/concepts/signals/traces/)


---

## 05. 포트원 결제 시스템과 PG 추상화

### 1. 포트원의 문제를 한 문장으로 정의하기

> 고객사에는 일관된 결제 API와 운영 경험을 제공하되, 내부에서는 PG사와 결제수단마다 다른 상태, 파라미터, 비동기 결과, 오류, 취소 정책을 정보 손실 없이 다루는 문제입니다.

공통 모델을 깔끔하게 만드는 것만큼 원본 provider 정보와 예외를 보존하는 것이 중요하다.

### 2. 결제 플로우 설계 질문

#### 예상 질문

`결제 완료 API를 설계해 보세요.`

#### 모범 답변

> 브라우저가 보낸 성공 여부와 금액을 신뢰하지 않습니다. 고객사 서버가 payment ID를 받고 포트원 서버 API로 결제 상태를 조회한 뒤, 내부 주문의 order ID, expected amount, currency, store와 비교합니다. 검증이 끝난 뒤 주문 상태를 조건부로 갱신합니다. client callback이 유실될 수 있으므로 검증된 webhook도 같은 상태 머신으로 들어오게 합니다. callback과 webhook이 동시에 와도 payment ID와 event ID를 기준으로 멱등 처리합니다. 가상계좌 발급은 결제 완료가 아니므로 `VIRTUAL_ACCOUNT_ISSUED`와 `PAID`를 분리합니다.

이는 포트원 공식 V2 가이드의 서버 측 결제 조회와 금액 검증 원칙에 맞는다.

#### 상태 예시

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

### 3. PG Adapter 추상화

#### 예상 질문

`여러 PG사를 어떻게 추상화하겠습니까?`

#### 모범 답변

> core domain에는 `authorize`, `capture`, `cancel`, `query`, `issueBillingKey` 같은 capability를 정의하고 provider adapter가 각 PG request와 response를 변환하게 합니다. 하지만 모든 PG가 같은 기능을 지원한다고 가장하면 안 됩니다. channel별 capability matrix를 별도로 두고, 지원하지 않는 기능은 명시적인 domain error로 반환합니다. normalized status와 error category를 제공하면서도 원본 provider code, message, transaction ID, raw event reference를 보존해 고객 지원과 장애 분석에 사용합니다. provider별 특수 옵션은 core request를 오염시키지 않도록 namespaced extension 또는 typed provider option으로 격리합니다.

#### 계층 예시

| 계층 | 책임 |
|---|---|
| Public API | 일관된 request, validation, idempotency contract |
| Payment orchestration | 상태 머신, invariant, routing, retry decision |
| Provider capability | channel과 payment method별 지원 기능 |
| PG adapter | provider protocol, auth, mapping, timeout |
| Raw event store | 원본 webhook과 request reference 보존 |
| Normalized model | 공통 payment, transaction, cancellation 모델 |

#### 추상화에서 버리면 안 되는 것

- provider transaction ID와 channel
- original status와 error code
- sync response인지 webhook인지 polling인지
- 금액, currency, tax-free amount
- 부분 취소 history와 remaining cancellable amount
- provider event occurred time과 received time
- test/live mode
- provider-specific receipt와 settlement reference

### 4. 오류 모델

#### 예상 질문

`PG 오류 코드를 어떻게 공통화하나요?`

#### 모범 답변

> 고객이 행동을 바꿀 수 있는 공통 category와 운영 분석용 원본 code를 함께 제공합니다. 예를 들어 `INVALID_REQUEST`, `AUTHENTICATION_FAILED`, `PAYMENT_DECLINED`, `RATE_LIMITED`, `PROVIDER_UNAVAILABLE`, `UNKNOWN_RESULT`로 normalize하되 `pgCode`와 `pgMessage`는 별도 필드로 보존합니다. retry 가능 여부를 HTTP status 하나로 정하지 않고 operation과 결과 확실성까지 포함합니다. 특히 timeout은 실패가 아니라 결과 불명일 수 있어 query 또는 reconciliation으로 확인합니다.

#### 절대 하면 안 되는 것

- 모든 provider 오류를 500으로 변환
- 모든 5xx를 동일 request ID 없이 자동 재시도
- decline을 장애로 집계
- 내부 secret이나 provider raw payload를 고객에게 그대로 노출
- 새로운 provider code를 enum parse 실패로 버림

### 5. Webhook 수신과 발신

#### 예상 질문

`포트원 웹훅을 고객사가 안전하게 처리하도록 어떻게 설계할까요?`

#### 모범 답변

> endpoint는 빠르게 signature와 timestamp를 검증하고 raw event를 durable하게 저장한 뒤 2xx를 반환하며 실제 처리는 비동기로 넘깁니다. Standard Webhooks 규격의 message ID와 timestamp, signature를 검증하고 replay window와 event ID dedup을 적용합니다. 고객사가 signature 검증을 쓰지 않는 경우에는 webhook body 자체를 신뢰하지 않고 포트원 API로 payment를 재조회할 수 있어야 합니다. 동일 event 재전송과 서로 다른 event의 상태 역전을 모두 고려합니다.

#### 포트원 공식 문서와 연결되는 포인트

- V2 웹훅은 공개 endpoint이므로 본문을 그대로 신뢰하면 안 된다.
- Standard Webhooks 기반 signature 검증 또는 API 재조회 전략이 있다.
- `Payment.Paid`, `Payment.Cancelled`, `BillingKey.Issued`처럼 event type이 나뉜다.
- 가상계좌 입금과 비동기 취소 승인처럼 결과가 나중에 확정되는 trigger가 있다.

#### 웹훅 발송 시스템 설계

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

### 6. 부분 취소

#### 예상 질문

`부분 취소 API의 동시성을 어떻게 처리합니까?`

#### 모범 답변

> `sum(successful cancellation amounts) <= paid amount`가 불변식입니다. cancel request마다 idempotency key를 받고 payment의 version과 remaining cancellable amount를 조건부로 예약합니다. provider 호출 전에 내부 상태를 `CANCEL_REQUESTED`로 저장하고, 성공하면 cancellation transaction을 확정합니다. 두 요청이 동시에 같은 잔액을 취소하지 못하게 version 또는 reservation을 사용합니다. timeout은 provider transaction 조회로 확정하고, webhook이 먼저 와도 같은 cancellation ID와 상태 전이를 사용합니다.

#### 가상계좌 환불의 차이

포트원 공식 문서처럼 가상계좌는 환불 수령 계좌가 추가로 필요할 수 있고 PG별 특약, 수수료, 처리 시간이 다르다. 공통 cancel API에 무리하게 숨기기보다 payment method capability와 conditional required fields로 명확하게 표현한다.

### 7. 직접 PG에서 취소했을 때

#### 예상 질문

`고객이 PG 콘솔에서 직접 취소해 포트원 상태와 달라지면요?`

#### 모범 답변

> 공식 경로를 포트원 API와 콘솔로 제한하는 것이 우선이지만 외부 현실을 완전히 통제할 수는 없습니다. provider webhook 또는 주기적 inquiry로 차이를 감지하고 raw provider transaction과 normalized payment를 대사합니다. 자동 보정 가능한 전이는 idempotent state machine으로 반영하고, 금액이나 ownership이 불명확하면 manual review로 보냅니다. 수정은 audit log와 adjustment reason을 남깁니다.

### 8. Smart routing

#### 예상 질문

`PG 라우팅을 어떻게 설계할까요?`

#### 모범 답변

> 먼저 merchant contract와 payment method capability, 국가와 currency 같은 hard constraint로 후보를 좁힙니다. 그다음 성공률, latency, cost, 장애 상태를 점수화하되 짧은 구간의 noise에 과민 반응하지 않게 window와 minimum sample을 둡니다. 같은 payment의 retry가 다른 provider로 넘어가 중복 승인되지 않도록 routing decision을 attempt에 고정하고, 결과 불명 상태에서는 failover보다 기존 provider 조회를 우선합니다. 정책 변경은 version과 audit을 남기고 shadow evaluation과 제한된 rollout으로 검증합니다.

### 9. 글로벌 결제에서 추가되는 문제

- currency와 minor unit 차이, zero-decimal currency
- FX rate 시점과 환율 source
- timezone, settlement date, business day
- 3DS와 SCA, 지역별 인증 흐름
- async payment method와 긴 pending 상태
- chargeback, dispute, refund 기간
- data residency와 PCI scope
- provider별 idempotency와 webhook semantics
- local payment method의 redirect와 mobile app return

#### 답변 연결

> 저는 KCB와 KCS 연동에서 외부 기관의 redirect, callback, 점검 시간, provider-specific error를 내부 상태로 흡수해 본 경험이 있습니다. 포트원에서는 그 원칙을 결제수단과 PG별 capability, raw data 보존, normalized state machine으로 확장할 수 있습니다.

### 10. 간단한 시스템 디자인 답변 구조

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

#### 공식 참고 자료

- [포트원 V2 인증 결제 연동](https://developers.portone.io/opi/ko/integration/start/v2/checkout?v=v2)
- [포트원 V2 웹훅 연동](https://developers.portone.io/opi/ko/integration/webhook/readme-v2?v=v2)
- [포트원 V2 결제 취소](https://developers.portone.io/opi/ko/integration/cancel/v2/readme?v=v2)
- [포트원 V2 REST API](https://developers.portone.io/api/rest-v2/payment?v=v2)


---

## 06. 프로젝트, 오픈소스, 모의 면접

### 1. Moonberg

#### 예상 질문

`PGMQ를 선택한 이유와 전달 보장은 무엇인가요?`

#### 모범 답변

> 한 대뿐인 Bloomberg Terminal을 여러 운영자가 공유하므로 terminal 작업을 직렬화하고 상태와 결과를 영속화하는 것이 핵심이었습니다. 이미 PostgreSQL을 사용하고 있었고 별도 broker 운영을 늘리지 않기 위해 PGMQ를 선택했습니다. Go API가 요청을 queue에 넣고 Python worker가 visibility timeout 기반으로 가져와 처리합니다. worker가 죽으면 메시지는 다시 보일 수 있으므로 at-least-once를 전제로 job ID와 상태를 멱등하게 갱신했습니다. 다만 terminal UI automation 자체는 외부 idempotency key가 없어 중간 실패 위치에 따라 재실행 위험이 있습니다. 그래서 진행 상태와 결과 artifact를 분리하고, 성공 marker가 있는 step은 건너뛰며 ambiguous step은 운영자 확인 상태로 보내는 게 안전합니다.

#### `왜 SQS나 Temporal이 아니었나요?`

> 작업이 한 로컬 Windows Terminal에 묶여 있고 초기 규모가 작아 PostgreSQL 안에서 queue와 업무 데이터를 함께 운영하는 단순성이 컸습니다. SQS는 cloud decoupling과 scale에 유리하지만 로컬 worker 연결과 추가 운영 경계가 생깁니다. Temporal은 긴 workflow의 durable timer와 recovery에 강하지만 당시 workflow 수와 팀 규모에는 도입 비용이 더 컸습니다. step과 보상이 복잡해지고 장시간 대기가 많아진다면 Temporal을 재검토하겠습니다.

#### `60~80% 절감은 어떻게 계산했나요?`

> 면접 전 실제 표본을 확인해 `작업당 평균 수작업 시간 x 월 작업 건수`의 전후를 설명해야 합니다. 계측이 아니라 운영자 인터뷰 기반 추정이면 반드시 추정치라고 말합니다. 절감률만 말하지 말고 어떤 단계가 남았는지 설명합니다.

### 2. alembic dump

#### 예상 질문

`개발 도구에서 가장 위험한 보안 문제는 무엇이었나요?`

#### 모범 답변

> staging DB credential과 실제 데이터 취급입니다. CLI가 Secrets Manager에서 secret을 읽고 SSH tunnel과 SSL connection을 만들더라도 secret을 stdout, shell history, process argument, 임시 파일에 남기지 않아야 합니다. 최소 권한 read-only credential과 짧은 세션, tunnel 종료 보장, dump 파일 권한과 보존 기간이 필요합니다. 실제 데이터를 로컬로 가져오는 기능이라면 masking과 sampling, 승인 절차가 더 중요합니다. 편의 도구가 기존 접근 통제를 우회하지 않게 해야 합니다.

#### `migration branch 충돌을 어떻게 해결했나요?`

> Alembic revision graph의 현재 heads를 확인하고, 여러 head가 생기면 무조건 순서를 덮어쓰기보다 독립 migration이면 merge revision을 만들고 의존성이 있으면 rebase 또는 새 revision으로 조정합니다. 실제 staging schema와 migration history가 일치하는지 확인하고, data migration은 데이터 양과 lock 시간을 production 유사 환경에서 검증합니다. downgrade 가능 여부와 forward-fix 전략도 구분합니다.

### 3. Zenith

#### 예상 질문

`파일 삭제 도구의 안전성을 어떻게 보장합니까?`

#### 모범 답변

> 발견과 삭제를 분리하고 scan result를 immutable plan으로 만든 뒤 사용자 승인을 받습니다. allowlist된 root 아래의 canonical path만 처리하고 symlink를 따라 root 밖으로 나가지 않게 합니다. plan 생성과 실행 사이 TOCTOU를 줄이기 위해 실행 시 inode, path type, size 같은 precondition을 다시 확인합니다. source, Keychain, credential path는 denylist만으로 찾기보다 애초에 scanner scope에서 제외합니다. 기본은 dry-run과 trash 같은 복구 가능한 동작을 사용하고, 실제 삭제는 per-item audit과 오류 격리를 둡니다.

#### 꼬리 질문

`canonicalize만 하면 symlink race가 해결되나요?`

> 아닙니다. 확인 뒤 교체되는 race가 남습니다. 가능한 platform API에서 directory handle 기준의 relative operation과 no-follow flag를 쓰고, 실행 직전에 metadata를 재검증합니다. 완전한 방어 가능 범위는 OS API에 따라 다르므로 권한 자체를 최소화하고 broad recursive delete를 피합니다.

### 4. Temporal Python SDK 문서 기여

#### 예상 질문

`FunctionTool과 Activity tool의 차이를 설명해 주세요.`

#### 모범 답변

> Temporal Workflow code는 replay되므로 결정적이어야 하고 network나 file 같은 side effect를 직접 수행하면 안 됩니다. 일반 FunctionTool이 workflow context에서 실행되는 구성이라면 결정성 제약을 따라야 하고, Activity 기반 tool은 worker의 Activity로 실행돼 외부 I/O와 retry, timeout, heartbeat를 사용할 수 있습니다. 문서에서 실행 위치가 불명확해 SDK 구현을 따라가고 그 차이를 다이어그램으로 설명했습니다. 중요한 것은 tool 이름이 아니라 replay boundary와 failure semantics입니다.

#### `Activity가 성공했지만 결과 기록 전에 worker가 죽으면요?`

> Activity는 다시 실행될 수 있으므로 side effect가 멱등해야 합니다. provider idempotency key, durable operation ID, 결과 조회를 사용합니다. heartbeat가 있어도 exactly-once가 되는 것은 아닙니다.

### 5. Google Genkit Provider

#### 예상 질문

`OpenAI-compatible API면 provider 구현이 쉬운 것 아닌가요?`

#### 모범 답변

> wire format이 비슷해도 base URL, auth, model naming, streaming chunk, finish reason, tool call, usage, error mapping, cancellation이 다를 수 있습니다. SDK의 provider registration과 model capability contract를 맞추고, 정상 응답뿐 아니라 malformed response, 401, 429, 5xx, stream interruption을 테스트해야 합니다. 호환이라는 이름을 믿기보다 지원 capability를 명시적으로 선언하는 게 중요합니다.

### 6. AWS Chalice 기여

#### 예상 질문

`Lambda version과 alias가 SnapStart에 왜 필요한가요?`

#### 모범 답변

> Lambda version은 immutable deployment snapshot이고 alias는 특정 version을 가리키는 stable name입니다. SnapStart 같은 기능은 published version과 연결되므로 `$LATEST`만 배포해서는 원하는 lifecycle을 만들기 어렵습니다. alias를 쓰면 traffic을 특정 version으로 안정적으로 가리키고 점진 전환이나 rollback을 할 수 있습니다. 리뷰에서는 잘못된 alias 설정이 배포 후 실패하기보다 배포 전에 validation되게 하는 부분을 봤습니다. 단, 제가 해당 PR의 전체 구현자처럼 말하지 않고 리뷰에서 기여한 정확한 범위를 구분하겠습니다.

### 7. 30문항 모의 면접

각 질문은 먼저 소리 내어 60초 안에 답한 뒤 모범 포인트와 비교한다.

#### Q1. 왜 결제와 한도에 트랜잭션이 필요했나요?

불변식, 두 동시 승인 예시, 조건부 갱신, 결제와 Outbox의 atomic commit을 말한다.

#### Q2. 분산 락 없이 해결할 수 있나요?

가능성을 인정한다. DB invariant는 condition과 transaction이 보장하며 락은 경쟁 감소 또는 외부 흐름 직렬화에만 둔다.

#### Q3. 락 owner가 멈췄다가 살아나면요?

lease만으로 부족하다. owner token, version, fencing을 말한다.

#### Q4. DynamoDB TTL을 락 만료에 쓰면 안 되나요?

TTL 삭제는 지연될 수 있다. expiry attribute를 condition에서 직접 검사한다.

#### Q5. transaction 성공 응답을 잃으면요?

stable request token과 business ID로 재시도하고 조회로 확인한다.

#### Q6. Outbox도 유실될 수 있나요?

DB와 함께 저장돼 publish intent 유실을 줄이지만 relay 중복, stream retention, poison event는 남는다.

#### Q7. EventBridge와 SQS를 둘 다 둔 이유는요?

EventBridge는 routing과 fan-out, SQS는 buffering, backpressure, retry, consumer 격리라는 책임을 설명한다. 실제 fan-out 요구가 약했다면 과설계 가능성도 인정한다.

#### Q8. SQS FIFO가 exactly-once인가요?

delivery dedup 범위와 application side effect를 구분한다. consumer idempotency가 필요하다.

#### Q9. customer Message Group ID의 단점은요?

heavy customer hot group과 head-of-line blocking. payment 단위 분리 가능성을 말한다.

#### Q10. 상태를 먼저 읽어 skip하면 멱등한가요?

check-then-act race가 있다. conditional claim과 transaction이 필요하다.

#### Q11. 외부 API timeout 시 재시도하면 되나요?

unknown outcome이다. provider idempotency key와 inquiry를 먼저 사용한다.

#### Q12. DLQ 메시지를 바로 redrive해도 되나요?

원인 해결과 상태 대사 후 같은 ID로 소량 재처리한다.

#### Q13. DynamoDB projection을 PostgreSQL과 어떻게 맞추나요?

source of truth, version, dedup, lag metric, reconciliation과 rebuild를 말한다.

#### Q14. sending_date partition key의 문제는요?

하루 대상 집중으로 hot key 가능. write sharding과 fan-out tradeoff를 말한다.

#### Q15. 정산 데이터를 mutable row로 두면 안 되나요?

확정 후 audit이 중요하다. immutable entry와 adjustment를 선호한다.

#### Q16. QueryCriteria가 SQL injection을 막나요?

SQLAlchemy binding만 믿지 않고 field/operator allowlist와 tenant scope를 둔다.

#### Q17. offset pagination의 문제는요?

deep offset 비용과 concurrent insert의 중복/누락. stable keyset을 설명한다.

#### Q18. HMAC이 callback replay를 막나요?

아니다. timestamp, nonce, one-time transaction state가 필요하다.

#### Q19. CBC의 문제는요?

인증되지 않은 암호화다. 신규는 AEAD, 외부 규격이면 random IV와 MAC, 오류 비노출을 본다.

#### Q20. CI hash는 익명정보인가요?

아니다. deterministic pseudonym이며 개인정보 통제가 계속 필요하다.

#### Q21. CORS가 API 호출을 막나요?

브라우저 response 읽기 정책일 뿐이다. 인증과 인가를 별도로 한다.

#### Q22. JWT body overwrite면 IDOR 방어가 끝나나요?

하위 resource ownership과 repository tenant scope까지 강제해야 한다.

#### Q23. 내부 API key의 한계는요?

shared blast radius, attribution, rotation 문제. IAM, SigV4, mTLS 대안을 말한다.

#### Q24. warmer가 cold start를 보장하나요?

아니다. scale-out에는 약하다. 엄격한 SLO는 Provisioned Concurrency를 비교한다.

#### Q25. 여러 PG를 하나의 interface로 만들 때 무엇이 깨지나요?

capability 차이와 provider 원본 정보 손실. capability matrix와 raw fields를 둔다.

#### Q26. 결제 성공 callback을 믿으면 안 되는 이유는요?

브라우저 위변조와 유실 가능. server-side inquiry와 주문 금액 검증이 필요하다.

#### Q27. webhook과 callback이 동시에 오면요?

같은 state machine, event/payment ID 멱등, version 조건을 쓴다.

#### Q28. 부분 취소 두 건이 동시에 오면요?

remaining cancellable amount를 version 조건으로 예약하고 합계 불변식을 지킨다.

#### Q29. MSA 15개가 좋은 선택이었나요?

장점과 팀 운영 비용을 동시에 말하고 지금은 modular monolith에서 시작할 수 있다고 답한다.

#### Q30. 포트원에 바로 기여할 수 있는 경험은요?

외부 provider 차이를 공통 상태로 흡수한 경험, 결제와 한도의 정합성, unknown outcome, webhook 멱등성, 운영 복구를 연결한다.

### 8. 행동 질문에 기술 깊이 넣기

#### `가장 어려운 장애는?`

STAR만 말하지 말고 다음을 포함한다.

1. 사용자 또는 금액 영향
2. 탐지 signal과 최초 가설
3. timeline과 좁혀 간 증거
4. 완화와 영구 수정
5. 재발 방지 test, alarm, runbook
6. 본인의 구체적 결정

#### `의견 충돌은?`

기술 선택 자체보다 판단 기준을 보여준다.

> 동기 호출을 유지할지 queue로 분리할지 의견이 달랐습니다. 외부 지연이 결제 응답 SLO에 미치는 영향, 처리 결과가 즉시 필요한지, 재시도와 운영 복구 비용을 표로 비교했습니다. 결제 승인과 한도는 동기로 유지하고 계약과 알림은 비동기로 분리했습니다. 이후 queue age와 end-to-end completion time을 측정해 가정을 확인했습니다.

### 9. 포트원에 물어볼 좋은 질문

- PG 원본 상태와 포트원의 normalized 상태가 충돌할 때 source of truth와 reconciliation 원칙은 무엇인가요?
- 새로운 PG를 붙일 때 공통 abstraction을 변경하는 경우와 provider extension으로 격리하는 기준은 무엇인가요?
- 결제 callback, webhook, polling 결과가 경쟁할 때 상태 전이와 멱등성을 어떤 계층에서 책임지나요?
- 기술 면접에서 합류 후 맡을 가능성이 높은 문제와 현재 팀이 가장 개선하고 싶은 reliability metric은 무엇인가요?
- V1과 V2를 함께 운영하면서 schema와 behavior compatibility를 어떻게 검증하나요?
- 장애 시 고객사별 영향 범위와 provider별 오류를 얼마나 빠르게 구분할 수 있나요?
- FDE와 backend engineer가 고객 이슈를 제품 abstraction으로 승격시키는 과정은 어떻게 나뉘나요?


---

## 07. 면접 직전 압축 복습

### 1. 내 핵심 메시지 세 문장

1. 결제와 한도처럼 함께 바뀌어야 하는 상태는 조건부 쓰기와 트랜잭션으로 불변식을 지켰다.
2. 외부 기관과 후속 작업은 실패와 중복을 전제로 상태를 영속화하고 멱등성, 재시도, DLQ, 대사 경로를 만들었다.
3. 포트원에서도 PG 차이를 공통 모델로 추상화하되 원본 상태와 오류를 잃지 않고 운영 가능한 시스템을 만드는 데 기여할 수 있다.

### 2. 10초 정의

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

### 3. 숫자와 제한

- DynamoDB `TransactWriteItems`: 최대 100 actions, 4MB, 같은 account와 Region.
- SQS FIFO dedup: 제한된 deduplication interval만 제공. 영구 dedup으로 사용하지 않는다.
- Lambda SQS: batch가 중복 전달될 수 있다. partial batch failure를 고려한다.
- DynamoDB TTL: 만료 시각에 즉시 삭제된다고 보장하지 않는다.
- 금액: float 금지, integer minor unit + currency.

### 4. 무조건 먼저 말할 실패 시나리오

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

### 5. 위험 문장 금지

- `완벽하게 보장했습니다`
- `정확히 한 번 처리됩니다`
- `CORS로 보안을 막았습니다`
- `ULID라 해킹할 수 없습니다`
- `HMAC이 replay를 막습니다`
- `암호화했으니 개인정보가 아닙니다`
- `MSA라 확장성이 좋습니다`
- `NoSQL이 RDB보다 빠릅니다`

대신 보장 범위와 남은 실패를 정확히 말한다.

### 6. 모범 답변 만능 골격

> 당시 문제의 핵심 불변식은 ___였습니다. ___ 경쟁 또는 실패가 생기면 ___가 깨질 수 있었습니다. 그래서 ___를 적용했고, 구체적으로 ___ 조건에서만 상태를 변경했습니다. 전달과 외부 side effect는 중복될 수 있어 ___로 멱등하게 처리했습니다. 운영에서는 ___ metric과 ___ 복구 경로를 뒀습니다. 이 선택의 한계는 ___이고, 현재 다시 설계한다면 규모와 요구에 따라 ___도 비교하겠습니다.

### 7. 마지막으로 확인할 개인 사실

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
