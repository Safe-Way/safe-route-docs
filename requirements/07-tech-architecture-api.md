<!-- 원본: https://claude.ai/artifact/FQvgiTtePsoNKVNBhhHdTy 의 "7~9. 기술·구조·API" 탭을 내보낸 것. 이 파일을 직접 고치지 말고 원본을 고친 뒤 다시 내보낸다. -->

## 7. 기술 스택 결정

서버는 **Java 21 + Spring Boot + MySQL**, 길찾기는 **JGraphT**, 화면은 **Leaflet 웹 + PWA**로 한다. 학교 서버가 MySQL이고 언어는 Java로 정했기 때문에, 그 안에서 직접 고칠 수 있는 것을 골랐다.

| 영역 | 결정 | 이유 | 검토한 대안 |
| --- | --- | --- | --- |
| 언어 | Java 21 (Temurin) | 사용자 결정 | Python |
| 프레임워크 | Spring Boot 4.1.2 (Web MVC) | REST API와 화면 파일을 한 서버에서 제공 | — |
| 빌드·도구 | Maven(래퍼 포함), IntelliJ | 사용자 결정 | Gradle |
| DB | MySQL 8.4 (지금 로컬, 나중에 학교 서버) | 학교 서버가 MySQL. 공간 인덱스와 `ST_Distance_Sphere` 동작 확인 | PostgreSQL + PostGIS (기능은 더 많지만 서버가 없음) |
| DB 접근 | JdbcTemplate | 수백만 건 적재와 공간 SQL을 직접 쓰기 쉬움 | JPA |
| 길찾기 | JGraphT (다익스트라·A\*) | 비용 함수를 마음대로 바꾸고, 경로 위 시설을 세기 쉬움 | GraphHopper (빠르지만 상황별 비용·설명을 붙이기 어려움), pgRouting (PostgreSQL 필요) |
| 전국 길찾기 구조 | 길·감점은 DB에, 요청 때 출발·도착 주변 범위만 읽어 그래프 생성 | 전국 그래프를 메모리에 두지 않아도 됨. 도보는 거리가 짧아 범위가 작음 | 전국 그래프 상주 |
| 도로망 | OpenStreetMap 대한민국 PBF, osm4j로 읽음 | 보행길·계단·덮개 정보가 있고 무료 | 국가교통DB 보행네트워크 (지역이 제한적) |
| 공간 계산 | JTS (버퍼, STRtree), 좌표 변환은 공식 직접 구현 | 시설→길 결합을 메모리에서 빠르게 | Proj4J (EPSG:3857·4326만 쓰므로 불필요) |
| 일몰 계산 | commons-suncalc | 위도·경도·날짜만으로 계산, API 호출 없음 | 한국천문연구원 API |
| 화면 | Leaflet 1.9 웹 + PWA | 휴대폰에서 앱처럼 설치. 앱을 따로 만들지 않음 | Android 앱, Capacitor |
| 배경 지도 | OpenStreetMap 타일 (시연 규모) | 무료, Leaflet 기본 | VWorld, 카카오맵 |
| 장소 검색 | 카카오 로컬 API (서버가 대신 호출) | 한국 장소 검색 품질, 하루 10만 번 무료 | 네이버 검색 API |
| 데이터 내려받기 | Python 3 표준 라이브러리 스크립트 | 가끔 돌리는 도구라 설치 없이 실행. 적재는 Java | Java로 통일 |
| 테스트 | JUnit 5 + Spring Boot Test | 기본 포함 | — |
| 버전 관리 | git + GitHub 비공개 저장소 | `.env`·원본 데이터는 제외 | — |

## 8. 시스템 구조

무거운 일(내려받기, 적재, 길마다 감점 계산)은 **데이터를 갱신할 때 한 번**만 하고, 경로 요청 때는 출발·도착 주변의 길과 감점만 읽어 가중치를 곱해 계산한다.

> 그림: 시스템 구성 · 준비 단계와 요청 단계 ([원본 문서](https://claude.ai/artifact/FQvgiTtePsoNKVNBhhHdTy)에서 볼 수 있음)

브라우저는 서버하고만 통신하고, 인증키가 필요한 외부 API는 서버가 대신 부른다.

### 8.1 경로 요청 처리 순서

1. 출발·도착 좌표를 받는다 (장소 이름이면 카카오로 좌표를 찾는다).
2. 일몰 계산·기상청·에어코리아로 상황을 판단하고 가중치를 정한다.
3. 두 점을 감싼 사각 범위를 사방 500m 넓혀 그 안의 길·감점을 DB에서 읽는다.
4. JGraphT로 최단 경로와 상황 경로를 계산한다 (최대 우회 1.3배 규칙 적용).
5. 두 경로가 지나는 시설을 세어 차이 설명을 만든다.
6. 경로 선, 시설 위치, 설명문을 JSON으로 돌려준다.

### 8.2 DB 표 설계

| 표 | 내용 | 주요 칸 | 만든 때 |
| --- | --- | --- | --- |
| `facility` | 시설 한 개가 한 줄 (10종류) | type, subtype, source, source\_id, name, address, lon, lat, unit\_count, geom(공간 인덱스), attrs(JSON) | 만듦 |
| `data_source` | 출처별 적재 현황 | type, source, total\_count, loaded\_count, skipped\_count, complete, loaded\_at | 만듦 |
| `road_node` | 길이 만나는 점 | id(OSM 번호), lon, lat, geom | 2주차 |
| `road_edge` | 두 점 사이의 길 | id, from\_node, to\_node, length\_m, highway, covered, steps, geom(선) | 2주차 |
| `edge_penalty` | 길마다 미리 계산한 감점 | edge\_id, light, cctv, help, cover, stairs, heat, cold, traffic, data\_gap | 2주차 |

좌표는 WGS84 경도·위도를 `POINT(경도, 위도)`로 넣고(SRID 0) 거리는 `ST_Distance_Sphere`로 잰다. MySQL의 SRID 4326은 위도·경도 순서라 실수하기 쉽기 때문이다.

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

**기상청 격자:** 위도·경도를 람베르트 원추 투영 공식으로 nx·ny에 바꾼다. 조선대는 nx=60, ny=74다.

**실패 처리:** 외부 API는 3초 안에 답이 없으면 포기하고 마지막으로 저장해 둔 값을 쓴다. 저장한 값도 없으면 그 상황은 "정보 없음"으로 둔다(NFR-03).
