# 투자제안 알림톡 — 상품변경 지원 구현 계획

**SPEC**: [SPEC.md](./SPEC.md)

---

## 1. 변경 계획

| 작업 | 경로 | 유형 | 설명 |
|------|------|------|------|
| 탭 구조 변경 | `QbPropose/index.tsx` | 수정 | 3탭 → 2탭 (send 삭제, contract → '계약 관리') |
| PortfolioSendTab 삭제 | `QbPropose/Board/PortfolioSendTab.tsx` | 삭제 | 발송 이력은 계약 상세 API로 대체 |
| PortfolioContractTab 재작성 | `QbPropose/Board/PortfolioContractTab.tsx` | 수정 | 가이드 문구 + 4그룹 + 미결/기계약 카드 + 발송 이력 |
| 미결계약 카드 컴포넌트 | `QbPropose/Board/PendingContractItem.tsx` | 신규 | [미결] 배지, 포트폴리오명, 생성일, 진행단계, 상품변경 버튼 |
| 기계약 카드 수정 | `QbPropose/Board/ContractItem.tsx` | 수정 | 상품변경 제안 버튼 추가 |
| 사이드모달 확장 | `QbPropose/Board/PortfolioModal.tsx` | 수정 | 신규/상품변경 모드 분기, 동일 MP 비활성화 |
| 오버뷰 계약 영역 | `src/features/customer/overview/cards/Contract.tsx` | 수정 | 미결 표시, 네비게이션 통일 |
| 타입 정의 | `packages/shared-consts/src/types/quarterback/contract.ts` | 수정 | 미결계약 타입 추가 |
| API 훅 확장 | `src/api/customer.ts` | 수정 | 계약 조회 응답에 미결계약 포함 |
| 정책서 반영 | `policy/bw-fa/` | 완료 | 투자설계-투자제안.md, 오버뷰.md |

## 2. 할 일 목록

- [x] SPEC.md 작성 및 심층 인터뷰
- [x] 정책서 반영 (투자설계-투자제안.md, 오버뷰.md)
- [x] PLAN.md 작성 및 확정
- [ ] 백엔드 API 스펙 협의 (미결계약 응답 구조, 상품변경 발송 API)
- [ ] 타입 정의 확장 (`PendingContractInfo`)
- [ ] 탭 구조 변경 (send 삭제, `?tab=send` fallback 처리)
- [ ] PortfolioSendTab 삭제
- [ ] PortfolioContractTab 전면 재작성
- [ ] PendingContractItem 컴포넌트 신규 작성
- [ ] ContractItem에 상품변경 버튼 추가
- [ ] PortfolioModal 확장 (신규/상품변경 모드)
- [ ] 오버뷰 계약 영역 수정
- [ ] 테스트 및 검증
- [ ] PR 오픈 및 리뷰 요청

## 3. 구현 계획

### 3.1 타입 정의 확장

**파일**: `packages/shared-consts/src/types/quarterback/contract.ts`

```typescript
interface PendingContractInfo {
  portfolioName: string;              // FA 화면·알림톡 노출용 MP명 (§4.10 규칙 적용)
  createdDate: string;                // YYYY-MM-DD
  progressStateCode: string;          // 진행단계 (미결계약 진행 단계 코드)
  accountType: AccountType;
}
```

- 기존 계약 조회 API 응답에 미결계약이 포함되는 형태로 변경 예정
- 정확한 응답 구조는 백엔드 협의 후 확정

---

### 3.2 탭 구조 변경

**파일**: `src/features/investment/investment-propose/QbPropose/index.tsx`

**현재**:
```typescript
const TABS = [
  { id: 0, title: '포트폴리오 제안', value: 'propose' },
  { id: 1, title: '포트폴리오 실행', value: 'send' },
  { id: 2, title: '계약 현황', value: 'contract' }
];
```

**변경**:
```typescript
const TABS = [
  { id: 0, title: '포트폴리오 제안', value: 'propose' },
  { id: 1, title: '계약 관리', value: 'contract' }
];
```

- `PortfolioSendTab` import 및 조건부 렌더링 제거
- `?tab=send` 진입 시 `?tab=contract`로 fallback (기존 링크 호환):
  ```typescript
  const tab = searchParams.get('tab');
  const resolvedTab = tab === 'send' ? 'contract' : tab;
  ```

---

### 3.3 PortfolioContractTab 전면 재작성

**파일**: `src/features/investment/investment-propose/QbPropose/Board/PortfolioContractTab.tsx`

**사용 API**:
- `useGetCustomerQbContract` — 기계약 + 미결계약 + 발송 이력 (확장)

