# 03. 보안 심층 면접 대비

## 0. 가장 먼저 말할 원칙

> 보안 기능의 이름을 나열하기보다 위협 모델, 신뢰 경계, 보호할 데이터, 키와 자격증명의 수명, 실패 시 동작을 먼저 설명하겠습니다. 당시 외부 기관이 요구한 프로토콜을 구현한 부분과 제가 새로 선택한 설계를 구분해서 말씀드리겠습니다.

이 문장이 중요한 이유는 첨부 보안 문서에 강한 표현이 일부 있기 때문이다. 아래 모범 답변은 그 표현을 정확하게 교정한다.

## 1. 전체 신뢰 경계

### 예상 질문

`보안 아키텍처를 2분 안에 설명해 주세요.`

### 모범 답변

> 외부 클라이언트, 공개 진입점, 내부 서비스, 데이터 저장소, KCB와 KCS 같은 외부 기관을 서로 다른 신뢰 경계로 봤습니다. 공개 요청은 JWT를 검증하고 토큰의 customer claim으로 대상 리소스를 결정해 client가 보낸 customer ID를 신뢰하지 않았습니다. 내부 서비스 호출은 당시 공유 API key로 인증했고, 민감정보와 외부 기관 secret은 Secrets Manager와 KMS 기반 저장 암호화를 사용했습니다. KCB callback은 HMAC으로 파라미터 무결성을 검증하고, 외부 규격에 따라 암호화된 본인인증 결과를 복호화했습니다. 로그에는 주민번호, CI, token, secret이 남지 않게 마스킹했습니다. 다만 공유 API key는 mTLS 수준의 상호 인증은 아니고, CORS도 인가 경계가 아닙니다. 현재 설계라면 workload identity, 짧은 수명의 자격증명, callback replay 방지, 민감정보 allowlist logging까지 보완하겠습니다.

## 2. JWT와 IDOR

### 예상 질문

`body의 customerId를 JWT cid로 덮어쓰면 IDOR이 완전히 해결되나요?`

### 모범 답변

> 중요한 방어지만 그것만으로 완전하다고 말하지는 않겠습니다. 먼저 JWT의 signature와 허용 algorithm, issuer, audience, expiration, not-before를 검증하고 key rotation을 고려해야 합니다. 그 뒤 subject와 customer claim의 관계를 서버가 신뢰할 수 있어야 합니다. endpoint는 client가 보낸 owner ID를 무시하고 검증된 principal에서 tenant scope를 만들며, repository query에도 같은 scope를 강제해야 합니다. 하위 리소스 ID만 받는 endpoint는 그 리소스가 해당 customer 소유인지 다시 확인합니다. 관리자나 파트너 권한은 별도의 role과 permission으로 분리하고 audit log를 남깁니다.

### 베스트 프랙티스 체크리스트

- `alg` allowlist를 고정하고 token header가 알고리즘을 선택하게 두지 않는다.
- `iss`, `aud`, `exp`, `nbf`를 검증한다.
- key ID와 rotation, 폐기 시나리오를 둔다.
- customer ID를 request에서 덮는 것에 그치지 않고 data access layer의 tenant scope를 강제한다.
- opaque resource ID라도 authorization check를 생략하지 않는다.
- 401은 인증 실패, 403은 인증됐지만 권한 부족으로 구분하되 과도한 존재 정보는 노출하지 않는다.
- raw token과 claims 전체를 로그에 남기지 않는다.

### 꼬리 질문

`JWT audience를 Origin으로 검증하는 게 맞나요?`

> 일반적으로 audience는 token이 의도된 수신 서비스나 API를 식별하고, browser Origin과 동일한 개념은 아닙니다. Origin은 CORS 입력이고 spoof 가능한 non-browser client도 있습니다. 실제 구현이 Origin 값을 aud로 사용했다면 당시 token 발급 계약을 먼저 확인해야 하고, 일반적인 베스트 프랙티스로는 issuer가 발급한 고정 API audience를 검증하고 Origin allowlist는 별도로 처리하겠습니다.

