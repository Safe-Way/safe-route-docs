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
| 도로망 | OpenStreetMap. 시연 지역은 Overpass API로 받아 Python 스크립트로 넣음(2026-11-02). 전국은 대한민국 PBF를 osm4j로 읽을 예정 | 보행길·계단·덮개 정보가 있고 무료 | 국가교통DB 보행네트워크 (지역이 제한적) |
| 공간 계산 | JTS (버퍼, STRtree), 좌표 변환은 공식 직접 구현 | 시설→길 결합을 메모리에서 빠르게 | Proj4J (EPSG:3857·4326만 쓰므로 불필요) |
| 일몰 계산 | commons-suncalc | 위도·경도·날짜만으로 계산, API 호출 없음 | 한국천문연구원 API |
| 화면 | 안드로이드 앱, Android Studio + Kotlin으로 개발 | SOS 경보음·문자·잠금 화면 버튼·화면이 꺼져도 위치 받기는 앱이어야 됨. 팀 결정(결정 10·30번). 앱 언어는 Kotlin(2026-10-30 확정) | Leaflet 웹 + PWA (처음 추천), Capacitor |
| 배경 지도 | 카카오맵 Android SDK (일반·위성) | 국내 지도 품질, 위성 지도 제공, 장소 검색도 카카오 | OpenStreetMap 타일, VWorld |
| 장소 검색 | 카카오 로컬 API (서버가 대신 호출) | 한국 장소 검색 품질, 하루 10만 번 무료 | 네이버 검색 API |
| 데이터 내려받기 | Python 3 표준 라이브러리 스크립트 | 가끔 돌리는 도구라 설치 없이 실행. 시설 적재는 Java, 도로망·행정구역 경계·edge\_stat은 Python이 SQL을 만들어 mysql로 넣음(2026-11-02) | Java로 통일 |
| 테스트 | JUnit 5 + Spring Boot Test | 기본 포함 | — |
| 버전 관리 | git + GitHub 비공개 저장소 3개: 서버 코드는 safe-route, 안드로이드 앱은 safe-route-android(패키지 com.deungbul.anshimgil, Jetpack Compose, minSdk 26), 요구사항 문서는 safe-route-docs (2026-10-30에 나눔) | `.env`·원본 데이터는 제외 | — |

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
| 시설 표 3개 더 (footbridge, crosswalk, child\_protection\_zone) | 육교·횡단보도·어린이보호구역. 시설 표와 같은 모양이지만 facility\_all 뷰에는 넣지 않음 | 공통 칸 + 원본 항목(육교 장애인 편의시설, 횡단보도 보도턱 낮춤·음향신호기, 보호구역 도로폭·CCTV 등) | 만듦 (2026-10-30\~11-02) |
| facility\_all (뷰) | 시설 표 10개를 한꺼번에 볼 때. 큰 표를 찾을 때는 공간 인덱스를 타도록 종류별 표에 직접 묻는다 | type(표 이름)과 공통 칸 | 만듦 |
| `data_source` | 출처별 적재 현황 | type, source, total\_count, loaded\_count, skipped\_count, complete, loaded\_at | 만듦 |
| `road_node` | 길이 만나는 점 | id(OSM 번호), lon, lat, geom | 만듦 (2026-11-02) |
| `road_edge` | 두 점 사이의 길 | id, from\_node, to\_node, length\_m, highway, covered, steps, wheelchair\_ramp(계단의 휠체어 경사로: 1 있음·0 없음·NULL 모름. OSM ramp:wheelchair·ramp로 DB가 계산, 2026-11-03 추가·학교 DB 반영. 시연 범위 계단 27개 중 있음 0·없음 11·모름 16), geom(선) | 만듦 (2026-11-02) |
| `edge_stat` | 길마다 감점의 원값. 0\~1 감점은 서버가 route.yml 기준값으로 만듦(6.5). edge\_penalty에서 이름을 바꿈 | edge\_id, light\_n·light\_cover\_m, cctv\_n·cctv\_cover\_m, help\_m·help\_night\_m(밤에 열린 곳: 24시간 경찰·길가 비상벨·편의점)·guard\_m·rest\_m·heat\_m·cold\_m(+1km 안 수), school\_zone\_m(어린이보호구역 대상 시설까지, 300m 안이면 보호구역 안), xing\_signal·nosignal·unknown, curb, footbridge\_ok, road\_class, sgg\_cd·dong\_cd, data\_gap | 만듦 (2026-11-02, 길 8,569개. 원일 님 확인 필요) |
| admin\_area | 행정구역 경계 (SGIS 2025년 2분기, 전국 시도 17·시군구 252, 광주 행정동 96). 데이터 부족 판정과 길·시설의 구·동 찾기에 씀 | level(sido·sigungu·dong), code(SGIS 코드), name, full\_name, base\_date, geom(다각형, 공간 인덱스) | 만듦 (2026-11-02) |
| region\_density | 시군구마다 시설 종류별 밀도와 데이터 부족 판정(전국 중간값의 10% 아래). edge\_stat.data\_gap을 만드는 근거 | sgg\_cd, kind(light·cctv·help·guard·rest), n, area\_km2, density, ratio, gap | 만듦 (2026-11-02, 전국 252곳) |
| poi | 지도 표시·경로 응답용 정제 시설(광주 경계 안). 시설 표 13개를 한 모양으로 모으고, 같은 종류 10m 안 중복을 합침(CCTV는 전국·광주 파일). 서버는 지도 아이콘과 경로 주변 시설을 이 표에서 꺼냄. 화장실·쉼터는 운영 기간(season\_from·season\_to)과 여는 시각(open\_from·open\_to·open\_days)을 오늘·지금과 비교해 '이용 가능 / 닫힘 / 운영 기간 아님 / 확인 안 됨'을 붙임. 무더위쉼터·한파쉼터는 계절이 달라 따로 두고, 운영 기간은 원본에 없어 정부 대책 기간(폭염 5.20\~9.30, 한파 11.15\~3.15)을 넣음. 화장실·건물 안 비상벨은 indoor=1로 표시(지도에는 모두 보임). CCTV는 교통·쓰레기 단속, 시설·재난 관리용을 빼고 넣음. 쉼터는 계절과 상관없이 모두 지도에 보이고, 길찾기 결과의 화장실·쉼터 상태는 poi\_status 뷰의 now\_status와 그 근거 now\_note로 꺼냄(DB 시각 기준). 쉼터 중 행정복지센터·은행·도서관·복지관·보건소·지하철역은 쉼터 기간이 아니어도 일반 업무 시간(place\_type·biz\_from·biz\_to·biz\_days, 예: 은행 평일 09\~16시, 지하철역 매일 05:30\~24시)이면 '이용 가능'으로 보고, 경로당은 회원 위주라 기간 밖이면 '운영 기간 아님'. 업무 시간은 기관마다 다를 수 있는 일반값. 같은 방식으로 경찰(지구대·파출소·경찰서 24시간, 치안센터 평일 09\~18시), 비상벨(길가 24시간, 화장실 안은 50m 안 화장실 개방 시간), 아동안전지킴이집(편의점·문구점·약국·마트 등 가게 종류별 일반 영업시간, 시간 밖은 닫혔을 수 있어 확인 안 됨), 무더위쉼터(기간 안 평일 09\~18시, 원본의 야간·주말 개방 표시 반영)에도 상태를 붙임. 평일 규칙은 공휴일(holiday 표, 2026\~2027 대체공휴일 포함)을 주말처럼 봄. 원본을 다시 넣으면 scripts/build\_poi.py로 다시 만듦 | kind(light·cctv·police·bell·store·guard\_house·shelter·toilet·footbridge·crosswalk·school\_zone), sub, name, address, lon, lat, geom, cnt(합친 수·카메라 대수), wheelchair, open\_hours·open\_from·open\_to·open\_days, season\_from·season\_to, indoor, phone, modes(common·child·senior·wheelchair), detail(앱에 바로 띄울 문장), sgg\_cd·dong\_cd, sources | 만듦 (2026-11-02, 97,233곳: 등 85,754·CCTV 4,514·횡단보도 631(OSM 348 포함)·계단 27·쉼터 2,936·화장실 1,205 등) |
| edge\_poi | 길마다 50m 안 시설(poi, 등 제외)과 길 위 위치. 서버는 경로의 길 id로 이 표를 읽어 시설 목록을 만들고, 길 순서와 pos로 "80m 앞 편의점", "쉬는 곳이 300m마다"를 계산. poi를 다시 만들면 scripts/build\_edge\_poi.py도 다시 돌림 | edge\_id, poi\_id, kind, dist\_m(길에서 거리), pos(from\_node 0 → to\_node 1) | 만듦 (2026-11-02, 문화전당 2km 길 8,569개, 21,034줄) |
| cctv 목적 분류 (칸 추가) | 설치 목적 원문 11가지를 8개로 묶음. 목적이 빈 광주 파일은 10m 안 행안부 파일 카메라의 목적을 가져옴(6,614대 중 252대). 나머지는 '모름'이고, 나중에 다른 자료나 현장 확인으로 채울 때는 purpose\_from='직접 입력'으로 두면 다시 돌려도 덮어쓰지 않음. 길 계산·데이터 부족 판정·poi는 교통·쓰레기·시설·재난을 빼고 셈(2026-11-02 결정, 목적 모름은 셈) | purpose\_group(방범·어린이·다목적·교통·쓰레기·시설·재난·기타·모름), purpose\_from(원본 / 10m 안 행안부 파일 / 직접 입력) | 만듦 (2026-11-02, scripts/classify\_cctv.py. 방범 242,002·다목적 49,135·어린이 30,297·교통 22,558·모름 6,362 등) |
| facility\_unlocated | 위치를 믿을 수 없어 시설 표에서 뺀 줄을 지우지 않고 모아 둔 표. 좌표가 한 점에 몰렸거나(지자체가 기관 좌표·기본값을 넣음) 좌표가 없었는데 카카오 주소·장소 검색으로도 못 찾았거나 번지 있는 주소가 없는 줄. 원본 줄 전체를 attrs에 둬서, 나중에 좌표를 찾으면 원래 표로 되돌릴 수 있음. scripts/fix\_cctv\_coords.py·fix\_pile\_coords.py·fill\_missing\_coords.py가 지우는 대신 여기로 옮김 | tbl(원래 표), source, source\_id, kind, name, address, lon·lat(원본 좌표), reason(뺀 이유), tried(찾아본 검색어), attrs, moved\_at | 만듦 (2026-11-02, 6,057줄: 보안등 4,128·CCTV 1,870·횡단보도 39·비상벨 17·지킴이집 3) |
| holiday | 공휴일(관공서 공휴일 기준, 대체공휴일 포함). poi\_status가 평일 규칙을 판단할 때 주말처럼 봄. 직접 넣은 값이라 임시공휴일이 생기면 한 줄 추가 | day, name | 만듦 (2026-11-02, 2026\~2027년 40일) |
| subway\_station | 광주 1호선 20역과 역마다 엘리베이터 등 장애인 편의시설 9종의 수(광주교통공사 API, 5.1). 고장 수는 바뀌므로 하루 한 번 다시 받는 안 | station\_id, name, address, lon, lat, geom, elevator\_total·elevator\_available·elevator\_disabled\_gates (에스컬레이터·휠체어리프트·장애인화장실 등도 같은 꼴), fetched\_at | 만듦 (2026-11-03, 20역. CSV로 먼저 받아 두었다가 넣음, 결정 42. migration 019, scripts/load\_transit.py) |
| subway\_entrance | 역 출입구 좌표(OSM 92곳). 400m 안 가장 가까운 역에 붙임 | osm\_id, station\_id, exit\_no, wheelchair, lon, lat, geom | 만듦 (2026-11-03, 92곳) |
| subway\_timetable | 역·방향·요일별 열차 도착 시각(2026-11-01 기준). 실시간 도착 API가 없어 다음 열차를 이 표로 계산 | station\_code, day\_type(평일·토요일·휴일·명절), direction(1 평동행·2 소태·녹동행), end\_station, arrive\_time, base\_date | 만듦 (2026-11-03, 14,730줄. 원본에 한 칸 밀린 역 코드 238줄은 역 이름으로 고침) |
| subway\_transfer · subway\_exit\_bus · subway\_exit\_place | 역 환승(내리는 칸 → 출구 → 정류장까지 걸리는 시간, 2022)·출구별 연계 버스(2021)·출구 주변 장소(2021) | 역, 출구 번호 + (내리는 칸·정류장 번호·노선·환승 시간 / 노선·행선지 / 장소·거리·걸어서 걸리는 분) | 만듦 (2026-11-03, 6,001·283·524줄) |
| bus\_stop · bus\_line | 광주 버스 정류장 5,625곳·노선 127개(광주버스정보 API). 버스 도착 정보는 저장하지 않고 요청 때 부름(9장) | busstop\_id, name, ars\_id, lon, lat, geom / line\_id, line\_name, first\_run, last\_run, run\_interval | 만듦 (2026-11-03, 정류장 5,625곳·노선 127개. 좌표 없는 정류장 2곳은 뺌) |
| lowfloor\_bus | 차량마다 저상 여부(2025-12-31 기준 1,044대, 저상 513). 다음 버스가 저상인지는 도착 API의 LOW\_BUS로 알 수 있어, 이 표는 노선별 저상버스 비율 안내에 씀 | vehicle\_no, low\_floor, company, route\_name, base\_date | 만듦 (2026-11-03, 1,044대) |
| accident\_zone (추가안) | 사고다발 영역 한 곳이 한 줄 (FR-44, 보류) | kind(노인·어린이), year, 사고 건수, geom(다각형, 공간 인덱스). 길마다 영역 안인지는 edge\_stat에 미리 계산 | 추가안 |
| user\_place (추후) | 사용자가 제보한 장소 (FR-45 2단계). 1단계는 휴대폰에만 저장하므로 표가 필요 없음 | kind, name, lon, lat, geom, 제보 수, 확인 여부, 신고 수 | 추후 (추가안) |

