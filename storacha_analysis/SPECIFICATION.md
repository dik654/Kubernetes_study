# Storacha 생태계 분석 - 문서화 명세서

## 목적
Storacha (web3.storage) 생태계의 전체 아키텍처, 데이터 저장/관리 메커니즘, 그리고 구현 세부사항을 체계적으로 분석하고 문서화합니다.

## 분석 범위

### 1. 핵심 레포지토리
- **upload-service** (github.com/storacha/upload-service) - 활성 개발 중인 메인 서비스
- **w3up** (github.com/storacha/w3up) - UCAN 프로토콜 구현 (deprecated, 하지만 참조용)
- **w3infra** (github.com/storacha/w3infra) - 인프라스트럭처 및 서버사이드 구현
- **specs** (github.com/storacha/specs) - 프로토콜 스펙 문서들
- **w3link** (github.com/storacha-network/w3link) - IPFS 게이트웨이
- **console** (github.com/storacha/console) - 웹 대시보드
- **w3ui** (github.com/storacha/w3ui) - UI 컴포넌트

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
- [ ] 문서 구조 정의 및 SPECIFICATION 작성
- [ ] TODO.md 작성
- [ ] 각 메인 문서 작성 (00-04)
- [ ] 컴포넌트별 상세 문서 작성 (05-08)
- [ ] 구현 세부사항 문서 작성 (09-12)
- [ ] 실전 가이드 작성 (13-15)
- [ ] 최종 검토 및 업데이트