## 3. CORS의 정확한 의미

### 예상 질문

`CORS 화이트리스트가 어떤 공격을 막나요?`

### 모범 답변

> CORS는 브라우저가 다른 origin의 response를 script에서 읽는 것을 제한하는 정책입니다. curl이나 서버 간 요청을 막지 않으므로 인증이나 네트워크 접근 제어가 아닙니다. credential을 쓰는 경우 정확한 origin allowlist와 `Vary: Origin`을 설정하고 필요한 method와 header만 허용합니다. CSRF는 cookie 인증을 쓴다면 SameSite, CSRF token, Origin 검증 등으로 별도 방어합니다.

## 4. 내부 API key

### 예상 질문

`x-pymt-api-key는 충분히 안전한 서비스 간 인증인가요?`

### 모범 답변

> 당시에는 외부에 노출되지 않은 내부 endpoint에서 공유 secret을 검증해 무인증 호출을 막는 실용적인 통제였습니다. 하지만 장기 수명의 공용 key는 한 서비스가 침해되면 다른 서비스로 가장할 수 있고 호출 주체 구분과 rotation이 어렵습니다. 그래서 상호 인증이라고 부르지 않겠습니다. 개선한다면 서비스별 IAM role과 SigV4, private API resource policy, mTLS 또는 service mesh workload identity 중 환경에 맞는 방식을 사용하고, 서비스별 최소 권한과 짧은 수명, rotation, auditability를 확보하겠습니다.

### 헤더 strip에 대한 꼬리 질문

> hop-by-hop header와 infrastructure header를 제거하는 것은 좋지만, 임의의 forwarded header를 신뢰하면 spoofing이 생깁니다. 허용할 header를 allowlist로 새로 구성하는 편이 denylist보다 안전합니다. 내부 인증 header도 외부 요청에서 받은 값을 그대로 전달하지 않고 gateway가 제거한 뒤 새로 주입해야 합니다.

## 5. HMAC callback 서명

### 예상 질문

`HMAC으로 무엇을 보장했고 무엇은 못 보장하나요?`

### 모범 답변

> 공유 secret을 아는 당사와 외부 기관 사이에서 callback parameter가 변조되지 않았고 예상한 발신 규격에서 왔다는 무결성과 인증을 제공합니다. 암호화는 아니어서 payload를 숨기지 않고, 서명만으로 freshness나 replay는 막지 못합니다. 검증할 때는 수신한 raw bytes 또는 명확히 canonicalize한 representation을 사용하고 constant-time comparison을 합니다. timestamp 허용 범위, nonce나 transaction ID의 1회성 소비, 완료 상태의 conditional transition을 함께 적용해 replay를 막습니다.

### URL과 JSON canonicalization

- URL은 query parameter 순서, percent encoding, 공백, duplicate key 처리 규칙이 양쪽에서 같아야 한다.
- JSON을 임의로 parse 후 dump하면 key order, 숫자, Unicode 표현이 바뀔 수 있다.
- 가장 안전한 방식은 provider가 정의한 raw request body와 지정 header를 정확히 연결해 서명하는 것이다.
- 자체 webhook 발신 시 문서화된 canonical format 또는 raw bytes 서명을 제공하고 version을 붙인다.
- signature header에 key ID와 timestamp를 포함하면 rotation과 replay 방어가 쉬워진다.

### `같은 payload면 같은 서명`에 대한 정정

> 결정적 직렬화는 수신자 검증을 안정적으로 만들지만 replay를 쉽게 식별해 주지는 않습니다. event ID, timestamp, nonce와 delivery attempt를 포함하고 receiver가 event ID를 멱등 처리해야 합니다.

## 6. AES-CBC와 외부 기관 프로토콜

### 예상 질문

`AES-CBC 구현을 설명하고 보안상 주의점을 말해 주세요.`

### 모범 답변