**렌더링 구조**:
```
<가이드 문구>
  "베러웰스 앱 내 KB증권 연계 쿼터백 투자 서비스에 가입한 고객의 계약 정보가 표시됩니다."
  "각 증권사 별 상세 제안 방법은 가이드북을 참고해주세요." (링크)

<AccountTypeGroup accountType="IRP">   ← 4개 항상 렌더
  <GroupHeader>
    "IRP · 자문"  [신규 제안 버튼 (조건부)]
  </GroupHeader>
  <SendHistory />                       ← 있을 때만
  <PendingContractItem />               ← 있을 때만
  <ContractItem showDetail />           ← 있을 때만
  <EmptyState />                        ← 계약·미결·발송이력 모두 없을 때
</AccountTypeGroup>
```

**신규 제안 버튼 노출 로직**:
- IRP/ISA: `contracts.length === 0 && !pendingContract`
- PSA/BA: `!pendingContract` (기계약 있어도 노출)

**그룹 순서** (기존 `accountTypeOrder` 재사용): IRP → ISA → PSA → BA

---

### 3.4 미결계약 카드

**파일**: `src/features/investment/investment-propose/QbPropose/Board/PendingContractItem.tsx` (신규)

**표시 필드**:
| 필드 | 소스 |
|------|------|
| `[미결]` 배지 | 고정 |
| 포트폴리오명 | `portfolioName` (§4.10 규칙 적용 — FA 화면은 항상 `portfolioName` 사용) |
| 미결 생성일 | `createdDate` (YYYY-MM-DD) |
| 진행단계 배지 | `progressStateCode` |
| 상품변경 제안 버튼 | 항상 노출 |

**버튼 클릭 시**: `PortfolioModal` 오픈
- `mode: 'change'`
- `source: 'pending'`
- `currentPortfolioName: portfolioName`

---

### 3.5 기계약 카드 수정

**파일**: `src/features/investment/investment-propose/QbPropose/Board/ContractItem.tsx`

**추가 사항**:
- `showDetail=true` 시 카드 하단에 **"상품변경 제안"** 버튼 추가
- 새 prop: `onChangePropose?: () => void`
- 해지진행중 (`contractStatus`가 해지 관련) → 버튼 미렌더
- 해지완료 → 상위에서 필터링하여 카드 자체 미렌더

**버튼 클릭 시**: 상위에서 전달받은 `onChangePropose` 콜백 실행
- PortfolioModal 오픈 (mode: 'change', source: 'existing')

---

### 3.6 포트폴리오 선택 사이드모달 확장

**파일**: `src/features/investment/investment-propose/QbPropose/Board/PortfolioModal.tsx`

**현재**: 신규 발송만 지원 (포트폴리오 선택 → 발송)
**변경**: 신규 + 상품변경 모두 지원

**Props 추가**:
```typescript
mode: 'new' | 'change';
currentPortfolioNo?: string;   // 상품변경 시 현재 MP 코드
```

**상품변경 시 UI 변경**:
- 상단에 "현재 포트폴리오: {MP명}" 표시 — 기계약/미결계약 구분 없이 동일 문구
- CTA 비활성화 조건: `선택MP코드 === currentPortfolioNo`

**발송 로직**:
- 기존 `sendInvestStrategy` API 재사용 (계약 식별정보 파라미터 추가 — 백엔드 협의 필요)
- 성공 시: 사이드모달 유지 (닫지 않음) + `customer-qb-contract-${userId}` 쿼리 invalidate (계약 상세 재조회)
- 실패 시: 모달 유지, 재시도 가능

**포트폴리오 목록 필터링**: 기존에 `accountType + orgCode`로 필터링 중 → 그대로 유지

---

### 3.7 발송 이력 표시

**기존 위치**: `PortfolioSendTab`에서 각 계좌유형별 `sendDatetime`, `portfolioName` 표시
**변경 위치**: `PortfolioContractTab` 각 그룹 타이틀 바로 아래

**데이터**: 계약 상세 API 응답의 `proposalPortfolio` 필드 (계좌유형별)
- 표시: `최근 발송: {YYYY-MM-DD}  {포트폴리오명}`
- 이력 없는 accountType은 미표시

---

### 3.8 오버뷰 계약 영역 수정

**파일**: `src/features/customer/overview/cards/Contract.tsx`

**현재**:
- 기계약 있는 accountType만 렌더
- 타이틀: "계약 현황"
- 클릭: 계약 없으면 `?tab=send`, 있으면 `?tab=contract`

**변경**:
- 미결계약 있는 accountType도 렌더
- 미결계약 행: 포트폴리오명, 미결 생성일
- 표시 순서: 미결 → 기계약
- 타이틀: "계약 관리"
- 클릭: 항상 `?tab=contract`

---

## 4. 테스트 케이스

### 계약 관리 탭 — 미결계약 표시

