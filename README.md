# PortOne technical interview preparation

이 저장소는 코리아포트원 기술 면접을 위해 이력서의 주장과 과거 보안 구현을 질문, 답변, 꼬리 질문 형태로 검증한 자료입니다.

## 추천 학습 순서

1. [`portone/00-interview-map.md`](portone/00-interview-map.md): 면접 전략과 위험 표현
2. [`portone/01-consistency-and-payments.md`](portone/01-consistency-and-payments.md): DynamoDB 트랜잭션, 동시성, Outbox
3. [`portone/02-event-driven-reliability.md`](portone/02-event-driven-reliability.md): SQS FIFO, 멱등성, 재시도, DLQ
4. [`portone/03-security-deep-dive.md`](portone/03-security-deep-dive.md): KCB/KCS, JWT, HMAC, AES, KMS
5. [`portone/04-data-api-observability.md`](portone/04-data-api-observability.md): PostgreSQL, QueryCriteria, Lambda, 관측성
6. [`portone/05-portone-system-design.md`](portone/05-portone-system-design.md): PG 추상화와 결제 시스템 설계
7. [`portone/06-projects-and-mock-interview.md`](portone/06-projects-and-mock-interview.md): 프로젝트, 오픈소스, 모의 면접
8. [`portone/07-last-minute-cheatsheet.md`](portone/07-last-minute-cheatsheet.md): 면접 직전 압축 복습

## 답변 원칙

모범 답변은 외워 읽는 대본이 아니다. 아래 순서만 유지한다.

1. 문제와 깨져서는 안 되는 불변식
2. 당시 선택과 구체적인 동작
3. 실패 시나리오와 운영 방법
4. 선택의 한계와 대안
5. 실제 수치 중 확실히 기억하는 것

구현을 직접 확인하지 못한 부분은 단정하지 않는다. `이력서 작성 과정에서 코드 분석으로 확인했지만 제가 직접 설계한 범위는 여기까지입니다`처럼 소유 범위를 분리한다.

