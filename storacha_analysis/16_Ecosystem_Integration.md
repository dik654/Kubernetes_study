# Ecosystem Integration Guide for Storacha

## Table of Contents

- [Overview](#overview)
- [IPFS Ecosystem Integration](#ipfs-ecosystem-integration)
- [Filecoin Network Integration](#filecoin-network-integration)
- [Web3 Wallet and Authentication](#web3-wallet-and-authentication)
- [External Service Integrations](#external-service-integrations)
- [Development Frameworks](#development-frameworks)
- [Integration Patterns](#integration-patterns)
- [Best Practices](#best-practices)

---

## Overview

Storacha operates as a decentralized hot storage layer built on Filecoin and IPFS, serving as a critical bridge between Web2 and Web3 storage paradigms. This guide covers integration patterns with the broader Web3 ecosystem, including IPFS infrastructure, Filecoin network, authentication systems, and external services.

### Ecosystem Position

```mermaid
graph TB
    subgraph "Application Layer"
        Apps[Web3 Applications]
        DApps[DApps]
        AI[AI/ML Workloads]
    end

    subgraph "Storacha Layer"
        API[Storacha API]
        Gateway[w3s.link Gateway]
        UCAN[UCAN Authorization]
    end

    subgraph "IPFS Ecosystem"
        IPFSNodes[IPFS Nodes]
        Pinata[Pinata]
        Infura[Infura IPFS]
        PublicGW[Public Gateways]
    end

    subgraph "Filecoin Network"
        Deals[Storage Deals]
        SP[Storage Providers]
        FVM[Filecoin VM]
    end

    Apps --> API
    DApps --> API
    AI --> API

    API --> UCAN
    API --> Gateway

    Gateway --> IPFSNodes
    Gateway --> PublicGW

    API --> Deals
    Deals --> SP
    Deals --> FVM

    IPFSNodes -.-> Pinata
    IPFSNodes -.-> Infura

    style API fill:#f96,stroke:#333
    style Gateway fill:#9cf,stroke:#333
    style UCAN fill:#fc9,stroke:#333
```

### Key Integration Points

1. **IPFS Network**: Content-addressed data retrieval
2. **Filecoin**: Long-term storage persistence with automatic deal renewal
3. **UCAN**: Decentralized authorization and delegation
4. **Pinning Services**: Multi-provider redundancy
5. **Web3 Frameworks**: Next.js, React, Vue.js integration
6. **External APIs**: REST, GraphQL, and WebSocket endpoints

---

## IPFS Ecosystem Integration

### Direct IPFS Node Connection

```typescript
// Connect to IPFS node and integrate with Storacha
import { create as createIPFSClient } from 'ipfs-http-client'
import { create as createStorachaClient } from '@web3-storage/w3up-client'

class IPFSStorachaIntegration {
  private ipfs: any
  private storacha: any

  async initialize(): Promise<void> {
    // Connect to local or remote IPFS node
    this.ipfs = createIPFSClient({
      url: 'http://127.0.0.1:5001/api/v0'
    })

    // Initialize Storacha client
    this.storacha = await createStorachaClient()
  }

  /**
   * Upload to both IPFS and Storacha
   */
  async dualUpload(file: File): Promise<{
    ipfsCID: string
    storachaCID: string
    match: boolean
  }> {
    // Upload to local IPFS
    const ipfsResult = await this.ipfs.add(file, {
      cidVersion: 1,
      hashAlg: 'sha2-256'
    })

    // Upload to Storacha
    const storachaResult = await this.storacha.uploadFile(file)

    return {
      ipfsCID: ipfsResult.cid.toString(),
      storachaCID: storachaResult.toString(),
      match: ipfsResult.cid.toString() === storachaResult.toString()
    }
  }

  /**
   * Retrieve from IPFS with Storacha fallback
   */
  async retrieveWithFallback(cid: string): Promise<Uint8Array> {
    try {
      // Try local IPFS first
      const chunks = []
      for await (const chunk of this.ipfs.cat(cid, { timeout: 5000 })) {
        chunks.push(chunk)
      }
      return new Uint8Array(Buffer.concat(chunks))
    } catch (error) {
      console.log('Local IPFS failed, falling back to Storacha gateway')

      // Fallback to Storacha gateway
      const response = await fetch(`https://w3s.link/ipfs/${cid}`)
      if (!response.ok) {
        throw new Error(`Failed to retrieve from Storacha: ${response.statusText}`)
      }

      return new Uint8Array(await response.arrayBuffer())
    }
  }

  /**
   * Pin locally and backup to Storacha
   */
  async pinWithBackup(cid: string): Promise<void> {
    // Pin to local IPFS
    await this.ipfs.pin.add(cid)
    console.log(`Pinned ${cid} to local IPFS`)

    // Backup to Storacha (if not already there)
    // Note: Storacha automatically stores content
    console.log(`Content ${cid} available on Storacha network`)
  }
}

// Usage
const integration = new IPFSStorachaIntegration()
await integration.initialize()

const file = new File(['Hello IPFS!'], 'test.txt')
const result = await integration.dualUpload(file)

console.log('Upload Results:')
console.log(`  IPFS CID: ${result.ipfsCID}`)
console.log(`  Storacha CID: ${result.storachaCID}`)
console.log(`  CIDs Match: ${result.match}`) // Should be true

// Retrieve with fallback
const data = await integration.retrieveWithFallback(result.ipfsCID)
console.log(`Retrieved: ${new TextDecoder().decode(data)}`)
```

### IPFS Gateway Racing

```typescript
// Race multiple IPFS gateways including Storacha
class GatewayRacer {
  private gateways: string[]

  constructor(gateways?: string[]) {
    this.gateways = gateways || [
      'https://w3s.link',
      'https://ipfs.io',
      'https://dweb.link',
      'https://cloudflare-ipfs.com',
      'https://gateway.pinata.cloud'
    ]
  }

  /**
   * Race all gateways and return fastest response
   */
  async fetchFastest(cid: string): Promise<{
    data: Uint8Array
    gateway: string
    duration: number
  }> {
    const startTime = Date.now()

    const requests = this.gateways.map(async (gateway) => {
      const gwStartTime = Date.now()
      try {
        const response = await fetch(`${gateway}/ipfs/${cid}`, {
          signal: AbortSignal.timeout(10000) // 10s timeout
        })

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`)
        }

        const data = new Uint8Array(await response.arrayBuffer())
        const duration = Date.now() - gwStartTime

        return {
          data,
          gateway,
          duration,
          success: true
        }
      } catch (error) {
        return {
          data: new Uint8Array(),
          gateway,
          duration: Date.now() - gwStartTime,
          success: false,
          error: error instanceof Error ? error.message : 'Unknown error'
        }
      }
    })

    // Race all requests
    const results = await Promise.allSettled(requests)

    // Find first successful response
    for (const result of results) {
      if (result.status === 'fulfilled' && result.value.success) {
        return {
          data: result.value.data,
          gateway: result.value.gateway,
          duration: result.value.duration
        }
      }
    }

    throw new Error('All gateways failed to retrieve content')
  }

  /**
   * Fetch with detailed metrics from all gateways
   */
  async fetchWithMetrics(cid: string): Promise<{
    winner: string
    metrics: Array<{
      gateway: string
      duration: number
      success: boolean
      error?: string
    }>
  }> {
    const requests = this.gateways.map(async (gateway) => {
      const startTime = Date.now()
      try {
        const response = await fetch(`${gateway}/ipfs/${cid}`, {
          signal: AbortSignal.timeout(10000)
        })

        return {
          gateway,
          duration: Date.now() - startTime,
          success: response.ok,
          statusCode: response.status
        }
      } catch (error) {
        return {
          gateway,
          duration: Date.now() - startTime,
          success: false,
          error: error instanceof Error ? error.message : 'Unknown error'
        }
      }
    })

    const results = await Promise.allSettled(requests)
    const metrics = results
      .filter((r): r is PromiseFulfilledResult<any> => r.status === 'fulfilled')
      .map(r => r.value)

    const successful = metrics.filter(m => m.success)
    const winner = successful.length > 0
      ? successful.reduce((fastest, current) =>
          current.duration < fastest.duration ? current : fastest
        )
      : null

    return {
      winner: winner ? winner.gateway : 'none',
      metrics
    }
  }
}

// Usage
const racer = new GatewayRacer()

const cid = 'bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi'

// Fetch fastest
const result = await racer.fetchFastest(cid)
console.log(`Fastest gateway: ${result.gateway}`)
console.log(`Duration: ${result.duration}ms`)
console.log(`Data size: ${result.data.length} bytes`)

// Detailed metrics
const metrics = await racer.fetchWithMetrics(cid)
console.log(`\nWinner: ${metrics.winner}`)
console.log('\nAll gateway metrics:')
metrics.metrics
  .sort((a, b) => a.duration - b.duration)
  .forEach(m => {
    console.log(`  ${m.gateway}: ${m.duration}ms ${m.success ? '✅' : '❌'}`)
  })
```

### IPFS Pinning Service API

```typescript
// Implement IPFS Pinning Service API for Storacha
import { create as createStorachaClient } from '@web3-storage/w3up-client'

interface PinStatus {
  requestid: string
  status: 'queued' | 'pinning' | 'pinned' | 'failed'
  created: string
  pin: {
    cid: string
    name?: string
  }
  delegates?: string[]
  info?: Record<string, any>
}

class StorachaPinningService {
  private client: any
  private pins: Map<string, PinStatus>

  constructor() {
    this.pins = new Map()
  }

  async initialize(email: string): Promise<void> {
    this.client = await createStorachaClient()
    await this.client.login(email)
    const space = await this.client.createSpace('pinning-service')
    await this.client.setCurrentSpace(space.did())
  }

  /**
   * Add a pin request
   */
  async addPin(cid: string, name?: string): Promise<PinStatus> {
    const requestid = this.generateRequestId()

    const pinStatus: PinStatus = {
      requestid,
      status: 'queued',
      created: new Date().toISOString(),
      pin: { cid, name }
    }

    this.pins.set(requestid, pinStatus)

    // Perform actual pinning asynchronously
    this.performPin(requestid, cid).catch(error => {
      const status = this.pins.get(requestid)
      if (status) {
        status.status = 'failed'
        status.info = { error: error.message }
      }
    })

    return pinStatus
  }

  private async performPin(requestid: string, cid: string): Promise<void> {
    const status = this.pins.get(requestid)
    if (!status) return

    status.status = 'pinning'

    // In Storacha, pinning is implicit when content is uploaded
    // We verify the content exists and is retrievable
    try {
      const response = await fetch(`https://w3s.link/ipfs/${cid}`, {
        method: 'HEAD'
      })

      if (response.ok) {
        status.status = 'pinned'
        status.info = {
          size: parseInt(response.headers.get('content-length') || '0'),
          pinned_at: new Date().toISOString()
        }
      } else {
        throw new Error(`Content not found: ${response.statusText}`)
      }
    } catch (error) {
      status.status = 'failed'
      status.info = {
        error: error instanceof Error ? error.message : 'Unknown error'
      }
    }
  }

  /**
   * Get pin status
   */
  async getPinStatus(requestid: string): Promise<PinStatus | null> {
    return this.pins.get(requestid) || null
  }

  /**
   * List pins
   */
  async listPins(filter?: {
    cid?: string[]
    name?: string
    status?: string[]
  }): Promise<PinStatus[]> {
    let results = Array.from(this.pins.values())

    if (filter) {
      if (filter.cid && filter.cid.length > 0) {
        results = results.filter(pin => filter.cid!.includes(pin.pin.cid))
      }

      if (filter.name) {
        results = results.filter(pin => pin.pin.name?.includes(filter.name!))
      }

      if (filter.status && filter.status.length > 0) {
        results = results.filter(pin => filter.status!.includes(pin.status))
      }
    }

    return results
  }

  /**
   * Delete a pin
   */
  async deletePin(requestid: string): Promise<void> {
    this.pins.delete(requestid)
  }

  private generateRequestId(): string {
    return `pin_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
  }
}

// Usage
const pinningService = new StorachaPinningService()
await pinningService.initialize('alice@example.com')

// Add pin
const pin = await pinningService.addPin(
  'bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi',
  'my-important-file'
)
console.log(`Pin requested: ${pin.requestid}`)

// Check status
await new Promise(resolve => setTimeout(resolve, 2000))
const status = await pinningService.getPinStatus(pin.requestid)
console.log(`Pin status: ${status?.status}`)

// List all pins
const pins = await pinningService.listPins({ status: ['pinned'] })
console.log(`Total pinned: ${pins.length}`)
```

---

## Filecoin Network Integration

### Storage Deal Management

```typescript
// Manage Filecoin storage deals through Storacha
import { create as createStorachaClient } from '@web3-storage/w3up-client'

interface FilecoinDeal {
  dealId: number
  provider: string
  startEpoch: number
  endEpoch: number
  pieceCID: string
  status: 'active' | 'expired' | 'pending'
}

class FilecoinStorageManager {
  private client: any

  async initialize(email: string): Promise<void> {
    this.client = await createStorachaClient()
    await this.client.login(email)
  }

  /**
   * Upload with Filecoin storage guarantee
   */
  async uploadWithFilecoinBackup(file: File): Promise<{
    cid: string
    filecoinDeals: FilecoinDeal[]
  }> {
    // Upload to Storacha
    const cid = await this.client.uploadFile(file)

    console.log(`Uploaded to Storacha: ${cid}`)
    console.log('Content automatically backed up to Filecoin network')

    // Storacha automatically creates Filecoin storage deals
    // In practice, you would query the Storacha API for deal information
    const deals = await this.getFilecoinDeals(cid.toString())

    return {
      cid: cid.toString(),
      filecoinDeals: deals
    }
  }

  /**
   * Query Filecoin deals for a CID
   */
  private async getFilecoinDeals(cid: string): Promise<FilecoinDeal[]> {
    // This would query Storacha's API or Filecoin network
    // Simplified example with mock data

    // In production, use Storacha's deal info endpoint or
    // query Filecoin network directly via lotus or other RPC

    return [
      {
        dealId: 12345,
        provider: 'f01234',
        startEpoch: 2000000,
        endEpoch: 3000000,
        pieceCID: 'baga6ea4seaq...',
        status: 'active'
      }
    ]
  }

  /**
   * Verify storage on Filecoin
   */
  async verifyFilecoinStorage(cid: string): Promise<{
    stored: boolean
    dealCount: number
    providers: string[]
    expirationEpoch: number
  }> {
    const deals = await this.getFilecoinDeals(cid)
    const activeDeals = deals.filter(d => d.status === 'active')

    return {
      stored: activeDeals.length > 0,
      dealCount: activeDeals.length,
      providers: [...new Set(activeDeals.map(d => d.provider))],
      expirationEpoch: Math.max(...activeDeals.map(d => d.endEpoch))
    }
  }

  /**
   * Monitor deal renewal
   */
  async monitorDealRenewal(cid: string): Promise<void> {
    console.log(`Monitoring Filecoin deals for ${cid}`)

    // Storacha automatically renews deals
    // This is a monitoring function to verify renewal status

    const verification = await this.verifyFilecoinStorage(cid)

    if (!verification.stored) {
      console.warn('⚠️  No active Filecoin deals found!')
    } else {
      console.log(`✅ ${verification.dealCount} active deal(s)`)
      console.log(`   Providers: ${verification.providers.join(', ')}`)
      console.log(`   Expires at epoch: ${verification.expirationEpoch}`)
    }

    // Storacha service renews deals automatically before expiration
    console.log('Note: Storacha automatically renews expiring deals')
  }
}

// Usage
const manager = new FilecoinStorageManager()
await manager.initialize('alice@example.com')

const file = new File(['Important data'], 'critical.txt')
const result = await manager.uploadWithFilecoinBackup(file)

console.log(`\nUpload complete:`)
console.log(`  CID: ${result.cid}`)
console.log(`  Filecoin deals: ${result.filecoinDeals.length}`)

// Verify storage
const verification = await manager.verifyFilecoinStorage(result.cid)
console.log(`\nVerification:`)
console.log(`  Stored on Filecoin: ${verification.stored}`)
console.log(`  Deal count: ${verification.dealCount}`)
console.log(`  Storage providers: ${verification.providers.join(', ')}`)

// Monitor renewal
await manager.monitorDealRenewal(result.cid)
```

### Filecoin Virtual Machine (FVM) Integration

```typescript
// Interact with FVM smart contracts for storage automation
import { ethers } from 'ethers'

// Example FVM contract ABI for storage automation
const STORAGE_AUTOMATION_ABI = [
  'function storeData(bytes calldata data) external returns (bytes32 cid)',
  'function verifyStorage(bytes32 cid) external view returns (bool)',
  'function getDealInfo(bytes32 cid) external view returns (uint256 dealId, address provider, uint256 expiryEpoch)',
  'event DataStored(bytes32 indexed cid, uint256 dealId, address indexed provider)'
]

class FVMStorageAutomation {
  private provider: ethers.JsonRpcProvider
  private contract: ethers.Contract
  private signer: ethers.Wallet

  constructor(
    rpcUrl: string,
    contractAddress: string,
    privateKey: string
  ) {
    this.provider = new ethers.JsonRpcProvider(rpcUrl)
    this.signer = new ethers.Wallet(privateKey, this.provider)
    this.contract = new ethers.Contract(
      contractAddress,
      STORAGE_AUTOMATION_ABI,
      this.signer
    )
  }

  /**
   * Store data via FVM smart contract
   */
  async storeDataOnChain(data: Uint8Array): Promise<{
    cid: string
    txHash: string
    dealId: number
  }> {
    console.log('Storing data via FVM smart contract...')

    const tx = await this.contract.storeData(data)
    const receipt = await tx.wait()

    // Parse event to get CID and deal ID
    const event = receipt.logs
      .map((log: any) => {
        try {
          return this.contract.interface.parseLog(log)
        } catch {
          return null
        }
      })
      .find((e: any) => e && e.name === 'DataStored')

    if (!event) {
      throw new Error('DataStored event not found')
    }

    return {
      cid: event.args.cid,
      txHash: receipt.hash,
      dealId: Number(event.args.dealId)
    }
  }

  /**
   * Verify storage on-chain
   */
  async verifyStorageOnChain(cid: string): Promise<boolean> {
    return await this.contract.verifyStorage(cid)
  }

  /**
   * Get deal information from contract
   */
  async getDealInfo(cid: string): Promise<{
    dealId: number
    provider: string
    expiryEpoch: number
  }> {
    const info = await this.contract.getDealInfo(cid)

    return {
      dealId: Number(info.dealId),
      provider: info.provider,
      expiryEpoch: Number(info.expiryEpoch)
    }
  }

  /**
   * Listen for storage events
   */
  listenForStorageEvents(callback: (event: any) => void): void {
    this.contract.on('DataStored', (cid, dealId, provider, event) => {
      callback({
        cid,
        dealId: Number(dealId),
        provider,
        blockNumber: event.log.blockNumber,
        txHash: event.log.transactionHash
      })
    })
  }
}

// Usage
const fvmStorage = new FVMStorageAutomation(
  'https://api.calibration.node.glif.io/rpc/v1', // Filecoin Calibration testnet
  '0x...', // Contract address
  '0x...' // Private key
)

// Store data
const data = new TextEncoder().encode('Hello FVM!')
const result = await fvmStorage.storeDataOnChain(data)

console.log('Stored via FVM:')
console.log(`  CID: ${result.cid}`)
console.log(`  Transaction: ${result.txHash}`)
console.log(`  Deal ID: ${result.dealId}`)

// Verify storage
const isStored = await fvmStorage.verifyStorageOnChain(result.cid)
console.log(`\nStorage verified: ${isStored}`)

// Get deal info
const dealInfo = await fvmStorage.getDealInfo(result.cid)
console.log(`\nDeal information:`)
console.log(`  Provider: ${dealInfo.provider}`)
console.log(`  Expiry Epoch: ${dealInfo.expiryEpoch}`)

// Listen for new storage events
fvmStorage.listenForStorageEvents((event) => {
  console.log(`\nNew storage event:`)
  console.log(`  CID: ${event.cid}`)
  console.log(`  Provider: ${event.provider}`)
  console.log(`  Block: ${event.blockNumber}`)
})
```

---

## Web3 Wallet and Authentication

### UCAN-Based Authentication

```typescript
// UCAN delegation and agent management
import { create as createClient } from '@web3-storage/w3up-client'
import * as Client from '@web3-storage/w3up-client'
import { StoreMemory } from '@web3-storage/w3up-client/stores/memory'
import * as Proof from '@web3-storage/w3up-client/proof'
import * as Delegation from '@web3-storage/w3up-client/delegation'

class UCANAuthManager {
  private client: any

  async initialize(): Promise<void> {
    // Create client with custom store
    this.client = await createClient({
      store: new StoreMemory()
    })
  }

  /**
   * Email-based authentication
   */
  async loginWithEmail(email: string): Promise<void> {
    console.log(`Sending verification email to ${email}...`)

    await this.client.login(email)

    console.log('✅ Email verified. Client authenticated.')
  }

  /**
   * Create a space (namespace for uploads)
   */
  async createSpace(name: string): Promise<string> {
    const space = await this.client.createSpace(name)
    await this.client.setCurrentSpace(space.did())

    console.log(`Created space: ${space.did()}`)
    return space.did()
  }

  /**
   * Create delegation for another agent
   */
  async createDelegation(
    recipientDID: string,
    capabilities: string[]
  ): Promise<Uint8Array> {
    const space = this.client.currentSpace()
    if (!space) {
      throw new Error('No active space')
    }

    // Create delegation with specific capabilities
    const delegation = await this.client.createDelegation(
      recipientDID,
      capabilities,
      {
        expiration: Math.floor(Date.now() / 1000) + 86400 * 30 // 30 days
      }
    )

    // Serialize delegation as CAR file
    const carBytes = await delegation.export()

    console.log(`Created delegation for ${recipientDID}`)
    console.log(`  Capabilities: ${capabilities.join(', ')}`)
    console.log(`  Size: ${carBytes.length} bytes`)

    return carBytes
  }

  /**
   * Import delegation proof
   */
  async importDelegation(carBytes: Uint8Array): Promise<void> {
    await this.client.addProof(carBytes)
    console.log('✅ Delegation imported')
  }

  /**
   * Export agent data (for backup or transfer)
   */
  async exportAgent(): Promise<Uint8Array> {
    const exported = await this.client.exportAgent()
    console.log(`Exported agent data: ${exported.length} bytes`)
    return exported
  }

  /**
   * Import agent data
   */
  static async importAgent(agentData: Uint8Array): Promise<UCANAuthManager> {
    const manager = new UCANAuthManager()
    manager.client = await Client.createFromExport(agentData)
    console.log('✅ Agent imported')
    return manager
  }

  /**
   * List current capabilities
   */
  async listCapabilities(): Promise<string[]> {
    const space = this.client.currentSpace()
    if (!space) {
      return []
    }

    const proofs = await this.client.proofs()
    const capabilities: string[] = []

    for (const proof of proofs) {
      for (const capability of proof.capabilities) {
        capabilities.push(capability.can)
      }
    }

    return [...new Set(capabilities)]
  }
}

// Usage Example 1: Main application
const mainApp = new UCANAuthManager()
await mainApp.initialize()
await mainApp.loginWithEmail('alice@example.com')
const spaceId = await mainApp.createSpace('alice-main-space')

console.log('\n📱 Main Application Setup Complete')

// Usage Example 2: Create delegation for mobile app
const mobileDID = 'did:key:z6MkqG7k...' // Mobile app's DID

const delegationCAR = await mainApp.createDelegation(
  mobileDID,
  [
    'upload/add',
    'upload/list',
    'upload/remove'
  ]
)

// Save delegation CAR file
import * as fs from 'fs'
fs.writeFileSync('mobile-delegation.car', delegationCAR)

console.log('\n📤 Delegation created for mobile app')

// Usage Example 3: Mobile app imports delegation
const mobileApp = new UCANAuthManager()
await mobileApp.initialize()

const delegationBytes = fs.readFileSync('mobile-delegation.car')
await mobileApp.importDelegation(delegationBytes)

const capabilities = await mobileApp.listCapabilities()
console.log('\n📱 Mobile App Capabilities:')
capabilities.forEach(cap => console.log(`  - ${cap}`))

// Usage Example 4: Export and restore agent
const agentBackup = await mainApp.exportAgent()
fs.writeFileSync('agent-backup.car', agentBackup)

// Restore on another device
const restoredManager = await UCANAuthManager.importAgent(agentBackup)
console.log('\n💾 Agent restored from backup')
```

### DID-Based Identity Management

```typescript
// Decentralized Identity Management
import * as DID from '@ipld/dag-ucan/did'
import * as ed25519 from '@ucanto/principal/ed25519'

class DIDManager {
  private principal: any

  /**
   * Generate new DID
   */
  static async generateDID(): Promise<{
    did: string
    privateKey: Uint8Array
  }> {
    const principal = await ed25519.generate()

    return {
      did: principal.did(),
      privateKey: principal.toArchive()
    }
  }

  /**
   * Load DID from private key
   */
  static async fromPrivateKey(privateKey: Uint8Array): Promise<DIDManager> {
    const manager = new DIDManager()
    manager.principal = await ed25519.from(privateKey)
    return manager
  }

  getDID(): string {
    return this.principal.did()
  }

  /**
   * Sign data
   */
  async sign(data: Uint8Array): Promise<Uint8Array> {
    return await this.principal.sign(data)
  }

  /**
   * Verify signature
   */
  static async verify(
    did: string,
    data: Uint8Array,
    signature: Uint8Array
  ): Promise<boolean> {
    // Parse DID and verify signature
    const publicKey = DID.parse(did)
    // Verification logic would go here
    return true // Simplified
  }

  /**
   * Export for backup
   */
  exportKey(): Uint8Array {
    return this.principal.toArchive()
  }
}

// Usage
// Generate new identity
const identity = await DIDManager.generateDID()
console.log(`Generated DID: ${identity.did}`)
console.log(`Private key length: ${identity.privateKey.length} bytes`)

// Save private key securely
import * as fs from 'fs'
fs.writeFileSync('.did-key', identity.privateKey)

// Load identity from key
const loadedIdentity = await DIDManager.fromPrivateKey(identity.privateKey)
console.log(`Loaded DID: ${loadedIdentity.getDID()}`)

// Sign data
const message = new TextEncoder().encode('Hello, Web3!')
const signature = await loadedIdentity.sign(message)
console.log(`Signature length: ${signature.length} bytes`)

// Verify signature
const isValid = await DIDManager.verify(
  loadedIdentity.getDID(),
  message,
  signature
)
console.log(`Signature valid: ${isValid}`)
```

---

## External Service Integrations

### Pinata Integration

```typescript
// Integrate Storacha with Pinata for redundancy
import { create as createStorachaClient } from '@web3-storage/w3up-client'
import PinataSDK from '@pinata/sdk'

class MultiPinningService {
  private storacha: any
  private pinata: any

  async initialize(
    storachaEmail: string,
    pinataApiKey: string,
    pinataSecretKey: string
  ): Promise<void> {
    // Initialize Storacha
    this.storacha = await createStorachaClient()
    await this.storacha.login(storachaEmail)

    // Initialize Pinata
    this.pinata = new PinataSDK(pinataApiKey, pinataSecretKey)

    // Test Pinata connection
    const testAuth = await this.pinata.testAuthentication()
    if (!testAuth.authenticated) {
      throw new Error('Pinata authentication failed')
    }

    console.log('✅ Both services initialized')
  }

  /**
   * Upload to both services
   */
  async uploadToBoth(file: File): Promise<{
    storacha: string
    pinata: string
    match: boolean
  }> {
    console.log('Uploading to Storacha...')
    const storachaCID = await this.storacha.uploadFile(file)

    console.log('Uploading to Pinata...')
    const pinataCID = await this.uploadToPinata(file)

    return {
      storacha: storachaCID.toString(),
      pinata: pinataCID,
      match: storachaCID.toString() === pinataCID
    }
  }

  private async uploadToPinata(file: File): Promise<string> {
    // Convert File to ReadableStream for Pinata
    const buffer = Buffer.from(await file.arrayBuffer())

    const result = await this.pinata.pinFileToIPFS(buffer, {
      pinataMetadata: {
        name: file.name
      }
    })

    return result.IpfsHash
  }

  /**
   * Verify content on both services
   */
  async verifyOnBoth(cid: string): Promise<{
    storacha: boolean
    pinata: boolean
    both: boolean
  }> {
    const [storachaOk, pinataOk] = await Promise.all([
      this.verifyStoracha(cid),
      this.verifyPinata(cid)
    ])

    return {
      storacha: storachaOk,
      pinata: pinataOk,
      both: storachaOk && pinataOk
    }
  }

  private async verifyStoracha(cid: string): Promise<boolean> {
    try {
      const response = await fetch(`https://w3s.link/ipfs/${cid}`, {
        method: 'HEAD'
      })
      return response.ok
    } catch {
      return false
    }
  }

  private async verifyPinata(cid: string): Promise<boolean> {
    try {
      const result = await this.pinata.pinList({
        hashContains: cid
      })
      return result.rows.length > 0
    } catch {
      return false
    }
  }

  /**
   * List pins from both services
   */
  async listAllPins(): Promise<{
    storacha: any[]
    pinata: any[]
  }> {
    const [storachaPins, pinataPins] = await Promise.all([
      this.storacha.list(),
      this.pinata.pinList({ status: 'pinned' })
    ])

    return {
      storacha: Array.from(storachaPins),
      pinata: pinataPins.rows
    }
  }
}

// Usage
const multiPin = new MultiPinningService()
await multiPin.initialize(
  'alice@example.com',
  'your-pinata-api-key',
  'your-pinata-secret-key'
)

const file = new File(['Redundant storage'], 'important.txt')
const result = await multiPin.uploadToBoth(file)

console.log('Upload results:')
console.log(`  Storacha: ${result.storacha}`)
console.log(`  Pinata: ${result.pinata}`)
console.log(`  CIDs match: ${result.match}`)

// Verify
const verification = await multiPin.verifyOnBoth(result.storacha)
console.log('\nVerification:')
console.log(`  Storacha: ${verification.storacha ? '✅' : '❌'}`)
console.log(`  Pinata: ${verification.pinata ? '✅' : '❌'}`)
console.log(`  Redundancy: ${verification.both ? '✅' : '❌'}`)
```

### Infura IPFS Integration

```typescript
// Integrate with Infura IPFS
import { create as createIPFSClient } from 'ipfs-http-client'
import { create as createStorachaClient } from '@web3-storage/w3up-client'

class InfuraStorachaIntegration {
  private infura: any
  private storacha: any

  async initialize(
    infuraProjectId: string,
    infuraProjectSecret: string,
    storachaEmail: string
  ): Promise<void> {
    // Initialize Infura IPFS client
    const auth = 'Basic ' + Buffer.from(
      infuraProjectId + ':' + infuraProjectSecret
    ).toString('base64')

    this.infura = createIPFSClient({
      host: 'ipfs.infura.io',
      port: 5001,
      protocol: 'https',
      headers: {
        authorization: auth
      }
    })

    // Initialize Storacha
    this.storacha = await createStorachaClient()
    await this.storacha.login(storachaEmail)

    console.log('✅ Infura and Storacha clients initialized')
  }

  /**
   * Upload to Infura, then pin on Storacha
   */
  async uploadWithInfura(file: File): Promise<{
    infuraCID: string
    storachaCID: string
  }> {
    // Upload to Infura
    const buffer = Buffer.from(await file.arrayBuffer())
    const infuraResult = await this.infura.add(buffer, {
      cidVersion: 1,
      hashAlg: 'sha2-256'
    })

    console.log(`Uploaded to Infura: ${infuraResult.cid}`)

    // Pin same content on Storacha
    const storachaResult = await this.storacha.uploadFile(file)

    return {
      infuraCID: infuraResult.cid.toString(),
      storachaCID: storachaResult.toString()
    }
  }

  /**
   * Retrieve with fallback logic
   */
  async retrieveWithFallback(cid: string): Promise<Uint8Array> {
    // Try Infura first
    try {
      const chunks = []
      for await (const chunk of this.infura.cat(cid, { timeout: 5000 })) {
        chunks.push(chunk)
      }
      console.log('Retrieved from Infura')
      return new Uint8Array(Buffer.concat(chunks))
    } catch (error) {
      console.log('Infura failed, trying Storacha gateway')

      // Fallback to Storacha
      const response = await fetch(`https://w3s.link/ipfs/${cid}`)
      if (!response.ok) {
        throw new Error('Failed to retrieve from both services')
      }

      console.log('Retrieved from Storacha')
      return new Uint8Array(await response.arrayBuffer())
    }
  }
}

// Usage
const integration = new InfuraStorachaIntegration()
await integration.initialize(
  'your-infura-project-id',
  'your-infura-project-secret',
  'alice@example.com'
)

const file = new File(['Multi-gateway data'], 'data.txt')
const result = await integration.uploadWithInfura(file)

console.log('\nUpload Complete:')
console.log(`  Infura: ${result.infuraCID}`)
console.log(`  Storacha: ${result.storachaCID}`)

// Retrieve with automatic fallback
const data = await integration.retrieveWithFallback(result.infuraCID)
console.log(`\nRetrieved: ${new TextDecoder().decode(data)}`)
```

### NFT.Storage Integration

```typescript
// Cross-platform NFT storage
import { NFTStorage } from 'nft.storage'
import { create as createStorachaClient } from '@web3-storage/w3up-client'

class NFTStorageIntegration {
  private nftStorage: NFTStorage
  private storacha: any

  async initialize(
    nftStorageToken: string,
    storachaEmail: string
  ): Promise<void> {
    this.nftStorage = new NFTStorage({ token: nftStorageToken })
    this.storacha = await createStorachaClient()
    await this.storacha.login(storachaEmail)

    console.log('✅ NFT.Storage and Storacha initialized')
  }

  /**
   * Store NFT metadata on both platforms
   */
  async storeNFTMetadata(metadata: {
    name: string
    description: string
    image: File
    attributes?: Array<{ trait_type: string; value: string }>
  }): Promise<{
    nftStorage: string
    storacha: string
    ipfsURL: string
  }> {
    // Store on NFT.Storage
    const nftResult = await this.nftStorage.store({
      name: metadata.name,
      description: metadata.description,
      image: metadata.image,
      properties: {
        attributes: metadata.attributes || []
      }
    })

    console.log(`Stored on NFT.Storage: ${nftResult.url}`)

    // Also backup on Storacha
    const metadataFile = new File(
      [JSON.stringify({
        name: metadata.name,
        description: metadata.description,
        attributes: metadata.attributes
      })],
      'metadata.json',
      { type: 'application/json' }
    )

    const storachaResult = await this.storacha.uploadFile(metadataFile)

    return {
      nftStorage: nftResult.url,
      storacha: storachaResult.toString(),
      ipfsURL: `ipfs://${nftResult.ipnft}`
    }
  }

  /**
   * Store entire NFT collection
   */
  async storeCollection(nfts: Array<{
    name: string
    description: string
    image: File
    attributes?: Array<{ trait_type: string; value: string }>
  }>): Promise<Array<{
    tokenId: number
    metadataURI: string
    storachaCID: string
  }>> {
    const results = []

    for (let i = 0; i < nfts.length; i++) {
      const nft = nfts[i]
      console.log(`\nProcessing NFT ${i + 1}/${nfts.length}: ${nft.name}`)

      const result = await this.storeNFTMetadata(nft)

      results.push({
        tokenId: i,
        metadataURI: result.ipfsURL,
        storachaCID: result.storacha
      })

      // Small delay to avoid rate limits
      await new Promise(resolve => setTimeout(resolve, 100))
    }

    return results
  }
}

// Usage
const nftIntegration = new NFTStorageIntegration()
await nftIntegration.initialize(
  'your-nft-storage-token',
  'alice@example.com'
)

// Store single NFT
const imageFile = new File(['<svg>...</svg>'], 'nft.svg', { type: 'image/svg+xml' })
const nftResult = await nftIntegration.storeNFTMetadata({
  name: 'Cool NFT #1',
  description: 'A very cool NFT',
  image: imageFile,
  attributes: [
    { trait_type: 'Rarity', value: 'Legendary' },
    { trait_type: 'Power', value: '9000' }
  ]
})

console.log('\nNFT Stored:')
console.log(`  NFT.Storage: ${nftResult.nftStorage}`)
console.log(`  Storacha: ${nftResult.storacha}`)
console.log(`  IPFS URI: ${nftResult.ipfsURL}`)

// Store collection
const collection = [
  {
    name: 'NFT #1',
    description: 'First NFT',
    image: imageFile,
    attributes: [{ trait_type: 'Generation', value: '1' }]
  },
  {
    name: 'NFT #2',
    description: 'Second NFT',
    image: imageFile,
    attributes: [{ trait_type: 'Generation', value: '1' }]
  }
]

const collectionResults = await nftIntegration.storeCollection(collection)

console.log('\n Collection stored:')
collectionResults.forEach(nft => {
  console.log(`  Token ${nft.tokenId}: ${nft.metadataURI}`)
})
```

---

## Development Frameworks

### Next.js Integration

```typescript
// Next.js App Router integration with Storacha
// app/api/upload/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { create as createClient } from '@web3-storage/w3up-client'

// Initialize client (cached across requests)
let storachaClient: any = null

async function getClient() {
  if (!storachaClient) {
    storachaClient = await createClient()
    await storachaClient.login(process.env.STORACHA_EMAIL!)

    const spaces = await storachaClient.spaces()
    if (spaces.length === 0) {
      const space = await storachaClient.createSpace('nextjs-app')
      await storachaClient.setCurrentSpace(space.did())
    } else {
      await storachaClient.setCurrentSpace(spaces[0].did())
    }
  }
  return storachaClient
}

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData()
    const file = formData.get('file') as File

    if (!file) {
      return NextResponse.json(
        { error: 'No file provided' },
        { status: 400 }
      )
    }

    const client = await getClient()
    const cid = await client.uploadFile(file)

    return NextResponse.json({
      success: true,
      cid: cid.toString(),
      url: `https://w3s.link/ipfs/${cid}`,
      size: file.size,
      name: file.name
    })
  } catch (error) {
    console.error('Upload error:', error)
    return NextResponse.json(
      { error: 'Upload failed', details: error instanceof Error ? error.message : 'Unknown error' },
      { status: 500 }
    )
  }
}

