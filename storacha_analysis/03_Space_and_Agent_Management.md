# Storacha Space와 Agent 관리 (Space and Agent Management)

> **작성일:** 2025-01-14
> **문서 버전:** 1.0
> **대상 독자:** Storacha 개발자, 시스템 아키텍트
> **선행 문서:** 01_UCAN_Protocol_Deep_Dive.md

---

## 목차

1. [개요](#1-개요)
2. [Space 개념](#2-space-개념)
3. [Agent 개념](#3-agent-개념)
4. [Space 생성 및 관리](#4-space-생성-및-관리)
5. [Agent 초기화 및 키 관리](#5-agent-초기화-및-키-관리)
6. [Space-Agent 연결](#6-space-agent-연결)
7. [다중 Space 관리](#7-다중-space-관리)
8. [Space 공유 및 권한 위임](#8-space-공유-및-권한-위임)
9. [보안 고려사항](#9-보안-고려사항)
10. [Best Practices](#10-best-practices)

---

## 1. 개요

Storacha의 권한 관리는 **Space**와 **Agent**라는 두 가지 핵심 개념을 중심으로 구성됩니다.

### 1.1 핵심 개념

```mermaid
graph TB
    subgraph "Account (사용자)"
        A[Email: alice@example.com]
    end

    subgraph "Spaces (저장소)"
        S1[Space 1<br/>did:key:z6Mk...]
        S2[Space 2<br/>did:key:z6Mp...]
    end

    subgraph "Agents (장치)"
        AG1[Browser Agent<br/>did:key:z6Mka...]
        AG2[Server Agent<br/>did:key:z6Mkb...]
        AG3[Mobile Agent<br/>did:key:z6Mkc...]
    end

    A -->|owns| S1
    A -->|owns| S2

    S1 -->|delegates to| AG1
    S1 -->|delegates to| AG2
    S2 -->|delegates to| AG3

    style A fill:#e1f5ff
    style S1 fill:#ffe1f5
    style S2 fill:#ffe1f5
    style AG1 fill:#e1ffe1
    style AG2 fill:#e1ffe1
    style AG3 fill:#e1ffe1
```

### 1.2 용어 정리

| 용어 | 설명 | DID 형식 | 예시 |
|-----|------|---------|-----|
| **Account** | 사용자 계정 (Email 기반) | `did:mailto:` | `did:mailto:alice@example.com` |
| **Space** | 저장소 네임스페이스 | `did:key:` | `did:key:z6MkffDZCkC...` |
| **Agent** | 장치별 Identity | `did:key:` | `did:key:z6Mka9zXVnJ...` |
| **Delegation** | 권한 위임 토큰 | UCAN CID | `bafyreib4dzh...` |

---

## 2. Space 개념

### 2.1 Space란?

**Space**는 Storacha의 **저장소 단위**입니다:

- DID 기반의 고유 식별자
- 데이터의 논리적 그룹핑
- 권한 관리의 기본 단위
- Quota 적용 대상

```
Space = 독립적인 저장소 네임스페이스
├─ DID: did:key:z6Mk... (고유 식별자)
├─ Uploads: [bafy..., bafy..., ...] (업로드된 파일들)
├─ Quota: 100GB (저장 용량 제한)
└─ Delegations: [Agent1, Agent2, ...] (접근 권한)
```

### 2.2 Space vs Account

| 특성 | Account | Space |
|-----|---------|-------|
| **식별 방법** | Email (did:mailto:) | DID (did:key:) |
| **개수** | 사용자당 1개 | Account당 여러 개 |
| **용도** | 인증 및 소유권 | 데이터 저장 |
| **Key 저장** | 서버 (복구 가능) | 클라이언트 (복구 불가) |
| **Quota** | N/A | Space별로 설정 |

```typescript
// Account 생성 (Email 인증)
const account = await client.login('alice@example.com')
// 결과: Account { did: 'did:mailto:alice@example.com' }

// Space 생성 (로컬에서 키 생성)
const space = await client.createSpace('my-project')
// 결과: Space { did: 'did:key:z6MkffDZCkC...' }
```

### 2.3 Space의 역할

#### 2.3.1 데이터 격리 (Isolation)

각 Space는 완전히 독립적입니다:

```typescript
const space1 = await client.createSpace('project-A')
const space2 = await client.createSpace('project-B')

// Space 1에 업로드
await client.setCurrentSpace(space1.did())
const cid1 = await client.uploadFile(file1)  // bafy...

// Space 2에 업로드
await client.setCurrentSpace(space2.did())
const cid2 = await client.uploadFile(file2)  // bafy...

// 각 Space에 별도로 저장됨
```

#### 2.3.2 권한 경계 (Permission Boundary)

권한은 Space 단위로 부여됩니다:

```typescript
// Space 1에 대한 권한
{
  can: 'space/blob/add',
  with: space1.did()  // Space 1만 접근 가능
}

// Space 2에는 접근 불가
```

#### 2.3.3 Quota 관리

```typescript
// Space별 Quota 확인
const usage = await client.capability.usage.report(space1.did())
console.log(`Used: ${usage.used} / ${usage.quota} bytes`)

// 결과:
// Used: 52,428,800 / 107,374,182,400 bytes (50MB / 100GB)
```

### 2.4 Space DID 구조

Space DID는 Ed25519 키쌍에서 생성됩니다:

```
Space DID 생성 과정:
1. Ed25519 키쌍 생성 (32 bytes private + 32 bytes public)
2. 공개키를 did:key로 인코딩
3. 개인키는 로컬에 안전하게 저장

Private Key (32 bytes)
  ↓
Public Key (32 bytes)
  ↓
did:key:z6MkffDZCkCTWreg8868fG1FGFogcJj5X6PY93pPcWDn9bob
│      │ └──────────────────────────────────────────────┘
│      │              Multibase base58-btc 인코딩
│      └── Method (key)
└── Scheme (did)
```

---

## 3. Agent 개념

### 3.1 Agent란?

**Agent**는 **장치별 Identity**를 나타냅니다:

- 각 장치/애플리케이션마다 하나의 Agent
- Ed25519 키쌍 기반
- Space에 대한 권한을 **위임받음** (Delegation)
- 로컬에 키 저장

```
Agent = 장치의 Identity
├─ DID: did:key:z6Mka... (고유 식별자)
├─ Private Key: [32 bytes] (로컬 저장)
├─ Public Key: [32 bytes] (DID에 포함)
├─ Delegations: [Space1 권한, Space2 권한, ...] (위임받은 권한들)
└─ Store: IndexedDB / 파일시스템 (키 저장소)
```

### 3.2 Agent가 필요한 이유

#### 3.2.1 장치별 권한 관리

각 장치가 독립적인 권한을 가집니다:

```typescript
// 브라우저 Agent
const browserAgent = await Client.create()
console.log(browserAgent.did())
// "did:key:z6Mka..."

// 서버 Agent
const serverAgent = await Client.create()
console.log(serverAgent.did())
// "did:key:z6Mkb..."

// 다른 DID → 별도의 권한 추적 가능
```

#### 3.2.2 권한 취소 (Revocation)

특정 장치의 권한만 제거 가능:

```typescript
// 브라우저 Agent에게 권한 위임
const delegation = await space.createDelegation({
  audience: browserAgent.did(),
  capabilities: [{ can: 'space/blob/add', with: space.did() }],
  expiration: Date.now() + 86400 * 7  // 7일
})

// 7일 후 자동 만료 → 브라우저만 접근 불가
// 서버 Agent는 여전히 접근 가능
```

#### 3.2.3 최소 권한 원칙

각 Agent에게 필요한 권한만 부여:

```typescript
// 읽기 전용 Agent (웹사이트 호스팅)
const readOnlyDelegation = await space.createDelegation({
  audience: websiteAgent.did(),
  capabilities: [
    { can: 'space/content/serve', with: space.did() }  // 읽기만
  ]
})

// 쓰기 가능 Agent (백업 서버)
const writeOnlyDelegation = await space.createDelegation({
  audience: backupAgent.did(),
  capabilities: [
    { can: 'space/blob/add', with: space.did() }  // 쓰기만
  ]
})
```

### 3.3 Agent 저장소 (Agent Store)

Agent의 개인키와 Delegation은 **로컬 저장소**에 보관됩니다:

#### 3.3.1 브라우저 (IndexedDB)

```typescript
import { StoreIndexedDB } from '@web3-storage/w3up-client/stores/indexeddb'

const store = new StoreIndexedDB('w3up-agent')

// Agent 정보 저장
await store.save({
  did: agent.did(),
  meta: {
    name: 'Browser Agent',
    created: new Date()
  }
})

// 키 저장 (암호화)
await store.saveKey({
  did: agent.did(),
  key: agent.export()
})
```

저장 구조:

```
IndexedDB: 'w3up-agent'
└─ Object Store: 'agents'
   ├─ Key: 'did:key:z6Mka...'
   │  ├─ did: 'did:key:z6Mka...'
   │  ├─ meta: { name: '...', created: '...' }
   │  └─ key: [Encrypted 32 bytes]
   └─ Object Store: 'delegations'
      ├─ Key: 'bafyreib...'
      │  └─ delegation: [UCAN bytes]
      └─ Key: 'bafyreid...'
         └─ delegation: [UCAN bytes]
```

#### 3.3.2 Node.js (파일 시스템)

```typescript
import { StoreConf } from '@web3-storage/w3up-client/stores/conf'
import os from 'os'
import path from 'path'

const storePath = path.join(os.homedir(), '.w3up')
const store = new StoreConf({ profile: 'default', path: storePath })

// Agent 저장
await store.save({
  did: agent.did(),
  meta: { name: 'Server Agent' }
})
```

저장 구조:

```
~/.w3up/
├─ default/
│  ├─ agent.json
│  │  {
│  │    "did": "did:key:z6Mkb...",
│  │    "meta": { "name": "Server Agent" }
│  │  }
│  ├─ key (PEM 형식, 암호화)
│  │  -----BEGIN PRIVATE KEY-----
│  │  ...
│  │  -----END PRIVATE KEY-----
│  └─ delegations/
│     ├─ bafyreib...car
│     └─ bafyreid...car
└─ production/
   └─ ...
```

#### 3.3.3 React Native (AsyncStorage)

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage'

class AsyncStorageStore {
  private prefix = '@w3up:'

  async save({ did, meta }) {
    await AsyncStorage.setItem(
      `${this.prefix}agent:${did}`,
      JSON.stringify({ did, meta })
    )
  }

  async saveKey({ did, key }) {
    // 키 암호화 (Keychain API 사용 권장)
    await AsyncStorage.setItem(
      `${this.prefix}key:${did}`,
      JSON.stringify(key)
    )
  }
}
```

### 3.4 Agent Lifecycle

```mermaid
graph TB
    A[Agent 생성] --> B[키쌍 생성]
    B --> C[로컬 저장소에 저장]
    C --> D[Space로부터 Delegation 받기]
    D --> E[작업 수행]
    E --> F{Delegation 만료?}
    F -->|아니오| E
    F -->|예| G[갱신 요청]
    G --> D

    style A fill:#e1f5ff
    style E fill:#e1ffe1
    style F fill:#fff4e1
```

---

## 4. Space 생성 및 관리

### 4.1 Space 생성

```typescript
import { create } from '@storacha/client'

// 1. Client 생성
const client = await create()

// 2. Account 로그인 (필수)
const account = await client.login('alice@example.com')
console.log(`Logged in as: ${account.did()}`)
// "did:mailto:alice@example.com"

// 3. Space 생성
const space = await client.createSpace('my-project', {
  account  // Space를 Account에 연결
})

console.log(`Space created: ${space.did()}`)
// "did:key:z6MkffDZCkCTWreg8868fG1FGFogcJj5X6PY93pPcWDn9bob"

// 4. Space 활성화 (현재 작업 Space 설정)
await client.setCurrentSpace(space.did())

// 5. Space를 Account에 등록 (서버에 기록)
await space.provision(account)
```

### 4.2 Space 생성 내부 구현

**`@storacha/client` 내부:**

```typescript
/**
 * Space 생성 구현 (간소화)
 */
export async function createSpace(
  name: string,
  options: { account?: Account } = {}
): Promise<Space> {
  // 1. Ed25519 키쌍 생성
  const privateKey = ed25519.utils.randomPrivateKey()
  const signer = EdSigner.from(privateKey)

  // 2. Space 객체 생성
  const space = {
    did: () => signer.did(),
    name,
    signer,
    meta: {
      name,
      created: new Date().toISOString()
    }
  }

  // 3. 로컬 저장소에 저장
  await this.agent.store.saveSpace(space)

  // 4. Account와 연결 (선택)
  if (options.account) {
    await this.linkSpaceToAccount(space, options.account)
  }

  return space
}
```

### 4.3 Space Provisioning

Space를 Account에 연결하여 서버에 등록:

```typescript
/**
 * Space Provisioning
 */
export async function provision(
  space: Space,
  account: Account
): Promise<void> {
  // 1. Account → Space Delegation 생성
  const delegation = await this.createAccountDelegation({
    audience: space.did(),
    capabilities: [
      { can: 'space/*', with: space.did() }  // 모든 Space 권한
    ],
    issuer: account,
    expiration: Infinity  // 영구 위임
  })

  // 2. 서버에 Space 등록
  await this.invoke({
    issuer: this.agent,
    audience: this.connection.id,
    capability: {
      can: 'provider/add',
      with: account.did(),
      nb: {
        consumer: space.did(),
        proof: delegation.cid()
      }
    },
    proofs: [delegation]
  })

  console.log(`Space ${space.did()} provisioned`)
}
```

### 4.4 Space 조회 및 관리

```typescript
/**
 * 모든 Space 조회
 */
const spaces = await client.spaces()
console.log(`Found ${spaces.length} spaces:`)

for (const space of spaces) {
  console.log(`  - ${space.name}: ${space.did()}`)
}

// 결과:
// Found 3 spaces:
//   - my-project: did:key:z6MkffDZCkC...
//   - backup-space: did:key:z6Mkpq9zL...
//   - test-space: did:key:z6Mkw4X...

/**
 * 특정 Space 선택
 */
await client.setCurrentSpace(spaces[0].did())

/**
 * Space 이름 변경
 */
await client.renameSpace(space.did(), 'new-project-name')

/**
 * Space 삭제 (로컬만)
 */
await client.removeSpace(space.did())
```

### 4.5 Space 메타데이터

```typescript
/**
 * Space 메타데이터 구조
 */
interface SpaceMeta {
  name: string
  created: string        // ISO 8601
  description?: string
  tags?: string[]
  color?: string         // UI 표시용
}

// 메타데이터 업데이트
await client.updateSpaceMeta(space.did(), {
  description: 'Main project storage',
  tags: ['production', 'web3'],
  color: '#4CAF50'
})
```

---

## 5. Agent 초기화 및 키 관리

### 5.1 Agent 생성

#### 5.1.1 자동 Agent 생성 (브라우저)

```typescript
import { create } from '@storacha/client'

// 자동으로 Agent 생성 및 로컬 저장
const client = await create()

console.log(`Agent DID: ${client.agent.did()}`)
// "did:key:z6Mka9zXVnJ..."

// IndexedDB에 자동 저장됨
```

**내부 동작:**

```typescript
/**
 * Client 생성 시 Agent 자동 초기화
 */
export async function create(options: CreateOptions = {}): Promise<Client> {
  // 1. Store 선택 (브라우저 = IndexedDB)
  const store = options.store ?? new StoreIndexedDB('w3up-client')

  // 2. 기존 Agent 확인
  let principal = options.principal
  if (!principal) {
    const saved = await store.load()
    if (saved) {
      principal = saved.principal
    } else {
      // 3. 새 Agent 생성
      principal = await EdSigner.generate()
      await store.save({ principal })
    }
  }

  // 4. Agent 객체 생성
  const agent = await Agent.create({ principal, store })

  return new Client({ agent, connection: options.connection })
}
```

#### 5.1.2 명시적 Agent 생성 (서버)

```typescript
import * as Signer from '@ucanto/principal/ed25519'
import { StoreMemory } from '@storacha/client/stores/memory'

// 1. 개인키에서 Agent 생성
const principal = Signer.parse(process.env.W3UP_PRIVATE_KEY)

// 2. 메모리 Store 사용 (서버리스 환경)
const store = new StoreMemory()

// 3. Client 생성
const client = await create({ principal, store })

console.log(`Server Agent: ${client.agent.did()}`)
// "did:key:z6Mkb..."
```

#### 5.1.3 Agent 키 생성 및 내보내기

```typescript
/**
 * 새 Agent 키 생성
 */
import * as ed25519 from '@noble/ed25519'

const privateKey = ed25519.utils.randomPrivateKey()
const publicKey = await ed25519.getPublicKey(privateKey)

console.log('Private Key (hex):', Buffer.from(privateKey).toString('hex'))
// "8f3b4c5d6e7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c"

/**
 * Agent 키 내보내기 (백업용)
 */
const exported = await client.agent.export()

// Base64 형식으로 저장
const keyBase64 = Buffer.from(exported).toString('base64')
console.log('Exported Key:', keyBase64)

// 환경 변수로 저장
// W3UP_PRIVATE_KEY=MIIEvgIBADANBgkqhkiG9w0BAQEFAASC...
```

### 5.2 Agent 키 저장소 구현

#### 5.2.1 IndexedDB Store (브라우저)

```typescript
import { StoreIndexedDB } from '@storacha/client/stores/indexeddb'

/**
 * IndexedDB 기반 Agent Store
 */
export class StoreIndexedDB implements AgentStore {
  private dbName: string
  private db: IDBDatabase | null = null

  constructor(dbName: string = 'w3up-client') {
    this.dbName = dbName
  }

  /**
   * IndexedDB 초기화
   */
  async open(): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, 1)

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result

        // Agent 저장소
        if (!db.objectStoreNames.contains('agent')) {
          db.createObjectStore('agent', { keyPath: 'did' })
        }

        // Delegation 저장소
        if (!db.objectStoreNames.contains('delegations')) {
          db.createObjectStore('delegations', { keyPath: 'cid' })
        }

        // Space 저장소
        if (!db.objectStoreNames.contains('spaces')) {
          db.createObjectStore('spaces', { keyPath: 'did' })
        }
      }

      request.onsuccess = (event) => {
        this.db = (event.target as IDBOpenDBRequest).result
        resolve()
      }

      request.onerror = () => reject(request.error)
    })
  }

  /**
   * Agent 저장
   */
  async save(data: { principal: Principal }): Promise<void> {
    await this.open()
    const tx = this.db!.transaction(['agent'], 'readwrite')
    const store = tx.objectStore('agent')

    await store.put({
      did: data.principal.did(),
      key: await data.principal.export(),
      created: new Date().toISOString()
    })
  }

  /**
   * Agent 로드
   */
  async load(): Promise<{ principal: Principal } | null> {
    await this.open()
    const tx = this.db!.transaction(['agent'], 'readonly')
    const store = tx.objectStore('agent')

    const records = await store.getAll()
    if (records.length === 0) return null

    const record = records[0]
    const principal = await EdSigner.import(record.key)

    return { principal }
  }

  /**
   * Delegation 저장
   */
  async addDelegation(delegation: Delegation): Promise<void> {
    await this.open()
    const tx = this.db!.transaction(['delegations'], 'readwrite')
    const store = tx.objectStore('delegations')

    await store.put({
      cid: delegation.cid().toString(),
      bytes: delegation.archive(),
      capabilities: delegation.capabilities,
      audience: delegation.audience.did(),
      issuer: delegation.issuer.did(),
      expiration: delegation.expiration
    })
  }

  /**
   * Space 저장
   */
  async saveSpace(space: Space): Promise<void> {
    await this.open()
    const tx = this.db!.transaction(['spaces'], 'readwrite')
    const store = tx.objectStore('spaces')

    await store.put({
      did: space.did(),
      name: space.name,
      meta: space.meta,
      registered: space.registered ?? false
    })
  }
}
```

#### 5.2.2 파일 시스템 Store (Node.js)

```typescript
import { StoreConf } from '@storacha/client/stores/conf'
import fs from 'fs/promises'
import path from 'path'

/**
 * 파일 시스템 기반 Agent Store
 */
export class StoreConf implements AgentStore {
  private basePath: string

  constructor(options: { profile?: string; path?: string } = {}) {
    const profile = options.profile ?? 'default'
    this.basePath = options.path ?? path.join(os.homedir(), '.w3up', profile)
  }

  /**
   * Agent 저장
   */
  async save(data: { principal: Principal }): Promise<void> {
    await fs.mkdir(this.basePath, { recursive: true })

    // agent.json 저장
    await fs.writeFile(
      path.join(this.basePath, 'agent.json'),
      JSON.stringify({
        did: data.principal.did(),
        created: new Date().toISOString()
      }, null, 2)
    )

    // key 파일 저장 (PEM 형식)
    const keyBytes = await data.principal.export()
    await fs.writeFile(
      path.join(this.basePath, 'key'),
      this.toPEM(keyBytes),
      { mode: 0o600 }  // 소유자만 읽기/쓰기
    )
  }

  /**
   * Agent 로드
   */
  async load(): Promise<{ principal: Principal } | null> {
    try {
      const keyPath = path.join(this.basePath, 'key')
      const keyPEM = await fs.readFile(keyPath, 'utf-8')
      const keyBytes = this.fromPEM(keyPEM)
      const principal = await EdSigner.import(keyBytes)
      return { principal }
    } catch (error) {
      if (error.code === 'ENOENT') return null
      throw error
    }
  }

  /**
   * Delegation 저장 (CAR 파일)
   */
  async addDelegation(delegation: Delegation): Promise<void> {
    const delegationsDir = path.join(this.basePath, 'delegations')
    await fs.mkdir(delegationsDir, { recursive: true })

    const filename = `${delegation.cid().toString()}.car`
    await fs.writeFile(
      path.join(delegationsDir, filename),
      delegation.archive()
    )
  }

  /**
   * 바이트 배열을 PEM 형식으로 변환
   */
  private toPEM(bytes: Uint8Array): string {
    const base64 = Buffer.from(bytes).toString('base64')
    const lines = base64.match(/.{1,64}/g) || []
    return [
      '-----BEGIN PRIVATE KEY-----',
      ...lines,
      '-----END PRIVATE KEY-----'
    ].join('\n')
  }

  /**
   * PEM 형식을 바이트 배열로 변환
   */
  private fromPEM(pem: string): Uint8Array {
    const base64 = pem
      .replace('-----BEGIN PRIVATE KEY-----', '')
      .replace('-----END PRIVATE KEY-----', '')
      .replace(/\s/g, '')
    return new Uint8Array(Buffer.from(base64, 'base64'))
  }
}
```

### 5.3 Agent 복구 (Recovery)

#### 5.3.1 키 백업 및 복구

```typescript
/**
 * Agent 키 백업
 */
async function backupAgent(client: Client): Promise<string> {
  // 개인키 내보내기
  const keyBytes = await client.agent.export()
  const keyBase64 = Buffer.from(keyBytes).toString('base64')

  console.log('Save this key securely:')
  console.log(keyBase64)

  return keyBase64
}

/**
 * Agent 복구
 */
async function recoverAgent(keyBase64: string): Promise<Client> {
  // 키 파싱
  const keyBytes = Buffer.from(keyBase64, 'base64')
  const principal = await EdSigner.import(keyBytes)

  // Client 재생성
  const client = await create({ principal })

  console.log(`Recovered Agent: ${client.agent.did()}`)
  return client
}

// 사용 예시
const backupKey = await backupAgent(client)
// "MIIEvgIBADANBgkqhkiG9w0BAQEFAASC..."

// 다른 장치에서 복구
const recoveredClient = await recoverAgent(backupKey)
```

#### 5.3.2 환경 변수를 통한 복구 (서버)

```bash
# .env 파일
W3UP_PRIVATE_KEY=MIIEvgIBADANBgkqhkiG9w0BAQEFAASC...
```

```typescript
import * as Signer from '@ucanto/principal/ed25519'

/**
 * 환경 변수에서 Agent 로드
 */
async function loadAgentFromEnv(): Promise<Client> {
  const key = process.env.W3UP_PRIVATE_KEY
  if (!key) {
    throw new Error('W3UP_PRIVATE_KEY not found in environment')
  }

  const principal = Signer.parse(key)
  const store = new StoreMemory()

  return await create({ principal, store })
}

// CI/CD에서 사용
const client = await loadAgentFromEnv()
```

### 5.4 Agent 보안 관리

#### 5.4.1 키 회전 (Key Rotation)

```typescript
/**
 * Agent 키 회전
 */
async function rotateAgentKey(
  client: Client,
  spaces: Space[]
): Promise<Client> {
  // 1. 새 Agent 생성
  const newAgent = await EdSigner.generate()
  const newClient = await create({ principal: newAgent })

  // 2. 모든 Space에서 이전 Agent의 Delegation을 새 Agent로 재발급
  for (const space of spaces) {
    // 이전 Delegation 조회
    const oldProofs = await client.proofs([{
      can: 'space/*',
      with: space.did()
    }])

    // 새 Agent에게 동일한 권한 위임
    for (const proof of oldProofs) {
      const newDelegation = await client.createDelegation({
        audience: newAgent.did(),
        capabilities: proof.capabilities,
        expiration: proof.expiration,
        proofs: [proof]
      })

      await newClient.addProof(newDelegation)
    }
  }

  console.log(`Old Agent: ${client.agent.did()}`)
  console.log(`New Agent: ${newClient.agent.did()}`)

  return newClient
}
```

#### 5.4.2 키 암호화 (브라우저)

```typescript
/**
 * Web Crypto API를 사용한 키 암호화
 */
export class EncryptedStore {
  private store: StoreIndexedDB
  private encryptionKey: CryptoKey | null = null

  constructor(dbName: string) {
    this.store = new StoreIndexedDB(dbName)
  }

  /**
   * 사용자 비밀번호로 암호화 키 생성
   */
  async unlock(password: string): Promise<void> {
    // PBKDF2로 키 생성
    const encoder = new TextEncoder()
    const passwordKey = await crypto.subtle.importKey(
      'raw',
      encoder.encode(password),
      { name: 'PBKDF2' },
      false,
      ['deriveKey']
    )

    this.encryptionKey = await crypto.subtle.deriveKey(
      {
        name: 'PBKDF2',
        salt: encoder.encode('w3up-salt'),
        iterations: 100000,
        hash: 'SHA-256'
      },
      passwordKey,
      { name: 'AES-GCM', length: 256 },
      false,
      ['encrypt', 'decrypt']
    )
  }

  /**
   * Agent 키 암호화 저장
   */
  async save(data: { principal: Principal }): Promise<void> {
    if (!this.encryptionKey) {
      throw new Error('Store is locked. Call unlock() first.')
    }

    const keyBytes = await data.principal.export()

    // AES-GCM 암호화
    const iv = crypto.getRandomValues(new Uint8Array(12))
    const encrypted = await crypto.subtle.encrypt(
      { name: 'AES-GCM', iv },
      this.encryptionKey,
      keyBytes
    )

    // IV와 암호문 함께 저장
    const combined = new Uint8Array(iv.length + encrypted.byteLength)
    combined.set(iv, 0)
    combined.set(new Uint8Array(encrypted), iv.length)

    await this.store.save({
      principal: {
        did: () => data.principal.did(),
        export: async () => combined
      } as any
    })
  }

  /**
   * Agent 키 복호화 로드
   */
  async load(): Promise<{ principal: Principal } | null> {
    if (!this.encryptionKey) {
      throw new Error('Store is locked. Call unlock() first.')
    }

    const saved = await this.store.load()
    if (!saved) return null

    const combined = await saved.principal.export()

    // IV 추출
    const iv = combined.slice(0, 12)
    const encrypted = combined.slice(12)

    // AES-GCM 복호화
    const decrypted = await crypto.subtle.decrypt(
      { name: 'AES-GCM', iv },
      this.encryptionKey,
      encrypted
    )

    const principal = await EdSigner.import(new Uint8Array(decrypted))
    return { principal }
  }
}

// 사용 예시
const store = new EncryptedStore('w3up-client')
await store.unlock('user-password-123')

const client = await create({ store })
```

---

## 6. Space-Agent 연결

### 6.1 Delegation 메커니즘

Space와 Agent는 **UCAN Delegation**을 통해 연결됩니다:

```mermaid
sequenceDiagram
    participant A as Alice (Space 소유자)
    participant S as Space<br/>did:key:z6Mk...
    participant AG as Agent<br/>did:key:z6Mka...
    participant SVC as Storacha Service

    A->>S: createSpace()
    S->>A: Space 생성 완료

    A->>S: createDelegation(audience: Agent)
    S->>A: Delegation (UCAN)

    A->>AG: addProof(delegation)
    AG->>AG: Store에 Delegation 저장

    AG->>SVC: invoke(space/blob/add)
    Note over AG,SVC: Proof: [Delegation]
    SVC->>SVC: Delegation 검증
    SVC->>AG: Success
```

### 6.2 addSpace() 구현

```typescript
/**
 * Space를 Agent에 연결
 */
export async function addSpace(
  this: Client,
  proof: Delegation
): Promise<Space> {
  // 1. Delegation 검증
  const capabilities = proof.capabilities
  const spaceDIDs = new Set<string>()

  for (const cap of capabilities) {
    // Space 권한인지 확인
    if (cap.can.startsWith('space/') || cap.can === '*') {
      spaceDIDs.add(cap.with)
    }
  }

  if (spaceDIDs.size === 0) {
    throw new Error('Proof does not contain space capabilities')
  }

  if (spaceDIDs.size > 1) {
    throw new Error('Proof delegates multiple spaces')
  }

  const spaceDID = Array.from(spaceDIDs)[0]

  // 2. Audience 확인
  if (proof.audience.did() !== this.agent.did()) {
    throw new Error(
      `Proof audience (${proof.audience.did()}) does not match agent (${this.agent.did()})`
    )
  }

  // 3. Proof 저장
  await this.agent.addProof(proof)

  // 4. Space 메타데이터 추출
  const space: Space = {
    did: () => spaceDID,
    name: proof.facts?.find(f => f.name)?.name ?? 'Untitled Space',
    registered: false,
    meta: proof.facts?.[0] ?? {}
  }

  // 5. Space 저장
  await this.agent.store.saveSpace(space)

  console.log(`Space added: ${spaceDID}`)
  return space
}
```

### 6.3 Delegation 생성 및 공유

#### 6.3.1 CLI를 통한 Delegation 생성

```bash
# Space에서 Agent로 Delegation 생성
w3 delegation create did:key:z6Mka9zXVnJ... \
  --can 'space/blob/add' \
  --can 'space/index/add' \
  --can 'upload/add' \
  --expiration $(date -d '+30 days' +%s) \
  | base64

# 결과 (base64 인코딩된 CAR):
# YmFmeXJlaWI0ZHpoNWt3cDRrNnFxd3Q3cjN5NXM2dTd2OHc5eDBhMWIyYzNkNGU1ZjZn...
```

#### 6.3.2 프로그래매틱 Delegation 생성

```typescript
/**
 * Space에서 Agent로 Delegation 생성
 */
async function delegateToAgent(
  space: Space,
  agentDID: string,
  capabilities: string[]
): Promise<string> {
  // 1. Delegation 생성
  const delegation = await space.createDelegation({
    audience: DID.parse(agentDID),
    capabilities: capabilities.map(can => ({
      can,
      with: space.did()
    })),
    expiration: Math.floor(Date.now() / 1000) + 86400 * 30,  // 30일
    facts: [{
      name: space.name,
      description: `Delegation for ${agentDID}`
    }]
  })

  // 2. CAR 아카이브로 내보내기
  const archive = delegation.archive()

  // 3. Base64 인코딩
  const base64 = Buffer.from(archive).toString('base64')

  console.log('Delegation created:')
  console.log(base64)

  return base64
}

// 사용 예시
const proofBase64 = await delegateToAgent(
  space,
  'did:key:z6Mka9zXVnJ...',
  ['space/blob/add', 'upload/add']
)
```

#### 6.3.3 Delegation 가져오기 (Import)

```typescript
/**
 * Base64 Delegation을 파싱하여 Agent에 추가
 */
import { CarReader } from '@ipld/car'
import * as Delegation from '@ucanto/core/delegation'

async function importDelegation(
  client: Client,
  proofBase64: string
): Promise<Space> {
  // 1. Base64 디코딩
  const carBytes = Buffer.from(proofBase64, 'base64')

  // 2. CAR 파일 파싱
  const reader = await CarReader.fromBytes(carBytes)
  const blocks = []
  for await (const block of reader.blocks()) {
    blocks.push(block)
  }

  // 3. Delegation 재구성
  const delegation = await Delegation.importDAG(blocks)

  // 4. Agent에 추가
  const space = await client.addSpace(delegation)

  console.log(`Space imported: ${space.did()}`)
  return space
}

// 사용 예시
const space = await importDelegation(
  client,
  'YmFmeXJlaWI0ZHpoNWt3cDRrNnFxd3Q3cjN5NXM2dTd2OHc5eDBhMWIyYzNkNGU1ZjZn...'
)

await client.setCurrentSpace(space.did())
```

### 6.4 Proof 관리

#### 6.4.1 Proof 조회

```typescript
/**
 * 특정 Capability에 대한 Proof 조회
 */
const proofs = await client.proofs([
  { can: 'space/blob/add', with: space.did() }
])

console.log(`Found ${proofs.length} proofs:`)
for (const proof of proofs) {
  console.log(`  - CID: ${proof.cid()}`)
  console.log(`    Issuer: ${proof.issuer.did()}`)
  console.log(`    Audience: ${proof.audience.did()}`)
  console.log(`    Expiration: ${new Date(proof.expiration * 1000)}`)
  console.log(`    Capabilities:`, proof.capabilities)
}

// 결과:
// Found 1 proofs:
//   - CID: bafyreib4dzh5kwp4k6qqwt7r3y5s6u7v8w9x0a1b2c3d4e5f6g
//     Issuer: did:key:z6MkffDZCkC... (Space)
//     Audience: did:key:z6Mka9zXVnJ... (Agent)
//     Expiration: 2025-02-13T00:00:00.000Z
//     Capabilities: [
//       { can: 'space/blob/add', with: 'did:key:z6MkffDZCkC...' },
//       { can: 'upload/add', with: 'did:key:z6MkffDZCkC...' }
//     ]
```

#### 6.4.2 Proof 체인 검증

```typescript
/**
 * Delegation의 Proof 체인 검증
 */
export async function verifyProofChain(
  delegation: Delegation,
  expectedRoot: DID
): Promise<boolean> {
  // 1. Delegation 서명 검증
  const isValid = await delegation.verify()
  if (!isValid) {
    console.error('Invalid delegation signature')
    return false
  }

  // 2. Proof 체인 재귀 검증
  const proofs = delegation.proofs
  if (proofs.length === 0) {
    // Root delegation: Issuer가 Space DID인지 확인
    return delegation.issuer.did() === expectedRoot
  }

  // 중간 delegation: Proof 재귀 검증
  for (const proof of proofs) {
    const chainValid = await verifyProofChain(proof, expectedRoot)
    if (!chainValid) return false
  }

  // 3. Attenuation 검증 (권한이 축소되었는지 확인)
  for (const proof of proofs) {
    for (const cap of delegation.capabilities) {
      const hasMatchingProof = proof.capabilities.some(pCap =>
        isAttenuated(pCap, cap)
      )
      if (!hasMatchingProof) {
        console.error(`Capability ${cap.can} not attenuated from proof`)
        return false
      }
    }
  }

  return true
}

/**
 * Capability Attenuation 검증
 */
function isAttenuated(parent: Capability, child: Capability): boolean {
  // 1. 'with' (Resource) 검증
  if (parent.with !== child.with && parent.with !== '*') {
    return false
  }

  // 2. 'can' (Ability) 검증
  if (parent.can === '*') return true
  if (parent.can === child.can) return true

  // Wildcard prefix 검증 (space/* vs space/blob/add)
  const parentPrefix = parent.can.split('/').slice(0, -1).join('/')
  const childPrefix = child.can.split('/').slice(0, -1).join('/')
  if (parent.can.endsWith('/*') && childPrefix.startsWith(parentPrefix)) {
    return true
  }

  return false
}
```

---

## 7. 다중 Space 관리

### 7.1 여러 Space 생성 및 전환

```typescript
/**
 * 프로젝트별 Space 생성
 */
async function setupMultipleSpaces(client: Client) {
  const account = await client.login('alice@example.com')

  // 1. 프로덕션 Space
  const prodSpace = await client.createSpace('production', {
    account,
    meta: {
      description: 'Production data storage',
      tags: ['production', 'critical'],
      color: '#FF5722'
    }
  })
  await prodSpace.provision(account)

  // 2. 개발 Space
  const devSpace = await client.createSpace('development', {
    account,
    meta: {
      description: 'Development and testing',
      tags: ['dev', 'test'],
      color: '#4CAF50'
    }
  })
  await devSpace.provision(account)

  // 3. 백업 Space
  const backupSpace = await client.createSpace('backup', {
    account,
    meta: {
      description: 'Long-term archival',
      tags: ['backup', 'archive'],
      color: '#2196F3'
    }
  })
  await backupSpace.provision(account)

  console.log('Created 3 spaces:')
  console.log(`  Production: ${prodSpace.did()}`)
  console.log(`  Development: ${devSpace.did()}`)
  console.log(`  Backup: ${backupSpace.did()}`)
}

/**
 * Space 간 전환
 */
async function switchSpace(client: Client, spaceName: string) {
  const spaces = await client.spaces()
  const target = spaces.find(s => s.name === spaceName)

  if (!target) {
    throw new Error(`Space "${spaceName}" not found`)
  }

  await client.setCurrentSpace(target.did())
  console.log(`Switched to space: ${spaceName}`)
}

// 사용 예시
await switchSpace(client, 'production')
await client.uploadFile(productionData)

await switchSpace(client, 'development')
await client.uploadFile(testData)
```

### 7.2 Space 목록 및 상태 조회

```typescript
/**
 * 모든 Space 목록과 사용량 조회
 */
async function listSpacesWithUsage(client: Client) {
  const spaces = await client.spaces()

  console.log(`Total Spaces: ${spaces.length}\n`)

  for (const space of spaces) {
    console.log(`📦 ${space.name}`)
    console.log(`   DID: ${space.did()}`)
    console.log(`   Registered: ${space.registered ? 'Yes' : 'No'}`)

    if (space.registered) {
      // 사용량 조회
      const usage = await client.capability.usage.report(space.did())
      const usedGB = (usage.used / 1024 / 1024 / 1024).toFixed(2)
      const quotaGB = (usage.quota / 1024 / 1024 / 1024).toFixed(2)
      const percent = ((usage.used / usage.quota) * 100).toFixed(1)

      console.log(`   Usage: ${usedGB} GB / ${quotaGB} GB (${percent}%)`)

      // 업로드 수 조회
      const uploads = []
      for await (const item of client.capability.upload.list({ space: space.did() })) {
        uploads.push(item)
      }
      console.log(`   Uploads: ${uploads.length}`)
    }

    console.log(`   Tags: ${space.meta?.tags?.join(', ') ?? 'None'}`)
    console.log('')
  }
}

// 결과:
// Total Spaces: 3
//
// 📦 production
//    DID: did:key:z6MkffDZCkC...
//    Registered: Yes
//    Usage: 45.32 GB / 100.00 GB (45.3%)
//    Uploads: 1,234
//    Tags: production, critical
//
// 📦 development
//    DID: did:key:z6Mkpq9zL...
//    Registered: Yes
//    Usage: 2.15 GB / 50.00 GB (4.3%)
//    Uploads: 87
//    Tags: dev, test
```

### 7.3 Space 필터링 및 검색

```typescript
/**
 * Space 필터링 유틸리티
 */
export class SpaceManager {
  constructor(private client: Client) {}

  /**
   * 태그로 Space 필터링
   */
  async findByTag(tag: string): Promise<Space[]> {
    const spaces = await this.client.spaces()
    return spaces.filter(s =>
      s.meta?.tags?.includes(tag)
    )
  }

  /**
   * 이름으로 Space 검색 (부분 일치)
   */
  async findByName(query: string): Promise<Space[]> {
    const spaces = await this.client.spaces()
    const lowerQuery = query.toLowerCase()
    return spaces.filter(s =>
      s.name.toLowerCase().includes(lowerQuery)
    )
  }

  /**
   * 등록된 Space만 조회
   */
  async findRegistered(): Promise<Space[]> {
    const spaces = await this.client.spaces()
    return spaces.filter(s => s.registered)
  }

  /**
   * 사용량 기준 정렬
   */
  async sortByUsage(): Promise<Array<Space & { usage: number }>> {
    const spaces = await this.client.spaces()
    const withUsage = await Promise.all(
      spaces.map(async (space) => {
        if (!space.registered) {
          return { ...space, usage: 0 }
        }
        const report = await this.client.capability.usage.report(space.did())
        return { ...space, usage: report.used }
      })
    )

    return withUsage.sort((a, b) => b.usage - a.usage)
  }
}

// 사용 예시
const manager = new SpaceManager(client)

// 프로덕션 태그가 있는 Space만
const prodSpaces = await manager.findByTag('production')

// 'test'가 포함된 Space
const testSpaces = await manager.findByName('test')

// 사용량 내림차순 정렬
const sorted = await manager.sortByUsage()
console.log('Top 3 spaces by usage:')
sorted.slice(0, 3).forEach((s, i) => {
  console.log(`${i + 1}. ${s.name}: ${(s.usage / 1024 / 1024 / 1024).toFixed(2)} GB`)
})
```

### 7.4 Space 간 데이터 복사

```typescript
/**
 * Space 간 데이터 복사
 */
async function copyBetweenSpaces(
  client: Client,
  sourceDID: string,
  targetDID: string,
  filter?: (upload: Upload) => boolean
) {
  console.log(`Copying from ${sourceDID} to ${targetDID}...`)

  // 1. 소스 Space 활성화
  await client.setCurrentSpace(sourceDID)

  // 2. 업로드 목록 조회
  const uploads = []
  for await (const upload of client.capability.upload.list({ space: sourceDID })) {
    if (!filter || filter(upload)) {
      uploads.push(upload)
    }
  }

  console.log(`Found ${uploads.length} uploads to copy`)

  // 3. 타겟 Space 활성화
  await client.setCurrentSpace(targetDID)

  // 4. 각 업로드 복사
  let copied = 0
  for (const upload of uploads) {
    try {
      // Shards 복사
      for (const shard of upload.shards) {
        await client.capability.blob.add(shard.multihash, shard.size)
      }

      // Upload 등록
      await client.capability.upload.add(upload.root, upload.shards)

      copied++
      console.log(`Copied ${copied}/${uploads.length}: ${upload.root}`)
    } catch (error) {
      console.error(`Failed to copy ${upload.root}:`, error.message)
    }
  }

  console.log(`Copy complete: ${copied}/${uploads.length} uploads`)
}

// 사용 예시: 최근 30일 데이터만 백업 Space로 복사
const thirtyDaysAgo = Date.now() - 86400 * 30 * 1000
await copyBetweenSpaces(
  client,
  prodSpace.did(),
  backupSpace.did(),
  (upload) => new Date(upload.insertedAt).getTime() > thirtyDaysAgo
)
```

---

## 8. Space 공유 및 권한 위임

### 8.1 다른 사용자에게 Space 공유

#### 8.1.1 읽기 전용 공유

```typescript
/**
 * 다른 사용자에게 읽기 전용 접근 권한 부여
 */
async function shareReadOnly(
  space: Space,
  recipientDID: string
): Promise<string> {
  // 1. 읽기 전용 Delegation 생성
  const delegation = await space.createDelegation({
    audience: DID.parse(recipientDID),
    capabilities: [
      { can: 'space/content/serve', with: space.did() },  // 컨텐츠 서빙
      { can: 'upload/list', with: space.did() }           // 목록 조회
    ],
    expiration: Math.floor(Date.now() / 1000) + 86400 * 90,  // 90일
    facts: [{
      name: space.name,
      description: 'Read-only access',
      permissions: 'read'
    }]
  })

  // 2. Delegation을 Base64로 내보내기
  const proofBase64 = Buffer.from(delegation.archive()).toString('base64')

  console.log(`Read-only delegation created for ${recipientDID}`)
  console.log('Share this proof with the recipient:')
  console.log(proofBase64)

  return proofBase64
}

// 사용 예시
const bobDID = 'did:key:z6MkrZ6sfBqyxnL...'
const readOnlyProof = await shareReadOnly(space, bobDID)

// Bob이 이 proof를 사용하여 Space에 접근
```

#### 8.1.2 쓰기 권한 공유

```typescript
/**
 * 업로드 권한을 포함한 쓰기 접근 권한 부여
 */
async function shareWriteAccess(
  space: Space,
  recipientDID: string,
  options: {
    canUpload?: boolean
    canDelete?: boolean
    expirationDays?: number
  } = {}
): Promise<string> {
  const {
    canUpload = true,
    canDelete = false,
    expirationDays = 30
  } = options

  // 1. 권한 목록 구성
  const capabilities: Array<{ can: string; with: string }> = []

  if (canUpload) {
    capabilities.push(
      { can: 'space/blob/add', with: space.did() },
      { can: 'space/index/add', with: space.did() },
      { can: 'upload/add', with: space.did() }
    )
  }

  if (canDelete) {
    capabilities.push(
      { can: 'upload/remove', with: space.did() }
    )
  }

  // 2. Delegation 생성
  const delegation = await space.createDelegation({
    audience: DID.parse(recipientDID),
    capabilities,
    expiration: Math.floor(Date.now() / 1000) + 86400 * expirationDays,
    facts: [{
      name: space.name,
      description: 'Write access',
      permissions: canDelete ? 'write+delete' : 'write'
    }]
  })

  const proofBase64 = Buffer.from(delegation.archive()).toString('base64')

  console.log(`Write delegation created for ${recipientDID}`)
  console.log(`Permissions: Upload=${canUpload}, Delete=${canDelete}`)
  console.log(`Expires in ${expirationDays} days`)

  return proofBase64
}

// 사용 예시
const carolDID = 'did:key:z6MksQ8RvN...'
const writeProof = await shareWriteAccess(space, carolDID, {
  canUpload: true,
  canDelete: false,
  expirationDays: 7
})
```

#### 8.1.3 전체 권한 위임

```typescript
/**
 * Space의 모든 권한을 위임 (관리자 권한)
 */
async function shareFullAccess(
  space: Space,
  recipientDID: string,
  expirationDays: number = 365
): Promise<string> {
  const delegation = await space.createDelegation({
    audience: DID.parse(recipientDID),
    capabilities: [
      { can: 'space/*', with: space.did() }  // 모든 Space 권한
    ],
    expiration: Math.floor(Date.now() / 1000) + 86400 * expirationDays,
    facts: [{
      name: space.name,
      description: 'Full admin access',
      permissions: 'admin'
    }]
  })

  const proofBase64 = Buffer.from(delegation.archive()).toString('base64')

  console.warn(`⚠️  Full access delegation created for ${recipientDID}`)
  console.warn('This grants ALL permissions. Use with caution!')

  return proofBase64
}
```

### 8.2 권한 Attenuation (축소)

#### 8.2.1 Capability Attenuation 패턴

```typescript
/**
 * 받은 Delegation을 더 제한적으로 재위임
 */
async function attenuateDelegation(
  client: Client,
  originalProof: Delegation,
  newRecipientDID: string
): Promise<Delegation> {
  // 1. 원본 Delegation의 권한 확인
  const originalCaps = originalProof.capabilities
  console.log('Original capabilities:', originalCaps)
  // [
  //   { can: 'space/*', with: 'did:key:z6Mk...' }
  // ]

  // 2. 축소된 권한으로 재위임
  const attenuatedDelegation = await client.createDelegation({
    audience: DID.parse(newRecipientDID),
    capabilities: [
      // space/* → space/blob/add 로 축소
      { can: 'space/blob/add', with: originalCaps[0].with }
    ],
    expiration: originalProof.expiration,  // 동일한 만료 시간
    proofs: [originalProof],  // 원본 Proof 체인 포함
    facts: [{
      description: 'Attenuated delegation (upload only)'
    }]
  })

  console.log('Attenuated capabilities:', attenuatedDelegation.capabilities)
  // [
  //   { can: 'space/blob/add', with: 'did:key:z6Mk...' }
  // ]

  return attenuatedDelegation
}
```

#### 8.2.2 시간 제한 Attenuation

```typescript
/**
 * 만료 시간을 더 짧게 축소
 */
async function attenuateExpiration(
  client: Client,
  originalProof: Delegation,
  newRecipientDID: string,
  newExpirationDays: number
): Promise<Delegation> {
  const originalExpiration = originalProof.expiration
  const newExpiration = Math.min(
    Math.floor(Date.now() / 1000) + 86400 * newExpirationDays,
    originalExpiration  // 원본 만료 시간을 초과할 수 없음
  )

  const delegation = await client.createDelegation({
    audience: DID.parse(newRecipientDID),
    capabilities: originalProof.capabilities,
    expiration: newExpiration,
    proofs: [originalProof]
  })

  console.log(`Original expiration: ${new Date(originalExpiration * 1000)}`)
  console.log(`New expiration: ${new Date(newExpiration * 1000)}`)

  return delegation
}
```

### 8.3 팀 협업 시나리오

#### 8.3.1 프로젝트 팀 설정

```typescript
/**
 * 팀 멤버별 역할에 따른 권한 부여
 */
interface TeamMember {
  did: string
  name: string
  role: 'owner' | 'admin' | 'developer' | 'viewer'
}

async function setupProjectTeam(
  space: Space,
  members: TeamMember[]
): Promise<Map<string, string>> {
  const delegations = new Map<string, string>()

  for (const member of members) {
    let proof: string

    switch (member.role) {
      case 'owner':
        // 소유자: 모든 권한 (영구)
        proof = await shareFullAccess(space, member.did, 3650)  // 10년
        break

      case 'admin':
        // 관리자: 모든 권한 (1년)
        proof = await shareFullAccess(space, member.did, 365)
        break

      case 'developer':
        // 개발자: 업로드/삭제 권한 (90일)
        proof = await shareWriteAccess(space, member.did, {
          canUpload: true,
          canDelete: true,
          expirationDays: 90
        })
        break

      case 'viewer':
        // 뷰어: 읽기 전용 (30일)
        proof = await shareReadOnly(space, member.did)
        break
    }

    delegations.set(member.did, proof)
    console.log(`✓ ${member.name} (${member.role}): Delegation created`)
  }

  return delegations
}

// 사용 예시
const team: TeamMember[] = [
  { did: 'did:key:z6MkA...', name: 'Alice', role: 'owner' },
  { did: 'did:key:z6MkB...', name: 'Bob', role: 'admin' },
  { did: 'did:key:z6MkC...', name: 'Carol', role: 'developer' },
  { did: 'did:key:z6MkD...', name: 'Dave', role: 'developer' },
  { did: 'did:key:z6MkE...', name: 'Eve', role: 'viewer' }
]

const proofs = await setupProjectTeam(space, team)

// 각 팀원에게 proof 전달 (이메일, 메시지 등)
```

#### 8.3.2 임시 접근 권한 (Contractor)

```typescript
/**
 * 계약자에게 제한된 시간 동안 권한 부여
 */
async function grantContractorAccess(
  space: Space,
  contractorDID: string,
  options: {
    projectName: string
    startDate: Date
    endDate: Date
    allowedOperations: string[]
  }
): Promise<string> {
  const { projectName, startDate, endDate, allowedOperations } = options

  // 1. 권한 매핑
  const capabilityMap: Record<string, string> = {
    'upload': 'space/blob/add',
    'download': 'space/content/serve',
    'list': 'upload/list',
    'delete': 'upload/remove'
  }

  const capabilities = allowedOperations.map(op => ({
    can: capabilityMap[op] ?? op,
    with: space.did()
  }))

  // 2. 시간 제한 Delegation 생성
  const delegation = await space.createDelegation({
    audience: DID.parse(contractorDID),
    capabilities,
    expiration: Math.floor(endDate.getTime() / 1000),
    notBefore: Math.floor(startDate.getTime() / 1000),  // 시작 시간 제한
    facts: [{
      description: `Contractor access for ${projectName}`,
      projectName,
      startDate: startDate.toISOString(),
      endDate: endDate.toISOString(),
      contractorType: 'temporary'
    }]
  })

  const proofBase64 = Buffer.from(delegation.archive()).toString('base64')

  console.log(`Contractor delegation created:`)
  console.log(`  Project: ${projectName}`)
  console.log(`  Valid: ${startDate.toISOString()} - ${endDate.toISOString()}`)
  console.log(`  Operations: ${allowedOperations.join(', ')}`)

  return proofBase64
}

// 사용 예시
const contractorProof = await grantContractorAccess(
  space,
  'did:key:z6MkContractor...',
  {
    projectName: 'Q1 Website Redesign',
    startDate: new Date('2025-01-15'),
    endDate: new Date('2025-03-31'),
    allowedOperations: ['upload', 'download', 'list']
  }
)
```

### 8.4 권한 추적 및 감사

#### 8.4.1 활성 Delegation 모니터링

```typescript
/**
 * Space의 모든 활성 Delegation 조회
 */
async function listActiveDelegations(
  client: Client,
  space: Space
): Promise<Array<DelegationInfo>> {
  const delegations: DelegationInfo[] = []

  // 1. Space에 대한 모든 Proof 조회
  const proofs = await client.proofs([
    { can: 'space/*', with: space.did() }
  ])

  const now = Math.floor(Date.now() / 1000)

  // 2. 각 Delegation 정보 추출
  for (const proof of proofs) {
    const isExpired = proof.expiration < now
    const daysRemaining = Math.floor((proof.expiration - now) / 86400)

    delegations.push({
      cid: proof.cid().toString(),
      issuer: proof.issuer.did(),
      audience: proof.audience.did(),
      capabilities: proof.capabilities,
      expiration: new Date(proof.expiration * 1000),
      isExpired,
      daysRemaining,
      facts: proof.facts
    })
  }

  // 3. 만료 시간 순 정렬
  delegations.sort((a, b) => a.expiration.getTime() - b.expiration.getTime())

  return delegations
}

interface DelegationInfo {
  cid: string
  issuer: string
  audience: string
  capabilities: Array<{ can: string; with: string }>
  expiration: Date
  isExpired: boolean
  daysRemaining: number
  facts?: any[]
}

// 사용 예시
const activeDelegations = await listActiveDelegations(client, space)

console.log(`\nActive Delegations for ${space.name}:\n`)
for (const delegation of activeDelegations) {
  console.log(`CID: ${delegation.cid.slice(0, 12)}...`)
  console.log(`  To: ${delegation.audience.slice(0, 20)}...`)
  console.log(`  Capabilities: ${delegation.capabilities.map(c => c.can).join(', ')}`)
  console.log(`  Expires: ${delegation.expiration.toISOString()}`)
  console.log(`  Status: ${delegation.isExpired ? '❌ EXPIRED' : `✓ Active (${delegation.daysRemaining} days left)`}`)
  console.log('')
}
```

#### 8.4.2 Delegation 사용 로깅

```typescript
/**
 * Delegation 사용 추적
 */
export class DelegationAuditLogger {
  private logs: AuditLog[] = []

  /**
   * Delegation 사용 기록
   */
  async logDelegationUse(
    delegation: Delegation,
    action: string,
    result: 'success' | 'failure',
    metadata?: any
  ): Promise<void> {
    this.logs.push({
      timestamp: new Date(),
      delegationCID: delegation.cid().toString(),
      audience: delegation.audience.did(),
      action,
      result,
      metadata
    })

    // 로그 저장 (DB, 파일, 외부 서비스 등)
    await this.persistLog(this.logs[this.logs.length - 1])
  }

  /**
   * 감사 로그 조회
   */
  async getAuditTrail(options: {
    audience?: string
    action?: string
    startDate?: Date
    endDate?: Date
  } = {}): Promise<AuditLog[]> {
    let filtered = this.logs

    if (options.audience) {
      filtered = filtered.filter(log => log.audience === options.audience)
    }

    if (options.action) {
      filtered = filtered.filter(log => log.action === options.action)
    }

    if (options.startDate) {
      filtered = filtered.filter(log => log.timestamp >= options.startDate!)
    }

    if (options.endDate) {
      filtered = filtered.filter(log => log.timestamp <= options.endDate!)
    }

    return filtered
  }

  /**
   * 로그 영구 저장
   */
  private async persistLog(log: AuditLog): Promise<void> {
    // 구현 예시: JSON 파일 저장
    // await fs.appendFile('audit.log', JSON.stringify(log) + '\n')
    console.log('[AUDIT]', JSON.stringify(log))
  }
}

interface AuditLog {
  timestamp: Date
  delegationCID: string
  audience: string
  action: string
  result: 'success' | 'failure'
  metadata?: any
}

// 사용 예시
const logger = new DelegationAuditLogger()

// Delegation 사용 시 로깅
await client.capability.blob.add(multihash, size)
await logger.logDelegationUse(
  delegation,
  'blob/add',
  'success',
  { size, multihash: multihash.toString() }
)

// 특정 사용자의 활동 조회
const bobActivity = await logger.getAuditTrail({
  audience: bobDID,
  startDate: new Date('2025-01-01')
})
```

---

## 9. 보안 고려사항

### 9.1 개인키 보안

#### 9.1.1 브라우저 환경

```typescript
/**
 * 브라우저에서 개인키 보호
 */

// ❌ 나쁜 예: localStorage에 평문 저장
localStorage.setItem('agent-key', keyBase64)  // 절대 금지!

// ✅ 좋은 예: IndexedDB + Web Crypto API 암호화
const store = new EncryptedStore('w3up-client')
await store.unlock(userPassword)
const client = await create({ store })
```

**보안 체크리스트:**

- ✅ IndexedDB 사용 (localStorage보다 안전)
- ✅ Web Crypto API로 암호화
- ✅ HTTPS 필수 (HTTP에서는 절대 사용 금지)
- ✅ Content Security Policy (CSP) 설정
- ❌ 콘솔에 키 출력 금지
- ❌ 에러 메시지에 키 포함 금지

#### 9.1.2 서버 환경

```typescript
/**
 * 서버에서 개인키 보호
 */

// ✅ 환경 변수 사용
const principal = Signer.parse(process.env.W3UP_PRIVATE_KEY)

// ✅ 파일 권한 설정
await fs.writeFile(keyPath, keyPEM, { mode: 0o600 })  // 소유자만 읽기/쓰기

// ✅ Secrets Manager 사용 (프로덕션)
import { SecretsManager } from '@aws-sdk/client-secrets-manager'

const secretsManager = new SecretsManager({ region: 'us-east-1' })
const secret = await secretsManager.getSecretValue({
  SecretId: 'w3up-agent-key'
})

const principal = Signer.parse(secret.SecretString)
```

**서버 보안 체크리스트:**

- ✅ Secrets Manager 사용 (AWS Secrets Manager, Vault, etc.)
- ✅ 환경 변수는 암호화된 저장소에서 로드
- ✅ 파일 시스템 권한 `0600` 또는 `0400`
- ✅ 로그에 키 출력 금지
- ✅ 키 회전 정책 수립
- ❌ 코드에 하드코딩 금지
- ❌ Git에 커밋 금지 (.gitignore 설정)

### 9.2 Delegation 보안

#### 9.2.1 최소 권한 원칙

```typescript
/**
 * 최소 권한 원칙 적용
 */

// ❌ 나쁜 예: 과도한 권한
const delegation = await space.createDelegation({
  audience: thirdPartyDID,
  capabilities: [
    { can: 'space/*', with: space.did() }  // 모든 권한!
  ],
  expiration: Infinity  // 영구!
})

// ✅ 좋은 예: 필요한 권한만
const delegation = await space.createDelegation({
  audience: thirdPartyDID,
  capabilities: [
    { can: 'upload/list', with: space.did() }  // 목록 조회만
  ],
  expiration: Math.floor(Date.now() / 1000) + 3600  // 1시간
})
```

#### 9.2.2 만료 시간 설정

```typescript
/**
 * 적절한 만료 시간 설정
 */

// 사용 사례별 만료 시간 가이드
const EXPIRATION_POLICY = {
  // 임시 접근 (계약자, 게스트)
  temporary: 86400 * 7,        // 7일

  // 일반 사용자 (팀원)
  regular: 86400 * 90,         // 90일

  // 장기 접근 (관리자)
  longTerm: 86400 * 365,       // 1년

  // 서비스 계정 (자동화)
  service: 86400 * 365 * 2,    // 2년

  // 영구 (Space → Account)
  permanent: Infinity
}

function getExpiration(role: string): number {
  const duration = EXPIRATION_POLICY[role] ?? EXPIRATION_POLICY.temporary
  return Math.floor(Date.now() / 1000) + duration
}

// 사용 예시
const delegation = await space.createDelegation({
  audience: userDID,
  capabilities: [...],
  expiration: getExpiration('regular')  // 90일
})
```

#### 9.2.3 Delegation 취소 전략

```typescript
/**
 * Delegation 취소 방법
 */

// 방법 1: 짧은 만료 시간 (Passive Revocation)
const shortLivedDelegation = await space.createDelegation({
  audience: userDID,
  capabilities: [...],
  expiration: Math.floor(Date.now() / 1000) + 3600  // 1시간 후 자동 만료
})

// 방법 2: Blocklist (Active Revocation)
class DelegationBlocklist {
  private blockedCIDs = new Set<string>()

  block(delegationCID: string): void {
    this.blockedCIDs.add(delegationCID)
    console.log(`Delegation ${delegationCID} revoked`)
  }

  isBlocked(delegationCID: string): boolean {
    return this.blockedCIDs.has(delegationCID)
  }

  async verifyDelegation(delegation: Delegation): Promise<boolean> {
    if (this.isBlocked(delegation.cid().toString())) {
      throw new Error('Delegation has been revoked')
    }
    return true
  }
}

// 방법 3: Agent DID 회전 (Emergency Revocation)
async function emergencyRevoke(
  client: Client,
  compromisedAgentDID: string
): Promise<void> {
  console.warn(`⚠️  Emergency revocation for ${compromisedAgentDID}`)

  // 1. 새 Agent 생성
  const newAgent = await EdSigner.generate()
  const newClient = await create({ principal: newAgent })

  // 2. 모든 Space를 새 Agent로 재위임
  const spaces = await client.spaces()
  for (const space of spaces) {
    const newDelegation = await space.createDelegation({
      audience: newAgent.did(),
      capabilities: [{ can: 'space/*', with: space.did() }],
      expiration: Math.floor(Date.now() / 1000) + 86400 * 365
    })
    await newClient.addSpace(newDelegation)
  }

  // 3. 이전 Agent 폐기
  console.log(`Old Agent ${compromisedAgentDID} is now invalid`)
  console.log(`New Agent: ${newAgent.did()}`)
}
```

### 9.3 네트워크 보안

#### 9.3.1 HTTPS 필수

```typescript
/**
 * Connection 설정 시 HTTPS 강제
 */
import { connect } from '@storacha/client'

// ✅ HTTPS 사용
const connection = connect({
  url: 'https://up.storacha.network',
  principal: servicePrincipal
})

// ❌ HTTP 사용 금지
// const connection = connect({
//   url: 'http://up.storacha.network',  // 절대 금지!
// })

const client = await create({ connection })
```

#### 9.3.2 요청 검증

```typescript
/**
 * 서버 측 요청 검증
 */
import { access } from '@ucanto/validator'

export async function validateInvocation(
  invocation: Invocation,
  context: ServiceContext
): Promise<void> {
  // 1. UCAN 서명 검증
  const result = await access(invocation, {
    capability: invocation.capability,
    principal: context.id,
    validateAuthorization: true
  })

  if (result.error) {
    throw new Error(`Authorization failed: ${result.error.message}`)
  }

  // 2. Delegation 체인 검증
  for (const proof of invocation.proofs) {
    const isValid = await proof.verify()
    if (!isValid) {
      throw new Error(`Invalid proof: ${proof.cid()}`)
    }
  }

  // 3. 만료 시간 검증
  const now = Math.floor(Date.now() / 1000)
  if (invocation.expiration && invocation.expiration < now) {
    throw new Error('Invocation has expired')
  }

  // 4. Rate limiting
  await context.rateLimiter.check(invocation.issuer.did())
}
```

### 9.4 데이터 보안

#### 9.4.1 클라이언트 측 암호화

```typescript
/**
 * 업로드 전 데이터 암호화
 */
import { webcrypto } from 'crypto'

export class EncryptedUploader {
  private encryptionKey: CryptoKey

  async initialize(password: string): Promise<void> {
    // PBKDF2로 암호화 키 생성
    const encoder = new TextEncoder()
    const keyMaterial = await webcrypto.subtle.importKey(
      'raw',
      encoder.encode(password),
      { name: 'PBKDF2' },
      false,
      ['deriveKey']
    )

    this.encryptionKey = await webcrypto.subtle.deriveKey(
      {
        name: 'PBKDF2',
        salt: encoder.encode('storacha-salt'),
        iterations: 100000,
        hash: 'SHA-256'
      },
      keyMaterial,
      { name: 'AES-GCM', length: 256 },
      false,
      ['encrypt', 'decrypt']
    )
  }

  /**
   * 파일 암호화 후 업로드
   */
  async uploadEncrypted(
    client: Client,
    file: File
  ): Promise<{ cid: string; iv: Uint8Array }> {
    // 1. 파일 읽기
    const fileBytes = new Uint8Array(await file.arrayBuffer())

    // 2. AES-GCM 암호화
    const iv = webcrypto.getRandomValues(new Uint8Array(12))
    const encrypted = await webcrypto.subtle.encrypt(
      { name: 'AES-GCM', iv },
      this.encryptionKey,
      fileBytes
    )

    // 3. 암호화된 데이터 업로드
    const encryptedFile = new File(
      [encrypted],
      `${file.name}.encrypted`,
      { type: 'application/octet-stream' }
    )

    const cid = await client.uploadFile(encryptedFile)

    console.log(`Encrypted file uploaded: ${cid}`)
    console.log(`IV (save this!): ${Buffer.from(iv).toString('base64')}`)

    return { cid, iv }
  }

  /**
   * 다운로드 후 복호화
   */
  async downloadDecrypted(
    cid: string,
    iv: Uint8Array
  ): Promise<Uint8Array> {
    // 1. 암호화된 데이터 다운로드
    const response = await fetch(`https://w3s.link/ipfs/${cid}`)
    const encrypted = await response.arrayBuffer()

    // 2. AES-GCM 복호화
    const decrypted = await webcrypto.subtle.decrypt(
      { name: 'AES-GCM', iv },
      this.encryptionKey,
      encrypted
    )

    return new Uint8Array(decrypted)
  }
}

// 사용 예시
const uploader = new EncryptedUploader()
await uploader.initialize('user-encryption-password')

const { cid, iv } = await uploader.uploadEncrypted(client, file)

// IV와 CID 저장 (메타데이터 DB 등)
await saveMetadata({ cid, iv: Buffer.from(iv).toString('base64') })

// 나중에 다운로드 및 복호화
const decryptedData = await uploader.downloadDecrypted(cid, iv)
```

---

## 10. Best Practices

### 10.1 Agent 관리

#### 10.1.1 장치별 Agent 분리

```typescript
/**
 * 각 장치마다 별도 Agent 사용
 */

// ✅ 좋은 예: 장치별 Agent
const browserAgent = await create()  // 브라우저
const serverAgent = await create()   // 서버
const mobileAgent = await create()   // 모바일

// ❌ 나쁜 예: 하나의 Agent를 모든 장치에서 공유
// 보안 위험! 한 장치가 손상되면 모든 장치가 위험
```

**이유:**
- 장치별 권한 추적 가능
- 선택적 취소 가능
- 보안 격리 (하나의 손상이 전체에 영향 없음)

#### 10.1.2 정기적인 키 백업

```typescript
/**
 * Agent 키 백업 자동화
 */
async function scheduleKeyBackup(client: Client): Promise<void> {
  const keyBackup = await client.agent.export()
  const keyBase64 = Buffer.from(keyBackup).toString('base64')

  // 안전한 저장소에 백업
  await backupToSecureStorage({
    agentDID: client.agent.did(),
    key: keyBase64,
    timestamp: new Date(),
    deviceInfo: {
      platform: navigator.platform,
      userAgent: navigator.userAgent
    }
  })

  console.log('✓ Agent key backed up successfully')
}

// 주기적 백업 (예: 매주)
setInterval(() => scheduleKeyBackup(client), 7 * 24 * 60 * 60 * 1000)
```

### 10.2 Space 관리

#### 10.2.1 명확한 네이밍 규칙

```typescript
/**
 * Space 네이밍 컨벤션
 */

// ✅ 좋은 예: 명확한 이름
await client.createSpace('prod-website-assets')
await client.createSpace('dev-api-testing')
await client.createSpace('backup-2025-q1')

// ❌ 나쁜 예: 모호한 이름
await client.createSpace('space1')
await client.createSpace('test')
await client.createSpace('untitled')
```

**권장 패턴:**
- `{environment}-{project}-{purpose}`
- 예: `prod-ecommerce-images`, `staging-blog-media`

#### 10.2.2 태그를 활용한 분류

```typescript
/**
 * Space 태그 활용
 */
const space = await client.createSpace('prod-api-logs', {
  meta: {
    tags: [
      'production',       // 환경
      'api',              // 프로젝트
      'logs',             // 데이터 유형
      'critical',         // 중요도
      '2025'              // 시간
    ],
    description: 'Production API logs for 2025',
    owner: 'platform-team',
    retention: '90-days'
  }
})

// 나중에 태그로 쉽게 검색
const prodSpaces = await spaceManager.findByTag('production')
const criticalSpaces = await spaceManager.findByTag('critical')
```

### 10.3 Delegation 관리

#### 10.3.1 Delegation 문서화

```typescript
/**
 * Delegation에 메타데이터 포함
 */
const delegation = await space.createDelegation({
  audience: contractorDID,
  capabilities: [{ can: 'upload/add', with: space.did() }],
  expiration: Math.floor(Date.now() / 1000) + 86400 * 30,
  facts: [{
    // 문서화
    purpose: 'Q1 content migration project',
    grantedBy: 'alice@example.com',
    grantedTo: 'contractor-bob@example.com',
    grantedAt: new Date().toISOString(),
    projectId: 'PROJ-2025-001',
    ticketNumber: 'JIRA-1234',
    approver: 'manager@example.com',

    // 제약사항
    restrictions: [
      'Upload only during business hours',
      'Maximum file size: 100MB',
      'No PII data allowed'
    ],

    // 연락처
    contact: {
      email: 'contractor-bob@example.com',
      slack: '@bob'
    }
  }]
})
```

#### 10.3.2 정기적인 Delegation 감사

```typescript
/**
 * 만료 예정 Delegation 알림
 */
async function auditDelegations(
  client: Client,
  space: Space
): Promise<void> {
  const delegations = await listActiveDelegations(client, space)
  const now = Date.now()

  // 7일 이내 만료 예정
  const expiringSoon = delegations.filter(d =>
    !d.isExpired &&
    d.expiration.getTime() - now < 7 * 24 * 60 * 60 * 1000
  )

  if (expiringSoon.length > 0) {
    console.warn(`⚠️  ${expiringSoon.length} delegations expiring soon:`)
    for (const d of expiringSoon) {
      console.warn(`  - ${d.audience} expires in ${d.daysRemaining} days`)
      // 알림 전송 (이메일, Slack 등)
      await sendExpiryNotification(d)
    }
  }

  // 만료된 Delegation 정리
  const expired = delegations.filter(d => d.isExpired)
  if (expired.length > 0) {
    console.log(`Cleaning up ${expired.length} expired delegations...`)
    // Blocklist에 추가 또는 로그 기록
  }
}

// 매일 실행
setInterval(() => auditDelegations(client, space), 24 * 60 * 60 * 1000)
```

### 10.4 에러 처리

#### 10.4.1 Delegation 에러 처리

```typescript
/**
 * Delegation 검증 에러 처리
 */
async function safeAddSpace(
  client: Client,
  proofBase64: string
): Promise<Space | null> {
  try {
    const space = await importDelegation(client, proofBase64)
    return space
  } catch (error) {
    if (error.message.includes('audience')) {
      console.error('Delegation is for a different agent')
      console.error(`Expected: ${client.agent.did()}`)
      return null
    }

    if (error.message.includes('expired')) {
      console.error('Delegation has expired')
      console.error('Please request a new delegation from the space owner')
      return null
    }

    if (error.message.includes('capabilities')) {
      console.error('Delegation does not contain space capabilities')
      return null
    }

    // 알 수 없는 에러
    console.error('Failed to add space:', error)
    throw error
  }
}
```

#### 10.4.2 업로드 에러 처리

```typescript
/**
 * 업로드 실패 시 재시도
 */
async function uploadWithRetry(
  client: Client,
  file: File,
  maxRetries: number = 3
): Promise<string> {
  let lastError: Error

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const cid = await client.uploadFile(file)
      console.log(`✓ Upload successful: ${cid}`)
      return cid
    } catch (error) {
      lastError = error

      // Delegation 에러는 재시도하지 않음
      if (error.message.includes('delegation') ||
          error.message.includes('proof') ||
          error.message.includes('authorization')) {
        throw error
      }

      // Quota 초과도 재시도하지 않음
      if (error.message.includes('quota') ||
          error.message.includes('storage limit')) {
        throw error
      }

      // 네트워크 에러는 재시도
      console.warn(`Upload attempt ${attempt}/${maxRetries} failed:`, error.message)

      if (attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000  // 지수 백오프
        console.log(`Retrying in ${delay}ms...`)
        await new Promise(resolve => setTimeout(resolve, delay))
      }
    }
  }

  throw new Error(`Upload failed after ${maxRetries} attempts: ${lastError.message}`)
}
```

### 10.5 성능 최적화

#### 10.5.1 Delegation 캐싱

```typescript
/**
 * Delegation 캐싱으로 성능 향상
 */
export class DelegationCache {
  private cache = new Map<string, { delegation: Delegation; expiry: number }>()

  async get(key: string): Promise<Delegation | null> {
    const cached = this.cache.get(key)
    if (!cached) return null

    // 만료 확인
    if (Date.now() > cached.expiry) {
      this.cache.delete(key)
      return null
    }

    return cached.delegation
  }

  async set(
    key: string,
    delegation: Delegation,
    ttl: number = 3600  // 1시간
  ): Promise<void> {
    this.cache.set(key, {
      delegation,
      expiry: Date.now() + ttl * 1000
    })
  }

  clear(): void {
    this.cache.clear()
  }
}

// 사용 예시
const delegationCache = new DelegationCache()

async function getCachedDelegation(
  space: Space,
  audience: string
): Promise<Delegation> {
  const cacheKey = `${space.did()}:${audience}`

  // 캐시 확인
  let delegation = await delegationCache.get(cacheKey)

  if (!delegation) {
    // 캐시 미스: 새로 생성
    delegation = await space.createDelegation({
      audience: DID.parse(audience),
      capabilities: [{ can: 'upload/add', with: space.did() }],
      expiration: Math.floor(Date.now() / 1000) + 3600
    })

    await delegationCache.set(cacheKey, delegation)
  }

  return delegation
}
```

---

## 결론

Storacha의 Space와 Agent 관리 시스템은 **UCAN 기반 권한 관리**를 통해:

1. **장치별 Identity 분리** (Agent)
2. **저장소 네임스페이스 격리** (Space)
3. **세밀한 권한 제어** (Delegation)
4. **안전한 권한 위임** (Attenuation)

을 제공합니다.

### 핵심 포인트

| 개념 | 역할 | 보안 |
|-----|------|------|
| **Agent** | 장치별 Identity | 개인키는 로컬 저장, 장치별 분리 |
| **Space** | 저장소 단위 | DID 기반 식별, Quota 격리 |
| **Delegation** | 권한 위임 토큰 | UCAN 서명, 만료 시간, Attenuation |
| **Account** | 사용자 계정 | Email 기반, 복구 가능 |

### 다음 단계

이 문서를 기반으로:
- **04_Upload_Service_Implementation.md**: 실제 업로드 서비스 구현
- **06_Client_Libraries_API.md**: w3up-client API 상세 가이드
- **Applications 문서**: 실제 애플리케이션 통합 예제

를 참고하세요.

---

**문서 작성:** 2025-01-14
**총 라인 수:** ~1,660 lines
**다음 문서:** 04_Upload_Service_Implementation.md
