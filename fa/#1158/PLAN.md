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
| 계약목록 테이블 (목데이터) | 완료 |
| 배지 색상: 운용 green / 그 외 yellow | 완료 |
| 계약만료일 남은기간 포맷 (`formatRemainingPeriod`) | 완료 |
| 빈 화면 EmptyView 처리 | 완료 |
| API 응답 스펙 적용 | 완료 |

---

## 코드 구조

| 항목 | 현황 |
|------|------|
| 테이블 컴포넌트 | `src/components/ui/DataTable/index.tsx` (TanStack React Table v8) |
| 페이지 패턴 참고 | `src/features/customer/CustomerList.tsx` (고객목록) |
| 구현 파일 | `src/features/customer/contracts/CustomersContracts.tsx` |
| 상태 표시 함수 | `QbContract.getQbContractStatus` (`packages/shared-consts`) |
| 계좌유형 변환 | `QbContract.getAccountTypeName` (`packages/shared-consts`) |
| 남은기간 포맷 | `formatRemainingPeriod` (`src/utils/formatters.ts`) |

---

## 수정/생성 파일

| 파일 | 변경 내용 |
|------|-----------|
| `src/features/customer/contracts/CustomersContracts.tsx` | 계약목록 테이블 구현 (목데이터 포함, API 연동 시 제거) |
| `src/features/investment/investment-propose/QbPropose/Board/ContractItem.tsx` | 수익금액/수익률 표시 정책 적용, 계약만료일 포맷 통일, 공통 포맷 함수로 교체 |
| `src/utils/formatters.ts` | 아래 함수 추가 |

### formatters.ts 추가 함수

| 함수 | 설명 | 유사 기존 함수 |
|------|------|--------------|
| `formatRemainingPeriod(endDate)` | 계약만료일 남은기간 (일/개월/년+개월) | 없음 |
| `formatContractDate(value)` | YYYYMMDD → YYYY-MM-DD, 빈값 `-` | `dateFormatter` — 유사하나 빈값 시 `''` 반환으로 다름 |
| `formatEvalDateTime(value)` | YYYYMMDDHHmmss → YYYY-MM-DD HH:mm:ss, 빈값 `-` | 없음 |
| `formatContractAmount(value)` | Math.ceil + 콤마 (계약 금액용) | 없음 |
| `formatProfitRate(value)` | 소수 → 퍼센트 문자열 소수점 2자리 | 없음 |

> ⚠️ formatters.ts에 함수 추가 시 유사 기존 함수 먼저 확인. 빈값 처리(빈 문자열 vs `-`)가 다르면 별도 함수로 분리.

---

## API 응답 구조

백엔드 API 응답 스펙 확정. 컴포넌트 내 `ContractListItem` 인터페이스로 정의.

```typescript
interface ContractListItem {
  betterId: number;              // Better 고객 ID
  customer: {
    id: number;                  // FA 고객 ID
    userName: string;            // 고객 이름
  };
  contract: {
    ctrNo: string;               // 계약번호
    actNo: string;               // 계좌관리번호
    contractType: string;        // 계약유형 코드
    contractTypeName: string;    // "자문" | "일임"
    progressStateCode: string;   // 계약 진행 상태
    contractStatus: string;      // 운용 상태 코드
    contractStatusName: string;
    securityName: string;        // 증권사명
    accountType: string;         // 계좌유형 코드 (IRP/ISA/PSA/BA)
    accountNo: string;           // 계좌번호 (마스킹)
    contractDate: string;        // YYYYMMDD
    contractPeriod: number;      // 월 단위
    contractEndDate: string;     // YYYYMMDD
    minAmount: number;           // 최소 투자금액
    feeType: string;             // 수수료 유형 코드
    feeTypeName: string;         // "정률형" | "성과형"
    feeTypePeriod: string;       // 수수료 출금 주기 코드
    feeRate: number;             // 수수료율 (0.01 = 1%)
    feeWithdrawalDate: string;   // YYYYMMDD
    portfolioName: string;
    portfolioType: string;       // "ETF" | "STOCK" | "ETC"
    algoId: string;
    portfolioNo: string;
    withdrawalAccountNumber: string;   // 출금 계좌 (마스킹)
    withdrawalAccountType: string;     // 출금 기관 코드
    withdrawalAccountTypeName: string; // 출금 기관명
    evaluation: {
      navAmount: number;           // 평가금액
      profitAmount: number;        // 수익금액 (음수 가능)
      profitRate: number;          // 수익률 (소수, 0.05 = 5%)
      investedPrincipal: number;   // 투자원금 ⚠️ 기존 principalAmount에서 변경
      evalDttm: string;            // YYYYMMDDHHmmss (빈 문자열 가능)
    };
    orderStatus: OrderStatusItem | null;
  };
}
```

