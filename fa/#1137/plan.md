# 투자제안 다중계좌 구현 계획 — #1137

> 기획/정책: `spec.md` 참조

---

## 브랜치

`feature/#1137-invest-proposal-multi-account`

---

## 구현 단계 요약

```
Phase A (@fa-be 기본 기능)      ✅ 완료
Phase B (@betterapp 연동)       ✅ 완료
Phase C (@fa-be deeplink)       ✅ 완료
Phase D (2차 — 발송 가능 여부)  ⏸ 미착수
```

---

## Phase A: @fa-be 기본 기능 ✅

### 수정/생성 파일

| 파일 | 작업 |
|------|------|
| `prisma/schema.prisma` | `CustomerInvestStrategyHistory`에 `isRead` 추가, `messageId` nullable |
| `prisma/migrations/20260326000000_add_is_read_to_invest_strategy_history` | 단일 ALTER TABLE로 합침 |
| `customer/diag/dto/customer-diag.dto.ts` | `InvestStrategyPortfolioItemDto`, `InvestStrategyPortfolioResponseDto` 추가 |
| `customer/diag/customer-diag-invest.service.ts` | 신규 — 전체 계좌 조회, 읽음 마킹 |
| `customer/diag/customer-diag-invest.controller.ts` | 신규 — GET + PUT 엔드포인트 |
| `customer/customer.module.ts` | Controller, Service 등록 |

### 엔드포인트

- `GET :id/invest/strategy/last-portfolio/:consumer` — `JwtAuthGuard(ALLIANCE_JWT)`
- `PUT :betterId/invest/strategy/last-portfolio/mark-read` — `AllianceJwtAuthGuard`, body: `{ historyIds: number[] }`

### 주요 구현

- `ADVISORY_ACCOUNT_TYPES: ['IRP', 'ISA', 'PSA', 'BA']` — 클래스 멤버 상수
- `getCustomerInvestStrategyLastPortfolioAll`: `Promise.all`로 계좌 유형 병렬 조회
  - `consumer=BETTER` → `isRead=true` 제외
  - `consumer=FA` → 전체 반환
- `markCustomerInvestStrategyLastPortfolioAsRead`: `updateMany`로 일괄 처리

---

## Phase B: @betterapp 연동 ✅

### 수정/생성 파일

| 파일 | 작업 |
|------|------|
| `libs/https/src/qbis.https.service.ts` | `getCustomerContracts`, `getCustomerPendingContracts` 추가 |
| `libs/https/src/fa-be.https.service.ts` | 신규 (`getCustomerLastPortfolio`, `markCustomerLastPortfolioAsRead`) |
| `libs/https/src/constants/https.constants.ts` | `FA_BE_HTTP_SERVICE` 상수 추가 |
| `libs/https/src/https.module.ts` | `MongoDB`, `FaBeHttpsService` 등록 |
| `libs/https/src/index.ts` | `FaBeHttpsService` export |
| `libs/common/src/dto/invest-strategy.dto.ts` | 신규 공유 DTO |
| `main/asset/qbis/dto/qbis.dto.ts` | `BwaContractStatus` enum, Contract DTOs |
| `main/asset/qbis/home-invest-status.service.ts` | 신규 — 케이스 판단 로직 |
| `main/asset/qbis/home-invest-status.controller.ts` | 신규 — `GET main-asset/qbis/users/invest-contract/home-status` |
| `main/asset/qbis/qbis.module.ts` | 신규 모듈 |
| `main/asset/asset.module.ts` | `QbisModule` import |
| `alliance/firm/invest-strategy.service.ts` | 신규 — 조회 + 읽음 마킹 |
| `alliance/firm/invest-strategy.controller.ts` | 신규 — `GET alliance/firm/users/:betterId/...` |
| `alliance/firm/firm.module.ts` | `HttpsModule`, InvestStrategy 등록 |
| `alliance/firm/dto/invest-strategy.dto.ts` | 삭제 → `libs/common`으로 이동 |
| `alliance/firm/dto/bwa-contract-status.enum.ts` | 삭제 → `qbis/dto`로 이동 |

