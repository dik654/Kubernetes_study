# Storacha 생태계 분석 및 문서화 TODO

## 📋 프로젝트 개요
Storacha (web3.storage) 생태계의 전체 아키텍처, 데이터 저장/관리 메커니즘을 체계적으로 분석하고, 실제 코드 구현 세부사항까지 포함한 종합 문서를 작성합니다.

## ✅ 완료된 작업
- [x] 주요 레포지토리 조사 및 식별
- [x] 생태계 아키텍처 개요 파악
- [x] 문서화 명세서 (SPECIFICATION.md) 작성
- [x] TODO 계획 수립

---

## 📝 Phase 1: 핵심 개념 및 생태계 개요 문서

### 1.1 생태계 전체 개요
- [ ] **00_Storacha_Ecosystem_Overview.md** 작성
  - [ ] Storacha란 무엇인가? (비전, 목표, 위치)
  - [ ] 전체 아키텍처 다이어그램
  - [ ] 주요 컴포넌트 맵
  - [ ] 핵심 개념 소개 (UCAN, Space, Agent, CAR)
  - [ ] 레포지토리 구조 및 관계
  - [ ] 기술 스택 요약
  - [ ] 사용 사례 및 시나리오

### 1.2 UCAN 프로토콜 심층 분석
- [ ] **01_UCAN_Protocol_Deep_Dive.md** 작성
  - [ ] UCAN 기본 개념 및 원리
  - [ ] Capability-based 인증 시스템
  - [ ] Delegation 체인 메커니즘
  - [ ] DID (Decentralized Identity) 구조
  - [ ] 실제 구현 분석:
    - [ ] `ucanto` 패키지 코드 분석
    - [ ] UCAN 토큰 생성 함수
    - [ ] 검증 및 권한 체크 로직
    - [ ] 서명 및 암호화 메커니즘
  - [ ] UCAN 토큰 예시 및 구조
  - [ ] 보안 고려사항

### 1.3 데이터 흐름 아키텍처
- [ ] **02_Data_Flow_Architecture.md** 작성
  - [ ] End-to-end 데이터 흐름 다이어그램
  - [ ] 각 단계별 상세 설명:
    - [ ] 1단계: 파일 입력 및 준비
    - [ ] 2단계: DAG 인코딩 (IPLD/UnixFS)
    - [ ] 3단계: CAR 샤딩
    - [ ] 4단계: Blob 저장
    - [ ] 5단계: Index 생성
    - [ ] 6단계: Upload 등록
    - [ ] 7단계: Filecoin 제공
  - [ ] 각 단계의 책임 컴포넌트
  - [ ] 데이터 변환 과정
  - [ ] 에러 처리 및 재시도 로직
  - [ ] 성능 최적화 전략

### 1.4 Space와 Agent 관리
- [ ] **03_Space_and_Agent_Management.md** 작성
  - [ ] Space 개념 및 목적
  - [ ] Space 생성 및 관리 로직
  - [ ] Agent 개념 및 역할
  - [ ] Agent 초기화 및 키 관리
  - [ ] Space-Agent 연결 메커니즘
  - [ ] 실제 구현 분석:
    - [ ] Space 생성 함수 및 코드
    - [ ] Agent 인증 로직
    - [ ] DID 생성 및 관리
    - [ ] 로컬 키 저장 메커니즘
  - [ ] 다중 Space 관리
  - [ ] Space 공유 및 권한 위임

---

## 🔧 Phase 2: 컴포넌트별 상세 구현 분석

### 2.1 Upload Service 구현
- [ ] **04_Upload_Service_Implementation.md** 작성
  - [ ] upload-service 레포지토리 구조
  - [ ] 주요 패키지별 분석:
    - [ ] `@storacha/client` 패키지
      - [ ] `uploadFile()` 함수 구현
      - [ ] `uploadDirectory()` 함수 구현
      - [ ] 클라이언트 초기화 로직
    - [ ] `@storacha/upload-client` 패키지
      - [ ] 저수준 업로드 API
      - [ ] CAR 파일 처리
      - [ ] 청크 업로드 로직
    - [ ] `@storacha/access` 패키지
      - [ ] 인증 흐름
      - [ ] 토큰 관리
  - [ ] upload-api 서비스 분석:
    - [ ] API 엔드포인트 정의
    - [ ] 요청 처리 핸들러
    - [ ] DynamoDB 스키마 및 쿼리
    - [ ] Lambda 함수 구조
  - [ ] 에러 처리 및 검증
  - [ ] 테스트 코드 분석

