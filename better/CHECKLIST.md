# 이슈 작업 가이드

> 이슈 시작 전 확인, 문서 작성 방법, Claude 요청 패턴, 자주 쓰는 컴포넌트 패턴

---

## 문서 작성 위치 기준

| 기준 | 위치 |
|------|------|
| 다른 이슈에도 반드시 적용되어야 하는 전체 규칙 | `CHECKLIST.md` 또는 `CLAUDE.md` |
| 이번 이슈에만 국한된 결정 / 구현 방식 | `#이슈번호/PLAN.md` 내부 |

> 이슈 작업 중 발견한 패턴이 **프로젝트 전체에 적용되어야 한다면** 이슈 폴더가 아닌 `CHECKLIST.md`에 작성하고, 이슈 폴더 내 중복 내용은 제거한다.

---

## 새 이슈 시작 전 체크리스트

- [ ] `#이슈번호/` 폴더 생성했는가?
- [ ] SPEC.md에 **재진입 정책** 명시했는가?
- [ ] SPEC.md에 **바텀시트/팝업 재노출 조건** 명시했는가?
- [ ] SPEC.md에 **미확정 항목** 명시했는가?
- [ ] PLAN.md에 **삭제되는 파일** 명시했는가?
- [ ] PLAN.md에 **localStorage/상태 저장 범위** 명시했는가?
- [ ] PLAN.md에 **API 연동 대기 항목** 표로 정리했는가?
- [ ] 문서 수정 시 **변경 이력** 기록했는가?

---

## 문서 작성 순서

```
1. SPEC.md 작성 (기획 확정)
      ↓ Claude에게 검토 요청 + 미확정 항목 논의
2. PLAN.md 작성 (구현 방법 결정)
      ↓ Claude에게 검토 요청 + 방향 합의
3. 코드 작업
      ↓ 기획/구현 변경 발생 시
4. SPEC.md 먼저 수정 → PLAN.md 수정 → 코드 수정 순서 유지
```

> ⚠️ 코드 먼저 수정하고 문서를 나중에 업데이트하면 문서와 코드가 어긋남.

---

## Claude 요청 방법

### SPEC.md 작성 요청

```
#이슈번호 작업 시작할게.
#이슈번호/SPEC.md 작성해줘.

[기능 설명]
- 기능 목적: ...
- 진입 조건: ...
- 전체 플로우: ...
- 화면 목록: ...
- 미확정 항목: ...

templates/SPEC.md 참고해서 작성해줘.
```

### PLAN.md 작성 요청

```
SPEC.md 검토 완료됐어.
#이슈번호/PLAN.md 작성해줘.

고려해줘야 할 것들:
- 기존에 비슷한 기능 있으면 그 패턴 따라줘 (예: investment-diagnosis)
- 상태 관리 방법: [localStorage / Redux / RHF 등]
- API 연동 여부: [연동 완료 / 임시 처리 / 미연동]

templates/PLAN.md 참고해서 작성해줘.
```

### 코드 작업 요청

```
SPEC.md랑 PLAN.md 기반으로 코드 작업 시작해줘.
#이슈번호/SPEC.md
#이슈번호/PLAN.md
```

### 기획 변경 시

```
수정할 부분이 있어.
[변경 내용 설명]

SPEC.md 먼저 수정하고 PLAN.md 수정한 다음에 코드 수정해줘.
```

### 좋은 요청의 특징

- **What보다 Why**: 왜 이렇게 해야 하는지 설명하면 Claude가 더 나은 판단을 함
- **미확정 항목 명시**: "이 부분은 아직 결정 안 됐어" → 가정 대신 placeholder 처리
- **참고 파일 지목**: "X 파일 패턴 따라줘"처럼 기존 코드를 레퍼런스로 제공
- **제약 조건 명시**: 삭제하면 안 되는 파일, 건드리면 안 되는 로직 미리 명시

---

## 자주 쓰는 컴포넌트 패턴

### 레이아웃

```tsx
import Layout from 'layouts/renewal/main/Layout';

<Layout
  header={<NaviBar title="제목" />}
  content={<div>...</div>}
  footer={<FixedButton>다음</FixedButton>}
/>
```