| 구분 | 입력 / 조건 | 기대 결과 |
|------|------------|---------|
| 정상 | IRP에 미결계약 1건 존재 | IRP 그룹에 미결카드 표시, 신규 제안 버튼 미노출 |
| 정상 | 연금저축에 미결 + 기계약 동시 | 미결카드 위, 기계약카드 아래 순서. 신규 제안 버튼 미노출 |
| 정상 | 연금저축에 기계약만 (미결 없음) | 기계약카드 + 신규 제안 버튼 노출 |
| 엣지 | IRP에 계약·미결 모두 없음, 발송이력 있음 | 신규 제안 버튼 + 발송 이력 표시 |
| 엣지 | BA에 계약·미결·발송이력 모두 없음 | 신규 제안 버튼 + empty state 카드 ("아직 정보가 없습니다") |
| 엣지 | 미결계약 포트폴리오 미선택 상태 | FA 제안 MP명 표시 |
| 엣지 | 해지 진행중 기계약 | 카드 표시, 상품변경 버튼 숨김 |
| 엣지 | 해지 완료 기계약 | 카드 미노출 (백엔드에서 필터링) |

### 포트폴리오 선택 사이드모달

| 구분 | 입력 / 조건 | 기대 결과 |
|------|------------|---------|
| 정상 | 기계약 상품변경 → 다른 MP 선택 | 발송 버튼 활성화 |
| 정상 | 미결계약 상품변경 → 다른 MP 선택 | 발송 버튼 활성화 |
| 정상 | 신규 제안 → MP 선택 | 발송 버튼 활성화 |
| 엣지 | 상품변경 → 동일 MP 선택 | 발송 버튼 비활성화 |
| 엣지 | IRP에서 진입 → ISA용 MP 표시 | ISA용 MP 미표시 (계좌유형별 필터링) |

### 알림톡 발송

| 구분 | 입력 / 조건 | 기대 결과 |
|------|------------|---------|
| 정상 | 발송 성공 | 성공 피드백, 사이드모달 유지, 계약 상세 API 재조회 |
| 실패 | 발송 실패 | 실패 피드백, 모달 유지, 재시도 가능 |

### 오버뷰 계약 영역

| 구분 | 입력 / 조건 | 기대 결과 |
|------|------------|---------|
| 정상 | 미결계약만 있는 계좌유형 | 오버뷰에 해당 계좌유형 표시 |
| 정상 | 미결 + 기계약 동시 | 미결 행 위, 기계약 행 아래 |
| 정상 | 타이틀 클릭 | 투자제안 > 계약 관리 탭 이동 |
| 엣지 | 계약·미결 모두 없음 | 해당 계좌유형 미표시 |

## 5. 미결 사항

| # | 질문 | 담당 | 결론 |
|---|------|------|------|
| 1 | 미결계약 응답 구조 — 기존 ContractInfo[]에 포함? 별도 필드? | 백엔드 | 기존 계약 API에 미결 포함 예정 |
| 2 | 미결계약 식별 필드명 (contractId? pctId?) | 백엔드 | 협의 필요 |
| 3 | 상품변경 발송 API — 기존 sendInvestStrategy에 파라미터 추가? | 백엔드 | 협의 필요 |
| 4 | 다중계약(PSA/BA) 기계약 카드 정렬 순서 | PO | 미확정 |

## 6. 결정 사항 (Decision Log)

| 날짜 | 결정 내용 | 결정자 |
|------|---------|-------|
| 2026-04-20 | 계좌유형당 미결계약 최대 1개 제한 | PO |
| 2026-04-20 | D-N 만료일 표시 이번 scope 제외 | PO |
| 2026-04-20 | 발송 이력은 신규/상품변경 구분 없이 계좌유형당 1건 | PO |
| 2026-04-20 | 미결계약 삭제 시 실시간 반영 불필요 (다음 진입 시 반영) | PO |
| 2026-04-20 | 상품변경 버튼 진행단계 무관 항상 노출 (발송 차단은 백엔드) | PO |
| 2026-04-20 | 재시도 횟수 제한 없음 | PO |
| 2026-04-20 | MP 목록 계좌유형별 필터링 적용 | PO |
| 2026-04-20 | 미결계약 포트폴리오명: 고객 선택 전 FA 제안 MP명, 선택 후 고객 선택 MP명 | PO |
| 2026-04-21 | 발송 이력은 계약 상세 API 응답의 proposalPortfolio 필드 사용 (useGetLastInvestStrategyAll은 레거시 — 삭제 검토 필요) | 프론트 |
| 2026-04-22 | 발송 성공 시 사이드모달 유지 + 계약 상세 API 재조회 | PO |
| 2026-04-22 | 오버뷰 미결계약 표시에서 진행단계 제외 (포트폴리오명, 미결 생성일만 표시) | PO |
| 2026-04-22 | FA 화면은 portfolioName 필드 사용 (portfolioAltName은 쿼터백 앱용) | PO |
| 2026-04-21 | 미결계약은 기존 계약 조회 API에 포함되는 형태 | 백엔드 |
| 2026-04-21 | ?tab=send 접근 시 ?tab=contract로 fallback (기존 링크 호환) | 프론트 |