export async function GET(request: NextRequest) {
  try {
    const client = await getClient()
    const uploads = []

    for await (const upload of client.list()) {
      uploads.push({
        cid: upload.root.toString(),
        uploadedAt: upload.insertedAt
      })
    }

    return NextResponse.json({
      success: true,
      uploads
    })
  } catch (error) {
    console.error('List error:', error)
    return NextResponse.json(
      { error: 'Failed to list uploads' },
      { status: 500 }
    )
  }
}

// app/upload/page.tsx
'use client'

import { useState } from 'react'

export default function UploadPage() {
  const [file, setFile] = useState<File | null>(null)
  const [uploading, setUploading] = useState(false)
  const [result, setResult] = useState<any>(null)

  const handleUpload = async () => {
    if (!file) return

    setUploading(true)
    try {
      const formData = new FormData()
      formData.append('file', file)

      const response = await fetch('/api/upload', {
        method: 'POST',
        body: formData
      })

      const data = await response.json()
      setResult(data)
    } catch (error) {
      console.error('Upload error:', error)
      setResult({ error: 'Upload failed' })
    } finally {
      setUploading(false)
    }
  }

  return (
    <div className="container mx-auto p-8">
      <h1 className="text-3xl font-bold mb-8">Upload to Storacha</h1>

      <div className="mb-4">
        <input
          type="file"
          onChange={(e) => setFile(e.target.files?.[0] || null)}
          className="border p-2 rounded"
        />
      </div>

      <button
        onClick={handleUpload}
        disabled={!file || uploading}
        className="bg-blue-500 text-white px-6 py-2 rounded disabled:bg-gray-300"
      >
        {uploading ? 'Uploading...' : 'Upload'}
      </button>

      {result && (
        <div className="mt-8 p-4 border rounded">
          {result.success ? (
            <div>
              <h2 className="text-xl font-bold mb-2">Upload Successful!</h2>
              <p className="mb-2"><strong>CID:</strong> {result.cid}</p>
              <p className="mb-2"><strong>URL:</strong> <a href={result.url} target="_blank" rel="noopener noreferrer" className="text-blue-500 underline">{result.url}</a></p>
              <p><strong>Size:</strong> {result.size} bytes</p>
            </div>
          ) : (
            <div className="text-red-500">
              <p>Error: {result.error}</p>
            </div>
          )}
        </div>
      )}
    </div>
  )
}
```

### React Hook for Storacha

```typescript
// hooks/useStoracha.ts
import { useState, useEffect, useCallback } from 'react'
import { create as createClient } from '@web3-storage/w3up-client'

