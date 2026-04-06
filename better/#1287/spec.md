# QB 알림톡 ctrNo 딥링크 파싱 — #1287

## 개요

QB(쿼터백) 알림톡 딥링크에 `ctrNo`(계약번호) 쿼리 파라미터가 추가됨.
딥링크로 앱에 진입할 때 계약번호를 파싱해 QB 웹뷰 진입 시 `qbAccessToken`에 포함시켜야 한다.

---

## 진입 조건

- **위치**: 알림톡 딥링크 클릭 → 앱 실행
- **조건**: `/qb/` 경로를 포함한 딥링크에 `ctrNo` 쿼리파라미터가 있는 경우
- **노출 형태**: QB Landing 페이지 → 웹뷰 자동 진입

---

## 전체 플로우

```
알림톡 딥링크 클릭
    ↓
useDynamicLink.parseDynamicLink()
    ↓ /qb/ 경로 감지
    ├─ qbLinkKey = 경로 마지막 세그먼트 (ex: deposit)
    └─ ctrNo = 쿼리파라미터 추출 (없으면 '')
    ↓ Redux dispatch
    ├─ setQbLinkKey('deposit')
    └─ setQbCtrNo('ABC123')
    ↓
DashboardBridge
    ↓ qbLinkKey 감지
    navigate('/qb/landing?routeParams=deposit&ctrNo=ABC123')
    ↓
Landing.tsx
    ├─ routeParams = 'deposit'
    ├─ ctrNo = 'ABC123'
    ├─ mpList = useQueries (getLastInvestStrategy × 4 계좌타입)
    │     → 캐시 히트(정상 흐름) or 자동 fetch(딥링크 직입)
    │     → isLastStrategyLoading 완료까지 웹뷰 진입 대기
    └─ qbAccessToken = { token, routeParams, mpList, ctrNo }
    ↓
QB 웹뷰 오픈
```

---

## 딥링크 URL 형식

### 알림톡 발송 URL (onelink)
```
https://betterday.onelink.me/4qkl/redirect
  ?link=https://betterday.co.kr/qb/deposit?ctrNo=ABC123
  &apn=com.io.finset.betterday.app
  &isi=1598857295
  &ibi=io.finset.betterday.app
  &timemillis=<타임스탬프>
```

### 지원 routeKey 목록
| 값 | 설명 |
|----|------|
| `deposit` | 최소 투자 금액 입금 요청 |
| `approval` | 포트폴리오 승인 요청 |
| `rebalanced` | 매매(리밸런싱) 완료 |
| `fee` | 수수료계좌 안내 |

### ctrNo 없는 기존 딥링크 (하위 호환)
```
https://betterday.co.kr/qb/deposit
```
→ `ctrNo = ''` 으로 처리, 기존 동작 유지

---

## 주요 체크사항

### Redux 상태 초기화
- Landing 컴포넌트 마운트 시 `setQbLinkKey('')` + `setQbCtrNo('')` 초기화
- 재진입 시 이전 딥링크 값이 남아있는 것 방지

### mpList 딥링크 직입 문제
- 정상 흐름: `QuarterbackInvestment`(main)에서 API 호출 후 캐시 존재 → Landing에서 캐시 히트
- 딥링크 직입: main 미경유로 캐시 없음 → Landing에서 직접 fetch
- 해결: Landing에서 동일 queryKey로 `useQueries` 호출 (캐시 공유)
- 웹뷰 진입 전 로딩 완료 대기: `isLastStrategyLoading` 체크

### getLastInvestStrategy 404 처리
- 404(DATA_NOT_FOUND) = 데이터 없음 → `null` 반환
- 이전 캐시 데이터가 남는 문제 방지 (React Query v3 background refetch 특성)

### QuarterbackInvestment pendings/contracts 데이터 정합성
- `proposedMpList`가 비면 `pendings`, `contracts`도 null로 처리
- `isProposedCustomer`를 useState 대신 직접 derive → 타이밍 gap 제거

---

---

## QB 홈 화면 상태 통합 API 전환

### 개요

기존 홈 화면은 QB 상태 표출을 위해 3개의 API를 각각 호출했음.
백엔드에서 이를 하나로 통합한 엔드포인트를 제공하여 단일 호출로 교체.

### 기존 → 변경 후

| 기존 (3개) | 변경 후 (1개) |
|------------|--------------|
| `getLastInvestStrategy` × 4 (계좌타입별) | `GET /asset-main/qbis/users/invest-contract/home-status` |
| `getPendingContractStatus` | ↑ 동일 응답에 포함 |
| `getContractStatus` | ↑ 동일 응답에 포함 |

### 응답 구조 (ContractInvestStatusDto)

| 필드 | 타입 | 설명 |
|------|------|------|
| `showInvestProposal` | boolean | 맞춤 포트폴리오 도착 카드 (C2-a) |
| `showNavAmount` | boolean | 총 평가금액/수익 영역 표시 여부 |
| `showNeedCheck` | boolean | 확인 필요 카드 (C2-b/c/d) |
| `showWellManaged` | boolean | 잘 운용중 카드 (C2-e) |
| `totalNavAmount` | number\|null | 총 평가금액 |
| `totalProfitAmount` | number\|null | 총 수익금액 |
| `totalProfitRate` | number\|null | 총 수익률 |
| `investProposals` | array | 투자제안 목록 |
| `contracts` | array\|null | 기계약 목록 |
| `pendingContracts` | array\|null | 미결계약 목록 |

