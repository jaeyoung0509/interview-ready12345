# 06. 프로젝트, 오픈소스, 모의 면접

## 1. Moonberg

### 예상 질문

`PGMQ를 선택한 이유와 전달 보장은 무엇인가요?`

### 모범 답변

> 한 대뿐인 Bloomberg Terminal을 여러 운영자가 공유하므로 terminal 작업을 직렬화하고 상태와 결과를 영속화하는 것이 핵심이었습니다. 이미 PostgreSQL을 사용하고 있었고 별도 broker 운영을 늘리지 않기 위해 PGMQ를 선택했습니다. Go API가 요청을 queue에 넣고 Python worker가 visibility timeout 기반으로 가져와 처리합니다. worker가 죽으면 메시지는 다시 보일 수 있으므로 at-least-once를 전제로 job ID와 상태를 멱등하게 갱신했습니다. 다만 terminal UI automation 자체는 외부 idempotency key가 없어 중간 실패 위치에 따라 재실행 위험이 있습니다. 그래서 진행 상태와 결과 artifact를 분리하고, 성공 marker가 있는 step은 건너뛰며 ambiguous step은 운영자 확인 상태로 보내는 게 안전합니다.

### `왜 SQS나 Temporal이 아니었나요?`

> 작업이 한 로컬 Windows Terminal에 묶여 있고 초기 규모가 작아 PostgreSQL 안에서 queue와 업무 데이터를 함께 운영하는 단순성이 컸습니다. SQS는 cloud decoupling과 scale에 유리하지만 로컬 worker 연결과 추가 운영 경계가 생깁니다. Temporal은 긴 workflow의 durable timer와 recovery에 강하지만 당시 workflow 수와 팀 규모에는 도입 비용이 더 컸습니다. step과 보상이 복잡해지고 장시간 대기가 많아진다면 Temporal을 재검토하겠습니다.

### `60~80% 절감은 어떻게 계산했나요?`

> 면접 전 실제 표본을 확인해 `작업당 평균 수작업 시간 x 월 작업 건수`의 전후를 설명해야 합니다. 계측이 아니라 운영자 인터뷰 기반 추정이면 반드시 추정치라고 말합니다. 절감률만 말하지 말고 어떤 단계가 남았는지 설명합니다.

## 2. alembic dump

### 예상 질문

`개발 도구에서 가장 위험한 보안 문제는 무엇이었나요?`

### 모범 답변

> staging DB credential과 실제 데이터 취급입니다. CLI가 Secrets Manager에서 secret을 읽고 SSH tunnel과 SSL connection을 만들더라도 secret을 stdout, shell history, process argument, 임시 파일에 남기지 않아야 합니다. 최소 권한 read-only credential과 짧은 세션, tunnel 종료 보장, dump 파일 권한과 보존 기간이 필요합니다. 실제 데이터를 로컬로 가져오는 기능이라면 masking과 sampling, 승인 절차가 더 중요합니다. 편의 도구가 기존 접근 통제를 우회하지 않게 해야 합니다.

### `migration branch 충돌을 어떻게 해결했나요?`

> Alembic revision graph의 현재 heads를 확인하고, 여러 head가 생기면 무조건 순서를 덮어쓰기보다 독립 migration이면 merge revision을 만들고 의존성이 있으면 rebase 또는 새 revision으로 조정합니다. 실제 staging schema와 migration history가 일치하는지 확인하고, data migration은 데이터 양과 lock 시간을 production 유사 환경에서 검증합니다. downgrade 가능 여부와 forward-fix 전략도 구분합니다.

## 3. Zenith

### 예상 질문

`파일 삭제 도구의 안전성을 어떻게 보장합니까?`

### 모범 답변

> 발견과 삭제를 분리하고 scan result를 immutable plan으로 만든 뒤 사용자 승인을 받습니다. allowlist된 root 아래의 canonical path만 처리하고 symlink를 따라 root 밖으로 나가지 않게 합니다. plan 생성과 실행 사이 TOCTOU를 줄이기 위해 실행 시 inode, path type, size 같은 precondition을 다시 확인합니다. source, Keychain, credential path는 denylist만으로 찾기보다 애초에 scanner scope에서 제외합니다. 기본은 dry-run과 trash 같은 복구 가능한 동작을 사용하고, 실제 삭제는 per-item audit과 오류 격리를 둡니다.

### 꼬리 질문

`canonicalize만 하면 symlink race가 해결되나요?`

> 아닙니다. 확인 뒤 교체되는 race가 남습니다. 가능한 platform API에서 directory handle 기준의 relative operation과 no-follow flag를 쓰고, 실행 직전에 metadata를 재검증합니다. 완전한 방어 가능 범위는 OS API에 따라 다르므로 권한 자체를 최소화하고 broad recursive delete를 피합니다.

## 4. Temporal Python SDK 문서 기여

