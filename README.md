# qb-yj-archive

Claude와의 작업을 위한 문서 아카이브. 실제 코드는 여기 없고, 작업 가이드와 이슈별 기획/구현 문서를 관리한다.

---

## 폴더 구조

```
qb-yj-archive/
├── CLAUDE.md          # Claude용 프로젝트 경로 안내
├── README.md          # 이 파일
├── better/            # betterday-frontend 관련 문서
│   ├── CLAUDE.md      # 프로젝트 전체 설정 (아키텍처, 개발 커맨드 등)
│   ├── @ME.md         # 개인 설정 (응답 규칙, 코드 컨벤션 등)
│   ├── CHECKLIST.md   # 이슈 작업 가이드 (체크리스트, 패턴 모음)
│   ├── templates/
│   │   ├── SPEC.md    # 기획 문서 템플릿
│   │   └── PLAN.md    # 구현 계획 템플릿
│   └── #이슈번호/
│       ├── SPEC.md    # 이슈별 기획 문서
│       └── PLAN.md    # 이슈별 구현 계획
└── fa/                # better-wealth-fa 관련 문서 (추가 예정)
```

## 실제 코드 위치

| 문서 폴더 | 코드 경로 |
|-----------|-----------|
| `better/` | `/Users/yj/Desktop/code/betterday-frontend` |
| `fa/` | `/Users/yj/Desktop/code/better-wealth-fa` |

---

## 이슈 작업 시작하는 법

1. `better/CHECKLIST.md` 열고 체크리스트 확인
2. `#이슈번호/` 폴더 생성
3. `templates/SPEC.md` 형식 참고해서 `SPEC.md` 작성
4. `templates/PLAN.md` 형식 + 해당 이슈의 `SPEC.md` 내용 참고해서 `PLAN.md` 작성
5. Claude에게 코드 작업 요청