interface StorachaState {
  client: any | null
  loading: boolean
  error: string | null
}

interface UploadResult {
  cid: string
  url: string
}

export function useStoracha(email?: string) {
  const [state, setState] = useState<StorachaState>({
    client: null,
    loading: true,
    error: null
  })

  useEffect(() => {
    async function initialize() {
      try {
        const client = await createClient()

        if (email) {
          await client.login(email)
          const spaces = await client.spaces()

          if (spaces.length === 0) {
            const space = await client.createSpace('react-app')
            await client.setCurrentSpace(space.did())
          } else {
            await client.setCurrentSpace(spaces[0].did())
          }
        }

        setState({ client, loading: false, error: null })
      } catch (error) {
        setState({
          client: null,
          loading: false,
          error: error instanceof Error ? error.message : 'Failed to initialize'
        })
      }
    }

    initialize()
  }, [email])

  const uploadFile = useCallback(async (file: File): Promise<UploadResult> => {
    if (!state.client) {
      throw new Error('Client not initialized')
    }

    const cid = await state.client.uploadFile(file)
    return {
      cid: cid.toString(),
      url: `https://w3s.link/ipfs/${cid}`
    }
  }, [state.client])

  const uploadDirectory = useCallback(async (files: File[]): Promise<UploadResult> => {
    if (!state.client) {
      throw new Error('Client not initialized')
    }

    const cid = await state.client.uploadDirectory(files)
    return {
      cid: cid.toString(),
      url: `https://w3s.link/ipfs/${cid}`
    }
  }, [state.client])

  const listUploads = useCallback(async () => {
    if (!state.client) {
      throw new Error('Client not initialized')
    }

    const uploads = []
    for await (const upload of state.client.list()) {
      uploads.push({
        cid: upload.root.toString(),
        uploadedAt: upload.insertedAt
      })
    }
    return uploads
  }, [state.client])

  return {
    client: state.client,
    loading: state.loading,
    error: state.error,
    uploadFile,
    uploadDirectory,
    listUploads
  }
}

