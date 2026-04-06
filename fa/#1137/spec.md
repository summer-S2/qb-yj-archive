# 투자제안 다중계좌 — #1137

## 개요

FA가 고객에게 투자 제안(알림톡)을 발송할 때 IRP 외 ISA, 연금저축(PSA), 종합위탁(BA) 계좌 타입을 추가 지원.
BetterApp 홈에서 고객의 투자 상태(제안/미결/기계약)를 통합 조회해 케이스별 UI를 제공.

---

## 계좌 타입 (accountType)

- as-is: IRP
- to-be: IRP + ISA + PSA(연금저축) + BA(종합위탁)

---

## 시스템 구성

```
FA플랫폼 (@fa-be)
    ↕ (모든 호출은 BetterApp 경유)
BetterApp (@betterapp)
    ↕
투자플랫폼 (@qbis)
```

> ⚠️ 투자플랫폼 ↔ FA플랫폼 간 직접 호출 금지. 반드시 BetterApp 경유.

---

## 기능 요구사항

### FA플랫폼 — 투자 제안 알림톡 발송

- FA가 단일 accountType 선택 후 발송 요청
- deeplink 파라미터: `historyId`, `portfolioCode`, `accountType`
- 발송 가능 여부 체크 (2차 개발):
  - IRP / ISA: 운용중(`OPERATION`) 상태면 발송 불가
  - PSA / BA: 항상 발송 가능

### FA플랫폼 — 투자 제안 발송 히스토리 조회

- as-is: 단일 accountType 단건 조회
- to-be: 전체 accountType 배열 반환
  - `consumer=BETTER`: `isRead=false` 미읽음 필터 적용
  - `consumer=FA`: 필터 없음 (전체 조회)

### BetterApp 홈 — 투자 상태 통합 케이스 판단

기계약/미결계약/투자제안을 병렬 조회 후 merge해 케이스 판단:

| 케이스 | 조건 | 메시지 | 평가금액 표시 |
|--------|------|--------|--------------|
| a | 제안만 있음 | 맞춤 포트폴리오가 도착했어요 | N |
| b | 미결계약 1개 이상, 기계약 없음 | 확인 필요한 사항이 있어요 | N |
| c | 기계약 + 미결계약 모두 존재 | 확인 필요한 사항이 있어요 | Y |
| d | 운용중 아닌 기계약 1개 이상 | 확인 필요한 사항이 있어요 | Y |
| e | 모든 기계약 운용중 | 투자 자산이 잘 운용되고 있어요 | Y |

---

## 전체 플로우

### 흐름 1 — FA 알림톡 발송 (1차)

```
FA → @fa-be: 발송 요청 { portfolioCode, accountType }
    → 발송 히스토리 저장 (historyId 확보)
    → 알림톡 발송 (deeplink: historyId, portfolioCode, accountType)
    → 고객에게 알림톡 전달
```

### 흐름 2 — BetterApp 홈 진입 (경로 A: deeplink)

```
고객 알림톡 클릭
    → @betterapp bridge
    → @qbis 쿼터백 홈 (historyId, portfolioCode, accountType 전달)
```

### 흐름 2 — BetterApp 홈 진입 (경로 B: 홈)

```
고객 홈 진입
    → 병렬 조회:
        @qbis  기계약 조회
        @qbis  미결계약 조회
        @fa-be 투자제안 목록 조회 (isRead 필터)
    → 케이스 판단 → UI 노출
    → 고객 쿼터백 진입
        → @qbis 약관 동의 처리
        → @fa-be 투자제안 조회 + isRead=true 마킹
```

---

## 개발 범위

### 1차 개발 (완료)
- Phase A: @fa-be — 다중계좌 조회 API, isRead 마킹 API
- Phase B: @betterapp — FaBeHttpsService, 홈 상태 통합 API, 투자제안 조회 API
- Phase C: @fa-be — deeplink queryParams 기반 URL 빌드 구조 변경

### 2차 개발 (미착수)
- Phase D: 발송 가능 여부 판단 (계약 상태 기반)
- Phase E: 계약 완료/상태 변경 콜백 처리

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|-----------|
| 2026-03-23 | 최초 작성 |
| 2026-03-25 | SPEC/PLAN 문서 분리 |
| 2026-03-26 | Phase A 구현 완료 기준으로 내용 확정 |
| 2026-03-27 | Phase C deeplink 파라미터 변경 반영 |