### 2.2 인프라스트럭처 컴포넌트
- [ ] **05_Infrastructure_Components.md** 작성
  - [ ] w3infra 레포지토리 구조
  - [ ] 각 인프라 컴포넌트 상세 분석:

    **Storage & Content Management:**
    - [ ] **upload-api**
      - [ ] Lambda 핸들러 함수들
      - [ ] DynamoDB 테이블 스키마
      - [ ] API Gateway 설정
      - [ ] 인증 미들웨어
    - [ ] **carpark**
      - [ ] CAR 파일 버킷 관리
      - [ ] S3 이벤트 처리
      - [ ] 알림 메커니즘
    - [ ] **replicator**
      - [ ] 복제 로직
      - [ ] R2 연동 코드
      - [ ] 동기화 전략

    **Filecoin Integration:**
    - [ ] **filecoin** 서비스
      - [ ] 딜 제안 생성
      - [ ] 스토리지 프로바이더 통신
      - [ ] 검증 및 모니터링
    - [ ] **indexer**
      - [ ] Elastic IPFS 연동
      - [ ] 인덱싱 로직
      - [ ] 쿼리 최적화

    **Additional Services:**
    - [ ] **billing** 서비스
      - [ ] 사용량 추적
      - [ ] 비용 계산 로직
      - [ ] 결제 시스템 연동
    - [ ] **psa** (Pinning Service API)
      - [ ] 마이그레이션 로직
      - [ ] 호환성 레이어
    - [ ] **roundabout**
      - [ ] CID 라우팅
      - [ ] URL 서명 및 생성
    - [ ] **egress-tracking**
      - [ ] 트래픽 모니터링
      - [ ] 메트릭 수집

  - [ ] SST (Serverless Stack) 설정 분석
  - [ ] 배포 파이프라인
  - [ ] 모니터링 및 로깅

### 2.3 클라이언트 라이브러리 API
- [ ] **06_Client_Libraries_API.md** 작성
  - [ ] JavaScript/TypeScript 클라이언트:
    - [ ] `@storacha/client` API 레퍼런스
      - [ ] 초기화 및 설정
      - [ ] 파일 업로드 메서드
      - [ ] Space 관리 API
      - [ ] 권한 위임 API
    - [ ] `@storacha/upload-client` API
      - [ ] 저수준 작업 API
      - [ ] CAR 파일 처리 API
    - [ ] `@storacha/access` API
      - [ ] 로그인/로그아웃
      - [ ] 토큰 관리
  - [ ] CLI 도구:
    - [ ] `@storacha/cli` 명령어 분석
      - [ ] `storacha login` 구현
      - [ ] `storacha space create` 구현
      - [ ] `storacha up` 구현
      - [ ] 설정 파일 관리
  - [ ] Go 클라이언트 (Guppy):
    - [ ] go-w3up 패키지 구조
    - [ ] 주요 함수 및 타입
  - [ ] UI 컴포넌트 (w3ui):
    - [ ] React 컴포넌트 분석
    - [ ] 상태 관리
    - [ ] 통합 예시
  - [ ] 사용 예시 및 베스트 프랙티스

### 2.4 프로토콜 스펙 상세
- [ ] **07_Protocol_Specifications.md** 작성
  - [ ] specs 레포지토리 구조
  - [ ] Stable Specifications 상세:
    - [ ] **w3-account**
      - [ ] 스펙 정의
      - [ ] 구현 코드 분석
      - [ ] capability 정의
      - [ ] 데이터 구조
    - [ ] **w3-session**
      - [ ] 이메일 인증 흐름
      - [ ] Magic link 생성 및 검증
      - [ ] 세션 관리
    - [ ] **w3-store**
      - [ ] 저장 프로토콜
      - [ ] CAR 파일 처리
      - [ ] 샤드 관리
    - [ ] **w3-filecoin**
      - [ ] Filecoin 통합 스펙
      - [ ] 딜 관리 프로토콜
  - [ ] Additional Specifications:
    - [ ] w3-access, w3-admin 분석
    - [ ] w3-blob, w3-index 분석
    - [ ] w3-provider, w3-space 분석
    - [ ] 기타 스펙 개요
  - [ ] 프로토콜 간 상호작용
  - [ ] 버전 관리 및 호환성

---

## 🎯 Phase 3: 구현 세부사항 및 심화 주제