좌표는 WGS84 경도·위도를 `POINT(경도, 위도)`로 넣고(SRID 0) 거리는 `ST_Distance_Sphere`로 잰다. MySQL의 SRID 4326은 위도·경도 순서라 실수하기 쉽기 때문이다.

B파트가 `edge_penalty`에 칸 추가(지킴이집·횡단·쉼 곳·보도턱·육교)와 감점 원값 저장을 제안했다(2026-10-30). 도로망 표를 만들기 전에 정할 것은 6.5 "A파트와 맞출 것"(C1\~C6)에 있다.

도로망은 시연 지역(국립아시아문화전당 반경 2km)부터 `road_node`(6,431)·`road_edge`(8,569)에 넣었다(2026-11-02, OSM). `road_edge`에는 위 표의 칸에 더해 OSM 길 번호(`way_id`)·이름·인도/횡단 구분(`footway`)·다리·바닥 재질을 두고, `road_node`에는 횡단보도 신호(`crossing`)·보도턱(`kerb`) 표시를 둔다. 행정구역 경계는 `admin_area`(SGIS 2025년 2분기, 전국 시도 17·시군구 252, 광주 행정동 96)에 있고, 시군구별 시설 밀도와 데이터 부족 판정은 region\_density에 있다(6.5).

### 8.3 앱↔서버 API (1차)

