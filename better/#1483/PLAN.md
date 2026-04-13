# enterBanner state 전달 — Plan

## 접근 방식 (미확정)

`location.state`만으로는 인증 단계(네이티브 딥링크 콜백)에서 체인이 끊기므로, 아래 옵션 중 선택 필요.

### Option A: Redux 하이브리드

state로 전달 가능한 구간은 state 사용, 끊기는 구간만 Redux로 브릿지.

**수정 파일:**

| 파일 | 변경 내용 |
|------|-----------|
| `redux/slices/app.ts` 또는 `redux/slices/assetConnection.ts` | `enterBanner: boolean` 필드 추가 |
| `Result.tsx:72` | `navigate(connectAssets, { state: { enterBanner } })` — state 전달 추가 |
| `ConnectAssets.tsx` | state에서 `enterBanner` 읽고, selectOrgs navigate 시 전달 + **Redux에 dispatch** |
| `AssetConnectionController.tsx:198,206` | Redux에서 `enterBanner` 읽어서 navigate state에 포함 |
| `SignOrgsResult.tsx:200` | 동일하게 enterBanner state 포함 |
| `SignAssetsResult.tsx` | state에서 `enterBanner` 읽어서 TIP 조건부 렌더링 + **Redux reset** |

**장점:** 이탈 시 Redux reset이 SignAssetsResult에서만 일어나면 됨
**단점:** 이탈 시 Redux에 enterBanner가 남아있을 수 있음 → 다음 진입 시 오작동 가능

**cleanup 방안:**
- ConnectAssets unmount 시 reset
- 또는 홈 화면 진입 시 reset

### Option B: assetConnection slice 활용

기존 `assetConnection` Redux slice에 `enterBanner`를 추가. 이 slice는 자산연결 플로우 전용이라 의미적으로 적합.
`resetAssetConnectionState()` 호출 시 자동으로 함께 초기화됨 (SignAssetsResult useEffect에서 이미 호출 중).

**수정 파일:**

| 파일 | 변경 내용 |
|------|-----------|
| `redux/slices/assetConnection.ts` | `enterBanner: boolean` 추가, `resetAssetConnectionState`에서 자동 초기화 |
| `Result.tsx:72` | navigate 시 state 전달 |
| `ConnectAssets.tsx` | state 읽고 → `dispatch(setEnterBanner(true))` |
| `AssetConnectionController.tsx` | Redux에서 읽어서 state에 포함 |
| `SignOrgsResult.tsx` | 동일 |
| `SignAssetsResult.tsx` | state에서 읽고 TIP 조건부 렌더링. `resetSignAssets()` 시 자동 cleanup |

**장점:** 기존 reset 로직(`resetAssetConnectionState`)에 자연스럽게 편승 → cleanup 문제 해소
**단점:** assetConnection slice의 역할이 살짝 확장됨

### Option C: ConnectAssets에만 TIP 표출

SignAssetsResult까지 전달하지 않고, ConnectAssets 페이지 자체에 TIP UI를 넣는 방법.

**장점:** 수정 범위 최소 (Result.tsx + ConnectAssets.tsx 2개만)
**단점:** 디자인 의도와 다를 수 있음 (자산연결 완료 후 TIP이 더 적절한 위치일 수 있음)

## 미결정 사항

- [ ] 어떤 Option으로 진행할지
- [ ] TIP UI가 ConnectAssets / SignAssetsResult 중 어디에 노출되어야 하는지