> KCB와 KCS가 정한 연동 규격에 맞춰 AES-CBC와 PKCS#7 padding을 사용했습니다. CBC는 기밀성만 제공하고 자체 무결성은 없기 때문에 일반적인 신규 설계라면 AES-GCM 같은 AEAD를 우선합니다. 부득이하게 CBC 규격을 따라야 하면 무작위이며 예측 불가능하고 재사용되지 않는 IV가 필요하고, ciphertext와 IV 전체에 별도 MAC을 적용하는 encrypt-then-MAC 구조가 필요합니다. 복호화 오류도 padding 여부를 외부에 구분해 노출하지 않아야 합니다. KCS 규격처럼 key material의 일부를 고정 IV로 쓰는 구조였다면 제가 선택한 베스트 프랙티스가 아니라 provider compatibility 요구였다고 구분해서 설명하겠습니다.

### 왜 E2EE라고 부르면 위험한가

자사 backend가 주민번호 평문을 받고 암호화한다면 자사 backend는 한 endpoint다. 이는 TLS 위에 적용한 application-layer payload encryption이지, client에서 최종 KCS만 복호화할 수 있게 암호화한 진정한 end-to-end encryption이라고 단정하기 어렵다.

### 2단계 KCB 복호화 설명

> 본인인증 시작 시 transaction별 IV와 provider token, encrypted key material을 임시 저장했습니다. 완료 후 규격상 첫 단계에서 data encryption key를 얻고, 두 번째 단계에서 CI와 사용자 정보를 복호화했습니다. 복호화 후에는 필요한 최소 데이터만 남기고 임시 key material을 삭제했습니다. 여기서 중요한 것은 algorithm 이름보다 transaction binding, key 수명, 실패 시 삭제, 로그 비노출, 재처리 시 one-time token의 의미입니다.

## 7. CI 해시와 개인정보

### 예상 질문

`CI + salt를 SHA-256한 값은 안전한 익명정보인가요?`

### 모범 답변

> 익명정보라기보다 결정적 가명 식별자입니다. 같은 CI를 같은 값으로 연결해야 해서 linkability가 남고, secret salt와 원본 시스템이 있으면 다시 개인과 연결될 수 있습니다. 따라서 여전히 개인정보로 취급하고 접근 통제, 보존 기간, 삭제, audit을 적용해야 합니다. 여러 환경이나 파트너 사이의 상관관계를 줄이려면 목적별 secret key를 사용한 HMAC 기반 pseudonymous ID도 고려할 수 있습니다. raw CI가 실제로 저장되지 않았다는 점과, 어떤 업무에서 복호화 또는 조회가 가능한지는 별도로 설명해야 합니다.

### Salt와 pepper의 구분

- user별 공개 salt는 password hash의 rainbow table을 어렵게 한다.
- 시스템 공통 secret을 붙였다면 성격상 pepper에 가깝고 Secrets Manager 보호와 rotation 문제가 생긴다.
- deterministic ID의 key를 rotation하면 기존 lookup이 깨질 수 있어 versioned key와 migration 계획이 필요하다.

## 8. ULID

### 예상 질문

`ULID가 보안 식별자로 좋은 이유는 무엇인가요?`

### 모범 답변

> 정렬 가능성과 분산 생성 편의가 주목적입니다. random component 때문에 단순 순차 정수보다 추측은 어렵지만 timestamp가 포함돼 생성 시점이 드러나고, 식별자의 불투명성이 authorization을 대신하지는 않습니다. 모든 object access는 소유권 검증을 해야 합니다. 높은 쓰기량에서 시간순 key가 partition hot spot을 만드는지도 저장소별로 확인해야 합니다.

## 9. Secrets Manager와 KMS

### 예상 질문

`KMS로 암호화했다고 했는데 실제로 어떤 방식인가요?`

### 모범 답변