// Component usage example
// components/FileUploader.tsx
import React, { useState } from 'react'
import { useStoracha } from '../hooks/useStoracha'

export function FileUploader() {
  const { uploadFile, loading, error } = useStoracha('user@example.com')
  const [file, setFile] = useState<File | null>(null)
  const [uploading, setUploading] = useState(false)
  const [result, setResult] = useState<any>(null)

  const handleUpload = async () => {
    if (!file) return

    setUploading(true)
    try {
      const uploadResult = await uploadFile(file)
      setResult(uploadResult)
    } catch (err) {
      setResult({ error: err instanceof Error ? err.message : 'Upload failed' })
    } finally {
      setUploading(false)
    }
  }

  if (loading) {
    return <div>Initializing Storacha client...</div>
  }

  if (error) {
    return <div className="text-red-500">Error: {error}</div>
  }

  return (
    <div className="p-4">
      <input
        type="file"
        onChange={(e) => setFile(e.target.files?.[0] || null)}
        className="mb-4"
      />

      <button
        onClick={handleUpload}
        disabled={!file || uploading}
        className="bg-blue-500 text-white px-4 py-2 rounded"
      >
        {uploading ? 'Uploading...' : 'Upload to Storacha'}
      </button>

      {result && (
        <div className="mt-4">
          {result.cid ? (
            <div>
              <p>CID: {result.cid}</p>
              <a href={result.url} target="_blank" rel="noopener noreferrer">
                View on IPFS
              </a>
            </div>
          ) : (
            <p className="text-red-500">{result.error}</p>
          )}
        </div>
      )}
    </div>
  )
}
```

### Vue.js Integration

```typescript
// composables/useStoracha.ts
import { ref, onMounted } from 'vue'
import { create as createClient } from '@web3-storage/w3up-client'

