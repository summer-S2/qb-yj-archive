# 인출설계 구현 계획 — #943

> spec.md를 기반으로 작성. 코드 수준의 구현 방법과 결정 사항을 정리한다.
> 기획이 변경되면 spec.md를 먼저 수정하고, 이 파일의 변경 이력 섹션에 사유를 기록한다.

---

## 브랜치

`feature/#943-withdrawal-plan`

---

## 파일 구조

> ⚠️ 2026-03-18: 백엔드 API 경로(`/v1/user/withdrawal-survey/*`) 기준으로 전체 용어 통일.
> `WithdrawalPlan` → `WithdrawalSurvey`, `withdrawal-plan` → `withdrawal-survey`

### 신규 생성

```
src/
├── features/WithdrawalSurvey/
│   ├── types.ts                    # 폼 데이터 타입, 단계 ID, 단계 목록 함수
│   ├── Intro.tsx                   # 인출설계 인트로
│   ├── NoAsset.tsx                 # 자산 없음 안내
│   ├── Confirm.tsx                 # 입력 내용 확인 + 약관 동의 바텀시트
│   ├── Loading.tsx                 # FA 전달 중 로딩
│   ├── Result.tsx                  # 완료/실패
│   └── Survey/
│       ├── index.tsx               # 단계 관리, 프로그레스바, 네비게이션
│       ├── RetirementAge.tsx       # Step 1: 은퇴 시기 (슬라이더)
│       ├── LifeExpectancy.tsx      # Step 2: 기대수명 (슬라이더)
│       ├── MonthlyExpense.tsx      # Step 3: 월 희망 수령액 (슬라이더)
│       ├── Nps.tsx                 # Step 4: 국민연금
│       ├── WorkerSalary.tsx        # Step 5: 근로소득
│       └── OtherIncome.tsx         # Step 6: 기타 정기소득
│
└── pages/dashboard/withdrawal-survey/
    ├── Intro.tsx
    ├── Survey.tsx
    ├── NoAsset.tsx
    ├── Confirm.tsx
    ├── Loading.tsx
    └── Result.tsx
```

> ⚠️ `Terms.tsx` (feature + page) 및 `terms` 라우트 삭제. 약관 동의는 Confirm.tsx 내 BottomSheet로 처리.

### 수정된 파일

| 파일 | 수정 내용 |
|------|-----------|
| `src/routes/paths.ts` | `withdrawalSurvey` 경로 상수 추가 |
| `src/routes/dashboard/dashboardRoutes.tsx` | 7개 페이지 lazy 로딩 + 라우트 등록 |
| `src/pages/dashboard/mydata/fa/AssetSyncResult.tsx` | 진입 바텀시트 추가 |
| `src/hooks/renewal/useWithdrawalSurveyForm.ts` | defaultValues에서 npsKnown → isReceivingNps |

---

## 라우트 경로

```
/dashboard/withdrawal-survey/intro
/dashboard/withdrawal-survey/survey    ← ?step=STEP_ID&mode=edit
/dashboard/withdrawal-survey/no-asset
/dashboard/withdrawal-survey/confirm
/dashboard/withdrawal-survey/loading
/dashboard/withdrawal-survey/result
```

경로 상수: `PATH_DASHBOARD.withdrawalSurvey.*` (`src/routes/paths.ts`)

---

## 데이터 모델

### 폼 데이터 (`WithdrawalSurveyFormData`)

```
hasDcAsset?         boolean   - Intro에서 판단 후 saveFormData로 저장
retirementAge       number    - 기본값 60
lifeExpectancy      number    - 기본값 90
monthlyExpense      number
isReceivingNps?     boolean   - 국민연금 수령 여부 (true=수령중, false=미수령)
npsMonthlyAmount?   number    - 실제 또는 예상 월 수령액 (직접입력 or 계산기 결과)
isWorker            boolean   - hasDcAsset=true 이면 수집 안 함
salary?             number    - isWorker=true 시 입력
workingYears?       number    - isWorker=true 시 입력
hasOtherIncome      boolean
otherAnnualIncome?  number    - hasOtherIncome=true 시 입력
```

