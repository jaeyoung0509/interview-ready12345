# 04. 데이터 모델, API 추상화, Lambda, 관측성

## 1. DynamoDB와 PostgreSQL을 왜 함께 썼나

### 예상 질문

`결제는 DynamoDB, 정산은 PostgreSQL로 나눈 이유가 무엇인가요?`

### 모범 답변

> 결제와 고객의 온라인 상태 변경은 key 기반 접근, 조건부 쓰기, 트랜잭션, Streams 연동이 중요해 DynamoDB를 사용했습니다. 반면 정산과 운영 리포트는 기간, 파트너, 상태, 금액을 조합한 조회와 집계가 많아 PostgreSQL이 적합했습니다. 중요한 점은 두 DB를 동시에 source of truth로 두지 않는 것입니다. 결제 상태는 DynamoDB를 기준으로 하고 Streams event로 PostgreSQL projection을 갱신했습니다. projection은 지연되거나 누락될 수 있으므로 event ID와 aggregate version으로 중복과 역전을 막고, lag metric과 정기 reconciliation으로 원본과 대사해야 합니다.

### 꼬리 질문

`Streams event가 누락되거나 순서가 바뀌면요?`

> consumer checkpoint와 DLQ만으로 끝내지 않고 iterator age, failed batch, projection lag를 감시합니다. 각 projection row에 마지막 적용 version을 저장해 오래된 event를 무시하고, 정기적으로 source와 projection의 count와 금액 checksum을 비교합니다. 차이가 나면 원본에서 해당 기간 또는 aggregate를 다시 build할 수 있어야 합니다.

### 베스트 프랙티스

- 하나의 데이터에 대한 authoritative source를 명시한다.
- projection은 rebuild 가능하게 만든다.
- event ID unique constraint와 aggregate version 조건을 둔다.
- 금액과 건수 reconciliation을 자동화한다.
- 정산 확정 시점에는 projection lag를 고려한 cutoff와 close 절차를 둔다.
- 개인정보 삭제와 retention이 두 저장소에 모두 적용되게 한다.

## 2. 상환 알림 데이터 모델

### 예상 질문

`sending_date를 partition key로 쓰면 hot partition 문제가 없나요?`

### 모범 답변

> 해당 날짜 대상만 읽는 access pattern에는 맞지만 모든 고객이 같은 날짜 key에 몰리면 write와 read가 한 partition key에 집중될 수 있습니다. 실제 규모에서 throttling이 없었는지 metric으로 확인해야 합니다. 규모가 커지면 `sending_date#shard`로 write sharding하고 당일 모든 shard를 병렬 조회하거나, 날짜 bucket과 partner를 함께 key로 구성할 수 있습니다. shard 수는 peak 건수와 처리량을 기반으로 정하고, 무작정 늘리면 query fan-out 비용이 커집니다.

### 합산과 취소 불변식

> 고객과 발송일 단위의 예정 금액을 transaction으로 갱신하되, 각 payment event ID를 중복 반영하지 않도록 해야 합니다. 단순히 합계만 저장하면 재처리 시 중복 합산될 수 있으므로 event ledger나 applied-event record를 같이 저장하는 방법이 있습니다. 합계가 0이면 soft delete를 했다고 되어 있는데, 이후 재승인이나 복구가 가능한 상태 전이라면 tombstone과 version을 유지해 오래된 event가 되살리지 못하게 해야 합니다. 발송 직전에는 최신 CMS 동의와 실제 미납 금액을 다시 확인해 projection 지연으로 잘못 발송하는 것을 막습니다.

## 3. 정산 모델

### 예상 질문

`결제와 정산을 어떻게 분리해서 모델링합니까?`

### 모범 답변