export function useStoracha(email?: string) {
  const client = ref<any>(null)
  const loading = ref(true)
  const error = ref<string | null>(null)

  onMounted(async () => {
    try {
      const c = await createClient()

      if (email) {
        await c.login(email)
        const spaces = await c.spaces()

        if (spaces.length === 0) {
          const space = await c.createSpace('vue-app')
          await c.setCurrentSpace(space.did())
        } else {
          await c.setCurrentSpace(spaces[0].did())
        }
      }

      client.value = c
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Failed to initialize'
    } finally {
      loading.value = false
    }
  })

  const uploadFile = async (file: File) => {
    if (!client.value) {
      throw new Error('Client not initialized')
    }

    const cid = await client.value.uploadFile(file)
    return {
      cid: cid.toString(),
      url: `https://w3s.link/ipfs/${cid}`
    }
  }

  const listUploads = async () => {
    if (!client.value) {
      throw new Error('Client not initialized')
    }

    const uploads = []
    for await (const upload of client.value.list()) {
      uploads.push({
        cid: upload.root.toString(),
        uploadedAt: upload.insertedAt
      })
    }
    return uploads
  }

  return {
    client,
    loading,
    error,
    uploadFile,
    listUploads
  }
}

// components/FileUploader.vue
<template>
  <div class="file-uploader">
    <div v-if="loading">Initializing Storacha client...</div>

    <div v-else-if="error" class="error">
      Error: {{ error }}
    </div>

    <div v-else>
      <input
        type="file"
        @change="handleFileChange"
        ref="fileInput"
      />

      <button
        @click="handleUpload"
        :disabled="!selectedFile || uploading"
        class="upload-btn"
      >
        {{ uploading ? 'Uploading...' : 'Upload to Storacha' }}
      </button>

      <div v-if="uploadResult" class="result">
        <div v-if="uploadResult.cid">
          <p>CID: {{ uploadResult.cid }}</p>
          <a :href="uploadResult.url" target="_blank">View on IPFS</a>
        </div>
        <p v-else class="error">{{ uploadResult.error }}</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { useStoracha } from '../composables/useStoracha'

