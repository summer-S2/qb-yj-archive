# enterBanner state 유실 문제 — Spec

## 배경

배너(TopBanner, 회원가입 완료, 딥링크)를 통해 투자진단 플로우에 진입하면 `enterBanner: true`를 `location.state`로 전달한다.
이 값에 따라 플로우 분기와 특정 UI(맞춤 투자진단 TIP) 노출 여부가 결정된다.

## 현재 enterBanner 진입점 (3곳)

| 파일 | 조건 |
|------|------|
| `src/features/MainHome/Banner/TopBanner.tsx` | CUSTOMER_TENDENCY / CUSTOMER_SYNC 배너 클릭 |
| `src/pages/renewal/auth/BetterSignUpComplete.tsx` | 회원가입 완료 후 |
| `src/hooks/useDynamicLink.tsx` | `invest-diag` 딥링크 |

## 현재 state 전달 체인

```
TopBanner (enterBanner: true)
  → investmentDiagnosis.intro (읽음 ✅, 다음으로 전달 ✅)
    → investmentDiagnosis.questions (읽음 ✅, 다음으로 전달 ✅)
      → investmentDiagnosis.loading (전달 ✅)
        → investmentDiagnosis.result (읽음 ✅)
          → investmentDiagnosis.connectAssets (❌ enterBanner 미전달)
            → mydata.transferRequest.selectOrgs (❌ enterBanner 미전달)
              → 인증 (Naver/Toss/Kakao/공인인증 — 네이티브 딥링크)
                → AssetConnectionController (❌ enterBanner를 알 수 없음)
                  → SignOrgsResult (❌)
                    → SignAssetsResult (❌ enterBanner 없음)
```

## 문제점

### 1. Result → ConnectAssets 전달 누락
`Result.tsx:72`에서 `navigate(connectAssets)` 호출 시 `enterBanner` state를 넘기지 않음.

### 2. ConnectAssets → selectOrgs 전달 누락
`ConnectAssets.tsx:40`에서 `navigate(selectOrgs)` 호출 시 state 없음.

### 3. 인증 단계에서 state 체인 단절 (핵심)
selectOrgs 이후 인증 과정(Naver/Toss/Kakao 등)은 네이티브 딥링크 콜백으로 동작.
인증 완료 후 `AssetConnectionController`가 Redux 상태를 감시하다가 **자체적으로** SignAssetsResult로 navigate한다.
이 시점에서 `location.state`에 `enterBanner`가 존재하지 않는다.

```
// AssetConnectionController.tsx:198
navigate(signAssets.result + '/' + signAssets.provider,
  { state: { selectedAccounts, isReconnection: true } }  // enterBanner 없음
);

// SignOrgsResult.tsx:200 (일반 연결 경로)
navigate(signAssets.result + '/' + providerType,
  { state: { selectedAccounts, isReconnection: reconnectAsset } }  // enterBanner 없음
);
```

## 요구사항

1. `SignAssetsResult.tsx`의 "맞춤 투자진단 TIP" 영역은 `enterBanner: true`일 때만 표출
2. 플로우 중간에 이탈하면 enterBanner 상태가 자동으로 사라져야 함 (= cleanup 불필요)

## 제약조건

- `location.state`는 이탈 시 자연 소멸이라 이상적이지만, 인증 단계에서 체인이 끊김
- Redux는 이탈 시 reset 타이밍을 잡기 어려움