### 주요 설계 결정

| 항목 | 결정 | 이유 |
|------|------|------|
| Guard (@fa-be 조회) | `JwtAuthGuard(ALLIANCE_JWT)` | FA·BetterApp 공용 인증 |
| Guard (@fa-be mark-read) | `AllianceJwtAuthGuard` | @betterapp 전용 호출 |
| Guard (@betterapp qbis) | `AllianceSignatureAuthGuard` | qbis 서명 인증 전용 |
| contractStatus 비교 | `BwaContractStatus` enum (숫자 문자열 코드) | qbis API가 code로 응답 |
| T2 케이스 반환 | boolean 4개 (`showInvestProposal/NavAmount/NeedCheck/WellManaged`) | frontend UI 노출 조건 직접 반영 |
| 공유 DTO 위치 | `libs/common/src/dto/invest-strategy.dto.ts` | alliance/firm + main/asset/qbis 양쪽 사용 |
| 흐름 2 모듈 위치 | `main/asset/qbis` (신규 모듈) | @betterapp 홈 API는 main 앱 asset 하위 |
| 병렬화 | `Promise.allSettled` | 기계약/미결계약/투자제안 3개 동시 조회 |

---

## Phase C: @fa-be deeplink 파라미터 변경 ✅

### 수정 파일

| 파일 | 변경 내용 |
|------|-----------|
| `alarm.tok.template.type.ts` | 전 타입에 `queryParams` 필드 추가, `buildQueryString()` 정적 메서드 구현 |
| `alarm.tok.template.type.spec.ts` | `buildQueryString()` 검증 테스트 추가 |
| `environments/.env.*` (3개) | `KAKAO_MESSAGE_DIRECT_URL` 템플릿 수정 (`?{queryString}` 방식으로 변경) |
| `customer/message/customer.message.service.ts` | queryParams 기반 URL 빌드, `extraParams` 시그니처 추가, 트랜잭션 순서 변경 |
| `customer/message/customer.message.service.spec.ts` | `sendCustomerInvestmentStrategy` 통합 테스트 추가 |
| `customer/customer.common.service.ts` | `buildQueryString()` 호출로 동기화 |
| `prisma/schema.prisma` | `messageId` nullable (`BigInt?`) |

### 트랜잭션 순서 (sendCustomerInvestmentStrategy)

```
history 생성 (messageId=null)
    → historyId 확보
    → deeplink 생성 (historyId 포함)
    → 메시지 발송
    → history에 messageId 업데이트
```

### 주의사항

- `buildQueryString` 값 인코딩 미처리: 현재(숫자, 영문 코드)는 안전. 한글/특수문자 추가 시 `encodeURIComponent` 필요.
- `extraParams` spread 순서: `fa_id`, `fa_cust_id` 오버라이드 가능. 새 호출부 추가 시 주의.

---

## Phase D: 발송 가능 여부 판단 (미착수)

> spec.md 2차 개발 범위 참고

- FA 알림톡 발송 시 qbis 미결/기계약 조회 → accountType별 발송 가능 여부 판단
- 계약 완료/상태 변경 콜백 처리 엔드포인트

---

## 변경 이력

| 날짜 | 변경 내용 | 관련 파일 |
|------|-----------|-----------|
| 2026-03-23 | 최초 작성 | 전체 |
| 2026-03-25 | SPEC/PLAN 분리, 구현 계획으로 재작성 | 전체 |
| 2026-03-26 | Phase A 구현 완료 | `customer-diag-invest.*`, `schema.prisma` |
| 2026-03-27 | Phase B, C 구현 완료. deeplink 파라미터 변경 | 전체 |
| 2026-03-30 | 코드리뷰 반영 (I-1, S-1~3): ParseEnumPipe 추가, skipUrlCache 명칭 변경, 테스트 보강 | `customer-diag-invest.controller.ts`, `alarm.tok.template.type.ts`, `nshort.url.service.ts` |