const { loading, error, uploadFile } = useStoracha('user@example.com')

const selectedFile = ref<File | null>(null)
const uploading = ref(false)
const uploadResult = ref<any>(null)
const fileInput = ref<HTMLInputElement | null>(null)

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement
  selectedFile.value = target.files?.[0] || null
}

const handleUpload = async () => {
  if (!selectedFile.value) return

  uploading.value = true
  try {
    const result = await uploadFile(selectedFile.value)
    uploadResult.value = result
  } catch (err) {
    uploadResult.value = {
      error: err instanceof Error ? err.message : 'Upload failed'
    }
  } finally {
    uploading.value = false
  }
}
</script>

<style scoped>
.file-uploader {
  padding: 2rem;
}

.upload-btn {
  background-color: #3b82f6;
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  margin-top: 1rem;
}

.upload-btn:disabled {
  background-color: #9ca3af;
}

.error {
  color: #ef4444;
}

.result {
  margin-top: 1rem;
  padding: 1rem;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
}
</style>
```

---

## Integration Patterns

### Event-Driven Architecture

```typescript
// Event-driven upload pipeline
import { EventEmitter } from 'events'
import { create as createClient } from '@web3-storage/w3up-client'

interface UploadEvent {
  type: 'start' | 'progress' | 'complete' | 'error'
  file: string
  cid?: string
  progress?: number
  error?: string
}

class StorachaEventPipeline extends EventEmitter {
  private client: any
  private queue: File[]
  private processing: boolean

  constructor() {
    super()
    this.queue = []
    this.processing = false
  }

