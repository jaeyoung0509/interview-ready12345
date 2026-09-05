# 01. 결제 정합성, DynamoDB 트랜잭션, 분산 락, Outbox

## 1. 어떤 정합성 문제였나

### 예상 질문

`결제 승인과 한도 변경이 어긋난다는 게 구체적으로 어떤 상황인가요?`

### 1분 모범 답변

> 고객의 잔여 한도가 10만 원일 때 8만 원 승인 요청 두 건이 동시에 들어오는 상황이 대표적입니다. 둘 다 10만 원을 읽은 뒤 각각 승인하면 총 16만 원이 승인될 수 있습니다. 저희의 핵심 불변식은 승인된 결제 총액이 가용 한도를 넘지 않는 것, 그리고 결제 승인 생성과 한도 차감이 함께 성공하거나 함께 실패하는 것이었습니다. 이를 위해 한도 레코드에 조건부 갱신을 걸고, 결제 승인, 한도 차감, Outbox 이벤트 생성을 `TransactWriteItems` 하나로 커밋했습니다. 조건 충돌은 정상적인 동시성 결과로 보고 제한적으로 재시도하거나 한도 부족으로 반환했습니다.

### 3분 심화 포인트

- 읽고 계산한 뒤 쓰는 흐름만으로는 lost update가 발생한다.
- 가능한 경우 `remaining_limit >= amount` 조건과 차감 연산을 한 item의 atomic update로 표현한다.
- 결제 item 생성에는 `attribute_not_exists(pk)`를 넣어 같은 결제 ID의 재생성을 막는다.
- Outbox item도 같은 트랜잭션에 넣어 DB 커밋과 이벤트 발행 의도의 dual write를 없앤다.
- 트랜잭션 응답을 잃어 결과가 불명확할 수 있으므로 안정적인 `ClientRequestToken` 또는 비즈니스 idempotency key가 필요하다.
- 충돌 재시도에는 작은 횟수 제한과 jitter를 둔다. 한 고객이 hot key가 되면 무작정 재시도하지 않는다.

### 면접관의 꼬리 질문

`왜 락까지 필요했나요? 트랜잭션 조건부 쓰기면 충분하지 않나요?`

> 맞습니다. DB 내부 불변식만 보면 트랜잭션과 조건부 쓰기가 최종 안전장치이고, 별도 락이 항상 필요한 것은 아닙니다. 당시 락은 같은 고객의 긴 승인 흐름이 겹치는 것을 줄이고 충돌을 앞단에서 제어하려는 목적이었습니다. 다만 락이 트랜잭션보다 안전한 것은 아니고, lease 만료나 소유권 상실 문제가 있습니다. 지금 다시 설계한다면 외부 API 호출을 포함한 흐름을 꼭 직렬화해야 하는지 먼저 확인하고, DB 변경은 조건부 트랜잭션만으로 처리하는 단순한 설계를 우선 검토하겠습니다.

## 2. DynamoDB 트랜잭션은 무엇을 보장하나

### 예상 질문

`TransactWriteItems의 isolation 수준과 제한은 무엇인가요?`

### 모범 답변

> 한 리전과 한 계정 안에서 최대 100개의 서로 다른 item에 대한 쓰기를 all-or-nothing으로 수행하고, 전체 item 크기는 4MB 제한이 있습니다. 트랜잭션 쓰기끼리와 일반적인 단일 item 쓰기 사이에는 serializable isolation이 적용되는 범위가 있지만, GSI와 Streams에서 즉시 하나의 원자적 스냅샷으로 관찰된다고 생각하면 안 됩니다. 트랜잭션은 각 item마다 prepare와 commit에 해당하는 용량을 사용해 일반 쓰기보다 비용이 높고, 같은 item에 경쟁이 많으면 transaction conflict가 납니다. 따라서 작은 aggregate의 핵심 불변식에만 쓰고, 리포트성 데이터나 후속 처리는 비동기로 분리하는 게 좋습니다.

### 반드시 기억할 제한