### 3.1 CAR 파일 처리
- [ ] **08_CAR_File_Processing.md** 작성
  - [ ] CAR (Content Addressable aRchive) 형식 설명
  - [ ] IPLD DAG 구조
  - [ ] UnixFS 인코딩
  - [ ] 실제 구현 분석:
    - [ ] DAG 생성 함수
    - [ ] CAR 인코더/디코더
    - [ ] 샤딩 알고리즘
    - [ ] 청크 크기 최적화
  - [ ] CID (Content Identifier) 생성
  - [ ] Merkle tree 구조
  - [ ] 파일 재구성 로직
  - [ ] 성능 측정 및 최적화

### 3.2 Filecoin 통합
- [ ] **09_Filecoin_Integration.md** 작성
  - [ ] Filecoin 아키텍처 개요
  - [ ] Storacha-Filecoin 연동 구조
  - [ ] 실제 구현 분석:
    - [ ] 딜 제안 생성 코드
    - [ ] 스토리지 프로바이더 선택 로직
    - [ ] 검증 및 증명 처리
    - [ ] 리트리벌 메커니즘
  - [ ] Piece CID 생성 및 관리
  - [ ] 딜 상태 추적
  - [ ] 비용 계산 및 최적화
  - [ ] 실패 처리 및 재시도

### 3.3 보안 및 권한 관리
- [ ] **10_Security_and_Authorization.md** 작성
  - [ ] 전체 보안 모델
  - [ ] UCAN 기반 권한 체계
  - [ ] Capability 정의 및 검증:
    - [ ] blob/add capability 구현
    - [ ] index/add capability 구현
    - [ ] upload/add capability 구현
    - [ ] filecoin/offer capability 구현
  - [ ] Delegation 체인 검증 로직
  - [ ] 서명 및 암호화:
    - [ ] 키 생성 및 관리
    - [ ] 서명 알고리즘
    - [ ] 검증 프로세스
  - [ ] Revocation 메커니즘
  - [ ] 공격 벡터 및 방어 전략
  - [ ] 보안 베스트 프랙티스

### 3.4 데이터베이스 및 저장소
- [ ] **11_Database_and_Storage.md** 작성
  - [ ] DynamoDB 스키마 설계:
    - [ ] Spaces 테이블
    - [ ] Uploads 테이블
    - [ ] Blobs 테이블
    - [ ] Delegations 테이블
  - [ ] 인덱스 및 쿼리 최적화
  - [ ] S3 버킷 구조:
    - [ ] CAR 파일 저장소
    - [ ] 버킷 정책
    - [ ] 생명주기 관리
  - [ ] R2 복제 전략
  - [ ] IPFS 통합:
    - [ ] Pinning 전략
    - [ ] Gateway 구성
  - [ ] 데이터 일관성 보장
  - [ ] 백업 및 복구 전략
  - [ ] 확장성 고려사항

---

## 🚀 Phase 4: 실전 가이드 및 운영

### 4.1 개발 환경 설정
- [ ] **12_Development_Guide.md** 작성
  - [ ] 로컬 개발 환경 구축:
    - [ ] 필수 도구 설치 (Node.js, pnpm, etc.)
    - [ ] 레포지토리 클론 및 설정
    - [ ] 환경 변수 설정
  - [ ] local.storage 실행:
    - [ ] 로컬 서비스 시작
    - [ ] 설정 및 초기화
  - [ ] 개발 워크플로우:
    - [ ] 코드 수정 및 테스트
    - [ ] 디버깅 팁
    - [ ] Hot reload 설정
  - [ ] 테스트 실행:
    - [ ] 유닛 테스트
    - [ ] 통합 테스트
    - [ ] E2E 테스트
  - [ ] 코드 기여 가이드:
    - [ ] 브랜치 전략
    - [ ] PR 프로세스
    - [ ] 코드 리뷰 체크리스트

### 4.2 배포 및 운영
- [ ] **13_Deployment_Operations.md** 작성
  - [ ] 배포 프로세스:
    - [ ] Staging 배포 (자동)
    - [ ] Production 배포 (수동)
    - [ ] PR Preview 환경
  - [ ] SST 배포 명령어 및 설정
  - [ ] seed.run 관리:
    - [ ] 환경 관리
    - [ ] 시크릿 관리
    - [ ] 프로모션 프로세스
  - [ ] 모니터링:
    - [ ] CloudWatch 대시보드
    - [ ] 알람 설정
    - [ ] 로그 분석
  - [ ] 성능 튜닝:
    - [ ] Lambda 최적화
    - [ ] DynamoDB 용량 관리
    - [ ] 캐싱 전략
  - [ ] 장애 대응:
    - [ ] 트러블슈팅 가이드
    - [ ] 롤백 절차
    - [ ] 인시던트 대응

