# UCAN 프로토콜 심층 분석

> **문서 버전**: 1.0
> **작성일**: 2025-11-14
> **관련 문서**: 00_Storacha_Ecosystem_Overview.md, 10_Security_and_Authorization.md

## 목차
- [UCAN이란?](#ucan이란)
- [UCAN의 필요성](#ucan의-필요성)
- [핵심 개념](#핵심-개념)
- [Capability-based 인증](#capability-based-인증)
- [UCAN 토큰 구조](#ucan-토큰-구조)
- [Delegation 메커니즘](#delegation-메커니즘)
- [DID (Decentralized Identity)](#did-decentralized-identity)
- [ucanto 구현](#ucanto-구현)
- [실제 사용 예시](#실제-사용-예시)
- [보안 고려사항](#보안-고려사항)

---

## UCAN이란?

### 정의

**UCAN (User Controlled Authorization Networks)**은 분산형 권한 부여 시스템으로, 사용자가 자신의 데이터와 권한을 완전히 제어할 수 있게 하는 프로토콜입니다.

UCAN은:
- **JWT 기반**: JSON Web Token 형식을 확장
- **공개키 암호화**: 중앙 서버 없이 검증 가능
- **Capability-based**: 특정 작업을 수행할 수 있는 "능력" 표현
- **위임 가능**: 권한을 다른 사용자/서비스에 안전하게 전달
- **취소 가능**: 부여한 권한을 나중에 철회

### 기존 인증 시스템과의 차이

#### 전통적인 OAuth 2.0
```
사용자 → 중앙 서버 → 권한 확인 → 승인/거부
         ↓
    세션 저장
    권한 관리
    토큰 발급
```

**문제점**:
- 중앙 서버 의존성
- 단일 실패 지점 (Single Point of Failure)
- 사용자가 권한을 직접 제어하지 못함
- 오프라인 권한 부여 불가능

#### UCAN 방식
```
사용자 (키 소유) → UCAN 토큰 생성 → 서명 → 전달
                                        ↓
                              누구나 공개키로 검증 가능
                              중앙 서버 불필요
```

**장점**:
- ✅ 분산화: 중앙 서버 불필요
- ✅ 사용자 제어: 사용자가 직접 권한 관리
- ✅ 오프라인 작동: 인터넷 없이도 권한 부여 가능
- ✅ 프라이버시: 최소한의 정보만 공유
- ✅ 상호운용성: 여러 서비스 간 권한 공유

---

## UCAN의 필요성

### Web3 환경의 과제

Web3 애플리케이션은 다음과 같은 고유한 과제를 가집니다:

1. **탈중앙화 요구사항**
   - 중앙 권한 서버가 없어야 함
   - 사용자가 자신의 데이터를 소유해야 함

2. **상호운용성**
   - 여러 DApp과 서비스 간 권한 공유
   - 표준화된 권한 형식 필요

3. **확장성**
   - 수백만 사용자와 수십억 권한 관리
   - 서버 부하 최소화

4. **보안과 프라이버시**
   - 개인키는 사용자만 관리
   - 최소 권한 원칙 (Principle of Least Privilege)

### Storacha에서 UCAN이 해결하는 문제

#### 문제 1: Space에 대한 권한 관리

**시나리오**: 사용자 Alice가 자신의 Space를 팀원 Bob과 공유하고 싶음

**전통적 방식의 문제**:
```javascript
// ❌ 중앙 서버가 필요
await server.grantAccess({
  space: 'alice-space',
  user: 'bob@example.com',
  permissions: ['read', 'write']
})
// 서버 다운 시 Bob은 접근 불가
// Alice가 서버를 신뢰해야 함
```

**UCAN 방식**:
```javascript
// ✅ Alice가 직접 권한 부여
const delegation = await alice.createDelegation({
  audience: bob.did(), // Bob의 DID
  capabilities: [{
    can: 'space/blob/add',
    with: aliceSpace.did()
  }],
  expiration: Date.now() + 30 * 24 * 60 * 60 * 1000 // 30일
})

// Bob에게 delegation 전달 (이메일, QR 코드 등)
await sendToB(delegation.archive())

// Bob이 사용 시 서버는 단순히 서명 검증만 수행
// 중앙 데이터베이스 조회 불필요!
```

#### 문제 2: 다단계 권한 위임

**시나리오**: Alice → Bob → Charlie 순서로 권한 위임

**전통적 방식**:
- 복잡한 ACL (Access Control List) 관리
- 데이터베이스에 모든 관계 저장
- 권한 체인 추적 어려움

**UCAN 방식**:
```javascript
// Alice가 Bob에게
const aliceToBob = await alice.delegate({
  to: bob.did(),
  capabilities: ['space/blob/add']
})

// Bob이 Charlie에게 재위임
const bobToCharlie = await bob.delegate({
  to: charlie.did(),
  capabilities: ['space/blob/add'],
  proofs: [aliceToBob] // Alice의 delegation을 증명으로 첨부
})

// Charlie가 작업 수행 시
await charlie.invoke('space/blob/add', {
  space: aliceSpace.did(),
  blob: myData
}, {
  proofs: [bobToCharlie, aliceToBob] // 전체 체인 제공
})

// 서버는 체인 검증:
// 1. Charlie의 서명 확인
// 2. Bob의 위임 확인
// 3. Alice의 위임 확인
// 4. Alice가 Space 소유자인지 확인
// → 모두 공개키 암호화로 검증, DB 불필요!
```

---

## 핵심 개념

UCAN을 이해하기 위한 핵심 개념들입니다.

### 1. Principal (주체)

**Principal**은 UCAN에서 권한의 소유자 또는 대상을 나타냅니다.

#### DID로 식별
```
did:key:z6MkwHhAdxP...  ← Alice의 DID
did:key:z6MkrD5YcZ4...  ← Bob의 DID
did:key:z6Mkf5rGMqR...  ← Space의 DID
```

#### 종류
- **Issuer (발급자)**: UCAN을 생성하고 서명하는 주체
- **Audience (수신자)**: UCAN을 받는 주체
- **Subject (대상)**: 권한이 적용되는 리소스

### 2. Capability (능력)

**Capability**는 특정 리소스에 대해 특정 작업을 수행할 수 있는 권한입니다.

#### 구조
```typescript
interface Capability {
  can: string      // 작업 (예: 'space/blob/add')
  with: string     // 리소스 (예: 'did:key:z6Mkf...')
  nb?: object      // 추가 제약사항 (caveats)
}
```

#### Storacha의 Capabilities
```javascript
// Blob 저장 권한
{
  can: 'space/blob/add',
  with: 'did:key:z6Mkf5rGMqR...',  // Space DID
}

// Upload 등록 권한
{
  can: 'upload/add',
  with: 'did:key:z6Mkf5rGMqR...',
}

// Index 추가 권한
{
  can: 'index/add',
  with: 'did:key:z6Mkf5rGMqR...',
}

// Filecoin 제공 권한
{
  can: 'filecoin/offer',
  with: 'did:key:z6Mkf5rGMqR...',
}

// 크기 제한이 있는 Blob 저장
{
  can: 'space/blob/add',
  with: 'did:key:z6Mkf5rGMqR...',
  nb: {
    size: { max: 1024 * 1024 * 100 } // 최대 100MB
  }
}
```

### 3. Proof (증명)

**Proof**는 권한을 가지고 있음을 증명하는 UCAN 토큰입니다.

#### Delegation Chain
```
UCAN 1: Alice → Bob     [space/blob/add]
UCAN 2: Bob → Charlie   [space/blob/add]
         ↓
Charlie가 작업 시 UCAN 1 + UCAN 2를 proofs로 제공
```

#### 예시
```javascript
// Charlie가 Space에 blob 추가
const result = await charlie.capability.invoke({
  issuer: charlie,
  audience: service,
  capability: {
    can: 'space/blob/add',
    with: aliceSpace.did()
  },
  proofs: [
    aliceToBob,      // UCAN: Alice → Bob
    bobToCharlie     // UCAN: Bob → Charlie
  ]
})
```

### 4. Attenuation (감쇠)

**Attenuation**은 권한을 위임할 때 범위를 축소하는 것입니다.

#### 규칙
- 더 많은 권한은 부여할 수 없음
- 같거나 적은 권한만 부여 가능
- 더 긴 유효기간은 불가능

#### 예시
```javascript
// Alice가 가진 권한
const aliceCapabilities = [
  { can: 'space/blob/add', with: space.did() },
  { can: 'space/blob/remove', with: space.did() },
  { can: 'upload/add', with: space.did() }
]

// Bob에게 일부만 위임 (✅ 허용)
const delegation = await alice.delegate({
  audience: bob.did(),
  capabilities: [
    { can: 'space/blob/add', with: space.did() }  // 일부만
  ],
  expiration: Date.now() + 7 * 24 * 60 * 60 * 1000  // 7일
})

// Bob이 Charlie에게 더 많은 권한 위임 시도 (❌ 거부됨)
const invalid = await bob.delegate({
  audience: charlie.did(),
  capabilities: [
    { can: 'space/blob/remove', with: space.did() }  // Bob이 없는 권한!
  ],
  proofs: [delegation]
})
// → 검증 실패: Bob은 'space/blob/remove' 권한이 없음
```

### 5. Expiration (만료)

모든 UCAN은 만료 시간을 가집니다.

#### 타임스탬프
```javascript
const delegation = await issuer.delegate({
  audience: receiver.did(),
  capabilities: [...],
  expiration: Math.floor(Date.now() / 1000) + 3600  // Unix timestamp (초)
})
```

#### 검증
```javascript
function isExpired(ucan) {
  return ucan.expiration < Math.floor(Date.now() / 1000)
}
```

#### 만료된 UCAN 처리
```
요청 → 서버
       ↓
   UCAN 검증
       ↓
   만료 확인 ← 현재 시간
       ↓
   만료됨? → 거부 (401 Unauthorized)
       ↓
   유효함 → 계속 진행
```

---

## Capability-based 인증

UCAN은 전통적인 ACL(Access Control List) 대신 **Capability-based** 접근 방식을 사용합니다.

### ACL vs Capability

#### ACL (전통적 방식)
```
서버에 저장된 테이블:
┌─────────┬─────────┬────────────┐
│ User    │ Space   │ Permissions│
├─────────┼─────────┼────────────┤
│ Alice   │ space-1 │ read,write │
│ Bob     │ space-1 │ read       │
│ Charlie │ space-2 │ write      │
└─────────┴─────────┴────────────┘

요청 처리:
1. 사용자 인증 (username/password)
2. DB 조회: 이 사용자가 이 리소스에 접근 가능?
3. 승인/거부
```

**문제점**:
- 중앙 DB 의존
- 확장성 문제
- 오프라인 불가
- 사용자가 권한 제어 못함

#### Capability (UCAN 방식)
```
사용자가 토큰을 소유:
┌──────────────────────────────────┐
│ UCAN Token                       │
├──────────────────────────────────┤
│ Issuer: did:key:alice            │
│ Audience: did:key:bob            │
│ Capabilities:                    │
│   - can: space/blob/add          │
│     with: did:key:space-1        │
│ Signature: [Alice's signature]   │
└──────────────────────────────────┘

요청 처리:
1. Bob이 UCAN 토큰 제시
2. 서명 검증 (공개키로)
3. 권한 확인 (토큰 내용 확인)
4. 만료 확인
5. 승인/거부

DB 조회 불필요!
```

### Object Capability Model

UCAN은 **Object Capability Model**을 따릅니다.

#### 핵심 원칙

**1. Unforgeable (위조 불가능)**
```javascript
// UCAN은 개인키로 서명됨
// 개인키 없이는 유효한 UCAN 생성 불가능
const ucan = await issuer.createDelegation({...})
// → 서명: issuer의 개인키로 생성
```

**2. Exclusive (배타적)**
```javascript
// UCAN의 audience는 특정 DID
// 다른 사람은 사용 불가
{
  audience: 'did:key:bob...',  // Bob만 사용 가능
  // ...
}
```

**3. Attenuated (감쇠됨)**
```javascript
// 위임 시 권한 축소만 가능
// Alice: [read, write, delete]
// → Bob: [read, write]  ✅
// → Bob: [read, write, delete, admin]  ❌
```

### Capability Invocation (호출)

실제로 권한을 행사하는 과정입니다.

#### 구조
```typescript
interface Invocation {
  issuer: Principal      // 호출하는 주체
  audience: Principal    // 서비스 DID
  capability: Capability // 수행할 작업
  proofs: UCAN[]        // 권한 증명
  signature: Signature   // 호출자의 서명
}
```

#### 예시
```javascript
// Bob이 Alice의 Space에 blob 추가
const invocation = await bob.invoke({
  audience: service.did(),
  capability: {
    can: 'space/blob/add',
    with: aliceSpace.did(),
  },
  proofs: [aliceToBob]  // Alice → Bob delegation
})

/*
서버 검증 프로세스:
1. invocation의 서명 확인 (Bob이 진짜 서명했나?)
2. proofs 체인 검증:
   - aliceToBob 토큰의 서명 확인 (Alice가 진짜 서명했나?)
   - aliceToBob의 audience가 Bob인가?
   - aliceToBob이 'space/blob/add' 권한을 포함하나?
   - aliceToBob이 만료되지 않았나?
3. Alice가 Space의 소유자인가?
4. 모든 검증 통과 → 승인
*/
```

---

## UCAN 토큰 구조

UCAN 토큰은 **JWT (JSON Web Token)**를 확장한 형식입니다.

### JWT 기본 구조

```
header.payload.signature

예시:
eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJkaWQ6a2V5Ono2TWt3SGhBZHhQIiwiYXVkIjoiZGlkOmtleTp6Nk1rckQ1WWNaNCIsImNhcCI6eyJzcGFjZS9ibG9iL2FkZCI6eyJ3aXRoIjoiZGlkOmtleTp6Nk1rZjVyR01xUiJ9fSwiZXhwIjoxNzM1NzUyMDAwfQ.signature
```

### Header (헤더)

```json
{
  "alg": "EdDSA",        // 서명 알고리즘
  "typ": "JWT",          // 토큰 타입
  "ucv": "1.0.0"         // UCAN 버전
}
```

**지원하는 알고리즘**:
- **EdDSA**: Ed25519 (권장)
- **ES256K**: secp256k1 (Ethereum 호환)
- **RS256**: RSA

### Payload (페이로드)

```json
{
  "iss": "did:key:z6MkwHhAdxP...",  // Issuer (발급자)
  "aud": "did:key:z6MkrD5YcZ4...",  // Audience (수신자)
  "exp": 1735752000,                 // Expiration (만료 시간)
  "nbf": 1704216000,                 // Not Before (활성화 시간)
  "att": [                           // Attenuations (권한들)
    {
      "can": "space/blob/add",
      "with": "did:key:z6Mkf5rGMqR..."
    }
  ],
  "prf": [                           // Proofs (증명 UCAN들)
    "bafyreib..."                    // CID of parent UCAN
  ],
  "fct": [                           // Facts (사실 정보)
    {
      "space": "my-project",
      "size": 1024
    }
  ]
}
```

#### 필드 설명

**iss (Issuer)**
- UCAN을 생성하고 서명한 주체
- DID 형식
- 개인키를 가지고 있어야 함

**aud (Audience)**
- UCAN을 받는 대상
- DID 형식
- 이 DID만 UCAN 사용 가능

**exp (Expiration)**
- Unix timestamp (초 단위)
- 이 시간 이후에는 UCAN 무효

**nbf (Not Before, 선택적)**
- Unix timestamp (초 단위)
- 이 시간 이전에는 UCAN 사용 불가
- 미래의 권한 예약 시 사용

**att (Attenuations)**
- 부여하는 권한(capabilities) 배열
- 각 권한은 `can`과 `with` 필드 포함

**prf (Proofs)**
- 부모 UCAN들의 CID 배열
- Delegation chain 구성

**fct (Facts, 선택적)**
- 추가 메타데이터
- 검증에는 사용되지 않음

### Signature (서명)

```javascript
// 서명 생성
const message = base64url(header) + '.' + base64url(payload)
const signature = await sign(message, issuerPrivateKey)

// 최종 UCAN
const ucan = message + '.' + base64url(signature)
```

#### 서명 검증
```javascript
async function verifyUCAN(ucan) {
  const [headerB64, payloadB64, signatureB64] = ucan.split('.')

  const header = JSON.parse(base64url.decode(headerB64))
  const payload = JSON.parse(base64url.decode(payloadB64))
  const signature = base64url.decode(signatureB64)

  // 1. Issuer의 공개키 추출
  const publicKey = extractPublicKey(payload.iss)

  // 2. 서명 검증
  const message = headerB64 + '.' + payloadB64
  const valid = await verify(message, signature, publicKey)

  if (!valid) {
    throw new Error('Invalid signature')
  }

  // 3. 만료 확인
  if (payload.exp < Math.floor(Date.now() / 1000)) {
    throw new Error('UCAN expired')
  }

  // 4. Proofs 재귀 검증
  for (const proofCID of payload.prf || []) {
    const proofUCAN = await fetchUCAN(proofCID)
    await verifyUCAN(proofUCAN)
  }

  return payload
}
```

---

## 4. Delegation 메커니즘 상세

UCAN의 가장 강력한 기능 중 하나는 **Delegation (위임)**입니다. 위임을 통해 권한 소유자는 자신의 권한 일부 또는 전부를 다른 주체에게 전달할 수 있습니다.

### 4.1 Delegation의 동작 원리

Delegation은 새로운 UCAN 토큰을 생성하는 과정입니다:

```javascript
// Alice가 Bob에게 권한 위임
const delegation = await Client.delegate({
  issuer: alice,                    // 위임자 (권한을 주는 사람)
  audience: bob.principal,          // 수신자 (권한을 받는 사람)
  capabilities: [                   // 위임할 권한들
    {
      can: 'space/blob/add',       // 할 수 있는 작업
      with: `did:key:z6Mk...`,     // 대상 리소스
    },
  ],
  expiration: Date.now() + 86400,  // 만료 시간 (24시간 후)
  proofs: [parentDelegation]       // 부모 권한 증명
})
```

#### 위임의 핵심 요소:

1. **Issuer (발급자)**: 권한을 위임하는 주체
   - 자신이 소유한 권한이거나
   - 이전에 위임받은 권한이어야 함

2. **Audience (수신자)**: 권한을 받는 주체
   - DID로 식별됨
   - 해당 DID의 개인키를 가진 자만 사용 가능

3. **Capabilities**: 위임되는 구체적인 권한들
   - 원본 권한보다 **약화**되거나 **같아야** 함 (Attenuation)
   - 절대 강화될 수 없음

4. **Proofs**: 발급자가 해당 권한을 위임할 수 있다는 증명
   - 부모 UCAN 토큰들의 CID 배열
   - 재귀적으로 검증됨

### 4.2 Delegation Chain (위임 체인)

여러 단계의 위임이 연결되어 **Chain**을 형성합니다:

```
[Root Authority]
       ↓ delegates
   [Alice] did:key:z6Mk...
       ↓ delegates (subset)
   [Bob] did:key:z6Mk...
       ↓ delegates (subset)
   [Charlie] did:key:z6Mk...
```

각 단계에서 권한은 **점점 약해집니다** (Attenuation):

```javascript
// 1단계: Root가 Alice에게 모든 권한 위임
{
  iss: "did:key:zRoot...",
  aud: "did:key:zAlice...",
  att: [{
    can: "*",                           // 모든 작업
    with: "storage:alice-space"         // Alice의 Space 전체
  }]
}

// 2단계: Alice가 Bob에게 읽기/쓰기만 위임
{
  iss: "did:key:zAlice...",
  aud: "did:key:zBob...",
  att: [{
    can: "space/blob/add",             // 추가만 가능
    with: "storage:alice-space"
  }],
  prf: ["bafyAliceProof"]              // 1단계 UCAN 참조
}

// 3단계: Bob이 Charlie에게 더 제한된 권한 위임
{
  iss: "did:key:zBob...",
  aud: "did:key:zCharlie...",
  att: [{
    can: "space/blob/add",
    with: "storage:alice-space/subfolder",  // 더 제한된 경로
    nb: { size: 1048576 }                    // 추가 제약: 1MB 이하만
  }],
  prf: ["bafyBobProof"]                // 2단계 UCAN 참조
}
```

### 4.3 ucanto의 `delegate()` 구현

**`ucanto/packages/core/src/delegation.js`**의 핵심 함수:

```javascript
/**
 * 서명된 UCAN 토큰 생성
 * @param {Object} options
 * @param {Signer} options.issuer - 발급자 (서명자)
 * @param {Principal} options.audience - 수신자
 * @param {Capability[]} options.capabilities - 위임할 권한들
 * @param {Delegation[]} options.proofs - 증명 체인 (기본값: [])
 * @param {Block[]} options.attachedBlocks - 함께 저장할 블록들
 * @returns {Delegation}
 */
export async function delegate(options, encodeOptions) {
  // 1. proofs를 CID Link로 변환
  const proofLinks = convertProofsToLinks(options.proofs || [])

  // 2. UCAN 데이터 작성 (JWT 형식)
  const ucanBlock = await writeUCANData({
    issuer: options.issuer,
    audience: options.audience,
    capabilities: options.capabilities,
    proofs: proofLinks,
    expiration: options.expiration,
    notBefore: options.notBefore,
  })

  // 3. Delegation 인스턴스 반환 (블록 캐시 포함)
  return new Delegation({
    root: ucanBlock,
    blocks: collectBlocks(options.proofs, options.attachedBlocks),
    attachedLinks: new Set(proofLinks)
  })
}
```

### 4.4 Delegation의 특수 형태

#### 4.4.1 `ucan:*` - 재위임 권한

`with: "ucan:*"`는 특별한 의미를 가집니다:

```javascript
{
  can: "*",           // 모든 작업
  with: "ucan:*"      // 자신이 가진 모든 권한 재위임 가능
}
```

이 권한을 가진 주체는:
- 자신이 직접 소유한 리소스에 대한 권한 위임
- 이전에 위임받은 권한들을 다시 위임 (재위임)

**ucanto의 `iterateCapabilities()` 함수**가 이를 처리합니다:

```javascript
/**
 * Delegation에서 모든 Capability를 확장하여 yield
 * 특수 형태(ucan:*, can:*)를 구체적인 형태로 확장
 */
export function* iterateCapabilities(delegation) {
  for (const capability of delegation.capabilities) {
    // "ucan:*" 처리: 자신의 모든 권한 확장
    if (capability.with === 'ucan:*') {
      // 자신이 소유한 권한들 yield
      yield* expandOwnCapabilities(delegation.issuer)

      // 위임받은 권한들 yield
      for (const proof of delegation.proofs) {
        yield* iterateCapabilities(proof)  // 재귀적 확장
      }
    }
    // "can: *" 처리: 모든 작업 가능
    else if (capability.can === '*') {
      yield* expandAllAbilities(capability.with)
    }
    // 일반 Capability
    else {
      yield capability
    }
  }
}
```

#### 4.4.2 Wildcard Patterns

Path와 Ability에 와일드카드 사용 가능:

```javascript
// Path wildcard
{
  can: "space/blob/add",
  with: "storage:alice-space/*"     // 하위 모든 경로
}

// Ability wildcard
{
  can: "space/*",                   // space/blob/add, space/index/add 등
  with: "storage:alice-space"
}

// 완전한 wildcard
{
  can: "*",                         // 모든 작업
  with: "storage:*"                 // 모든 리소스
}
```

**`matchAbility()` 함수**가 패턴 매칭을 처리:

```javascript
/**
 * 위임된 Ability가 요청된 Ability와 매칭되는지 확인
 * @param {string} provided - 위임된 ability (예: "space/*")
 * @param {string} claimed - 요청된 ability (예: "space/blob/add")
 * @returns {string|null} - 더 구체적인 ability 또는 null
 */
export function matchAbility(provided, claimed) {
  // 1. 완전 일치
  if (provided === claimed) {
    return claimed
  }

  // 2. 완전한 wildcard
  if (provided === '*') {
    return claimed
  }

  // 3. Path pattern (예: "space/*")
  if (provided.endsWith('/*')) {
    const prefix = provided.slice(0, -2)
    if (claimed.startsWith(prefix + '/')) {
      return claimed
    }
  }

  // 4. 매칭 실패
  return null
}
```

### 4.5 Delegation의 저장과 전송

Delegation은 **CAR (Content Addressable aRchive)** 형식으로 인코딩됩니다:

```javascript
/**
 * Delegation 체인을 CAR 버퍼로 인코딩
 * @param {Delegation} delegation
 * @returns {Uint8Array} CAR 형식의 바이너리
 */
export async function archive(delegation) {
  const blocks = []

  // 1. 모든 블록 수집 (root + proofs + attached)
  for await (const block of delegation.iterate()) {
    blocks.push(block)
  }

  // 2. CAR variant descriptor 생성
  const descriptor = {
    version: 1,
    roots: [delegation.root.cid],
    blocks: blocks
  }

  // 3. CAR로 인코딩
  return encodeCAR(descriptor)
}

/**
 * CAR 버퍼에서 Delegation 복원
 * @param {Uint8Array} archive - CAR 바이너리
 * @returns {Delegation}
 */
export async function extract(archive) {
  // 1. CAR 파싱
  const { roots, blocks } = await parseCAR(archive)

  // 2. Root 블록 찾기
  const rootCID = roots[0]
  const rootBlock = blocks.find(b => b.cid.equals(rootCID))

  if (!rootBlock) {
    throw new Error('Root block not found in CAR')
  }

  // 3. Delegation 인스턴스 재구성
  return new Delegation({
    root: rootBlock,
    blocks: new Map(blocks.map(b => [b.cid.toString(), b])),
    attachedLinks: extractProofLinks(rootBlock)
  })
}
```

이렇게 인코딩된 Delegation은:
- 네트워크를 통해 전송 가능
- IPFS/Filecoin에 저장 가능
- 다른 주체에게 공유 가능

---

## 5. DID (Decentralized Identity) 구조

UCAN에서 **Principal (주체)**는 **DID (Decentralized Identifier)**로 식별됩니다.

### 5.1 `did:key` 메서드

Storacha와 ucanto는 **`did:key`** 메서드를 주로 사용합니다:

```
did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK
│      │ └────────────────────────────────────────────────┘
│      │                  Multibase 인코딩된 공개키
│      └── Method name
└── Scheme
```

### 5.2 `did:key` 인코딩 구조

`did:key`는 공개키를 직접 DID로 인코딩합니다 (별도의 레지스트리 불필요):

```
did:key:<MULTIBASE(base58-btc, MULTICODEC(key-type, raw-public-key))>
```

#### 인코딩 단계:

1. **Raw Public Key Bytes 추출**
   ```javascript
   // Ed25519 공개키 (32 bytes)
   const publicKey = new Uint8Array([
     0x3b, 0x6a, 0x27, 0xbc, 0xce, 0xb6, 0xa4, 0x2d,
     0x62, 0xa3, 0xa8, 0xd0, 0x2a, 0x6f, 0x0d, 0x73,
     // ... 32 bytes total
   ])
   ```

2. **Multicodec Prefix 추가**
   ```javascript
   // Ed25519 public key의 multicodec = 0xed (varint: [0xed, 0x01])
   const multicodecValue = new Uint8Array([0xed, 0x01, ...publicKey])
   ```

3. **Multibase 인코딩 (base58-btc)**
   ```javascript
   // 'z'는 base58-btc를 나타냄
   const multibaseValue = 'z' + base58.encode(multicodecValue)
   // 결과: "z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
   ```

4. **최종 DID 생성**
   ```javascript
   const did = `did:key:${multibaseValue}`
   // 결과: "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
   ```

### 5.3 ucanto의 Ed25519 DID 생성

**`ucanto/packages/principal/src/ed25519/signer.js`**:

```javascript
import * as ed25519 from '@noble/ed25519'
import { base58btc } from 'multiformats/bases/base58'
import { varint } from 'multiformats'

/**
 * Ed25519 서명자 생성 (DID 포함)
 */
export class EdSigner {
  constructor(privateKey) {
    this.privateKey = privateKey
    this.publicKey = ed25519.getPublicKey(privateKey)
  }

  /**
   * DID 문자열 반환
   * @returns {string} "did:key:z6Mk..."
   */
  did() {
    return toDID(this.publicKey)
  }

  /**
   * Principal 객체 반환
   */
  get principal() {
    return { did: () => this.did() }
  }

  /**
   * 데이터 서명
   * @param {Uint8Array} data
   * @returns {Promise<Uint8Array>} 서명 (64 bytes)
   */
  async sign(data) {
    return await ed25519.sign(data, this.privateKey)
  }

  /**
   * 새 서명자 생성 (랜덤 키)
   */
  static generate() {
    const privateKey = ed25519.utils.randomPrivateKey()
    return new EdSigner(privateKey)
  }

  /**
   * 시드에서 서명자 생성 (deterministic)
   */
  static fromSeed(seed) {
    // seed를 해시하여 32-byte private key 생성
    const privateKey = sha256(seed)
    return new EdSigner(privateKey)
  }
}

/**
 * 공개키를 did:key DID로 변환
 * @param {Uint8Array} publicKey - Ed25519 공개키 (32 bytes)
 * @returns {string} did:key DID
 */
function toDID(publicKey) {
  // Ed25519 multicodec prefix: 0xed (varint)
  const ED25519_MULTICODEC = 0xed

  // 1. Multicodec 인코딩
  const multicodecBytes = varint.encode(ED25519_MULTICODEC)
  const multicodecKey = new Uint8Array([
    ...multicodecBytes,
    ...publicKey
  ])

  // 2. Multibase base58-btc 인코딩
  const multibaseKey = base58btc.encode(multicodecKey)

  // 3. DID 조립
  return `did:key:${multibaseKey}`
}
```

**`ucanto/packages/principal/src/ed25519/verifier.js`**:

```javascript
/**
 * Ed25519 검증자 (DID에서 복원)
 */
export class EdVerifier {
  constructor(publicKey) {
    this.publicKey = publicKey
  }

  /**
   * DID 문자열 반환
   */
  did() {
    return toDID(this.publicKey)
  }

  /**
   * 서명 검증
   * @param {Uint8Array} data - 원본 데이터
   * @param {Uint8Array} signature - 서명 (64 bytes)
   * @returns {Promise<boolean>}
   */
  async verify(data, signature) {
    return await ed25519.verify(signature, data, this.publicKey)
  }

  /**
   * DID에서 검증자 생성
   * @param {string} did - "did:key:z6Mk..."
   * @returns {EdVerifier}
   */
  static fromDID(did) {
    const publicKey = parsePublicKey(did)
    return new EdVerifier(publicKey)
  }
}

/**
 * did:key DID를 공개키로 파싱
 * @param {string} did
 * @returns {Uint8Array} 공개키 (32 bytes)
 */
function parsePublicKey(did) {
  // 1. "did:key:" prefix 제거
  if (!did.startsWith('did:key:')) {
    throw new Error('Invalid DID format')
  }
  const multibaseKey = did.slice(8)  // "z6Mk..." 부분

  // 2. Multibase 디코딩
  if (!multibaseKey.startsWith('z')) {
    throw new Error('Expected base58-btc encoding (z prefix)')
  }
  const multicodecKey = base58btc.decode(multibaseKey)

  // 3. Multicodec 파싱
  const [codec, keyBytes] = varint.decode(multicodecKey)
  if (codec !== 0xed) {
    throw new Error(`Expected Ed25519 key (0xed), got ${codec}`)
  }

  // 4. 공개키 추출
  return keyBytes  // 32 bytes
}
```

### 5.4 DID를 사용한 완전한 예시

```javascript
import { EdSigner } from '@ucanto/principal/ed25519'

// 1. Alice의 identity 생성
const alice = EdSigner.generate()
console.log(alice.did())
// "did:key:z6Mkf4nkGJMVdCeKzguWrW3hg4uyLpFqF5vbHCPCQNJsQpqj"

// 2. Bob의 identity 생성
const bob = EdSigner.generate()
console.log(bob.did())
// "did:key:z6MkpwFqTg3UHQdLaL7vsLT4fKwTCqPpbZ9rmTZhZz8QNgCL"

// 3. Alice가 데이터 서명
const message = new TextEncoder().encode("Hello, UCAN!")
const signature = await alice.sign(message)

// 4. Bob이 Alice의 서명 검증
const aliceVerifier = EdVerifier.fromDID(alice.did())
const valid = await aliceVerifier.verify(message, signature)
console.log(valid)  // true

// 5. DID가 변조되면 검증 실패
const fakeVerifier = EdVerifier.fromDID("did:key:z6Mk...")
const invalid = await fakeVerifier.verify(message, signature)
console.log(invalid)  // false
```

### 5.5 DID의 장점

1. **자체 증명 (Self-Certifying)**
   - DID 자체가 공개키를 포함
   - 별도의 PKI (Public Key Infrastructure) 불필요

2. **탈중앙화 (Decentralized)**
   - 중앙 레지스트리 없음
   - 누구나 즉시 DID 생성 가능

3. **영구성 (Persistent)**
   - 공개키가 변하지 않는 한 DID도 불변
   - URL이나 도메인처럼 만료되지 않음

4. **검증 가능 (Verifiable)**
   - DID에서 공개키 추출 → 서명 검증
   - 중간자 공격 방지

---

## 6. ucanto RPC 프레임워크

ucanto는 UCAN 토큰을 기반으로 하는 **RPC (Remote Procedure Call)** 프레임워크입니다. HTTP나 WebSocket 등의 전송 계층 위에서 Capability-based 권한 검증을 제공합니다.

### 6.1 ucanto 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────┐
│                         Client                              │
│  ┌────────────┐       ┌───────────┐      ┌──────────────┐  │
│  │  Invoke    │  -->  │ Delegation│  --> │  Connection  │  │
│  │ (capability)│       │  (proofs) │      │  (transport) │  │
│  └────────────┘       └───────────┘      └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              ↓
                    ┌──────────────────┐
                    │  Network (HTTP)  │
                    └──────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                         Server                              │
│  ┌────────────┐       ┌───────────┐      ┌──────────────┐  │
│  │  Service   │  <--  │ Validation│  <-- │   Decoder    │  │
│  │ (handler)  │       │  (UCAN)   │      │  (invocations)│ │
│  └────────────┘       └───────────┘      └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Capability 정의

ucanto에서 모든 작업은 **Capability**로 정의됩니다:

**`@web3-storage/capabilities` 패키지의 예시:**

```typescript
import { capability, URI, Link, Schema } from '@ucanto/validator'

/**
 * space/blob/add Capability 정의
 * - 특정 Space에 Blob 추가 권한
 */
export const add = capability({
  // 1. 작업 이름
  can: 'space/blob/add',

  // 2. 리소스 URI 검증 (Space DID)
  with: URI.match({ protocol: 'did:' }),

  // 3. 추가 제약사항 (caveats)
  nb: Schema.struct({
    blob: Schema.struct({
      digest: Schema.bytes(),      // Blob의 SHA-256 해시
      size: Schema.integer(),       // Blob 크기 (bytes)
    })
  }),

  // 4. Capability 포함 관계 검증 (derives)
  derives: (claimed, delegated) => {
    // 요청된 권한이 위임된 권한에 포함되는지 확인

    // 1. 리소스(Space) 일치 확인
    if (claimed.with !== delegated.with) {
      return new Failure(
        `Expected 'with: "${delegated.with}"' instead got '${claimed.with}'`
      )
    }

    // 2. Blob 크기 제약 확인
    if (delegated.nb?.maxSize &&
        claimed.nb.blob.size > delegated.nb.maxSize) {
      return new Failure(
        `Blob size ${claimed.nb.blob.size} exceeds limit ${delegated.nb.maxSize}`
      )
    }

    // 검증 성공
    return true
  }
})

/**
 * space/blob/remove Capability
 */
export const remove = capability({
  can: 'space/blob/remove',
  with: URI.match({ protocol: 'did:' }),
  nb: Schema.struct({
    digest: Schema.bytes()
  }),
  derives: (claimed, delegated) => {
    return claimed.with === delegated.with ||
           new Failure(`Space mismatch`)
  }
})

/**
 * space/* wildcard Capability
 */
export const spaceStar = capability({
  can: 'space/*',
  with: URI.match({ protocol: 'did:' }),
  derives: (claimed, delegated) => {
    // space/blob/add, space/blob/remove 등 모두 포함
    if (!claimed.can.startsWith('space/')) {
      return new Failure(`Capability must be space/*`)
    }
    return claimed.with === delegated.with
  }
})
```

### 6.3 Server - Service Handler 구현

**`ucanto/server`**를 사용한 서버 구현:

```typescript
import { provide } from '@ucanto/server'
import * as SpaceBlob from '@web3-storage/capabilities/space/blob'

/**
 * Service 컨텍스트 (DI)
 */
interface ServiceContext {
  blobStore: BlobStore           // Blob 저장소
  allowList: Set<string>         // 허용된 Space DID 목록
  maxBlobSize: number            // 최대 Blob 크기
}

/**
 * space/blob/add Handler
 */
const createBlobAddHandler = (context: ServiceContext) => {
  return provide(
    SpaceBlob.add,                // Capability 정의
    async ({ capability, invocation }) => {
      const { with: space, nb } = capability

      // 1. Space가 허용 목록에 있는지 확인
      if (!context.allowList.has(space)) {
        return {
          error: {
            name: 'UnauthorizedSpace',
            message: `Space ${space} is not authorized`
          }
        }
      }

      // 2. Blob 크기 제한 확인
      if (nb.blob.size > context.maxBlobSize) {
        return {
          error: {
            name: 'BlobTooLarge',
            message: `Blob size ${nb.blob.size} exceeds ${context.maxBlobSize}`
          }
        }
      }

      // 3. Blob 저장
      try {
        await context.blobStore.put({
          space,
          digest: nb.blob.digest,
          size: nb.blob.size
        })

        return {
          ok: {
            with: space,
            digest: nb.blob.digest,
            size: nb.blob.size
          }
        }
      } catch (error) {
        return {
          error: {
            name: 'StorageError',
            message: error.message
          }
        }
      }
    }
  )
}

/**
 * space/blob/remove Handler
 */
const createBlobRemoveHandler = (context: ServiceContext) => {
  return provide(
    SpaceBlob.remove,
    async ({ capability }) => {
      const { with: space, nb } = capability

      await context.blobStore.delete({
        space,
        digest: nb.digest
      })

      return {
        ok: { with: space, digest: nb.digest }
      }
    }
  )
}

/**
 * Service 조립
 */
export const createService = (context: ServiceContext) => {
  return {
    space: {
      blob: {
        add: createBlobAddHandler(context),
        remove: createBlobRemoveHandler(context)
      }
    }
  }
}
```

**Server 인스턴스 생성:**

```typescript
import { Server } from '@ucanto/server'
import { CAR, CBOR, HTTP } from '@ucanto/transport'

// 1. Service ID (서버의 DID)
const serverID = EdSigner.fromSeed(process.env.SERVER_SEED)

// 2. Service 생성
const service = createService({
  blobStore: new S3BlobStore(),
  allowList: new Set(['did:key:z6Mk...']),
  maxBlobSize: 100 * 1024 * 1024  // 100 MB
})

// 3. Server 생성
const server = Server.create({
  id: serverID,
  service,
  codec: {
    inbound: CAR,    // Invocation 디코딩
    outbound: CBOR   // Response 인코딩
  },
  // Capability 발급 가능 여부 확인
  canIssue: (capability, issuer) => {
    // serverID가 발급한 Delegation만 신뢰
    return issuer === serverID.did()
  }
})

// 4. HTTP 서버 연결
import express from 'express'
const app = express()

app.post('/rpc', async (req, res) => {
  const { headers, body } = await server.request({
    headers: req.headers,
    body: req.body
  })

  res.set(headers)
  res.send(body)
})

app.listen(3000)
```

### 6.4 Client - Invocation 실행

**단일 Invocation:**

```typescript
import { Client } from '@ucanto/client'
import { CAR, CBOR, HTTP } from '@ucanto/transport'
import * as SpaceBlob from '@web3-storage/capabilities/space/blob'

// 1. Agent (클라이언트 Identity)
const agent = EdSigner.generate()

// 2. Connection 설정
const connection = Client.connect({
  id: serverID,  // 서버 DID
  codec: {
    outbound: CAR,    // Invocation 인코딩
    inbound: CBOR     // Response 디코딩
  },
  channel: HTTP.open({
    url: new URL('https://api.storacha.network/rpc')
  })
})

// 3. Invocation 생성
const invocation = Client.invoke({
  issuer: agent,
  audience: serverID,
  capability: {
    can: 'space/blob/add',
    with: 'did:key:z6MkSpaceDID...',
    nb: {
      blob: {
        digest: new Uint8Array([...]),  // SHA-256
        size: 1024000
      }
    }
  },
  proofs: [delegation]  // 권한 증명
})

// 4. 실행
const result = await invocation.execute(connection)

if (result.out.ok) {
  console.log('Success:', result.out.ok)
} else {
  console.error('Error:', result.out.error)
}
```

**Batch Invocation (다중 실행):**

```typescript
// 여러 작업을 하나의 요청으로 묶기
const [addResult, removeResult] = await connection.execute([
  Client.invoke({
    issuer: agent,
    audience: serverID,
    capability: {
      can: 'space/blob/add',
      with: spaceDID,
      nb: { blob: { digest: digest1, size: size1 } }
    },
    proofs: [delegation]
  }),
  Client.invoke({
    issuer: agent,
    audience: serverID,
    capability: {
      can: 'space/blob/remove',
      with: spaceDID,
      nb: { digest: digest2 }
    },
    proofs: [delegation]
  })
])

// 각 결과 처리
addResult.out.ok    // { with: '...', digest: ..., size: ... }
removeResult.out.ok // { with: '...', digest: ... }
```

### 6.5 UCAN 검증 프로세스

Server에서 Invocation 수신 시 다음 단계로 검증합니다:

```typescript
/**
 * ucanto/server의 내부 검증 로직 (간소화)
 */
async function validateInvocation(invocation: Invocation) {
  const { capability, proofs, issuer, audience } = invocation

  // 1. Audience 확인 (서버 자신인가?)
  if (audience !== server.id.did()) {
    throw new Error('Invalid audience')
  }

  // 2. 서명 검증
  const valid = await verifySignature(invocation)
  if (!valid) {
    throw new Error('Invalid signature')
  }

  // 3. 만료 확인
  if (invocation.expiration < Date.now() / 1000) {
    throw new Error('Invocation expired')
  }

  // 4. Proofs 재귀 검증
  const validProofs = []
  for (const proof of proofs) {
    // 4-1. Proof 서명 검증
    const proofValid = await verifyDelegation(proof)
    if (!proofValid) {
      throw new Error('Invalid proof')
    }

    // 4-2. Proof 체인 검증 (issuer → audience 연결)
    if (proof.audience !== issuer) {
      throw new Error('Proof chain broken')
    }

    validProofs.push(proof)
  }

  // 5. Capability 포함 관계 검증
  const authorized = validProofs.some(proof => {
    return proof.capabilities.some(delegatedCap => {
      // derives 함수로 검증
      const result = capability.derives(capability, delegatedCap)
      return result === true
    })
  })

  if (!authorized) {
    throw new Error('Capability not delegated')
  }

  // 6. canIssue 확인 (서버 정책)
  if (!server.canIssue(capability, issuer)) {
    throw new Error('Issuer not trusted')
  }

  return true
}
```

### 6.6 Transport Layer

ucanto는 **pluggable transport**를 지원합니다:

#### 6.6.1 HTTP Transport

```typescript
import { HTTP } from '@ucanto/transport'

// Client 측
const channel = HTTP.open({
  url: new URL('https://api.storacha.network/rpc'),
  method: 'POST',
  headers: {
    'Authorization': 'Bearer ...'
  }
})

// Server 측
const handler = HTTP.server(server, {
  codec: { inbound: CAR, outbound: CBOR }
})

// Express/Next.js 등에서
app.post('/rpc', handler)
```

#### 6.6.2 WebSocket Transport (예시)

```typescript
import { WebSocket } from '@ucanto/transport'

const channel = WebSocket.open({
  url: new URL('wss://api.storacha.network/ws')
})

// 실시간 통신 지원
const subscription = await Client.invoke({
  capability: { can: 'space/blob/watch', with: spaceDID },
  proofs: [delegation]
}).execute(channel)

for await (const event of subscription) {
  console.log('Blob added:', event)
}
```

#### 6.6.3 Custom Transport

```typescript
import { Channel } from '@ucanto/interface'

class CustomChannel implements Channel {
  async request(request: { body: Uint8Array }) {
    // 커스텀 전송 로직 (gRPC, QUIC, etc.)
    const response = await myCustomProtocol.send(request.body)

    return {
      status: 200,
      body: response
    }
  }
}

const connection = Client.connect({
  id: serverID,
  codec: { outbound: CAR, inbound: CBOR },
  channel: new CustomChannel()
})
```

---

## 7. Storacha에서의 실제 사용

### 7.1 w3up-client를 사용한 파일 업로드

**완전한 예시:**

```typescript
import { create } from '@web3-storage/w3up-client'
import { filesFromPaths } from 'files-from-path'

// 1. Client 초기화
const client = await create()

// 2. Email 인증 (새 사용자)
const account = await client.login('alice@example.com')
await account.plan.set('free')  // 플랜 설정

// 3. Space 생성 (Storage 네임스페이스)
const space = await client.createSpace('my-project')
await client.setCurrentSpace(space.did())

// 4. Space 등록 (서버에)
await client.registerSpace(account)

// 5. 파일 업로드
const files = await filesFromPaths(['./dist'])
const directoryCID = await client.uploadDirectory(files)

console.log(`Uploaded to: https://${directoryCID}.ipfs.w3s.link`)
```

#### 내부 동작 분석:

```typescript
/**
 * client.uploadDirectory()의 내부 구조
 */
async function uploadDirectory(files: File[]) {
  // 1. UnixFS로 변환
  const { root, blocks } = await UnixFS.encodeDirectory(files)

  // 2. CAR 파일 생성
  const car = await CAR.encode({ roots: [root], blocks })

  // 3. space/blob/add Invocation
  const blobAddResult = await client.capability.blob.add(
    client.currentSpace(),  // Space DID
    {
      blob: {
        digest: await sha256(car),
        size: car.byteLength
      }
    }
  )

  if (blobAddResult.out.error) {
    throw new Error(blobAddResult.out.error.message)
  }

  // 4. space/index/add Invocation (색인 등록)
  const indexAddResult = await client.capability.index.add(
    client.currentSpace(),
    { index: root }
  )

  // 5. upload/add Invocation (논리적 업로드 완료)
  const uploadAddResult = await client.capability.upload.add(
    client.currentSpace(),
    {
      root,
      shards: [blobAddResult.out.ok.link]
    }
  )

  return root  // CID
}
```

### 7.2 Delegation을 통한 권한 공유

**Alice가 Bob에게 업로드 권한 위임:**

```typescript
// Alice의 client
const alice = await create()
await alice.login('alice@example.com')

const aliceSpace = await alice.createSpace('alice-space')
await alice.setCurrentSpace(aliceSpace.did())

// Bob의 DID 가져오기
const bobDID = 'did:key:z6MkBobPublicKey...'

// Delegation 생성
const delegation = await alice.createDelegation({
  audience: bobDID,
  capabilities: [
    {
      can: 'space/blob/add',
      with: aliceSpace.did()
    },
    {
      can: 'space/index/add',
      with: aliceSpace.did()
    },
    {
      can: 'upload/add',
      with: aliceSpace.did()
    }
  ],
  expiration: Math.floor(Date.now() / 1000) + 86400 * 30  // 30일
})

// Delegation을 CAR로 직렬화
const archive = await delegation.archive()

// Bob에게 전달 (이메일, 메시지 등)
await sendToEmail('bob@example.com', archive)
```

**Bob이 Delegation 사용:**

```typescript
// Bob의 client
const bob = await create()

// Delegation 복원
const delegation = await bob.addProof(archive)

// Alice의 Space에 파일 업로드
const files = await filesFromPaths(['./myfile.txt'])
const cid = await bob.uploadDirectory(files, {
  space: aliceSpace.did()  // Alice의 Space 사용
})

console.log('Uploaded to Alice\'s space:', cid)
```

### 7.3 서버 측 검증 (upload-service)

**`upload-service/blob/add.js` Handler:**

```typescript
import { provide } from '@ucanto/server'
import * as BlobCapabilities from '@web3-storage/capabilities/blob'
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'

/**
 * space/blob/add 핸들러 구현
 */
export const createBlobAddHandler = (context) => {
  return provide(
    BlobCapabilities.add,
    async ({ capability, invocation }) => {
      const { with: space, nb } = capability
      const { digest, size } = nb.blob

      // 1. Space가 존재하는지 확인
      const spaceExists = await context.db.spaces.has(space)
      if (!spaceExists) {
        return {
          error: {
            name: 'SpaceNotFound',
            message: `Space ${space} not found`
          }
        }
      }

      // 2. Storage quota 확인
      const usage = await context.db.spaces.getUsage(space)
      const quota = await context.db.spaces.getQuota(space)
      if (usage + size > quota) {
        return {
          error: {
            name: 'QuotaExceeded',
            message: `Space quota exceeded`
          }
        }
      }

      // 3. Blob 중복 확인
      const existing = await context.db.blobs.get({ space, digest })
      if (existing) {
        return {
          ok: {
            with: space,
            link: existing.link,
            size: existing.size,
            status: 'done'
          }
        }
      }

      // 4. Presigned URL 생성 (S3)
      const key = `${space}/${digestToString(digest)}`
      const putCommand = new PutObjectCommand({
        Bucket: context.bucket,
        Key: key,
        ContentLength: size
      })
      const presignedUrl = await getSignedUrl(context.s3, putCommand, {
        expiresIn: 3600
      })

      // 5. Blob 레코드 생성
      await context.db.blobs.create({
        space,
        digest,
        size,
        status: 'uploading'
      })

      // 6. Presigned URL 반환
      return {
        ok: {
          with: space,
          link: CID.create(1, 0x55, digest),  // raw CID
          size,
          url: presignedUrl,
          status: 'uploading'
        }
      }
    }
  )
}
```

### 7.4 Capability 체인의 실제 흐름

```
[Root Authority - Storacha Server]
         did:web:storacha.network
                  │
                  │ delegates all space/* capabilities
                  ↓
         [Account - Alice]
         did:mailto:alice@example.com
                  │
                  │ delegates space/blob/add, upload/add
                  ↓
         [Agent - Alice's Device]
         did:key:z6MkAliceAgent...
                  │
                  │ delegates space/blob/add with size limit
                  ↓
         [Agent - Bob]
         did:key:z6MkBob...
```

**검증 시 Proof Chain:**

```javascript
// Bob의 Invocation
{
  iss: "did:key:z6MkBob...",
  aud: "did:web:storacha.network",
  att: [{
    can: "space/blob/add",
    with: "did:key:z6MkAliceSpace...",
    nb: { blob: { digest: [...], size: 1024 } }
  }],
  prf: [
    "bafyProof1",  // Alice Agent → Bob delegation
    "bafyProof2",  // Alice Account → Alice Agent delegation
    "bafyProof3"   // Storacha → Alice Account delegation
  ]
}
```

서버는 역순으로 검증:
1. `bafyProof3`: Storacha → Alice Account (space/* 위임)
2. `bafyProof2`: Alice Account → Alice Agent (space/blob/add 위임)
3. `bafyProof1`: Alice Agent → Bob (space/blob/add, size≤1MB 위임)
4. Invocation: Bob → Server (size=1024, 조건 만족 ✓)

---

## 8. 보안 고려사항 및 Best Practices

### 8.1 최소 권한 원칙 (Principle of Least Authority)

UCAN 위임 시 **가능한 최소한의 권한**만 부여해야 합니다:

```typescript
// ❌ 나쁜 예: 모든 권한 위임
const badDelegation = await alice.createDelegation({
  audience: bob.did(),
  capabilities: [{
    can: '*',                    // 모든 작업
    with: aliceSpace.did()       // 모든 리소스
  }],
  expiration: Infinity           // 만료 없음
})

// ✅ 좋은 예: 최소 권한만 위임
const goodDelegation = await alice.createDelegation({
  audience: bob.did(),
  capabilities: [{
    can: 'space/blob/add',       // 특정 작업만
    with: aliceSpace.did(),
    nb: {
      maxSize: 10 * 1024 * 1024  // 10MB 제한
    }
  }],
  expiration: Math.floor(Date.now() / 1000) + 3600  // 1시간
})
```

#### 권한 최소화 전략:

1. **Ability 제한**
   ```typescript
   // space/* 대신 구체적인 ability
   { can: 'space/blob/add' }      // ✅
   { can: 'space/*' }             // ⚠️ 필요한 경우만
   { can: '*' }                   // ❌ 피하기
   ```

2. **Resource 제한**
   ```typescript
   // 전체 Space 대신 하위 경로
   { with: `${spaceDID}/uploads/temp` }  // ✅
   { with: spaceDID }                     // ⚠️
   ```

3. **Caveats (nb) 추가**
   ```typescript
   nb: {
     maxSize: 1048576,           // 크기 제한
     contentType: 'image/*',     // 타입 제한
     expires: Date.now() + 3600  // 추가 만료 시간
   }
   ```

### 8.2 짧은 만료 시간 (Short Expiration)

UCAN은 **Revocation (취소)**가 어렵기 때문에 짧은 만료 시간이 중요합니다:

```typescript
/**
 * 만료 시간 권장사항
 */
const EXPIRATION_TIMES = {
  // 일회성 작업 (단일 파일 업로드)
  oneTime: 5 * 60,              // 5분

  // 세션 기반 (브라우저 세션)
  session: 24 * 60 * 60,        // 24시간

  // 단기 협업 (임시 공유)
  shortTerm: 7 * 24 * 60 * 60,  // 7일

  // 장기 자동화 (CI/CD, 백업)
  longTerm: 90 * 24 * 60 * 60,  // 90일

  // 영구 위임은 피하기
  // permanent: Infinity         // ❌ 절대 사용 금지
}

// 사용 예시
const delegation = await alice.createDelegation({
  audience: cicdAgent.did(),
  capabilities: [{ can: 'space/blob/add', with: spaceDID }],
  expiration: Math.floor(Date.now() / 1000) + EXPIRATION_TIMES.longTerm
})
```

#### 만료 시간 vs. 사용 목적:

| 사용 목적 | 만료 시간 | 이유 |
|----------|----------|-----|
| 단일 API 호출 | 5-15분 | 네트워크 지연만 고려 |
| 웹 애플리케이션 세션 | 1-24시간 | 브라우저 세션 유지 |
| 모바일 앱 | 7-30일 | 백그라운드 작업 지원 |
| CI/CD 파이프라인 | 30-90일 | 정기적 갱신 필요 |
| 서버 간 통신 | 1시간 (자동 갱신) | 빠른 순환 |

### 8.3 Revocation 전략

UCAN은 **직접적인 Revocation이 불가능**하지만, 여러 우회 방법이 있습니다:

#### 8.3.1 Passive Revocation (만료 기반)

```typescript
/**
 * 갱신 가능한 단기 토큰 패턴
 */
class RenewableDelegate {
  constructor(issuer, audience) {
    this.issuer = issuer
    this.audience = audience
    this.revoked = false
    this.currentDelegation = null
  }

  // 짧은 만료 시간으로 Delegation 생성
  async issue(capabilities) {
    if (this.revoked) {
      throw new Error('Delegation has been revoked')
    }

    this.currentDelegation = await Client.delegate({
      issuer: this.issuer,
      audience: this.audience,
      capabilities,
      expiration: Math.floor(Date.now() / 1000) + 300  // 5분만 유효
    })

    return this.currentDelegation
  }

  // 갱신 요청 (아직 revoked 안된 경우만)
  async renew() {
    if (this.revoked) {
      throw new Error('Cannot renew revoked delegation')
    }

    // 새로운 Delegation 발급
    return await this.issue(this.currentDelegation.capabilities)
  }

  // Revocation (더 이상 갱신 불가)
  revoke() {
    this.revoked = true
  }
}

// 사용 예시
const renewable = new RenewableDelegate(alice, bob.principal)
const delegation = await renewable.issue([
  { can: 'space/blob/add', with: spaceDID }
])

// Bob이 주기적으로 갱신 (5분마다)
setInterval(async () => {
  try {
    delegation = await renewable.renew()
  } catch (error) {
    console.error('Delegation revoked or expired')
  }
}, 4 * 60 * 1000)  // 4분마다 갱신

// Alice가 revoke
renewable.revoke()
// 다음 갱신 시도 시 실패
```

#### 8.3.2 Blocklist 기반 Revocation

```typescript
/**
 * 서버 측 Blocklist 구현
 */
interface BlocklistEntry {
  delegationCID: string
  revokedAt: number
  reason: string
}

class DelegationBlocklist {
  private blocklist = new Map<string, BlocklistEntry>()

  // Delegation 차단
  async revoke(delegationCID: string, reason: string) {
    this.blocklist.set(delegationCID, {
      delegationCID,
      revokedAt: Date.now(),
      reason
    })

    // 영구 저장소에 기록
    await db.blocklist.insert({
      cid: delegationCID,
      revoked_at: new Date(),
      reason
    })
  }

  // Delegation 검증 시 확인
  async isRevoked(delegationCID: string): boolean {
    return this.blocklist.has(delegationCID)
  }

  // Proof chain 전체 검증
  async validateProofChain(proofs: Delegation[]): Promise<boolean> {
    for (const proof of proofs) {
      const cid = await proof.cid()

      if (await this.isRevoked(cid.toString())) {
        throw new Error(`Delegation ${cid} has been revoked`)
      }

      // 재귀적으로 하위 proofs 검증
      if (proof.proofs.length > 0) {
        await this.validateProofChain(proof.proofs)
      }
    }

    return true
  }
}

// Server handler에 통합
const createBlobAddHandler = (context) => {
  return provide(BlobCapabilities.add, async ({ invocation }) => {
    // 1. Proof chain 검증
    await context.blocklist.validateProofChain(invocation.proofs)

    // 2. 나머지 로직...
  })
}
```

#### 8.3.3 DID Document Rotation

```typescript
/**
 * DID Document를 변경하여 이전 키 무효화
 * (did:web, did:ion 등 mutable DID methods만 가능)
 */

// 이전 키로 발급한 모든 Delegation 무효화
await didDocument.rotateKey({
  oldKey: 'z6MkOldKey...',
  newKey: 'z6MkNewKey...'
})

// 이후 검증 시 oldKey로 서명된 UCAN은 모두 실패
```

### 8.4 Proof Chain 검증 최적화

긴 Delegation chain은 검증 비용이 높습니다:

```typescript
/**
 * Proof chain 깊이 제한
 */
const MAX_PROOF_DEPTH = 10

async function validateProofDepth(
  proofs: Delegation[],
  depth: number = 0
): Promise<void> {
  if (depth > MAX_PROOF_DEPTH) {
    throw new Error(`Proof chain too deep (max: ${MAX_PROOF_DEPTH})`)
  }

  for (const proof of proofs) {
    if (proof.proofs.length > 0) {
      await validateProofDepth(proof.proofs, depth + 1)
    }
  }
}
```

### 8.5 Private Key 보안

**DID 생성에 사용되는 Private Key는 절대 유출되어서는 안 됩니다:**

```typescript
/**
 * ✅ 좋은 예: 안전한 Key 저장
 */

// 1. 브라우저: IndexedDB (암호화)
import { Store } from '@web3-storage/access/stores/store'

const store = new Store({
  name: 'w3up-keystore',
  version: 1
})

await store.save({
  principal: agent.did(),
  key: await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv: randomIV },
    masterKey,
    agent.export()
  )
})

// 2. Node.js: OS Keychain (macOS/Linux/Windows)
import keytar from 'keytar'

await keytar.setPassword(
  'storacha-app',
  agent.did(),
  JSON.stringify(agent.export())
)

// 3. 서버: AWS Secrets Manager / HashiCorp Vault
import { SecretsManager } from '@aws-sdk/client-secrets-manager'

const sm = new SecretsManager({ region: 'us-west-2' })
await sm.createSecret({
  Name: `storacha/agent/${agent.did()}`,
  SecretString: JSON.stringify(agent.export())
})
```

```typescript
/**
 * ❌ 나쁜 예: 절대 하지 말 것
 */

// 평문으로 파일 저장
fs.writeFileSync('./agent-key.json', JSON.stringify(agent.export()))

// 환경 변수에 직접 저장
process.env.AGENT_PRIVATE_KEY = agent.export().toString()

// Git에 커밋
// .gitignore에 반드시 추가!
```

### 8.6 UCAN의 한계와 트레이드오프

#### 8.6.1 Revocation의 어려움

**한계:**
- UCAN 토큰은 Bearer Token (소지자 토큰)
- 일단 발급되면 만료 전까지 유효
- 실시간 Revocation 불가능

**트레이드오프:**
- ✅ 장점: 오프라인 검증 가능, 확장성 높음
- ❌ 단점: 권한 즉시 회수 불가

**해결책:**
- 짧은 만료 시간 + 갱신 메커니즘
- 서버 측 Blocklist
- DID Document rotation (mutable DID methods)

#### 8.6.2 검증 오버헤드

**한계:**
- 긴 Delegation chain → 많은 서명 검증
- DID resolution (did:web, did:ion) → 네트워크 요청
- 매 요청마다 전체 chain 검증

**트레이드오프:**
- ✅ 장점: 완전한 권한 추적, 감사 가능
- ❌ 단점: CPU 사용량 증가, 지연 시간 증가

**해결책:**
- Attestation (검증 결과 캐싱)
- Proof chain 깊이 제한
- 병렬 검증

#### 8.6.3 Privacy vs. Traceability

**한계:**
- Delegation chain이 공개되면 권한 흐름 노출
- CID 기반 참조 → IPFS에 저장 시 공개

**트레이드오프:**
- ✅ 장점: 투명성, 감사 가능성
- ❌ 단점: 프라이버시 침해 가능

**해결책:**
- Encrypted Delegations (수신자만 복호화 가능)
- Private IPFS network
- Off-chain proof storage

---

## 9. 성능 최적화

### 9.1 UCAN Attestation

**Attestation**은 UCAN 검증 결과를 캐싱하여 성능을 향상시킵니다:

```typescript
/**
 * ucan/attest Capability
 * - UCAN 검증 결과를 권위 있는 주체가 증명
 */
interface AttestationCapability {
  can: 'ucan/attest'
  with: string              // Attester의 DID
  nb: {
    proof: Link             // 검증된 UCAN의 CID
    validAt: number         // 검증 시각 (Unix timestamp)
    expiresAt: number       // Attestation 만료 시각
  }
}

/**
 * Attestation 생성 (서버)
 */
async function createAttestation(
  delegation: Delegation,
  serverID: Signer
): Promise<Delegation> {
  // 1. Delegation 검증
  await validateDelegation(delegation)

  // 2. Attestation 발급
  const attestation = await Client.delegate({
    issuer: serverID,
    audience: delegation.issuer,
    capabilities: [{
      can: 'ucan/attest',
      with: serverID.did(),
      nb: {
        proof: await delegation.cid(),
        validAt: Math.floor(Date.now() / 1000),
        expiresAt: Math.floor(Date.now() / 1000) + 3600  // 1시간
      }
    }],
    expiration: Math.floor(Date.now() / 1000) + 3600
  })

  return attestation
}

/**
 * Attestation을 사용한 빠른 검증
 */
async function validateWithAttestation(
  invocation: Invocation,
  attestations: Delegation[]
): Promise<boolean> {
  for (const proof of invocation.proofs) {
    const proofCID = await proof.cid()

    // Attestation 찾기
    const attestation = attestations.find(att => {
      const nb = att.capabilities[0].nb as any
      return nb.proof.equals(proofCID)
    })

    if (attestation) {
      // Attestation 검증 (서명 + 만료)
      const valid = await verifyAttestation(attestation)
      if (valid) {
        // ✅ 전체 proof chain 검증 생략!
        return true
      }
    }
  }

  // Attestation 없으면 전체 검증
  return await validateProofChain(invocation.proofs)
}
```

### 9.2 Delegation 캐싱

```typescript
/**
 * LRU Cache로 Delegation 캐싱
 */
import { LRUCache } from 'lru-cache'

const delegationCache = new LRUCache<string, {
  delegation: Delegation
  validatedAt: number
}>({
  max: 10000,                         // 최대 10,000개
  ttl: 5 * 60 * 1000,                 // 5분 TTL
  updateAgeOnGet: true,
  updateAgeOnHas: false
})

async function getCachedDelegation(cid: CID): Promise<Delegation | null> {
  const key = cid.toString()
  const cached = delegationCache.get(key)

  if (cached) {
    // 캐시 히트
    return cached.delegation
  }

  // 캐시 미스 - IPFS에서 fetch
  const delegation = await fetchFromIPFS(cid)

  // 캐시 저장
  delegationCache.set(key, {
    delegation,
    validatedAt: Date.now()
  })

  return delegation
}
```

### 9.3 병렬 검증

```typescript
/**
 * Proof chain 병렬 검증
 */
async function validateProofChainParallel(
  proofs: Delegation[]
): Promise<boolean> {
  // 모든 proof를 병렬로 검증
  const results = await Promise.all(
    proofs.map(async (proof) => {
      // 1. 서명 검증
      const signatureValid = await verifySignature(proof)

      // 2. 만료 확인
      const notExpired = proof.expiration > Date.now() / 1000

      // 3. 재귀 검증
      if (proof.proofs.length > 0) {
        const proofsValid = await validateProofChainParallel(proof.proofs)
        return signatureValid && notExpired && proofsValid
      }

      return signatureValid && notExpired
    })
  )

  return results.every(valid => valid)
}
```

### 9.4 Batch Invocation 활용

```typescript
/**
 * 여러 작업을 하나의 요청으로 묶기
 */
async function uploadMultipleFiles(files: File[]) {
  const invocations = files.map(file =>
    Client.invoke({
      issuer: agent,
      audience: serverID,
      capability: {
        can: 'space/blob/add',
        with: spaceDID,
        nb: {
          blob: {
            digest: file.digest,
            size: file.size
          }
        }
      },
      proofs: [delegation]  // 모든 invocation이 같은 proof 공유
    })
  )

  // 단일 HTTP 요청으로 실행
  const results = await connection.execute(invocations)

  // Proof는 한 번만 전송되고 검증됨!
  return results
}
```

---

## 10. 실전 Tips

### 10.1 개발 환경 설정

```typescript
/**
 * 개발/프로덕션 환경 분리
 */
const config = {
  development: {
    serverURL: 'http://localhost:3000',
    defaultExpiration: 24 * 60 * 60,      // 24시간
    logLevel: 'debug',
    cacheEnabled: false                   // 캐시 비활성화
  },
  production: {
    serverURL: 'https://api.storacha.network',
    defaultExpiration: 60 * 60,           // 1시간
    logLevel: 'error',
    cacheEnabled: true
  }
}[process.env.NODE_ENV || 'development']

const client = await create({
  serviceConf: {
    access: new URL(config.serverURL),
    upload: new URL(config.serverURL)
  }
})
```

### 10.2 에러 처리

```typescript
/**
 * UCAN 관련 에러 핸들링
 */
async function handleInvocationError(error: any) {
  // 1. 만료된 Delegation
  if (error.name === 'ExpiredDelegation') {
    console.log('Delegation expired, requesting renewal...')
    const newDelegation = await requestRenewal()
    return await retryWithNewProof(newDelegation)
  }

  // 2. 권한 부족
  if (error.name === 'InsufficientCapability') {
    console.error('Missing required capability:', error.required)
    console.error('Available capabilities:', error.available)
    throw new Error('Please request additional permissions')
  }

  // 3. 서명 검증 실패
  if (error.name === 'InvalidSignature') {
    console.error('UCAN signature invalid - possible tampering')
    throw new Error('Security violation detected')
  }

  // 4. Revoked Delegation
  if (error.name === 'RevokedDelegation') {
    console.error('Delegation has been revoked')
    throw new Error('Your access has been revoked')
  }

  // 5. 일반 에러
  throw error
}
```

### 10.3 Delegation 모니터링

```typescript
/**
 * Delegation 사용 추적
 */
class DelegationMonitor {
  private usage = new Map<string, {
    count: number
    lastUsed: number
    errors: number
  }>()

  async trackInvocation(delegation: Delegation, success: boolean) {
    const cid = (await delegation.cid()).toString()

    const stats = this.usage.get(cid) || {
      count: 0,
      lastUsed: 0,
      errors: 0
    }

    stats.count++
    stats.lastUsed = Date.now()
    if (!success) stats.errors++

    this.usage.set(cid, stats)

    // 임계치 초과 시 알림
    if (stats.errors > 10) {
      await this.alert(`Delegation ${cid} has ${stats.errors} errors`)
    }
  }

  async getStats(delegation: Delegation) {
    const cid = (await delegation.cid()).toString()
    return this.usage.get(cid)
  }
}
```

### 10.4 테스트 전략

```typescript
/**
 * UCAN 테스트 유틸리티
 */
import { EdSigner } from '@ucanto/principal/ed25519'
import { Client } from '@ucanto/client'

// 테스트용 Agent 생성 (deterministic)
function createTestAgent(seed: string): EdSigner {
  return EdSigner.fromSeed(new TextEncoder().encode(seed))
}

// 테스트용 Delegation 생성
async function createTestDelegation(options: {
  issuer?: EdSigner
  audience?: EdSigner
  capabilities?: Capability[]
  expiration?: number
}) {
  const issuer = options.issuer || createTestAgent('test-issuer')
  const audience = options.audience || createTestAgent('test-audience')

  return await Client.delegate({
    issuer,
    audience: audience.principal,
    capabilities: options.capabilities || [{
      can: 'test/action',
      with: issuer.did()
    }],
    expiration: options.expiration || Math.floor(Date.now() / 1000) + 3600
  })
}

// 만료된 Delegation 생성 (에러 테스트용)
async function createExpiredDelegation() {
  return await createTestDelegation({
    expiration: Math.floor(Date.now() / 1000) - 3600  // 1시간 전 만료
  })
}
```

---

## 11. 결론

UCAN Protocol은 **탈중앙화된 권한 관리**를 위한 강력한 프레임워크입니다.

### 핵심 요약:

1. **Capability-based Authorization**
   - Object Capability Model
   - Delegation과 Attenuation
   - Bearer Token 기반

2. **DID 기반 Identity**
   - did:key로 Self-Certifying
   - Ed25519 서명
   - 탈중앙화된 Identity

3. **ucanto RPC**
   - UCAN 기반 RPC 프레임워크
   - Type-safe Capability 정의
   - Pluggable Transport

4. **Storacha 구현**
   - w3up protocol
   - Space/Blob/Upload Capabilities
   - Delegation 기반 공유

5. **보안 고려사항**
   - 최소 권한 원칙
   - 짧은 만료 시간
   - Revocation 전략

6. **성능 최적화**
   - Attestation (검증 캐싱)
   - Batch Invocation
   - 병렬 검증

### 언제 UCAN을 사용해야 하는가?

✅ **적합한 경우:**
- 탈중앙화 애플리케이션 (dApp)
- P2P 권한 위임이 필요한 시스템
- 오프라인 검증이 중요한 환경
- Long-lived delegation이 필요한 경우

⚠️ **부적합한 경우:**
- 실시간 Revocation이 필수적인 경우
- 중앙 집중식 권한 관리가 충분한 경우
- 검증 오버헤드가 문제가 되는 경우

### 추가 학습 자료:

- [UCAN Specification](https://github.com/ucan-wg/spec)
- [ucanto Documentation](https://github.com/storacha/ucanto)
- [Storacha Docs](https://docs.storacha.network)
- [w3up Examples](https://github.com/storacha/w3up-examples)

---

**문서 작성 완료!** 🎉

본 문서는 UCAN Protocol의 개념부터 Storacha에서의 실제 구현까지 심층적으로 다루었습니다.
