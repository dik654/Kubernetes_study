# Development Guide

## Table of Contents

- [Local Development Environment](#local-development-environment)
  - [Prerequisites](#prerequisites)
  - [System Requirements](#system-requirements)
  - [Essential Tools](#essential-tools)
- [Repository Setup](#repository-setup)
  - [Clone Repositories](#clone-repositories)
  - [Install Dependencies](#install-dependencies)
  - [Environment Configuration](#environment-configuration)
- [local.storage Setup](#localstorage-setup)
- [upload-service Development](#upload-service-development)
- [w3infra Development](#w3infra-development)
- [Testing Guide](#testing-guide)
- [Debugging and Troubleshooting](#debugging-and-troubleshooting)
- [Contribution Guidelines](#contribution-guidelines)

---

## Local Development Environment

### Prerequisites

To develop with the Storacha ecosystem, you'll need a properly configured development environment with the right tools and dependencies.

#### System Requirements

**Operating Systems:**
- macOS (recommended)
- Linux (Ubuntu 20.04+, Debian 11+)
- Windows (with WSL2)

**Hardware Recommendations:**
- CPU: Multi-core processor (4+ cores recommended)
- RAM: 8 GB minimum, 16 GB recommended
- Storage: 20 GB free space minimum
- Network: Stable internet connection for package downloads

### Essential Tools

#### 1. Node.js

**Required Versions:**
- **upload-service**: Node.js v18.x
- **local.storage**: Node.js v20.11+
- **w3infra**: Node.js v16+

**Installation:**

```bash
# Using nvm (recommended for version management)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Install Node.js 18 (for upload-service)
nvm install 18
nvm use 18

# Install Node.js 20 (for local.storage)
nvm install 20

# Verify installation
node --version
npm --version
```

**Alternative Installation Methods:**

```bash
# macOS with Homebrew
brew install node@18

# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify
node --version  # Should show v18.x.x
```

#### 2. pnpm Package Manager

**Why pnpm?**
- Faster than npm/yarn
- Efficient disk space usage (content-addressable storage)
- Strict dependency resolution
- Better monorepo support

**Installation:**

```bash
# Using npm
npm install -g pnpm

# Using Homebrew (macOS)
brew install pnpm

# Using standalone script (Linux/macOS)
curl -fsSL https://get.pnpm.io/install.sh | sh -

# Verify installation
pnpm --version  # Should show 8.x or higher
```

**Configure pnpm:**

```bash
# Set store directory (optional)
pnpm config set store-dir ~/.pnpm-store

# Enable workspace support
pnpm config set recursive-install true
```

#### 3. Git

```bash
# macOS
brew install git

# Ubuntu/Debian
sudo apt-get install git

# Verify
git --version
```

**Configure Git:**

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main
```

#### 4. Docker (for testing)

**Required for running integration tests and some local services.**

```bash
# macOS
brew install --cask docker

# Ubuntu
sudo apt-get install docker.io docker-compose
sudo usermod -aG docker $USER

# Verify
docker --version
docker-compose --version
```

#### 5. AWS CLI (for w3infra)

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure (use dummy credentials for local dev)
aws configure
# AWS Access Key ID: test
# AWS Secret Access Key: test
# Default region name: us-east-1
# Default output format: json
```

#### 6. Code Editor

**Recommended: Visual Studio Code**

```bash
# macOS
brew install --cask visual-studio-code

# Extensions to install:
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension ms-vscode.vscode-typescript-next
```

**VS Code Settings for Storacha Development:**

`.vscode/settings.json`:
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "files.eol": "\n"
}
```

---

## Repository Setup

### Clone Repositories

#### Main Repositories

```bash
# Create workspace directory
mkdir ~/storacha-dev
cd ~/storacha-dev

# Clone upload-service (main service repository)
git clone https://github.com/storacha/upload-service.git
cd upload-service

# Clone w3infra (infrastructure)
cd ~/storacha-dev
git clone https://github.com/storacha/w3infra.git

# Clone local.storage (local development backend)
cd ~/storacha-dev
git clone https://github.com/storacha/local.storage.git

# Clone console (web interface)
cd ~/storacha-dev
git clone https://github.com/storacha/console.git
```

### Install Dependencies

#### upload-service Setup

```bash
cd ~/storacha-dev/upload-service

# Install dependencies with pnpm
pnpm install

# Verify installation
pnpm list --depth 0
```

**Expected Workspace Structure:**

```
upload-service/
├── packages/
│   ├── access/         # Access control & delegation
│   ├── capabilities/   # UCAN capabilities definitions
│   ├── cli/           # Command-line interface
│   ├── client/        # High-level client library
│   ├── upload-client/ # Low-level upload client
│   ├── w3up-client/   # Unified client (main entry point)
│   └── ... (additional packages)
├── pnpm-workspace.yaml
├── package.json
└── tsconfig.json
```

**Workspace Commands:**

```bash
# Install all workspace dependencies
pnpm install

# Run command in specific workspace
pnpm --filter @storacha/client build

# Run command in all workspaces
pnpm -r build

# Add dependency to specific package
pnpm --filter @storacha/client add multiformats
```

#### local.storage Setup

```bash
cd ~/storacha-dev/local.storage

# Install dependencies
npm install

# Verify installation
npm list --depth=0
```

#### w3infra Setup

```bash
cd ~/storacha-dev/w3infra

# Install dependencies
npm install

# Install SST globally (optional)
npm install -g sst

# Verify SST
sst version
```

---

## Environment Configuration

### upload-service Environment

The upload-service packages typically don't require environment configuration for development, but you can configure client endpoints:

**For testing against local services:**

```bash
# Create .env file in project root
cd ~/storacha-dev/upload-service

cat > .env <<EOF
# Local development
W3UP_SERVICE_URL=http://localhost:3000
W3UP_SERVICE_DID=did:key:your-service-did
W3_STORE_NAME=w3cli-local
EOF
```

### local.storage Environment

**Required Configuration:**

```bash
cd ~/storacha-dev/local.storage

# Copy template
cp .env.template .env
```

**Edit `.env`:**

```bash
# Required: Private key for service identity
# Generate with: w3 key create
PRIVATE_KEY=Mb64padEncoded_Ed25519_PrivateKey_Here

# Optional: Storage directory
DATA_DIR=./data

# Optional: API port
API_PORT=3000

# Optional: Public URLs
PUBLIC_API_URL=http://localhost:3000
PUBLIC_UPLOAD_URL=http://localhost:3000
```

**Generate Private Key:**

```bash
# Install w3cli first
npm install -g @storacha/cli

# Create new key
w3 key create

# Output will include a multibase-encoded private key
# Copy the "Secret:" value to PRIVATE_KEY in .env
```

### w3infra Environment

**For local development:**

```bash
cd ~/storacha-dev/w3infra

# Copy template
cp .env.tpl .env.local
```

**Edit `.env.local`:**

```typescript
// Minimum configuration for local development
{
  // Domain configuration
  HOSTED_ZONE: 'up.local.storacha.network',

  // Service identity (generate with w3 key create)
  PRIVATE_KEY: 'Mb64pad...',

  // Or provide DID directly
  UPLOAD_API_DID: 'did:key:z6Mk...',

  // Stripe (use test keys for local dev)
  STRIPE_SECRET_KEY: 'sk_test_...',
  STRIPE_PRICING_TABLE_ID: 'prctbl_test_...',
  STRIPE_PUBLISHABLE_KEY: 'pk_test_...',

  // R2 configuration (optional for local dev)
  R2_ACCESS_KEY_ID: 'test',
  R2_SECRET_ACCESS_KEY: 'test',
  R2_BUCKET_NAME: 'carpark-dev',

  // DynamoDB (will use local Docker if available)
  DYNAMODB_ENDPOINT: 'http://localhost:8000'
}
```

**Full Environment Variables Reference:**

| Variable | Purpose | Required | Example |
|----------|---------|----------|---------|
| `HOSTED_ZONE` | Root domain for API | Yes | `up.storacha.network` |
| `PRIVATE_KEY` | Service signing key | Yes (or DID) | `Mb64pad...` |
| `UPLOAD_API_DID` | Service identifier | Yes (or KEY) | `did:key:z6Mk...` |
| `STRIPE_SECRET_KEY` | Payment processing | For billing | `sk_test_...` |
| `R2_BUCKET_NAME` | CAR storage bucket | For storage | `carpark-prod-0` |
| `R2_ACCESS_KEY_ID` | R2 credentials | For storage | `...` |
| `R2_SECRET_ACCESS_KEY` | R2 credentials | For storage | `...` |
| `DYNAMODB_TABLE_PREFIX` | Table naming | No | `w3infra-dev-` |

---

## local.storage Setup

local.storage provides a self-hosted w3up backend for local development, fully compatible with w3 CLI and SDK.

### Why local.storage?

- **No cloud dependencies**: No AWS, DynamoDB, or S3 required
- **Disk-based storage**: Uses Pail (DAG-based key-value store)
- **Fast iteration**: Test changes without deploying to cloud
- **Offline development**: Work without internet connection
- **Full compatibility**: Works with existing w3 tools

### Start local.storage

```bash
cd ~/storacha-dev/local.storage

# Start the service
npm start

# Expected output:
# > local.storage@1.0.0 start
# > node src/index.js
#
# Service listening on http://localhost:3000
# Service DID: did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...
#
# To use with w3cli:
# export W3UP_SERVICE_URL=http://localhost:3000
# export W3UP_SERVICE_DID=did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...
# export W3_STORE_NAME=w3cli-local
```

### Configure w3cli to use local.storage

```bash
# Set environment variables
export W3UP_SERVICE_URL=http://localhost:3000
export W3UP_SERVICE_DID=did:key:z6MkqG7kBP8wFWCqnGmjf8AqXBLqJ9sZ...
export W3_STORE_NAME=w3cli-local

# Verify configuration
w3 whoami
# Should show local service DID

# Create a space
w3 space create my-dev-space

# Upload a file
echo "Hello from local.storage!" > test.txt
w3 up test.txt

# Verify upload
w3 ls
```

### Directory Structure

local.storage creates the following structure:

```
./data/
├── store/          # Blob storage (CAR files)
├── uploads/        # Upload metadata
├── spaces/         # Space configurations
└── delegations/    # UCAN delegations
```

### Persistence

All data is stored in the `DATA_DIR` (default: `./data`). To reset:

```bash
# Stop the service (Ctrl+C)

# Remove data directory
rm -rf ./data

# Restart service
npm start

# Reconfigure w3cli with new DID
export W3UP_SERVICE_DID=<new-did-from-startup>
```

---

## upload-service Development

### Workspace Structure

```typescript
// pnpm-workspace.yaml
packages:
  - 'packages/*'
```

Each package in `packages/` is independently testable and publishable.

### Common Development Tasks

#### 1. Build Packages

```bash
cd ~/storacha-dev/upload-service

# Build all packages
pnpm -r build

# Build specific package
pnpm --filter @storacha/client build

# Watch mode for development
pnpm --filter @storacha/client build:watch
```

#### 2. Run Type Checking

```bash
# Type check all packages
pnpm -r typecheck

# Type check specific package
pnpm --filter @storacha/w3up-client typecheck
```

#### 3. Linting

```bash
# Lint all packages
pnpm -r lint

# Lint and fix
pnpm -r lint:fix

# Lint specific package
pnpm --filter @storacha/client lint
```

#### 4. Format Code

```bash
# Format all code with Prettier
pnpm -r format

# Check formatting without changes
pnpm -r format:check
```

### Working with Individual Packages

#### Example: Developing @storacha/client

```bash
cd ~/storacha-dev/upload-service/packages/client

# Install dependencies (if not done)
pnpm install

# Build the package
pnpm build

# Run tests
pnpm test

# Watch mode for tests
pnpm test:watch

# Type check
pnpm typecheck
```

#### Link Local Package for Testing

```bash
# In the package you're developing
cd ~/storacha-dev/upload-service/packages/client
pnpm link --global

# In your test project
cd ~/my-test-project
pnpm link --global @storacha/client

# Now your test project uses the local version
```

### Example: Creating a Test Client

```javascript
// test-client.js
import { create } from '@storacha/w3up-client'

async function testUpload() {
  // Create client pointing to local.storage
  const client = await create({
    serviceURL: new URL('http://localhost:3000'),
    servicePrincipal: { did: () => process.env.W3UP_SERVICE_DID }
  })

  // Login
  await client.login('your-email@example.com')

  // Create space
  const space = await client.createSpace('test-space')
  await client.setCurrentSpace(space.did())

  // Upload file
  const file = new File(['Hello World!'], 'hello.txt')
  const cid = await client.uploadFile(file)

  console.log('Uploaded with CID:', cid.toString())
}

testUpload().catch(console.error)
```

**Run the test:**

```bash
node test-client.js
```

### Package Scripts Reference

Common scripts available in most packages:

| Script | Purpose |
|--------|---------|
| `pnpm build` | Compile TypeScript to JavaScript |
| `pnpm test` | Run unit tests |
| `pnpm test:watch` | Run tests in watch mode |
| `pnpm lint` | Check code style |
| `pnpm lint:fix` | Fix code style issues |
| `pnpm typecheck` | Run TypeScript type checking |
| `pnpm clean` | Remove build artifacts |

### Monorepo Tips

**1. Dependency Management:**

```bash
# Add dependency to workspace root
pnpm add -w <package>

# Add dependency to specific package
pnpm --filter @storacha/client add <package>

# Add dev dependency
pnpm --filter @storacha/client add -D <package>

# Update all dependencies
pnpm up -r
```

**2. Working Across Packages:**

```bash
# Run script in all packages
pnpm -r <script-name>

# Run script in packages matching pattern
pnpm --filter "@storacha/*" build

# Run scripts in topological order (respecting dependencies)
pnpm -r --workspace-concurrency=1 build
```

**3. Debugging Dependency Resolution:**

```bash
# Check why a package is installed
pnpm why <package-name>

# List all installed packages
pnpm list --depth=1

# Check for outdated packages
pnpm outdated
```

---

## w3infra Development

w3infra uses **SST (Serverless Stack)** for infrastructure as code.

### SST Overview

SST is a framework for building serverless applications on AWS:
- Live Lambda development (test locally, run on AWS)
- Type-safe infrastructure definitions
- Automatic environment management
- Built on AWS CDK

### Local Development Mode

**Option 1: No AWS Deployment (Fastest)**

```bash
cd ~/storacha-dev/w3infra

# Start SST without deploying to AWS
npx sst dev --no-deploy

# Expected output:
# SST v2.x.x
# Stage: dev
# Region: us-east-1
#
# Starting Live Lambda Development
# Debug session started. Listening on ws://localhost:12557
```

**Option 2: Full AWS Development**

```bash
# Deploy to your AWS account
npm start

# This runs: npx sst dev
#
# Expected output:
# ✓ Compiled successfully.
# ✓ Deployed stacks:
#   - UploadDbStack
#   - UploadApiStack
#   - CarparkStack
#   ...
#
# Local console: https://console.sst.dev/...
```

**Access the SST Console:**

The console provides:
- Real-time logs
- Function invocations
- DynamoDB tables
- S3 buckets
- CloudWatch metrics

### Stack Structure

```typescript
// sst.config.ts structure
export default {
  config(input) {
    return {
      name: 'w3infra',
      region: 'us-east-1',
      profile: input.stage === 'prod' ? 'prod' : 'dev'
    }
  },
  stacks(app) {
    // Database stacks
    app.stack(BillingDbStack)
    app.stack(UploadDbStack)

    // API stacks
    app.stack(UploadApiStack)
    app.stack(UcanInvocationStack)

    // Storage stacks
    app.stack(CarparkStack)
    app.stack(ReplicatorStack)

    // Filecoin stacks
    app.stack(FilecoinStack)

    // Additional stacks
    app.stack(BusStack)
    app.stack(IndexerStack)
    // ...
  }
}
```

### Environment-Specific Development

```bash
# Development stage (default)
npx sst dev

# Custom stage
npx sst dev --stage my-feature

# Production (careful!)
npx sst dev --stage prod
```

Each stage gets isolated AWS resources:
- DynamoDB tables: `w3infra-{stage}-upload-table`
- S3 buckets: `carpark-{stage}-0`
- Lambda functions: `w3infra-{stage}-upload-handler`

### Working with Stacks

#### View Stack Resources

```bash
# List all stacks
npx sst list

# Get stack outputs
npx sst secrets list

# Get function info
npx sst functions list
```

#### Invoke Functions Locally

```bash
# Invoke a function
npx sst invoke functions/upload-handler.handler --data '{"test": true}'

# Invoke with file
npx sst invoke functions/upload-handler.handler --path test-event.json
```

#### Managing Secrets

```bash
# Set a secret
npx sst secrets set STRIPE_SECRET_KEY sk_test_...

# Set for specific stage
npx sst secrets set STRIPE_SECRET_KEY sk_test_... --stage dev

# List secrets
npx sst secrets list

# Remove secret
npx sst secrets remove SECRET_NAME
```

### Live Lambda Development

SST's killer feature - edit Lambda code and see changes immediately without redeployment.

**How it works:**
1. Your Lambda functions run in AWS
2. SST proxies invocations to your local code
3. You debug with breakpoints and console.log
4. Changes reflect instantly

**Example Workflow:**

```javascript
// functions/upload-handler.ts
export async function handler(event) {
  console.log('Event:', event)  // Visible in local terminal

  // Set breakpoint here in VS Code
  const result = await processUpload(event)

  return {
    statusCode: 200,
    body: JSON.stringify(result)
  }
}
```

**Start live development:**

```bash
npx sst dev

# In another terminal, trigger the function:
curl -X POST https://dev.up.storacha.network/upload \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'

# See output in first terminal
```

### Testing Infrastructure

```bash
# Run all tests
npm test

# Run specific package tests
npm test -w packages/upload-api

# Run integration tests (requires deployment)
npm run test:integration
```

### Deployment

```bash
# Deploy to dev
npx sst deploy --stage dev

# Deploy to production (via Seed.run in CI/CD)
# Don't do this locally!
```

---

## Testing Guide

### Overview of Testing Strategy

The Storacha ecosystem uses a comprehensive testing approach:

1. **Unit Tests**: Test individual functions and modules
2. **Integration Tests**: Test component interactions
3. **End-to-End Tests**: Test complete workflows
4. **Contract Tests**: Verify UCAN protocol compliance

### upload-service Testing

#### Running Unit Tests

```bash
cd ~/storacha-dev/upload-service

# Run all tests
pnpm test

# Run tests for specific package
pnpm --filter @storacha/client test

# Run tests in watch mode
pnpm --filter @storacha/client test:watch

# Run tests with coverage
pnpm --filter @storacha/client test:coverage
```

#### Test File Structure

```typescript
// packages/client/test/client.test.js
import { test } from '@storacha/client/test'
import { create } from '@storacha/client'

test('should create client', async () => {
  const client = await create()
  assert.ok(client)
  assert.ok(client.did())
})

test('should upload file', async () => {
  const client = await create()
  const space = await client.createSpace('test')
  await client.setCurrentSpace(space.did())

  const file = new File(['test content'], 'test.txt')
  const cid = await client.uploadFile(file)

  assert.ok(cid)
  assert.equal(cid.version, 1)
})
```

#### Writing Tests

**Test Utilities:**

```javascript
import { test } from '@storacha/client/test'
import assert from 'assert'
import { CID } from 'multiformats/cid'
import { sha256 } from 'multiformats/hashes/sha2'
import * as raw from 'multiformats/codecs/raw'

// Helper: Create test CID
async function createTestCID(data) {
  const bytes = new TextEncoder().encode(data)
  const hash = await sha256.digest(bytes)
  return CID.create(1, raw.code, hash)
}

// Helper: Create test client
async function createTestClient() {
  const { create } = await import('@storacha/client')
  return create()
}

test('my feature test', async () => {
  const client = await createTestClient()
  const cid = await createTestCID('test data')

  // Test implementation
  const result = await client.someMethod(cid)

  assert.equal(result.success, true)
})
```

**Mocking Services:**

```javascript
import { test } from '@storacha/client/test'
import { mockService } from '@storacha/client/test/helpers'

test('should handle service errors', async () => {
  const service = mockService({
    store: {
      add: async () => {
        throw new Error('Storage full')
      }
    }
  })

  const client = await create({ service })

  await assert.rejects(
    async () => await client.uploadFile(file),
    /Storage full/
  )
})
```

### w3infra Testing

#### Prerequisites for Testing

```bash
# 1. Install Docker Desktop (required for local DynamoDB)
# macOS: Install from docker.com
# Linux:
sudo apt-get install docker.io

# 2. Set AWS credentials (can be dummy for local tests)
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_REGION=us-east-1
```

#### Unit Tests

```bash
cd ~/storacha-dev/w3infra

# Run all unit tests
npm test

# Run tests for specific package
npm test -w packages/upload-api

# Run with verbose output
npm test -- --verbose

# Run specific test file
npm test -- test/upload.test.js
```

#### Integration Tests

Integration tests require actual deployment:

```bash
# Deploy to test environment
npm run deploy --stage test

# Run integration tests
npm run test:integration

# Clean up test environment
npx sst remove --stage test
```

**Example Integration Test:**

```typescript
// test/integration/upload.test.ts
import { test } from 'node:test'
import assert from 'node:assert'
import { create } from '@storacha/client'

test('full upload workflow', async () => {
  // Create client pointing to deployed test environment
  const client = await create({
    serviceURL: new URL(process.env.TEST_SERVICE_URL)
  })

  // Login
  await client.login('test@example.com')

  // Create space
  const space = await client.createSpace('test-space')
  await client.setCurrentSpace(space.did())

  // Upload file
  const file = new File(['integration test'], 'test.txt')
  const cid = await client.uploadFile(file)

  // Verify upload
  const uploads = await client.listUploads()
  const upload = uploads.find(u => u.root.equals(cid))

  assert.ok(upload, 'Upload should be listed')
  assert.equal(upload.root.toString(), cid.toString())
})
```

#### Test Fixtures

```typescript
// test/fixtures/delegations.ts
import { Delegation } from '@ucanto/core'
import * as ed25519 from '@ucanto/principal/ed25519'

export async function createTestDelegation() {
  const issuer = await ed25519.generate()
  const audience = await ed25519.generate()

  return await Delegation.delegate({
    issuer,
    audience,
    capabilities: [{
      can: 'store/add',
      with: issuer.did()
    }],
    expiration: Math.floor(Date.now() / 1000) + 86400
  })
}
```

### Test-Driven Development Workflow

**1. Write Failing Test:**

```typescript
// test/new-feature.test.ts
test('should support new feature', async () => {
  const result = await client.newFeature()
  assert.equal(result.success, true)
})

// Run test - it should fail
// npm test
```

**2. Implement Feature:**

```typescript
// src/client.ts
class Client {
  async newFeature() {
    // Implementation
    return { success: true }
  }
}
```

**3. Run Test - Should Pass:**

```bash
npm test
```

**4. Refactor and Verify:**

```bash
# Ensure tests still pass after refactoring
npm test
```

### Testing Best Practices

**1. Test Isolation:**

```javascript
// Good: Each test is independent
test('test A', async () => {
  const client = await create()  // Fresh client
  // Test A logic
})

test('test B', async () => {
  const client = await create()  // Fresh client
  // Test B logic
})

// Bad: Tests share state
let sharedClient
test('test A', async () => {
  sharedClient = await create()
  // Test A logic
})

test('test B', async () => {
  // Uses sharedClient from test A - fragile!
})
```

**2. Descriptive Test Names:**

```javascript
// Good
test('should reject upload when space storage limit exceeded', async () => {
  // ...
})

// Bad
test('upload test 1', async () => {
  // ...
})
```

**3. Arrange-Act-Assert Pattern:**

```javascript
test('should calculate correct CID for file', async () => {
  // Arrange: Set up test data
  const file = new File(['test content'], 'test.txt')
  const expectedCID = 'bafybeig...'

  // Act: Execute the operation
  const actualCID = await calculateCID(file)

  // Assert: Verify the result
  assert.equal(actualCID.toString(), expectedCID)
})
```

**4. Test Coverage Goals:**

```bash
# Generate coverage report
pnpm test:coverage

# View coverage in browser
open coverage/index.html

# Aim for:
# - Statements: 80%+
# - Branches: 70%+
# - Functions: 80%+
# - Lines: 80%+
```

---

## Debugging and Troubleshooting

### Debugging Tools

#### 1. VS Code Debugger

**`.vscode/launch.json` Configuration:**

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Tests",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "pnpm",
      "runtimeArgs": ["test"],
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen",
      "env": {
        "NODE_ENV": "test"
      }
    },
    {
      "name": "Debug SST",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/node_modules/sst/bin/sst.mjs",
      "args": ["dev", "--no-deploy"],
      "console": "integratedTerminal"
    },
    {
      "name": "Debug local.storage",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/src/index.js",
      "cwd": "${workspaceFolder}",
      "envFile": "${workspaceFolder}/.env"
    }
  ]
}
```

**Usage:**
1. Set breakpoints in code (click left margin)
2. Press F5 or click "Run and Debug"
3. Select configuration
4. Step through code with F10 (step over) / F11 (step into)

#### 2. Chrome DevTools for Node

```bash
# Run with inspector
node --inspect-brk src/index.js

# Open chrome://inspect in Chrome
# Click "inspect" on your Node process
```

#### 3. Console Debugging

```javascript
// Strategic console.log placement
console.log('Upload params:', { file, space, options })

// Pretty-print objects
console.dir(complexObject, { depth: null, colors: true })

// Trace function calls
console.trace('Execution path')

// Performance timing
console.time('upload')
await client.uploadFile(file)
console.timeEnd('upload')
```

### Common Issues and Solutions

#### Issue 1: EACCES Permission Errors

**Symptom:**
```
Error: EACCES: permission denied, mkdir '/usr/local/lib/node_modules'
```

**Solution:**
```bash
# Don't use sudo with npm/pnpm!
# Fix npm permissions:
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Or use nvm (recommended)
```

#### Issue 2: pnpm Workspace Resolution Errors

**Symptom:**
```
ERR_PNPM_NO_MATCHING_VERSION  No matching version found for @storacha/client@workspace:*
```

**Solution:**
```bash
# Clear pnpm cache
pnpm store prune

# Remove node_modules
rm -rf node_modules
rm -rf packages/*/node_modules

# Reinstall
pnpm install
```

#### Issue 3: SST Deployment Stuck

**Symptom:**
```
Deploying...
⠋ Creating resources...
(hangs for 10+ minutes)
```

**Solution:**
```bash
# Cancel deployment (Ctrl+C)

# Check AWS CloudFormation stacks
aws cloudformation list-stacks --region us-east-1

# If stack is stuck, delete it
aws cloudformation delete-stack --stack-name w3infra-dev-UploadApiStack

# Restart SST
npx sst dev
```

#### Issue 4: Test Timeout Errors

**Symptom:**
```
Error: Test timeout of 2000ms exceeded
```

**Solution:**
```javascript
// Increase timeout for slow tests
test('slow operation', { timeout: 10000 }, async () => {
  await slowOperation()
})

// Or configure globally
// test/setup.js
import { setTimeout } from 'node:timers/promises'
global.TEST_TIMEOUT = 10000
```

#### Issue 5: Docker Not Running

**Symptom:**
```
Error: Cannot connect to Docker daemon
```

**Solution:**
```bash
# macOS: Start Docker Desktop
open -a Docker

# Linux: Start Docker service
sudo systemctl start docker

# Verify
docker ps
```

### Logging Best Practices

#### Structured Logging

```javascript
import pino from 'pino'

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: {
    target: 'pino-pretty',
    options: {
      colorize: true,
      translateTime: 'SYS:standard',
      ignore: 'pid,hostname'
    }
  }
})

// Usage
logger.info({ userId, spaceId }, 'Creating space')
logger.error({ err, context }, 'Upload failed')
logger.debug({ cid, size }, 'Blob stored')
```

#### Log Levels

```javascript
// Production: Use appropriate log levels
logger.trace('Fine-grained debug info')     // Very verbose
logger.debug('Debug information')           // Development
logger.info('Informational messages')       // Normal operation
logger.warn('Warning conditions')           // Potential issues
logger.error('Error conditions')            // Errors
logger.fatal('Fatal errors')                // Crash imminent
```

#### Environment-Specific Logging

```javascript
const isDevelopment = process.env.NODE_ENV === 'development'

const logger = pino({
  level: isDevelopment ? 'debug' : 'info',
  transport: isDevelopment
    ? { target: 'pino-pretty' }  // Pretty logs in dev
    : undefined                   // JSON logs in prod
})
```

### Debugging UCAN Issues

#### Inspect UCAN Tokens

```javascript
import { Delegation } from '@ucanto/core'

// Decode UCAN
const delegation = await Delegation.extract(ucanBytes)

console.log('Issuer:', delegation.issuer.did())
console.log('Audience:', delegation.audience.did())
console.log('Capabilities:', delegation.capabilities)
console.log('Expiration:', new Date(delegation.expiration * 1000))
console.log('Proofs:', delegation.proofs.map(p => p.toString()))

// Verify delegation
const result = await delegation.verify()
if (result.error) {
  console.error('Verification failed:', result.error)
}
```

#### Trace Delegation Chains

```javascript
async function traceDelegationChain(delegation, depth = 0) {
  const indent = '  '.repeat(depth)

  console.log(`${indent}Delegation:`)
  console.log(`${indent}  Issuer: ${delegation.issuer.did()}`)
  console.log(`${indent}  Audience: ${delegation.audience.did()}`)
  console.log(`${indent}  Capabilities:`, delegation.capabilities)

  if (delegation.proofs.length > 0) {
    console.log(`${indent}  Proofs:`)
    for (const proof of delegation.proofs) {
      await traceDelegationChain(proof, depth + 1)
    }
  }
}

// Usage
await traceDelegationChain(myDelegation)
```

### Performance Profiling

#### Node.js Built-in Profiler

```bash
# Generate CPU profile
node --cpu-prof src/index.js

# Analyze with Chrome DevTools
# 1. Open chrome://inspect
# 2. Click "Open dedicated DevTools for Node"
# 3. Go to "Profiler" tab
# 4. Load the generated .cpuprofile file
```

#### Memory Profiling

```bash
# Generate heap snapshot
node --heapsnapshot-signal=SIGUSR2 src/index.js &
PID=$!

# Trigger snapshot
kill -SIGUSR2 $PID

# Analyze with Chrome DevTools Memory tab
```

#### Benchmark Tests

```javascript
import { benchmark } from '@storacha/client/test'

benchmark('CAR encoding performance', async () => {
  const file = new File([randomBytes(1024 * 1024)], 'test.bin')  // 1 MB
  await encodeFile(file)
})

// Run benchmarks
npm run benchmark
```

---

## Contribution Guidelines

### Getting Started

**1. Fork and Clone:**

```bash
# Fork the repository on GitHub
# Then clone your fork
git clone https://github.com/YOUR_USERNAME/upload-service.git
cd upload-service

# Add upstream remote
git remote add upstream https://github.com/storacha/upload-service.git

# Verify remotes
git remote -v
```

**2. Create Feature Branch:**

```bash
# Update main branch
git checkout main
git pull upstream main

# Create feature branch
git checkout -b feat/my-new-feature

# Or for bug fixes
git checkout -b fix/issue-123
```

### Branch Naming Conventions

| Prefix | Purpose | Example |
|--------|---------|---------|
| `feat/` | New features | `feat/add-batch-upload` |
| `fix/` | Bug fixes | `fix/memory-leak-in-parser` |
| `docs/` | Documentation | `docs/update-readme` |
| `test/` | Test additions/fixes | `test/add-upload-tests` |
| `refactor/` | Code refactoring | `refactor/simplify-delegation-logic` |
| `chore/` | Maintenance tasks | `chore/update-dependencies` |
| `perf/` | Performance improvements | `perf/optimize-car-encoding` |

### Code Style Guidelines

#### 1. JavaScript/TypeScript Style

```javascript
// Use TypeScript for new code
// Use JSDoc for JavaScript files

/**
 * Upload a file to the specified space.
 *
 * @param {File} file - The file to upload
 * @param {Object} options - Upload options
 * @param {string} options.space - Space DID
 * @param {Function} [options.onProgress] - Progress callback
 * @returns {Promise<CID>} The content identifier
 */
async function uploadFile(file, options) {
  const { space, onProgress } = options

  // Validate inputs
  if (!file) throw new Error('File is required')
  if (!space) throw new Error('Space is required')

  // Implementation
  const cid = await encodeAndUpload(file, { onProgress })

  return cid
}
```

#### 2. Formatting Rules

```javascript
// 2 spaces for indentation (configured in .prettierrc.json)
// Single quotes for strings
// Semicolons required
// Trailing commas in multi-line constructs

const config = {
  endpoint: 'https://up.storacha.network',
  timeout: 30000,
  retries: 3,  // Trailing comma
}

// Max line length: 100 characters
// Break long lines at logical points
const result = await client.uploadFile(
  file,
  {
    space: spaceDID,
    onProgress: (progress) => console.log(progress),
  }
)
```

**Auto-format before committing:**

```bash
# Format all code
pnpm format

# Or set up pre-commit hook
cat > .git/hooks/pre-commit <<'EOF'
#!/bin/sh
pnpm format
git add -u
EOF

chmod +x .git/hooks/pre-commit
```

#### 3. TypeScript Guidelines

```typescript
// Use explicit return types for public APIs
export async function uploadFile(file: File): Promise<CID> {
  // Implementation
}

// Use interfaces for object shapes
interface UploadOptions {
  space: string
  onProgress?: (bytes: number) => void
  signal?: AbortSignal
}

// Use types for unions and complex types
type UploadStatus = 'pending' | 'uploading' | 'done' | 'failed'

// Avoid 'any' - use 'unknown' if needed
function processData(data: unknown) {
  if (typeof data === 'string') {
    return data.toUpperCase()
  }
  throw new Error('Invalid data type')
}
```

### Commit Message Format

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only changes
- `style`: Code style changes (formatting, missing semi-colons, etc.)
- `refactor`: Code refactoring
- `perf`: Performance improvement
- `test`: Adding missing tests
- `chore`: Changes to build process or auxiliary tools

**Examples:**

```
feat(client): add batch upload support

Implements batch upload to allow uploading multiple files in a single
request. This reduces overhead and improves performance for bulk uploads.

Closes #123

---

fix(parser): handle malformed CAR headers

Previously, malformed CAR headers would cause a crash. Now we detect
invalid headers and throw a descriptive error.

Fixes #456

---

docs: update development guide

Add section on debugging UCAN issues and profiling performance.
```

### Pull Request Process

**1. Prepare Your Changes:**

```bash
# Ensure tests pass
pnpm test

# Ensure linting passes
pnpm lint

# Build packages
pnpm build

# Update documentation if needed
```

**2. Push to Your Fork:**

```bash
git push origin feat/my-new-feature
```

**3. Create Pull Request:**

Go to GitHub and create a pull request from your fork to `storacha/upload-service:main`.

**PR Template:**

```markdown
## Description

Brief description of the changes.

## Related Issues

Closes #123
Related to #456

## Changes Made

- Added batch upload API
- Updated client documentation
- Added tests for batch operations

## Testing

- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing completed

## Checklist

- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] Tests added for new functionality
- [ ] All tests passing
- [ ] No breaking changes (or documented if required)
```

**4. Code Review Process:**

- Maintainers will review your PR
- Address feedback by pushing additional commits
- Once approved, a maintainer will merge

**5. After Merge:**

```bash
# Update your local main
git checkout main
git pull upstream main

# Delete feature branch
git branch -d feat/my-new-feature
git push origin --delete feat/my-new-feature
```

### Testing Requirements

**For all PRs:**
- [ ] Existing tests must pass
- [ ] New features must include tests
- [ ] Bug fixes should include regression tests
- [ ] Coverage should not decrease

**Test Checklist:**

```javascript
// For new features:
test('feature works in happy path', async () => { })
test('feature handles errors gracefully', async () => { })
test('feature validates inputs', async () => { })
test('feature integrates with existing code', async () => { })

// For bug fixes:
test('regression test for issue #123', async () => {
  // Reproduce the bug
  // Verify the fix works
})
```

### Documentation Requirements

**Update documentation for:**
- New features (README, API docs)
- Breaking changes (CHANGELOG, migration guide)
- Configuration changes (.env.template, docs)
- New dependencies (justify in PR description)

**Documentation Files:**

```
docs/
├── API.md              # API reference
├── ARCHITECTURE.md     # System architecture
├── CONTRIBUTING.md     # Contribution guide
└── DEVELOPMENT.md      # Development guide

packages/*/README.md    # Package-specific docs
```

### Code Review Checklist

**For Reviewers:**
- [ ] Code follows style guidelines
- [ ] Tests are comprehensive
- [ ] Documentation is updated
- [ ] No obvious bugs or security issues
- [ ] Performance implications considered
- [ ] Breaking changes are justified and documented

**For Authors:**
- [ ] Self-review completed before requesting review
- [ ] PR description is clear and complete
- [ ] All CI checks passing
- [ ] Conflicts resolved with main branch
- [ ] Ready for review (not WIP)

### Getting Help

**Communication Channels:**

1. **GitHub Issues**: For bugs and feature requests
   - Search existing issues first
   - Use issue templates
   - Provide reproducible examples

2. **GitHub Discussions**: For questions and ideas
   - Architecture discussions
   - Feature proposals
   - General questions

3. **Discord/Slack**: Real-time communication (if available)
   - Quick questions
   - Community chat
   - Pair programming

**Tips for Getting Good Help:**

```markdown
## Good Issue Report

**Description:**
When uploading files larger than 100MB, the upload fails with "Request timeout".

**Steps to Reproduce:**
1. Create client with default configuration
2. Upload file > 100MB: `client.uploadFile(largeFile)`
3. Observe timeout error after 30 seconds

**Expected Behavior:**
Upload should complete successfully or show progress.

**Actual Behavior:**
Error: Request timeout after 30s

**Environment:**
- @storacha/client version: 1.2.3
- Node.js version: 18.17.0
- OS: macOS 13.4

**Additional Context:**
Works fine with files < 100MB.
```

### Continuous Integration

The project uses GitHub Actions for CI/CD:

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'pnpm'
      - run: pnpm install
      - run: pnpm build
      - run: pnpm test
      - run: pnpm lint
```

**CI Checks for PRs:**
- ✅ Tests pass
- ✅ Linting passes
- ✅ Build succeeds
- ✅ Type checking passes
- ✅ No security vulnerabilities

### Release Process

Releases are managed by maintainers:

1. **Version Bump**: Update version in `package.json`
2. **Changelog**: Update `CHANGELOG.md`
3. **Tag**: Create git tag (e.g., `v1.2.3`)
4. **Publish**: Publish to npm
5. **GitHub Release**: Create release notes

**Semantic Versioning:**
- **Major**: Breaking changes (v1.0.0 → v2.0.0)
- **Minor**: New features (v1.0.0 → v1.1.0)
- **Patch**: Bug fixes (v1.0.0 → v1.0.1)

---

## Summary

This development guide covers:

1. **Environment Setup**: Tools, prerequisites, and configuration
2. **Local Development**: Working with upload-service, local.storage, and w3infra
3. **Testing**: Unit tests, integration tests, and testing best practices
4. **Debugging**: Tools and techniques for troubleshooting
5. **Contributing**: Code style, commit format, and PR process

**Next Steps:**
- Set up your local environment
- Run the tests to verify setup
- Explore the codebase
- Pick an issue and start contributing!

**Resources:**
- [Storacha Documentation](https://docs.storacha.network)
- [GitHub Repositories](https://github.com/storacha)
- [API Reference](https://storacha.network/docs/api)
- [Community Discussions](https://github.com/storacha/upload-service/discussions)

Happy coding! 🚀