1차에서 앱이 부르는 서버 주소는 3개다. SOS·경보음·보호자 문자는 휴대폰 안에서 끝나므로 서버 API가 없다. 걷는 동안의 위치도 서버에 보내지 않는다(NFR-07). 모든 주소는 `/api`로 시작하고 JSON(UTF-8)으로 주고받는다. 좌표는 `lat`(위도)·`lon`(경도)로 이름을 붙여 쓰고, 경로 선(`geometry`)만 GeoJSON 순서인 `[경도, 위도]` 배열이다.

| 주소 | 하는 일 | 관련 FR | 만드는 파트 |
| --- | --- | --- | --- |
| `GET /api/places` | 장소 이름·주소 검색. 서버가 카카오 키워드 검색을 대신 부름 | FR-01 | C |
| `GET /api/places/nearby` | 현재 위치에서 지금 이용 가능한 가장 가까운 화장실·경찰서·지킴이집·쉼터. DB의 poi\_status 뷰에서 꺼냄(카카오 안 부름) | FR-50 | C |
| `POST /api/route` | 출발·도착 → 최단·안전 경로, 차이 설명, 경로 주변 시설 | FR-02\~07 | B(계산), C(주소) |
| `GET /api/health` | 서버·DB 상태와 데이터 기준일. 시연 전 점검용 | — | C |
| POST /api/explain (3차) | 기본 설명 문장을 LLM으로 다듬어 돌려줌. 실패하면 기본 문장 | FR-43 | C |