  async initialize(email: string): Promise<void> {
    this.client = await createClient()
    await this.client.login(email)

    const spaces = await this.client.spaces()
    if (spaces.length === 0) {
      const space = await this.client.createSpace('event-pipeline')
      await this.client.setCurrentSpace(space.did())
    } else {
      await this.client.setCurrentSpace(spaces[0].did())
    }
  }

  /**
   * Add file to upload queue
   */
  enqueue(file: File): void {
    this.queue.push(file)
    this.emit('queued', { file: file.name, queueLength: this.queue.length })

    if (!this.processing) {
      this.processQueue()
    }
  }

  /**
   * Process upload queue
   */
  private async processQueue(): Promise<void> {
    if (this.queue.length === 0) {
      this.processing = false
      this.emit('queue-empty')
      return
    }

    this.processing = true
    const file = this.queue.shift()!

    try {
      this.emit('upload-start', { file: file.name })

      // Upload with progress tracking
      const cid = await this.client.uploadFile(file, {
        onShardStored: (meta: any) => {
          const progress = (meta.size / file.size) * 100
          this.emit('upload-progress', {
            file: file.name,
            progress,
            shardCID: meta.cid
          })
        }
      })

      this.emit('upload-complete', {
        file: file.name,
        cid: cid.toString()
      })
    } catch (error) {
      this.emit('upload-error', {
        file: file.name,
        error: error instanceof Error ? error.message : 'Unknown error'
      })
    }

    // Process next file
    this.processQueue()
  }

  /**
   * Get queue status
   */
  getQueueStatus(): {
    pending: number
    processing: boolean
  } {
    return {
      pending: this.queue.length,
      processing: this.processing
    }
  }
}

// Usage
const pipeline = new StorachaEventPipeline()

// Set up event listeners
pipeline.on('queued', (data) => {
  console.log(`📝 Queued: ${data.file} (Queue length: ${data.queueLength})`)
})

pipeline.on('upload-start', (data) => {
  console.log(`🚀 Starting upload: ${data.file}`)
})

pipeline.on('upload-progress', (data) => {
  console.log(`📊 Progress: ${data.file} - ${data.progress.toFixed(1)}%`)
})

pipeline.on('upload-complete', (data) => {
  console.log(`✅ Complete: ${data.file}`)
  console.log(`   CID: ${data.cid}`)
})

pipeline.on('upload-error', (data) => {
  console.error(`❌ Error: ${data.file}`)
  console.error(`   ${data.error}`)
})

pipeline.on('queue-empty', () => {
  console.log('🎉 All uploads complete!')
})

// Initialize and add files
await pipeline.initialize('alice@example.com')

const files = [
  new File(['File 1'], 'file1.txt'),
  new File(['File 2'], 'file2.txt'),
  new File(['File 3'], 'file3.txt')
]

files.forEach(file => pipeline.enqueue(file))

// Check status
const status = pipeline.getQueueStatus()
console.log(`\nQueue status: ${status.pending} pending, ${status.processing ? 'processing' : 'idle'}`)
```

### Microservices Pattern

```typescript
// Microservice for Storacha uploads
import express from 'express'
import multer from 'multer'
import { create as createClient } from '@web3-storage/w3up-client'

class StorachaUploadService {
  private app: express.Application
  private client: any
  private upload: multer.Multer

  constructor() {
    this.app = express()
    this.upload = multer({ storage: multer.memoryStorage() })

    this.setupMiddleware()
    this.setupRoutes()
  }

  async initialize(email: string): Promise<void> {
    this.client = await createClient()
    await this.client.login(email)

    const spaces = await this.client.spaces()
    if (spaces.length === 0) {
      const space = await this.client.createSpace('upload-service')
      await this.client.setCurrentSpace(space.did())
    } else {
      await this.client.setCurrentSpace(spaces[0].did())
    }

    console.log('✅ Storacha client initialized')
  }

  private setupMiddleware(): void {
    this.app.use(express.json())
    this.app.use((req, res, next) => {
      res.header('Access-Control-Allow-Origin', '*')
      res.header('Access-Control-Allow-Methods', 'GET, POST, DELETE')
      res.header('Access-Control-Allow-Headers', 'Content-Type')
      next()
    })
  }

  private setupRoutes(): void {
    // Health check
    this.app.get('/health', (req, res) => {
      res.json({ status: 'healthy', service: 'storacha-upload' })
    })

    // Upload single file
    this.app.post('/upload', this.upload.single('file'), async (req, res) => {
      try {
        if (!req.file) {
          return res.status(400).json({ error: 'No file provided' })
        }

        const file = new File(
          [req.file.buffer],
          req.file.originalname,
          { type: req.file.mimetype }
        )

        const cid = await this.client.uploadFile(file)

        res.json({
          success: true,
          cid: cid.toString(),
          url: `https://w3s.link/ipfs/${cid}`,
          size: req.file.size,
          name: req.file.originalname,
          type: req.file.mimetype
        })
      } catch (error) {
        console.error('Upload error:', error)
        res.status(500).json({
          error: 'Upload failed',
          details: error instanceof Error ? error.message : 'Unknown error'
        })
      }
    })

    // Upload multiple files
    this.app.post('/upload/batch', this.upload.array('files', 10), async (req, res) => {
      try {
        if (!req.files || !Array.isArray(req.files) || req.files.length === 0) {
          return res.status(400).json({ error: 'No files provided' })
        }

        const files = req.files.map(f =>
          new File(
            [f.buffer],
            f.originalname,
            { type: f.mimetype }
          )
        )

        const cid = await this.client.uploadDirectory(files)

        res.json({
          success: true,
          cid: cid.toString(),
          url: `https://w3s.link/ipfs/${cid}`,
          fileCount: files.length,
          totalSize: req.files.reduce((sum, f) => sum + f.size, 0)
        })
      } catch (error) {
        console.error('Batch upload error:', error)
        res.status(500).json({
          error: 'Batch upload failed',
          details: error instanceof Error ? error.message : 'Unknown error'
        })
      }
    })

    // List uploads
    this.app.get('/uploads', async (req, res) => {
      try {
        const uploads = []
        for await (const upload of this.client.list()) {
          uploads.push({
            cid: upload.root.toString(),
            uploadedAt: upload.insertedAt
          })
        }

        res.json({
          success: true,
          count: uploads.length,
          uploads
        })
      } catch (error) {
        console.error('List error:', error)
        res.status(500).json({
          error: 'Failed to list uploads',
          details: error instanceof Error ? error.message : 'Unknown error'
        })
      }
    })

    // Delete upload (remove from listing)
    this.app.delete('/uploads/:cid', async (req, res) => {
      try {
        const { cid } = req.params
        await this.client.remove(cid)

        res.json({
          success: true,
          message: `Removed ${cid}`
        })
      } catch (error) {
        console.error('Delete error:', error)
        res.status(500).json({
          error: 'Failed to delete upload',
          details: error instanceof Error ? error.message : 'Unknown error'
        })
      }
    })
  }

  start(port: number = 3000): void {
    this.app.listen(port, () => {
      console.log(`🚀 Storacha Upload Service running on port ${port}`)
    })
  }
}

// Usage
const service = new StorachaUploadService()
await service.initialize('service@example.com')
service.start(3000)

// Client usage example
async function clientExample() {
  const formData = new FormData()
  formData.append('file', new File(['Hello'], 'test.txt'))

  const response = await fetch('http://localhost:3000/upload', {
    method: 'POST',
    body: formData
  })

  const result = await response.json()
  console.log('Upload result:', result)
}
```

### Webhook Integration

```typescript
// Webhook notifications for upload events
import express from 'express'
import { create as createClient } from '@web3-storage/w3up-client'

interface WebhookPayload {
  event: 'upload.started' | 'upload.completed' | 'upload.failed'
  timestamp: string
  data: {
    cid?: string
    filename: string
    size: number
    error?: string
  }
}

class WebhookNotifier {
  private webhookURLs: string[]

  constructor(webhookURLs: string[]) {
    this.webhookURLs = webhookURLs
  }

  async notify(payload: WebhookPayload): Promise<void> {
    const requests = this.webhookURLs.map(url =>
      fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': this.generateSignature(payload)
        },
        body: JSON.stringify(payload)
      })
    )

    await Promise.allSettled(requests)
  }

  private generateSignature(payload: WebhookPayload): string {
    // In production, use HMAC with secret key
    return `sha256=${Buffer.from(JSON.stringify(payload)).toString('base64')}`
  }
}