> ⚠️ 이전 `npsKnown: 'yes'|'no'` 필드는 `isReceivingNps: boolean`으로 대체됨
> ⚠️ 계산기 보조 입력값(연소득, 최초 가입 시기)은 폼/localStorage에 저장하지 않음. Nps.tsx 로컬 state로만 관리.

### localStorage
- 키: `withdrawal_survey_form_data`
- 폼 값 변경 시마다 자동 저장 (`watch()` 구독)
- Intro에서 `hasDcAsset` 판단 후 `saveFormData({ hasDcAsset })` 명시적 저장
- FA 전달 성공 시 자동 삭제
- Confirm 페이지는 localStorage를 직접 읽어서 렌더링

---

## 핵심 구현 사항

### 1. Survey 네비게이션 — 쿼리 파라미터

`useState` 대신 `useSearchParams`(react-router)를 사용해 단계를 URL로 관리한다.

**이 방식을 선택한 이유**
- 브라우저 뒤로가기가 단계별로 자연스럽게 동작
- Confirm → 뒤로가기 시 마지막 Survey 단계 URL로 복귀
- 앱 이탈 후 재진입해도 URL에 단계 정보가 남아있어 이어서 진행 가능
- 수정 모드도 URL로 표현 가능해 location.state 의존도 제거

**구현 방식**
```typescript
// 현재 단계 읽기
const [searchParams, setSearchParams] = useSearchParams();
const currentStep = (searchParams.get('step') as SurveyStepId) ?? surveySteps[0];
const isEditMode = searchParams.get('mode') === 'edit';

// 다음 단계 이동 (히스토리에 쌓임)
setSearchParams({ step: nextStep });

// 수정 모드 이동 (Confirm → Survey)
navigate(`${PATH_DASHBOARD.withdrawalSurvey.survey}?step=NPS&mode=edit`);
```

**동작 시나리오**
| 상황 | 동작 |
|------|------|
| Step 3에서 뒤로가기 | URL → step=LIFE_EXPECTANCY (자동) |
| Confirm에서 뒤로가기 | URL → step=OTHER_INCOME (마지막 Survey URL) |
| Step 1에서 뒤로가기 | `navigate(-1)` → Intro로 이동 |
| 수정 모드 뒤로가기/수정완료 | `navigate(confirm)` |

### 2. hasDcAsset 흐름

```
Intro.tsx
  → useAssetList() 조회
  → hasPensionAsset = data.some(account_type === 'irp' || 'dc')
  → hasDcAsset = data.some(account_type === 'dc') ?? false
  → saveFormData({ hasDcAsset })   ← 명시적 저장 (기존 저장값과 merge)
  → navigate(`${survey}?step=RETIREMENT_AGE`)

Survey/index.tsx
  → useWithdrawalSurveyForm() → localStorage에서 hasDcAsset 복원
  → getSurveySteps(hasDcAsset) → 5 or 6단계 결정

Confirm.tsx
  → formData.hasDcAsset → 근로소득 행 표시 여부 결정
```

### 3. NPS 화면 — 바텀시트 계산기

```
Nps.tsx
  ├── isReceivingNps 선택 카드 (네/아니요)
  ├── npsMonthlyAmount 인풋 (isReceivingNps 선택 후 표시, getInputRef={monthlyAmountInputRef})
  │     - 네: "월 수령액 (세전)" 레이블
  │     - 아니요: "예상 월 수령액 (세전)" 레이블
  ├── "월 수령액 계산기" 버튼 → setIsCalcOpen(true)
  └── BottomSheet (open=isCalcOpen, fullHeight, title="국민연금 수령액 계산기", onClose=handleCalcClose)
        ├── npsAnnualIncome 인풋 (로컬 state)
        ├── npsStartYear / npsStartMonth 셀렉트 (로컬 state)
        ├── 납입 종료 예정일: 캡션 텍스트 옆 인라인 표시 ("만 60세 기준으로 자동 적용돼요 · YYYY-MM")
        └── "계산 후 적용하기" 버튼 (npsAnnualIncome && npsStartYear && npsStartMonth 모두 입력 시 활성화)
              → 계산 → setValue('npsMonthlyAmount') → setIsCalcOpen(false) → resetCalcState() → monthlyAmountInputRef.current?.focus()
```

