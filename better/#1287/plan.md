# QB 알림톡 ctrNo 딥링크 구현 계획 — #1287

> spec.md를 기반으로 작성. 코드 수준의 구현 방법과 결정 사항을 정리한다.
> 기획이 변경되면 spec.md를 먼저 수정하고, 이 파일의 변경 이력 섹션에 사유를 기록한다.

---

## 브랜치

`feature/#1287-multiple-contracts`

---

## 수정된 파일

| 파일 | 수정 내용 |
|------|-----------|
| `src/redux/slices/app.ts` | `qbCtrNo` 상태/reducer/selector 추가 |
| `src/hooks/useDynamicLink.tsx` | `/qb` 링크 파싱 시 ctrNo 추출 + dispatch |
| `src/pages/dashboard/DashboardBridge.tsx` | split('-') 로직 제거, Redux qbCtrNo 직접 사용 |
| `src/pages/qb/Landing.tsx` | ctrNo searchParams 읽기, useQueries 추가, 초기화 |
| `src/tailwind-features/home/Quarterback/index.tsx` | navigate state mpList 전달 제거 |
| `src/pages/renewal/quarterback/index.tsx` | isProposedCustomer derive 방식 변경, pendings/contracts 게이팅 |
| `src/api/quarterback.ts` | getLastInvestStrategy 404 → null 처리 |

---

## 핵심 구현 사항

### 1. Redux qbCtrNo 상태 추가

`src/redux/slices/app.ts`

```typescript
// InitialState
qbCtrNo: string;

// initialState
qbCtrNo: '',

// reducer
setQbCtrNo: (state, action: PayloadAction<string>) => {
  state.qbCtrNo = action.payload;
},

// selector
export const selectQbCtrNo = createSelector(selectSelf, (state) => state.qbCtrNo);
```

---

### 2. useDynamicLink — ctrNo 파싱

`src/hooks/useDynamicLink.tsx` (44-52줄)

```typescript
if (link.includes('/qb')) {
  // 경로의 마지막 세그먼트에서 쿼리파라미터 제거 후 routeKey 추출 (ex: deposit)
  const qbLinkKey = link.split('/').pop()?.split(/[?&]/)[0] ?? '';
  // 알림톡 딥링크에 포함된 계약번호(ctrNo) 추출 (없으면 빈 문자열)
  const ctrNoMatch = link.match(/[?&]ctrNo=([^&]*)/);
  const ctrNo = ctrNoMatch?.[1] ?? '';
  dispatch(setQbLinkKey(qbLinkKey));
  dispatch(setQbCtrNo(ctrNo));
  nativeUtil.closeQbWebExternal();
  return 'qb-' + qbLinkKey;
}
```

**기존 문제**: 이전에 DashboardBridge에서 `qbLinkKey.split('-')`로 ctrNo를 추출하려 했으나,
`useDynamicLink`가 쿼리파라미터를 버리고 경로만 저장했기 때문에 한 번도 동작하지 않았음.

---

### 3. DashboardBridge — split('-') 로직 제거

`src/pages/dashboard/DashboardBridge.tsx` (102줄 근처)

```typescript
// 변경 전 (동작하지 않던 코드)
const paramsData = qbLinkKey.split('-');
let paramCtrNo = '';
if (paramsData.length > 1) {
  paramCtrNo = paramsData[paramsData.length - 1];
}
const flowParam = paramsData[0];
navigate(PATH_DASHBOARD.qb.landingWebview + `?routeParams=${flowParam}&ctrNo=${paramCtrNo}`);

// 변경 후
navigate(PATH_DASHBOARD.qb.landingWebview + `?routeParams=${qbLinkKey}&ctrNo=${qbCtrNo}`);
```

---

### 4. Landing.tsx — mpList useQueries 추가 + ctrNo 초기화

**딥링크 직입 시 mpList 문제 해결 방식**

| 방식 | 이유 |
|------|------|
| Landing에서 동일 queryKey로 useQueries 직접 호출 | 캐시 공유로 정상 흐름에서 추가 API 없음. 딥링크 직입 시 자동 fetch |
| location.state.mpList 제거 | 소스 단일화 (캐시가 항상 최신 정보) |
| Quarterback/index.tsx navigate state 제거 | mpList 전달 불필요 |

**웹뷰 진입 조건**
```typescript
// hasAgreedQbiTerms AND mpList 로딩 완료 시에만 웹뷰 오픈
useEffect(() => {
  if (hasAgreedQbiTerms && !isLastStrategyLoading) {
    const time = setTimeout(() => handleOpenWebview(), 1000);
    return () => clearTimeout(time);
  }
}, [hasAgreedQbiTerms, isLastStrategyLoading]);
```

