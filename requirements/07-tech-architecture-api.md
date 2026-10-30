<!-- 원본: https://claude.ai/artifact/FQvgiTtePsoNKVNBhhHdTy 의 "7~9. 기술·구조·API" 탭을 내보낸 것. 이 파일을 직접 고치지 말고 원본을 고친 뒤 다시 내보낸다. -->

## 7. 기술 스택 결정

서버는 **Java 21 + Spring Boot + MySQL**, 길찾기는 **JGraphT**, 앱은 **안드로이드(Android Studio, Kotlin) + 카카오맵 SDK**로 한다. 학교 서버가 MySQL이고 언어는 Java로 정했기 때문에, 그 안에서 직접 고칠 수 있는 것을 골랐다. 처음에는 웹(Leaflet + PWA)을 추천했지만, SOS 경보음·문자·화면이 꺼져도 위치 받기가 필요해 안드로이드 앱으로 정했다(결정 10·30번).

| 영역 | 결정 | 이유 | 검토한 대안 |
| --- | --- | --- | --- |
| 언어 | Java 21 (Temurin) | 사용자 결정 | Python |
| 프레임워크 | Spring Boot 4.1.2 (Web MVC) | 앱이 부르는 REST API를 제공 (8.3) | — |
| 빌드·도구 | Maven(래퍼 포함), IntelliJ | 사용자 결정 | Gradle |
| DB | MySQL 8.0.41 (학교 서버) · 8.4 (로컬 개발용) | 학교 서버가 MySQL 8.0이고 로컬에는 8.4를 깔음. 쓰는 기능(공간 인덱스, JSON 가상 칸)은 두 버전에서 같게 동작. 공간 인덱스와 `ST_Distance_Sphere` 동작 확인 | PostgreSQL + PostGIS (기능은 더 많지만 서버가 없음) |
| DB 접근 | JdbcTemplate | 수백만 건 적재와 공간 SQL을 직접 쓰기 쉬움 | JPA |
| 길찾기 | JGraphT (다익스트라·A\*) | 비용 함수를 마음대로 바꾸고, 경로 위 시설을 세기 쉬움 | GraphHopper (빠르지만 상황별 비용·설명을 붙이기 어려움), pgRouting (PostgreSQL 필요) |
| 전국 길찾기 구조 | 길·감점은 DB에, 요청 때 출발·도착 주변 범위만 읽어 그래프 생성 | 전국 그래프를 메모리에 두지 않아도 됨. 도보는 거리가 짧아 범위가 작음 | 전국 그래프 상주 |
| 도로망 | OpenStreetMap 대한민국 PBF, osm4j로 읽음 | 보행길·계단·덮개 정보가 있고 무료 | 국가교통DB 보행네트워크 (지역이 제한적) |
| 공간 계산 | JTS (버퍼, STRtree), 좌표 변환은 공식 직접 구현 | 시설→길 결합을 메모리에서 빠르게 | Proj4J (EPSG:3857·4326만 쓰므로 불필요) |
| 일몰 계산 | commons-suncalc | 위도·경도·날짜만으로 계산, API 호출 없음 | 한국천문연구원 API |
| 화면 | 안드로이드 앱, Android Studio + Kotlin으로 개발 | SOS 경보음·문자·잠금 화면 버튼·화면이 꺼져도 위치 받기는 앱이어야 됨. 팀 결정(결정 10·30번). 앱 언어는 Kotlin(2026-10-30 확정) | Leaflet 웹 + PWA (처음 추천), Capacitor |
| 배경 지도 | 카카오맵 Android SDK (일반·위성) | 국내 지도 품질, 위성 지도 제공, 장소 검색도 카카오 | OpenStreetMap 타일, VWorld |
| 장소 검색 | 카카오 로컬 API (서버가 대신 호출) | 한국 장소 검색 품질, 하루 10만 번 무료 | 네이버 검색 API |
| 데이터 내려받기 | Python 3 표준 라이브러리 스크립트 | 가끔 돌리는 도구라 설치 없이 실행. 적재는 Java | Java로 통일 |
| 테스트 | JUnit 5 + Spring Boot Test | 기본 포함 | — |
| 버전 관리 | git + GitHub 비공개 저장소 2개: 코드는 safe-route, 요구사항 문서는 safe-route-docs (2026-10-30에 나눔) | `.env`·원본 데이터는 제외 | — |

