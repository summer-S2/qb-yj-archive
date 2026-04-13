# qb-yj-archive

이 레포는 Claude와의 작업을 위한 문서 아카이브. 실제 코드는 아래 경로에 있음.

---

## 프로젝트 경로

| 아카이브 폴더 | 실제 코드 경로 |
|--------------|---------------|
| `better/` | `/Users/yj/Desktop/code/betterday-frontend` |
| `fa/` | `/Users/yj/Desktop/code/better-wealth-fa` |

작업 요청 시 해당 폴더의 아카이브 문서를 참고하고, 위 경로에서 코드 작업을 진행한다.

## 정책 문서

| 경로 | 설명 |
|------|------|
| `/Users/yj/Desktop/code/better-wealth-pb` | 프로젝트 토탈 정책 문서 |

요청 시 또는 참고 파일을 지정받았을 때만 읽는다. 매번 자동으로 읽지 않는다.

---

## 브랜치 작업 순서

1. 원격 브랜치 목록 fetch (`git fetch`)
2. 해당 이슈 브랜치 존재 여부 확인
   - 있으면 → 체크아웃
   - 없으면 → `feature/#이슈번호-기능명` 형식으로 새 브랜치 생성
