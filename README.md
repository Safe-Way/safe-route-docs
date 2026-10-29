# 문서

| 문서 | 내용 |
|---|---|
| [requirements.md](requirements.md) | 요구사항 분석서 목차 (탭 7개로 나뉜 파일 목록) |
| [requirements/](requirements/) | 요구사항 분석서 본문: 원본 문서의 탭 하나가 파일 하나 |
| [CHANGELOG.md](CHANGELOG.md) | 요구사항 분석서 변경 이력: 무엇이 바뀌었고 왜 바꿨는지 |

원본은 Claude 문서 [세잎지도 요구사항 분석서](https://claude.ai/artifact/FQvgiTtePsoNKVNBhhHdTy)이고,
`requirements/` 폴더의 파일은 원본의 탭을 하나씩 Markdown으로 내보낸 것이다. 이 파일들을 직접 고치지 않는다.

## 요구사항을 바꿀 때

1. 원본 문서를 고친다.
2. 바뀐 탭을 Markdown으로 내보내 `docs/requirements/` 안의 해당 파일을 통째로 덮어쓴다. 탭이 늘거나 이름이 바뀌면 `docs/requirements.md` 목차도 고친다.
3. [CHANGELOG.md](CHANGELOG.md) 표 맨 위에 한 줄 추가: 바뀐 곳, 무엇이, **왜**, 요청한 사람.
4. `feature/docs-…` 브랜치에서 커밋하고 `develop`으로 PR을 연다. PR 본문에도 왜 바꿨는지 적는다.

## 바뀐 내용 보는 법

- **한 번에 바뀐 것:** PR의 Files changed 탭. 문서 파일은 오른쪽 위 문서 아이콘(rich diff)을 누르면 표와 글이 렌더링된 상태로 비교된다.
- **지금까지 바뀐 순서:** `requirements/` 안 각 파일 화면 오른쪽 위 History. 탭으로 나누기 전(v5까지)의 기록은 `requirements.md`의 History에 있다.
- **왜 바뀌었나:** CHANGELOG.md의 "왜" 칸.