**장소 검색** `GET /api/places?query=국립아시아문화전당&lat=35.1473&lon=126.9214`

- `query`는 필수(2자 이상). `lat`·`lon`은 선택이고, 주면 가까운 순으로 준다.
- 결과는 최대 10개다.

```json
{ "places": [
  { "name": "국립아시아문화전당", "address": "전남광주통합특별시 동구 문화전당로 38", "lat": 35.1473, "lon": 126.9214, "distanceM": 0 }
] }
```

- 카카오가 실패하면 `503`과 `SEARCH_UNAVAILABLE`을 준다. 앱은 "검색이 잠시 안 됩니다. 지도를 길게 눌러 고를 수 있어요"를 띄운다(4.2).

**가까운 곳** `GET /api/places/nearby?lat=35.1473&lon=126.9214&kind=toilet&mode=senior` (FR-50, 김재휘 안)

- `kind`: `toilet`·`police`·`guard_house`·`school_zone`·`shelter`·`bell` 중 하나(앱이 설정에서 고른 종류마다 부름). `mode=wheelchair`이면 화장실은 장애인 화장실(`poi.wheelchair = 1`)만. `school_zone`·`bell`은 여는 시간이 없어 상태 없이 가까운 순서로 준다.
- 서버는 `poi_status`에서 1km 안, `now_status`가 '이용 가능'인 곳을 직선으로 가까운 5곳까지 돌려준다. 없으면 '확인 안 됨'인 곳을, 그마저 없으면 빈 목록과 가장 가까운 닫힌 곳(`closest_closed`)을 준다.
- 앱은 목록의 곳들로 `/api/route`를 불러 걷는 거리가 가장 짧은 곳을 ③에 보여 준다(처음에는 찫 번째 곳만 불러도 됨).