- 계산기 보조 입력값은 **로컬 state**(`useState`)로 관리 — RHF Controller 사용 안 함
- 납입 종료 예정일: `birth.getFullYear() + 60`, `YYYY-MM` 형식 (retirementAge와 무관). 별도 박스 없이 캡션 옆 인라인
- BottomSheet 닫힘(계산 후 적용하기 / X 버튼) 시 npsAnnualIncome, npsStartYear, npsStartMonth 초기화
- BottomSheet props: `open`, `title`, `onClose`, `fullHeight` (okText/onClick/disabledOkButton 없음 → 푸터 버튼 미표시)
- `npsCalculated` 상태 제거 — 계산과 동시에 적용하므로 중간 결과 표시 불필요

### 인풋 자동 포커싱

패턴: 조건이 충족(선택지 선택 or 화면 진입)될 때 useEffect로 setTimeout 100ms 후 focus

```typescript
const inputRef = useRef<HTMLInputElement>(null);

// isReceivingNps가 boolean이 되는 시점 (선택 직후 + 수정모드 진입 시)
useEffect(() => {
  if (isBoolean(isReceivingNps)) {
    const timer = setTimeout(() => inputRef.current?.focus(), 100);
    return () => clearTimeout(timer);
  }
}, [isReceivingNps]);

// NumericFormat에 ref 연결
<NumericFormat getInputRef={inputRef} ... />
```

적용 파일 및 트리거:
| 파일 | 트리거 조건 | 포커싱 대상 |
|------|-------------|-------------|
| `Nps.tsx` | `isBoolean(isReceivingNps)` | `npsMonthlyAmount` 인풋 |
| `Nps.tsx` (계산기 닫힌 후) | `handleCalcApply` 실행 후 | `npsMonthlyAmount` 인풋 |
| `WorkerSalary.tsx` | `isWorker === true` | `salary` 인풋 |
| `OtherIncome.tsx` | `hasOtherIncome === true` | `otherAnnualIncome` 인풋 |

### 4. NPS 단계 완료 조건

```
case 'NPS':
  return !!values.npsMonthlyAmount;   // 직접입력이든 계산기 결과든 값 있으면 통과
```

### 5. Confirm → FA에게 전달하기 + 약관 동의 BottomSheet

약관 동의는 별도 페이지 없이 Confirm 내 BottomSheet로 처리.
`goToLoading` 공통 함수를 두 경로(이미 동의 / BottomSheet 동의 완료)에서 공유.

```typescript
// Confirm.tsx
const goToLoading = () => navigate(PATH_DASHBOARD.withdrawalSurvey.loading);

const handleFaSubmit = () => {
  const hasAgreedT0054 = (termsCache.userAgreeTerms.read() || [])
    .some((item) => item.term_id === TERM_CODES._54);
  if (hasAgreedT0054) {
    goToLoading();         // 이미 동의 → 바로 Loading
  } else {
    setIsTermsOpen(true);  // 미동의 → BottomSheet 오픈
  }
};

const handleTermsSubmit = () => {
  setIsTermsLoading(true);
  mutateSubmitTerms.mutate(
    { input: [{ agr_yn: true, term_id: recentTerm.TRM_ID, term_ver: ... }], userId: user.id },
    {
      onSuccess: () => { termsCache.userAgreeTerms.refetch(); goToLoading(); },
      onError: () => showToast('서비스 에러가 발생했습니다.\n잠시 후 다시 시도해주세요.'),
      onSettled: () => setIsTermsLoading(false)
    }
  );
};
```