### 카드 분기 조건 (우선순위 순)

| 우선순위 | 조건 | 카드 문구 | 평가금액 |
|---------|------|-----------|---------|
| 1 | `showWellManaged = true` | 투자 자산이 잘 운용되고 있어요 | O |
| 2 | `showNeedCheck = true` | 확인이 필요한 내용이 있어요 | showNavAmount 따름 |
| 3 | `showInvestProposal = true` | 맞춤 포트폴리오가 도착했어요 | X |

모든 플래그가 false면 카드 미표출.

---

---

## QB 딥링크 파라미터 단일 객체 통합 (qbLinkParams)

### 개요

기존에 Redux에 `qbLinkKey`, `qbCtrNo` 등을 개별 상태로 관리하던 것을 `qbLinkParams` 단일 객체로 통합.
`queryParamKeys` 배열로 지원 파라미터 목록을 중앙 관리하여 파싱 로직 자동 반영.

### 변경 전 → 후

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| Redux 상태 | `qbLinkKey: string`, `qbCtrNo: string` (개별) | `qbLinkParams: Record<string, string>` (단일 객체) |
| 딥링크 파싱 | `ctrNo`만 추출 | `queryParamKeys` 배열 전체 순회하여 자동 추출 |
| DashboardBridge → Landing | `?routeParams=X&ctrNo=Y` | `?routeParams=X&ctrNo=Y&accountType=Z&...` (파라미터 전부 전달) |

### queryParamKeys 관리 (`src/constants/qb-key.ts`)

```typescript
export const queryParamKeys = [
  'ctrNo',
  'ptrNo',
  'accountType',
  'portfolioCode',
  // TODO snake_case는 투자제안 알림톡 수정되면 지우기
  'account_type',
  'portfolio_code'
] as const;
```

딥링크 파싱 시 이 배열을 순회하며 URL에 존재하는 키만 추출 → Redux `qbLinkParams`에 저장.

### snake_case → camelCase 변환 (`src/pages/qb/Landing.tsx`)

투자제안 알림톡이 기존 snake_case로 파라미터를 전달하다가 camelCase로 전환 예정.
전환 기간 중 두 버전 모두 처리하기 위해 keyMap으로 정규화.

```typescript
const keyMap: Record<string, string> = {
  account_type: 'accountType',
  portfolio_code: 'portfolioNo',
  portfolioCode: 'portfolioNo'   // 신버전 camelCase도 portfolioNo로 통일
};
// URL에 없는 키는 searchParams.entries()에서 아예 안 나오므로 중복 걱정 없음
```

> **주의**: `portfolioCode`(URL 파라미터명)와 `portfolioNo`(QB 웹뷰 수신 키명)는 다름.
> snake_case, camelCase 모두 `portfolioNo`로 변환해야 QB 웹뷰에서 정상 처리됨.

---

## 투자진단 enterBanner → Redux isInvestDiagnosisActive 전환

### 문제

투자진단 플로우 진입 시 `enterBanner` 플래그를 `location.state`로 전달하던 방식이
여러 진입 경로(배너 클릭, 딥링크, 회원가입 완료)를 거치며 state 전파가 누락되는 문제.

### 해결

`isInvestDiagnosisActive` Redux 상태를 `app` slice에 추가.
진입 경로(배너, 딥링크, 회원가입)에서 직접 dispatch → 투자진단 컴포넌트에서 Redux로 읽음.

### 주요 변경 파일

| 파일 | 변경 내용 |
|------|-----------|
| `src/redux/slices/app.ts` | `isInvestDiagnosisActive` 상태/reducer 추가 |
| `src/redux/rootReducer.ts` | `appPersistConfig` blacklist에 추가 (persist 제외) |
| `src/features/MainHome/Banner/TopBanner.tsx` | 배너 클릭 시 `setIsInvestDiagnosisActive(true)` dispatch |
| `src/hooks/useDynamicLink.tsx` | `invest-diag` 딥링크 진입 시 dispatch |
| `src/pages/renewal/auth/BetterSignUpComplete.tsx` | 회원가입 완료 진입 시 dispatch |
| `src/features/InvestmentDiagnosis/Qustions/index.tsx` | enterBanner location.state 제거 |
| `src/features/InvestmentDiagnosis/Qustions/Qustions.tsx` | enterBanner 제거 |
| `src/features/InvestmentDiagnosis/Qustions/Result.tsx` | Redux에서 읽도록 변경. 완료 후 assetSyncLoading 이동 |

> persist blacklist에 추가한 이유: logout/탈퇴 시 자동 초기화 보장.
> persist되면 앱 재진입 시 이전 플래그가 남아 의도치 않게 투자진단이 활성화될 수 있음.

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|-----------|
| 2026-03-24 | 최초 작성 |
| 2026-03-27 | QB 홈 화면 상태 통합 API 전환 내용 추가 |
| 2026-03-31 | QB 딥링크 파라미터 단일 객체 통합(qbLinkParams), queryParamKeys 관리, snake_case→camelCase 변환 추가 |
| 2026-03-31 | 투자진단 enterBanner → Redux isInvestDiagnosisActive 전환 내용 추가 |
| 2026-04-06 | 투자제안 알림톡 portfolioCode camelCase 대응 (portfolioNo로 통일) 내용 추가 |