## 8. 시스템 구조

무거운 일(내려받기, 적재, 길마다 감점 계산)은 **데이터를 갱신할 때 한 번**만 하고, 경로 요청 때는 출발·도착 주변의 길과 감점만 읽어 가중치를 곱해 계산한다.

> 그림: 시스템 구성 · 준비 단계와 요청 단계 ([원본 문서](https://claude.ai/artifact/FQvgiTtePsoNKVNBhhHdTy)에서 볼 수 있음)

앱은 서버하고만 통신하고, 인증키가 필요한 외부 API는 서버가 대신 부른다. 앱이 부르는 서버 주소는 8.3에 있다.

### 8.1 경로 요청 처리 순서

1. 출발·도착 좌표를 받는다 (장소 이름이면 카카오로 좌표를 찾는다).
2. 일몰 계산·기상청·에어코리아로 상황을 판단하고 가중치를 정한다 (1차는 일몰 계산으로 밤만, 비·폭염 등은 2차).
3. 두 점을 감싼 사각 범위를 사방 500m 넓혀 그 안의 길·감점을 DB에서 읽는다.
4. JGraphT로 최단 경로와 상황 경로를 계산한다 (최대 우회 1.3배 규칙 적용).
5. 두 경로가 지나는 시설을 세어 차이 설명을 만든다.
6. 경로 선, 시설 위치, 설명문을 JSON으로 돌려준다.

### 8.2 DB 표 설계

| 표 | 내용 | 주요 칸 | 만든 때 |
| --- | --- | --- | --- |
| 시설 표 10개 (`streetlight`, `security_light`, `cctv`, `police`, `emergency_bell`, `convenience_store`, `child_safety_house`, `heat_shelter`, `cold_shelter`, `public_toilet`) | 시설 한 개가 한 줄. 종류마다 표 하나 (2026-10-29에 한 표 `facility`에서 나눔) | kind, source, source\_id, name, address, sido\_code, sgg\_code, lon, lat, geom(공간 인덱스), attrs(JSON), 종류별 칸(예: lamp\_count, camera\_count) | 만듦 |
| facility\_all (뷰) | 시설 표 10개를 한꺼번에 볼 때. 큰 표를 찾을 때는 공간 인덱스를 타도록 종류별 표에 직접 묻는다 | type(표 이름)과 공통 칸 | 만듦 |
| `data_source` | 출처별 적재 현황 | type, source, total\_count, loaded\_count, skipped\_count, complete, loaded\_at | 만듦 |
| `road_node` | 길이 만나는 점 | id(OSM 번호), lon, lat, geom | 2주차 |
| `road_edge` | 두 점 사이의 길 | id, from\_node, to\_node, length\_m, highway, covered, steps, geom(선) | 2주차 |
| `edge_penalty` | 길마다 미리 계산한 감점 | edge\_id, light, cctv, help, cover, stairs, heat, cold, traffic, data\_gap | 2주차 |
| accident\_zone (추가안) | 사고다발 영역 한 곳이 한 줄 (FR-44) | kind(노인·어린이), year, 사고 건수, geom(다각형, 공간 인덱스). 길마다 영역 안인지는 edge\_penalty에 미리 계산 | 추가안 |
| user\_place (추후) | 사용자가 제보한 장소 (FR-45 2단계). 1단계는 휴대폰에만 저장하므로 표가 필요 없음 | kind, name, lon, lat, geom, 제보 수, 확인 여부, 신고 수 | 추후 (추가안) |

좌표는 WGS84 경도·위도를 `POINT(경도, 위도)`로 넣고(SRID 0) 거리는 `ST_Distance_Sphere`로 잰다. MySQL의 SRID 4326은 위도·경도 순서라 실수하기 쉽기 때문이다.

### 8.3 앱↔서버 API (1차)

1차에서 앱이 부르는 서버 주소는 3개다. SOS·경보음·보호자 문자는 휴대폰 안에서 끝나므로 서버 API가 없다. 걷는 동안의 위치도 서버에 보내지 않는다(NFR-07). 모든 주소는 `/api`로 시작하고 JSON(UTF-8)으로 주고받는다. 좌표는 `lat`(위도)·`lon`(경도)로 이름을 붙여 쓰고, 경로 선(`geometry`)만 GeoJSON 순서인 `[경도, 위도]` 배열이다.

| 주소 | 하는 일 | 관련 FR | 만드는 파트 |
| --- | --- | --- | --- |
| `GET /api/places` | 장소 이름·주소 검색. 서버가 카카오 키워드 검색을 대신 부름 | FR-01 | C |
| `POST /api/route` | 출발·도착 → 최단·안전 경로, 차이 설명, 경로 주변 시설 | FR-02\~07 | B(계산), C(주소) |
| `GET /api/health` | 서버·DB 상태와 데이터 기준일. 시연 전 점검용 | — | C |
| POST /api/explain (3차) | 기본 설명 문장을 LLM으로 다듬어 돌려줌. 실패하면 기본 문장 | FR-43 | C |

**장소 검색** `GET /api/places?query=조선대학교&lat=35.1401&lon=126.9336`

- `query`는 필수(2자 이상). `lat`·`lon`은 선택이고, 주면 가까운 순으로 준다.
- 결과는 최대 10개다.

```json
{ "places": [
  { "name": "조선대학교", "address": "광주 동구 필문대로 309", "lat": 35.1401, "lon": 126.9336, "distanceM": 0 }
] }
```

- 카카오가 실패하면 `503`과 `SEARCH_UNAVAILABLE`을 준다. 앱은 "검색이 잠시 안 됩니다. 지도를 길게 눌러 고를 수 있어요"를 띄운다(4.2).

**경로 계산** `POST /api/route`

요청:

```json
{
  "origin":      { "lat": 35.1468, "lon": 126.9195, "name": "충장로" },
  "destination": { "lat": 35.1401, "lon": 126.9336, "name": "조선대 후문" },
  "night": "auto"
}
```

- `night`: `auto`(기본, 일몰 계산), `on`, `off` 중 하나(FR-03). 낮에 시연할 때는 `on`으로 보낸다.
- `mode`: `child`(어린이·청소년)·`elder`(노인)·`wheelchair`(휠체어)·`basic`(기본, 그 밖의 보행자) 중 하나. 1차부터 보낸다. 없으면 `basic`.

응답 (좌표와 시설 목록은 줄인 예):

```json
{
  "conditions": { "night": true, "nightSource": "auto", "sunset": "17:41" },
  "routes": [
    { "kind": "shortest", "distanceM": 1820, "durationMin": 23,
      "geometry": [[126.9195, 35.1468], [126.9201, 35.1462]],
      "counts": { "cctv": 8, "light": 41, "bell": 1, "help": 2 } },
    { "kind": "safe", "distanceM": 2000, "durationMin": 25,
      "geometry": [[126.9195, 35.1468], [126.9210, 35.1470]],
      "counts": { "cctv": 20, "light": 71, "bell": 4, "help": 4 } }
  ],
  "explanation": {
    "text": "180m(2분) 더 돌지만 CCTV +12대, 가로등 +30개, 비상벨 +3개",
    "extraM": 180, "extraMin": 2,
    "diff": { "cctv": 12, "light": 30, "bell": 3, "help": 2 }
  },
  "facilities": [
    { "type": "CCTV", "lat": 35.1455, "lon": 126.9220, "on": "safe" }
  ],
  "dataQuality": { "status": "OK", "message": null, "dataDate": "2026-10-28" }
}
```

| 필드 | 뜻 |
| --- | --- |
| `routes[].kind` | `shortest`(지도에 회색) 또는 `safe`(초록). 두 경로가 같으면 `safe` 하나만 오고 설명은 "가장 짧은 길이 가장 안전한 길입니다" |
| `distanceM`, `durationMin` | 거리(m)와 걸리는 시간(분). 시간은 모드별 속도(기본 분당 80m, 어린이 70m, 노인·휠체어 60m)로 계산 |
| `counts` | 경로에서 결합 반경(결정 12번) 안에 있는 시설 수. `light`는 가로등+보안등, `bell`은 비상벨, `help`는 경찰·편의점·아동안전지킴이집 |
| `explanation` | 차이 설명 문장(FR-06)과 그 근거 숫자. 앱은 `text`를 그대로 보여 준다 |
| `facilities[]` | 지도 아이콘용 시설(FR-07). `on`은 `safe`·`shortest`·`both`. 최대 300개 |
| `dataQuality.status` | `OK`, `PARTIAL`(경로 일부가 데이터 없는 곳), `NO_DATA` 중 하나. `OK`가 아니면 앱은 "이 지역은 안전 데이터가 부족합니다"를 띄우고 안전하다고 말하지 않는다(NFR-04) |

오류는 `{ "code": "TOO_FAR", "message": "…" }` 형식으로 준다.

| HTTP | code | 언제 | 앱에 띄우는 말 |
| --- | --- | --- | --- |
| 400 | `SAME_POINT` | 출발과 도착이 50m 안 | "출발과 도착이 너무 가까워요" |
| 400 | `TOO_FAR` | 직선거리 5km 넘음(설정값) | "걸어가기엔 멀어요" (대중교통은 3차, FR-37) |
| 422 | `OUT_OF_AREA` | 도로망을 넣지 않은 지역 (1차는 광주만) | "아직 광주만 길찾기가 됩니다" |
| 404 | `NO_ROUTE` | 도로망에서 두 점이 이어지지 않음 | "경로를 찾지 못했어요" |
| 503 | `DB_UNAVAILABLE` | DB 연결 실패 | "잠시 후 다시 시도해 주세요" |

응답 시간 목표는 직선 3km 안에서 2초다(0.3). 서버 안의 처리 순서는 8.1을 따른다.

**서버 상태** `GET /api/health` → `{ "status": "UP", "db": "UP", "dataDate": "2026-10-28", "roadArea": "광주" }`

**2차에 늘어나는 것.** 요청에 `safeBoost`(안심 강화)가 붙는다. 응답의 `facilities[]`마다 경로의 몇 m 지점·왼쪽/오른쪽(FR-30)이 붙는다. 다시 안내(FR-35)는 새 주소 없이 같은 `/api/route`를 현재 위치로 다시 부른다. 경로마다 `arriveAt`(도착 예상 시각, FR-01-2)은 1차부터 넣는다. 비·폭염 등 다른 상황의 근거값은 3차다. 앱은 모르는 필드를 무시하므로 필드가 늘어도 깨지지 않는다.

**3차: 설명 다듬기** `POST /api/explain` (FR-43, 3.5). 앱은 `/api/route`의 기본 문장을 먼저 보여 준 뒤 이 주소를 따로 부른다. 요청 본문은 `/api/route` 응답의 `conditions`·`explanation`·`dataQuality`를 그대로 보낸다(좌표와 장소 이름은 빼고). 서버는 상태를 남기지 않고, LLM을 3초까지만 기다린다.

```json
{ "text": "곧 해가 지니까, 120m만 더 걸으면 …", "source": "llm" }
```

`source`는 `llm`(검사를 통과한 다듬은 문장) 또는 `template`(LLM 실패·3초 초과·검사 탈락으로 기본 문장을 그대로 돌려줌)다. 앱은 `llm`일 때만 카드의 문장을 바꾼다. 이 주소가 멈춰도 경로 화면은 기본 문장으로 그대로 동작한다.

## 9. 외부 API 명세

TMAP을 뺀 모든 API는 실제 호출까지 확인했다. 카카오 도보 길찾기는 제휴 파트너 전용이라 우리 키로는 403(permission denied)이 나와 TMAP으로 바꿨다. 기상청·에어코리아 한도는 개발계정 기준 대략값이다. 키 값은 `.env`에만 있고, 아래는 `.env`의 키 이름이다.

| API | 주소 | 필수 값 | 한도 | `.env` 키 |
| --- | --- | --- | --- | --- |
| 기상청 초단기실황 | `apis.data.go.kr/1360000/VilageFcstInfoService_2.0/getUltraSrtNcst` | `base_date`, `base_time`(매시 정각, 40분 뒤 제공), `nx`, `ny`, `dataType=JSON` | 개발계정 하루 1만 번 | `DATA_GO_KR_KEY` |
| 기상청 초단기예보 | `…/getUltraSrtFcst` | `base_time`(매시 30분, 45분 뒤 제공), 6시간치 | 같음 | `DATA_GO_KR_KEY` |
| 에어코리아 시도별 실시간 | `apis.data.go.kr/B552584/ArpltnInforInqireSvc/getCtprvnRltmMesureDnsty` | `sidoName`, `ver=1.0`, `returnType=json` | 개발계정 하루 500번 | `DATA_GO_KR_KEY` |
| 재난안전데이터 경찰 | `www.safetydata.go.kr/V2/api/DSSP-IF-00080` | `serviceKey`, `pageNo`, `numOfRows`(최대 1,000), `returnType=json` | 키당 하루 1,000번, 2027-10-28 만료 | `SAFETYDATA_POLICE_KEY` |
| 재난안전데이터 보안등 | `…/DSSP-IF-00083` | 같음 | 같음 | `SAFETYDATA_SECURITYLIGHT_KEY` |
| 재난안전데이터 가로등 | `…/DSSP-IF-00084` | 같음 | 같음 | `SAFETYDATA_STREETLIGHT_KEY` |
| 생활안전지도 데이터 | `www.safemap.go.kr/openapi2/IF_xxxx` | `serviceKey`, `pageNo`, `numOfRows`(최대 1,000), `returnType=JSON` | 하루 1,000번 | `SAFEMAP_KEY` |
| 생활안전지도 그림(WMS) | `www.safemap.go.kr/openapi2/IF_xxxx_WMS` | `srs`, `bbox`, `width`, `height`, `format`, `transparent` | 하루 1,000번 | `SAFEMAP_KEY` |
| 카카오 키워드·주소 검색 | `dapi.kakao.com/v2/local/search/keyword.json` | 요청 머리글 `Authorization: KakaoAK {키}`, `query` | 하루 10만 번 무료, 넘으면 429 | `KAKAO_REST_KEY` |
| TMAP 보행자 경로 (비교용) | TMAP routes/pedestrian (SK open API) | 출발·도착 좌표 | 키 발급 후 확인 | `TMAP_APP_KEY` |
| ODsay LAB 대중교통 길찾기 (FR-37) | `api.odsay.com/v1/api/searchPubTransPathT` (노선 선 좌표는 `loadLane`) | `apiKey`, `SX`·`SY`(출발 경도·위도), `EX`·`EY`(도착 경도·위도) | 키 발급 후 확인. 서버용 키는 호출하는 서버 IP 등록이 필요할 수 있음 | `ODSAY_API_KEY` |
| LLM 설명 다듬기 (FR-43, 3차) | 정할 것 | 사실 목록(숫자·상황·모드·부족한 데이터 종류)과 기본 문장. 좌표·장소 이름은 보내지 않음 | 정할 것. 3초 안에 답이 없으면 기본 문장. 같은 사실은 저장해 둔 문장을 다시 씀 | 정할 것 (서버 .env에만) |

**기상청 격자:** 위도·경도를 람베르트 원추 투영 공식으로 nx·ny에 바꾼다. 조선대는 nx=60, ny=74다.

**실패 처리:** 외부 API는 3초 안에 답이 없으면 포기하고 마지막으로 저장해 둔 값을 쓴다. 저장한 값도 없으면 그 상황은 "정보 없음"으로 둔다(NFR-03).