### 선택형 카드

```tsx
import SelectList from 'components/tailwindcss/Form/Selection/SelectList';

<SelectList
  text="선택지 텍스트"
  isActive={value === true}
  onClick={() => setValue('field', true)}
/>
```

### 숫자 입력

```tsx
import CurrencyField from 'components/tailwindcss/Form/TextInput/CurrencyField';

// onChange: (e) => field.onChange(Number(e.target.value.replace(/,/g, '')) || undefined)
// ref 연결: ref={inputRef} (forwardRef 지원)
// 천단위 없는 경우: thousandSeparator={false}
```

### 바텀시트

```tsx
import BottomSheet from 'components/tailwindcss/Modal/BottomSheet';

// ⚠️ 반드시 닫힘 태그 사용
<BottomSheet
  open={isOpen}
  title="제목"
  okText="확인"
  onClick={handleConfirm}
  onClose={handleClose}
>
  내용
</BottomSheet>
```

**푸터 버튼 없는 바텀시트** (fullHeight, X 버튼으로만 닫기):

```tsx
// okText / onClick 없이 사용하면 푸터 버튼 미표시
<BottomSheet
  open={isOpen}
  title="제목"
  fullHeight
  onClose={handleClose}
>
  내용
</BottomSheet>
```

### 인풋 자동 포커싱

```typescript
const inputRef = useRef<HTMLInputElement>(null);

useEffect(() => {
  if (조건) {
    const timer = setTimeout(() => inputRef.current?.focus(), 100);
    return () => clearTimeout(timer);
  }
}, [조건]);
```

### 자동 계산 날짜 (생년월일 기반)

```typescript
// useAuth().user.birth_dd → 만 N세가 되는 년/월
const birth = new Date(user.birth_dd);
const targetYear = birth.getFullYear() + targetAge;
const targetMonth = String(birth.getMonth() + 1).padStart(2, '0');
// 표시: "YYYY년 M월" 또는 "YYYY-MM"
// ⚠️ 별도 util 파일 생성하지 않고 컴포넌트 내 인라인 계산
```

---

## React Query 패턴

### 404 → null 처리 (stale 캐시 방지)

React Query v3는 background refetch 실패 시 `isSuccess`가 `true`로 유지되어 이전 캐시 데이터가 잔존함.
데이터 없음(404)을 에러가 아닌 `null`로 처리하면 이전 캐시 덮어쓰기 가능.

```typescript
export async function getSomething(id: string): Promise<SomeType | null> {
  try {
    const { data } = await http.get(...);
    return data;
  } catch (e) {
    if ((e as AxiosError)?.response?.status === 404) return null;
    return Promise.reject(e);
  }
}

// useQuery select에도 null 가드 추가
useQuery(key, getSomething, {
  select: (data) => data ? { ...data, extraField } : null,
});
```

### 파생값은 useState 대신 직접 derive

비동기 업데이트로 인한 타이밍 gap을 방지하기 위해, 다른 값에서 계산 가능한 값은 `useState` + `useEffect` 대신 직접 계산한다.

```typescript
// ❌ 비동기 업데이트 → 타이밍 gap
const [isDerived, setIsDerived] = useState(false);
useEffect(() => {
  setIsDerived(list.some((d) => d.value));
}, [list]);

// ✅ 즉시 반영
const isDerived = list.some((d) => d?.value);
```

---

## 상태 관리 분리 기준

| 상태 종류 | 위치 |
|-----------|------|
| UI 상태, 사용자 세션, 설정값 | Redux |
| 서버 데이터 (API 응답) | React Query |
| 화면 이탈 후 재진입 시 복원 필요한 임시 폼 데이터 | localStorage |

---

## 디렉토리 구조 원칙

- 페이지 컴포넌트는 간결하게 유지
- 실제 로직은 `features/` 하위에 작성
- 신규 컴포넌트는 **Tailwind CSS** 기반 (`components/tailwindcss/`)
- 경로는 **하드코딩 금지** — 반드시 `routes/paths.ts` 상수 사용
- React Query 키는 `constants/query-key.ts` 중앙 관리