- 같은 item에 `ConditionCheck`와 `Update`를 동시에 넣을 수 없다. 조건은 Update의 condition expression으로 합친다.
- GSI를 대상으로 직접 transaction action을 수행할 수 없다.
- Streams record가 트랜잭션 단위로 묶여 한 번에 보인다고 가정하지 않는다.
- global table의 일반적인 multi-region eventual consistency 모드에서는 다른 리전까지 하나의 글로벌 ACID 트랜잭션이 아니다.
- 취소 사유에는 조건 실패, 충돌, 처리량, validation 등이 섞일 수 있어 구분과 metric이 필요하다.

### 베스트 프랙티스

1. transaction item 수를 작고 고정되게 유지한다.
2. user input 전체가 아니라 안정적인 business request ID로 idempotency를 건다.
3. `CancellationReasons`를 관측해 한도 부족과 시스템 충돌을 구분한다.
4. hot partition과 특정 고객의 경합률을 metric으로 본다.
5. transaction 바깥 side effect는 절대 같은 성공으로 간주하지 않고 Outbox로 연결한다.

## 3. 분산 락을 구현했다면 어디까지 설명해야 하나

### 예상 질문

`DynamoDB 조건부 쓰기 락의 구조를 설명해 주세요.`

### 안전한 모범 답변

> 락 item은 customer ID를 key로 하고 owner token과 lease expiry를 저장합니다. 획득은 item이 없거나 lease가 만료된 경우에만 성공하는 조건부 쓰기로 하고, 갱신과 해제는 owner token이 현재 실행자와 같을 때만 허용해야 합니다. 프로세스가 죽어도 lease가 지나면 다른 실행자가 획득할 수 있습니다. 다만 오래 멈춘 실행자가 lease 만료 후 다시 살아나 쓰기를 수행하는 stale owner 문제가 남기 때문에, 중요한 쓰기는 락만 믿지 않고 DB version 조건이나 fencing token으로 거부해야 합니다. DynamoDB TTL은 삭제 시각을 보장하지 않으므로 lock correctness에 TTL 삭제 자체를 사용하지 않고, 애플리케이션이 expiry 값을 조건식으로 판단해야 합니다.

### 면접 전에 실제 코드에서 확인할 것

- lock item의 PK
- owner token이 매 요청마다 고유했는지
- lease 기간과 연장 방식
- release가 소유자 조건부 delete였는지
- Lambda timeout보다 lease가 길었는지
- stale owner를 막는 version 또는 fencing token이 있었는지
- 락 획득 후 외부 API를 호출했는지

확인하지 못한 항목은 구현했다고 단정하지 않는다.

## 4. Outbox는 어떤 문제를 풀었나

### 예상 질문

`결제 저장 후 EventBridge에 바로 publish하면 안 되나요?`

### 모범 답변

> DB 저장과 메시지 publish는 서로 다른 시스템이라 원자적으로 묶이지 않습니다. DB는 성공했는데 publish가 실패하면 후속 계약이나 정산이 영원히 시작되지 않고, publish 후 DB가 실패하면 존재하지 않는 결제를 소비자가 보게 됩니다. 그래서 결제 상태와 발행할 Outbox record를 같은 DynamoDB 트랜잭션에 저장했습니다. Streams relay는 Outbox record를 읽어 EventBridge로 전달합니다. 다만 이것은 exactly-once가 아니라 at-least-once relay이므로 event ID를 고정하고 소비자가 중복을 제거해야 합니다.

### Relay 설계 베스트 프랙티스

- Outbox event에는 `event_id`, `aggregate_id`, `aggregate_version`, `event_type`, `occurred_at`, `schema_version`, 최소 payload를 둔다.
- 소비자가 DB를 다시 읽어야 한다면 현재 상태와 event 시점 상태가 다를 수 있음을 고려한다.
- publish 성공 표시를 별도 update할 경우 그 update가 다시 Stream을 만들지 filter한다.
- event source mapping의 partial batch response와 실패 record 보존 전략을 정한다.
- 오래된 stream record가 retention을 넘기기 전에 장애를 알리는 iterator age alarm을 둔다.
- event schema는 additive change를 기본으로 하고 producer와 consumer의 독립 배포를 테스트한다.
- 삭제 여부와 보존 기간은 감사 요구와 재처리 전략으로 결정한다.

