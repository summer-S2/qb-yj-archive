# 계약목록 구현 계획 — #1158

> 기획/정책: `SPEC.md` 참조
> 프론트엔드(`apps/fe-react-app`) 변경사항만 정리

---

## 브랜치

`feature/#1158-contract-list`

---

## 완료된 작업

| 항목 | 상태 |
|------|------|
| 라우트 (`PATH_DASHBOARD.advisor.customers.contracts`) | 완료 |
| 페이지 (`src/pages/dashboard/contracts/ContractsPage.tsx`) | 완료 |
| GNB 메뉴 "계약관리" (`src/config/gnb.ts`) | 완료 |
| 라우트 엘리먼트 (`src/routes/elements.tsx`) | 완료 |

---

## 코드 구조

| 항목 | 현황 |
|------|------|
| 테이블 컴포넌트 | `src/components/ui/DataTable/index.tsx` (TanStack React Table v8) |
| 페이지 패턴 참고 | `src/features/customer/CustomerList.tsx` (고객목록) |
| 구현 대상 | `src/features/customer/contracts/CustomersContracts.tsx` (현재 stub) |
| 계약 타입 정의 | `packages/shared-consts/src/types/quarterback/contract.ts` |
| 상태 표시 함수 | `getQbContractStatus` (같은 파일) |
| 계약 컴포넌트 참고 | `src/features/investment/investment-propose/QbPropose/Board/PortfolioContractTab.tsx` |
| 상태 라벨 로직 참고 | `src/features/investment/investment-propose/QbPropose/Board/ContractItem.tsx` |

---

## 수정/생성 파일

| 파일 | 변경 내용 |
|------|-----------|
| `src/features/customer/contracts/CustomersContracts.tsx` | stub → 계약목록 테이블 구현 (목데이터 컴포넌트 하단에 포함, API 연동 시 제거) |

---

## 데이터 구조

API 미완성 → 목데이터로 작업. 기존 `QbContract.ContractInfo` 래퍼 타입 사용.

```typescript
// 계약목록 행 타입 — API 나오면 응답 타입으로 교체
interface ContractListItem {
  customerName: string;
  customerId: number;
  contractInfo: QbContract.ContractInfo;
}
```

### QbContract.ContractInfo 구조 (참고)

```typescript
{
  evaluation: {
    navAmount: number;          // 평가금액
    profitAmount: number;       // 수익금액
    profitRate: number;         // 수익률 (소수, 0.02 = 2%)
    principalAmount: number;    // 투자원금
    evalDttm: string;           // 평가일시 (YYYYMMDDhhmmss)
  }
  contract: {
    ctrNo: string;              // 계약번호
    contractStatus: string;     // 상태코드
    contractStatusName: string; // 상태명
    contractTypeName: string;   // 계약유형 (자문/일임)
    contractDate: string;       // 계약시작일 (YYYYMMDD)
    contractPeriod: string;     // 계약기간
    contractEndDate: string;    // 계약만료일 (YYYYMMDD)
    securityName: string;       // 증권사
    accountType: AccountType;   // 계좌유형 (IRP/ISA/PSA/BA)
    actNo: string;              // 운용계좌
    feeTypeName: string;        // 수수료유형 (정률형/성과형)
    feeRate: string;            // 수수료율
    feeTypePeriod: string;      // 수수료 출금 주기
    feeWithdrawalDate: string;  // 수수료 출금 예정일
    portfolioName: string;      // 포트폴리오명
    withdrawalAccountNumber: string;   // 수수료 출금 계좌
    withdrawalAccountTypeName: string; // 수수료 출금 기관
  }
  orderStatus: OrderStatus | null;
}
```

### 테이블 컬럼 → 필드 매핑

| 컬럼명 | 데이터 필드 |
|--------|------------|
| 고객명 | `customerName` |
| 평가일시 | `evaluation.evalDttm` |
| 계약상태 | `getQbContractStatus(contract.contractStatus, orderStatus)` |
| 투자원금 | `evaluation.principalAmount` |
| 평가금액 | `evaluation.navAmount` |
| 수익금액 | `evaluation.profitAmount` |
| 수익률 | `evaluation.profitRate` |
| 증권사 | `contract.securityName` |
| 계좌유형 | `contract.accountType` |
| 계약유형 | `contract.contractTypeName` |
| 포트폴리오 | `contract.portfolioName` |
| 계약시작일 | `contract.contractDate` |
| 계약만료일 | `contract.contractEndDate` |
| 계약기간 | `contract.contractPeriod` |
| 운용계좌 | `contract.actNo` |
| 수수료 유형 | `contract.feeTypeName` |
| 수수료율 | `contract.feeRate` |
| 수수료 출금 기관 | `contract.withdrawalAccountTypeName` |
| 수수료 출금 계좌 | `contract.withdrawalAccountNumber` |
| 수수료 출금 주기 | `contract.feeTypePeriod` |
| 수수료 출금 예정일 | `contract.feeWithdrawalDate` |

---

## 상태 표시 로직

`getQbContractStatus` + 일임 오버라이드 (ContractItem.tsx 패턴 참고)

| 코드 | 표시명 | 조건 |
|------|--------|------|
| 2200 | 입금 대기 중 | - |
| 2202 | 승인 대기 중 | - |
| 2210 | 승인 대기 중 | approveFlag=N, processFlag=N |
| 2210 | 매매 대기 중 | approveFlag=Y, processFlag=N |
| 2210 | 운용 중 | approveFlag=Y, processFlag=Y |
| 2222 | 매매 진행 중 | - |
| 2231 | 해지 수수료 계산 중 | - |
| 2234 | 출금 대기 중 | - |
| 2235 | 해지 대기 중 | - |

> 일임 계약에서 '승인 대기 중' → '매매 대기 중'으로 오버라이드

배지 색상: '운용 중' → green, 나머지 → yellow (ContractItem.tsx 패턴)

---

## 금액 표시 규칙

쿼터백 계약 관련 금액 데이터(투자원금, 평가금액, 수익금액)가 소숫점 포함돼서 내려오는 경우 **절상(Math.ceil)** 처리.

---

## v2 (API 기능 추가 후)

- 페이지네이션 (서버 사이드)
- 검색 (고객명·계약번호·포트폴리오명)
- 정렬 (서버 사이드)

---

## 변경 이력

| 날짜 | 변경 내용 | 관련 파일 |
|------|-----------|-----------|
| 2026-04-13 | 최초 작성 | 전체 |