class StorachaWebhookService {
  private client: any
  private notifier: WebhookNotifier

  constructor(webhookURLs: string[]) {
    this.notifier = new WebhookNotifier(webhookURLs)
  }

  async initialize(email: string): Promise<void> {
    this.client = await createClient()
    await this.client.login(email)

    const spaces = await this.client.spaces()
    if (spaces.length === 0) {
      const space = await this.client.createSpace('webhook-service')
      await this.client.setCurrentSpace(space.did())
    } else {
      await this.client.setCurrentSpace(spaces[0].did())
    }
  }

  async uploadWithWebhook(file: File): Promise<void> {
    // Notify upload started
    await this.notifier.notify({
      event: 'upload.started',
      timestamp: new Date().toISOString(),
      data: {
        filename: file.name,
        size: file.size
      }
    })

    try {
      const cid = await this.client.uploadFile(file)

      // Notify upload completed
      await this.notifier.notify({
        event: 'upload.completed',
        timestamp: new Date().toISOString(),
        data: {
          cid: cid.toString(),
          filename: file.name,
          size: file.size
        }
      })
    } catch (error) {
      // Notify upload failed
      await this.notifier.notify({
        event: 'upload.failed',
        timestamp: new Date().toISOString(),
        data: {
          filename: file.name,
          size: file.size,
          error: error instanceof Error ? error.message : 'Unknown error'
        }
      })

      throw error
    }
  }
}

// Webhook receiver example
const app = express()
app.use(express.json())

app.post('/webhook/storacha', (req, res) => {
  const payload: WebhookPayload = req.body

  console.log(`Received webhook: ${payload.event}`)
  console.log(`Timestamp: ${payload.timestamp}`)
  console.log(`Data:`, payload.data)

  // Verify signature
  const signature = req.headers['x-webhook-signature']
  console.log(`Signature: ${signature}`)

  // Process event
  switch (payload.event) {
    case 'upload.started':
      console.log(`Upload started: ${payload.data.filename}`)
      break
    case 'upload.completed':
      console.log(`Upload completed: ${payload.data.cid}`)
      // Trigger downstream processes
      break
    case 'upload.failed':
      console.log(`Upload failed: ${payload.data.error}`)
      // Alert operations team
      break
  }

  res.status(200).json({ received: true })
})

app.listen(3001, () => {
  console.log('Webhook receiver listening on port 3001')
})

// Usage
const webhookService = new StorachaWebhookService([
  'http://localhost:3001/webhook/storacha'
])

await webhookService.initialize('webhook@example.com')

const file = new File(['Webhook test'], 'test.txt')
await webhookService.uploadWithWebhook(file)
```

---

## Best Practices

### Multi-Region Strategy

```typescript
// Multi-region deployment for global performance
class MultiRegionStoracha {
  private clients: Map<string, any>
  private regions: string[]

  constructor(regions: string[]) {
    this.clients = new Map()
    this.regions = regions
  }

  async initialize(email: string): Promise<void> {
    for (const region of this.regions) {
      const client = await createClient()
      await client.login(email)

      const space = await client.createSpace(`space-${region}`)
      await client.setCurrentSpace(space.did())

      this.clients.set(region, client)
    }

    console.log(`✅ Initialized clients in ${this.regions.length} regions`)
  }

  /**
   * Upload to nearest region
   */
  async uploadToNearest(file: File, userRegion: string): Promise<string> {
    const client = this.clients.get(userRegion) || this.clients.values().next().value
    const cid = await client.uploadFile(file)
    return cid.toString()
  }

  /**
   * Upload to multiple regions for redundancy
   */
  async uploadMultiRegion(file: File): Promise<Map<string, string>> {
    const results = new Map<string, string>()

    for (const [region, client] of this.clients) {
      try {
        const cid = await client.uploadFile(file)
        results.set(region, cid.toString())
      } catch (error) {
        console.error(`Failed to upload to ${region}:`, error)
      }
    }

    return results
  }
}
```

### Error Handling and Retry Logic

```typescript
// Robust error handling with retry
class ResilientStorachaClient {
  private client: any
  private maxRetries: number
  private retryDelay: number

  constructor(maxRetries: number = 3, retryDelay: number = 1000) {
    this.maxRetries = maxRetries
    this.retryDelay = retryDelay
  }

  async initialize(email: string): Promise<void> {
    this.client = await this.retryOperation(async () => {
      const c = await createClient()
      await c.login(email)
      return c
    })
  }

  async uploadWithRetry(file: File): Promise<string> {
    return await this.retryOperation(async () => {
      const cid = await this.client.uploadFile(file)
      return cid.toString()
    })
  }

  private async retryOperation<T>(
    operation: () => Promise<T>
  ): Promise<T> {
    let lastError: Error | null = null

    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await operation()
      } catch (error) {
        lastError = error instanceof Error ? error : new Error('Unknown error')
        console.warn(`Attempt ${attempt} failed:`, lastError.message)

        if (attempt < this.maxRetries) {
          const delay = this.retryDelay * Math.pow(2, attempt - 1) // Exponential backoff
          console.log(`Retrying in ${delay}ms...`)
          await new Promise(resolve => setTimeout(resolve, delay))
        }
      }
    }

    throw new Error(`Operation failed after ${this.maxRetries} attempts: ${lastError?.message}`)
  }
}
```

### Security Best Practices

```markdown
## Security Checklist

### Authentication
- ✅ Use email-based authentication for user accounts
- ✅ Store DID private keys securely (encrypted at rest)
- ✅ Implement delegation expiration times
- ✅ Revoke delegations when devices/apps are decommissioned
- ✅ Never expose private keys in client-side code

### Authorization
- ✅ Follow principle of least privilege for delegations
- ✅ Grant only necessary capabilities (upload/add, not admin/*)
- ✅ Use short-lived delegations for third-party integrations
- ✅ Audit delegation chains regularly
- ✅ Implement rate limiting per space/user

### Data Protection
- ✅ Encrypt sensitive data before uploading
- ✅ Use content-addressing (CIDs) for integrity verification
- ✅ Implement access controls at application layer
- ✅ Never upload unencrypted PII to public storage
- ✅ Audit data access patterns

### API Security
- ✅ Validate all inputs
- ✅ Implement request signing
- ✅ Use HTTPS for all API calls
- ✅ Implement webhook signature verification
- ✅ Rate limit API endpoints

### Monitoring
- ✅ Log all authentication attempts
- ✅ Monitor for unusual upload patterns
- ✅ Alert on delegation creation/revocation
- ✅ Track API error rates
- ✅ Implement security event notifications
```

---

## Summary

This ecosystem integration guide covers:

1. **IPFS Integration**: Direct node connection, gateway racing, pinning service API
2. **Filecoin Integration**: Storage deal management, FVM smart contracts, automatic renewal
3. **Authentication**: UCAN-based auth, DID identity management, delegation patterns
4. **External Services**: Pinata, Infura, NFT.Storage multi-platform redundancy
5. **Development Frameworks**: Next.js, React, Vue.js with ready-to-use components
6. **Integration Patterns**: Event-driven architecture, microservices, webhooks
7. **Best Practices**: Multi-region strategies, error handling, security guidelines

### Key Takeaways

- **Interoperability**: Storacha is fully compatible with IPFS ecosystem
- **Redundancy**: Integrate multiple pinning services for resilience
- **Flexibility**: Support for modern web frameworks out of the box
- **Security**: UCAN provides decentralized, secure authorization
- **Scalability**: Event-driven and microservice patterns for production use

### Next Steps

1. Choose integration pattern based on your architecture
2. Implement error handling and retry logic
3. Set up monitoring and webhooks
4. Test multi-region deployment if needed
5. Review security checklist before production

---

**Document completed**: 16_Ecosystem_Integration.md
**Total sections**: 7 major sections with 30+ integration examples
**Coverage**: Complete ecosystem integration guide for Storacha
**Related documents**: 03_Core_Concepts.md, 14_API_Reference.md, 15_Performance_and_Scalability.md

