# 세잎지도 요구사항 분석서

요구사항 분석서는 탭(파일) 8개로 나뉘어 있다. 처음 읽는다면 [0. 한눈에 보기](requirements/00-overview.md)부터 본다.

| 파일 | 내용 |
|---|---|
| [0. 한눈에 보기](requirements/00-overview.md) | 무엇을 만드는지 한 문장, 대상(어린이·노인·장애인)과 주요 기능 3가지, 시연 순서, 사용자별 기능, 결과물, 단계별 범위(1·2·3차), 아직 정할 것 |
| [1~2. 개요·사용 상황](requirements/01-intro-and-usage.md) | 개요, 제외 범위(넣고 싶었지만 못 넣은 기능 포함), 상황별 사용 |
| [3. 기능 요구사항](requirements/03-functional.md) | FR 목록(1·2차)과 나중에 할 것(3차·보류·뺌), 사용자 모드, 귀가 지킴이, 걷는 동안 안내, 1차 화면 흐름 |
| [4. 비기능 요구사항](requirements/04-nonfunctional.md) | NFR 목록, 전국 사용 조건, 외부 서비스 장애 대응, 실시간 구현 문제 |
| [5~6. 데이터·경로 비용](requirements/05-data-and-cost-model.md) | 데이터 목록(받은 곳·DB 건수·DB 표), 받기와 적재, 경로 비용 모델(기준 항목, 모드별 기준, 규칙, 쓰지 않는 것) |
| [7~9. 기술·구조·API](requirements/07-tech-architecture-api.md) | 기술 스택, 시스템 구조와 구현 상태, DB 표(묶음별)와 ERD, 앱↔서버 API(1차)와 연결 방법, 외부 API 명세 |
| [10~12. 평가·일정·결정](requirements/10-evaluation-schedule-decisions.md) | 평가 방법, 일정과 위험, 결정 기록(한 줄씩), 열린 질문 |
| [13. 부록](requirements/13-appendix.md) | A. 경로 엔진 기준을 정한 시험 결과와 결정 과정, B. 카카오맵 키 넣는 방법과 앱 설정 |

원본은 Claude 문서 [세잎지도 요구사항 분석서](https://claude.ai/artifact/FQvgiTtePsoNKVNBhhHdTy)이고, 위 파일들은 원본의 탭을 하나씩 Markdown으로 내보낸 것이다. 무엇이 왜 바뀌었는지는 [CHANGELOG.md](CHANGELOG.md)에 있다.