**Redux 초기화**
```typescript
useEffect(() => {
  // Landing 진입 시 딥링크 관련 Redux 상태 초기화 (재진입 시 이전 값 유입 방지)
  dispatch(setQbLinkKey(''));
  dispatch(setQbCtrNo(''));
}, []);
```

---

### 5. getLastInvestStrategy 404 → null 처리

> 일반 원칙: `CHECKLIST.md` > React Query 패턴 > 404 → null 처리 참고

```typescript
// 404 throw 대신 null 반환 → stale 캐시 덮어쓰기
export async function getLastInvestStrategy(
  customerId: string,
  accountType: string
): Promise<Quarterback.ILastInvestStrategy | null> {
  try {
    const { data } = await http.get(...);
    return data;
  } catch (e) {
    if ((e as AxiosError)?.response?.status === 404) return null;
    return Promise.reject(e);
  }
}
```

`select`도 null 가드 추가:
```typescript
select: (data: Quarterback.ILastInvestStrategy | null) =>
  data ? { ...data, accountType } : null,
```

---

### 6. QuarterbackInvestment — isProposedCustomer + 데이터 정합성

> 일반 원칙: `CHECKLIST.md` > React Query 패턴 > 파생값은 useState 대신 직접 derive 참고

**문제**: `proposedMpList`가 비어도 캐시된 `pendings`/`contracts` 데이터가 남음

```typescript
// 파생값 직접 derive
const isProposedCustomer = proposedMpList.some((d) => d?.portfolioCode);

// 캐시 데이터 게이팅
const pendings = isProposedCustomer ? pendingsRaw : null;
const contracts = isProposedCustomer ? contractsRaw : null;
```

**동작 시나리오**
| 상황 | 동작 |
|------|------|
| proposedMpList 있음 | isProposedCustomer = true, 캐시 데이터 그대로 사용 |
| proposedMpList 비어짐 (404) | isProposedCustomer = false 즉시 반영, pendings/contracts = null |
| enabled = false | 불필요한 API 호출 차단 |

---

## 검증 시나리오

### ctrNo 있는 딥링크
```
https://betterday.co.kr/qb/deposit?ctrNo=TEST_CTR_001
```
- `useDynamicLink` → `qbLinkKey='deposit'`, `qbCtrNo='TEST_CTR_001'`
- `DashboardBridge` → `?routeParams=deposit&ctrNo=TEST_CTR_001`
- `Landing.tsx` → `qbAccessToken.ctrNo = 'TEST_CTR_001'` ✓

### ctrNo 없는 기존 딥링크
```
https://betterday.co.kr/qb/deposit
```
- `qbCtrNo = ''`
- `qbAccessToken.ctrNo = ''` ✓ (기존 동작 유지)

### 딥링크 직입 mpList
- main 미경유 → Landing에서 getLastInvestStrategy × 4 자동 fetch
- 로딩 완료 전 웹뷰 진입 차단 ✓
- 완료 후 mpList 포함된 qbAccessToken 전달 ✓

### Landing 재진입
- Redux qbLinkKey = '', qbCtrNo = '' 초기화 확인 ✓

---

---

## QB 홈 화면 상태 통합 API 전환

### 수정 파일

| 파일 | 수정 내용 |
|------|-----------|
| `src/types/quarterback/index.ts` | `IContractInvestHomeStatus` 인터페이스 추가. `contracts`, `pendingContracts` null 허용 |
| `src/api/quarterback.ts` | `URLS.GET_CONTRACT_INVEST_HOME_STATUS`, `QUERY_KEYS.CONTRACT_INVEST_HOME_STATUS` 추가. `getContractInvestHomeStatus()` 함수 및 `useGetContractInvestHomeStatus()` 훅 추가 |
| `src/tailwind-features/home/Quarterback/index.tsx` | 훅 호출을 컴포넌트 내부로 통합. `data: IContractInvestHomeStatus` 단일 props. boolean 플래그로 카드 분기. 평가금액 계산 제거 → API 제공값 직접 사용 |
| `src/utils/quarterback/index.ts` | `getContractInfo` 파라미터를 `{ showWellManaged, showNeedCheck, showInvestProposal }`로 변경. 우선순위 기반 분기 |
| `src/pages/renewal/quarterback/index.tsx` | re-export만 남김 (실질 로직 QuarterbackCard로 이전) |
| `src/pages/renewal/main-home/index.tsx` | `QuarterbackInvestment` → `QuarterbackCard` 직접 import로 변경 |