### 예상 질문

`FunctionTool과 Activity tool의 차이를 설명해 주세요.`

### 모범 답변

> Temporal Workflow code는 replay되므로 결정적이어야 하고 network나 file 같은 side effect를 직접 수행하면 안 됩니다. 일반 FunctionTool이 workflow context에서 실행되는 구성이라면 결정성 제약을 따라야 하고, Activity 기반 tool은 worker의 Activity로 실행돼 외부 I/O와 retry, timeout, heartbeat를 사용할 수 있습니다. 문서에서 실행 위치가 불명확해 SDK 구현을 따라가고 그 차이를 다이어그램으로 설명했습니다. 중요한 것은 tool 이름이 아니라 replay boundary와 failure semantics입니다.

### `Activity가 성공했지만 결과 기록 전에 worker가 죽으면요?`

> Activity는 다시 실행될 수 있으므로 side effect가 멱등해야 합니다. provider idempotency key, durable operation ID, 결과 조회를 사용합니다. heartbeat가 있어도 exactly-once가 되는 것은 아닙니다.

## 5. Google Genkit Provider

### 예상 질문

`OpenAI-compatible API면 provider 구현이 쉬운 것 아닌가요?`

### 모범 답변

> wire format이 비슷해도 base URL, auth, model naming, streaming chunk, finish reason, tool call, usage, error mapping, cancellation이 다를 수 있습니다. SDK의 provider registration과 model capability contract를 맞추고, 정상 응답뿐 아니라 malformed response, 401, 429, 5xx, stream interruption을 테스트해야 합니다. 호환이라는 이름을 믿기보다 지원 capability를 명시적으로 선언하는 게 중요합니다.

## 6. AWS Chalice 기여

### 예상 질문

`Lambda version과 alias가 SnapStart에 왜 필요한가요?`

### 모범 답변

> Lambda version은 immutable deployment snapshot이고 alias는 특정 version을 가리키는 stable name입니다. SnapStart 같은 기능은 published version과 연결되므로 `$LATEST`만 배포해서는 원하는 lifecycle을 만들기 어렵습니다. alias를 쓰면 traffic을 특정 version으로 안정적으로 가리키고 점진 전환이나 rollback을 할 수 있습니다. 리뷰에서는 잘못된 alias 설정이 배포 후 실패하기보다 배포 전에 validation되게 하는 부분을 봤습니다. 단, 제가 해당 PR의 전체 구현자처럼 말하지 않고 리뷰에서 기여한 정확한 범위를 구분하겠습니다.

## 7. 30문항 모의 면접

각 질문은 먼저 소리 내어 60초 안에 답한 뒤 모범 포인트와 비교한다.

### Q1. 왜 결제와 한도에 트랜잭션이 필요했나요?

불변식, 두 동시 승인 예시, 조건부 갱신, 결제와 Outbox의 atomic commit을 말한다.

### Q2. 분산 락 없이 해결할 수 있나요?

가능성을 인정한다. DB invariant는 condition과 transaction이 보장하며 락은 경쟁 감소 또는 외부 흐름 직렬화에만 둔다.

### Q3. 락 owner가 멈췄다가 살아나면요?

lease만으로 부족하다. owner token, version, fencing을 말한다.

### Q4. DynamoDB TTL을 락 만료에 쓰면 안 되나요?

TTL 삭제는 지연될 수 있다. expiry attribute를 condition에서 직접 검사한다.

### Q5. transaction 성공 응답을 잃으면요?

stable request token과 business ID로 재시도하고 조회로 확인한다.

### Q6. Outbox도 유실될 수 있나요?

DB와 함께 저장돼 publish intent 유실을 줄이지만 relay 중복, stream retention, poison event는 남는다.

### Q7. EventBridge와 SQS를 둘 다 둔 이유는요?

EventBridge는 routing과 fan-out, SQS는 buffering, backpressure, retry, consumer 격리라는 책임을 설명한다. 실제 fan-out 요구가 약했다면 과설계 가능성도 인정한다.

### Q8. SQS FIFO가 exactly-once인가요?

delivery dedup 범위와 application side effect를 구분한다. consumer idempotency가 필요하다.

### Q9. customer Message Group ID의 단점은요?

heavy customer hot group과 head-of-line blocking. payment 단위 분리 가능성을 말한다.

### Q10. 상태를 먼저 읽어 skip하면 멱등한가요?

check-then-act race가 있다. conditional claim과 transaction이 필요하다.

### Q11. 외부 API timeout 시 재시도하면 되나요?

unknown outcome이다. provider idempotency key와 inquiry를 먼저 사용한다.

### Q12. DLQ 메시지를 바로 redrive해도 되나요?

원인 해결과 상태 대사 후 같은 ID로 소량 재처리한다.

### Q13. DynamoDB projection을 PostgreSQL과 어떻게 맞추나요?