> 결제는 고객이 지불했는지에 대한 거래 상태이고, 정산은 그 거래를 기준으로 누구에게 얼마를 언제 지급할지 계산하고 확정하는 별도 도메인입니다. 하나의 payment가 부분 취소, 수수료, 세금, 프로모션, 여러 하위 상점 때문에 여러 settlement line으로 나뉠 수 있습니다. 계산 결과는 versioned snapshot 또는 immutable ledger entry로 남기고, 확정 후 변경은 기존 row 수정이 아니라 adjustment entry로 추적하는 편이 감사에 유리합니다. PG 원본 금액, 포트원 normalized 값, 고객사 주문 값 사이 대사를 별도로 둡니다.

### 핵심 테이블 또는 aggregate 개념

- payment, payment_attempt, provider_transaction
- cancellation 또는 refund
- settlement_batch, settlement_line
- fee, tax, promotion allocation
- payout와 payout failure
- adjustment와 reconciliation_result
- raw_provider_event와 normalized_event

## 4. QueryCriteria 추상화

### 예상 질문

`SQLAlchemy QueryCriteria는 어떤 문제를 어떻게 해결했나요?`

### 모범 답변

> 운영 API마다 기간, 상태, 파트너, 정렬, pagination 조건을 반복 구현하면서 validation과 join 방식이 달라지는 문제가 있었습니다. 허용된 field와 operator를 typed criteria로 제한하고, domain filter를 SQLAlchemy expression으로 변환하는 공통 계층을 만들었습니다. endpoint는 filter 조합만 선언하고 tenant scope와 기본 정렬, pagination은 공통 적용했습니다. 그래서 대표적인 조회 API 개발 시간을 줄였습니다. 다만 범용 query language로 키우면 임의 join, 느린 sort, SQL injection, count query 폭증 위험이 있어 field와 operator를 allowlist하고 query plan과 index를 access pattern별로 검증했습니다.

### 구조 예시

```text
HTTP query params
  -> validation DTO
  -> typed QueryCriteria
  -> repository-specific compiler
  -> SQLAlchemy expression
  -> stable sort + cursor
```

### 꼬리 질문

`offset pagination과 cursor pagination 중 무엇을 썼나요?`

> 실제 구현을 기준으로 답해야 합니다. 베스트 프랙티스로는 깊은 페이지와 변경이 잦은 결제 목록에는 stable unique ordering을 가진 keyset pagination을 선호합니다. 예를 들어 `created_at DESC, id DESC`를 cursor에 넣습니다. offset은 단순하지만 깊어질수록 비용이 늘고 중간 insert 때문에 중복이나 누락이 생길 수 있습니다.

`공통화가 잘못된 사례는요?`

> filter DSL이 domain의 특수한 의미를 숨기거나, 모든 관계를 dynamic join으로 풀게 되면 성능 예측이 어렵습니다. 공통화는 validation과 반복되는 operator까지만 두고, 복잡한 정산 query는 명시적인 repository method나 read model로 빼겠습니다.

## 5. 파트너별 정책과 공통 규칙

### 예상 질문

`파트너별 if문을 어떻게 줄였나요?`

### 모범 답변

> 먼저 모든 파트너에 동일한 domain invariant와 실제로 달라지는 policy를 분리했습니다. 예를 들어 승인 금액이 가용 한도를 넘지 않는 것은 공통 규칙이고, 한도 계산 방식, 계약 필요 여부, 정산 주기는 파트너 policy가 될 수 있습니다. use case는 공통 흐름을 소유하고 policy interface를 호출하며, partner ID에 따른 구현 선택은 composition root에서 합니다. 단, 차이가 단순 parameter라면 별도 class를 늘리지 않고 configuration이나 decision table로 표현합니다. 규칙 변경 이력을 남기고 당시 payment가 어느 policy version으로 계산됐는지도 보존하는 게 중요합니다.

### 위험 신호

- strategy class가 사실상 if문을 파일로 옮긴 것뿐이다.
- partner마다 aggregate와 상태 이름이 달라 공통 domain model이 무너진다.
- 설정 변경이 과거 거래 계산을 바꾼다.
- 운영자가 규칙을 바꿨는데 audit과 approval이 없다.
- partner custom field가 core table에 무한히 늘어난다.

## 6. DDD와 Hexagonal Architecture

### 예상 질문

`DDD를 실제로 어디에 적용했나요?`