**BottomSheet 구성**
- `open={isTermsOpen}`, `title="인출설계 서비스 이용을 위해"`, `title2="약관에 동의해 주세요"`
- `okText="동의하고 전달하기"`, `onClick={handleTermsSubmit}`, `isLoading={isTermsLoading}`
- `onClose={() => setIsTermsOpen(false)}`
- children: `<TermsComponent terms={[recentTerm]} hasAllAgree={false} hasCheckIcon={false} />`
- T0054 term: `termCache.boManagedTerms.read()` → `TRM_ID === TERM_CODES._54` 필터 → 최신 버전

### 6. 단계 완료 조건 (다음 버튼 활성화)
| 단계 | 조건 |
|------|------|
| RETIREMENT_AGE | retirementAge 값 존재 |
| LIFE_EXPECTANCY | lifeExpectancy 값 존재 |
| MONTHLY_EXPENSE | monthlyExpense 값 존재 |
| NPS | npsMonthlyAmount 값 존재 (직접 입력 또는 계산기 결과) |
| WORKER_SALARY | isWorker가 false이거나, true이면 salary + workingYears 모두 입력 |
| OTHER_INCOME | hasOtherIncome가 false이거나, true이면 otherAnnualIncome 입력 |

### 6. 레이아웃 규칙
- 모든 페이지: `Layout` 컴포넌트 (`layouts/renewal/main/Layout`) 사용
  - `header` / `content` / `footer` prop으로 분리
  - NaviBar는 header에, FixedButton은 footer에
- Props 인터페이스 이름: 컴포넌트명 무관하게 `IProps`로 통일

### 7. 자동 계산 날짜 (생년월일 기반)
- `useAuth().user.birth_dd` → `new Date(birth_dd)`
- 만 N세가 되는 년/월: `birth.getFullYear() + targetAge` , `birth.getMonth() + 1`
- 표시 형식: `"YYYY년 M월"` (OtherIncome) / `"YYYY-MM"` (Nps 계산기)
- 적용 위치:
  - Nps.tsx — 납입 종료 예정일 (만 60세 고정, 캡션 옆 인라인)
  - OtherIncome.tsx — 소득 시작 시기 (retirementAge), 소득 종료 시기 (lifeExpectancy)

### 8. 진입 바텀시트 (AssetSyncResult.tsx)
- `useEffect`로 `isSyncSuccess && 만나이 > 55` 체크 → 자동 오픈
- 만 나이 계산: `useAuth().user.birth_dd` (ISO 형식) 기반
- "네" → `navigate(PATH_DASHBOARD.withdrawalSurvey.intro)`
- "아니요" → `setIsWithdrawalSheetOpen(false)` 만 실행, 기존 화면 동작 완전 유지
- BottomSheet는 반드시 children이 있는 닫힘 태그 사용 (`<BottomSheet>내용</BottomSheet>`)

---

## 미구현 / API 연동 대기

| 항목 | 현재 상태 | 연동 시 작업 위치 |
|------|-----------|------------------|
| IRP/DC 자산 확인 | `useAssetList()` API 연동 완료 | `Intro.tsx` |
| DC 자산 여부 → hasDcAsset 저장 | Intro에서 판단 후 saveFormData 호출 | `Intro.tsx` |
| 국민연금 수령액 계산 | 임시 계산식 (연소득 × 4.5% ÷ 12) | `Survey/Nps.tsx` `handleCalculate` |
| FA 전달 API | 2초 딜레이 임시 처리 | `Loading.tsx` `onSubmit` prop |
| 약관 본문/링크 | T0054 연동 완료. 보기 버튼은 Terms 컴포넌트 내부 BottomSheet 처리 | `Confirm.tsx` 내 BottomSheet |

---

## 변경 이력