### 핵심 구현 사항

#### 1. 단일 API 호출

```typescript
// api/quarterback.ts
export const URLS = {
  GET_CONTRACT_INVEST_HOME_STATUS: `${API_GATEWAY_URL}/asset-main/qbis/users/invest-contract/home-status`
};

export function useGetContractInvestHomeStatus(options?) {
  return useQuery(
    [QUERY_KEYS.CONTRACT_INVEST_HOME_STATUS],
    () => getContractInvestHomeStatus(),
    { retry: false, ...options }
  );
}
```

#### 2. QuarterbackCard 통합 구조

```
QuarterbackCard (tailwind-features/home/Quarterback/index.tsx)
  ├─ useGetContractInvestHomeStatus({ enabled: !!isFaUser })
  ├─ !isFaUser || !data → null
  ├─ 3개 플래그 모두 false → null
  ├─ isLoading → Skeleton
  └─ isFetching → 데이터 영역 Skeleton (카드 유지)
```

#### 3. 에러 처리 전략

React Query 기본 동작 활용:
| 상황 | 동작 |
|------|------|
| 첫 조회 실패 | `data = undefined` → 카드 미표출 |
| refetch 실패, 이전 데이터 없음 | `data = undefined` → 카드 미표출 |
| refetch 실패, 이전 데이터 있음 | 이전 `data` 유지 → 기존 데이터 표출 |

#### 4. getContractInfo 우선순위 로직

```typescript
// utils/quarterback/index.ts
export const getContractInfo = (status: {
  showWellManaged: boolean;
  showNeedCheck: boolean;
  showInvestProposal: boolean;
}): BannerInfo => {
  if (status.showWellManaged) return { icon: 'Money', text: '투자 자산이 잘 운용되고 있어요.' };
  if (status.showNeedCheck) return { icon: 'IconText2', text: '확인이 필요한 내용이 있어요.' };
  if (status.showInvestProposal) return { icon: 'IconBell2', text: '맞춤 포트폴리오가 도착했어요.' };
  return { icon: 'IconText2', text: '확인이 필요한 내용이 있어요.' };
};
```

---

## 변경 이력

| 날짜 | 변경 내용 | 관련 파일 |
|------|-----------|-----------|
| 2026-03-24 | 최초 작성. ctrNo Redux 상태 추가, 딥링크 파싱, DashboardBridge split 로직 제거 | `app.ts`, `useDynamicLink.tsx`, `DashboardBridge.tsx`, `Landing.tsx` |
| 2026-03-24 | Landing mpList 직접 조회 추가. location.state 의존 제거. 로딩 완료 전 웹뷰 진입 차단 | `Landing.tsx`, `Quarterback/index.tsx` |
| 2026-03-24 | getLastInvestStrategy 404 → null 처리. select null 가드 추가 | `api/quarterback.ts`, `renewal/quarterback/index.tsx`, `Landing.tsx` |
| 2026-03-24 | isProposedCustomer derive 방식 변경. pendings/contracts 게이팅으로 데이터 정합성 확보 | `renewal/quarterback/index.tsx` |
| 2026-03-27 | QB 홈 화면 상태 통합 API 전환. 3개 API → 단일 호출. QuarterbackCard로 로직 통합 | `types/quarterback/index.ts`, `api/quarterback.ts`, `Quarterback/index.tsx`, `utils/quarterback/index.ts` |
| 2026-03-31 | qbLinkKey/qbCtrNo → qbLinkParams 단일 객체로 통합. queryParamKeys 배열로 파싱 자동화 | `app.ts`, `useDynamicLink.tsx`, `DashboardBridge.tsx`, `Landing.tsx`, `qb-key.ts` |
| 2026-03-31 | enterBanner location.state → Redux isInvestDiagnosisActive 전환. appPersistConfig blacklist 추가 | `app.ts`, `rootReducer.ts`, `TopBanner.tsx`, `useDynamicLink.tsx`, `BetterSignUpComplete.tsx`, 투자진단 컴포넌트 |
| 2026-03-31 | 회원탈퇴 계약 조회 API를 useGetContractInvestHomeStatus로 교체. 사용하지 않는 getContractStatus 제거 | `api/quarterback.ts`, `withdrawal/index.tsx`, `withdrawal/CheckService.tsx` |
| 2026-04-06 | 투자제안 알림톡 파라미터 camelCase 전환 대응. portfolioCode → portfolioNo keyMap에 추가 | `Landing.tsx` |