### 기존 타입과 차이점

| 항목 | 기존 (`QbContract.ContractInfo`) | 새 API 응답 |
|------|----------------------------------|-------------|
| 최상위 구조 | `customerName`, `customerId`, `contractInfo` | `betterId`, `customer`, `contract` |
| evaluation 위치 | `contractInfo.evaluation` | `contract.evaluation` |
| orderStatus 위치 | `contractInfo.orderStatus` | `contract.orderStatus` |
| 투자원금 필드명 | `principalAmount` | **`investedPrincipal`** |
| 계약기간 타입 | `string` ("12") | `number` (12) |
| 수수료율 타입 | `string` ("0.012") | `number` (0.012) |
| 계좌유형 | 코드 (IRP/ISA/PSA/BA) | 코드 동일 → `getAccountTypeName`으로 변환 |

> ⚠️ **`principalAmount` → `investedPrincipal` 변경 주의**
> 현재 `packages/shared-consts`의 `QbContract.Evaluation` 타입은 아직 `principalAmount`를 사용.
> 계약목록 컴포넌트는 API 응답 전용 로컬 인터페이스를 사용하므로 직접적 충돌은 없으나,
> 향후 shared-consts 타입 통합 시 이 변경 반영 필요.

### `-` 표시 기준

- 값이 `null` 또는 `undefined`일 때만 `-` (gray) 표시
- `0`은 유효한 값 → 수익금액 `0원`, 수익률 `0.00%`로 정상 표시
- 모든 계약 관련 화면(계약목록, 계약현황, Overview)에서 동일 기준 적용

### 테이블 컬럼 → 필드 매핑

| 컬럼명 | 데이터 필드 |
|--------|------------|
| 고객명 | `customer.userName` |
| 평가일시 | `contract.evaluation.evalDttm` |
| 계약상태 | `getQbContractStatus(contract.contractStatus, contract.orderStatus)` |
| 투자원금 | `contract.evaluation.investedPrincipal` |
| 평가금액 | `contract.evaluation.navAmount` |
| 수익금액 | `contract.evaluation.profitAmount` |
| 수익률 | `contract.evaluation.profitRate` |
| 증권사 | `contract.securityName` |
| 계좌유형 | `getAccountTypeName(contract.accountType)` |
| 계약유형 | `contract.contractTypeName` |
| 포트폴리오 | `contract.portfolioName` |
| 계약시작일 | `contract.contractDate` |
| 계약만료일(남은일수) | `contract.contractEndDate` + `formatRemainingPeriod` |
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

> 일임 계약에서 '승인 대기 중' → '매매 대기 중'으로 오버라이드

배지 색상: `getStatusLabel(contract) === '운용 중'` → `green`, 그 외 → `yellow`

> `getQbContractStatus`는 2210이더라도 orderStatus 플래그(approveFlag/processFlag)에 따라
> '승인 대기 중' / '매매 대기 중' / '운용 중'을 다르게 반환하므로, 코드가 아닌 라벨로 판단.

---

## 계약만료일 표시

- 헤더: `계약만료일(남은일수)`
- 만료 전: `YYYY-MM-DD (NNN일)` — 남은 일수
- 만료 후: `YYYY-MM-DD (만료)` — 해지 완료 전까지만 노출
- 30일 이하: 일 단위, 빨강(`text-red-default`)
- 30일 초과 ~ 1년 미만: 개월 단위, 회색(`text-gray-500`)
- 1년 이상: 년+개월, 회색(`text-gray-500`)
- 개월 계산: `dayjs.diff(today, 'month')` 달력 기준
- 유틸 함수: `formatRemainingPeriod(endDate)` (`src/utils/formatters.ts`)

---

## 빈 화면 처리

데이터가 없을 때 테이블 대신 `EmptyView` 컴포넌트 표시.

- `data.length === 0`일 때 DataTable 자체를 렌더링하지 않고 EmptyView로 대체

---

## 금액 표시 규칙

금액 데이터(투자원금, 평가금액, 수익금액)가 소숫점 포함 시 **절상(Math.ceil)** 처리.

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|-----------|
| 2026-04-13 | 최초 작성 |
| 2026-04-14 | 계약만료일 표시, 빈 화면 처리, 배지 색상 변경 |
| 2026-04-14 | API 응답 스펙 적용 — 데이터 구조 전면 교체, `principalAmount` → `investedPrincipal` 변경 |