| 날짜 | 변경 내용 | 관련 파일 |
|------|-----------|-----------|
| 2026-03-12 | 최초 작성. 화면 구현 완료 (API 연동 전) | 전체 |
| 2026-03-12 | 설문 단계 구조 변경: 별도 페이지 → 단일 Survey 내 단계별 컴포넌트로 전환 | `Survey/` |
| 2026-03-12 | Confirm → Review 이름 변경 취소, Confirm 유지 | `Confirm.tsx`, `paths.ts` |
| 2026-03-12 | DC 자산 질문 단계 제거. 대신 마이데이터에서 자동 판단하도록 변경 | `types.ts`, `Survey/index.tsx` |
| 2026-03-12 | 자동설정 항목(납입종료, 소득시기)을 실제 생년월일 기반 계산으로 변경 | `Nps.tsx`, `OtherIncome.tsx` |
| 2026-03-12 | NPS 화면 개편: 단일 인풋 + 계산기 바텀시트 방식으로 변경. npsKnown → isReceivingNps | `Nps.tsx`, `types.ts` |
| 2026-03-12 | Survey 네비게이션: useState → 쿼리 파라미터 방식으로 변경 | `Survey/index.tsx`, `Confirm.tsx` |
| 2026-03-12 | hasDcAsset: Intro에서 판단 후 saveFormData로 저장하는 흐름 확립 | `Intro.tsx` |
| 2026-03-12 | Confirm → FA 전달하기 버튼에 T0054 약관 동의 여부 분기 추가 (동의 → Loading, 미동의 → Terms) | `Confirm.tsx` |
| 2026-03-12 | Terms 화면 전환: 별도 페이지 → Confirm 내 BottomSheet. goToLoading 공통함수로 전달 로직 단일화 | `Confirm.tsx`, `Terms.tsx`(삭제), 라우트 |
| 2026-03-12 | 국민연금 계산기: 납입 종료 예정일 만 60세 고정 + YYYY-MM 형식. 보조 입력값 로컬 state로 분리. 닫힘 시 초기화 | `Nps.tsx`, `types.ts`, `useWithdrawalSurveyForm.ts` |
| 2026-03-13 | 계산기 UI 개편: 납입 종료 예정일 → 캡션 인라인. 계산하기+확인 → [계산 후 적용하기] 단일 버튼. npsCalculated 상태 제거. 인풋 자동 포커싱 (Nps/WorkerSalary/OtherIncome) | `Nps.tsx`, `WorkerSalary.tsx`, `OtherIncome.tsx` |
| 2026-03-13 | 최초 가입 시기 날짜 제한: 현재 달 이전만 선택 가능 (미래 선택 불가) | `Nps.tsx` |
| 2026-03-13 | 숫자 입력 컴포넌트 교체: NumericFormat 직접 사용 → CurrencyField 통일 | `Nps.tsx`, `WorkerSalary.tsx`, `OtherIncome.tsx` |
| 2026-03-18 | 퇴직설문 API 3개 연동: `getWithdrawalSurveyResult`, `postWithdrawalSurveyResult`, `getLatestWithdrawalSurvey` (`src/api/better/withdrawal-survey.ts` 신규) | `withdrawal-survey.ts`, `Loading.tsx`, `Intro.tsx`, `Result.tsx` |
| 2026-03-18 | 용어 통일: 프론트 `WithdrawalPlan`/`withdrawal-plan` → 백엔드 기준 `WithdrawalSurvey`/`withdrawal-survey`로 전면 변경 (18개 파일). 한글 UI 문자열("인출설계") 유지. localStorage 키도 `withdrawal_survey_form_data`로 변경 (기존 임시 저장 폼 데이터 초기화) | `paths.ts`, `dashboardRoutes.tsx`, `features/WithdrawalSurvey/`, `pages/dashboard/withdrawal-survey/`, `hooks/renewal/useWithdrawalSurveyForm.ts`, `main-home/index.tsx`, `Debugger.tsx` |