### 4.3 API 레퍼런스
- [ ] **14_API_Reference.md** 작성
  - [ ] HTTP API 엔드포인트:
    - [ ] `/space/*` 엔드포인트
    - [ ] `/upload/*` 엔드포인트
    - [ ] `/blob/*` 엔드포인트
    - [ ] 인증 헤더
    - [ ] 요청/응답 형식
  - [ ] JavaScript API:
    - [ ] 모든 public 메서드 문서화
    - [ ] 타입 정의
    - [ ] 예시 코드
  - [ ] CLI 명령어 레퍼런스:
    - [ ] 모든 명령어 상세 설명
    - [ ] 옵션 및 플래그
    - [ ] 사용 예시
  - [ ] Go API:
    - [ ] 패키지 및 타입
    - [ ] 함수 시그니처
    - [ ] 예시 코드
  - [ ] 에러 코드 및 처리:
    - [ ] 에러 코드 목록
    - [ ] 원인 및 해결 방법

---

## 📊 Phase 5: 고급 주제 및 보완

### 5.1 성능 및 확장성
- [ ] **15_Performance_and_Scalability.md** 작성
  - [ ] 성능 메트릭 및 벤치마크
  - [ ] 병목 지점 분석
  - [ ] 확장성 전략:
    - [ ] 수평 확장
    - [ ] 수직 확장
    - [ ] 샤딩 전략
  - [ ] 캐싱 아키텍처:
    - [ ] CDN 활용
    - [ ] 메모리 캐시
    - [ ] 분산 캐시
  - [ ] 부하 테스트 및 결과
  - [ ] 최적화 체크리스트

### 5.2 에코시스템 통합
- [ ] **16_Ecosystem_Integration.md** 작성
  - [ ] IPFS 생태계 통합:
    - [ ] Public gateway 연동
    - [ ] w3link 게이트웨이 분석
    - [ ] Pinning Service 호환성
  - [ ] Filecoin 네트워크 통합
  - [ ] Web3 지갑 통합 (가능하다면)
  - [ ] 외부 서비스 연동:
    - [ ] GitHub Action
    - [ ] CI/CD 파이프라인
    - [ ] 써드파티 도구
  - [ ] 이벤트 및 웹훅

### 5.3 마이그레이션 가이드
- [ ] **17_Migration_Guide.md** 작성
  - [ ] web3.storage (legacy) → Storacha 마이그레이션
  - [ ] w3up → upload-service 마이그레이션
  - [ ] 기존 데이터 이전 전략
  - [ ] API 호환성 매핑
  - [ ] 단계별 마이그레이션 절차
  - [ ] 주의사항 및 체크리스트

### 5.4 사용 사례 및 튜토리얼
- [ ] **18_Use_Cases_and_Tutorials.md** 작성
  - [ ] 기본 사용 사례:
    - [ ] 단순 파일 업로드
    - [ ] 디렉토리 업로드
    - [ ] 대용량 파일 처리
  - [ ] 고급 사용 사례:
    - [ ] NFT 메타데이터 저장
    - [ ] 분산 웹사이트 호스팅
    - [ ] 데이터 아카이빙
    - [ ] 협업 스토리지
  - [ ] 단계별 튜토리얼:
    - [ ] React 앱에서 Storacha 사용
    - [ ] Node.js 백엔드 통합
    - [ ] CI/CD 파이프라인 구축
  - [ ] 실제 프로젝트 예시
  - [ ] 베스트 프랙티스 및 팁

### 5.5 트러블슈팅 및 FAQ
- [ ] **19_Troubleshooting_and_FAQ.md** 작성
  - [ ] 자주 발생하는 문제:
    - [ ] 인증 실패
    - [ ] 업로드 실패
    - [ ] 권한 오류
    - [ ] 네트워크 문제
  - [ ] 디버깅 가이드:
    - [ ] 로그 확인 방법
    - [ ] 상태 확인 명령어
    - [ ] 네트워크 트레이싱
  - [ ] FAQ:
    - [ ] 개념 관련 질문
    - [ ] 사용 방법 질문
    - [ ] 기술적 질문
  - [ ] 커뮤니티 리소스:
    - [ ] Discord/Slack
    - [ ] GitHub Issues
    - [ ] 포럼

---

## 🔍 Phase 6: 심층 코드 분석

### 6.1 주요 레포지토리 코드 분석

