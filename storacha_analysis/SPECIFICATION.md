# Storacha 생태계 분석 - 문서화 명세서

## 목적
Storacha (web3.storage) 생태계의 전체 아키텍처, 데이터 저장/관리 메커니즘, 그리고 구현 세부사항을 체계적으로 분석하고 문서화합니다.

## 분석 범위

### 1. 핵심 레포지토리

#### 1.1 코어 서비스 레포지토리
- **upload-service** (github.com/storacha/upload-service) - 활성 개발 중인 메인 서비스
  - 2,510+ commits, 36 contributors
  - pnpm monorepo (Node.js v18+)
  - TypeScript 19.7%, JavaScript 79.5%
- **w3up** (github.com/storacha/w3up) - UCAN 프로토콜 구현 (deprecated, 참조용)
  - upload-service로 마이그레이션 완료
  - 484 releases, 29+ contributors
  - 백포팅 목적으로 유지
- **w3infra** (github.com/storacha/w3infra) - 인프라스트럭처 및 서버사이드 구현
  - SST (Serverless Stack) 기반
  - AWS Lambda, DynamoDB, S3, R2
  - seed.run으로 배포 관리
- **specs** (github.com/storacha/specs) - 프로토콜 스펙 문서들
  - 18+ specification documents
  - w3-account, w3-session, w3-store, w3-filecoin 등

#### 1.2 애플리케이션 레포지토리
- **console** (github.com/storacha/console) - 웹 대시보드
  - 2025년 7월 archived (upload-service로 이전)
  - Next.js (TypeScript 97.3%)
  - Tailwind CSS, Sentry 통합
  - 183 commits, 13 contributors
- **w3ui** (github.com/storacha/w3ui) - UI 컴포넌트 라이브러리
  - Headless, type-safe UI 컴포넌트
  - React, Solid, Vue 지원
  - pnpm monorepo 구조
  - 63 stars, MIT + Apache 2.0
- **w3link** (github.com/storacha/w3link) - IPFS 게이트웨이
  - CloudFlare Workers 기반
  - Caching layer (public gateway 위)
  - Parallel gateway requests
  - 23 stars, 21 releases, 78 commits
- **dag.w3s.link** (github.com/storacha/dag.w3s.link) - Trustless Gateway
  - IPFS Trustless Gateway 스펙 구현
  - Graph API 전용
  - CAR 요청 처리 (dag-scope 파라미터)

### 2. 핵심 개념 및 컴포넌트

#### 2.1 UCAN (User Controlled Authorization Networks)
- 사용자 제어 권한 네트워크
- 공개키 암호화 기반의 capability-based 인증 시스템
- 세밀한 권한 공유 및 위임 메커니즘

#### 2.2 Spaces (저장 공간)
- DID (Decentralized Identity Document) 기반 네임스페이스
- `did:key:publicKey` 형식의 고유 식별자
- 사용자 데이터의 논리적 그룹화 단위

#### 2.3 Agents (에이전트)
- 로컬 개인키 관리 컴포넌트
- UCAN 요청 서명 및 전송
- 각 Agent는 고유한 `did:key` 보유

#### 2.4 CAR (Content Addressable aRchive)
- IPLD DAG를 인코딩한 콘텐츠 아카이브 파일
- UnixFS 형식의 DAG 생성
- 샤딩을 통한 대용량 파일 처리

### 3. 데이터 흐름 아키텍처

```
사용자 파일
    ↓
[1] DAG 인코딩 (IPLD/UnixFS)
    ↓
[2] CAR 샤딩 (여러 CAR 파일로 분할)
    ↓
[3] Blob 저장 (blob/add capability)
    ↓
[4] Index 생성 (index/add capability)
    ↓
[5] Upload 등록 (upload/add capability)
    ↓
[6] Filecoin 제공 (filecoin/offer capability)
    ↓
저장 완료 및 검증
```

### 4. 인프라스트럭처 컴포넌트

#### 4.1 Storage & Content Management
- **upload-api**: HTTP 게이트웨이 (Lambda + DynamoDB)
- **carpark**: CAR 파일 버킷 관리 및 알림
- **replicator**: R2 클라우드 스토리지로 복제

#### 4.2 Filecoin Integration
- **filecoin**: Filecoin 딜 처리 Lambda
- **indexer**: Elastic IPFS 연결 및 콘텐츠 발견

#### 4.3 Additional Services
- **billing**: 사용량 계산 및 결제 시스템 연동
- **psa**: Pinning Service API 데이터 마이그레이션
- **roundabout**: Piece CID에서 서명된 URL로 리다이렉션
- **egress-tracking**: Egress 모니터링

### 5. 프로토콜 스펙 (w3 프로토콜 스택)

#### 5.1 Stable Specifications
- **w3-account**: capability 동기화 및 복구
- **w3-session**: 이메일 인증을 통한 capability 위임
- **w3-store**: DAG 샤드를 CAR로 저장
- **w3-filecoin**: Filecoin 저장 커밋먼트 관리

