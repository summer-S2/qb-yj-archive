# 계약목록 구현 계획 — #1158

> 기획/정책: `SPEC.md` 참조
> 프론트엔드(`apps/fe-react-app`) 변경사항만 정리

---

## 브랜치

`feature/#1158-contracts-demos`

---

## 코드 구조

| 항목 | 현황 |
|------|------|
| 테이블 컴포넌트 | `src/components/ui/DataTable/index.tsx` (TanStack React Table v8) |
| 페이지 패턴 참고 | `src/features/customer/CustomerList.tsx` (고객목록) |
| 계약 타입 정의 | `packages/shared-consts/src/types/quarterback/contract.ts` |
| 상태 표시 함수 | `getQbContractStatus` (같은 파일) |

---

## 수정/생성 파일

| 파일 | 변경 내용 |
|------|-----------|
| `src/features/customer/contracts/CustomersContracts.tsx` | 계약목록 테이블 + 요약카드 + 상태필터 |
| `src/features/customer/contracts/contractUtils.ts` | 상태 라벨, 배지 색상, 필터 분류 유틸 |
| `src/features/customer/contracts/SummaryCard.tsx` | 요약 카드 컴포넌트 (총계약/운용중/대기중/만료임박) |
| `src/features/customer/contracts/ContractDetailPanel.tsx` | 행 클릭 시 사이드패널 — 성과 요약 + 계약 정보 + 수수료 정보 |
| `src/api/customer.ts` | `useAdvisorCustomersContracts` 훅 추가 |
| `src/pages/dashboard/contracts/ContractsPage.tsx` | 라우트 페이지 |
| `src/config/gnb.ts` | GNB 메뉴 "계약목록" 추가 |
| `src/routes/elements.tsx` | 라우트 엘리먼트 등록 |

---

## 데이터 구조

API 연동 완료. `QbContract.ContractListItem` 사용.

```typescript
// GET /main/advisor/customers/investments/contracts
// 응답: ContractListItem[]
interface ContractListItem {
  betterId: number;
  customer: {
    id: number;
    userName: string;
  };
  contract: ContractListContract;
}
```

### ContractListContract 구조

```typescript
interface ContractListContract {
  ctrNo: string;
  actNo: string;
  contractType: string;
  contractTypeName: string;        // 자문/일임
  progressStateCode: string;
  contractStatus: string;          // 상태코드
  contractStatusName: string;
  securityName: string;            // 증권사
  accountType: string;             // IRP/ISA/PSA/BA
  accountNo: string;               // 운용계좌
  contractDate: string;            // 계약시작일 (YYYYMMDD)
  contractPeriod: number;          // 계약기간 (개월)
  contractEndDate: string;         // 계약만료일 (YYYYMMDD)
  minAmount: number;
  feeType: string;
  feeTypeName: string;             // 수수료유형 (정률형/성과형)
  feeTypePeriod: string;           // 수수료 출금 주기
  feeRate: number;                 // 수수료율 (소수, 0.02 = 2%)
  feeWithdrawalDate: string;       // 수수료 출금 예정일
  portfolioName: string;           // 포트폴리오명
  portfolioType: string;
  algoId: string;
  portfolioNo: string;
  withdrawalAccountNumber: string; // 수수료 출금 계좌
  withdrawalAccountType: string;
  withdrawalAccountTypeName: string; // 수수료 출금 기관
  evaluation: ContractListEvaluation;
  orderStatus: OrderStatus | null;
}

interface ContractListEvaluation {
  navAmount: number;           // 평가금액
  profitAmount: number;        // 수익금액
  profitRate: number;          // 수익률 (소수)
  investedPrincipal: number;   // 투자원금
  evalDttm: string;            // 평가일시 (YYYYMMDDhhmmss)
}
```

---

## 화면 구성

### 요약 카드

| 카드 | 값 | tone |
|------|----|------|
| 총 계약 | 전체 건수 | default |
| 운용중 | 운용 중 상태 건수 | default |
| 대기중 | 대기 상태 건수 | default |
| 만료 임박 (30일 이내) | 만료일 30일 이내 건수 | warn (빨강) |

### 상태 필터

| 필터 | 분류 기준 |
|------|-----------|
| 전체 | 모든 계약 |
| 운용중 | `getStatusLabel` === '운용 중' |
| 대기중 | 라벨에 '대기' 포함 |
| 해지처리중 | 라벨에 '수수료', '해지', '출금' 포함 |
| 만료 임박 | 만료일까지 0~30일 |

### 테이블 컬럼 (10개)

