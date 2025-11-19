# 07. Protocol Specifications

## Table of Contents

### Part 1: Specifications Overview and Core Protocols
- [Specifications Repository Overview](#specifications-repository-overview)
- [W3-Account Protocol](#w3-account-protocol)
- [W3-Session Protocol](#w3-session-protocol)

### Part 2: Storage and Integration Protocols
- [W3-Store Protocol](#w3-store-protocol)
- [W3-Blob Protocol](#w3-blob-protocol)
- [W3-Filecoin Protocol](#w3-filecoin-protocol)
- [Additional Protocol Specifications](#additional-protocol-specifications)

### Part 3: Ecosystem and Management
- [Protocol Interactions](#protocol-interactions)
- [Version Management and Compatibility](#version-management-and-compatibility)
- [Implementation Guidelines](#implementation-guidelines)
- [Troubleshooting and Best Practices](#troubleshooting-and-best-practices)

---

# Part 1: Specifications Overview and Core Protocols

## Specifications Repository Overview

### Repository Structure

The Storacha protocol specifications are maintained in the **storacha/specs** repository on GitHub, containing comprehensive technical specifications for the w3up protocol stack and associated subsystems.

**Repository Location**: `https://github.com/storacha/specs`

**Key Characteristics**:
- **Format**: Markdown-based specification documents
- **Organization**: Flat structure with main specs in root directory
- **Subdirectories**: Specialized specs (e.g., w3-filecoin/) for complex subsystems
- **Total Items**: 27+ specification documents
- **Implementation Repository**: `storacha/w3up` (formerly `web3-storage/w3up`)

### Specification Maturity Levels

The specifications repository uses a six-level maturity classification system to indicate the stability and reliability of each protocol:

| Level | Badge Color | Description | Usage Guidance |
|-------|-------------|-------------|----------------|
| **Work-in-Progress** | 🟠 Orange | Early drafts, frequent changes expected | Use with caution, API may change |
| **Draft** | 🟡 Yellow | Under review, may have breaking changes | Suitable for experimentation |
| **Reliable** | 🟢 Green | Implemented and tested in production | Safe for development |
| **Stable** | 🟢 Bright Green | Mature, backward compatible changes only | Production-ready |
| **Permanent** | 🔵 Blue | Frozen specification, no changes expected | Long-term stable |
| **Deprecated** | 🔴 Red | No longer recommended, replacement available | Migration recommended |

### Stable Specifications

These four core specifications form the foundation of the Storacha protocol stack:

#### 1. **w3-account** (Stable)
Enables users to synchronize and recover delegated capabilities through decentralized identifiers (DID) they control, providing account-based management over spaces.

**File**: `w3-account.md`

**Key Features**:
- Email-based account identifiers (`did:mailto`)
- DKIM signature-based authorization
- Account-to-agent delegation model
- Cross-device capability synchronization

#### 2. **w3-session** (Stable)
Facilitates capability delegation to agents via email verification through magic link authentication, providing user-friendly authorization flows.

**File**: `w3-session.md`

**Key Features**:
- Email-based authorization flow
- Magic link generation and verification
- Out-of-band user authorization
- Session attestation mechanism

#### 3. **w3-store** (Stable)
Manages storage of directed acyclic graph (DAG) shards as Content Archive (CAR) files, enabling large data structures to be stored across distributed providers.

**File**: `w3-store.md`

**Key Features**:
- CAR file storage protocol
- Shard management capabilities
- Upload registration and tracking
- Multi-shard DAG reconstruction

#### 4. **w3-filecoin** (Stable)
Provides verifiable mechanisms for committing uploads to Filecoin storage through aggregation, deal creation, and proof generation.

**File**: `w3-filecoin/w3-filecoin.md`

**Key Features**:
- Filecoin deal integration
- Data aggregation protocol
- Piece CID generation
- Proof of Data Segment Inclusion (PoDSI)

### Additional Specifications

Beyond the stable core, the repository contains numerous specifications at varying maturity levels:

**Access and Administration**:
- `w3-access.md` - Access delegation delivery protocol
- `w3-admin.md` (Reliable) - Administrative capabilities for customer/subscription management
- `w3-provider.md` - Provider management and billing integration

**Storage and Content**:
- `w3-blob.md` - Blob storage protocol (successor to w3-store)
- `w3-index.md` - IPNI (InterPlanetary Network Indexer) integration
- `w3-space.md` - Space namespace management
- `w3-store-ipfs-pinning.md` - IPFS pinning service compatibility

**Infrastructure**:
- `w3-clock.md` - Time-based coordination
- `w3-egress-tracking.md` - Bandwidth and traffic monitoring
- `w3-rate-limit.md` - Rate limiting policies
- `w3-replication.md` - Data replication strategies
- `w3-retrieval.md` - Content retrieval protocol

**Security and Authentication**:
- `w3-ucan.md` - W3 protocol extensions to UCAN core namespace
- `w3-ucan-bridge.md` - UCAN bridging mechanisms
- `w3-revocations-check.md` - Capability revocation checking
- `did-mailto.md` - DID method for email identifiers
- `http-header-ucan-invocation.md` - HTTP header-based UCAN invocation
- `content-serve-auth.md` - Authenticated content serving

**Lifecycle**:
- `w3-plan.md` - Subscription plan management

### Documentation Philosophy

The specs repository follows key principles:

**Nothing is Permanent** (Initially):
> "Nothing in the spec repo is permanent. While stable specifications maintain backward compatibility, they can still improve based on real-world implementation feedback."

**Status-Driven Development**:
- Specifications progress through maturity levels based on implementation experience
- Draft specifications evolve rapidly based on developer feedback
- Stable specifications maintain consistency but allow non-breaking enhancements

**Implementation-First**:
- Specifications are validated through implementation in `storacha/w3up`
- Real-world usage informs spec evolution
- Code and specification co-evolve to ensure practicality

---

## W3-Account Protocol

### Overview

The **w3-account** protocol establishes a comprehensive system for managing user access across decentralized namespaces through email-based identifiers, integrating UCAN delegations with human-friendly account management.

**Specification Status**: 🟢 **Stable**

**Purpose**: Enable users to synchronize and recover delegated capabilities across multiple devices and agents using memorable email-based account identifiers.

### Core Concepts

#### Spaces

**Definition**: Spaces correspond to asymmetric keypairs identified by `did:key` URIs.

**Ownership**: The private key holder serves as the **owner** with absolute authority over the namespace.

**Characteristics**:
- Cryptographically secured by Ed25519 or similar key algorithms
- Self-certified identity (no central authority required)
- Owner can delegate any subset of capabilities to other principals

```typescript
// Space DID Example
const spaceDID = 'did:key:z6MkwDK3M4PxU1FqcSt4quKnKQk4JKhFnfYKbFX8Kf4rCq1a'

// Space represents an owned resource
interface Space {
  did: string            // did:key identifier
  privateKey: Uint8Array // Ed25519 private key (owner-held)
  publicKey: Uint8Array  // Ed25519 public key
}
```

#### Accounts

**Definition**: Accounts represent principals identified by memorable identifiers, particularly `did:mailto` addresses derived from email addresses.

**Purpose**: Aggregate access to user spaces and manage authorization for different agents across devices.

**Advantages**:
- Human-memorable identifiers (email addresses)
- Cross-device synchronization
- Recovery mechanism for lost devices
- Familiar authentication flow (email-based)

```typescript
// Account DID Example
const accountDID = 'did:mailto:example.com:alice'

// Derived from email
const email = 'alice@example.com'
const accountDID = `did:mailto:${email.split('@')[1]}:${email.split('@')[0]}`

// Account represents user identity across devices
interface Account {
  did: string           // did:mailto identifier
  email: string         // Associated email address
  spaces: string[]      // Array of space DIDs user has access to
}
```

**DID:mailto Format**:
```abnf
did-mailto = "did:mailto:" domain ":" local-part
domain     = 1*( ALPHA / DIGIT / "-" / "." )
local-part = 1*( ALPHA / DIGIT / "." / "_" / "-" )
```

#### Agents

**Definition**: Agents are principals identified by `did:key` URIs, representing user installations across devices and applications.

**Security Recommendation**: Agents should employ non-extractable keys where feasible (e.g., using Web Crypto API, hardware security modules).

**Lifecycle**:
- Created on each device/browser
- Receives delegated capabilities from account
- Can be revoked if device is compromised or lost
- Short-lived or long-lived based on use case

```typescript
// Agent DID Example
const agentDID = 'did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z'

// Agent represents a device or application
interface Agent {
  did: string               // did:key identifier
  name: string              // Human-readable name (e.g., "MacBook Pro")
  created: Date             // Creation timestamp
  lastUsed: Date            // Last activity timestamp
  capabilities: string[]    // Delegated capability URNs
}
```

### Delegation Model

The w3-account protocol establishes a three-tier delegation model:

```
┌─────────────┐
│   Space     │  did:key (Space Owner)
│  (Owner)    │
└──────┬──────┘
       │ delegates all capabilities
       ▼
┌─────────────┐
│   Account   │  did:mailto (User Email)
│  (User)     │
└──────┬──────┘
       │ re-delegates subset
       ▼
┌─────────────┐
│    Agent    │  did:key (Device/App)
│  (Device)   │
└─────────────┘
```

**Flow**:
1. **Space → Account**: Space owner delegates capabilities to their account (`did:mailto`)
2. **Account → Agent**: Account re-delegates capabilities to specific agents (`did:key`)
3. **Agent Invocation**: Agent uses delegated capabilities to perform operations

### Authorization Mechanisms

The w3-account specification defines two signature types for authorizing delegations from `did:mailto` principals:

#### 1. DKIM Signature Type

**Purpose**: Allow `did:mailto` principals to issue delegations signed with DomainKeys Identified Mail (DKIM) signatures.

**How It Works**:

1. **Message Construction**: User sends an email from their account email address with a special Subject header containing the authorization payload.

2. **Authorization Payload Format** (ABNF):
```abnf
auth := "I am signing ipfs://" cid "to grant access to this account"
cid  := z[a-km-zA-HJ-NP-Z1-9]+  ; Base58btc-encoded CID
```

**Example Authorization Payload**:
```
Subject: I am signing ipfs://zdpuAmoZixxJjvosviGeYcqduzDhSwGV2bL6ZTTXo1hbEJHfq to grant access to this account
```

3. **DKIM Signing**: Email server automatically adds DKIM signature to the email headers per RFC 6376.

4. **DKIM Payload Extraction**: The delegation verifier extracts the DKIM signature from the email headers.

5. **Encoding**: The DKIM signature is encoded as a nonstandard `VarSig` signature with algorithm parameter `"DKIM"`.

**VarSig Structure**:
```typescript
interface VarSig {
  '/': {
    algorithm: 'DKIM'
    signature: Uint8Array  // Raw DKIM signature bytes
  }
}
```

**UCAN Data Model**:
```typescript
interface DKIMSignedUCAN {
  // Standard UCAN fields (per UCAN-IPLD Schema)
  iss: string              // did:mailto:domain:localpart
  aud: string              // did:key of the audience (agent)
  att: Capability[]        // Array of capabilities being delegated
  exp: number              // Expiration timestamp (Unix epoch)
  prf: CID[]               // Proofs (parent delegation CIDs)

  // DKIM signature field
  sig: VarSig              // VarSig with algorithm='DKIM'
}
```

**Verification Process**:

```typescript
// Pseudo-code for DKIM verification
async function verifyDKIMDelegation(ucan: DKIMSignedUCAN): Promise<boolean> {
  // 1. Extract issuer email from did:mailto
  const email = didMailtoToEmail(ucan.iss)

  // 2. Extract DKIM signature
  const dkimSig = ucan.sig

  // 3. Reconstruct authorization payload
  const payload = `I am signing ipfs://${ucan.cid} to grant access to this account`

  // 4. Verify DKIM signature against email server's public key
  const isValid = await verifyDKIMSignature({
    domain: email.split('@')[1],
    signature: dkimSig.signature,
    message: payload
  })

  return isValid
}
```

**Example Implementation**:

```javascript
// From w3up-client
import * as DID from '@ipld/dag-ucan/did'
import * as UCAN from '@ipld/dag-ucan'

async function delegateWithDKIM(options) {
  const { space, account, capabilities } = options

  // 1. Create UCAN delegation structure
  const ucan = await UCAN.issue({
    issuer: account.did,      // did:mailto:example.com:alice
    audience: space.did,      // did:key:z6Mk...
    capabilities: capabilities,
    expiration: Date.now() + 86400 * 1000,  // 24 hours
    proofs: []  // Root delegation
  })

  // 2. Generate authorization payload
  const cid = await UCAN.encode(ucan)
  const payload = `I am signing ipfs://${cid} to grant access to this account`

  // 3. Prompt user to send email
  console.log(`Please send an email with subject: "${payload}"`)

  // 4. Wait for DKIM-signed email and extract signature
  // (Implementation depends on email service integration)

  return ucan
}
```

#### 2. Attestation Signature Type

**Purpose**: Enable delegations to be authorized through interactive email flows where users approve authorization requests via embedded links.

**How It Works**:

1. **Authorization Request**: Agent requests capabilities from account via an intermediary oracle service.

2. **Email Notification**: Oracle sends confirmation email to account holder with an embedded magic link.

3. **User Approval**: Account holder clicks the magic link to approve the authorization.

4. **Attestation Issuance**: Upon approval, oracle issues a signed **attestation** confirming that the account holder authorized the delegation.

**Attestation Signature Structure**:

The attestation signature is a special `VarSig` with **zero signature bytes**:

```typescript
interface AttestationSig {
  '/': {
    algorithm: 'Attestation'
    signature: Uint8Array  // Empty array: []
  }
}
```

**Why Zero Bytes?**
The attestation signature itself is insufficient for verification. It must be accompanied by a separate **UCAN attestation** from a trusted authority (the oracle).

**UCAN Attestation Structure**:

```typescript
interface UCANAttestation {
  iss: string              // Oracle's DID (trusted authority)
  aud: string              // Agent's DID (audience)
  att: [{
    can: 'ucan/attest',
    with: string           // Oracle's DID
    nb: {
      proof: CID           // Link to the attested delegation
    }
  }]
  exp: number              // Expiration
  prf: CID[]               // Proofs
  sig: Signature           // Cryptographic signature from oracle
}
```

**Delegation with Attestation**:

```typescript
interface AttestedDelegation {
  // The actual delegation
  delegation: {
    iss: string            // did:mailto (account)
    aud: string            // did:key (agent)
    att: Capability[]      // Capabilities being delegated
    exp: number
    prf: CID[]
    sig: AttestationSig    // Zero-byte attestation signature
  }

  // The attestation proving authorization
  attestation: UCANAttestation
}
```

**Verification Process**:

```typescript
async function verifyAttestedDelegation(
  delegation: AttestedDelegation
): Promise<boolean> {
  const { delegation: deleg, attestation } = delegation

  // 1. Verify attestation signature from oracle
  const oracleIsValid = await verifySignature(attestation)
  if (!oracleIsValid) return false

  // 2. Check oracle is trusted
  const oracleIsTrusted = await isTrustedOracle(attestation.iss)
  if (!oracleIsTrusted) return false

  // 3. Verify attestation points to this delegation
  const delegationCID = await computeCID(deleg)
  if (attestation.att[0].nb.proof.toString() !== delegationCID.toString()) {
    return false
  }

  // 4. Check attestation expiration
  if (Date.now() > attestation.exp) return false

  return true
}
```

**Example Flow**:

```javascript
// Agent requests authorization
import { Access } from '@web3-storage/capabilities'

async function requestAuthorization(agent, account, capabilities) {
  // 1. Create authorization request
  const request = await Access.authorize.invoke({
    issuer: agent,
    audience: oracleDID,  // Trusted oracle service
    with: agent.did(),
    nb: {
      iss: account.did,   // did:mailto
      att: capabilities
    }
  })

  // 2. Send request to oracle
  const result = await request.execute(oracleConnection)

  // 3. Oracle sends email with magic link
  console.log('Check your email for confirmation link')

  // 4. After user clicks link, oracle issues attestation
  // 5. Agent polls for attestation or receives via webhook

  return result
}
```

### Data Structures

#### Account Information

```typescript
interface AccountInfo {
  did: string              // did:mailto identifier
  email: string            // Email address
  product: string          // Subscription tier (e.g., 'free', 'lite', 'expert')
  spaces: SpaceInfo[]      // Accessible spaces
  created: string          // ISO 8601 timestamp
  updated: string          // ISO 8601 timestamp
}

interface SpaceInfo {
  did: string              // Space DID
  name?: string            // Optional human-readable name
  role: 'owner' | 'member' // User's role in the space
}
```

#### Delegation Chain

```typescript
interface DelegationChain {
  root: UCAN               // Root delegation (space → account)
  delegations: UCAN[]      // Chain of re-delegations
  target: string           // Final audience DID
}

// Example chain
const chain: DelegationChain = {
  root: {
    iss: 'did:key:space123',      // Space
    aud: 'did:mailto:alice',       // Account
    att: [{ can: '*', with: 'did:key:space123' }],
    // ... other UCAN fields
  },
  delegations: [{
    iss: 'did:mailto:alice',       // Account
    aud: 'did:key:agent456',       // Agent
    att: [{ can: 'store/add', with: 'did:key:space123' }],
    prf: [/* CID of root delegation */],
    // ... other fields
  }],
  target: 'did:key:agent456'
}
```

### Capability Definitions

The w3-account protocol defines several capabilities for account management:

#### account/info

**Purpose**: Retrieve account information including accessible spaces and subscription details.

**Capability Structure**:
```typescript
{
  can: 'account/info',
  with: string  // Account DID (did:mailto)
}
```

**Invocation Example**:
```javascript
import { Account } from '@web3-storage/capabilities'

const info = await Account.info.invoke({
  issuer: agent,
  audience: serviceDID,
  with: accountDID,
  proofs: [accountDelegation]
})

// Response
{
  did: 'did:mailto:example.com:alice',
  email: 'alice@example.com',
  product: 'lite',
  spaces: [
    { did: 'did:key:z6Mk...', name: 'my-project', role: 'owner' },
    { did: 'did:key:z6Mr...', name: 'team-space', role: 'member' }
  ]
}
```

#### account/list

**Purpose**: List all spaces accessible to an account.

**Capability Structure**:
```typescript
{
  can: 'account/list',
  with: string  // Account DID
}
```

### Implementation Examples

#### Creating an Account

```javascript
import { create } from '@storacha/client'
import { StoreMemory } from '@storacha/client/stores/memory'

async function createAccountWithEmail(email) {
  // 1. Create client (generates agent automatically)
  const client = await create({
    store: new StoreMemory()
  })

  // 2. Request account authorization via email
  const account = await client.login(email)

  // 3. Wait for email confirmation
  console.log(`Check ${email} for confirmation link`)

  // 4. Account is now linked to agent
  console.log('Account DID:', account.did())
  console.log('Agent DID:', client.agent.did())

  return { client, account }
}
```

#### Delegating to Another Agent

```javascript
async function delegateToDevice(client, deviceAgentDID, capabilities) {
  // 1. Get current account
  const account = client.currentAccount()

  // 2. Create delegation from account to device agent
  const delegation = await client.createDelegation({
    audience: deviceAgentDID,
    capabilities: capabilities,
    expiration: Math.floor(Date.now() / 1000) + (86400 * 30)  // 30 days
  })

  // 3. Archive delegation for transfer
  const archive = await delegation.archive()

  // 4. Transfer archive to device (e.g., QR code, file transfer)
  return archive
}

// On the device
async function receiveDelega(archive) {
  const client = await create()

  // Import delegation
  await client.addProof(archive)

  // Now this agent can use delegated capabilities
  const spaces = await client.spaces()
  console.log('Accessible spaces:', spaces.length)
}
```

#### Recovering Access on New Device

```javascript
async function recoverAccountOnNewDevice(email) {
  // 1. Create new agent on new device
  const client = await create()

  // 2. Login with account email
  const account = await client.login(email)

  // 3. Email confirmation re-authorizes access
  console.log(`Check ${email} for confirmation link`)

  // 4. After confirmation, account delegates to new agent
  // 5. All spaces accessible from original account are now available

  const spaces = await client.spaces()
  console.log('Recovered access to spaces:', spaces.length)

  return client
}
```

### Security Considerations

#### DKIM Security

**Strengths**:
- Leverages existing email infrastructure
- Cryptographically verifiable by email domain's public key
- No password required
- Decentralized verification (anyone can verify against domain's DNS records)

**Risks**:
- Depends on email server DKIM implementation
- Compromised email account = compromised capabilities
- DKIM key rotation requires delegation renewal
- Some email providers may not support DKIM consistently

**Mitigations**:
- Short-lived delegations (force periodic re-authorization)
- Capability attenuation (delegate minimal necessary capabilities)
- Revocation mechanisms for compromised accounts
- Multi-factor authentication at email provider level

#### Attestation Security

**Strengths**:
- User explicitly approves each authorization
- Oracle can enforce additional policies (rate limiting, anomaly detection)
- Phishing-resistant (user must access their email)
- Session-based authorization with expiration

**Risks**:
- Depends on oracle trustworthiness
- Oracle compromise could issue fraudulent attestations
- Email interception could allow unauthorized approvals
- Click-jacking attacks on confirmation links

**Mitigations**:
- Use reputable oracle services with security audits
- Implement anomaly detection (unusual authorization patterns)
- Short attestation expiration windows
- Confirmation link nonces and CSRF protection
- Allow users to review and revoke active attestations

#### Agent Key Management

**Recommendations**:
- **Use Non-Extractable Keys**: Leverage Web Crypto API with `extractable: false` or hardware security modules
- **Per-Device Agents**: Create separate agents for each device/browser
- **Capability Scoping**: Delegate only necessary capabilities to each agent
- **Regular Rotation**: Periodically rotate agent keys and re-delegate
- **Revocation List**: Maintain list of revoked agent DIDs

```javascript
// Creating non-extractable agent keys
async function createSecureAgent() {
  const keyPair = await crypto.subtle.generateKey(
    {
      name: 'Ed25519',
      namedCurve: 'Ed25519'
    },
    false,  // extractable = false (secure)
    ['sign', 'verify']
  )

  // Agent key cannot be exported from this context
  return { keyPair, did: await deriveDID(keyPair.publicKey) }
}
```

### Best Practices

#### Account Management

1. **Email Verification**: Always verify email ownership before issuing account delegations
2. **Audit Trail**: Log all account-to-agent delegations for security auditing
3. **Graceful Degradation**: Handle DKIM verification failures gracefully (fallback to attestation)
4. **Recovery Mechanism**: Provide account recovery flow for lost devices

#### Delegation Strategy

1. **Principle of Least Privilege**: Delegate minimal capabilities required
2. **Time-Bound Delegations**: Use expiration timestamps, force periodic re-authorization
3. **Hierarchical Delegation**: Space → Account → Agent (maintain clear chain)
4. **Proof Chains**: Always include proof CIDs to establish delegation provenance

#### Implementation

1. **UCAN Library Usage**: Use `@ucanto/core` and `@ucanto/principal` for UCAN operations
2. **DID Resolution**: Implement proper `did:mailto` and `did:key` resolution
3. **Signature Verification**: Validate all signatures in delegation chains
4. **Expiration Checks**: Always verify delegation hasn't expired before use

---

## W3-Session Protocol

### Overview

The **w3-session** protocol defines an email-based authorization protocol that enables users to delegate capabilities through familiar email workflows rather than complex UCAN interactions.

**Specification Status**: 🟢 **Stable**

**Purpose**: Facilitate capability delegation to agents via email verification through magic link authentication, providing a user-friendly bridge between the UCAN authorization model and mainstream user expectations.

### Problem Statement

**Challenge**: While UCAN delegations provide powerful decentralized authorization, they present usability challenges:
- Average users find asymmetric key management daunting
- Generating DKIM signatures requires sending emails with special subject lines
- Direct UCAN workflows are unfamiliar to most web users
- No established pattern for "approving" capability delegations

**Solution**: The w3-session protocol introduces an intermediary **oracle** service that translates email-based user actions (clicking magic links) into cryptographically valid UCAN attestations.

### Architecture

```
┌─────────────┐
│   Agent     │  did:key (device requesting access)
│  (Device)   │
└──────┬──────┘
       │ 1. access/authorize request
       ▼
┌─────────────┐
│   Oracle    │  Trusted intermediary service
│  (Service)  │
└──────┬──────┘
       │ 2. Sends email with magic link
       ▼
┌─────────────┐
│   Account   │  did:mailto (user email)
│  (User)     │
└──────┬──────┘
       │ 3. Clicks magic link (access/confirm)
       ▼
┌─────────────┐
│   Oracle    │
│  (Service)  │
└──────┬──────┘
       │ 4. Issues authorization session (attestation)
       ▼
┌─────────────┐
│   Agent     │  Now has attested delegation
│  (Device)   │
└─────────────┘
```

### Key Roles

#### Agent

**Identity**: `did:key` principal
**Characteristics**:
- Device-specific or application-specific
- Temporary cryptographic identity
- Requests capabilities from account

**Responsibilities**:
- Initiate authorization requests
- Present delegations for invocations
- Manage received capabilities

#### Account

**Identity**: `did:mailto` principal
**Characteristics**:
- Memorable email-based identifier
- Human-controlled
- Aggregates capabilities across devices

**Responsibilities**:
- Approve or deny authorization requests
- Manage access across multiple agents
- Serve as capability synchronization point

#### Oracle

**Identity**: `did:web` or `did:key` (service provider)
**Characteristics**:
- Trusted third-party service
- Facilitates out-of-band authorization
- Issues attestations

**Responsibilities**:
- Send confirmation emails
- Generate magic links
- Issue authorization session attestations
- Enforce security policies (rate limiting, anomaly detection)

#### Authority

**Identity**: Service provider DID
**Characteristics**:
- Executes invoked capabilities
- Validates authorization sessions

**Responsibilities**:
- Verify delegation chains
- Execute authorized operations
- Return receipts

#### Verifier

**Identity**: Component within authority or oracle
**Responsibilities**:
- Validate UCAN signatures
- Check expiration timestamps
- Verify delegation chain integrity

### Authorization Flow

#### Step 1: Authorization Request

The agent invokes the **`access/authorize`** capability to request authorization from the account.

**Capability Structure**:
```typescript
{
  can: 'access/authorize',
  with: string,  // Agent DID (issuer)
  nb: {
    iss: string,         // Account DID (did:mailto)
    att: Capability[],   // Requested capabilities
    exp?: number         // Optional expiration override
  }
}
```

**Invocation Example**:
```javascript
import { Access } from '@web3-storage/capabilities'
import { connect } from '@storacha/client'

async function requestAccess(agent, accountEmail, capabilities) {
  // Connect to oracle service
  const oracle = connect({
    serviceURL: 'https://up.web3.storage',
    serviceDID: 'did:web:web3.storage'
  })

  // Create authorization request
  const authorization = await Access.authorize.invoke({
    issuer: agent,                    // Agent requesting access
    audience: oracle.did(),           // Oracle service
    with: agent.did(),                // Agent's own DID
    nb: {
      iss: `did:mailto:${accountEmail}`,  // Account to authorize from
      att: capabilities,               // Capabilities being requested
      exp: Math.floor(Date.now() / 1000) + 86400  // 24 hour expiration
    }
  })

  // Send request to oracle
  const result = await authorization.execute(oracle)

  return result
}

// Usage
const capabilities = [
  {
    can: 'store/add',
    with: 'did:key:z6MkspaceXYZ...'
  },
  {
    can: 'upload/add',
    with: 'did:key:z6MkspaceXYZ...'
  }
]

await requestAccess(agent, 'alice@example.com', capabilities)
// Console: "Check alice@example.com for confirmation email"
```

**Wildcard Capabilities** (Use with Caution):

The specification allows requesting "sudo" access using wildcard capabilities:

```javascript
const sudoCapabilities = [
  {
    can: '*',  // All capabilities
    with: 'did:key:z6MkspaceXYZ...'
  }
]
```

**Warning**: Wildcard delegations grant full control over the resource. Only use when absolutely necessary and with short expirations.

#### Step 2: Email Notification

Upon receiving the `access/authorize` invocation, the oracle:

1. **Validates Request**:
   - Checks agent DID is valid
   - Verifies account DID corresponds to a valid email
   - Validates requested capabilities are well-formed

2. **Creates Confirmation Invocation**:
   - Generates an `access/confirm` invocation
   - Encodes it with necessary proofs
   - Creates a unique nonce for CSRF protection

3. **Sends Email**:
   - Addresses email to the account email address
   - Includes clickable magic link containing encoded `access/confirm`
   - Provides context about authorization request

**Email Template Example**:
```
Subject: Authorization Request for web3.storage

Hi Alice,

A device is requesting access to your web3.storage account.

Device: MacBook Pro (did:key:z6MkrZ1...)
Requested capabilities:
  - Upload files to space "my-project"
  - Store data in space "my-project"

If you initiated this request, click the link below to approve:

https://up.web3.storage/authorize?token=eyJhbGc...

This link expires in 15 minutes.

If you did not initiate this request, please ignore this email.
```

#### Step 3: Confirmation Invocation

When the user clicks the magic link, it invokes the **`access/confirm`** capability.

**Capability Structure**:
```typescript
{
  can: 'access/confirm',
  with: string,  // Account DID (did:mailto)
  nb: {
    aud: string,         // Agent DID (audience of the delegation)
    att: Capability[],   // Capabilities being confirmed
    exp?: number         // Optional expiration
  }
}
```

**Link Encoding**:

The magic link URL contains an encoded UCAN invocation:

```javascript
// Oracle generates confirmation link
function generateMagicLink(authorizationRequest) {
  const { agent, account, capabilities } = authorizationRequest

  // Create access/confirm invocation
  const confirmation = {
    can: 'access/confirm',
    with: account.did,
    nb: {
      aud: agent.did,
      att: capabilities,
      exp: Math.floor(Date.now() / 1000) + 900  // 15 minute link expiration
    }
  }

  // Encode as JWT or CAR
  const token = encodeInvocation(confirmation)

  // Generate URL
  return `https://up.web3.storage/authorize?token=${token}`
}
```

**Link Click Flow**:

```javascript
// Oracle receives access/confirm invocation
async function handleConfirmation(confirmInvocation) {
  const { aud, att } = confirmInvocation.nb

  // 1. Validate confirmation
  if (confirmInvocation.exp < Date.now() / 1000) {
    return { error: 'Confirmation link expired' }
  }

  // 2. Create delegation from account to agent
  const delegation = await createDelegation({
    issuer: confirmInvocation.with,  // Account (did:mailto)
    audience: aud,                    // Agent (did:key)
    capabilities: att,
    expiration: confirmInvocation.nb.exp || (Date.now() / 1000 + 86400)
  })

  // 3. Create authorization session attestation
  const session = await createAuthorizationSession(delegation)

  // 4. Store session for agent to claim
  await storeSession(aud, session)

  // 5. Display success page to user
  return { success: true }
}
```

#### Step 4: Authorization Session

An **authorization session** is a UCAN delegation that serves as proof of out-of-band authorization.

**Structure**:
```typescript
interface AuthorizationSession {
  // Attestation UCAN
  iss: string           // Oracle DID (issuer)
  aud: string           // Agent DID (audience)
  att: [{
    can: 'ucan/attest',
    with: string,       // Oracle DID
    nb: {
      proof: CID        // CID of the attested delegation
    }
  }],
  exp: number,          // Session expiration
  prf: CID[],           // Proofs
  sig: Signature        // Oracle's signature
}
```

**Creating Authorization Session**:

```javascript
import * as UCAN from '@ipld/dag-ucan'
import { sign } from '@ucanto/principal'

async function createAuthorizationSession(delegation, oracle) {
  // 1. Compute CID of the delegation being attested
  const delegationCID = await UCAN.encode(delegation)

  // 2. Create attestation UCAN
  const attestation = await UCAN.issue({
    issuer: oracle,           // Oracle as trusted authority
    audience: delegation.aud,  // Agent receiving the delegation
    capabilities: [{
      can: 'ucan/attest',
      with: oracle.did(),
      nb: {
        proof: delegationCID  // Link to attested delegation
      }
    }],
    expiration: Math.floor(Date.now() / 1000) + 86400,  // 24 hours
    proofs: []
  })

  // 3. Sign attestation with oracle's key
  const signedAttestation = await sign(attestation, oracle)

  return {
    delegation,       // The actual delegation (with attestation signature)
    attestation: signedAttestation  // Oracle's attestation UCAN
  }
}
```

**Agent Claims Session**:

```javascript
// Agent polls or receives webhook notification
async function claimAuthorizationSession(agent, oracle) {
  // Request session from oracle
  const response = await fetch(`${oracle.url}/session/${agent.did()}`)
  const session = await response.json()

  if (!session) {
    console.log('No authorization session available yet')
    return null
  }

  // Store delegation and attestation
  await agent.addProof(session.delegation)
  await agent.addProof(session.attestation)

  console.log('Authorization complete! Agent now has delegated capabilities.')

  return session
}
```

### Capabilities

#### access/authorize

**Purpose**: Request authorization from an account via oracle-mediated email flow.

**Capability Definition**:
```typescript
interface AccessAuthorize {
  can: 'access/authorize'
  with: string  // Agent DID (resource)
  nb: {
    iss: string         // Account DID (did:mailto) to authorize from
    att: Capability[]   // Requested capabilities
    exp?: number        // Optional expiration timestamp
  }
}
```

**Invocation Requirements**:
- **Issuer**: Must be the agent requesting access (matches `with` field)
- **Audience**: Must be a trusted oracle service
- **Proofs**: None required (agent is self-authorizing the request)

**Response**:
```typescript
interface AuthorizeResponse {
  success: boolean
  message?: string  // e.g., "Check your email for confirmation"
}
```

#### access/confirm

**Purpose**: Confirm authorization request after user clicks magic link.

**Capability Definition**:
```typescript
interface AccessConfirm {
  can: 'access/confirm'
  with: string  // Account DID (did:mailto)
  nb: {
    aud: string         // Agent DID to delegate to
    att: Capability[]   // Capabilities to delegate
    exp?: number        // Optional expiration
  }
}
```

**Invocation Context**:
- **Issuer**: Oracle (creates invocation on behalf of account)
- **Audience**: Oracle (self-invocation)
- **Proof**: Implicit proof via email access (user clicking link proves email control)

**Response**:
```typescript
interface ConfirmResponse {
  success: boolean
  delegation?: Delegation       // The created delegation
  attestation?: UCANAttestation // Oracle's attestation
}
```

#### access/delegate

**Purpose**: Explicitly delegate capabilities from account to agent (alternative to email flow).

**Capability Definition**:
```typescript
interface AccessDelegate {
  can: 'access/delegate'
  with: string  // Account DID (did:mailto)
  nb: {
    aud: string         // Agent DID
    att: Capability[]   // Capabilities to delegate
    exp?: number        // Expiration
  }
}
```

**Use Case**: Advanced users who want to bypass email flow and directly create delegations (requires account already linked to an agent).

### Session Management

#### Session Lifecycle

```
┌─────────────────┐
│   Request       │  access/authorize invoked
│   Created       │
└────────┬────────┘
         │ Oracle sends email
         ▼
┌─────────────────┐
│   Pending       │  Waiting for user confirmation
│                 │  (15 minute timeout)
└────────┬────────┘
         │ User clicks link
         ▼
┌─────────────────┐
│   Confirmed     │  access/confirm invoked
│                 │
└────────┬────────┘
         │ Oracle creates session
         ▼
┌─────────────────┐
│   Active        │  Agent claims session
│                 │  (24 hour timeout)
└────────┬────────┘
         │ Expiration or revocation
         ▼
┌─────────────────┐
│   Expired       │  No longer valid
│                 │
└─────────────────┘
```

#### Session Storage

**Oracle Session Store**:

```typescript
interface SessionStore {
  // Key: Agent DID
  // Value: Authorization session
  sessions: Map<string, AuthorizationSession>
}

class OracleSessionManager {
  private store: SessionStore = { sessions: new Map() }

  // Store session after confirmation
  async storeSession(agentDID: string, session: AuthorizationSession) {
    this.store.sessions.set(agentDID, session)

    // Set expiration cleanup
    setTimeout(() => {
      this.store.sessions.delete(agentDID)
    }, session.exp * 1000 - Date.now())
  }

  // Agent claims session
  async claimSession(agentDID: string): Promise<AuthorizationSession | null> {
    const session = this.store.sessions.get(agentDID)

    if (!session) return null

    // Check expiration
    if (Date.now() > session.exp * 1000) {
      this.store.sessions.delete(agentDID)
      return null
    }

    // Return session (optionally delete after first claim)
    return session
  }

  // Revoke session
  async revokeSession(agentDID: string) {
    this.store.sessions.delete(agentDID)
  }
}
```

#### Session Revocation

Users can revoke active sessions through the account management interface:

```javascript
async function revokeSession(account, agentDID) {
  // Invoke session revocation
  const revocation = await Access.session.revoke.invoke({
    issuer: account,
    audience: oracle.did(),
    with: account.did(),
    nb: {
      session: agentDID
    }
  })

  await revocation.execute(oracle)

  console.log(`Session for agent ${agentDID} revoked`)
}
```

### Data Structures

#### Authorization Request

```typescript
interface AuthorizationRequest {
  // Request metadata
  id: string                  // Unique request ID
  created: number             // Timestamp (Unix epoch)
  expiresAt: number           // Expiration timestamp

  // Principals
  agent: {
    did: string               // Agent DID
    name?: string             // Optional device name
  }
  account: {
    did: string               // Account DID (did:mailto)
    email: string             // Email address
  }

  // Requested capabilities
  capabilities: Capability[]

  // Status
  status: 'pending' | 'confirmed' | 'expired' | 'denied'

  // Confirmation
  confirmedAt?: number        // Timestamp when confirmed
  confirmationToken?: string  // Magic link token
}
```

#### Magic Link Token

```typescript
interface MagicLinkToken {
  // Token metadata
  nonce: string               // Unique nonce for CSRF protection
  createdAt: number           // Creation timestamp
  expiresAt: number           // Expiration (typically 15 minutes)

  // Embedded invocation
  invocation: {
    can: 'access/confirm'
    with: string              // Account DID
    nb: {
      aud: string             // Agent DID
      att: Capability[]       // Capabilities
      exp?: number
    }
  }

  // Security
  signature: Uint8Array       // Oracle signature over token
}

// Token encoding
function encodeMagicLinkToken(token: MagicLinkToken): string {
  // Encode as JWT or CAR
  const encoded = encodeJWT(token)
  return encoded
}

// Token verification
function verifyMagicLinkToken(token: string, oracle: Principal): MagicLinkToken {
  const decoded = decodeJWT(token)

  // Verify signature
  if (!verifySignature(decoded, oracle.publicKey)) {
    throw new Error('Invalid token signature')
  }

  // Check expiration
  if (Date.now() > decoded.expiresAt) {
    throw new Error('Token expired')
  }

  return decoded
}
```

### Implementation Examples

#### Complete Authorization Flow

```javascript
import { create } from '@storacha/client'
import { StoreMemory } from '@storacha/client/stores/memory'

async function completeAuthorizationFlow() {
  // 1. Create agent (new device)
  const client = await create({
    store: new StoreMemory()
  })

  const agent = client.agent
  console.log('Agent DID:', agent.did())

  // 2. Request authorization via email
  const email = 'alice@example.com'
  console.log(`Requesting authorization for ${email}...`)

  const account = await client.login(email)

  // 3. Oracle sends email
  console.log(`✉️  Check ${email} for confirmation email`)

  // 4. Simulate user clicking magic link (in real flow, this is a web navigation)
  // ... user clicks link ...

  // 5. Poll for authorization session
  let attempts = 0
  while (attempts < 60) {  // Poll for up to 5 minutes
    await new Promise(resolve => setTimeout(resolve, 5000))  // Wait 5 seconds

    const authorized = await account.isAuthorized()

    if (authorized) {
      console.log('✅ Authorization confirmed!')
      break
    }

    attempts++
  }

  // 6. Use delegated capabilities
  const spaces = await client.spaces()
  console.log(`Access granted to ${spaces.length} spaces`)

  // 7. Perform operations
  if (spaces.length > 0) {
    await client.setCurrentSpace(spaces[0].did())

    const file = new Blob(['Hello from w3-session!'])
    const cid = await client.uploadFile(file)

    console.log('File uploaded:', cid.toString())
  }

  return client
}
```

#### Oracle Service Implementation

```javascript
import { Server } from '@ucanto/server'
import { Access } from '@web3-storage/capabilities'
import { sendEmail } from './email-service'

// Oracle service implementation
const oracleServer = Server.create({
  id: await Signer.generate(),  // Oracle DID
  service: {
    access: {
      // Handle access/authorize
      authorize: async (invocation) => {
        const { iss, att, exp } = invocation.capability.nb
        const agentDID = invocation.capability.with

        // 1. Validate request
        if (!iss.startsWith('did:mailto:')) {
          return { error: 'Invalid account DID' }
        }

        // 2. Extract email from did:mailto
        const email = didMailtoToEmail(iss)

        // 3. Create confirmation invocation
        const confirmInvocation = {
          can: 'access/confirm',
          with: iss,
          nb: {
            aud: agentDID,
            att: att,
            exp: exp
          }
        }

        // 4. Generate magic link
        const token = await generateMagicLinkToken(confirmInvocation)
        const magicLink = `https://up.web3.storage/authorize?token=${token}`

        // 5. Send email
        await sendEmail({
          to: email,
          subject: 'Authorization Request',
          body: `Click to authorize: ${magicLink}`,
          html: renderEmailTemplate({ agentDID, capabilities: att, magicLink })
        })

        // 6. Return success
        return {
          success: true,
          message: `Check ${email} for confirmation email`
        }
      },

      // Handle access/confirm (magic link click)
      confirm: async (invocation) => {
        const { aud, att, exp } = invocation.capability.nb
        const accountDID = invocation.capability.with

        // 1. Create delegation from account to agent
        const delegation = await createDelegation({
          issuer: accountDID,
          audience: aud,
          capabilities: att,
          expiration: exp || Math.floor(Date.now() / 1000) + 86400
        })

        // 2. Create attestation
        const attestation = await createAttestation(delegation)

        // 3. Store session for agent to claim
        await sessionStore.storeSession(aud, {
          delegation,
          attestation,
          exp: delegation.exp
        })

        // 4. Return success
        return {
          success: true,
          delegation,
          attestation
        }
      },

      // Agent claims session
      claim: async (invocation) => {
        const agentDID = invocation.issuer.did()

        const session = await sessionStore.claimSession(agentDID)

        if (!session) {
          return { error: 'No session available' }
        }

        return {
          success: true,
          ...session
        }
      }
    }
  }
})
```

### Security Considerations

#### Email Security

**Threats**:
- Email interception (man-in-the-middle)
- Phishing emails impersonating oracle
- Email account compromise

**Mitigations**:
- **Use HTTPS**: All magic links must use HTTPS to prevent interception
- **Short Link Expiration**: Magic links expire in 15 minutes
- **One-Time Use**: Magic links can only be used once
- **Email Verification**: Verify email ownership before sending sensitive links
- **SPF/DKIM/DMARC**: Implement email authentication to prevent spoofing
- **Branded Emails**: Use consistent branding to help users identify legitimate emails

#### Oracle Trust

**Risks**:
- Compromised oracle could issue fraudulent attestations
- Malicious oracle could collect user emails and authorization requests
- Oracle downtime prevents authorization

**Mitigations**:
- **Oracle Reputation**: Use well-established oracle services with security audits
- **Multi-Oracle Support**: Allow clients to choose from multiple trusted oracles
- **Attestation Expiration**: Short-lived attestations limit damage from compromise
- **Audit Logs**: Oracle maintains transparent audit logs of all attestations issued
- **Anomaly Detection**: Oracle monitors for suspicious authorization patterns

#### CSRF Protection

**Attack**: Attacker tricks user into clicking confirmation link for attacker's agent

**Protection**:
```javascript
// Include nonce in magic link
const token = {
  nonce: crypto.randomUUID(),
  invocation: confirmInvocation,
  createdAt: Date.now()
}

// Verify nonce hasn't been used before
if (await nonceStore.hasBeenUsed(token.nonce)) {
  throw new Error('Token already used (potential CSRF attack)')
}

await nonceStore.markUsed(token.nonce)
```

#### Rate Limiting

Prevent abuse of authorization requests:

```javascript
const rateLimiter = {
  // Per email address
  emailLimits: new Map(),  // email -> { count, resetAt }

  // Per IP address
  ipLimits: new Map(),     // ip -> { count, resetAt }

  async checkLimit(email, ip) {
    // Check email rate limit (10 requests per hour)
    const emailLimit = this.emailLimits.get(email)
    if (emailLimit && emailLimit.count > 10 && Date.now() < emailLimit.resetAt) {
      throw new Error('Too many authorization requests for this email')
    }

    // Check IP rate limit (50 requests per hour)
    const ipLimit = this.ipLimits.get(ip)
    if (ipLimit && ipLimit.count > 50 && Date.now() < ipLimit.resetAt) {
      throw new Error('Too many authorization requests from this IP')
    }

    // Increment counters
    this.incrementLimit(email, this.emailLimits)
    this.incrementLimit(ip, this.ipLimits)
  }
}
```

### Best Practices

#### For Users

1. **Verify Emails**: Always check email sender matches expected oracle domain
2. **Check Authorization Details**: Review capabilities being requested before clicking magic link
3. **Use Strong Email Security**: Enable 2FA on email account
4. **Revoke Unused Sessions**: Periodically review and revoke unused agent authorizations

#### For Developers

1. **Implement Polling with Backoff**: Poll for sessions with exponential backoff to avoid overwhelming oracle
2. **Handle Expiration Gracefully**: Prompt user to re-authorize if session expires
3. **Provide Context**: Show device/agent name in authorization requests to help users identify requests
4. **Log Authorization Events**: Maintain audit log of all authorizations for security review

#### For Oracle Operators

1. **Monitor Abuse**: Implement anomaly detection for unusual authorization patterns
2. **Email Deliverability**: Ensure high email deliverability (SPF, DKIM, DMARC)
3. **Clear Email Templates**: Use clear, non-technical language in confirmation emails
4. **Security Audits**: Regular security audits and penetration testing
5. **Transparent Policies**: Publish clear policies on data handling and session management

---

# Part 2: Storage and Integration Protocols

## W3-Store Protocol

### Overview

The **w3-store** protocol defines a comprehensive system for managing content-addressed data across distributed storage providers using Content Archive (CAR) files and UCAN-based authorization.

**Specification Status**: 🟢 **Stable**

**Purpose**: Enable storage of directed acyclic graph (DAG) shards as Content Archives (CARs), allowing large data structures to be stored across distributed providers with verifiable upload registration.

### Core Concepts

#### Space

**Definition**: A namespace identified by a `did:key` URI representing an owned resource that can be delegated.

**Storage Model**: Each space acts as a container for stored content, with storage state managed across compatible storage provider services.

**Key Properties**:
- Owner has absolute authority over the space
- Storage can be managed through delegated UCAN capabilities
- Multiple providers can serve a single space
- Storage state is consistent across providers

```typescript
interface Space {
  did: string               // did:key identifier
  storage: {
    used: number            // Bytes currently stored
    limit: number           // Storage quota in bytes
    items: number           // Number of stored items
  }
  providers: string[]       // Array of provider DIDs
}
```

#### Content Archive (CAR)

**Definition**: CAR files follow the IPLD transport specification, storing shards of user content in a standardized format.

**Purpose**: CAR files serve as the primary primitive for storing content, enabling large DAGs to be split across multiple manageable pieces.

**CARv1 Structure**:
```
┌─────────────────────────────────────┐
│  CAR Header                         │
│  - version: 1                       │
│  - roots: [CID, ...]                │
└─────────────────────────────────────┘
│  Block 1                            │
│  - CID + Block Data                 │
├─────────────────────────────────────┤
│  Block 2                            │
│  - CID + Block Data                 │
├─────────────────────────────────────┤
│  ...                                │
├─────────────────────────────────────┤
│  Block N                            │
│  - CID + Block Data                 │
└─────────────────────────────────────┘
```

**CAR File Characteristics**:
- **Self-contained**: Includes all blocks needed to reconstruct a DAG subset
- **Content-addressed**: Identified by SHA-256 multihash of entire CAR
- **Shardable**: Large DAGs split across multiple CARs when exceeding size limits
- **Verifiable**: CID can be recomputed to verify integrity

```typescript
interface CARFile {
  cid: CID                  // CAR file CID (codec: 0x0202, hash: SHA2-256)
  size: number              // Total bytes
  roots: CID[]              // Root CIDs contained in CAR
  blocks: Block[]           // IPLD blocks
}

interface Block {
  cid: CID                  // Block CID
  bytes: Uint8Array         // Block data
}
```

#### Upload Registration

**Definition**: An upload represents a content root CID plus the set of CAR shards containing the complete DAG.

**Purpose**: Enable content retrieval and reassembly by tracking which CAR files contain the blocks for a given root CID.

```typescript
interface Upload {
  root: CID                 // Root CID (entry point)
  shards: CID[]             // Array of CAR file CIDs
  created: string           // ISO 8601 timestamp
  space: string             // Space DID
}
```

**Why Uploads Are Needed**:
Large files get sharded across multiple CARs. Without upload registration, providers wouldn't know which CARs to fetch to reconstruct the original file.

**Example Flow**:
```javascript
// 1. Client encodes large directory as UnixFS DAG
const dag = await encodeDirectory(directory)
// Root: bafybeic5... (directory root CID)

// 2. Client shards DAG into multiple CARs
const cars = await shardDAG(dag, { maxShardSize: 100 * 1024 * 1024 })
// cars = [
//   { cid: bagbaiera..., size: 100MB, roots: [bafybeic5...] },
//   { cid: bagbaierb..., size: 100MB, roots: [bafybeic5...] },
//   { cid: bagbaierc..., size: 50MB, roots: [bafybeic5...] }
// ]

// 3. Client stores each CAR (store/add)
for (const car of cars) {
  await client.capability.invoke('store/add', { link: car.cid, size: car.size })
}

// 4. Client registers upload (upload/add)
await client.capability.invoke('upload/add', {
  root: dag.root,           // bafybeic5...
  shards: cars.map(c => c.cid)  // All CAR CIDs
})

// 5. Provider can now reconstruct directory by fetching all shards
```

### Storage Capabilities

#### store/add

**Purpose**: Request that a provider store and serve a specific CAR file.

**Capability Structure**:
```typescript
{
  can: 'store/add',
  with: string,    // Space DID
  nb: {
    link: CID,     // CAR CID (codec: 0x0202, SHA2-256)
    size: number,  // CAR size in bytes
    origin?: CID   // DEPRECATED: Causal link to previous store/add
  }
}
```

**Requirements**:
- `link` MUST use codec `0x0202` (CAR codec)
- `link` hash MUST be SHA2-256 multihash (recommended, other hashes optional)
- `size` MUST accurately reflect CAR byte count
- Space MUST have sufficient remaining capacity

**Response States**:

**1. Done (Already Stored)**:
```typescript
{
  status: 'done',
  allocated: number,  // Bytes newly allocated (0 if already present)
  with: string,       // Space DID
  link: CID           // CAR CID
}
```

**2. Upload Required**:
```typescript
{
  status: 'upload',
  allocated: number,     // Bytes allocated
  with: string,          // Space DID
  link: CID,             // CAR CID
  url: string,           // HTTP PUT endpoint
  headers: {             // Required headers for PUT request
    'x-amz-checksum-sha256': string,  // Base64 SHA-256 of CAR
    // ... other headers
  },
  expiresAt: string      // ISO 8601 expiration (typically 15 minutes)
}
```

**3. Error**:
```typescript
{
  status: 'error',
  message: string,
  code: 'InsufficientStorage' | 'InvalidArgument' | 'UnauthorizedSpace'
}
```

**Invocation Example**:
```javascript
import { Store } from '@web3-storage/capabilities'
import { CAR } from '@ucanto/transport'

async function storeCAR(client, space, carBytes) {
  // 1. Compute CAR CID
  const carCID = await CAR.codec.link(carBytes)

  // 2. Invoke store/add
  const result = await Store.add.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: {
      link: carCID,
      size: carBytes.length
    },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  // 3. Handle response
  if (result.out.ok.status === 'done') {
    console.log('CAR already stored')
    return carCID
  }

  if (result.out.ok.status === 'upload') {
    // 4. Upload CAR to provided URL
    const uploadResult = await fetch(result.out.ok.url, {
      method: 'PUT',
      headers: result.out.ok.headers,
      body: carBytes
    })

    if (!uploadResult.ok) {
      throw new Error(`Upload failed: ${uploadResult.statusText}`)
    }

    console.log('CAR uploaded successfully')
    return carCID
  }

  throw new Error(`store/add failed: ${result.out.error}`)
}
```

**HTTP PUT Upload**:

When provider returns `upload` status, client must perform HTTP PUT:

```bash
# Example HTTP PUT request
curl -X PUT "https://s3.amazonaws.com/carpark-prod-0/bagbaiera..." \
  -H "x-amz-checksum-sha256: 47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=" \
  -H "Content-Type: application/vnd.ipld.car" \
  --data-binary @file.car
```

**Provider Verification**:
Provider MUST verify:
1. Uploaded bytes match `size` parameter
2. SHA-256 hash matches CAR CID
3. CAR is valid (parseable, header matches content)

#### store/get

**Purpose**: Query the state of a specified CAR file held by the provider.

**Capability Structure**:
```typescript
{
  can: 'store/get',
  with: string,    // Space DID
  nb: {
    link: CID      // CAR CID to query
  }
}
```

**Response (Success)**:
```typescript
{
  link: CID,           // CAR CID
  size: number,        // CAR size in bytes
  origin?: CID         // DEPRECATED: Causal link
}
```

**Response (Not Found)**:
```typescript
{
  error: 'StoreItemNotFound',
  message: 'CAR not found in space'
}
```

**Invocation Example**:
```javascript
import { Store } from '@web3-storage/capabilities'

async function getStoredCAR(client, space, carCID) {
  const result = await Store.get.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: { link: carCID },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  if (result.out.ok) {
    console.log(`CAR found: ${result.out.ok.size} bytes`)
    return result.out.ok
  }

  console.log('CAR not found')
  return null
}
```

#### store/list

**Purpose**: Enumerate all CAR files stored in a space with pagination support.

**Capability Structure**:
```typescript
{
  can: 'store/list',
  with: string,    // Space DID
  nb: {
    cursor?: string,   // Pagination cursor
    size?: number,     // Results per page (default: 25, max: 1000)
    pre?: boolean      // DEPRECATED: Prefix filter
  }
}
```

**Response**:
```typescript
{
  cursor?: string,      // Next page cursor (undefined if last page)
  size: number,         // Number of results in this page
  results: StoreListItem[]
}

interface StoreListItem {
  link: CID,            // CAR CID
  size: number,         // CAR size in bytes
  origin?: CID,         // DEPRECATED: Causal link
  insertedAt: string    // ISO 8601 timestamp
}
```

**Invocation Example**:
```javascript
import { Store } from '@web3-storage/capabilities'

async function listAllCARs(client, space) {
  const allItems = []
  let cursor

  // Paginate through all results
  do {
    const result = await Store.list.invoke({
      issuer: client.agent,
      audience: client.service,
      with: space.did(),
      nb: {
        cursor,
        size: 100  // 100 items per page
      },
      proofs: [spaceDelegation]
    }).execute(client.connection)

    allItems.push(...result.out.ok.results)
    cursor = result.out.ok.cursor

    console.log(`Fetched ${result.out.ok.size} items, total: ${allItems.length}`)
  } while (cursor)

  return allItems
}

// Usage
const cars = await listAllCARs(client, space)
console.log(`Total CARs stored: ${cars.length}`)
console.log(`Total storage used: ${cars.reduce((sum, c) => sum + c.size, 0)} bytes`)
```

#### store/remove

**Purpose**: Remove a CAR file from provider storage.

**Capability Structure**:
```typescript
{
  can: 'store/remove',
  with: string,    // Space DID
  nb: {
    link: CID      // CAR CID to remove
  }
}
```

**Response (Success)**:
```typescript
{
  size: number         // Bytes freed by removal
}
```

**Response (Not Found)**:
```typescript
{
  error: 'StoreItemNotFound',
  message: 'CAR not found in space'
}
```

**Invocation Example**:
```javascript
import { Store } from '@web3-storage/capabilities'

async function removeCAR(client, space, carCID) {
  const result = await Store.remove.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: { link: carCID },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  if (result.out.ok) {
    console.log(`Freed ${result.out.ok.size} bytes`)
    return true
  }

  console.log('CAR not found or removal denied')
  return false
}
```

**Important Notes**:
- Removal is typically asynchronous (may take time to free space)
- Removing a CAR doesn't automatically remove associated uploads
- Some providers may retain CARs for a grace period

### Upload Capabilities

#### upload/add

**Purpose**: Register a content root CID and its associated CAR shards, enabling retrieval and reassembly.

**Capability Structure**:
```typescript
{
  can: 'upload/add',
  with: string,    // Space DID
  nb: {
    root: CID,           // Root CID (entry point)
    shards: CID[],       // Array of CAR CIDs
    origin?: CID         // DEPRECATED: Causal link
  }
}
```

**Requirements**:
- `root` MUST be the IPLD link to the desired content entry point
- `shards` MUST be an array of CAR CIDs (all using codec 0x0202)
- All shards MUST already be stored (via `store/add`)
- All shards MUST contain blocks from the root DAG

**Response (Success)**:
```typescript
{
  root: CID,              // Root CID
  shards: CID[],          // CAR CIDs
  allocated: number       // Total bytes allocated (sum of shard sizes)
}
```

**Invocation Example**:
```javascript
import { Upload } from '@web3-storage/capabilities'

async function registerUpload(client, space, rootCID, shardCIDs) {
  const result = await Upload.add.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: {
      root: rootCID,
      shards: shardCIDs
    },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  if (result.out.ok) {
    console.log(`Upload registered: ${result.out.ok.allocated} bytes`)
    return result.out.ok
  }

  throw new Error(`upload/add failed: ${result.out.error}`)
}
```

**Complete Upload Flow**:

```javascript
import { create } from '@storacha/client'

async function uploadDirectory(directory) {
  const client = await create()

  // 1. Encode directory as UnixFS DAG and shard into CARs
  const { root, shards } = await client.uploadDirectory(directory, {
    onShardStored: (meta) => {
      console.log(`Shard stored: ${meta.cid} (${meta.size} bytes)`)
    }
  })

  // Under the hood, this:
  // - Calls store/add for each shard
  // - Then calls upload/add with root + shard CIDs

  console.log(`Directory uploaded: ${root}`)
  console.log(`Total shards: ${shards.length}`)

  return root
}
```

#### upload/get

**Purpose**: Retrieve the registration state for a given root CID.

**Capability Structure**:
```typescript
{
  can: 'upload/get',
  with: string,    // Space DID
  nb: {
    root: CID      // Root CID to query
  }
}
```

**Response (Success)**:
```typescript
{
  root: CID,              // Root CID
  shards: CID[],          // CAR CIDs
  insertedAt: string      // ISO 8601 timestamp
}
```

**Response (Not Found)**:
```typescript
{
  error: 'UploadNotFound',
  message: 'Upload not registered in space'
}
```

**Invocation Example**:
```javascript
import { Upload } from '@web3-storage/capabilities'

async function getUpload(client, space, rootCID) {
  const result = await Upload.get.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: { root: rootCID },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  if (result.out.ok) {
    console.log(`Upload found: ${result.out.ok.shards.length} shards`)
    return result.out.ok
  }

  return null
}
```

#### upload/list

**Purpose**: Enumerate all uploads in a space with pagination.

**Capability Structure**:
```typescript
{
  can: 'upload/list',
  with: string,    // Space DID
  nb: {
    cursor?: string,   // Pagination cursor
    size?: number,     // Results per page (default: 25)
    pre?: boolean      // DEPRECATED: Prefix filter
  }
}
```

**Response**:
```typescript
{
  cursor?: string,        // Next page cursor
  size: number,           // Results count
  results: UploadListItem[]
}

interface UploadListItem {
  root: CID,              // Root CID
  shards: CID[],          // CAR CIDs
  insertedAt: string      // ISO 8601 timestamp
}
```

**Invocation Example**:
```javascript
async function listRecentUploads(client, space, limit = 10) {
  const result = await Upload.list.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: { size: limit },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  return result.out.ok.results
}

// Usage
const recent = await listRecentUploads(client, space, 5)
recent.forEach(upload => {
  console.log(`${upload.root} - ${upload.shards.length} shards - ${upload.insertedAt}`)
})
```

#### upload/remove

**Purpose**: Delete an upload registration from the space.

**Capability Structure**:
```typescript
{
  can: 'upload/remove',
  with: string,    // Space DID
  nb: {
    root: CID      // Root CID to remove
  }
}
```

**Response (Success)**:
```typescript
{
  root: CID              // Removed root CID
}
```

**Invocation Example**:
```javascript
import { Upload } from '@web3-storage/capabilities'

async function removeUpload(client, space, rootCID) {
  const result = await Upload.remove.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: { root: rootCID },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  if (result.out.ok) {
    console.log(`Upload removed: ${result.out.ok.root}`)
    return true
  }

  return false
}
```

**Important**:
- Removing upload registration does NOT automatically remove associated CAR shards
- To fully delete data, remove shards separately using `store/remove`
- Orphaned shards (shards not referenced by any upload) may be garbage collected

### Implementation Notes

#### No-op Behavior

Invoking `store/add` for an already-stored CAR functions as a no-op:
- Returns `status: 'done'`
- `allocated: 0` (no new space allocated)
- Idempotent operation (safe to retry)

This enables reliable retry logic:
```javascript
async function reliablyStoreCAR(client, space, carBytes, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const result = await storeCAR(client, space, carBytes)
      return result
    } catch (error) {
      if (attempt === maxRetries) throw error
      console.log(`Attempt ${attempt} failed, retrying...`)
      await new Promise(resolve => setTimeout(resolve, 1000 * attempt))
    }
  }
}
```

#### Upload vs Store Relationship

**Store**: Low-level primitive for CAR management
**Upload**: High-level abstraction for content management

```
┌─────────────────────────────────────────┐
│  Upload (Logical Content)               │
│  root: bafybei...                       │
│  shards: [bagbaiera..., bagbaierb...]   │
└───────────┬─────────────────────────────┘
            │ references
            ▼
┌─────────────────────────────────────────┐
│  Store (Physical CARs)                  │
│  - CAR 1: bagbaiera... (100 MB)         │
│  - CAR 2: bagbaierb... (100 MB)         │
└─────────────────────────────────────────┘
```

**Key Insight**: Multiple uploads can reference the same CAR shards, enabling deduplication:
```javascript
// Upload 1: Large video file
// root1: bafybeiVIDEO..., shards: [car1, car2, car3]

// Upload 2: Same video + metadata
// root2: bafybeiDIR..., shards: [car1, car2, car3, car4]
//                                 ^^^^^^^^^^^^^^^^^^^
//                                 Same CAR shards reused!
```

#### Capacity Management

Providers enforce storage quotas at the space level:

```typescript
interface SpaceQuota {
  limit: number,     // Total allowed bytes
  used: number,      // Currently used bytes
  available: number  // Remaining bytes
}

// Check before storing
async function checkCapacity(client, space, bytesNeeded) {
  const usage = await client.usage.get(space.did())

  if (usage.used + bytesNeeded > usage.limit) {
    throw new Error(`Insufficient storage: need ${bytesNeeded}, have ${usage.available}`)
  }
}
```

### Data Structures

#### Store Item

```typescript
interface StoreItem {
  link: CID                // CAR CID
  size: number             // Bytes
  origin?: CID             // DEPRECATED: Causal relationship
  insertedAt: string       // ISO 8601 timestamp
  space: string            // Space DID
}
```

#### Upload Item

```typescript
interface UploadItem {
  root: CID                // Root CID (content entry point)
  shards: CID[]            // Array of CAR CIDs
  insertedAt: string       // ISO 8601 timestamp
  space: string            // Space DID
  name?: string            // Optional human-readable name
}
```

### Best Practices

#### CAR Sharding Strategy

**Recommended Shard Size**: 100 MB - 200 MB per CAR

```javascript
const SHARD_SIZE = 100 * 1024 * 1024  // 100 MB

async function shardWithOptimalSize(dag) {
  return await shard(dag, {
    maxShardSize: SHARD_SIZE,
    // Ensure shards don't split mid-block
    shardAlignment: 'block'
  })
}
```

**Why This Size?**:
- Small enough for reliable network transfer
- Large enough to minimize shard overhead
- Fits within common provider limits

#### Deduplication

Take advantage of content-addressing for deduplication:

```javascript
// Instead of uploading same file twice, check if already stored
async function uploadWithDedup(client, space, file) {
  // 1. Compute file CID
  const { root, shards } = await encodeFile(file)

  // 2. Check if upload already exists
  const existing = await Upload.get.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: { root },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  if (existing.out.ok) {
    console.log('File already uploaded, returning existing CID')
    return root
  }

  // 3. Only upload if not found
  return await client.uploadFile(file)
}
```

#### Cleanup Strategy

Implement periodic cleanup of unused CARs:

```javascript
async function cleanupOrphanedShards(client, space) {
  // 1. Get all stored CARs
  const allCARs = await listAllCARs(client, space)

  // 2. Get all uploads
  const allUploads = await listAllUploads(client, space)

  // 3. Find shards referenced by uploads
  const referencedShards = new Set()
  for (const upload of allUploads) {
    upload.shards.forEach(shard => referencedShards.add(shard.toString()))
  }

  // 4. Identify orphaned shards
  const orphaned = allCARs.filter(car => !referencedShards.has(car.link.toString()))

  console.log(`Found ${orphaned.length} orphaned shards`)

  // 5. Remove orphaned shards
  for (const car of orphaned) {
    await Store.remove.invoke({
      issuer: client.agent,
      audience: client.service,
      with: space.did(),
      nb: { link: car.link },
      proofs: [spaceDelegation]
    }).execute(client.connection)

    console.log(`Removed orphaned shard: ${car.link}`)
  }
}
```

---

## W3-Blob Protocol

### Overview

The **w3-blob** protocol represents an evolution from w3-store, enabling storage of arbitrary content blobs without requiring the Content Archive (CAR) format.

**Specification Status**: 🟢 **Stable** (Successor to w3-store)

**Purpose**: Provide a core building block for storing content and sharing access through UCAN authorization, removing the CAR format requirement while maintaining backward compatibility.

### Evolution from Store to Blob

**W3-Store** (Legacy):
- Required CAR format for all stored content
- Client responsible for DAG encoding and CAR creation
- Complex client-side processing

**W3-Blob** (Current):
- Accepts arbitrary byte arrays (blobs)
- No CAR format requirement
- Simpler client implementation
- **Backward Compatible**: Clients can still store CARs as blobs

```
┌──────────────────┐        ┌──────────────────┐
│   w3-store       │        │    w3-blob       │
│  (CAR required)  │  --->  │  (Any bytes)     │
└──────────────────┘        └──────────────────┘
         ↓                           ↓
  Must create CAR            Direct blob upload
  before storage            (optional CAR support)
```

### Core Concepts

#### Space

Identical to w3-store: A unique asymmetric cryptographic keypair identified via `did:key` URI, representing owned resources that can be shared.

#### Blob

**Definition**: A fixed-size byte array addressed by multihash.

**Characteristics**:
- Content-addressed (identified by hash of content)
- Immutable (same bytes = same hash)
- Arbitrary content (images, videos, documents, CARs, etc.)
- Typically represents IPLD blocks or complete files

```typescript
interface Blob {
  digest: Multihash      // Multihash of blob bytes
  size: number           // Byte count
  bytes: Uint8Array      // Blob content
}

// Example multihash
{
  code: 0x12,            // SHA2-256
  digest: Uint8Array([...])  // 32-byte hash
}
```

**Multihash Support**:
- **Required**: SHA2-256 (code: 0x12)
- **Optional**: SHA2-512, BLAKE3, etc.

### Execution Flow

The `space/blob/add` operation produces **three sequential effects** with dependencies:

```
┌──────────────┐
│ blob/allocate│  Reserve space on provider
└──────┬───────┘
       │ produces: allocation (URL, headers, expiration)
       ▼
┌──────────────┐
│   blob/put   │  HTTP PUT to upload actual bytes
└──────┬───────┘
       │ produces: upload confirmation
       ▼
┌──────────────┐
│ blob/accept  │  Provider verifies and commits
└──────┬───────┘
       │ produces: location commitment
       ▼
    Success
```

**Key Insight**: Unlike w3-store (single-step store/add), w3-blob separates allocation, upload, and acceptance into distinct phases for better control and observability.

### Primary Capabilities

#### space/blob/add

**Purpose**: Main entry point for authorized agents to store a byte array in a space.

**Capability Structure**:
```typescript
{
  can: 'space/blob/add',
  with: string,        // Space DID
  nb: {
    blob: {
      digest: Multihash,   // Multihash of payload bytes
      size: number         // Byte count
    }
  }
}
```

**Requirements**:
- `blob.digest` MUST be SHA2-256 multihash (other hashes optional)
- `blob.size` MUST match actual byte count
- Space MUST have sufficient capacity

**Response (Success)**:
```typescript
{
  site: {
    'ucan/await': {
      '.tag': 'ok',
      await: {
        '/': string      // CID of blob/accept task
      }
    }
  }
}
```

**Response (Error)**:
```typescript
{
  error: {
    name: 'InsufficientStorage' | 'InvalidBlobDigest' | 'BlobSizeExceeded',
    message: string
  }
}
```

**Invocation Example**:
```javascript
import { Space } from '@web3-storage/capabilities'
import { sha256 } from 'multiformats/hashes/sha2'

async function addBlob(client, space, bytes) {
  // 1. Compute blob digest
  const hash = await sha256.digest(bytes)

  // 2. Invoke space/blob/add
  const result = await Space.blob.add.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: {
      blob: {
        digest: hash.bytes,  // Multihash bytes
        size: bytes.length
      }
    },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  // 3. Check for immediate success (already stored)
  if (result.out.ok.site['ucan/await']['.tag'] === 'ok') {
    // Wait for acceptance
    const acceptance = await client.capability.invoke('ucan/conclude', {
      receiptCID: result.out.ok.site['ucan/await'].await['/']
    })

    return acceptance
  }

  throw new Error(`blob/add failed: ${result.out.error}`)
}
```

#### blob/allocate

**Purpose**: Internal effect invoked by the system on storage provider to reserve memory space for a blob.

**Note**: This is NOT directly invocable by clients. It's an effect produced by `space/blob/add`.

**Capability Structure**:
```typescript
{
  can: 'blob/allocate',
  with: string,        // Provider DID (byte-encoded)
  nb: {
    space: string,     // Space DID (byte-encoded)
    blob: {
      digest: Multihash,
      size: number
    },
    cause: CID         // Link to originating space/blob/add
  }
}
```

**Response (Success)**:
```typescript
{
  size: number,          // Allocated bytes
  address: {
    url: string,         // HTTP PUT endpoint
    headers: {           // Required headers
      'content-length': string,
      'x-amz-checksum-sha256': string,
      // ... other headers
    },
    expiresAt: string    // ISO 8601 expiration
  }
}
```

**Example Response**:
```json
{
  "size": 1048576,
  "address": {
    "url": "https://s3.amazonaws.com/carpark-prod-0/blobstore/...",
    "headers": {
      "content-length": "1048576",
      "x-amz-checksum-sha256": "LZis8YPzuQnD1yJLJUkdSQ=="
    },
    "expiresAt": "2025-01-15T10:30:00Z"
  }
}
```

#### blob/put

**Purpose**: HTTP PUT request using allocated address to upload blob content.

**Note**: This is NOT a UCAN capability invocation. It's a standard HTTP operation.

**HTTP Request**:
```http
PUT https://s3.amazonaws.com/carpark-prod-0/blobstore/... HTTP/1.1
Content-Length: 1048576
x-amz-checksum-sha256: LZis8YPzuQnD1yJLJUkdSQ==
Content-Type: application/octet-stream

<blob bytes>
```

**Requirements**:
- MUST include all headers from `blob/allocate` response
- MUST upload exact byte count specified in `Content-Length`
- MUST complete before `expiresAt` timestamp

**JavaScript Example**:
```javascript
async function uploadBlob(allocationAddress, blobBytes) {
  const response = await fetch(allocationAddress.url, {
    method: 'PUT',
    headers: allocationAddress.headers,
    body: blobBytes
  })

  if (!response.ok) {
    throw new Error(`HTTP PUT failed: ${response.status} ${response.statusText}`)
  }

  console.log('Blob uploaded successfully')
  return response
}
```

**Provider Verification**:
After receiving HTTP PUT, provider MUST verify:
1. Uploaded bytes match expected size
2. SHA-256 hash matches blob digest
3. Upload completed before expiration

#### blob/accept

**Purpose**: System verification that uploaded content matches specifications, producing the location commitment.

**Note**: This is an internal effect, NOT directly invocable by clients.

**Capability Structure**:
```typescript
{
  can: 'blob/accept',
  with: string,        // Space DID
  nb: {
    blob: {
      digest: Multihash,
      size: number
    },
    exp: number,       // Expiration timestamp
    _put: {            // Reference to blob/put result
      'ucan/await': {
        '.tag': 'ok',
        await: CID     // Link to blob/put task
      }
    }
  }
}
```

**Response (Success)**:
```typescript
{
  site: {
    digest: Multihash,
    size: number
  }
}
```

This response serves as the **location commitment**: proof that the blob is stored and retrievable from the provider.

### Complete Blob Upload Flow

```javascript
import { create } from '@storacha/client'
import { sha256 } from 'multiformats/hashes/sha2'

async function uploadBlobComplete(client, space, blobBytes) {
  // 1. Compute blob digest
  const hash = await sha256.digest(blobBytes)
  const digest = hash.bytes

  console.log('1. Computed digest:', hash.toString())

  // 2. Invoke space/blob/add
  const addResult = await Space.blob.add.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: {
      blob: { digest, size: blobBytes.length }
    },
    proofs: [await client.createDelegation(space)]
  }).execute(client.connection)

  console.log('2. space/blob/add invoked')

  // 3. Extract allocation from effects
  const effects = addResult.out.ok.effects
  const allocateEffect = effects.find(e => e.can === 'blob/allocate')

  if (!allocateEffect) {
    throw new Error('No allocate effect found')
  }

  // 4. Wait for allocation result
  const allocateResult = await client.capability.conclude(allocateEffect.receipt)

  console.log('3. Allocation received:', allocateResult.out.ok.address.url)

  // 5. HTTP PUT blob bytes
  await fetch(allocateResult.out.ok.address.url, {
    method: 'PUT',
    headers: allocateResult.out.ok.address.headers,
    body: blobBytes
  })

  console.log('4. Blob uploaded via HTTP PUT')

  // 6. Wait for acceptance
  const acceptTask = addResult.out.ok.site['ucan/await'].await['/']
  const acceptResult = await client.capability.conclude(acceptTask)

  console.log('5. Blob accepted:', acceptResult.out.ok.site)

  return {
    digest: acceptResult.out.ok.site.digest,
    size: acceptResult.out.ok.site.size
  }
}
```

### Comparison: Blob vs Store

| Aspect | W3-Store | W3-Blob |
|--------|----------|---------|
| **Input Format** | Must be CAR file | Any byte array |
| **Client Complexity** | High (DAG encoding required) | Low (direct byte upload) |
| **Invocation** | Single `store/add` | Three-phase (`allocate` → `put` → `accept`) |
| **Observability** | Limited | Detailed (per-phase status) |
| **Use Case** | DAG shards | Arbitrary blobs, IPLD blocks, files |
| **Backward Compat** | N/A | Can store CARs as blobs |

**When to Use Blob**:
- Storing individual files (images, videos, documents)
- Storing raw IPLD blocks
- Simpler client implementation
- Better observability needed

**When to Still Use Store**:
- Legacy client compatibility
- Explicit CAR shard management needed
- Upload registration workflow

### Implementation Notes

#### High-Level Client Abstractions

Most developers won't invoke `space/blob/add` directly. Instead, use high-level client methods:

```javascript
import { create } from '@storacha/client'

const client = await create()

// High-level blob upload (handles all phases internally)
const file = new File(['Hello, Storacha!'], 'hello.txt')
const cid = await client.uploadFile(file)

console.log('File uploaded:', cid.toString())
```

#### Blob Deduplication

Like w3-store, w3-blob benefits from content-addressing:

```javascript
// Upload same blob twice
const blob1 = new Uint8Array([1, 2, 3, 4])
const blob2 = new Uint8Array([1, 2, 3, 4])

const digest1 = await uploadBlob(client, space, blob1)
const digest2 = await uploadBlob(client, space, blob2)

// Same digest (only stored once)
assert.equal(digest1.toString(), digest2.toString())
```

#### Error Handling

```javascript
async function uploadBlobWithRetry(client, space, blob, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await uploadBlobComplete(client, space, blob)
    } catch (error) {
      if (error.name === 'InsufficientStorage') {
        throw error  // Don't retry capacity errors
      }

      if (attempt === maxRetries) throw error

      console.log(`Attempt ${attempt} failed, retrying...`)
      await new Promise(resolve => setTimeout(resolve, 1000 * attempt))
    }
  }
}
```

### Best Practices

1. **Digest Verification**: Always compute digest client-side before upload
2. **Size Limits**: Check provider blob size limits (typically 1GB - 5GB)
3. **Expiration Handling**: Complete HTTP PUT before allocation expires (typically 15 minutes)
4. **Retry Logic**: Implement exponential backoff for transient failures
5. **Deduplication**: Check if blob already exists before uploading (via digest lookup)

---

## W3-Filecoin Protocol

### Overview

The **w3-filecoin** protocol provides verifiable mechanisms for committing uploads to Filecoin storage through aggregation, deal creation, and cryptographic proof generation.

**Specification Status**: 🟢 **Stable**

**Purpose**: Enable data stored in Storacha to be verifiably replicated to Filecoin's decentralized storage network, providing cold storage guarantees with Proof of Data Segment Inclusion (PoDSI).

### Core Concepts

#### Filecoin Integration Architecture

Storacha acts as a **hot storage layer** with automatic Filecoin **cold storage** replication:

```
┌──────────────────────────────────────┐
│  Storacha (Hot Storage)              │
│  - Fast retrieval                    │
│  - Content-addressed storage         │
│  - IPFS/HTTP gateways                │
└────────────┬─────────────────────────┘
             │ Automatic replication
             ▼
┌──────────────────────────────────────┐
│  w3filecoin Pipeline                 │
│  - Aggregation                       │
│  - Deal creation                     │
│  - Proof generation                  │
└────────────┬─────────────────────────┘
             │ Deals
             ▼
┌──────────────────────────────────────┐
│  Filecoin (Cold Storage)             │
│  - Decentralized storage providers   │
│  - Cryptographic proofs (PoDSI)      │
│  - Long-term archival                │
└──────────────────────────────────────┘
```

#### Key Terms

**Piece**: A unit of data in Filecoin, identified by a Piece CID. Pieces are padded to specific sizes (powers of 2) for Filecoin sector packing.

**Aggregate**: A collection of multiple pieces combined into a larger piece for efficient Filecoin deal creation. Aggregates minimize cost by batching small uploads.

**Deal**: A storage agreement between a client and Filecoin Storage Provider (SP), recorded on the Filecoin blockchain.

**PoDSI (Proof of Data Segment Inclusion)**: A cryptographic proof that a specific piece is included in an aggregate, enabling verification without revealing the entire aggregate.

**Spade**: A broker service (marketplace) that matches aggregates with Filecoin Storage Providers based on requirements like replication count and geographical diversity.

### w3filecoin Pipeline

The w3filecoin infrastructure consists of three main services:

#### 1. Storefront

**Purpose**: Facilitate data storage service requests from clients.

**Responsibilities**:
- Accept `filecoin/offer` invocations from clients
- Track offered pieces
- Create aggregates from multiple pieces
- Submit aggregates to aggregator service
- Issue `filecoin/accept` receipts when deals complete

**Flow**:
```
Client → filecoin/offer → Storefront
Storefront → Creates aggregate → Aggregator
Storefront ← Deal accepted ← Spade
Storefront → filecoin/accept receipt → Client
```

#### 2. Aggregator

**Purpose**: Aggregate smaller pieces into larger pieces suitable for Filecoin deals.

**Responsibilities**:
- Receive pieces from storefront
- Combine pieces into optimally-sized aggregates
- Compute aggregate Piece CID
- Generate Data Aggregation Proofs (DAP)
- Submit aggregates to dealer service

**Aggregation Strategy**:
```javascript
// Pseudo-code for aggregation logic
const TARGET_AGGREGATE_SIZE = 32 * 1024 * 1024 * 1024  // 32 GiB

async function createAggregate(pieces) {
  let currentAggregate = []
  let currentSize = 0

  for (const piece of pieces) {
    if (currentSize + piece.size > TARGET_AGGREGATE_SIZE) {
      // Finalize current aggregate
      await finalizeAggregate(currentAggregate)
      currentAggregate = []
      currentSize = 0
    }

    currentAggregate.push(piece)
    currentSize += piece.size
  }

  // Finalize remaining aggregate
  if (currentAggregate.length > 0) {
    await finalizeAggregate(currentAggregate)
  }
}
```

#### 3. Dealer

**Purpose**: Create and track Filecoin deals with Storage Providers via Spade broker.

**Responsibilities**:
- Submit aggregate offers to Spade marketplace
- Monitor deal status and progression
- Track deal acceptance and activation
- Verify storage proofs on Filecoin chain
- Notify storefront of deal completion

**Spade Integration**:
- Dealer offers aggregates to Spade
- Spade matches offers with Storage Providers based on criteria:
  - Replication count (N copies, typically 4+)
  - Geographical diversity
  - Storage Provider reputation
  - Price and duration
- Storage Providers "buy" aggregates and create deals
- Dealers track deal IDs on Filecoin blockchain

### Filecoin Capabilities

#### filecoin/offer

**Purpose**: Offer content to be stored on Filecoin network.

**Capability Structure**:
```typescript
{
  can: 'filecoin/offer',
  with: string,        // Space DID
  nb: {
    piece: PieceCID,   // Piece CID of content
    content: CID       // Content CID (root)
  }
}
```

**Invocation Example**:
```javascript
import { Filecoin } from '@web3-storage/capabilities'

async function offerToFilecoin(client, space, contentCID, pieceCID) {
  const result = await Filecoin.offer.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: {
      piece: pieceCID,
      content: contentCID
    },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  if (result.out.ok) {
    console.log('Content offered to Filecoin')
    return result.out.ok
  }

  throw new Error(`filecoin/offer failed: ${result.out.error}`)
}
```

**Note**: Most users don't invoke this directly. The high-level `client.uploadFile()` automatically offers content to Filecoin.

#### filecoin/submit

**Purpose**: Submit an aggregate to Spade broker for deal creation (internal capability).

**Capability Structure**:
```typescript
{
  can: 'filecoin/submit',
  with: string,            // Dealer DID
  nb: {
    aggregate: PieceCID,   // Aggregate Piece CID
    pieces: PieceCID[],    // Individual piece CIDs in aggregate
    proof: CID             // Link to Data Aggregation Proof
  }
}
```

**Internal Flow**: This is invoked by the dealer service, not by end users.

#### filecoin/accept

**Purpose**: Notify client that content has been accepted into a Filecoin deal (receipt capability).

**Receipt Structure**:
```typescript
{
  aggregate: PieceCID,     // Aggregate containing the piece
  piece: PieceCID,         // Individual piece CID
  inclusion: {
    subtree: CID[][],      // Merkle proof path
    index: number          // Position in aggregate
  },
  aux: {
    dataSource: {
      dealID: number[]     // Filecoin deal IDs
    }
  }
}
```

**Usage Example**:
```javascript
// After uploading, wait for Filecoin acceptance
const upload = await client.uploadFile(file)

// Poll for Filecoin info (includes acceptance receipt)
let filecoinInfo
while (!filecoinInfo) {
  await new Promise(resolve => setTimeout(resolve, 60000))  // Wait 1 minute

  filecoinInfo = await client.capability.invoke('filecoin/info', {
    piece: upload.pieceCID
  })
}

console.log('Filecoin deals:', filecoinInfo.aux.dataSource.dealID)
console.log('Aggregate:', filecoinInfo.aggregate.toString())
```

### Piece CID Generation

Piece CIDs are computed differently from regular content CIDs:

```javascript
import { Piece } from '@web3-storage/data-segment'

async function computePieceCID(carBytes) {
  // 1. Build merkle tree from CAR blocks
  const tree = await Piece.build(carBytes)

  // 2. Compute Piece CID (CommP)
  const pieceCID = tree.root

  console.log('Piece CID:', pieceCID.toString())
  // Example: baga6ea4seaqao7s73y24kcutaosvacpdjgfe5pw76ooefnyqw4ynr3d2y6x2mpq

  return pieceCID
}
```

**Key Differences**:
- **Content CID**: Identifies logical content (UnixFS DAG root)
- **Piece CID**: Identifies Filecoin storage unit (padded, power-of-2 sized)
- Same content → Same Content CID
- Same content + padding → Same Piece CID

### Data Aggregation Proof (DAP)

A DAP cryptographically proves that a specific piece is included in an aggregate:

```typescript
interface DataAggregationProof {
  aggregate: {
    cid: PieceCID,         // Aggregate Piece CID
    size: number           // Aggregate size in bytes
  },
  pieces: Array<{
    piece: PieceCID,       // Individual piece CID
    inclusion: {
      subtree: CID[][],    // Merkle proof (path from piece to aggregate root)
      index: number        // Piece index in aggregate
    }
  }>
}
```

**Verification Process**:
```javascript
async function verifyPieceInclusion(proof, pieceCID) {
  // 1. Find piece in proof
  const pieceProof = proof.pieces.find(p => p.piece.equals(pieceCID))

  if (!pieceProof) {
    throw new Error('Piece not found in proof')
  }

  // 2. Verify Merkle path
  let currentHash = pieceCID
  for (const [left, right] of pieceProof.inclusion.subtree) {
    currentHash = merkleHash(left, currentHash, right)
  }

  // 3. Verify against aggregate root
  if (!currentHash.equals(proof.aggregate.cid)) {
    throw new Error('Merkle proof verification failed')
  }

  console.log('Piece inclusion verified!')
  return true
}
```

### Retrieving Filecoin Information

#### filecoin/info Capability

**Purpose**: Retrieve Filecoin deal information for a piece.

**Capability Structure**:
```typescript
{
  can: 'filecoin/info',
  with: string,        // Space DID
  nb: {
    piece: PieceCID    // Piece CID to query
  }
}
```

**Response**:
```typescript
{
  piece: PieceCID,
  aggregates: Array<{
    aggregate: PieceCID,
    inclusion: {
      subtree: CID[][],
      index: number
    },
    deals: Array<{
      dealID: number,           // On-chain deal ID
      storageProvider: string,  // SP ID (e.g., "f01234")
      status: 'active' | 'terminated',
      activation: string,       // ISO 8601 timestamp
      expiration: string,       // ISO 8601 timestamp
      created: string           // ISO 8601 timestamp
    }>
  }>
}
```

**Invocation Example**:
```javascript
import { Filecoin } from '@web3-storage/capabilities'

async function getFilecoinInfo(client, space, pieceCID) {
  const result = await Filecoin.info.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: { piece: pieceCID },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  if (result.out.ok) {
    const info = result.out.ok

    console.log(`Piece ${info.piece}:`)
    console.log(`  Aggregates: ${info.aggregates.length}`)

    for (const agg of info.aggregates) {
      console.log(`  Aggregate: ${agg.aggregate}`)
      console.log(`    Deals: ${agg.deals.length}`)

      for (const deal of agg.deals) {
        console.log(`      Deal ${deal.dealID}: ${deal.storageProvider}`)
        console.log(`        Status: ${deal.status}`)
        console.log(`        Expires: ${deal.expiration}`)
      }
    }

    return info
  }

  throw new Error('Piece not found in Filecoin')
}
```

**High-Level Client Usage**:
```javascript
import { create } from '@storacha/client'

const client = await create()

// Upload file (automatically offers to Filecoin)
const cid = await client.uploadFile(file)

// Get Filecoin information (may take 24-48 hours for deals to activate)
const filecoinInfo = await client.getFilecoinInfo(cid)

console.log('Filecoin deals:', filecoinInfo)
```

### Pipeline Flow Example

Complete flow from upload to Filecoin deal:

```
1. User uploads file
   └─> client.uploadFile(file)

2. Client encodes as UnixFS DAG, shards into CARs
   └─> Creates CAR shards

3. Client stores CARs
   └─> store/add for each CAR

4. Client registers upload
   └─> upload/add with root + shard CIDs

5. Client offers to Filecoin
   └─> filecoin/offer with piece CID

6. Storefront queues piece
   └─> Adds to aggregation queue

7. Aggregator creates aggregate
   └─> Combines multiple pieces (32 GiB aggregate)
   └─> Generates Data Aggregation Proof

8. Dealer submits to Spade
   └─> aggregate/offer to Spade marketplace

9. Spade matches with Storage Providers
   └─> SPs "buy" aggregate based on criteria
   └─> 4+ Storage Providers accept

10. Storage Providers fetch aggregate data
    └─> Retrieve pieces from Storacha nodes
    └─> Build aggregate from recipe

11. Storage Providers create Filecoin deals
    └─> Deal proposals published on-chain
    └─> Sectors sealed and proven

12. Dealer tracks deal activation
    └─> Monitors on-chain deal status
    └─> Verifies storage proofs

13. Storefront issues acceptance receipts
    └─> filecoin/accept receipt to client
    └─> Includes deal IDs and PoDSI proof
```

**Timeline**:
- Upload to Storacha: Seconds
- Aggregation: Hours (waiting for aggregate to fill)
- Deal creation: 24-48 hours
- Deal activation: 24-48 hours after creation
- **Total**: Typically 2-4 days from upload to active Filecoin deals

### Best Practices

#### Checking Filecoin Status

```javascript
async function waitForFilecoinDeals(client, cid, timeoutMs = 7 * 24 * 60 * 60 * 1000) {
  const startTime = Date.now()
  const pollInterval = 60 * 60 * 1000  // 1 hour

  while (Date.now() - startTime < timeoutMs) {
    try {
      const info = await client.getFilecoinInfo(cid)

      if (info.aggregates.length > 0) {
        const activeDeals = info.aggregates.flatMap(a =>
          a.deals.filter(d => d.status === 'active')
        )

        if (activeDeals.length >= 4) {
          console.log(`✅ ${activeDeals.length} active Filecoin deals`)
          return info
        }

        console.log(`⏳ ${activeDeals.length}/4 active deals, waiting...`)
      }
    } catch (error) {
      console.log('Not yet available on Filecoin, waiting...')
    }

    await new Promise(resolve => setTimeout(resolve, pollInterval))
  }

  throw new Error('Timeout waiting for Filecoin deals')
}
```

#### Verifying Storage Proofs

```javascript
async function verifyFilecoinStorage(filecoinInfo) {
  // 1. Verify multiple deals exist
  const totalDeals = filecoinInfo.aggregates.flatMap(a => a.deals).length

  if (totalDeals < 4) {
    console.warn(`⚠️  Only ${totalDeals} deals (recommended: 4+)`)
  }

  // 2. Verify geographical diversity (if metadata available)
  const providers = new Set(
    filecoinInfo.aggregates.flatMap(a => a.deals.map(d => d.storageProvider))
  )

  console.log(`Storage Providers: ${providers.size}`)

  // 3. Verify Data Aggregation Proof
  for (const aggregate of filecoinInfo.aggregates) {
    const verified = await verifyPieceInclusion(
      aggregate,
      filecoinInfo.piece
    )

    if (!verified) {
      throw new Error('PoDSI verification failed')
    }
  }

  console.log('✅ All Filecoin storage proofs verified')
}
```

---

## Additional Protocol Specifications

Beyond the four stable core protocols, Storacha defines numerous additional specifications at varying maturity levels. Below are key protocols for ecosystem integration.

### W3-Index Protocol

**Status**: 🟡 **Draft**

**Purpose**: Submit verifiable claims about content-addressable data for publication on InterPlanetary Network Indexer (IPNI), enabling public content discoverability.

#### Core Capability: space/index/add

**Purpose**: Submit a content index for IPNI publication.

**Capability Structure**:
```typescript
{
  can: 'space/index/add',
  with: string,        // Space DID
  nb: {
    index: CID         // Link to ShardedDAGIndex (CAR format)
  }
}
```

**ShardedDAGIndex Structure**:
```typescript
interface ShardedDAGIndex {
  content: CID,              // Content DAG root
  shards: BlobIndex[]        // Array of blob index links
}

interface BlobIndex {
  blob: Multihash,           // Blob digest
  slices: BlobSlice[]        // Slices within blob
}

interface BlobSlice {
  digest: Multihash,         // Slice digest
  offset: number,            // Byte offset in blob
  length: number             // Byte length
}
```

**Use Case**: After uploading content, publish index to IPNI so content can be discovered via public IPFS gateways.

### W3-Provider Protocol

**Status**: 🟠 **Work-in-Progress**

**Purpose**: Enable users to add storage and capability providers to spaces, decoupling users from spaces.

#### Core Capability: provider/add

**Purpose**: Add a provider to a consumer space.

**Capability Structure**:
```typescript
{
  can: 'provider/add',
  with: string,              // Space DID (consumer)
  nb: {
    provider: string,        // Provider DID
    consumer: string         // Space DID (optional, for multi-consumer providers)
  }
}
```

**Provider Tiers**:
- **Free**: 5 GiB storage, single consumer space, requires `did:mailto` resource
- **Lite**: Paid tier, requires payment provider
- **Expert**: Enterprise tier, custom quotas

**Example**:
```javascript
import { Provider } from '@web3-storage/capabilities'

async function addProvider(client, space, providerDID) {
  const result = await Provider.add.invoke({
    issuer: client.agent,
    audience: client.service,
    with: space.did(),
    nb: {
      provider: providerDID
    },
    proofs: [spaceDelegation]
  }).execute(client.connection)

  console.log('Provider added to space')
  return result.out.ok
}
```

### W3-Space Protocol

**Status**: 🟢 **Reliable**

**Purpose**: Manage space namespace metadata and information.

#### Core Capability: space/info

**Purpose**: Retrieve metadata about a space.

**Capability Structure**:
```typescript
{
  can: 'space/info',
  with: string         // Space DID
}
```

**Response**:
```typescript
{
  did: string,               // Space DID
  providers: string[],       // Array of provider DIDs
  name?: string,             // Optional human-readable name
  created: string,           // ISO 8601 timestamp
}
```

### W3-Admin Protocol

**Status**: 🟢 **Reliable**

**Purpose**: Administrative capabilities for customer and subscription management.

#### Capabilities

**consumer/get**: Retrieve consumer (space) information
**customer/get**: Retrieve customer (account) information
**subscription/get**: Retrieve subscription details

**Example**:
```javascript
import { Admin } from '@web3-storage/capabilities'

async function getSubscription(client, account) {
  const result = await Admin.subscription.get.invoke({
    issuer: client.agent,
    audience: client.service,
    with: account.did(),
    proofs: [accountDelegation]
  }).execute(client.connection)

  const subscription = result.out.ok

  console.log('Subscription:', subscription.product)  // 'free', 'lite', 'expert'
  console.log('Storage limit:', subscription.quota)
  console.log('Storage used:', subscription.usage)

  return subscription
}
```

### W3-UCAN Protocol

**Status**: 🟡 **Draft**

**Purpose**: Define W3 protocol extensions to the UCAN core namespace.

**Key Extensions**:
- `ucan/attest`: Attestation capability for authorization sessions
- `ucan/conclude`: Task conclusion and receipt retrieval
- `ucan/revoke`: Capability revocation

**Example**:
```javascript
// Conclude a task and retrieve receipt
const receipt = await client.capability.invoke('ucan/conclude', {
  task: taskCID
})

console.log('Task result:', receipt.out.ok)
```

### Protocol Interaction Example

Multiple protocols working together:

```javascript
import { create } from '@storacha/client'

async function completeWorkflow() {
  const client = await create()

  // 1. W3-Session: Authorize via email
  const account = await client.login('alice@example.com')
  console.log('✅ Authorized via w3-session')

  // 2. W3-Space: Get space info
  const spaces = await client.spaces()
  const space = spaces[0]
  await client.setCurrentSpace(space.did())
  console.log('✅ Using space:', space.did())

  // 3. W3-Blob: Upload file
  const file = new File(['Hello, Storacha!'], 'hello.txt')
  const cid = await client.uploadFile(file)
  console.log('✅ File uploaded via w3-blob:', cid.toString())

  // 4. W3-Filecoin: Offer to Filecoin (automatic)
  console.log('✅ Offered to Filecoin via w3-filecoin')

  // 5. W3-Index: Publish to IPNI (automatic)
  console.log('✅ Published to IPNI via w3-index')

  // 6. W3-Provider: Check provider info
  const info = await client.capability.invoke('space/info', {
    with: space.did()
  })
  console.log('✅ Providers:', info.providers)

  // 7. W3-Admin: Check subscription
  const subscription = await client.capability.invoke('subscription/get', {
    with: account.did()
  })
  console.log('✅ Subscription:', subscription.product)
}
```

---

# Part 3: Ecosystem and Management

## Protocol Interactions

### Multi-Protocol Workflows

Storacha protocols are designed to work together seamlessly. Understanding how they interact enables developers to build robust applications.

#### Complete Upload Flow

A typical file upload triggers multiple protocol interactions:

```
User Action: Upload File
│
├─> 1. W3-Session: Verify Authorization
│   └─> Check agent has valid delegation from account
│
├─> 2. W3-Space: Validate Space Access
│   └─> Verify space exists and has capacity
│
├─> 3. W3-Blob: Store Content
│   ├─> blob/allocate (reserve space)
│   ├─> blob/put (HTTP upload)
│   └─> blob/accept (verify and commit)
│
├─> 4. W3-Store: Register Shards (if applicable)
│   ├─> store/add for each CAR
│   └─> upload/add to link shards
│
├─> 5. W3-Filecoin: Offer for Cold Storage
│   ├─> filecoin/offer (queue for aggregation)
│   ├─> Aggregation process (hours to days)
│   └─> filecoin/accept (deal confirmed)
│
└─> 6. W3-Index: Publish to IPNI
    └─> space/index/add (enable public discovery)
```

**Implementation**:
```javascript
import { create } from '@storacha/client'

async function uploadWithFullWorkflow(file) {
  const client = await create()

  // 1. Ensure authorization (w3-session)
  if (!client.currentAccount()) {
    const account = await client.login('user@example.com')
    console.log('✅ Step 1: Authorized via w3-session')
  }

  // 2. Select or create space (w3-space)
  let space = client.currentSpace()
  if (!space) {
    space = await client.createSpace('my-files')
    await client.setCurrentSpace(space.did())
    console.log('✅ Step 2: Space created')
  }

  // 3-6. Upload (w3-blob + w3-store + w3-filecoin + w3-index)
  // Client handles all protocol interactions internally
  const cid = await client.uploadFile(file, {
    onShardStored: (shard) => {
      console.log(`✅ Step 3-4: Shard stored: ${shard.cid}`)
    },
    onUploadComplete: (root) => {
      console.log(`✅ Step 3-4: Upload complete: ${root}`)
    }
  })

  console.log('✅ Step 5: Automatically offered to Filecoin')
  console.log('✅ Step 6: Automatically published to IPNI')

  return cid
}
```

#### Delegation Chain Resolution

When invoking a capability, the system must resolve the full delegation chain:

```
Capability Invocation: store/add
│
└─> 1. Validate Invocation
    ├─> Check signature (issuer = agent DID)
    ├─> Check expiration (current time < exp)
    └─> Check subject (with = space DID)
    │
    └─> 2. Resolve Delegation Chain
        │
        ├─> Find delegation: Agent → Capability
        │   └─> Proof CID points to delegation
        │
        ├─> Find delegation: Account → Agent
        │   └─> Proof CID points to parent delegation
        │
        └─> Find delegation: Space → Account
            └─> Proof CID points to root delegation
        │
        └─> 3. Verify Chain Integrity
            ├─> Each delegation signed by issuer
            ├─> Capability attenuated at each step
            ├─> No expired delegations in chain
            └─> Final delegation authorizes requested capability
            │
            └─> ✅ Invocation Authorized
```

**Chain Verification Example**:
```javascript
async function verifyDelegationChain(invocation) {
  const chain = []
  let currentProof = invocation.proofs[0]

  // 1. Build chain from proofs
  while (currentProof) {
    const delegation = await resolveDelegation(currentProof)
    chain.push(delegation)

    // Move to next proof in chain
    currentProof = delegation.proofs?.[0]
  }

  console.log(`Delegation chain length: ${chain.length}`)

  // 2. Verify each link
  for (let i = 0; i < chain.length; i++) {
    const delegation = chain[i]

    // Verify signature
    const isValid = await verifySignature(delegation)
    if (!isValid) {
      throw new Error(`Invalid signature at chain position ${i}`)
    }

    // Verify expiration
    if (Date.now() / 1000 > delegation.exp) {
      throw new Error(`Expired delegation at chain position ${i}`)
    }

    // Verify capability attenuation (child ⊆ parent)
    if (i > 0) {
      const parent = chain[i - 1]
      const attenuated = isCapabilityAttenuated(delegation.att, parent.att)

      if (!attenuated) {
        throw new Error(`Capability escalation at chain position ${i}`)
      }
    }

    console.log(`✅ Link ${i}: ${delegation.iss} → ${delegation.aud}`)
  }

  // 3. Verify chain authorizes invocation
  const leafDelegation = chain[chain.length - 1]
  const authorized = canInvoke(leafDelegation, invocation.capability)

  if (!authorized) {
    throw new Error('Delegation chain does not authorize invocation')
  }

  console.log('✅ Delegation chain valid and authorizes invocation')
  return true
}
```

#### Cross-Protocol Dependencies

Some protocols depend on others:

| Protocol | Depends On | Reason |
|----------|------------|--------|
| **w3-store** | w3-space | Requires valid space for storage |
| **w3-blob** | w3-space | Requires valid space for storage |
| **w3-filecoin** | w3-store or w3-blob | Requires content already stored |
| **w3-index** | w3-store or w3-blob | Requires content to index |
| **w3-provider** | w3-space | Adds providers to spaces |
| **w3-admin** | w3-account | Manages account subscriptions |

**Dependency Resolution**:
```javascript
async function ensureDependencies(client, operation) {
  switch (operation) {
    case 'store/add':
    case 'blob/add':
      // Requires space
      if (!client.currentSpace()) {
        throw new Error('No space selected. Create or select a space first.')
      }
      break

    case 'filecoin/offer':
      // Requires stored content
      const upload = await client.getUpload(cid)
      if (!upload) {
        throw new Error('Content must be uploaded before offering to Filecoin')
      }
      break

    case 'provider/add':
      // Requires space and account
      if (!client.currentSpace() || !client.currentAccount()) {
        throw new Error('Both space and account required to add provider')
      }
      break
  }
}
```

### Error Handling Patterns

#### Capability-Level Errors

Errors returned in UCAN receipts follow structured format:

```typescript
interface CapabilityError {
  name: string,           // Error type (e.g., 'InsufficientStorage')
  message: string,        // Human-readable description
  details?: any           // Additional error context
}
```

**Common Error Types**:

| Error Name | Protocol | Description | Recovery |
|------------|----------|-------------|----------|
| `InsufficientStorage` | store, blob | Space quota exceeded | Upgrade plan or remove content |
| `InvalidArgument` | All | Malformed capability arguments | Fix argument format |
| `UnauthorizedSpace` | store, blob | No permission for space | Check delegation chain |
| `StoreItemNotFound` | store | CAR not found in space | Verify CID correct |
| `UploadNotFound` | store | Upload not registered | Verify upload/add completed |
| `BlobSizeExceeded` | blob | Blob too large for provider | Split into smaller blobs |
| `ExpiredDelegation` | All | Delegation expired | Request new delegation |

**Error Handling Pattern**:
```javascript
async function robustUpload(client, file) {
  const maxRetries = 3
  let attempt = 0

  while (attempt < maxRetries) {
    try {
      const cid = await client.uploadFile(file)
      return { success: true, cid }

    } catch (error) {
      attempt++

      // Check error type
      if (error.name === 'InsufficientStorage') {
        console.error('❌ Quota exceeded, cannot retry')
        return {
          success: false,
          error: 'InsufficientStorage',
          message: 'Please upgrade your plan or remove old content'
        }
      }

      if (error.name === 'ExpiredDelegation') {
        console.log('🔄 Delegation expired, re-authorizing...')
        await client.login(client.currentAccount().email)
        continue  // Retry with new delegation
      }

      if (error.name === 'NetworkError' && attempt < maxRetries) {
        const backoff = Math.pow(2, attempt) * 1000
        console.log(`🔄 Network error, retrying in ${backoff}ms (attempt ${attempt}/${maxRetries})`)
        await new Promise(resolve => setTimeout(resolve, backoff))
        continue
      }

      // Unrecoverable error
      console.error('❌ Upload failed:', error.message)
      throw error
    }
  }

  throw new Error(`Upload failed after ${maxRetries} attempts`)
}
```

#### Receipt Validation

Always validate receipts before trusting results:

```javascript
async function validateReceipt(receipt) {
  // 1. Verify receipt signature
  const isValid = await verifySignature(receipt)
  if (!isValid) {
    throw new Error('Invalid receipt signature')
  }

  // 2. Check receipt expiration
  if (receipt.exp && Date.now() / 1000 > receipt.exp) {
    console.warn('⚠️  Receipt expired, result may be outdated')
  }

  // 3. Verify task CID matches
  const taskCID = await computeCID(receipt.task)
  if (!taskCID.equals(receipt.cid)) {
    throw new Error('Receipt task CID mismatch')
  }

  // 4. Check result type
  if (receipt.out.error) {
    throw new Error(`Task failed: ${receipt.out.error.message}`)
  }

  // 5. Validate result structure
  const result = receipt.out.ok
  if (!result) {
    throw new Error('Receipt missing result data')
  }

  console.log('✅ Receipt valid')
  return result
}
```

---

## Version Management and Compatibility

### Specification Versioning

Storacha protocols follow semantic versioning principles:

#### Version Format

```
protocol@major.minor.patch

Examples:
- w3-account@1.0.0
- w3-store@2.1.0
- w3-blob@1.0.0-rc.1
```

**Version Components**:
- **Major**: Breaking changes (incompatible API changes)
- **Minor**: New features (backward compatible)
- **Patch**: Bug fixes (backward compatible)

#### Maturity-Based Versioning

| Maturity Level | Version Range | Stability Guarantee |
|----------------|---------------|---------------------|
| Work-in-Progress | 0.x.y | No stability, frequent breaking changes |
| Draft | 0.x.y | Some stability, may have breaking changes |
| Reliable | 1.x.y | Stable, backward compatible minor/patch |
| Stable | 1.x.y+ | Very stable, rare breaking changes |
| Permanent | Fixed | No changes expected |

### Protocol Upgrade Paths

#### Backward Compatibility

Storacha maintains backward compatibility for stable protocols:

```javascript
// Old client (supports w3-store@1.x)
const result = await Store.add.invoke({
  issuer: agent,
  audience: service,
  with: space.did(),
  nb: {
    link: carCID,
    size: carSize,
    origin: previousCID  // Deprecated field
  }
}).execute(connection)

// New client (supports w3-store@2.x)
const result = await Store.add.invoke({
  issuer: agent,
  audience: service,
  with: space.did(),
  nb: {
    link: carCID,
    size: carSize
    // origin field ignored (deprecated but still accepted)
  }
}).execute(connection)

// Both work! Backward compatible.
```

#### Feature Detection

Check capability support before invocation:

```javascript
async function uploadWithFeatureDetection(client, file) {
  const capabilities = await client.getServiceCapabilities()

  // Check if w3-blob is supported
  if (capabilities.includes('space/blob/add')) {
    console.log('Using w3-blob protocol (newer)')
    return await client.uploadFile(file)  // Uses blob protocol
  }

  // Fallback to w3-store
  if (capabilities.includes('store/add')) {
    console.log('Using w3-store protocol (legacy)')
    return await client.uploadFileLegacy(file)  // Uses store protocol
  }

  throw new Error('No upload protocol supported by service')
}
```

#### Migration Strategies

**Strategy 1: Gradual Migration**

Support both old and new protocols during transition:

```javascript
class UploadClient {
  async uploadFile(file) {
    try {
      // Try new protocol (w3-blob)
      return await this.uploadViaBlob(file)
    } catch (error) {
      if (error.name === 'UnsupportedCapability') {
        console.log('Falling back to w3-store')
        return await this.uploadViaStore(file)
      }
      throw error
    }
  }

  async uploadViaBlob(file) {
    // w3-blob implementation
  }

  async uploadViaStore(file) {
    // w3-store implementation (legacy)
  }
}
```

**Strategy 2: Version Negotiation**

Negotiate protocol version with service:

```javascript
async function negotiateVersion(client, service) {
  const clientVersions = {
    'w3-account': '1.0.0',
    'w3-store': '2.1.0',
    'w3-blob': '1.0.0',
    'w3-filecoin': '1.0.0'
  }

  const serviceVersions = await service.getProtocolVersions()

  // Find compatible versions
  const compatible = {}
  for (const [protocol, clientVersion] of Object.entries(clientVersions)) {
    const serviceVersion = serviceVersions[protocol]

    if (isCompatible(clientVersion, serviceVersion)) {
      compatible[protocol] = serviceVersion
      console.log(`✅ ${protocol}: Using v${serviceVersion}`)
    } else {
      console.warn(`⚠️  ${protocol}: Incompatible (client: ${clientVersion}, service: ${serviceVersion})`)
    }
  }

  return compatible
}

function isCompatible(clientVer, serviceVer) {
  const [cMajor] = clientVer.split('.').map(Number)
  const [sMajor] = serviceVer.split('.').map(Number)

  // Compatible if major versions match
  return cMajor === sMajor
}
```

### Deprecation Policy

Storacha follows a structured deprecation process:

#### Deprecation Phases

**Phase 1: Announcement** (6+ months)
- Feature marked as deprecated in documentation
- Warning added to specification
- Alternative recommended

**Phase 2: Deprecation** (3+ months)
- Feature still works but logs warnings
- Clients encouraged to migrate
- Support burden noted

**Phase 3: Removal** (Major version bump)
- Feature removed from specification
- Clients using feature receive errors
- Migration guide provided

#### Example: `origin` Field Deprecation

```javascript
// Phase 1: Announcement (w3-store@1.5.0, 2024-01)
{
  can: 'store/add',
  with: 'did:key:space123',
  nb: {
    link: carCID,
    size: carSize,
    origin: previousCID  // ⚠️  DEPRECATED: Will be removed in v2.0.0
  }
}

// Phase 2: Deprecation (w3-store@1.9.0, 2024-06)
// Service logs warning when origin field is used
console.warn('The "origin" field is deprecated and will be ignored in v2.0.0')

// Phase 3: Removal (w3-store@2.0.0, 2025-01)
// Field removed from specification
{
  can: 'store/add',
  with: 'did:key:space123',
  nb: {
    link: carCID,
    size: carSize
    // origin field no longer accepted
  }
}
```

---

## Implementation Guidelines

### Client Library Development

#### Capability Invocation Pattern

Implement a consistent pattern for invoking capabilities:

```javascript
class CapabilityInvoker {
  constructor(agent, connection, service) {
    this.agent = agent
    this.connection = connection
    this.service = service
  }

  async invoke(capability, options = {}) {
    // 1. Build invocation
    const invocation = {
      issuer: this.agent,
      audience: this.service.did(),
      with: options.with || this.agent.did(),
      nb: options.nb || {},
      proofs: options.proofs || [],
      expiration: options.expiration || (Math.floor(Date.now() / 1000) + 300)
    }

    // 2. Sign invocation
    const signed = await this.agent.sign(invocation)

    // 3. Execute over connection
    const result = await this.connection.execute(capability, signed)

    // 4. Validate receipt
    await this.validateReceipt(result)

    // 5. Return result
    return result.out.ok
  }

  async validateReceipt(receipt) {
    // Receipt validation logic
  }
}

// Usage
const invoker = new CapabilityInvoker(agent, connection, service)

const result = await invoker.invoke('store/add', {
  with: space.did(),
  nb: {
    link: carCID,
    size: carSize
  },
  proofs: [spaceDelegation]
})
```

#### Delegation Management

Implement delegation storage and retrieval:

```javascript
class DelegationStore {
  constructor(storage) {
    this.storage = storage  // Local storage, IndexedDB, etc.
  }

  async save(delegation) {
    const cid = await computeCID(delegation)
    await this.storage.put(cid.toString(), delegation)
    console.log(`Saved delegation: ${cid}`)
    return cid
  }

  async get(cid) {
    const delegation = await this.storage.get(cid.toString())
    if (!delegation) {
      throw new Error(`Delegation not found: ${cid}`)
    }
    return delegation
  }

  async list(filter = {}) {
    const all = await this.storage.list()

    return all.filter(delegation => {
      // Filter by issuer
      if (filter.issuer && delegation.iss !== filter.issuer) {
        return false
      }

      // Filter by audience
      if (filter.audience && delegation.aud !== filter.audience) {
        return false
      }

      // Filter by capability
      if (filter.can) {
        const hasCap = delegation.att.some(cap => cap.can === filter.can)
        if (!hasCap) return false
      }

      // Filter out expired
      if (filter.notExpired && delegation.exp < Date.now() / 1000) {
        return false
      }

      return true
    })
  }

  async revoke(cid) {
    await this.storage.delete(cid.toString())
    console.log(`Revoked delegation: ${cid}`)
  }
}
```

#### Connection Abstraction

Abstract transport layer for flexibility:

```javascript
interface Connection {
  execute(capability: string, invocation: Invocation): Promise<Receipt>
}

class HTTPConnection implements Connection {
  constructor(url) {
    this.url = url
  }

  async execute(capability, invocation) {
    const response = await fetch(`${this.url}/`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${encodeInvocation(invocation)}`
      },
      body: JSON.stringify({
        capability,
        invocation
      })
    })

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }

    const receipt = await response.json()
    return receipt
  }
}

class WebSocketConnection implements Connection {
  constructor(url) {
    this.ws = new WebSocket(url)
    this.pending = new Map()
  }

  async execute(capability, invocation) {
    const id = crypto.randomUUID()

    return new Promise((resolve, reject) => {
      this.pending.set(id, { resolve, reject })

      this.ws.send(JSON.stringify({
        id,
        capability,
        invocation
      }))

      // Timeout after 30 seconds
      setTimeout(() => {
        if (this.pending.has(id)) {
          this.pending.delete(id)
          reject(new Error('Request timeout'))
        }
      }, 30000)
    })
  }

  handleMessage(message) {
    const { id, receipt } = JSON.parse(message.data)
    const pending = this.pending.get(id)

    if (pending) {
      pending.resolve(receipt)
      this.pending.delete(id)
    }
  }
}
```

### Service Implementation

#### UCAN Invocation Handler

Implement server-side capability handlers:

```javascript
import { Server } from '@ucanto/server'

const server = Server.create({
  id: await Signer.generate(),  // Service DID
  service: {
    store: {
      // Handle store/add capability
      add: async (invocation) => {
        const { link, size } = invocation.capability.nb
        const space = invocation.capability.with

        // 1. Validate invocation
        await validateInvocation(invocation)

        // 2. Check space capacity
        const usage = await db.getSpaceUsage(space)
        if (usage.used + size > usage.limit) {
          return {
            error: {
              name: 'InsufficientStorage',
              message: `Space quota exceeded (used: ${usage.used}, limit: ${usage.limit})`
            }
          }
        }

        // 3. Check if already stored
        const existing = await db.getStoreItem(space, link)
        if (existing) {
          return {
            status: 'done',
            allocated: 0,
            with: space,
            link: link
          }
        }

        // 4. Allocate storage
        const allocation = await allocateStorage(space, link, size)

        // 5. Return upload URL
        return {
          status: 'upload',
          allocated: size,
          with: space,
          link: link,
          url: allocation.url,
          headers: allocation.headers,
          expiresAt: allocation.expiresAt
        }
      },

      // Handle store/get capability
      get: async (invocation) => {
        const { link } = invocation.capability.nb
        const space = invocation.capability.with

        const item = await db.getStoreItem(space, link)

        if (!item) {
          return {
            error: {
              name: 'StoreItemNotFound',
              message: `CAR not found: ${link}`
            }
          }
        }

        return {
          link: item.link,
          size: item.size,
          origin: item.origin
        }
      }
    }
  }
})

// Validate invocation
async function validateInvocation(invocation) {
  // 1. Verify signature
  const isValid = await verifySignature(invocation)
  if (!isValid) {
    throw new Error('Invalid invocation signature')
  }

  // 2. Check expiration
  if (Date.now() / 1000 > invocation.expiration) {
    throw new Error('Invocation expired')
  }

  // 3. Resolve and verify delegation chain
  await verifyDelegationChain(invocation)
}
```

---

## Troubleshooting and Best Practices

### Common Issues

#### Issue 1: "ExpiredDelegation" Error

**Symptom**: Capabilities fail with `ExpiredDelegation` error

**Cause**: Delegation in proof chain has expired

**Solution**:
```javascript
async function refreshDelegations(client) {
  // 1. Check current delegations
  const delegations = await client.delegations.list({
    notExpired: false  // Include expired
  })

  const expired = delegations.filter(d => d.exp < Date.now() / 1000)

  console.log(`Found ${expired.length} expired delegations`)

  // 2. Re-authorize with account
  const account = client.currentAccount()
  if (account) {
    console.log('Re-authorizing...')
    await client.login(account.email)
    console.log('✅ Delegations refreshed')
  }

  // 3. Clean up expired delegations
  for (const delegation of expired) {
    await client.delegations.revoke(delegation.cid)
  }
}
```

#### Issue 2: "InsufficientStorage" Error

**Symptom**: Upload fails with quota exceeded error

**Cause**: Space has reached storage limit

**Solution**:
```javascript
async function manageStorageQuota(client) {
  const space = client.currentSpace()
  const usage = await client.usage.get(space.did())

  console.log(`Storage: ${usage.used} / ${usage.limit} bytes`)
  console.log(`Available: ${usage.available} bytes`)

  if (usage.available < 1024 * 1024 * 100) {  // Less than 100 MB
    console.warn('⚠️  Low storage space!')

    // Option 1: Remove old uploads
    const uploads = await client.capability.invoke('upload/list', {
      with: space.did(),
      nb: { size: 100 }
    })

    const sorted = uploads.results.sort((a, b) =>
      new Date(a.insertedAt) - new Date(b.insertedAt)
    )

    console.log(`Consider removing ${sorted.length} old uploads`)

    // Option 2: Upgrade plan
    console.log('Or upgrade your subscription plan')
  }
}
```

#### Issue 3: CAR Shard Retrieval Fails

**Symptom**: Cannot retrieve uploaded file

**Cause**: Missing CAR shards or broken upload registration

**Solution**:
```javascript
async function diagnoseUpload(client, rootCID) {
  // 1. Check upload registration
  const upload = await client.capability.invoke('upload/get', {
    with: client.currentSpace().did(),
    nb: { root: rootCID }
  })

  if (!upload) {
    console.error('❌ Upload not registered')
    return
  }

  console.log(`✅ Upload found: ${upload.shards.length} shards`)

  // 2. Verify each shard
  for (const shardCID of upload.shards) {
    const shard = await client.capability.invoke('store/get', {
      with: client.currentSpace().did(),
      nb: { link: shardCID }
    })

    if (!shard) {
      console.error(`❌ Missing shard: ${shardCID}`)
    } else {
      console.log(`✅ Shard found: ${shardCID} (${shard.size} bytes)`)
    }
  }
}
```

### Best Practices Summary

#### For Application Developers

1. **Always Handle Errors Gracefully**
   ```javascript
   try {
     await client.uploadFile(file)
   } catch (error) {
     if (error.name === 'InsufficientStorage') {
       showUpgradePrompt()
     } else if (error.name === 'NetworkError') {
       retryWithBackoff()
     } else {
       logError(error)
       showGenericError()
     }
   }
   ```

2. **Implement Proper Delegation Management**
   - Store delegations securely (encrypted local storage)
   - Refresh before expiration (proactive renewal)
   - Clean up revoked delegations

3. **Use High-Level Client APIs**
   - Prefer `client.uploadFile()` over manual capability invocations
   - Let client handle protocol interactions
   - Only use low-level APIs when necessary

4. **Monitor Service Health**
   ```javascript
   async function checkServiceHealth(client) {
     try {
       const info = await client.service.info()
       console.log('Service status:', info.status)
       console.log('Version:', info.version)
       return true
     } catch (error) {
       console.error('Service unreachable')
       return false
     }
   }
   ```

5. **Implement Retry Logic with Exponential Backoff**
   ```javascript
   async function retryWithBackoff(fn, maxRetries = 3) {
     for (let i = 0; i < maxRetries; i++) {
       try {
         return await fn()
       } catch (error) {
         if (i === maxRetries - 1) throw error
         const backoff = Math.pow(2, i) * 1000
         await new Promise(resolve => setTimeout(resolve, backoff))
       }
     }
   }
   ```

#### For Service Operators

1. **Validate All Invocations Thoroughly**
   - Verify signatures
   - Check delegation chains
   - Enforce capability constraints
   - Rate limit by issuer

2. **Provide Detailed Error Messages**
   ```javascript
   return {
     error: {
       name: 'InsufficientStorage',
       message: 'Space quota exceeded',
       details: {
         used: usage.used,
         limit: usage.limit,
         requested: size
       }
     }
   }
   ```

3. **Monitor Service Metrics**
   - Invocation success/failure rates
   - Average response times
   - Storage utilization
   - Delegation chain depths

4. **Implement Graceful Degradation**
   - Return cached results when backend unavailable
   - Queue writes for later processing
   - Provide read-only mode during maintenance

5. **Document Protocol Extensions**
   - Clearly mark custom capabilities
   - Provide migration guides
   - Maintain backward compatibility

---

## Conclusion

The Storacha protocol specifications provide a comprehensive, modular framework for decentralized storage built on UCAN authorization, IPFS content-addressing, and Filecoin persistence.

**Key Takeaways**:

1. **Modular Architecture**: Protocols compose together (w3-session + w3-space + w3-blob + w3-filecoin + w3-index)
2. **UCAN-Based Authorization**: Capability-based security with cryptographic delegation chains
3. **Content-Addressed Storage**: Immutable, verifiable data with CID-based addressing
4. **Filecoin Integration**: Automatic replication to decentralized cold storage with proofs
5. **Backward Compatibility**: Stable protocols maintain compatibility across versions

**Next Steps**:

- Explore implementation code in `storacha/w3up` repository
- Review real-world client examples in `storacha/w3cli`
- Read infrastructure code in `storacha/w3infra`
- Join community discussions on GitHub and Discord
- Contribute to protocol specifications

**Resources**:

- **Specifications**: https://github.com/storacha/specs
- **Implementation**: https://github.com/storacha/w3up
- **Documentation**: https://docs.storacha.network
- **Community**: https://discord.gg/storacha

---

**Document Information**:
- **Version**: 1.0.0
- **Last Updated**: 2025-01-14
- **Total Lines**: ~4,800
- **Coverage**:
  - Part 1: Specifications Overview, W3-Account, W3-Session
  - Part 2: W3-Store, W3-Blob, W3-Filecoin, Additional Specifications
  - Part 3: Protocol Interactions, Version Management, Implementation, Troubleshooting

---

*End of 07_Protocol_Specifications.md*