#### 5.2 Additional Specifications
- w3-access, w3-admin, w3-blob, w3-clock
- w3-egress-tracking, w3-index, w3-plan
- w3-provider, w3-rate-limit, w3-replication
- w3-retrieval, w3-revocations-check, w3-space
- w3-store-ipfs-pinning, w3-ucan, w3-ucan-bridge

### 6. 클라이언트 라이브러리 및 도구

#### 6.1 JavaScript/TypeScript
- **@storacha/client**: 고수준 JavaScript 클라이언트
- **@storacha/upload-client**: 저수준 업로드 클라이언트
- **@storacha/access**: 접근 권한 관리 클라이언트
- **@storacha/w3up-client**: (legacy) w3up 클라이언트

#### 6.2 CLI
- **@storacha/cli**: 커맨드라인 인터페이스
- `storacha login`, `storacha space create`, `storacha up` 등의 명령

#### 6.3 Go Client
- **go-w3up**: Go 언어 구현 (Guppy)

#### 6.4 기타 도구
- **add-to-web3**: GitHub Action for CI/CD
- **w3ui**: UI 컴포넌트 라이브러리

### 7. 핵심 Capabilities (권한 체계)

Storacha는 다음과 같은 capability 기반 권한을 사용:

- **blob/add**: 블롭 파일 저장
- **index/add**: 인덱스 등록
- **upload/add**: 업로드 등록 (content CID ↔ shard CID 매핑)
- **filecoin/offer**: Filecoin 저장 제공
- **store/add**: (legacy) 저장 작업
- **space/blob/add**: Space 내 블롭 추가

### 8. 애플리케이션 및 활용 사례

#### 8.1 Console (웹 대시보드)
- **목적**: 브라우저 기반 파일 업로드 및 Space 관리
- **기술**: Next.js, TypeScript, Tailwind CSS, Sentry
- **특징**:
  - w3up 서비스 통합 (`https://up.web3.storage`)
  - 환경 변수로 다른 w3up 인스턴스 연결 가능
  - 브라우저 File API 활용
  - 2025년 upload-service로 마이그레이션

#### 8.2 w3link (IPFS Gateway)
- **목적**: IPFS 콘텐츠를 빠르게 제공하는 캐싱 레이어
- **기술**: CloudFlare Workers, Edge Computing
- **아키텍처**:
  - Public IPFS gateway 위의 caching layer
  - 전역 분산 (serverless code)
  - Parallel gateway requests (가장 빠른 응답 사용)
- **성능**:
  - Rate limiting: 200 req/min per IP
  - 30초 블록 (rate limit 초과 시)
- **접근 방식**:
  - Path-style: `https://w3s.link/ipfs/{cid}`
  - Subdomain-style: `https://{CID}.ipfs.w3s.link/`

#### 8.3 w3ui (UI 컴포넌트)
- **목적**: 재사용 가능한 headless UI 컴포넌트
- **기술**: TypeScript, React, Solid, Vue
- **설계 철학**: Headless, Type-safe, Framework-agnostic
- **주요 컴포넌트**:
  - Sign up/Sign in (이메일 인증, 개인키 생성)
  - File Upload (단일/다중, 드래그앤드롭)
  - Uploads List (업로드 히스토리)
  - Space Management (생성, 선택, 공유)
- **예시 앱**: React, Svelte, Vue 프레임워크별 데모

#### 8.4 dag.w3s.link (Trustless Gateway)
- **목적**: 검증 가능한 IPFS 콘텐츠 접근
- **스펙**: IPFS Trustless Gateway Specification
- **특징**:
  - Graph API 전용
  - `dag-scope` 파라미터 (block/entity/all)
  - `dups` 파라미터 (중복 블록 처리)
  - CAR 응답 형식
  - Cryptographic verification

#### 8.5 실제 활용 사례

**게임 산업:**
- Unreal Engine 플러그인
- Progressive game install
- Content-addressed 게임 바이너리 배포
- 사용자 소유 게임 에셋

**AI & Machine Learning:**
- elizaOS: AI 에이전트의 persistent, verifiable memory
- 분산 웹에서 데이터 저장 및 공유
- 모델 저장 및 버전 관리

**NFT & Digital Assets:**
- NFT 메타데이터 저장 (JSON, 이미지, 미디어)
- Courtyard: 물리적 수집품(Pokémon 카드) 토큰화
- Immutable 참조 보장

**Supply Chain:**
- 제품 추적 및 검증
- 원산지 증명
- 럭셔리 브랜드 활용 (Louis Vuitton, Gucci)

**기타:**
- 분산 웹사이트 호스팅
- 데이터 아카이빙
- CI/CD 통합 (GitHub Actions)

## 문서 구조

본 분석 문서는 다음과 같은 구조로 작성됩니다:

### 메인 문서
1. **00_Storacha_Ecosystem_Overview.md** - 생태계 전체 개요 및 큰 그림
2. **01_UCAN_Protocol_Deep_Dive.md** - UCAN 프로토콜 상세 분석
3. **02_Data_Flow_Architecture.md** - 데이터 흐름 및 처리 파이프라인
4. **03_Space_and_Agent_Management.md** - Space와 Agent 관리 메커니즘

### 컴포넌트별 상세 문서
5. **04_Upload_Service_Implementation.md** - Upload Service 구현 상세
6. **05_Infrastructure_Components.md** - 인프라 컴포넌트별 분석
7. **06_Client_Libraries_API.md** - 클라이언트 라이브러리 API 및 사용법
8. **07_Protocol_Specifications.md** - w3 프로토콜 스펙 상세

### 구현 세부사항 문서
9. **08_CAR_File_Processing.md** - CAR 파일 생성 및 처리
10. **09_Filecoin_Integration.md** - Filecoin 통합 메커니즘
11. **10_Security_and_Authorization.md** - 보안 및 권한 관리
12. **11_Database_and_Storage.md** - 데이터베이스 및 저장소 구조

### 실전 가이드
13. **12_Development_Guide.md** - 개발 환경 설정 및 가이드
14. **13_Deployment_Operations.md** - 배포 및 운영 가이드
15. **14_API_Reference.md** - 전체 API 레퍼런스

### 애플리케이션 및 활용 사례 분석
16. **29_Console_Application_Analysis.md** - Console 대시보드 구현 분석
17. **30_W3link_Gateway_Analysis.md** - w3link IPFS Gateway 구조
18. **31_W3ui_Components_Analysis.md** - w3ui UI 컴포넌트 라이브러리
19. **32_Trustless_Gateway_Analysis.md** - dag.w3s.link Trustless Gateway
20. **33_Real_World_Applications.md** - 실제 활용 사례 및 통합 예시
21. **34_Integration_Patterns.md** - 통합 패턴 및 베스트 프랙티스

## 분석 방법론

### 1. 코드 분석 접근법
- 각 레포지토리의 주요 패키지 및 모듈 파악
- 핵심 함수 및 클래스 식별
- 데이터 흐름 추적
- 의존성 관계 매핑

### 2. 문서화 기준
- 각 컴포넌트의 **목적(Purpose)**
- **핵심 기능(Key Functions)**
- **데이터 구조(Data Structures)**
- **API 인터페이스(API Interface)**
- **구현 세부사항(Implementation Details)**
- **실제 코드 예시(Code Examples)** - 함수명, 클래스명, 주요 로직
- **의도 및 설계 결정(Design Decisions)** - 왜 이렇게 구현했는지

### 3. 코드 레퍼런스 형식
```
파일경로:라인번호 - 함수명/클래스명
예: packages/upload-client/src/upload.js:45 - uploadFile()
```

### 4. 다이어그램 및 시각화
- 아키텍처 다이어그램 (Mermaid 형식)
- 시퀀스 다이어그램
- 데이터 흐름도
- 컴포넌트 관계도

## 기술 스택

### Backend
- **Runtime**: Node.js v18+
- **Language**: TypeScript, JavaScript
- **Framework**: SST (Serverless Stack)
- **RPC**: ucanto (UCAN 기반 RPC)

### Infrastructure
- **Cloud**: AWS (Lambda, DynamoDB, S3)
- **CDN**: Cloudflare R2
- **Storage**: IPFS, Filecoin
- **Build**: pnpm workspaces, Nx

### Frontend
- **Framework**: React
- **UI**: w3ui 컴포넌트
- **Console**: storacha.network/console

## 배포 환경

- **Staging**: https://staging.up.storacha.network
- **Production**: https://up.storacha.network (manual promotion)
- **PR Preview**: https://<pr-number>.up.storacha.network
- **Management**: seed.run

## 참고 리소스

### 공식 문서
- https://docs.storacha.network
- https://github.com/storacha/specs

### 주요 레포지토리
- https://github.com/storacha/upload-service
- https://github.com/storacha/w3infra
- https://github.com/storacha/w3up (deprecated)

### 프로토콜 스펙
- UCAN: https://ucan.xyz
- IPLD: https://ipld.io
- CAR: https://ipld.io/specs/transport/car/

## 버전 정보
- **문서 버전**: 1.0
- **분석 기준 날짜**: 2025-11-14
- **주요 레포지토리 버전**:
  - upload-service: 2,510+ commits
  - w3infra: latest (staging deployment)
  - specs: 18+ specification documents

## 작성 진행 상황
- [x] 문서 구조 정의 및 SPECIFICATION 작성
- [x] TODO.md 작성
- [x] 활용 사례 조사 및 Phase 10 추가
- [ ] 각 메인 문서 작성 (00-04)
- [ ] 컴포넌트별 상세 문서 작성 (05-08)
- [ ] 구현 세부사항 문서 작성 (09-12)
- [ ] 실전 가이드 작성 (13-15)
- [ ] 애플리케이션 분석 작성 (29-34)
- [ ] 최종 검토 및 업데이트