### 모범 답변

> 폴더 구조보다 결제, 한도, 계약, 정산의 용어와 불변식을 코드 모델에 반영한 것이 핵심이었습니다. use case가 HTTP나 DynamoDB SDK에 직접 의존하지 않도록 repository와 external provider port를 두고 adapter에서 Chalice, PynamoDB, KCB/KCS API를 연결했습니다. 그 덕분에 외부 기관 response를 domain result로 변환하고 retry나 mapping을 adapter 경계에 둘 수 있었습니다. 다만 10개 이상 작은 서비스에 동일한 계층을 반복하면 ceremony가 커졌고, 팀 규모와 변경 빈도에 비해 MSA가 과한 부분도 있었습니다. 지금은 bounded context와 독립 배포 필요가 명확한 곳에만 경계를 두겠습니다.

### `MSA 15개가 정말 필요했나요?`

> 결과적으로 작은 팀에는 배포, 로컬 재현, observability, event contract 관리 비용이 컸습니다. 외부 장애 격리와 scaling 단위 분리에는 장점이 있었지만, 서비스 수 자체를 성과로 보지는 않습니다. 처음부터 다시 한다면 modular monolith 또는 더 큰 서비스 단위로 시작하고 팀 소유권, 독립 scale, 보안 경계, 변경 주기가 실제로 달라질 때 분리하겠습니다.

## 7. Lambda cold start

### 예상 질문

`P99를 2~3초에서 200ms대로 줄였다는 것을 설명해 주세요.`

### 모범 답변

> 먼저 CloudWatch에서 어떤 함수와 기간, memory, runtime 기준인지 분리해 cold invocation의 init duration과 전체 duration을 봤습니다. Python import graph에서 사용하지 않는 무거운 module을 handler import 시점에 모두 불러오던 부분을 필요한 경로에서만 lazy load하도록 바꿨고, 사용자 traffic이 있는 시간대에는 scheduler warm-up도 적용했습니다. 관측한 구간에서 P99가 2~3초에서 200ms대로 내려갔습니다. 다만 warmer는 scale-out이나 AWS의 environment recycle 때 보장되지 않고 비용과 noise가 있습니다. 엄격한 latency SLO라면 Provisioned Concurrency, package 축소, memory 조정, runtime 개선을 비용과 함께 비교하겠습니다.

### 반드시 준비할 측정 상세

- 전체 API latency인지 Lambda duration인지
- cold sample을 어떻게 식별했는지
- P99 집계 window와 invocation 수
- warm-up 호출을 percentile에서 제외했는지
- memory size와 deployment package 변화
- lazy import가 첫 해당 기능 호출로 지연을 옮긴 것은 아닌지
- 오류율과 비용 변화

## 8. 관측성

### 예상 질문

`이벤트 ID 하나만 전달하면 distributed tracing이 되나요?`

### 모범 답변

> correlation에는 도움이 되지만 완전한 distributed tracing은 아닙니다. trace ID와 span context를 표준 header와 message attribute로 전파하고, 각 hop에서 span과 구조화 로그에 payment ID, event ID를 연결해야 합니다. 비동기 queue에서는 producer trace와 consumer processing 사이를 link로 표현할지 child span으로 표현할지 도구 특성을 봅니다. 중요한 것은 한 요청을 찾는 것뿐 아니라 queue 대기 시간과 provider 호출 시간, retry를 분리해 병목 위치를 알 수 있게 하는 것입니다.

### Golden signals와 business metrics

- latency: API, queue wait, handler, provider별
- traffic: 결제 요청, 승인, 취소, webhook량
- errors: 기술 오류와 business decline 분리
- saturation: Lambda concurrency, queue age, DB connection, throttling
- business: 승인율, 상태별 체류, 취소율, 대사 불일치, 정산 지연

### 공식 참고 자료

- [AWS Lambda cold starts](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)
- [AWS Lambda Provisioned Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)
- [PostgreSQL multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
- [OpenTelemetry traces](https://opentelemetry.io/docs/concepts/signals/traces/)