#### upload-service 코드 분석
- [ ] **20_Upload_Service_Code_Analysis.md** 작성
  - [ ] 패키지 구조 상세 분석
  - [ ] 각 패키지별 주요 파일:
    - [ ] `packages/client/src/`
      - [ ] `client.js` - 클라이언트 초기화
      - [ ] `space.js` - Space 관리
      - [ ] `capability.js` - Capability 처리
    - [ ] `packages/upload-client/src/`
      - [ ] `upload.js` - 업로드 로직
      - [ ] `sharding.js` - 샤딩 알고리즘
      - [ ] `car.js` - CAR 파일 처리
    - [ ] `packages/access/src/`
      - [ ] `agent.js` - Agent 구현
      - [ ] `delegations.js` - Delegation 처리
  - [ ] 핵심 함수 분석 (함수명, 파라미터, 리턴값, 로직)
  - [ ] 클래스 및 인터페이스 정의
  - [ ] 타입 정의 (TypeScript)
  - [ ] 의존성 그래프

#### w3infra 코드 분석
- [ ] **21_W3infra_Code_Analysis.md** 작성
  - [ ] Lambda 함수들:
    - [ ] `upload-api` 핸들러들
    - [ ] `carpark` 이벤트 핸들러
    - [ ] `filecoin` 처리 로직
  - [ ] SST 스택 정의:
    - [ ] `stacks/` 디렉토리 분석
    - [ ] 각 스택의 리소스 정의
    - [ ] 환경 변수 및 설정
  - [ ] 데이터베이스 스키마:
    - [ ] DynamoDB 테이블 정의
    - [ ] 인덱스 설정
  - [ ] 유틸리티 함수들
  - [ ] 설정 파일 분석

#### specs 구현 코드 매핑
- [ ] **22_Specs_Implementation_Mapping.md** 작성
  - [ ] 각 스펙 문서와 실제 구현 코드 매핑
  - [ ] 스펙 변경 이력
  - [ ] 미구현 스펙 식별
  - [ ] 구현 차이점 분석

### 6.2 핵심 알고리즘 분석
- [ ] **23_Core_Algorithms.md** 작성
  - [ ] CAR 샤딩 알고리즘:
    - [ ] 샤드 크기 결정 로직
    - [ ] 경계 처리
    - [ ] 메모리 효율성
  - [ ] CID 계산 알고리즘
  - [ ] Merkle tree 생성 알고리즘
  - [ ] Delegation 체인 검증 알고리즘
  - [ ] 스토리지 프로바이더 선택 알고리즘
  - [ ] 시간/공간 복잡도 분석

### 6.3 테스트 코드 분석
- [ ] **24_Test_Code_Analysis.md** 작성
  - [ ] 테스트 전략 및 구조
  - [ ] 유닛 테스트:
    - [ ] 주요 테스트 케이스
    - [ ] Mock 및 Stub 사용
  - [ ] 통합 테스트:
    - [ ] 시나리오별 테스트
    - [ ] 엔드투엔드 플로우
  - [ ] 성능 테스트
  - [ ] 보안 테스트
  - [ ] 코드 커버리지 분석

---

## 📚 Phase 7: 최종 정리 및 메타 문서

### 7.1 용어집
- [ ] **25_Glossary.md** 작성
  - [ ] 모든 기술 용어 정의
  - [ ] 약어 및 축약어
  - [ ] 도메인 특화 용어
  - [ ] 알파벳순 정리

### 7.2 참고 자료 및 링크
- [ ] **26_References.md** 작성
  - [ ] 공식 문서 링크
  - [ ] GitHub 레포지토리 링크
  - [ ] 관련 프로토콜 스펙
  - [ ] 논문 및 블로그 포스트
  - [ ] 비디오 및 튜토리얼
  - [ ] 커뮤니티 리소스

### 7.3 버전 히스토리
- [ ] **27_Version_History.md** 작성
  - [ ] Storacha 주요 버전별 변경사항
  - [ ] 프로토콜 진화 과정
  - [ ] 중단된 기능 (Deprecations)
  - [ ] 마이그레이션 경로

### 7.4 아키텍처 결정 기록 (ADR)
- [ ] **28_Architecture_Decision_Records.md** 작성
  - [ ] 주요 아키텍처 결정 사항들
  - [ ] 결정 배경 및 이유
  - [ ] 대안 검토
  - [ ] 결과 및 영향
  - [ ] 교훈

### 7.5 종합 인덱스
- [ ] **00_INDEX.md** 작성 (문서 네비게이션)
  - [ ] 모든 문서 목록 및 링크
  - [ ] 주제별 분류
  - [ ] 난이도별 추천 순서
  - [ ] 빠른 참조 가이드

---

## 🎨 Phase 8: 다이어그램 및 시각화