| 컬럼명 | 데이터 필드 | 정렬 | 비고 |
|--------|------------|------|------|
| 고객명 | `customer.userName` | left | 고정 컬럼 (왼쪽 pin) |
| 계약상태 | `getStatusLabel(contract)` | left | Chip 배지 |
| 계좌유형 | `contract.accountType` | left | `getAccountTypeName` 변환 |
| 계약유형 | `contract.contractTypeName` | left | |
| 포트폴리오 | `contract.portfolioName` | left | |
| 계약만료일 | `contract.contractEndDate` | left | 남은일수 표시, 30일 이내 빨강 |
| 평가금액 | `evaluation.navAmount` | right | `formatContractAmount` |
| 수익률 | `evaluation.profitRate` | right | 양수=빨강, 음수=파랑 |
| 수수료출금예정일 | `contract.feeWithdrawalDate` | left | 남은일수 표시 |

### 행 클릭 → 사이드패널 (ContractDetailPanel)

사이드패널에 다음 정보 표시:

**헤더**: 고객명 + 계약상태 배지 + 포트폴리오명

**성과 요약** (회색 카드):
- 평가금액, 투자원금, 수익금액, 수익률
- 평가일시

**계약 정보**:
- 계약유형, 계좌유형, 증권사, 운용계좌
- 계약기간, 계약시작일, 계약만료일, 남은 계약 기간

**수수료 정보**:
- 수수료 유형, 수수료율, 출금 주기, 출금 기관, 출금 계좌, 출금 예정일

---

## 상태 표시 로직

`getQbContractStatus` + 일임 오버라이드 (`contractUtils.ts`)

| 코드 | 표시명 | 조건 |
|------|--------|------|
| 2200 | 입금 대기 중 | - |
| 2202 | 승인 대기 중 | - |
| 2210 | 승인 대기 중 | approveFlag=N, processFlag=N |
| 2210 | 매매 대기 중 | approveFlag=Y, processFlag=N |
| 2210 | 운용 중 | approveFlag=Y, processFlag=Y |
| 2222 | 매매 진행 중 | - |
| 2231 | 수수료 계산 중 | - |
| 2234 | 출금 대기 중 | - |
| 2235 | 해지 대기 중 | - |

> 일임 계약에서 '승인 대기 중' → '매매 대기 중'으로 오버라이드

### 배지 색상

| 상태 | 배지 색상 | 비고 |
| --- | --- | --- |
| 운용 중 | 초록색 (green) | |
| 해지 진행 중 (수수료 계산 중, 출금 대기 중, 해지 대기 중) | 빨간색 (red) | `contractStatus`: 2231, 2234, 2235 |
| 그 외 (입금 대기 중, 승인 대기 중, 매매 대기 중 등) | 노란색 (yellow) | |

---

## 계약만료일 표시

- 만료 전: `YYYY-MM-DD (NNN일)` — 남은 일수
- 만료 후: `YYYY-MM-DD (만료)` — 해지 완료 전까지만 노출
- 30일 이하: 일 단위 표시 (예: `1일`, `28일`), 빨강(`text-red-default`)
- 30일 초과 ~ 1년 미만: 개월 단위 표시 (예: `2개월`, `11개월`), 회색(`text-gray-500`)
- 1년 이상: 년+개월 표시 (예: `1년`, `1년 3개월`), 회색(`text-gray-500`)
- 개월 계산은 `dayjs.diff(today, 'month')` 달력 기준 (완전한 월만 카운트, 미만은 버림)
- 유틸 함수: `formatRemainingPeriod(endDate)` (`src/utils/formatters.ts`)

> '만료' 표시 노출 시점: 계약 만료일 도래 후 해지 완료 처리 전까지만.
> 해지 완료(2290) 시 목록에서 미노출. 해지 처리 소요: KBS D+4, SSI/KIS D+1~D+3 (`bw-is/해지.md` 참고)

---

## 수수료율 표시

- API에서 소수로 내려옴 (예: `0.02` = 2%)
- 사이드패널에서 `(feeRate * 100).toFixed(2)%` 로 변환 표시

---

## 금액 표시 규칙

투자원금, 평가금액, 수익금액이 소숫점 포함돼서 내려오는 경우 `formatContractAmount` (절상 Math.ceil) 처리.

---

## 빈 화면 처리

- 데이터 없을 때: `EmptyView` 컴포넌트 표시
- 필터 결과 없을 때: "조건에 맞는 계약이 없습니다." 메시지
- 테이블 대신 EmptyView로 대체 (가로 스크롤 문제 방지)

---

## v2 (미구현)

- 페이지네이션 (서버 사이드)
- 검색 (고객명·계약번호·포트폴리오명)
- 정렬 (서버 사이드)

---

## 변경 이력

| 날짜 | 변경 내용 | 관련 파일 |
|------|-----------|-----------|
| 2026-04-13 | 최초 작성 | 전체 |
| 2026-04-14 | 계약만료일 헤더/표시 변경, 빈 화면 처리 추가, 배지 gray 통일 반영 | CustomersContracts.tsx |
| 2026-04-27 | 실제 코드 기준 전면 업데이트 — API 연동, 타입 변경(ContractInfo→ContractListContract), 컬럼 10개로 축소, 사이드패널/요약카드/상태필터 추가, 배지 색상 규칙 변경, 파일 분리 반영 | 전체 |