source of truth, version, dedup, lag metric, reconciliation과 rebuild를 말한다.

### Q14. sending_date partition key의 문제는요?

하루 대상 집중으로 hot key 가능. write sharding과 fan-out tradeoff를 말한다.

### Q15. 정산 데이터를 mutable row로 두면 안 되나요?

확정 후 audit이 중요하다. immutable entry와 adjustment를 선호한다.

### Q16. QueryCriteria가 SQL injection을 막나요?

SQLAlchemy binding만 믿지 않고 field/operator allowlist와 tenant scope를 둔다.

### Q17. offset pagination의 문제는요?

deep offset 비용과 concurrent insert의 중복/누락. stable keyset을 설명한다.

### Q18. HMAC이 callback replay를 막나요?

아니다. timestamp, nonce, one-time transaction state가 필요하다.

### Q19. CBC의 문제는요?

인증되지 않은 암호화다. 신규는 AEAD, 외부 규격이면 random IV와 MAC, 오류 비노출을 본다.

### Q20. CI hash는 익명정보인가요?

아니다. deterministic pseudonym이며 개인정보 통제가 계속 필요하다.

### Q21. CORS가 API 호출을 막나요?

브라우저 response 읽기 정책일 뿐이다. 인증과 인가를 별도로 한다.

### Q22. JWT body overwrite면 IDOR 방어가 끝나나요?

하위 resource ownership과 repository tenant scope까지 강제해야 한다.

### Q23. 내부 API key의 한계는요?

shared blast radius, attribution, rotation 문제. IAM, SigV4, mTLS 대안을 말한다.

### Q24. warmer가 cold start를 보장하나요?

아니다. scale-out에는 약하다. 엄격한 SLO는 Provisioned Concurrency를 비교한다.

### Q25. 여러 PG를 하나의 interface로 만들 때 무엇이 깨지나요?

capability 차이와 provider 원본 정보 손실. capability matrix와 raw fields를 둔다.

### Q26. 결제 성공 callback을 믿으면 안 되는 이유는요?

브라우저 위변조와 유실 가능. server-side inquiry와 주문 금액 검증이 필요하다.

### Q27. webhook과 callback이 동시에 오면요?

같은 state machine, event/payment ID 멱등, version 조건을 쓴다.

### Q28. 부분 취소 두 건이 동시에 오면요?

remaining cancellable amount를 version 조건으로 예약하고 합계 불변식을 지킨다.

### Q29. MSA 15개가 좋은 선택이었나요?

장점과 팀 운영 비용을 동시에 말하고 지금은 modular monolith에서 시작할 수 있다고 답한다.

### Q30. 포트원에 바로 기여할 수 있는 경험은요?

외부 provider 차이를 공통 상태로 흡수한 경험, 결제와 한도의 정합성, unknown outcome, webhook 멱등성, 운영 복구를 연결한다.

## 8. 행동 질문에 기술 깊이 넣기

### `가장 어려운 장애는?`

STAR만 말하지 말고 다음을 포함한다.

1. 사용자 또는 금액 영향
2. 탐지 signal과 최초 가설
3. timeline과 좁혀 간 증거
4. 완화와 영구 수정
5. 재발 방지 test, alarm, runbook
6. 본인의 구체적 결정

### `의견 충돌은?`

기술 선택 자체보다 판단 기준을 보여준다.

> 동기 호출을 유지할지 queue로 분리할지 의견이 달랐습니다. 외부 지연이 결제 응답 SLO에 미치는 영향, 처리 결과가 즉시 필요한지, 재시도와 운영 복구 비용을 표로 비교했습니다. 결제 승인과 한도는 동기로 유지하고 계약과 알림은 비동기로 분리했습니다. 이후 queue age와 end-to-end completion time을 측정해 가정을 확인했습니다.

## 9. 포트원에 물어볼 좋은 질문

- PG 원본 상태와 포트원의 normalized 상태가 충돌할 때 source of truth와 reconciliation 원칙은 무엇인가요?
- 새로운 PG를 붙일 때 공통 abstraction을 변경하는 경우와 provider extension으로 격리하는 기준은 무엇인가요?
- 결제 callback, webhook, polling 결과가 경쟁할 때 상태 전이와 멱등성을 어떤 계층에서 책임지나요?
- 기술 면접에서 합류 후 맡을 가능성이 높은 문제와 현재 팀이 가장 개선하고 싶은 reliability metric은 무엇인가요?
- V1과 V2를 함께 운영하면서 schema와 behavior compatibility를 어떻게 검증하나요?
- 장애 시 고객사별 영향 범위와 provider별 오류를 얼마나 빠르게 구분할 수 있나요?
- FDE와 backend engineer가 고객 이슈를 제품 abstraction으로 승격시키는 과정은 어떻게 나뉘나요?