```json
{ "places": [
    { "id": 51234, "kind": "toilet", "name": "광주역 공중화장실", "lat": 35.1461, "lon": 126.9208, "distanceM": 140,
      "nowStatus": "이용 가능", "nowNote": "개방 시간 06:00~22:00", "wheelchair": true } ],
  "closestClosed": null }
```

**경로 계산** `POST /api/route`

요청:

```json
{
  "origin":      { "lat": 35.1473, "lon": 126.9214, "name": "국립아시아문화전당" },
  "destination": { "lat": 35.1393, "lon": 126.9230, "name": "남광주역" },
  "night": "auto"
}
```

- `night`: `auto`(기본, 일몰 계산), `on`, `off` 중 하나(FR-03). 낮에 시연할 때는 `on`으로 보낸다.
- `mode`: `child`(어린이·청소년)·`elder`(노인)·`wheelchair`(휠체어)·`basic`(기본, 그 밖의 보행자) 중 하나. 1차부터 보낸다. 없으면 `basic`.

  앱(safe-route-android PR #13)은 임시로 `child`·`senior`·`disabled`·`general`을, 경로 엔진(B파트 route.yml)은 adult·child·senior·wheelchair를 쓰고 있어 세 곳이 모두 다르다(6.5). 서버 API를 만들기 전에 B·C·D파트가 한쪽으로 맞춘다(2026-10-30 확인).

응답 (좌표와 시설 목록은 줄인 예):

```json
{
  "conditions": { "night": true, "nightSource": "auto", "sunset": "17:41" },
  "routes": [
    { "kind": "shortest", "distanceM": 1150, "durationMin": 15,
      "geometry": [[126.9214, 35.1473], [126.9217, 35.1462]],
      "counts": { "cctv": 8, "light": 41, "bell": 1, "help": 2 } },
    { "kind": "safe", "distanceM": 1330, "durationMin": 17,
      "geometry": [[126.9214, 35.1473], [126.9226, 35.1468]],
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
| `facilities[]` | 지도 아이콘용 시설(FR-07). `on`은 `safe`·`shortest`·`both`. 최대 300개. 서버는 경로의 길 id로 edge\_poi를 읽어 만든다. 화장실·쉼터·경찰·비상벨·지킴이집에는 nowStatus(이용 가능·닫힘·운영 기간 아님·확인 안 됨)와 nowNote(근거 문장)를 붙인다(poi\_status 뷰, 결정 40, 김재휘 안) |
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

**2차에 늘어나는 것.** 응답의 `facilities[]`마다 경로의 몇 m 지점·왼쪽/오른쪽(FR-30)이 붙는다. 다시 안내(FR-35)는 새 주소 없이 같은 `/api/route`를 현재 위치로 다시 부른다. 경로마다 `arriveAt`(도착 예상 시각, FR-01-2)은 1차부터 넣는다. 비·폭염 등 다른 상황의 근거값은 3차다. 앱은 모르는 필드를 무시하므로 필드가 늘어도 깨지지 않는다.

**3차: 설명 다듬기** `POST /api/explain` (FR-43, 3.5). 앱은 `/api/route`의 기본 문장을 먼저 보여 준 뒤 이 주소를 따로 부른다. 요청 본문은 `/api/route` 응답의 `conditions`·`explanation`·`dataQuality`를 그대로 보낸다(좌표와 장소 이름은 빼고). 서버는 상태를 남기지 않고, LLM을 3초까지만 기다린다.

```json
{ "text": "곧 해가 지니까, 120m만 더 걸으면 …", "source": "llm" }
```

`source`는 `llm`(검사를 통과한 다듬은 문장) 또는 `template`(LLM 실패·3초 초과·검사 탈락으로 기본 문장을 그대로 돌려줌)다. 앱은 `llm`일 때만 카드의 문장을 바꾼다. 이 주소가 멈춰도 경로 화면은 기본 문장으로 그대로 동작한다.

### 8.4 앱↔서버 연결에 필요한 것

앱은 서버 주소로 HTTP 요청을 보내고 JSON을 받는다(8.3). 중간에 따로 둘 서버(게이트웨이·메시지 큐 등)는 없다. 대신 아래 다섯 가지를 챙겨야 휴대폰에서 실제로 연결된다.

| 항목 | 할 일 | 파트 |
| --- | --- | --- |
| 휴대폰에서 서버까지 가는 길 | 학교 DB는 학교 네트워크 안에서만 접속되므로, 서버는 학교망에 연결된 노트북에서 돌린다. 개발: 에뮬레이터는 `http://10.0.2.2:8080`(노트북의 localhost), 실제 휴대폰은 노트북과 같은 와이파이에서 `http://노트북IP:8080`. 발표·학교 밖 시연: 노트북 서버를 터널(cloudflared 등)로 https 주소에 열거나 서버·DB를 클라우드로 옮김(0.5 "서버를 어디에") | C, D |
| http 허용 | 안드로이드는 기본으로 https만 허용한다. 앱은 network\_security\_config.xml에서 10.0.2.2·localhost·127.0.0.1만 http를 허용해 두었다. 같은 와이파이의 실제 휴대폰으로 시연하려면 노트북 IP를 여기에 추가한다. 터널(https)을 쓰면 필요 없음 | D |
| 통신 라이브러리 | 앱은 따로 라이브러리 없이 HttpURLConnection과 org.json으로 부른다(data/ServerApi.kt). 요청·응답 모양은 8.3을 따름. 서버 주소는 local.properties의 SERVER\_BASE\_URL에 넣고, 비워 두면 예시 데이터를 쓴다 | D |
| 키 나누기 | 앱에는 카카오맵 SDK용 네이티브 앱 키만 넣고, 카카오 개발자 콘솔에 앱의 키 해시를 등록. 카카오 REST·공공데이터·TMAP·LLM 키는 서버 `.env`에만 두고 앱은 서버를 거쳐 씀(NFR-05) | C, D |
| 실패 처리 | 앱 요청 타임아웃은 연결 5초, 응답 10초. 실패하면 "경로를 불러오지 못했어요"와 다시 시도 버튼(3.4). SOS는 서버 없이 동작 | D |

**먼저 할 일.** 서버에는 아직 앱이 부를 API가 없다(데이터 적재 코드만 있음, 2026-10-30). C파트가 `/api/health`와, 8.3 예시 응답을 그대로 돌려주는 가짜 `/api/places`·`/api/route`를 먼저 만들어 두면, D파트는 경로 엔진(B파트)을 기다리지 않고 화면과 통신을 붙일 수 있다. 진짜 계산이 준비되면 같은 주소의 내용만 바꾼다. 앱(PR #13)은 큰 칸 이름만 8.3을 따르고 안쪽 모양은 임시로 정해 두었으니, 가짜 API를 만들 때 8.3과 앱 중 한쪽으로 맞춘다.

**카카오맵 키 넣는 방법 (D파트).** 카카오맵 SDK 연결은 앱 저장소 safe-route-android에 이미 들어 있다(PR #9). 지도에는 서버의 `KAKAO_REST_KEY`가 아니라 같은 카카오 앱의 **네이티브 앱 키**를 쓴다. 서버 `.env`에는 넣지 않고 앱 프로젝트의 `local.properties`(git에서 빠짐)에 둔다. 키는 팀 채팅이 아니라 한 사람씩 따로 전달한다. 팀원이 처음 실행할 때 할 일:

1. 카카오 개발자 사이트 → 내 애플리케이션 → `KAKAO_REST_KEY`를 받은 앱 → 앱 키에서 **네이티브 앱 키**를 받는다. Admin 키는 앱에 넣지 않는다.
2. 같은 앱의 플랫폼 → Android에 패키지 이름 `com.deungbul.anshimgil`과 **자기 키 해시**를 등록한다. 키 해시는 컴퓨터마다 다르므로 팀원마다 한 번씩 추가한다(아래 명령으로 확인).
3. 제품 설정의 카카오맵 사용 설정이 꺼져 있으면 켠다.
4. 앱 프로젝트 맨 위 폴더의 `local.properties` 맨 아래에 아래 두 줄을 넣고 Sync한 뒤 실행한다.

```bash
keytool -exportcert -alias androiddebugkey -keystore ~/.android/debug.keystore -storepass android -keypass android | openssl sha1 -binary | openssl base64
```

```properties
# 카카오맵 SDK용 네이티브 앱 키 (git에 올리지 않음)
KAKAO_NATIVE_APP_KEY=여기에_네이티브_앱_키
# 서버 주소. 에뮬레이터에서 이 컴퓨터의 서버는 http://10.0.2.2:8080, 비워 두면 예시 데이터
SERVER_BASE_URL=
```

앱 저장소에서 관련 코드가 있는 곳:

```text
safe-route-android/
├─ settings.gradle.kts          ← 카카오맵 저장소 주소 (dependencyResolutionManagement 안)
├─ gradle/libs.versions.toml    ← SDK 버전 kakaoMap = "2.15.2"
├─ local.properties             ← 키·서버 주소 (각자 만듦, git에 없음)
└─ app/
   ├─ build.gradle.kts          ← local.properties를 읽어 BuildConfig로 넘김
   └─ src/main/
      ├─ AndroidManifest.xml    ← INTERNET 권한, android:name=".SeipjidoApp"
      ├─ res/xml/network_security_config.xml  ← http 허용 주소
      └─ java/com/deungbul/anshimgil/
         ├─ SeipjidoApp.kt      ← 앱이 켜질 때 SDK 초기화
         ├─ map/KakaoMapView.kt ← Compose에서 쓰는 지도 (조선대 중심, 줌 16)
         └─ data/ServerApi.kt   ← 서버 호출
```

`app/build.gradle.kts`에서 키와 서버 주소를 꺼내는 부분:

```kotlin
val localProperties = Properties().apply {
    val file = rootProject.file("local.properties")
    if (file.exists()) file.inputStream().use { load(it) }
}
val kakaoNativeAppKey: String = localProperties.getProperty("KAKAO_NATIVE_APP_KEY", "")
val serverBaseUrl: String = localProperties.getProperty("SERVER_BASE_URL", "")

android {
    defaultConfig {
        buildConfigField("String", "KAKAO_NATIVE_APP_KEY", "\"$kakaoNativeAppKey\"")
        buildConfigField("String", "SERVER_BASE_URL", "\"$serverBaseUrl\"")
    }
    buildFeatures {
        compose = true
        buildConfig = true
    }
}
```

`SeipjidoApp.kt` (키가 없으면 초기화를 건너뛰고, 지도 화면에 안내 문구가 나온다):

```kotlin
class SeipjidoApp : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.KAKAO_NATIVE_APP_KEY.isNotBlank()) {
            KakaoMapSdk.init(this, BuildConfig.KAKAO_NATIVE_APP_KEY)
        }
    }
}
```

지도 대신 "키와 키 해시 등록을 확인해 주세요"가 나오면 2번(키 해시)부터 확인한다. 에뮬레이터(API 37)에서 지도가 뜨는 것은 확인했다(PR #9, 2026-10-30). 지도 색까지 어둡게 바꿀 수 있는지(FR-41)는 아직 확인하지 못했다.

## 9. 외부 API 명세

아래 API는 모두 실제 호출까지 확인했다(TMAP은 2026-10-30, LLM은 아직 정할 것). 카카오 도보 길찾기는 제휴 파트너 전용이라 우리 키로는 403(permission denied)이 나와 TMAP으로 바꿨다. 기상청·에어코리아 한도는 개발계정 기준 대략값이다. 키 값은 `.env`에만 있고, 아래는 `.env`의 키 이름이다.

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
| 카카오 키워드·주소 검색 | `dapi.kakao.com/v2/local/search/keyword.json` (장소 검색), `.../address.json` (데이터 적재 때 주소 → 좌표: 공중화장실 약 5.4만 번, CCTV 좌표 오류 약 6천 번, 몰린 좌표 약 6천 번, 광주 보안등 약 9천 번, 2026-11-02). 주소로 못 찾으면 키워드 검색에 '시·군 + 이름'(화장실 이름, 비상벨 설치 장소)을 넣고, 결과 주소에 그 시·군이 있을 때만 씀 | 요청 머리글 `Authorization: KakaoAK {키}`, `query` | 하루 10만 번 무료, 넘으면 429 | `KAKAO_REST_KEY` |
| TMAP 보행자 경로 (비교용) | TMAP routes/pedestrian (SK open API) | 출발·도착 좌표 | 하루 1,000건 (무료 상품) | `TMAP_APP_KEY` |
| ODsay LAB 대중교통 길찾기 (FR-37) | `api.odsay.com/v1/api/searchPubTransPathT` (노선 선 좌표는 `loadLane`) | `apiKey`, `SX`·`SY`(출발 경도·위도), `EX`·`EY`(도착 경도·위도) | 한도는 확인 필요. 호출은 확인함(조선대→광주송정역 경로 16개). 서버용 키는 호출하는 서버 IP 등록이 필요할 수 있음 | `ODSAY_API_KEY` |
| 광주 버스 도착 (FR-37) | apis.data.go.kr/6290000/gj\_bis/arriveInfo (정류장 목록은 stationInfo, 노선은 lineInfo) | serviceKey, BUSSTOP\_ID, resultType=json. 응답: LINE\_NAME(노선), REMAIN\_MIN(남은 분), REMAIN\_STOP(남은 정류장), LOW\_BUS(1이면 저상), DIR\_END(방면), ARRIVE\_FLAG | 개발계정 한도라 서버가 정류장마다 잠깐(예: 30초) 저장해 두고 다시 씀(안). 2026-11-03 활용 신청 뒤 호출 확인 | DATA\_GO\_KR\_KEY |
| 광주교통공사 지하철 편의시설 | www.grtc.co.kr/subway/openapi/json/disabledFacilityInformation (역 주소는 stationInformation) | station\_id (1 평동 \~ 20 녹동). 응답: 편의시설 종류(DT\_01 엘리베이터 \~ DT\_09)마다 전체·이용 가능 수, 고장 난 출입구 | 키 없음. http로 부르면 https로 넘어가므로 https로 부름 | — |
| Overpass API (OSM 도로망) | overpass-api.de/api/interpreter (바쁘면 overpass.private.coffee 등 미러) | data = Overpass QL (걸을 수 있는 highway 길, 남·서·북·동 범위) | 키 없음. 공용 서버라 바쁘면 거절됨. 받은 응답은 data/raw/osm/에 남김 | — |
| 공공데이터포털 표준데이터 (보안등·육교·횡단보도) | api.data.go.kr/openapi/tn\_pubr\_public\_scrty\_lmp\_api, …\_overpass\_api, …\_crosswalk\_api | serviceKey, pageNo, numOfRows(1,000), type=json | 개발계정 한도. 어린이보호구역(…\_child\_prtc\_zn\_api)은 아직 오류 30(키 미등록)이라 CSV로 받음 | DATA\_GO\_KR\_KEY (데이터별 키가 있으면 그것을 먼저) |
| LLM 설명 다듬기 (FR-43, 3차) | 정할 것 | 사실 목록(숫자·상황·모드·부족한 데이터 종류)과 기본 문장. 좌표·장소 이름은 보내지 않음 | 정할 것. 3초 안에 답이 없으면 기본 문장. 같은 사실은 저장해 둔 문장을 다시 씀 | 정할 것 (서버 .env에만) |

**TMAP 구간 종류:** 보행자 경로 응답의 구간마다 `facilityType`이 온다(12 육교, 14 지하보도, 15 횡단보도, 17 계단). `searchOption=30`이면 계단을 뺀 경로를 준다. 비교 평가와 장애인 모드 확인에 쓴다.

광주 지하철: 실시간 도착 API는 공개되지 않았다(광주교통공사 stationTimeInfomation은 역 사이 걸리는 시간뿐, 2026-11-03 확인). 다음 열차는 열차 시간표(5.1)로 계산한다. 엘리베이터가 몇 번 출구에 있는지도 공개 자료가 없다.

**기상청 격자:** 위도·경도를 람베르트 원추 투영 공식으로 nx·ny에 바꾼다. 아시아문화전당은 nx=59, ny=74다(조선대는 60, 74).

**실패 처리:** 외부 API는 3초 안에 답이 없으면 포기하고 마지막으로 저장해 둔 값을 쓴다. 저장한 값도 없으면 그 상황은 "정보 없음"으로 둔다(NFR-03).
