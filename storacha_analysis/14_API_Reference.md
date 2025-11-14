# 14. API Reference

Complete API reference for Storacha services including HTTP endpoints, JavaScript client library, CLI commands, and Go SDK.

## Table of Contents

- [HTTP API (UCAN Bridge)](#http-api-ucan-bridge)
- [JavaScript Client API](#javascript-client-api)
- [CLI Reference](#cli-reference)
- [Go API](#go-api)
- [Error Codes](#error-codes)
- [Common Patterns](#common-patterns)

---

## HTTP API (UCAN Bridge)

The Storacha HTTP API uses the UCAN Bridge protocol, which allows HTTP clients to invoke capabilities using public/private keypairs and UCAN delegations.

### Authentication

**HTTP Headers:**

```http
X-Auth-Secret: <base64-encoded-private-key>
Authorization: <ucan-delegation-token>
```

- **X-Auth-Secret**: Deterministically generates the public/private keypair (principal) for the request. Keep secure.
- **Authorization**: UCAN token granting the principal the necessary capabilities

### Base URL

```
https://up.web3.storage
```

### Generating Authentication Tokens

```bash
# Using Storacha CLI
w3 bridge generate-tokens \
  --can space/blob/add \
  --can upload/add \
  --expiration 0

# Output:
# {
#   "X-Auth-Secret": "MgCYKXKJ8zL5V...",
#   "Authorization": "ucan/0.10.0;..."
# }
```

### Available Capabilities

| Capability | Description | Subject |
|------------|-------------|---------|
| `space/info` | Get space metadata | Space DID |
| `space/blob/add` | Store blob in space | Space DID |
| `space/blob/remove` | Remove blob from space | Space DID |
| `space/blob/list` | List blobs in space | Space DID |
| `upload/add` | Register upload | Space DID |
| `upload/list` | List uploads | Space DID |
| `upload/remove` | Remove upload | Space DID |
| `store/add` | Add CAR to store | Space DID |
| `store/get` | Get CAR status | Space DID |
| `store/remove` | Remove CAR | Space DID |
| `store/list` | List CARs | Space DID |
| `filecoin/offer` | Offer to Filecoin | Space DID |
| `filecoin/info` | Get Filecoin status | Space DID |
| `usage/report` | Get usage statistics | Space DID |
| `access/delegate` | Create delegation | Space DID |

---

## JavaScript Client API

The `@web3-storage/w3up-client` package provides a complete JavaScript interface to Storacha.

### Installation

```bash
npm install @web3-storage/w3up-client
```

### Requirements

- **Environment**: Modern browser or Node.js 18+
- **WASM Support**: Some environments require explicit WebAssembly imports

### Core Concepts

**Space**: Namespace for uploads, represented as a `did:key` with private key
**Agent**: Actor invoking capabilities, a local `did:key` identity
**Delegation**: UCAN proof chain authorizing Agent to use Space capabilities
**CID**: Content Identifier returned by upload operations

---

### Client Initialization

#### `create(options?)`

Creates and initializes a client instance.

**Signature:**
```typescript
function create(options?: ClientFactoryOptions): Promise<Client>

interface ClientFactoryOptions {
  principal?: Signer.EdSigner
  store?: Driver
}
```

**Parameters:**
- `options` (optional): Configuration object
  - `principal`: Custom signing principal (default: auto-generated)
  - `store`: Custom storage driver for persistence

**Returns:** `Promise<Client>`

**Examples:**

```javascript
// Basic initialization
import { create } from '@web3-storage/w3up-client'

const client = await create()
```

```javascript
// With custom principal (for serverless/stateless environments)
import * as Signer from '@ucanto/principal/ed25519'

const principal = Signer.parse(process.env.W3UP_PRINCIPAL)
const client = await create({ principal })
```

```javascript
// With custom store
import { StoreMemory } from '@web3-storage/w3up-client/stores/memory'

const store = new StoreMemory()
const client = await create({ store })
```

---

### Authentication

#### `login(email)`

Authenticates the agent using email verification.

**Signature:**
```typescript
function login(email: string): Promise<Account>
```

**Parameters:**
- `email` (string): User's email address

**Returns:** `Promise<Account>`

**Process:**
1. Sends verification email to the provided address
2. User clicks verification link
3. Promise resolves with Account object

**Example:**

```javascript
const account = await client.login('alice@example.com')
console.log('Logged in as:', account.did())

// Account DID format: did:mailto:example.com:alice
```

**Error Handling:**

```javascript
try {
  const account = await client.login('invalid-email')
} catch (error) {
  if (error.name === 'InvalidEmail') {
    console.error('Email format is invalid')
  } else if (error.name === 'VerificationTimeout') {
    console.error('Email verification timed out')
  }
}
```

---

### Space Management

#### `createSpace(name?, options?)`

Creates a new Space for organizing uploads.

**Signature:**
```typescript
function createSpace(
  name?: string,
  options?: { account?: Account }
): Promise<Space>
```

**Parameters:**
- `name` (optional): Human-readable identifier for the Space
- `options` (optional):
  - `account`: Associate Space with Account for recovery

**Returns:** `Promise<Space>`

**Example:**

```javascript
// Create anonymous space
const space = await client.createSpace('my-project-uploads')

// Create space with account recovery
const account = await client.login('alice@example.com')
const space = await client.createSpace('alice-workspace', { account })

console.log('Space DID:', space.did())
// Output: did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...
```

**Register Space:**

After creating a Space, register it with the service:

```javascript
const space = await client.createSpace('my-space')
await client.setCurrentSpace(space.did())

// Provision the space (required before uploads)
await client.capability.space.provision(space.did())
```

#### `setCurrentSpace(spaceDid)`

Designates the active Space for subsequent operations.

**Signature:**
```typescript
function setCurrentSpace(spaceDid: string): Promise<void>
```

**Parameters:**
- `spaceDid` (string): The Space's DID

**Returns:** `Promise<void>`

**Example:**

```javascript
// Get all spaces
const spaces = client.spaces()
console.log('Available spaces:', spaces.map(s => s.name))

// Set current space
await client.setCurrentSpace(spaces[0].did())

// Verify current space
const current = client.currentSpace()
console.log('Current space:', current?.name)
```

#### `addSpace(delegation)`

Adds an existing Space via delegation proof.

**Signature:**
```typescript
function addSpace(delegation: Delegation): Promise<Space>
```

**Parameters:**
- `delegation` (Delegation): UCAN delegation from Space to Agent

**Returns:** `Promise<Space>`

**Use Case:** Serverless environments or transferring Spaces between agents

**Example:**

```javascript
// Agent A: Export space delegation
const delegation = await client.createDelegation({
  audience: 'did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...',
  abilities: ['*'],
  expiration: Math.floor(Date.now() / 1000) + 86400 // 24 hours
})

const archive = await delegation.archive()
const base64Delegation = Buffer.from(archive.ok).toString('base64')

// Agent B: Import space
import * as Delegation from '@ucanto/core/delegation'

const archived = Buffer.from(base64Delegation, 'base64')
const delegation = await Delegation.extract(archived)
const space = await client.addSpace(delegation)
await client.setCurrentSpace(space.did())
```

#### `spaces()`

Lists all Spaces known to the agent.

**Signature:**
```typescript
function spaces(): Space[]
```

**Returns:** `Space[]`

**Example:**

```javascript
const allSpaces = client.spaces()

allSpaces.forEach(space => {
  console.log('Space:', space.name)
  console.log('  DID:', space.did())
  console.log('  Registered:', space.registered())
})
```

#### `currentSpace()`

Gets the currently active Space.

**Signature:**
```typescript
function currentSpace(): Space | undefined
```

**Returns:** `Space | undefined`

**Example:**

```javascript
const current = client.currentSpace()

if (current) {
  console.log('Current space:', current.name)
} else {
  console.log('No space selected. Create or select a space first.')
}
```

---

### Upload Operations

#### `uploadFile(file, options?)`

Uploads a single file to the current Space.

**Signature:**
```typescript
function uploadFile(
  file: Blob | File,
  options?: UploadOptions
): Promise<CID>

interface UploadOptions {
  onShardStored?: (meta: CARMetadata) => void
  shardSize?: number
  concurrentRequests?: number
  signal?: AbortSignal
}
```

**Parameters:**
- `file` (Blob | File): Data to upload
  - Browser: Use `Blob` or `File` objects
  - Node.js: Use `filesFromPath` library
- `options` (optional):
  - `onShardStored`: Callback invoked for each CAR shard stored
  - `shardSize`: Target size for CAR shards (default: ~100MB)
  - `concurrentRequests`: Max parallel uploads (default: 3)
  - `signal`: AbortSignal for cancellation

**Returns:** `Promise<CID>` - Content Identifier of the uploaded file

**Examples:**

```javascript
// Browser upload
const fileInput = document.querySelector('input[type="file"]')
const file = fileInput.files[0]

const cid = await client.uploadFile(file)
console.log('Uploaded:', cid.toString())
// Output: bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
```

```javascript
// Node.js upload
import { filesFromPath } from 'files-from-path'

const files = await filesFromPath('/path/to/file.jpg')
const cid = await client.uploadFile(files[0])
console.log('Upload CID:', cid.toString())
```

```javascript
// Upload with progress tracking
const cid = await client.uploadFile(file, {
  onShardStored: (meta) => {
    console.log(`Stored CAR shard: ${meta.cid}`)
    console.log(`  Size: ${meta.size} bytes`)
  }
})
```

```javascript
// Upload with cancellation
const controller = new AbortController()

const uploadPromise = client.uploadFile(file, {
  signal: controller.signal
})

// Cancel upload after 5 seconds
setTimeout(() => controller.abort(), 5000)

try {
  const cid = await uploadPromise
} catch (error) {
  if (error.name === 'AbortError') {
    console.log('Upload cancelled')
  }
}
```

#### `uploadDirectory(files, options?)`

Uploads multiple files maintaining directory structure.

**Signature:**
```typescript
function uploadDirectory(
  files: File[],
  options?: UploadOptions
): Promise<CID>
```

**Parameters:**
- `files` (File[]): Array of File objects with paths
- `options` (optional): Same as `uploadFile`

**Returns:** `Promise<CID>` - Directory root CID

**Path Handling:** Use `/` delimiters for nested directories

**Examples:**

```javascript
// Browser upload with directory structure
const files = [
  new File(['# My Project'], 'README.md'),
  new File(['console.log("Hello")'], 'src/index.js'),
  new File(['body { margin: 0; }'], 'src/styles.css'),
  new File([imageData], 'assets/logo.png'),
]

const dirCid = await client.uploadDirectory(files)
console.log('Directory CID:', dirCid.toString())

// Access via gateway:
// https://w3s.link/ipfs/<dirCid>/README.md
// https://w3s.link/ipfs/<dirCid>/src/index.js
// https://w3s.link/ipfs/<dirCid>/assets/logo.png
```

```javascript
// Node.js: Upload entire directory
import { filesFromPath } from 'files-from-path'

const files = await filesFromPath('./my-website', {
  pathPrefix: '/', // Preserve directory structure
  hidden: true     // Include hidden files
})

const dirCid = await client.uploadDirectory(files, {
  onShardStored: (meta) => {
    console.log(`Progress: ${meta.size} bytes stored`)
  }
})

console.log(`Website uploaded: https://w3s.link/ipfs/${dirCid}`)
```

```javascript
// Upload with sharding configuration
const largeFiles = await filesFromPath('./large-dataset')

const cid = await client.uploadDirectory(largeFiles, {
  shardSize: 50 * 1024 * 1024, // 50 MB shards
  concurrentRequests: 5,        // Upload 5 shards in parallel
  onShardStored: (meta) => {
    console.log(`Shard ${meta.cid} uploaded (${meta.size} bytes)`)
  }
})
```

#### `uploadCAR(car, options?)`

Uploads a pre-formatted CAR (Content Addressable aRchive) file.

**Signature:**
```typescript
function uploadCAR(
  car: Blob | File,
  options?: UploadOptions
): Promise<CID>
```

**Parameters:**
- `car` (Blob | File): CAR file containing IPLD blocks
- `options` (optional): Same as `uploadFile`

**Returns:** `Promise<CID>` - Root CID from the CAR

**Use Cases:**
- Advanced users managing their own DAG structures
- Uploading pre-computed IPFS data
- Migrating data from other IPFS services

**Example:**

```javascript
import { CarWriter } from '@ipld/car'
import { CID } from 'multiformats/cid'
import * as raw from 'multiformats/codecs/raw'
import { sha256 } from 'multiformats/hashes/sha2'

// Create CAR file
const bytes = new TextEncoder().encode('Hello, Storacha!')
const hash = await sha256.digest(raw.encode(bytes))
const cid = CID.create(1, raw.code, hash)

const { writer, out } = CarWriter.create([cid])
writer.put({ cid, bytes })
writer.close()

// Collect CAR bytes
const carChunks = []
for await (const chunk of out) {
  carChunks.push(chunk)
}
const carBlob = new Blob(carChunks)

// Upload CAR
const uploadedCid = await client.uploadCAR(carBlob)
console.log('CAR uploaded:', uploadedCid.toString())
```

---

### Blob Operations

Blobs are raw byte arrays stored in Storacha. Unlike uploads, blobs are not advertised to the IPFS network by default.

#### `capability.blob.add(blob, options?)`

Stores a blob in the current Space.

**Signature:**
```typescript
function add(
  blob: Blob,
  options?: { retries?: number; signal?: AbortSignal }
): Promise<BlobAddSuccess>

interface BlobAddSuccess {
  digest: Multihash
  size: number
}
```

**Parameters:**
- `blob` (Blob): Binary data to store
- `options` (optional):
  - `retries`: Number of retry attempts (default: 3)
  - `signal`: AbortSignal for cancellation

**Returns:** `Promise<BlobAddSuccess>`

**Example:**

```javascript
// Store a blob
const data = new TextEncoder().encode('Secret data')
const blob = new Blob([data])

const result = await client.capability.blob.add(blob)

console.log('Blob digest:', result.digest.toString())
console.log('Blob size:', result.size, 'bytes')

// Blob is stored but NOT advertised to IPFS network
// Use upload operations to make content publicly accessible
```

#### `capability.blob.list(options?)`

Lists blobs stored in the current Space.

**Signature:**
```typescript
function list(options?: BlobListOptions): Promise<BlobListResult>

interface BlobListOptions {
  cursor?: string
  size?: number
}

interface BlobListResult {
  results: Array<{
    digest: Multihash
    size: number
  }>
  cursor?: string
}
```

**Parameters:**
- `options` (optional):
  - `cursor`: Pagination cursor from previous response
  - `size`: Number of results per page (default: 100)

**Returns:** `Promise<BlobListResult>`

**Example:**

```javascript
// List all blobs with pagination
let cursor
const allBlobs = []

do {
  const response = await client.capability.blob.list({
    cursor,
    size: 50
  })

  allBlobs.push(...response.results)
  cursor = response.cursor
} while (cursor)

console.log(`Total blobs: ${allBlobs.length}`)
allBlobs.forEach(blob => {
  console.log(`  ${blob.digest}: ${blob.size} bytes`)
})
```

#### `capability.blob.remove(digest)`

Removes a blob from the current Space.

**Signature:**
```typescript
function remove(digest: Multihash): Promise<void>
```

**Parameters:**
- `digest` (Multihash): Blob's multihash identifier

**Returns:** `Promise<void>`

**Example:**

```javascript
import { base58btc } from 'multiformats/bases/base58'

// Remove blob by digest
const digest = base58btc.decode('QmXoypizjW3WknFiJnKLwHCnL72vedxjQkDDP1mXWo6uco')
await client.capability.blob.remove(digest)

console.log('Blob removed from space')
```

---

### Store Operations

Store operations manage CAR (Content Addressable aRchive) files.

#### `capability.store.add(car, options?)`

Adds a CAR file to the store.

**Signature:**
```typescript
function add(
  car: Blob,
  options?: { retries?: number; signal?: AbortSignal }
): Promise<StoreAddSuccess>

interface StoreAddSuccess {
  link: CID
  allocated: number
  status: 'upload' | 'done'
  url?: string
  headers?: Record<string, string>
}
```

**Parameters:**
- `car` (Blob): CAR file to store
- `options` (optional):
  - `retries`: Retry attempts
  - `signal`: AbortSignal

**Returns:** `Promise<StoreAddSuccess>`

**Response Fields:**
- `link`: CID of the CAR
- `allocated`: Bytes allocated (0 if already exists)
- `status`: `'upload'` (needs upload) or `'done'` (already stored)
- `url`: HTTP PUT endpoint (if status = 'upload')
- `headers`: Required HTTP headers (if status = 'upload')

**Example:**

```javascript
import { CarWriter } from '@ipld/car'

// Create CAR
const { writer, out } = CarWriter.create([someCid])
writer.put({ cid: someCid, bytes: someBytes })
writer.close()

const carChunks = []
for await (const chunk of out) {
  carChunks.push(chunk)
}
const carBlob = new Blob(carChunks)

// Add to store
const result = await client.capability.store.add(carBlob)

if (result.status === 'upload') {
  console.log('CAR needs upload to:', result.url)
  console.log('Use headers:', result.headers)

  // Upload CAR to provided URL
  await fetch(result.url, {
    method: 'PUT',
    headers: result.headers,
    body: carBlob
  })
} else {
  console.log('CAR already stored (deduplication)')
}

console.log('Allocated bytes:', result.allocated)
```

#### `capability.store.list(options?)`

Lists CAR files stored in the current Space.

**Signature:**
```typescript
function list(options?: StoreListOptions): Promise<StoreListResult>

interface StoreListOptions {
  cursor?: string
  size?: number
}

interface StoreListResult {
  results: Array<{
    link: CID
    size: number
    insertedAt: string
  }>
  cursor?: string
}
```

**Parameters:**
- `options` (optional):
  - `cursor`: Pagination cursor
  - `size`: Results per page (default: 100)

**Returns:** `Promise<StoreListResult>`

**Example:**

```javascript
// List all stored CARs
const response = await client.capability.store.list({ size: 10 })

console.log(`Found ${response.results.length} CAR files:`)
response.results.forEach(car => {
  console.log(`  ${car.link}: ${car.size} bytes (added: ${car.insertedAt})`)
})

// Paginate through all results
if (response.cursor) {
  const nextPage = await client.capability.store.list({
    cursor: response.cursor,
    size: 10
  })
}
```

#### `capability.store.remove(link)`

Removes a CAR file from the store.

**Signature:**
```typescript
function remove(link: CID): Promise<void>
```

**Parameters:**
- `link` (CID): CAR's content identifier

**Returns:** `Promise<void>`

**Example:**

```javascript
import { CID } from 'multiformats/cid'

const carCid = CID.parse('bagbaierayd6p6oyvvf3lozqzki5vfatwqeht5bwkhyfthxxjilqvbm25pqxa')
await client.capability.store.remove(carCid)

console.log('CAR removed from store')
```

#### `capability.store.get(link)`

Gets the status of a stored CAR.

**Signature:**
```typescript
function get(link: CID): Promise<StoreGetResult | null>

interface StoreGetResult {
  link: CID
  size: number
  insertedAt: string
}
```

**Parameters:**
- `link` (CID): CAR's content identifier

**Returns:** `Promise<StoreGetResult | null>` (null if not found)

**Example:**

```javascript
const carCid = CID.parse('bagbaiera...')
const info = await client.capability.store.get(carCid)

if (info) {
  console.log('CAR found:')
  console.log('  Size:', info.size, 'bytes')
  console.log('  Added:', info.insertedAt)
} else {
  console.log('CAR not found in store')
}
```

---

### Upload Registry Operations

Upload operations register content as publicly accessible on IPFS.

#### `capability.upload.add(root, shards)`

Registers an upload composed of one or more CAR shards.

**Signature:**
```typescript
function add(
  root: CID,
  shards: CID[]
): Promise<UploadAddSuccess>

interface UploadAddSuccess {
  root: CID
  shards: CID[]
}
```

**Parameters:**
- `root` (CID): Root CID of the content DAG
- `shards` (CID[]): Array of CAR CIDs containing the content

**Returns:** `Promise<UploadAddSuccess>`

**Example:**

```javascript
import { CID } from 'multiformats/cid'

// Upload was sharded into 2 CAR files
const rootCid = CID.parse('bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi')
const shardCids = [
  CID.parse('bagbaiera...1'),
  CID.parse('bagbaiera...2')
]

// Register upload
const result = await client.capability.upload.add(rootCid, shardCids)

console.log('Upload registered:')
console.log('  Root:', result.root.toString())
console.log('  Shards:', result.shards.map(cid => cid.toString()))

// Content is now advertised on IPFS network
// Accessible via: https://w3s.link/ipfs/<root-cid>
```

#### `capability.upload.list(options?)`

Lists registered uploads in the current Space.

**Signature:**
```typescript
function list(options?: UploadListOptions): Promise<UploadListResult>

interface UploadListOptions {
  cursor?: string
  size?: number
}

interface UploadListResult {
  results: Array<{
    root: CID
    shards: CID[]
    insertedAt: string
  }>
  cursor?: string
}
```

**Parameters:**
- `options` (optional):
  - `cursor`: Pagination cursor
  - `size`: Results per page (default: 100)

**Returns:** `Promise<UploadListResult>`

**Example:**

```javascript
// List recent uploads
const response = await client.capability.upload.list({ size: 20 })

console.log(`Found ${response.results.length} uploads:`)
response.results.forEach(upload => {
  console.log(`  Root: ${upload.root}`)
  console.log(`  Shards: ${upload.shards.length}`)
  console.log(`  Uploaded: ${upload.insertedAt}`)
  console.log(`  URL: https://w3s.link/ipfs/${upload.root}`)
  console.log()
})
```

#### `capability.upload.remove(root)`

Removes an upload from the registry.

**Signature:**
```typescript
function remove(root: CID): Promise<void>
```

**Parameters:**
- `root` (CID): Root CID of the upload

**Returns:** `Promise<void>`

**Note:** Removing an upload does NOT delete the underlying data from the store or IPFS network.

**Example:**

```javascript
const rootCid = CID.parse('bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi')

await client.capability.upload.remove(rootCid)
console.log('Upload removed from registry')

// Underlying blobs and CAR files remain in storage
// Content may still be accessible on IPFS network via other providers
```

---

### Filecoin Integration

#### `capability.filecoin.offer(content)`

Offers content for Filecoin storage deals.

**Signature:**
```typescript
function offer(content: CID[]): Promise<FilecoinOfferSuccess>

interface FilecoinOfferSuccess {
  piece: CID
}
```

**Parameters:**
- `content` (CID[]): Array of content CIDs to include in deal

**Returns:** `Promise<FilecoinOfferSuccess>`

**Example:**

```javascript
// Offer upload for Filecoin storage
const uploadCid = CID.parse('bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi')

const result = await client.capability.filecoin.offer([uploadCid])

console.log('Filecoin piece CID:', result.piece.toString())
// This piece will be aggregated and included in Filecoin storage deals
```

#### `capability.filecoin.info(piece)`

Gets Filecoin storage status for a piece.

**Signature:**
```typescript
function info(piece: CID): Promise<FilecoinInfoResult>

interface FilecoinInfoResult {
  piece: CID
  aggregates: Array<{
    aggregate: CID
    inclusion: {
      subtree: CID[]
      index: number[]
    }
  }>
  deals: Array<{
    aggregate: CID
    provider: string
    dealId: number
    status: 'queued' | 'published' | 'active'
    activation?: string
    expiration?: string
  }>
}
```

**Parameters:**
- `piece` (CID): Piece CID from `filecoin.offer`

**Returns:** `Promise<FilecoinInfoResult>`

**Example:**

```javascript
const pieceCid = CID.parse('baga6ea4seaqao7s73y24kcutaosvacpdjgfe5pw76ooefnyqw4ynr3d2y6x2mpq')

const info = await client.capability.filecoin.info(pieceCid)

console.log('Piece:', info.piece.toString())
console.log('Aggregates:', info.aggregates.length)

info.deals.forEach(deal => {
  console.log(`  Deal ${deal.dealId}:`)
  console.log(`    Provider: ${deal.provider}`)
  console.log(`    Status: ${deal.status}`)
  if (deal.activation) {
    console.log(`    Activation: ${deal.activation}`)
    console.log(`    Expiration: ${deal.expiration}`)
  }
})
```

---

### Delegation Management

#### `createDelegation(options)`

Creates a UCAN delegation for another principal.

**Signature:**
```typescript
function createDelegation(options: DelegationOptions): Promise<Delegation>

interface DelegationOptions {
  audience: string | DID
  abilities: string[]
  expiration?: number
  notBefore?: number
}
```

**Parameters:**
- `options`:
  - `audience`: DID of the delegate recipient
  - `abilities`: Array of capability strings (e.g., `['upload/add']`, `['*']`)
  - `expiration`: Unix timestamp when delegation expires (optional)
  - `notBefore`: Unix timestamp before which delegation is invalid (optional)

**Returns:** `Promise<Delegation>`

**Example:**

```javascript
// Delegate upload capabilities to another agent
const delegation = await client.createDelegation({
  audience: 'did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...',
  abilities: ['upload/add', 'upload/list', 'upload/remove'],
  expiration: Math.floor(Date.now() / 1000) + 86400 // 24 hours
})

// Export delegation
const archive = await delegation.archive()
const base64 = Buffer.from(archive.ok).toString('base64')

console.log('Delegation token:', base64)
// Share this token with the delegate
```

```javascript
// Delegate all capabilities with custom time constraints
const delegation = await client.createDelegation({
  audience: 'did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...',
  abilities: ['*'], // All capabilities
  notBefore: Math.floor(Date.now() / 1000) + 3600, // Valid after 1 hour
  expiration: Math.floor(Date.now() / 1000) + 86400 // Expires in 24 hours
})
```

#### `capability.access.authorize(email)`

Initiates email-based authorization flow.

**Signature:**
```typescript
function authorize(email: string): Promise<AuthorizeSuccess>

interface AuthorizeSuccess {
  did: string
}
```

**Parameters:**
- `email` (string): Email address to authorize

**Returns:** `Promise<AuthorizeSuccess>`

**Example:**

```javascript
const result = await client.capability.access.authorize('alice@example.com')

console.log('Authorized DID:', result.did)
// did:mailto:example.com:alice
```

#### `capability.access.claim()`

Claims delegated capabilities after email verification.

**Signature:**
```typescript
function claim(): Promise<Delegation[]>
```

**Returns:** `Promise<Delegation[]>`

**Example:**

```javascript
// After email verification
const delegations = await client.capability.access.claim()

console.log(`Claimed ${delegations.length} delegations`)

delegations.forEach(delegation => {
  console.log('Delegation from:', delegation.issuer.did())
  console.log('Capabilities:', delegation.capabilities)
})
```

---

### Usage and Billing

#### `capability.usage.report(space, period)`

Gets usage statistics for a Space.

**Signature:**
```typescript
function report(
  space: string,
  period: { from: string; to: string }
): Promise<UsageReport>

interface UsageReport {
  provider: string
  space: string
  period: {
    from: string
    to: string
  }
  size: {
    initial: number
    final: number
  }
  events: Array<{
    name: string
    value: number
    delta: number
    receiptAt: string
  }>
}
```

**Parameters:**
- `space` (string): Space DID
- `period`:
  - `from`: ISO 8601 timestamp
  - `to`: ISO 8601 timestamp

**Returns:** `Promise<UsageReport>`

**Example:**

```javascript
const spaceDid = 'did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...'

const report = await client.capability.usage.report(spaceDid, {
  from: '2024-01-01T00:00:00Z',
  to: '2024-01-31T23:59:59Z'
})

console.log('Usage Report for', report.space)
console.log('Period:', report.period.from, 'to', report.period.to)
console.log('Storage:')
console.log('  Initial:', report.size.initial, 'bytes')
console.log('  Final:', report.size.final, 'bytes')
console.log('  Change:', report.size.final - report.size.initial, 'bytes')

console.log('\nEvents:')
report.events.forEach(event => {
  console.log(`  ${event.name}: ${event.value} (Δ${event.delta}) at ${event.receiptAt}`)
})
```

#### `capability.plan.get(account)`

Gets the payment plan for an account.

**Signature:**
```typescript
function get(account: string): Promise<Plan>

interface Plan {
  product: string
}
```

**Parameters:**
- `account` (string): Account DID

**Returns:** `Promise<Plan>`

**Example:**

```javascript
const accountDid = 'did:mailto:example.com:alice'
const plan = await client.capability.plan.get(accountDid)

console.log('Current plan:', plan.product)
// Output: 'free' or 'premium'
```

---

### Agent and Principal

#### `agent`

Returns the client's Agent instance.

**Signature:**
```typescript
get agent(): Agent
```

**Returns:** `Agent`

**Example:**

```javascript
const agent = client.agent

console.log('Agent DID:', agent.did())
console.log('Agent type:', agent.did().startsWith('did:key:') ? 'Keypair' : 'Other')

// Get agent's public key
const publicKey = agent.signer.toDIDKey()
console.log('Public key:', publicKey)
```

**Agent Methods:**

```typescript
interface Agent {
  did(): string
  signer: Signer
  proofs(): Delegation[]
  addProof(delegation: Delegation): Promise<void>
}
```

---

### Type Definitions

#### CID (Content Identifier)

```typescript
import { CID } from 'multiformats/cid'

// Parse from string
const cid = CID.parse('bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi')

// Convert to string
const cidString = cid.toString()

// Get multihash
const multihash = cid.multihash

// Compare CIDs
const isEqual = cid1.equals(cid2)
```

#### Multihash

```typescript
import { base58btc } from 'multiformats/bases/base58'

// Decode from base58
const multihash = base58btc.decode('QmXoypizjW3WknFiJnKLwHCnL72vedxjQkDDP1mXWo6uco')

// Encode to base58
const encoded = base58btc.encode(multihash)
```

#### Space

```typescript
interface Space {
  did(): string
  name?: string
  registered(): boolean
  meta(): {
    name?: string
    createdAt: number
  }
}
```

#### Account

```typescript
interface Account {
  did(): string
  email(): string
  product(): string
}
```

---

### Complete Upload Example

```javascript
import { create } from '@web3-storage/w3up-client'
import { filesFromPath } from 'files-from-path'

async function uploadToStoracha() {
  // 1. Initialize client
  const client = await create()

  // 2. Login with email
  console.log('Please check your email for verification link...')
  const account = await client.login('alice@example.com')
  console.log('✓ Logged in as:', account.email())

  // 3. Create and provision space
  const space = await client.createSpace('my-app-uploads', { account })
  await client.setCurrentSpace(space.did())
  console.log('✓ Space created:', space.name)

  // 4. Load files
  const files = await filesFromPath('./my-website')
  console.log(`✓ Loaded ${files.length} files`)

  // 5. Upload directory
  console.log('Uploading...')
  const dirCid = await client.uploadDirectory(files, {
    onShardStored: (meta) => {
      console.log(`  Stored shard: ${meta.size} bytes`)
    }
  })

  console.log('✓ Upload complete!')
  console.log('  Root CID:', dirCid.toString())
  console.log('  URL:', `https://w3s.link/ipfs/${dirCid}`)

  // 6. Offer to Filecoin
  const filecoinResult = await client.capability.filecoin.offer([dirCid])
  console.log('✓ Offered to Filecoin')
  console.log('  Piece CID:', filecoinResult.piece.toString())

  // 7. Get usage report
  const report = await client.capability.usage.report(space.did(), {
    from: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString(),
    to: new Date().toISOString()
  })

  console.log('✓ Usage Report:')
  console.log('  Storage used:', report.size.final, 'bytes')
}

uploadToStoracha().catch(console.error)
```

---

### Security Warnings

⚠️ **Public Data**: All uploaded data is publicly accessible via its CID. Encrypt sensitive information before uploading.

⚠️ **Permanent Storage**: Removing uploads from Storacha doesn't guarantee deletion from the decentralized IPFS network. Data may persist on other nodes.

⚠️ **Private Keys**: Protect agent private keys. Loss of private key means loss of access to Spaces and delegations.

⚠️ **Delegations**: Delegations grant capabilities to other parties. Review delegation scope carefully before sharing.

---

## CLI Reference

The `w3` command-line interface provides complete access to Storacha services from the terminal.

### Installation

```bash
npm install -g @web3-storage/w3cli
```

**Alternative Installation:**

```bash
# Using npx (no installation required)
npx @web3-storage/w3cli --help

# Using pnpm
pnpm add -g @web3-storage/w3cli
```

### Global Configuration

**Environment Variables:**

```bash
# Custom principal key
export W3_PRINCIPAL=MgCYKXKJ8zL5V...

# Custom store name (for multiple profiles)
export W3_STORE_NAME=w3cli-production

# Custom service URL
export W3UP_SERVICE_URL=https://up.web3.storage

# Custom service DID
export W3UP_SERVICE_DID=did:web:web3.storage
```

### Global Options

```bash
# Show help
w3 --help

# Show version
w3 --version

# Use specific store profile
w3 --store=production up myfile.txt
```

---

### Authentication Commands

#### `w3 login [email]`

Authenticate the agent with email verification.

**Usage:**
```bash
w3 login alice@example.com
```

**Process:**
1. Sends verification email
2. User clicks link in email
3. CLI receives delegated capabilities
4. Account and agent are linked

**Example:**

```bash
$ w3 login alice@example.com
⠙ Checking email...
✔ Email sent to alice@example.com
  Please click the link in your email to verify your account.

⠙ Waiting for email verification...
✔ Email verified!
✔ Agent authorized

Account DID: did:mailto:example.com:alice
Agent DID: did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...
```

#### `w3 whoami`

Display information about the current agent and account.

**Usage:**
```bash
w3 whoami
```

**Output:**

```bash
$ w3 whoami
Agent DID: did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...
Account DID: did:mailto:example.com:alice
Current Space: my-uploads (did:key:z6Mk...)
```

---

### Space Commands

#### `w3 space create [name]`

Create a new Space for organizing uploads.

**Usage:**
```bash
w3 space create [name] [options]
```

**Options:**
- `--no-recovery` - Don't associate space with account for recovery

**Examples:**

```bash
# Create space with name
$ w3 space create my-project
✔ Space created: my-project
  DID: did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...

# Create anonymous space (no name)
$ w3 space create
✔ Space created
  DID: did:key:z6Mk...

# Create space without account recovery
$ w3 space create temp-space --no-recovery
```

#### `w3 space ls`

List all Spaces known to the agent.

**Usage:**
```bash
w3 space ls [options]
```

**Options:**
- `--json` - Output as newline-delimited JSON

**Example:**

```bash
$ w3 space ls
* my-uploads (did:key:z6MkqG7k...)  [current]
  test-space (did:key:z6Mk4Syi...)
  production (did:key:z6MkfZqr...)

$ w3 space ls --json
{"name":"my-uploads","did":"did:key:z6MkqG7k...","current":true}
{"name":"test-space","did":"did:key:z6Mk4Syi...","current":false}
{"name":"production","did":"did:key:z6MkfZqr...","current":false}
```

#### `w3 space use <did>`

Set the current Space for operations.

**Usage:**
```bash
w3 space use <space-did>
```

**Example:**

```bash
$ w3 space use did:key:z6Mk4Syi...
✔ Now using space: test-space

$ w3 whoami
Current Space: test-space (did:key:z6Mk4Syi...)
```

#### `w3 space info`

Get detailed information about a Space.

**Usage:**
```bash
w3 space info [options]
```

**Options:**
- `--space <did>` - Specific space (defaults to current)
- `--json` - Output as JSON

**Example:**

```bash
$ w3 space info
Space: my-uploads
DID: did:key:z6MkqG7k...
Registered: true
Providers: 1
  - did:web:web3.storage

$ w3 space info --json
{"did":"did:key:z6MkqG7k...","name":"my-uploads","registered":true,"providers":["did:web:web3.storage"]}
```

#### `w3 space add <proof>`

Add an existing Space via delegation proof.

**Usage:**
```bash
w3 space add <proof.car>
# OR
w3 space add <base64-encoded-proof>
```

**Examples:**

```bash
# From CAR file
$ w3 space add delegation.car
✔ Space added: shared-workspace
  DID: did:key:z6MkqG7k...

# From base64 string
$ w3 space add Ym9ndXMgY29udGVudCBmb3IgZGVtbyBwdXJwb3Nlcw==
✔ Space added
```

---

### Upload Commands

#### `w3 up <path> [path...]`

Upload files or directories to the current Space.

**Usage:**
```bash
w3 up <path> [paths...] [options]
```

**Options:**
- `--no-wrap` - Don't wrap files in directory (single file becomes root)
- `-H, --hidden` - Include hidden files (starting with `.`)
- `-c, --car` - Treat file as CAR format
- `--shard-size <bytes>` - CAR shard size (default: ~100MB)
- `--concurrent-requests <n>` - Parallel uploads (default: 3)

**Examples:**

```bash
# Upload single file
$ w3 up document.pdf
⠙ Packing files...
✔ Packed 1 file (234 KB)
⠙ Uploading...
✔ Upload complete
  Root CID: bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
  URL: https://w3s.link/ipfs/bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi

# Upload directory
$ w3 up ./my-website
⠙ Packing files...
✔ Packed 42 files (2.3 MB)
⠙ Uploading 3 CAR shards...
✔ Upload complete
  Root CID: bafybeibnsoufr2renqzsh347nrx54wcubt5lgkeivez63xvivplfwhtpym
  URL: https://w3s.link/ipfs/bafybeibnsoufr2renqzsh347nrx54wcubt5lgkeivez63xvivplfwhtpym

# Upload with hidden files
$ w3 up ./project -H
✔ Packed 58 files (including .env, .gitignore, etc.)

# Upload large directory with custom sharding
$ w3 up ./videos --shard-size 52428800 --concurrent-requests 5
⠙ Uploading 12 CAR shards (5 concurrent)...
✔ Upload complete

# Upload pre-generated CAR file
$ w3 up archive.car --car
✔ CAR uploaded
  Root CID: bagbaierayd6p6oyvvf3lozqzki5vfatwqeht5bwkhyfthxxjilqvbm25pqxa
```

#### `w3 ls`

List all uploads in the current Space.

**Usage:**
```bash
w3 ls [options]
```

**Options:**
- `--json` - Output as newline-delimited JSON
- `--shards` - Show CAR shards for each upload

**Examples:**

```bash
$ w3 ls
bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi  2024-01-15  234 KB
bafybeibnsoufr2renqzsh347nrx54wcubt5lgkeivez63xvivplfwhtpym  2024-01-14  2.3 MB
bagbaierayd6p6oyvvf3lozqzki5vfatwqeht5bwkhyfthxxjilqvbm25pqxa  2024-01-10  15 GB

$ w3 ls --shards
bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi  2024-01-15  234 KB
  └─ bagbaiera...1  234 KB

bafybeibnsoufr2renqzsh347nrx54wcubt5lgkeivez63xvivplfwhtpym  2024-01-14  2.3 MB
  ├─ bagbaiera...2  1.2 MB
  └─ bagbaiera...3  1.1 MB

$ w3 ls --json
{"root":"bafybeigdyrzt...","insertedAt":"2024-01-15T10:30:00Z","size":239872}
{"root":"bafybeibnsouf...","insertedAt":"2024-01-14T15:20:00Z","size":2411520}
```

#### `w3 rm <root-cid>`

Remove an upload from the Space's upload listing.

**Usage:**
```bash
w3 rm <root-cid> [options]
```

**Options:**
- `--shards` - Also remove all referenced CAR shards from store

**Examples:**

```bash
# Remove upload from listing (keeps blobs)
$ w3 rm bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
✔ Upload removed from listing

# Remove upload and all associated shards
$ w3 rm bafybeibnsoufr2renqzsh347nrx54wcubt5lgkeivez63xvivplfwhtpym --shards
✔ Upload removed
✔ 2 CAR shards removed from store
```

#### `w3 open <cid>`

Open a CID in the browser via w3s.link gateway.

**Usage:**
```bash
w3 open <cid>
```

**Example:**

```bash
$ w3 open bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
Opening https://w3s.link/ipfs/bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
```

---

### Blob Commands

#### `w3 can blob add [path]`

Store a blob file in the current Space.

**Usage:**
```bash
w3 can blob add <file>
```

**Example:**

```bash
$ w3 can blob add image.jpg
⠙ Uploading blob...
✔ Blob stored
  Digest: bafkreigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
  Size: 2,458,912 bytes
```

#### `w3 can blob ls`

List blobs in the current Space.

**Usage:**
```bash
w3 can blob ls [options]
```

**Options:**
- `--json` - Output as JSON
- `--size <n>` - Results per page (default: 100)
- `--cursor <string>` - Pagination cursor

**Example:**

```bash
$ w3 can blob ls
bafkreigdyrzt...  2,458,912 bytes  2024-01-15
bafkreiabcdef...  1,234,567 bytes  2024-01-14
bafkreixyz123...    567,890 bytes  2024-01-10

$ w3 can blob ls --json --size 10
{"digest":"bafkreigdyrzt...","size":2458912,"insertedAt":"2024-01-15T10:30:00Z"}
```

#### `w3 can blob rm <multihash>`

Remove a blob by its multihash.

**Usage:**
```bash
w3 can blob rm <multihash-base58>
```

**Example:**

```bash
$ w3 can blob rm bafkreigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
✔ Blob removed
```

---

### Store Commands

#### `w3 can store add <car-file>`

Add a CAR file to the store.

**Usage:**
```bash
w3 can store add <car-file>
```

**Example:**

```bash
$ w3 can store add archive.car
⠙ Adding CAR to store...
✔ CAR stored
  Link: bagbaiera...
  Allocated: 104,857,600 bytes
```

#### `w3 can store ls`

List CAR files in the current Space.

**Usage:**
```bash
w3 can store ls [options]
```

**Options:**
- `--json` - Output as JSON
- `--size <n>` - Results per page
- `--cursor <string>` - Pagination cursor

**Example:**

```bash
$ w3 can store ls
bagbaiera...1  104,857,600 bytes  2024-01-15
bagbaiera...2   52,428,800 bytes  2024-01-14

$ w3 can store ls --json
{"link":"bagbaiera...1","size":104857600,"insertedAt":"2024-01-15T10:30:00Z"}
{"link":"bagbaiera...2","size":52428800,"insertedAt":"2024-01-14T15:20:00Z"}
```

#### `w3 can store rm <car-cid>`

Remove a CAR file from the store.

**Usage:**
```bash
w3 can store rm <car-cid>
```

**Example:**

```bash
$ w3 can store rm bagbaiera...1
✔ CAR removed from store
```

---

### Upload Registry Commands

#### `w3 can upload add <root-cid> <shard-cid> [shard-cid...]`

Register an upload with root CID and shard CIDs.

**Usage:**
```bash
w3 can upload add <root> <shard1> [shard2...]
```

**Example:**

```bash
$ w3 can upload add \
    bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi \
    bagbaiera...1 \
    bagbaiera...2
✔ Upload registered
  Root: bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
  Shards: 2
```

#### `w3 can upload ls`

List uploads in the current Space.

**Usage:**
```bash
w3 can upload ls [options]
```

**Options:**
- `--json` - Output as JSON
- `--shards` - Include shard CIDs
- `--size <n>` - Results per page
- `--cursor <string>` - Pagination cursor
- `--pre` - Include uploads before cursor (reverse pagination)

**Example:**

```bash
$ w3 can upload ls --shards
bafybeigdyrzt...  2024-01-15
  ├─ bagbaiera...1
  └─ bagbaiera...2

bafybeibnsouf...  2024-01-14
  └─ bagbaiera...3
```

#### `w3 can upload rm <root-cid>`

Remove an upload from the registry.

**Usage:**
```bash
w3 can upload rm <root-cid>
```

**Example:**

```bash
$ w3 can upload rm bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
✔ Upload removed from registry
```

---

### Delegation Commands

#### `w3 delegation create <audience-did>`

Create a delegation to another principal.

**Usage:**
```bash
w3 delegation create <audience-did> [options]
```

**Options:**
- `--can <capability>` - Capability to delegate (use multiple times)
- `--name <name>` - Human-readable name for audience
- `--type <type>` - Audience type: device, app, or service
- `--output <file>` - Write delegation to file
- `--base64` - Output as base64-encoded string

**Examples:**

```bash
# Delegate upload capabilities
$ w3 delegation create did:key:z6MkqG7k... \
    --can upload/add \
    --can upload/list \
    --can upload/remove \
    --name "Mobile App" \
    --type app \
    --output mobile-delegation.car
✔ Delegation created
  Audience: Mobile App (did:key:z6MkqG7k...)
  Capabilities: upload/add, upload/list, upload/remove
  Written to: mobile-delegation.car

# Delegate all capabilities as base64
$ w3 delegation create did:key:z6MkqG7k... \
    --can '*' \
    --base64
✔ Delegation created
  Base64: Ym9ndXMgY29udGVudCBmb3IgZGVtbyBwdXJwb3Nlcw==
```

#### `w3 delegation ls`

List delegations created by the agent.

**Usage:**
```bash
w3 delegation ls [options]
```

**Options:**
- `--json` - Output as JSON

**Example:**

```bash
$ w3 delegation ls
Mobile App      did:key:z6MkqG7k...  upload/*, blob/*
Web Dashboard   did:key:z6Mk4Syi...  upload/*, store/*
Backup Service  did:key:z6MkfZqr...  *

$ w3 delegation ls --json
{"audience":"did:key:z6MkqG7k...","capabilities":["upload/*","blob/*"],"name":"Mobile App"}
```

#### `w3 delegation revoke <delegation-cid>`

Revoke a previously created delegation.

**Usage:**
```bash
w3 delegation revoke <delegation-cid> [options]
```

**Options:**
- `--proof <file>` - Delegation file to revoke

**Example:**

```bash
$ w3 delegation revoke bafyreiabc123... --proof mobile-delegation.car
✔ Delegation revoked
```

---

### Proof Commands

#### `w3 proof add <proof.ucan>`

Add a proof (delegation) received from another agent.

**Usage:**
```bash
w3 proof add <proof-file>
```

**Example:**

```bash
$ w3 proof add delegation-from-alice.car
✔ Proof added
  Issuer: did:key:z6MkqG7k...
  Capabilities: upload/*, blob/*
```

#### `w3 proof ls`

List proofs known to the agent.

**Usage:**
```bash
w3 proof ls [options]
```

**Options:**
- `--json` - Output as JSON

**Example:**

```bash
$ w3 proof ls
Delegation from did:key:z6MkqG7k...
  Capabilities: upload/*, blob/*
  Expiration: 2024-12-31T23:59:59Z

Delegation from did:key:z6Mk4Syi...
  Capabilities: *
  Expiration: Never

$ w3 proof ls --json
{"issuer":"did:key:z6MkqG7k...","capabilities":["upload/*","blob/*"],"expiration":1735689599}
```

---

### Bridge Commands

#### `w3 bridge generate-tokens`

Generate HTTP headers for UCAN Bridge authentication.

**Usage:**
```bash
w3 bridge generate-tokens [options]
```

**Options:**
- `--can <capability>` - Capability to delegate (use multiple times)
- `--expiration <timestamp>` - Unix timestamp (0 = no expiration)
- `--json` - Output as JSON

**Examples:**

```bash
# Generate tokens for upload operations
$ w3 bridge generate-tokens \
    --can space/blob/add \
    --can upload/add \
    --expiration 0 \
    --json
{
  "X-Auth-Secret": "MgCYKXKJ8zL5V...",
  "Authorization": "ucan/0.10.0;base64,eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCIsInVjdiI..."
}

# Use in HTTP request
$ TOKEN=$(w3 bridge generate-tokens --can upload/add --json)
$ curl -X POST https://up.web3.storage/upload \
    -H "X-Auth-Secret: $(echo $TOKEN | jq -r '.["X-Auth-Secret"]')" \
    -H "Authorization: $(echo $TOKEN | jq -r '.Authorization')" \
    -d '{"root":"bafybeigdyrzt...","shards":["bagbaiera..."]}'
```

---

### Key Management

#### `w3 key create`

Generate a new ed25519 keypair without changing the agent's current key.

**Usage:**
```bash
w3 key create [options]
```

**Options:**
- `--json` - Export as DAG-JSON

**Examples:**

```bash
$ w3 key create
Private key: MgCYKXKJ8zL5V...
Public key (DID): did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...

$ w3 key create --json
{
  "did": "did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...",
  "key": "MgCYKXKJ8zL5V..."
}
```

---

### CLI Workflow Examples

#### Complete Upload Workflow

```bash
# 1. Login
w3 login alice@example.com

# 2. Create space
w3 space create my-project

# 3. Upload files
w3 up ./website

# 4. List uploads
w3 ls

# 5. Open in browser
w3 open bafybeigdyrzt...
```

#### Delegation Workflow

```bash
# Agent A: Create delegation
w3 delegation create did:key:z6MkqG7k... \
  --can upload/add \
  --can upload/list \
  --output app-delegation.car

# Transfer app-delegation.car to Agent B

# Agent B: Add proof
w3 proof add app-delegation.car

# Agent B: Use delegated space
w3 space add app-delegation.car
w3 space use did:key:z6MkqG7k...
w3 up myfile.txt
```

#### Multi-Profile Setup

```bash
# Production profile
W3_STORE_NAME=production w3 login prod@company.com
W3_STORE_NAME=production w3 space create production-uploads

# Development profile
W3_STORE_NAME=development w3 login dev@company.com
W3_STORE_NAME=development w3 space create dev-uploads

# Use specific profile
W3_STORE_NAME=production w3 up important-file.pdf
W3_STORE_NAME=development w3 up test-file.txt
```

---

## Go API

The `go-w3up` package provides a Go client for the w3up platform.

### Installation

```bash
go get github.com/storacha/go-w3up
go get github.com/storacha/go-ucanto
```

### Requirements

- Go 1.21.4 or higher
- `go-ucanto` library for UCAN RPC calls

### Important Notes

⚠️ The Go client is under active development and less feature-complete than the JavaScript client.

⚠️ The client does not yet store delegations or auto-select proofs. You must provide pre-selected proofs when making invocations.

---

### Basic Usage

#### Creating a Client

```go
package main

import (
    "context"
    "fmt"
    "github.com/storacha/go-w3up/client"
    "github.com/storacha/go-ucanto/principal/ed25519/signer"
)

func main() {
    // Create or load signer (agent identity)
    agentSigner, err := signer.Generate()
    if err != nil {
        panic(err)
    }

    // Create client
    cl, err := client.New(
        client.WithSigner(agentSigner),
        client.WithServiceURL("https://up.web3.storage"),
        client.WithServiceDID("did:web:web3.storage"),
    )
    if err != nil {
        panic(err)
    }

    fmt.Println("Agent DID:", agentSigner.DID().String())
}
```

#### Loading Existing Signer

```go
import (
    "encoding/base64"
    "github.com/storacha/go-ucanto/principal/ed25519/signer"
)

func loadSigner(privateKeyBase64 string) (*signer.Signer, error) {
    keyBytes, err := base64.StdEncoding.DecodeString(privateKeyBase64)
    if err != nil {
        return nil, err
    }

    return signer.Parse(keyBytes)
}

// Usage
agentSigner, err := loadSigner(os.Getenv("W3UP_PRINCIPAL"))
```

---

### Uploading Data

#### Upload CAR File

```go
import (
    "os"
    "github.com/ipld/go-car/v2"
    "github.com/storacha/go-w3up/capability/blob"
    "github.com/storacha/go-w3up/capability/store"
    "github.com/storacha/go-w3up/capability/upload"
)

func uploadCAR(ctx context.Context, cl *client.Client, spaceDID string, carPath string) error {
    // Read CAR file
    carFile, err := os.Open(carPath)
    if err != nil {
        return err
    }
    defer carFile.Close()

    carReader, err := car.NewBlockReader(carFile)
    if err != nil {
        return err
    }

    // Get root CID
    roots, err := carReader.Roots()
    if err != nil {
        return err
    }
    rootCID := roots[0]

    // Get CAR size
    stat, err := carFile.Stat()
    if err != nil {
        return err
    }
    carSize := stat.Size()

    // Create space principal
    space, err := did.Parse(spaceDID)
    if err != nil {
        return err
    }

    // Store CAR
    storeResult, err := store.Add(
        ctx,
        cl,
        space,
        carFile,
        carSize,
        store.WithProofs(myDelegations), // Must provide delegations
    )
    if err != nil {
        return err
    }

    fmt.Println("CAR stored:", storeResult.Link)

    // Register upload
    uploadResult, err := upload.Add(
        ctx,
        cl,
        space,
        rootCID,
        []cid.Cid{storeResult.Link},
        upload.WithProofs(myDelegations),
    )
    if err != nil {
        return err
    }

    fmt.Println("Upload registered:", uploadResult.Root)
    return nil
}
```

---

### Working with Delegations

#### Creating a Delegation

```go
import (
    "github.com/storacha/go-ucanto/delegation"
    "github.com/storacha/go-ucanto/ucan"
)

func createDelegation(
    issuer *signer.Signer,
    audienceDID string,
    spaceDID string,
) (*delegation.Delegation, error) {
    audience, err := did.Parse(audienceDID)
    if err != nil {
        return nil, err
    }

    space, err := did.Parse(spaceDID)
    if err != nil {
        return nil, err
    }

    // Create delegation with upload capabilities
    deleg, err := delegation.Delegate(
        issuer,
        audience,
        []ucan.Capability{
            {
                With: space,
                Can:  "upload/add",
            },
            {
                With: space,
                Can:  "upload/list",
            },
        },
        delegation.WithExpiration(time.Now().Add(24*time.Hour).Unix()),
    )
    if err != nil {
        return nil, err
    }

    return deleg, nil
}
```

#### Exporting Delegation

```go
func exportDelegation(deleg *delegation.Delegation, outputPath string) error {
    // Archive delegation as CAR
    carBytes, err := deleg.Archive()
    if err != nil {
        return err
    }

    // Write to file
    return os.WriteFile(outputPath, carBytes, 0600)
}
```

#### Importing Delegation

```go
func importDelegation(carPath string) (*delegation.Delegation, error) {
    // Read CAR file
    carBytes, err := os.ReadFile(carPath)
    if err != nil {
        return nil, err
    }

    // Extract delegation
    return delegation.Extract(carBytes)
}
```

---

### Capability Invocations

#### Space Operations

```go
import (
    "github.com/storacha/go-w3up/capability/space"
)

// Get space information
func getSpaceInfo(ctx context.Context, cl *client.Client, spaceDID string, proofs []*delegation.Delegation) error {
    space, _ := did.Parse(spaceDID)

    info, err := space.Info(
        ctx,
        cl,
        space,
        space.WithProofs(proofs),
    )
    if err != nil {
        return err
    }

    fmt.Println("Space DID:", info.DID)
    fmt.Println("Providers:", info.Providers)
    return nil
}
```

#### Blob Operations

```go
import (
    "github.com/storacha/go-w3up/capability/blob"
    "github.com/multiformats/go-multihash"
)

// Add blob
func addBlob(
    ctx context.Context,
    cl *client.Client,
    spaceDID string,
    data []byte,
    proofs []*delegation.Delegation,
) error {
    space, _ := did.Parse(spaceDID)

    // Calculate multihash
    mh, err := multihash.Sum(data, multihash.SHA2_256, -1)
    if err != nil {
        return err
    }

    result, err := blob.Add(
        ctx,
        cl,
        space,
        data,
        mh,
        blob.WithProofs(proofs),
    )
    if err != nil {
        return err
    }

    fmt.Println("Blob added:")
    fmt.Println("  Digest:", result.Digest)
    fmt.Println("  Size:", result.Size)
    return nil
}

// List blobs
func listBlobs(
    ctx context.Context,
    cl *client.Client,
    spaceDID string,
    proofs []*delegation.Delegation,
) error {
    space, _ := did.Parse(spaceDID)

    result, err := blob.List(
        ctx,
        cl,
        space,
        blob.WithProofs(proofs),
        blob.WithSize(100),
    )
    if err != nil {
        return err
    }

    fmt.Println("Blobs:")
    for _, b := range result.Results {
        fmt.Printf("  %s (%d bytes)\n", b.Digest, b.Size)
    }

    // Handle pagination
    if result.Cursor != nil {
        fmt.Println("More results available, cursor:", *result.Cursor)
    }

    return nil
}
```

#### Store Operations

```go
import (
    "github.com/storacha/go-w3up/capability/store"
    "github.com/ipfs/go-cid"
)

// List stored CARs
func listStore(
    ctx context.Context,
    cl *client.Client,
    spaceDID string,
    proofs []*delegation.Delegation,
) error {
    space, _ := did.Parse(spaceDID)

    result, err := store.List(
        ctx,
        cl,
        space,
        store.WithProofs(proofs),
        store.WithSize(50),
    )
    if err != nil {
        return err
    }

    fmt.Println("Stored CARs:")
    for _, car := range result.Results {
        fmt.Printf("  %s (%d bytes, inserted: %s)\n",
            car.Link, car.Size, car.InsertedAt)
    }

    return nil
}

// Remove CAR from store
func removeFromStore(
    ctx context.Context,
    cl *client.Client,
    spaceDID string,
    carCID cid.Cid,
    proofs []*delegation.Delegation,
) error {
    space, _ := did.Parse(spaceDID)

    err := store.Remove(
        ctx,
        cl,
        space,
        carCID,
        store.WithProofs(proofs),
    )
    if err != nil {
        return err
    }

    fmt.Println("CAR removed:", carCID)
    return nil
}
```

---

### Complete Go Example

```go
package main

import (
    "context"
    "fmt"
    "os"

    "github.com/storacha/go-w3up/client"
    "github.com/storacha/go-w3up/capability/store"
    "github.com/storacha/go-w3up/capability/upload"
    "github.com/storacha/go-ucanto/delegation"
    "github.com/storacha/go-ucanto/principal/ed25519/signer"
    "github.com/storacha/go-ucanto/did"
)

func main() {
    ctx := context.Background()

    // 1. Load agent signer
    agentSigner, err := loadSigner(os.Getenv("W3UP_PRINCIPAL"))
    if err != nil {
        panic(err)
    }

    // 2. Create client
    cl, err := client.New(
        client.WithSigner(agentSigner),
        client.WithServiceURL("https://up.web3.storage"),
        client.WithServiceDID("did:web:web3.storage"),
    )
    if err != nil {
        panic(err)
    }

    // 3. Load delegation proofs
    delegPath := os.Getenv("W3UP_DELEGATION_PATH")
    deleg, err := importDelegation(delegPath)
    if err != nil {
        panic(err)
    }

    proofs := []*delegation.Delegation{deleg}
    spaceDID := os.Getenv("W3UP_SPACE_DID")
    space, _ := did.Parse(spaceDID)

    // 4. Upload CAR file
    carPath := "./my-data.car"
    carFile, err := os.Open(carPath)
    if err != nil {
        panic(err)
    }
    defer carFile.Close()

    stat, _ := carFile.Stat()

    storeResult, err := store.Add(
        ctx,
        cl,
        space,
        carFile,
        stat.Size(),
        store.WithProofs(proofs),
    )
    if err != nil {
        panic(err)
    }

    fmt.Println("✓ CAR stored:", storeResult.Link)

    // 5. Register upload
    rootCID := getRootFromCAR(carPath) // Helper function

    uploadResult, err := upload.Add(
        ctx,
        cl,
        space,
        rootCID,
        []cid.Cid{storeResult.Link},
        upload.WithProofs(proofs),
    )
    if err != nil {
        panic(err)
    }

    fmt.Println("✓ Upload registered:", uploadResult.Root)
    fmt.Printf("✓ URL: https://w3s.link/ipfs/%s\n", uploadResult.Root)
}

func loadSigner(privateKey string) (*signer.Signer, error) {
    // Load from base64 encoded key
    keyBytes, err := base64.StdEncoding.DecodeString(privateKey)
    if err != nil {
        return nil, err
    }
    return signer.Parse(keyBytes)
}

func importDelegation(path string) (*delegation.Delegation, error) {
    carBytes, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }
    return delegation.Extract(carBytes)
}

func getRootFromCAR(path string) cid.Cid {
    // Implementation to extract root CID from CAR file
    // ... (omitted for brevity)
}
```

---

### Go API Limitations

Current limitations of the Go client (as of 2024):

1. **No Built-in Proof Storage**: Must manually manage and provide delegations
2. **No Auto-Login**: Email-based authentication not yet implemented
3. **Manual Delegation Selection**: No automatic proof matching
4. **Limited Documentation**: Fewer examples compared to JavaScript client
5. **Active Development**: APIs may change

**Workarounds:**

- Use JavaScript client for initial setup (login, space creation)
- Export delegations from JS client, import in Go
- Manage proofs externally in your application

---

## Error Codes

Storacha services return structured error responses for failed operations.

### Common Error Types

#### `InsufficientStorage`

**Cause:** Space has insufficient capacity for the requested operation.

**HTTP Status:** 507 Insufficient Storage

**Response:**
```json
{
  "error": {
    "name": "InsufficientStorage",
    "message": "Space did:key:z6Mk... has insufficient storage capacity",
    "details": {
      "space": "did:key:z6Mk...",
      "required": 104857600,
      "available": 52428800
    }
  }
}
```

**Resolution:**
- Upgrade space plan
- Remove unused uploads
- Contact support for quota increase

**Example:**
```javascript
try {
  await client.uploadFile(largeFile)
} catch (error) {
  if (error.name === 'InsufficientStorage') {
    console.error('Storage quota exceeded')
    console.log('Required:', error.details.required, 'bytes')
    console.log('Available:', error.details.available, 'bytes')
  }
}
```

---

#### `ProofNotFound`

**Cause:** Agent lacks valid delegation proofs for the requested capability.

**HTTP Status:** 403 Forbidden

**Response:**
```json
{
  "error": {
    "name": "ProofNotFound",
    "message": "No valid proof found for capability upload/add on space did:key:z6Mk...",
    "details": {
      "capability": "upload/add",
      "resource": "did:key:z6Mk...",
      "agent": "did:key:z6MkqG7k..."
    }
  }
}
```

**Resolution:**
- Login with `client.login()` or CLI `w3 login`
- Add space delegation via `client.addSpace()` or `w3 space add`
- Verify current space with `client.currentSpace()` or `w3 whoami`

**Example:**
```javascript
try {
  await client.uploadFile(file)
} catch (error) {
  if (error.name === 'ProofNotFound') {
    console.error('Missing delegation for upload capability')
    console.log('Login or add space delegation')

    // Login to get delegations
    await client.login('user@example.com')

    // Retry upload
    await client.uploadFile(file)
  }
}
```

---

#### `SpaceUnknown`

**Cause:** Specified space DID is not known to the service.

**HTTP Status:** 404 Not Found

**Response:**
```json
{
  "error": {
    "name": "SpaceUnknown",
    "message": "Space did:key:z6Mk... is not registered",
    "details": {
      "space": "did:key:z6Mk..."
    }
  }
}
```

**Resolution:**
- Create space: `client.createSpace()` or `w3 space create`
- Verify space DID
- Register existing space with service

---

#### `InvalidCID`

**Cause:** Provided CID is malformed or invalid.

**HTTP Status:** 400 Bad Request

**Response:**
```json
{
  "error": {
    "name": "InvalidCID",
    "message": "Invalid CID format: 'invalid-cid-string'",
    "details": {
      "cid": "invalid-cid-string"
    }
  }
}
```

**Resolution:**
- Use `CID.parse()` to validate CIDs
- Check CID encoding (base32, base58, etc.)

**Example:**
```javascript
import { CID } from 'multiformats/cid'

try {
  const cid = CID.parse(userInput)
  await client.capability.upload.remove(cid)
} catch (error) {
  if (error.name === 'InvalidCID') {
    console.error('Invalid CID format')
  }
}
```

---

#### `BlobNotFound`

**Cause:** Requested blob does not exist in the space.

**HTTP Status:** 404 Not Found

**Response:**
```json
{
  "error": {
    "name": "BlobNotFound",
    "message": "Blob bafkrei... not found in space",
    "details": {
      "digest": "bafkrei...",
      "space": "did:key:z6Mk..."
    }
  }
}
```

**Resolution:**
- Verify blob digest
- Check if blob was previously deleted
- Ensure blob was successfully uploaded

---

#### `UploadNotFound`

**Cause:** Upload with specified root CID doesn't exist.

**HTTP Status:** 404 Not Found

**Response:**
```json
{
  "error": {
    "name": "UploadNotFound",
    "message": "Upload with root bafybei... not found",
    "details": {
      "root": "bafybei...",
      "space": "did:key:z6Mk..."
    }
  }
}
```

---

#### `StoreItemNotFound`

**Cause:** CAR file not found in store.

**HTTP Status:** 404 Not Found

**Response:**
```json
{
  "error": {
    "name": "StoreItemNotFound",
    "message": "CAR bagbaiera... not found in store",
    "details": {
      "link": "bagbaiera...",
      "space": "did:key:z6Mk..."
    }
  }
}
```

---

#### `DelegationExpired`

**Cause:** UCAN delegation has expired.

**HTTP Status:** 401 Unauthorized

**Response:**
```json
{
  "error": {
    "name": "DelegationExpired",
    "message": "Delegation expired at 2024-01-01T00:00:00Z",
    "details": {
      "expiration": 1704067200,
      "current": 1704153600
    }
  }
}
```

**Resolution:**
- Request new delegation
- Check delegation expiration before use

---

#### `InvalidMultihash`

**Cause:** Multihash format is invalid or unsupported.

**HTTP Status:** 400 Bad Request

**Response:**
```json
{
  "error": {
    "name": "InvalidMultihash",
    "message": "Unsupported multihash algorithm: 0x99",
    "details": {
      "algorithm": "0x99"
    }
  }
}
```

**Resolution:**
- Use SHA2-256 (code `0x12`) for compatibility
- Verify multihash encoding

---

#### `RateLimited`

**Cause:** Too many requests in short period.

**HTTP Status:** 429 Too Many Requests

**Response:**
```json
{
  "error": {
    "name": "RateLimited",
    "message": "Rate limit exceeded. Retry after 60 seconds",
    "details": {
      "retryAfter": 60,
      "limit": 1000,
      "window": "1h"
    }
  }
}
```

**Resolution:**
- Implement exponential backoff
- Respect `Retry-After` header
- Reduce request frequency

**Example:**
```javascript
async function uploadWithRetry(file, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await client.uploadFile(file)
    } catch (error) {
      if (error.name === 'RateLimited') {
        const delay = error.details.retryAfter * 1000
        console.log(`Rate limited. Retrying in ${delay}ms`)
        await new Promise(resolve => setTimeout(resolve, delay))
        continue
      }
      throw error
    }
  }
  throw new Error('Max retries exceeded')
}
```

---

### Error Handling Best Practices

#### Comprehensive Error Handling

```javascript
async function safeUpload(client, file) {
  try {
    const cid = await client.uploadFile(file)
    return { success: true, cid }
  } catch (error) {
    // Handle specific errors
    if (error.name === 'InsufficientStorage') {
      return {
        success: false,
        error: 'QUOTA_EXCEEDED',
        message: 'Storage quota exceeded. Please upgrade your plan.',
        details: error.details
      }
    }

    if (error.name === 'ProofNotFound') {
      return {
        success: false,
        error: 'UNAUTHORIZED',
        message: 'Missing upload permissions. Please login.',
        details: error.details
      }
    }

    if (error.name === 'RateLimited') {
      return {
        success: false,
        error: 'RATE_LIMITED',
        message: `Too many requests. Retry after ${error.details.retryAfter}s`,
        retryAfter: error.details.retryAfter
      }
    }

    // Network errors
    if (error.code === 'ECONNREFUSED' || error.code === 'ETIMEDOUT') {
      return {
        success: false,
        error: 'NETWORK_ERROR',
        message: 'Network connection failed. Please check your internet.'
      }
    }

    // Unknown errors
    return {
      success: false,
      error: 'UNKNOWN',
      message: error.message || 'An unexpected error occurred',
      stack: error.stack
    }
  }
}
```

#### Retry with Exponential Backoff

```javascript
async function uploadWithExponentialBackoff(client, file, maxRetries = 5) {
  let retryCount = 0
  let delay = 1000 // Start with 1 second

  while (retryCount < maxRetries) {
    try {
      return await client.uploadFile(file)
    } catch (error) {
      retryCount++

      // Don't retry on client errors (4xx)
      if (error.status >= 400 && error.status < 500 && error.name !== 'RateLimited') {
        throw error
      }

      if (retryCount >= maxRetries) {
        throw new Error(`Upload failed after ${maxRetries} retries: ${error.message}`)
      }

      console.log(`Upload failed (${retryCount}/${maxRetries}). Retrying in ${delay}ms...`)
      await new Promise(resolve => setTimeout(resolve, delay))

      // Exponential backoff: 1s, 2s, 4s, 8s, 16s
      delay *= 2
    }
  }
}
```

---

## Common Patterns

### Pattern 1: Upload with Progress Tracking

```javascript
import { create } from '@web3-storage/w3up-client'
import { filesFromPath } from 'files-from-path'

async function uploadWithProgress(dirPath) {
  const client = await create()
  const files = await filesFromPath(dirPath)

  let totalBytes = 0
  let uploadedBytes = 0

  // Calculate total size
  for (const file of files) {
    totalBytes += file.size
  }

  const cid = await client.uploadDirectory(files, {
    onShardStored: (meta) => {
      uploadedBytes += meta.size
      const progress = (uploadedBytes / totalBytes * 100).toFixed(2)
      console.log(`Progress: ${progress}% (${uploadedBytes}/${totalBytes} bytes)`)
      console.log(`Shard stored: ${meta.cid}`)
    }
  })

  console.log('Upload complete!')
  console.log(`URL: https://w3s.link/ipfs/${cid}`)
  return cid
}
```

---

### Pattern 2: Resumable Uploads

```javascript
import { CarWriter } from '@ipld/car'
import { CID } from 'multiformats/cid'

async function resumableUpload(client, files, storageKey = 'upload-progress') {
  // Load previous progress
  const savedProgress = localStorage.getItem(storageKey)
  const uploadedShards = savedProgress ? JSON.parse(savedProgress) : []

  const shards = []
  let rootCid

  // Create CAR shards
  for (let i = 0; i < files.length; i++) {
    const file = files[i]

    // Skip if already uploaded
    if (uploadedShards.includes(i)) {
      console.log(`Shard ${i} already uploaded, skipping...`)
      continue
    }

    // Create CAR for file
    const { writer, out } = CarWriter.create([someCid])
    // ... add blocks ...
    writer.close()

    const carChunks = []
    for await (const chunk of out) {
      carChunks.push(chunk)
    }
    const carBlob = new Blob(carChunks)

    // Upload CAR
    try {
      const result = await client.capability.store.add(carBlob)
      shards.push(result.link)

      // Save progress
      uploadedShards.push(i)
      localStorage.setItem(storageKey, JSON.stringify(uploadedShards))

      console.log(`Shard ${i} uploaded: ${result.link}`)
    } catch (error) {
      console.error(`Shard ${i} failed:`, error)
      throw error
    }
  }

  // Register upload
  await client.capability.upload.add(rootCid, shards)

  // Clear progress
  localStorage.removeItem(storageKey)

  return rootCid
}
```

---

### Pattern 3: Batch Operations

```javascript
async function batchUpload(client, files, batchSize = 10) {
  const results = []

  // Process in batches
  for (let i = 0; i < files.length; i += batchSize) {
    const batch = files.slice(i, i + batchSize)

    console.log(`Processing batch ${i / batchSize + 1} (${batch.length} files)`)

    // Upload batch concurrently
    const batchResults = await Promise.allSettled(
      batch.map(file => client.uploadFile(file))
    )

    // Collect results
    batchResults.forEach((result, index) => {
      if (result.status === 'fulfilled') {
        results.push({
          file: batch[index].name,
          cid: result.value.toString(),
          success: true
        })
      } else {
        results.push({
          file: batch[index].name,
          error: result.reason.message,
          success: false
        })
      }
    })

    // Rate limiting: pause between batches
    if (i + batchSize < files.length) {
      await new Promise(resolve => setTimeout(resolve, 1000))
    }
  }

  // Summary
  const successful = results.filter(r => r.success).length
  const failed = results.filter(r => !r.success).length

  console.log(`\nBatch upload complete:`)
  console.log(`  Successful: ${successful}`)
  console.log(`  Failed: ${failed}`)

  return results
}
```

---

### Pattern 4: Space Management

```javascript
async function manageSpaces(client) {
  // List all spaces
  const spaces = client.spaces()
  console.log('Available spaces:', spaces.length)

  // Create dedicated spaces for different purposes
  const productionSpace = await client.createSpace('production')
  const stagingSpace = await client.createSpace('staging')
  const devSpace = await client.createSpace('development')

  // Upload to specific space
  async function uploadToSpace(spaceDID, file) {
    await client.setCurrentSpace(spaceDID)
    return await client.uploadFile(file)
  }

  // Production upload
  const prodCid = await uploadToSpace(productionSpace.did(), prodFile)

  // Development upload
  const devCid = await uploadToSpace(devSpace.did(), devFile)

  // Export space delegation for sharing
  const delegation = await client.createDelegation({
    audience: 'did:key:z6MkqG7k...',
    abilities: ['upload/add', 'upload/list'],
    expiration: Math.floor(Date.now() / 1000) + 86400 // 24 hours
  })

  const archive = await delegation.archive()
  const base64Delegation = Buffer.from(archive.ok).toString('base64')

  return { prodCid, devCid, delegation: base64Delegation }
}
```

---

### Pattern 5: Delegation Management

```javascript
async function delegationWorkflow() {
  // Server: Create delegation for client
  const serverClient = await create({ principal: serverSigner })

  const delegation = await serverClient.createDelegation({
    audience: clientDID,
    abilities: ['upload/add', 'blob/add'],
    expiration: Math.floor(Date.now() / 1000) + 7 * 24 * 60 * 60 // 7 days
  })

  // Export delegation
  const archive = await delegation.archive()
  const delegationCAR = Buffer.from(archive.ok)

  // Transfer to client (e.g., via API)
  await sendToClient(delegationCAR)

  // Client: Import delegation
  const clientClient = await create({ principal: clientSigner })

  const receivedDelegation = await Delegation.extract(delegationCAR)
  const space = await clientClient.addSpace(receivedDelegation)

  await clientClient.setCurrentSpace(space.did())

  // Client can now upload
  const cid = await clientClient.uploadFile(file)

  return cid
}
```

---

### Pattern 6: Error Recovery

```javascript
async function uploadWithRecovery(client, files) {
  const failedUploads = []
  const successfulUploads = []

  for (const file of files) {
    try {
      const cid = await client.uploadFile(file)
      successfulUploads.push({ file: file.name, cid: cid.toString() })
    } catch (error) {
      console.error(`Failed to upload ${file.name}:`, error.message)
      failedUploads.push({ file: file.name, error: error.message })
    }
  }

  // Retry failed uploads
  if (failedUploads.length > 0) {
    console.log(`\nRetrying ${failedUploads.length} failed uploads...`)

    for (const failed of failedUploads) {
      const file = files.find(f => f.name === failed.file)

      try {
        // Wait before retry
        await new Promise(resolve => setTimeout(resolve, 2000))

        const cid = await client.uploadFile(file)
        successfulUploads.push({ file: file.name, cid: cid.toString() })

        // Remove from failed list
        const index = failedUploads.indexOf(failed)
        failedUploads.splice(index, 1)
      } catch (error) {
        console.error(`Retry failed for ${file.name}:`, error.message)
      }
    }
  }

  return {
    successful: successfulUploads,
    failed: failedUploads,
    totalFiles: files.length,
    successRate: (successfulUploads.length / files.length * 100).toFixed(2)
  }
}
```

---

## Conclusion

This API reference provides comprehensive documentation for all Storacha interfaces:

- **HTTP API**: UCAN Bridge protocol for direct HTTP access
- **JavaScript Client**: Full-featured browser and Node.js SDK
- **CLI**: Command-line tool for terminal workflows
- **Go API**: Go SDK for backend services

**Key Resources:**

- Official Documentation: https://docs.storacha.network
- JavaScript Client: https://github.com/storacha/w3up
- CLI Repository: https://github.com/storacha/w3cli
- Go Client: https://github.com/storacha/go-w3up
- Specifications: https://github.com/storacha/specs

**Next Steps:**

1. Start with the Quickstart guide to create your first upload
2. Explore JavaScript client examples for common use cases
3. Use CLI for rapid prototyping and testing
4. Implement production applications with proper error handling
5. Join the Storacha community for support and updates

