# 이슈 작업 가이드 — fa

> 이슈 시작 전 확인, 문서 작성 방법, Claude 요청 패턴

---

## 문서 작성 위치 기준

| 기준 | 위치 |
|------|------|
| 다른 이슈에도 반드시 적용되어야 하는 전체 규칙 | `CHECKLIST.md` 또는 `CLAUDE.md` |
| 이번 이슈에만 국한된 결정 / 구현 방식 | `#이슈번호/plan.md` 내부 |

> 이슈 작업 중 발견한 패턴이 **프로젝트 전체에 적용되어야 한다면** 이슈 폴더가 아닌 `CHECKLIST.md`에 작성하고, 이슈 폴더 내 중복 내용은 제거한다.

---

## 새 이슈 시작 전 체크리스트

- [ ] `#이슈번호/` 폴더 생성했는가?
- [ ] spec.md에 **영향받는 서비스** (`@fa-be` / `@betterapp` / `@qbis`) 명시했는가?
- [ ] spec.md에 **API 호출 경로** (직접 vs BetterApp 경유) 명시했는가?
- [ ] spec.md에 **미확정 항목** 명시했는가?
- [ ] plan.md에 **Guard 종류** 명시했는가?
- [ ] plan.md에 **삭제되는 파일** 명시했는가?
- [ ] plan.md에 **API 연동 대기 항목** 표로 정리했는가?
- [ ] 문서 수정 시 **변경 이력** 기록했는가?

---

## 문서 작성 순서

```
1. spec.md 작성 (기획 확정)
      ↓ Claude에게 검토 요청 + 미확정 항목 논의
2. plan.md 작성 (구현 방법 결정)
      ↓ Claude에게 검토 요청 + 방향 합의
3. 코드 작업
      ↓ 기획/구현 변경 발생 시
4. spec.md 먼저 수정 → plan.md 수정 → 코드 수정 순서 유지
```

---

## Claude 요청 방법

### spec.md 작성 요청

```
#이슈번호 작업 시작할게.
#이슈번호/spec.md 작성해줘.

[기능 설명]
- 기능 목적: ...
- 영향받는 서비스: @fa-be / @betterapp / @qbis
- API 호출 경로: ...
- 전체 플로우: ...
- 미확정 항목: ...
```

### plan.md 작성 요청

```
spec.md 검토 완료됐어.
#이슈번호/plan.md 작성해줘.

고려해줘야 할 것들:
- Guard 종류: [JwtAuthGuard / AllianceJwtAuthGuard / AllianceSignatureAuthGuard]
- 공유 DTO 필요 여부: [libs/common 활용 여부]
- 테스트: [unit / integration]
```

### 코드 작업 요청

```
spec.md랑 plan.md 기반으로 코드 작업 시작해줘.
#이슈번호/spec.md
#이슈번호/plan.md

프론트 작업이면 apps/fe-react-app 하위에서 작업해줘.
```

---

## 프로젝트 구조

```
better-wealth-fa/
├── apps/
│   ├── fe-react-app/     ← 프론트엔드 (주 작업 위치)
│   └── be-nest/          ← 백엔드 (NestJS)
│       └── apps/
│           ├── famain/   ← @fa-be
│           └── betterapp/← @betterapp
└── packages/
    └── shared-consts/    ← 공유 상수/enum (fe-react-app 밖에서 볼 일 있을 때)
```

> ⚠️ 프론트 작업 시 `apps/fe-react-app` 내부만 보면 됨. 공유 상수 확인이 필요할 때만 `packages/shared-consts` 참조.

---

## 시스템 간 호출 규칙

- 투자플랫폼 (@qbis) ↔ FA플랫폼 (@fa-be) 직접 호출 금지
- **모든 호출은 BetterApp (@betterapp) 경유**

---

## Guard 종류 및 사용 기준

| Guard | 사용 대상 |
|-------|-----------|
| `JwtAuthGuard(ALLIANCE_JWT)` | FA + BetterApp 공용 인증 |
| `AllianceJwtAuthGuard` | @betterapp 전용 호출 |
| `AllianceSignatureAuthGuard` | @qbis 서명 인증 전용 |
| `JwtAuthGuard` | @betterapp 자체 인증 |

---

## 상태 관리 / 코드 패턴

### 병렬 조회

```typescript
// 성공/실패 혼재 가능한 경우: Promise.allSettled
const [contracts, pending, proposals] = await Promise.allSettled([...]);

// 전체 성공이 보장되는 경우: Promise.all
const results = await Promise.all(ACCOUNT_TYPES.map(...));
```

### 공유 DTO 위치

- 여러 서비스에서 공통으로 쓰는 DTO → `libs/common/src/dto/`
- 특정 모듈 전용 DTO → 해당 모듈 하위 `dto/` 폴더

### enum 상수 위치

- `packages/shared-consts` — fe/be 양쪽에서 사용하는 공유 상수
- `apps/be-nest/...` 하위 — 백엔드 전용 enum