### 꼬리 질문

`Streams에서 같은 트랜잭션의 여러 변경 순서가 보장되나요?`

> 같은 item의 변경 순서는 shard 내에서 보존되지만, 서로 다른 item이나 shard 전체에 대한 전역 순서는 가정하면 안 됩니다. 따라서 결제와 한도 record가 보이는 순서에 의존하지 않고, 하나의 aggregate version이나 소비자의 상태 검증으로 처리해야 합니다.

## 5. 취소와 환불의 동시성

### 예상 질문

`승인과 취소가 동시에 들어오면 어떻게 합니까?`

### 모범 답변

> 먼저 허용 가능한 상태 전이를 정의합니다. 예를 들어 `APPROVED -> CANCEL_PENDING -> CANCELLED`를 두고, 현재 상태와 version이 기대값일 때만 다음 상태로 바꾸는 conditional update를 사용합니다. PG 취소처럼 외부 side effect가 있으면 내부 트랜잭션만으로 원자성을 만들 수 없습니다. 그래서 취소 요청 idempotency key를 저장하고 `CANCEL_PENDING`을 먼저 커밋한 뒤 외부 PG를 호출합니다. 타임아웃으로 성공 여부가 불명확하면 같은 키로 재시도하거나 PG 조회 API로 결과를 확인합니다. 최종 webhook과 능동 조회 결과가 경쟁할 수 있으므로 동일한 상태 머신과 version 조건을 통과하게 합니다. 한도 복원은 최종 취소 확인과 연결하고 중복 복원이 불가능하도록 취소 transaction ID를 dedup key로 둡니다.

### 결제 도메인의 핵심 불변식 예시

- 승인 합계는 고객 한도를 넘지 않는다.
- 한 결제의 누적 취소 금액은 승인 금액을 넘지 않는다.
- 동일 취소 요청은 한 번만 금액에 반영된다.
- 최종 상태가 된 거래가 이전 상태로 회귀하지 않는다. 단, 명시적 조정 이벤트는 별도 모델링한다.
- 금액은 정수 최소 화폐 단위와 currency로 표현한다. float를 쓰지 않는다.
- 외부 PG 원본 transaction ID는 provider namespace와 함께 unique해야 한다.

## 6. 이 주제의 압박 질문

### `락을 잡은 Lambda가 죽으면요?`

> lease expiry로 회수하되, TTL 삭제 시각은 믿지 않습니다. 새 owner가 생긴 뒤 이전 실행이 돌아오는 상황은 owner token, version 조건 또는 fencing token으로 차단합니다.

### `트랜잭션 응답 직전에 네트워크가 끊기면 성공인지 어떻게 아나요?`

> 동일한 business request ID와 `ClientRequestToken`으로 재시도하고, 결제 ID의 conditional put과 조회로 결과를 판별합니다. 새 ID를 발급해 다시 실행하면 중복 승인이 생길 수 있습니다.

### `DynamoDB 대신 PostgreSQL이면 더 쉽지 않나요?`

> 관계와 ad hoc query가 핵심이고 단일 DB transaction으로 충분하다면 PostgreSQL이 더 단순할 수 있습니다. 당시에는 서버리스 scale, key-value access pattern, Streams 통합이 강점이었지만 트랜잭션과 락을 과도하게 조립하고 있었다면 RDB 선택을 재검토해야 합니다. 데이터베이스 이름보다 access pattern, transaction boundary, 운영 복잡도로 판단하겠습니다.

### 공식 참고 자료

- [AWS DynamoDB transactions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)
- [AWS DynamoDB read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [AWS transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [AWS DynamoDB Lock Client](https://github.com/awslabs/amazon-dynamodb-lock-client)

