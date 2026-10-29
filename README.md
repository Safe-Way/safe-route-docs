# 문서

| 문서 | 내용 |
|---|---|
| [requirements.md](requirements.md) | 요구사항 분석서: 무엇을 만드는지, 기능·비기능 요구사항, 데이터, 경로 비용 모델, 평가 방법 |
| [CHANGELOG.md](CHANGELOG.md) | 요구사항 분석서 변경 이력: 무엇이 바뀌었고 왜 바꿨는지 |

원본은 Claude 문서 [세잎지도 요구사항 분석서](https://claude.ai/artifact/FQvgiTtePsoNKVNBhhHdTy)이고,
`requirements.md`는 그 원본을 Markdown으로 내보낸 것이다. 이 파일을 직접 고치지 않는다.

## 요구사항을 바꿀 때

1. 원본 문서를 고친다.
2. 원본을 Markdown으로 내보내 `docs/requirements.md`를 통째로 덮어쓴다.
3. [CHANGELOG.md](CHANGELOG.md) 표 맨 위에 한 줄 추가: 바뀐 곳, 무엇이, **왜**, 요청한 사람.
4. `feature/docs-…` 브랜치에서 커밋하고 `develop`으로 PR을 연다. PR 본문에도 왜 바꿨는지 적는다.

## 바뀐 내용 보는 법

- **한 번에 바뀐 것:** PR의 Files changed 탭. 문서 파일은 오른쪽 위 문서 아이콘(rich diff)을 누르면 표와 글이 렌더링된 상태로 비교된다.
- **지금까지 바뀐 순서:** `requirements.md` 화면 오른쪽 위 History.
- **왜 바뀌었나:** CHANGELOG.md의 "왜" 칸.
