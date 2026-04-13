# 투자제안 다중계좌 구현 계획 — #1137

> 기획/정책: `SPEC.md` 참조
> 프론트엔드(`apps/fe-react-app`) 변경사항만 정리

---

## 브랜치

`feature/#1137-portfolio`

---

## 수정/생성 파일

### 신규 생성

| 파일 | 내용 |
|------|------|
| `src/features/portfolio/consts.ts` | `AccountTypeEnum`, `getContractTypeByServiceCode` |

### 수정된 파일

| 파일 | 변경 내용 |
|------|-----------|
| `src/api/investment-propose.ts` | 단건 조회 → 다건 조회로 API/훅 변경 |
| `src/features/investment/investment-propose/QbPropose/Board/PortfolioSendTab.tsx` | IRP 버튼 → 계좌 타입별 리스트로 개편 |
| `src/features/investment/investment-propose/QbPropose/Board/FirmCard.tsx` | `caption` prop 제거 → `portfolioList: ReactNode` prop 추가 |
| `src/features/investment/investment-propose/QbPropose/Board/PortfolioModal.tsx` | 계좌/계약 타입 Chip 추가, 발송 후 쿼리 invalidate, 문구 변경 |
| `src/features/investment/investment-propose/QbPropose/Board/consts.ts` | `CONTRACT_LIST`, `getAdvisoryAccountType` 추가 |
| `src/features/investment/investment-propose/QbPropose/Propose/Portfolio/Detail.tsx` | `accountType` prop 추가 |
| `src/features/investment/investment-propose/QbPropose/Propose/Portfolio/index.tsx` | 계좌/계약 타입 Chip 추가 |
| `src/features/investment/investment-propose/tabs/Detail.tsx` | `accountType` prop 전달 |
| `src/features/portfolio/DetailContent.tsx` | `accountType` prop 추가, 계좌/계약 타입 Chip 추가 |
| `src/features/portfolio/DetailView.tsx` | `selectedAccountType` state 추가, `onAccountTypeChange` 연결 |
| `src/features/portfolio/detail/ListNavigation.tsx` | `onAccountTypeChange` 콜백 prop 추가 |

---

## 핵심 구현

### 1. API 단건 → 다건 변경

```typescript
// as-is
getLastInvestStrategy(customerId, accountType)  // 단건
useGetLastInvestStrategy(customerId, accountType, options)

// to-be
getLastInvestStrategyAll(customerId)  // 전체 계좌타입 배열 반환
useGetLastInvestStrategyAll(customerId, options)

// 엔드포인트
// as-is: /${customerId}/invest/strategy/last/${accountType}
// to-be: /${customerId}/invest/strategy/last-portfolio/FA
```

### 2. CONTRACT_LIST — 계좌 타입 고정 목록

```typescript
// Board/consts.ts
const CONTRACT_LIST = [
  { accountType: 'IRP',     advisoryAccountType: 'IRP', contractType: '자문', accountTypeName: 'IRP' },
  { accountType: 'ISA',     advisoryAccountType: 'ISA', contractType: '자문', accountTypeName: 'ISA' },
  { accountType: 'PENSION', advisoryAccountType: 'PSA', contractType: '일임', accountTypeName: '연금저축' },
  { accountType: 'GENERAL', advisoryAccountType: 'BA',  contractType: '일임', accountTypeName: '종합위탁' },
];

// QbMp.AccountType → AdvisoryAccountType.Keys 변환
// 맵핑 추가/변경은 CONTRACT_LIST에서만 관리
export const getAdvisoryAccountType = (accountType) =>
  CONTRACT_LIST.find((c) => c.accountType === accountType)?.advisoryAccountType;
```

### 3. sendInfoMap — 발송 현황 빠른 조회

```typescript
// accountType 기준으로 lastStrategyList 배열을 Map으로 변환
type SendInfoMap = Partial<Record<AdvisoryAccountType.Keys, ILastInvestStrategy>>;

const sendInfoMap = useMemo<SendInfoMap>(() => {
  if (!lastStrategyList) return {};
  return lastStrategyList.reduce((acc, item) => {
    acc[item.accountType] = item;
    return acc;
  }, {});
}, [lastStrategyList]);
```

### 4. 계약 타입 판단 — serviceCode 기반

```typescript
// features/portfolio/consts.ts
// serviceCode 마지막 자리: '0' → 자문, '1' → 일임
const SERVICE_CODE_SUFFIX_MAP = { '0': '자문', '1': '일임' };

export const getContractTypeByServiceCode = (serviceCode?: string | null) => {
  if (!serviceCode) return '';
  return SERVICE_CODE_SUFFIX_MAP[serviceCode.slice(-1)] ?? '';
};
```

### 5. selectedAccountType — DetailView ↔ ListNavigation 연결

ListNavigation에서 계좌 타입 선택 시 `onAccountTypeChange` 콜백으로 부모(DetailView)에 전달.
DetailView에서 `selectedAccountType` state로 관리 후 DetailContent에 전달.

```typescript
// DetailView.tsx
const [selectedAccountType, setSelectedAccountType] = useState<QbMp.AccountType | undefined>(type);

<ListNavigation onAccountTypeChange={setSelectedAccountType} ... />
<DetailContent accountType={selectedAccountType} ... />
```

---

## 변경 이력

| 날짜 | 변경 내용 | 관련 파일 |
|------|-----------|-----------|
| 2026-04-06 | 최초 작성 (프론트 변경사항 기준으로 재작성) | 전체 |