> 먼저 서비스 관리형 저장 암호화와 애플리케이션 필드 암호화를 구분하겠습니다. DynamoDB의 AWS managed key 암호화는 disk와 backup의 at-rest protection을 제공하지만, 해당 table을 읽을 IAM 권한이 있는 application에는 평문이 반환됩니다. 더 강한 분리가 필요하면 customer managed key 정책과 application-level envelope encryption을 고려합니다. 외부 API secret은 Secrets Manager에 두고 Lambda execution role별로 필요한 secret ARN과 KMS decrypt 권한만 허용합니다. secret 값은 invocation마다 원격 조회하지 않고 짧게 cache하되 rotation과 invalidation을 고려하고 절대 로그에 남기지 않습니다.

### Rotation 깊이 질문

- DB password rotation은 RDS Proxy와 client connection behavior까지 검증해야 한다.
- webhook HMAC key는 발신자와 수신자가 동시에 바뀌지 않으므로 old/new key 동시 검증 기간과 key ID가 필요하다.
- CI deterministic key는 단순 rotation이 ID 변경을 일으킨다.
- compromised key rotation과 정기 rotation은 절차가 다르다.
- secret을 환경 변수로 주입하지 않았다는 표현은 실제 runtime load 방식을 확인한 뒤 말한다.

## 10. 로깅과 마스킹

### 예상 질문

`DEBUG=false일 때 body를 마스킹하면 충분한가요?`

### 모범 답변

> 운영 플래그에 의존한 전체 body logging은 설정 실수 한 번으로 유출될 수 있습니다. 기본은 request body와 authorization header를 기록하지 않고, 필요한 field만 allowlist로 구조화해 남기는 방식이 안전합니다. customer ID도 가능하면 내부 correlation ID로 대체하고, error object나 third-party response가 PII를 포함하는지 확인합니다. 로그 접근 권한, 보존 기간, encryption, export 경로와 감사 기록도 함께 관리해야 합니다.

### 관측성과 개인정보의 균형

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

## 11. 보안 문서에서 검증이 필요한 주장

| 문서 표현 | 면접 답변 |
|---|---|
| 전 구간 TLS 1.3 강제 | 실제 API Gateway/ALB security policy 확인 전 단정 금지. 최소 TLS version과 cipher policy로 설명 |
| 내부 서비스 VPC 완전 은닉 | Lambda의 VPC 연결, API endpoint type, resource policy와 route를 실제 IaC에서 확인 |
| HMAC이 CSRF와 replay 차단 | 무결성 외에 state/nonce/expiry/one-time consume이 있었는지 확인 |
| 임시 transaction 즉시 영구 삭제 | DynamoDB delete 후 backup/PITR와 로그 사본까지 포함한 법적 삭제 의미는 아님 |
| PBKDF2 100,000회가 현재 OWASP 권고 충족 | 현재 권고와 비용은 바뀔 수 있음. 측정 후 상향 또는 Argon2id 검토 |
| AWS managed KMS | AWS owned key, AWS managed key, customer managed key를 정확히 구분 |
| RDS Proxy가 DB 직접 연결 차단 | network SG와 IAM 정책이 차단한다. Proxy 사용 자체가 직접 연결을 자동 차단하지 않음 |

## 12. 보안 사고 시나리오

### `웹훅 secret이 유출됐다면?`

> key ID로 영향 범위를 식별하고 해당 key를 폐기 또는 rotate합니다. old/new 병행 기간을 최소화하고, 유출 구간의 callback을 event ID와 provider 조회로 재검증합니다. secret access audit와 로그 유출 여부를 조사하고, replay된 상태 전이가 version 조건과 idempotency에서 차단됐는지 확인합니다.

### `주민번호가 로그에 남았다면?`

> 추가 기록을 즉시 막고 접근 범위를 제한한 뒤, 어느 log group과 export sink, backup에 복제됐는지 확인합니다. 보존 정책에 따라 삭제하고 접근 기록과 영향 대상, 노출 기간을 파악합니다. 법무와 개인정보 담당 절차에 따라 신고와 통지를 판단합니다. 이후 schema-aware redaction, logging allowlist, CI test로 재발을 막습니다.

### 공식 참고 자료

- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [AWS Secrets Manager rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [AWS DynamoDB encryption at rest](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/EncryptionAtRest.html)