- [ ] **diagrams/** 디렉토리 생성
  - [ ] 아키텍처 다이어그램 (Mermaid)
  - [ ] 시퀀스 다이어그램
  - [ ] 데이터 플로우 다이어그램
  - [ ] 컴포넌트 관계도
  - [ ] 배포 다이어그램
  - [ ] 네트워크 토폴로지

---

## ✨ Phase 9: 코드 예시 및 스니펫

- [ ] **examples/** 디렉토리 생성
  - [ ] 기본 사용 예시 코드
  - [ ] 고급 사용 패턴
  - [ ] 통합 예시
  - [ ] 테스트 예시
  - [ ] 유틸리티 함수들

---

## 🌐 Phase 10: Applications & Use Cases Deep Dive

### 10.1 Console (Dashboard Application)
- [ ] **29_Console_Application_Analysis.md** 작성
  - [ ] Console 개요 및 목적
  - [ ] Next.js 아키텍처 분석:
    - [ ] 페이지 구조 (`app/page.tsx`)
    - [ ] 라우팅 및 네비게이션
    - [ ] 서버/클라이언트 컴포넌트 구분
  - [ ] Storacha 서비스 통합:
    - [ ] w3up 서비스 연결 설정
    - [ ] DID 설정 및 인증
    - [ ] 환경 변수 구성
  - [ ] 주요 기능 구현:
    - [ ] 파일 업로드 UI 및 로직
    - [ ] Space 관리 인터페이스
    - [ ] 업로드 히스토리 표시
    - [ ] 사용량 대시보드
  - [ ] 브라우저 기반 업로드 메커니즘:
    - [ ] File API 활용
    - [ ] 청크 업로드 처리
    - [ ] 진행률 표시
  - [ ] 에러 모니터링 (Sentry):
    - [ ] 클라이언트/서버/엣지 설정
    - [ ] 에러 추적 및 보고
  - [ ] 배포 및 마이그레이션:
    - [ ] upload-service 레포로 이전 과정
    - [ ] 배포 환경 설정

### 10.2 w3link (IPFS Gateway)
- [ ] **30_W3link_Gateway_Analysis.md** 작성
  - [ ] w3link 개요 및 목적
  - [ ] 아키텍처 설계:
    - [ ] "Caching layer" 컨셉
    - [ ] Public gateway 위에서 동작하는 방식
  - [ ] CloudFlare Workers 구현:
    - [ ] Edge-gateway-link 패키지 분석
    - [ ] Worker 코드 구조
    - [ ] 전역 배포 메커니즘
  - [ ] 콘텐츠 검색 방식:
    - [ ] Path-style: `https://w3s.link/ipfs/{cid}`
    - [ ] Subdomain-style: `https://{CID}.ipfs.w3s.link/`
  - [ ] Caching 메커니즘:
    - [ ] Edge 캐시 전략
    - [ ] TTL 설정
    - [ ] Cache invalidation
  - [ ] Parallel Gateway Requests:
    - [ ] 여러 public gateway 동시 요청
    - [ ] 가장 빠른 응답 선택 로직
    - [ ] Fallback 처리
  - [ ] Performance 최적화:
    - [ ] Rate limiting (200 req/min)
    - [ ] IP 기반 제한
    - [ ] 부하 분산
  - [ ] 모니터링 및 메트릭:
    - [ ] 캐시 히트율
    - [ ] 응답 시간 측정
    - [ ] Gateway 가용성 추적

### 10.3 w3ui (UI Components Library)
- [ ] **31_W3ui_Components_Analysis.md** 작성
  - [ ] w3ui 개요 및 철학
  - [ ] Headless 컴포넌트 설계:
    - [ ] Type-safe 아키텍처
    - [ ] 프레임워크 독립적 설계
  - [ ] 패키지 구조:
    - [ ] `packages/` 디렉토리 분석
    - [ ] React 컴포넌트 패키지
    - [ ] Vanilla JS 모듈
  - [ ] 프레임워크별 구현:
    - [ ] React 컴포넌트
    - [ ] Solid 컴포넌트
    - [ ] Vue 컴포넌트
  - [ ] 핵심 컴포넌트 분석:
    - [ ] **인증 컴포넌트**:
      - [ ] 이메일 인증 플로우
      - [ ] 개인키 생성 및 관리
      - [ ] 세션 관리
    - [ ] **업로드 컴포넌트**:
      - [ ] 단일 파일 업로드
      - [ ] 디렉토리 업로드
      - [ ] 드래그 앤 드롭
      - [ ] 진행률 표시
    - [ ] **Space 관리 컴포넌트**:
      - [ ] Space 생성/선택
      - [ ] Space 공유
    - [ ] **업로드 리스트 컴포넌트**:
      - [ ] 업로드 히스토리
      - [ ] 필터링 및 검색
  - [ ] 상태 관리:
    - [ ] 로컬 상태 관리 패턴
    - [ ] 컨텍스트 활용
  - [ ] 예시 애플리케이션:
    - [ ] Sign up/Sign in 예시
    - [ ] Simple File Upload 예시
    - [ ] Advanced Upload 예시
  - [ ] 통합 가이드:
    - [ ] 프로젝트에 추가하기
    - [ ] 커스터마이징
    - [ ] 스타일링
  - [ ] TypeScript 타입 정의
  - [ ] 문서화 웹사이트 (w3ui-website)

### 10.4 dag.w3s.link (Trustless Gateway)
- [ ] **32_Trustless_Gateway_Analysis.md** 작성
  - [ ] Trustless Gateway 개념
  - [ ] IPFS Trustless Gateway 스펙:
    - [ ] 스펙 문서 분석
    - [ ] 표준 준수 사항
  - [ ] dag.w3s.link 구현:
    - [ ] Graph API 제공
    - [ ] 레포지토리 구조
    - [ ] 핵심 기능 구현
  - [ ] CAR 요청 처리:
    - [ ] `dag-scope` 파라미터 (block/entity/all)
    - [ ] DAG 형태 정의
    - [ ] 포함될 블록 결정
  - [ ] `dups` 파라미터:
    - [ ] 중복 블록 처리
    - [ ] Deterministic 블록 순서
  - [ ] 검증 가능한 콘텐츠 접근:
    - [ ] Trustless 검증 메커니즘
    - [ ] 크립토그래픽 증명
  - [ ] w3s.link와의 차이점:
    - [ ] Graph API 전용 vs 범용 gateway
    - [ ] 사용 사례 비교
  - [ ] 성능 고려사항
  - [ ] 보안 및 신뢰성

### 10.5 Real-World Applications & Integration Examples
- [ ] **33_Real_World_Applications.md** 작성
  - [ ] 게임 산업:
    - [ ] **Unreal Engine 플러그인**:
      - [ ] Progressive game install
      - [ ] 게임 바이너리 배포
      - [ ] Content-addressed storage 활용
      - [ ] 사용자 소유 게임 에셋
      - [ ] 구현 아키텍처
    - [ ] 게임 에셋 관리
    - [ ] Multiplayer 데이터 동기화
  - [ ] AI & Machine Learning:
    - [ ] **elizaOS 통합**:
      - [ ] AI 에이전트 persistent memory
      - [ ] 분산 웹에서 데이터 저장/공유
      - [ ] 검증 가능한 AI 메모리
      - [ ] 통합 아키텍처 및 API
    - [ ] 모델 저장 및 버전 관리
    - [ ] 학습 데이터셋 아카이빙
  - [ ] NFT & Digital Assets:
    - [ ] NFT 메타데이터 저장:
      - [ ] JSON 메타데이터 구조
      - [ ] 이미지 및 미디어 저장
      - [ ] Immutable 참조
    - [ ] **Courtyard** (Pokémon 카드):
      - [ ] 물리적 수집품 토큰화
      - [ ] 검증 및 저장 프로세스
    - [ ] 디지털 아트 저장
    - [ ] 로열티 및 라이선싱
  - [ ] Supply Chain & Logistics:
    - [ ] 제품 추적 및 검증
    - [ ] 원산지 증명
    - [ ] 메타데이터 저장
    - [ ] 럭셔리 브랜드 활용 (Louis Vuitton, Gucci)
  - [ ] 분산 웹 호스팅:
    - [ ] 정적 웹사이트 배포
    - [ ] SPA (Single Page App) 호스팅
    - [ ] CDN 활용
    - [ ] 도메인 연결
  - [ ] 데이터 아카이빙:
    - [ ] 장기 데이터 보존
    - [ ] 규제 준수 (compliance)
    - [ ] 백업 전략
  - [ ] 협업 및 공유:
    - [ ] 팀 간 파일 공유
    - [ ] 버전 관리
    - [ ] 접근 권한 제어
  - [ ] CI/CD 통합:
    - [ ] GitHub Actions (add-to-web3)
    - [ ] 빌드 아티팩트 저장
    - [ ] 배포 자동화
  - [ ] 각 사례별 코드 예시 및 구현 패턴

### 10.6 Integration Patterns & Best Practices
- [ ] **34_Integration_Patterns.md** 작성
  - [ ] Frontend 통합 패턴:
    - [ ] React 앱 통합
    - [ ] Vue 앱 통합
    - [ ] Vanilla JS 통합
  - [ ] Backend 통합 패턴:
    - [ ] Node.js 서버
    - [ ] Express 미들웨어
    - [ ] GraphQL 서버
  - [ ] 모바일 앱 통합:
    - [ ] React Native
    - [ ] 하이브리드 앱
  - [ ] 서버리스 함수 통합:
    - [ ] AWS Lambda
    - [ ] Cloudflare Workers
    - [ ] Vercel Functions
  - [ ] 베스트 프랙티스:
    - [ ] 에러 처리
    - [ ] 재시도 로직
    - [ ] 진행률 추적
    - [ ] 메모리 관리
    - [ ] 성능 최적화
  - [ ] Anti-patterns 및 주의사항

---

## 🔄 지속적인 업데이트

- [ ] 레포지토리 변경사항 모니터링
- [ ] 새로운 기능 문서화
- [ ] 버그 픽스 반영
- [ ] 커뮤니티 피드백 통합
- [ ] 문서 품질 개선
- [ ] 활용 사례 추가 및 업데이트

---

## 📝 작성 원칙 및 가이드라인

### 문서 작성 기준
1. **명확성**: 기술적으로 정확하고 이해하기 쉽게
2. **완전성**: 코드 예시, 함수명, 파라미터까지 상세히
3. **구조화**: 논리적 흐름과 계층 구조
4. **실용성**: 실제 사용 가능한 예시 포함
5. **최신성**: 2025년 기준 최신 정보

### 코드 레퍼런스 형식
```
레포지토리/파일경로:라인번호 - 함수명/클래스명
예: upload-service/packages/upload-client/src/upload.js:142 - async function uploadFile(file, options)
```

### 다이어그램 형식
- Mermaid 문법 사용
- 복잡도에 따라 여러 레벨로 분리
- 범례 및 설명 포함

### 예시 코드 형식
```javascript
// 의도: 파일을 Space에 업로드하고 CID를 반환
// 위치: packages/client/src/client.js:89
async function uploadFile(file, options = {}) {
  // 1. Space 확인
  const space = await this.currentSpace()

  // 2. DAG 생성 및 CAR 샤딩
  const car = await encodeFile(file)

  // 3. Blob 저장
  await this.capability.invoke('blob/add', car)

  // 4. Upload 등록
  const cid = await this.capability.invoke('upload/add', {
    root: car.cid,
    shards: [car.cid]
  })

  return cid
}
```

---

## 🎯 우선순위

### P0 (최우선)
1. Phase 1: 핵심 개념 및 생태계 개요 (00-03)
2. SPECIFICATION.md ✅
3. TODO.md ✅

### P1 (높음)
1. Phase 2: 컴포넌트별 상세 구현 (04-07)
2. Phase 3: 구현 세부사항 (08-11)
3. Phase 10: Applications & Use Cases Deep Dive (29-34) - **NEW**

### P2 (중간)
1. Phase 4: 실전 가이드 (12-14)
2. Phase 6: 심층 코드 분석 (20-24)

### P3 (낮음)
1. Phase 5: 고급 주제 (15-19)
2. Phase 7: 최종 정리 (25-28)
3. Phase 8-9: 다이어그램 및 예시

---

## 📊 진행 상황 트래킹

- **전체 진행률**: 2 / 100+ 작업 (약 2%)
- **완료된 Phase**: 0 / 10
- **진행 중인 Phase**: Phase 1 준비
- **새로 추가된 문서**: 6개 (Phase 10)
  - Console Application Analysis
  - w3link Gateway Analysis
  - w3ui Components Analysis
  - Trustless Gateway Analysis
  - Real-World Applications
  - Integration Patterns
- **예상 완료일**: TBD
- **마지막 업데이트**: 2025-11-14

---

## 🤝 기여 및 피드백

이 문서화 프로젝트는 지속적으로 업데이트됩니다.
- 오류 발견 시 수정
- 새로운 인사이트 추가
- 코드 변경사항 반영
- 커뮤니티 피드백 통합

---

## 📌 다음 단계

1. ✅ SPECIFICATION.md 작성 완료
2. ✅ TODO.md 작성 완료
3. ⏭️ **00_Storacha_Ecosystem_Overview.md** 작성 시작
4. ⏭️ 주요 레포지토리 코드 클론 및 분석 시작

---

*이 TODO는 프로젝트 진행에 따라 지속적으로 업데이트됩니다.*
