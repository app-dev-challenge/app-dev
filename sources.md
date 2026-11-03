# 경고등 · 차량정비 정보 출처 조사

> **조사일:** 2026-11-03
> **대상:** 차량 경고등(Telltale)의 의미 / 긴급도 / 조치 방법을 앱에 넣기 위한 1차 정보 출처
> **관련 문서:** [info.md](info.md) · [idea.md](idea.md) · [dev-environment.md](dev-environment.md) · [mvp-plan.md](mvp-plan.md)

---

## 0. 결론 요약

| 질문 | 결론 |
| --- | --- |
| 경고등 분류 체계의 기준은? | **Pleos Vehicle SDK의 `TelltaleType` enum 78종** (§2). 현대차그룹 플랫폼 자체의 공식 분류 체계이므로 심사에서 가장 강한 근거. |
| 국제 표준은? | **ISO 2575:2021 (+Amd 1:2024)** — 기호·색상 체계의 근거. 유료 문서 (§1.1). |
| 경고등 설명 본문은? | **자체 작성** + **현대 공식 취급설명서 딥링크**로 연결 (§6 채택안). |
| 리콜/무상수리 정보는? | **국토교통부 자동차 리콜정보 OpenAPI** (무료, JSON) (§3.1). |
| 정비소 위치는? | **공개 API 없음.** 공공데이터 정비업체 데이터 + NaviHelper 목적지 검색으로 우회 (§4). |
| OBD-II DTC 코드는? | **본 앱 범위 밖.** SDK가 DTC를 노출하지 않음 (§5). |

---

## 1. 1차 출처 — 표준 및 제조사 공식 문서

### 1.1 ISO 2575 (경고등 기호 국제표준) — 체계의 근거

| 항목 | 내용 |
| --- | --- |
| 정식 명칭 | *Road vehicles — Symbols for controls, indicators and tell-tales* |
| 현행판 | **ISO 2575:2021** (9판, 2021-09-07 발행) — 2010년판 대체 |
| 개정 | **ISO 2575:2021/Amd 1:2024** 발행됨 (별도 구매), Amd 2 진행 중 |
| 관리 | ISO/TC 22/SC 39 (Ergonomics) |
| 범위 | 승용차·경/중형 상용차·버스의 조작장치·표시장치·텔테일 기호. **광학 텔테일의 색상(정상 동작 vs 장치 고장)까지 규정** |
| 접근성 | **유료.** 약 2,780 SEK (스웨덴 표준기관 기준), 기관·판형별 상이 |
| 링크 | https://www.iso.org/standard/68409.html |

**활용 방식:** 전문(全文) 구매 없이도 **"색상 = 긴급도"라는 ISO의 설계 원칙**을 앱의 3단계 긴급도 분류 근거로 인용할 수 있습니다. 통상 체계는 다음과 같습니다.

| 색상 | ISO 상의 의미 | 앱 긴급도 매핑 |
| --- | --- | --- |
| 🔴 적색 | 즉각적 위험 / 장치 고장 | **Level 1 — 즉시 정차** |
| 🟡 황색(앰버) | 주의 / 점검 필요 | **Level 2 — 오늘 중 점검** |
| 🟢 녹색 · 🔵 청색 · ⚪ 백색 | 정상 동작 / 작동 중 표시 | **Level 3 — 참고** |

> ⚠️ **확인 필요:** 위 색상-의미 대응은 업계 통용 체계이며, ISO 2575 원문의 정확한 조문 번호는 유료 문서 확인이 필요합니다. 기획서에는 "ISO 2575의 색상 체계를 참조"로 기술하고, 조문 직접 인용은 피하는 것을 권장합니다.
> ⚠️ **미검증:** KS(한국산업표준) 부합화 여부(KS R ISO 2575 등), UN ECE R121(조작장치 식별 규정)과의 관계는 확인하지 못했습니다.

### 1.2 현대자동차 공식 취급설명서 — 설명 본문의 근거

| 항목 | 내용 |
| --- | --- |
| 사이트 | **ownersmanual.hyundai.com** |
| URL 패턴 | `https://ownersmanual.hyundai.com/full_pdf/{차종코드}/{연식}/ko_KR` |
| 해당 장 | **"경고등 및 표시등 확인하기"** (계기판 장 내부) |
| 특징 | 차종·연식별로 문서가 분리되어 있고, 같은 장의 페이지 번호가 모델마다 다름 |

**확인된 차종코드 · 해당 장 페이지 번호**

| 차종코드 | 연식 | "경고등 및 표시등" 페이지 | URL |
| --- | --- | --- | --- |
| LX3HEV (팰리세이드 HEV) | 2026 | p.138 | https://ownersmanual.hyundai.com/full_pdf/LX3HEV/2026/ko_KR |
| CE1 (아이오닉6) | 2025 | p.134 | https://ownersmanual.hyundai.com/full_pdf/CE1/2025/ko_KR |
| NE1 (아이오닉5) | 2026 | p.132 | https://ownersmanual.hyundai.com/full_pdf/NE1/2026/ko_KR |
| ME | 2027 | p.161 | https://ownersmanual.hyundai.com/full_pdf/ME/2027/ko_KR |
| CN7 / CN7N (아반떼) | 2021 / 2022 | 계기판 장 내 | https://ownersmanual.hyundai.com/full_pdf/CN7N/2022/ko_KR |
| DN8 / DN8HEV (쏘나타) | 2021 | 계기판 장 내 | https://ownersmanual.hyundai.com/full_pdf/DN8/2021/ko_KR |
| NX4 (투싼) | 2021 | 계기판 장 내 | https://ownersmanual.hyundai.com/full_pdf/NX4/2021/ko_KR |

**매뉴얼의 자체 표기 규약** (DB 구축 시 그대로 승계 권장)

| 표기 | 의미 |
| --- | --- |
| **경고** | 사람이 다치거나 사망의 우려가 있는 경우 |
| **주의** | 차량이 고장나거나 손상될 우려가 있는 경우 |

> 📌 매뉴얼은 모든 트림·선택사양을 포함해 설명하므로, 실제 차량에 없는 사양도 기술되어 있습니다. → 앱에서는 **`checkXXXCapabilities`로 실제 지원 여부를 확인한 항목만 노출**해야 합니다 ([dev-environment.md](dev-environment.md) §4 참조).

> ⚠️ **미검증:** 기아(kia.com) · 제네시스의 동등한 온라인 매뉴얼 사이트는 이번 조사에서 확인하지 못했습니다. 현대 기준으로 먼저 구축하고 이후 확장을 권장합니다.

---

## 2. ⭐ 플랫폼 내부 분류 체계 — `TelltaleType` enum 78종

**가장 가치 있는 출처입니다.** Pleos Vehicle SDK 소스에 **현대차그룹이 실제로 정의한 경고등 타입 전체 목록**이 ID와 함께 들어 있습니다. 공모전 심사에서 "현대차 플랫폼 체계에 정확히 맞춘 설계"의 근거가 됩니다.

| 항목 | 내용 |
| --- | --- |
| 출처 | `Vehicle-2.3.2-sources.jar` → `ai/pleos/playground/vehicle/constant/telltale/TelltaleType.kt` |
| 취득 경로 | `https://nexus-playground.pleos.ai/repository/maven-releases/ai/pleos/playground/Vehicle/2.3.2/Vehicle-2.3.2-sources.jar` (인증 없이 접근 가능) |
| 데이터 모델 | `TelltaleData(type: TelltaleType, state: TelltaleState, color: Int, priority: Int, warningCount: Int)` → `Telltales(telltales: List<TelltaleData>)` |

> 🔑 **`color`와 `priority` 필드가 플랫폼 차원에서 제공됩니다.** 즉 긴급도 분류를 자체 추정할 필요 없이 플랫폼 값을 그대로 쓸 수 있는 구조입니다. (단, 해당 API의 권한 제약은 [dev-environment.md](dev-environment.md) §4 참조)

### 2.1 전체 78종 (enum 이름 = ID)

**기본 안전 / 제동 (0–13)**

| ID | enum | 한국어 명칭(제안) | 긴급도(제안) |
| --- | --- | --- | --- |
| 0 | `PARKING_BRAKE` | 주차 브레이크 | 3 참고 |
| 1 | `ELECTRONIC_PARKING_BRAKE` | 전자식 주차 브레이크(EPB) | 2 점검 |
| 2 | `SEAT_BELT` | 안전벨트 미착용 | 1 즉시 |
| 3 | `DOOR_OPEN` | 도어 열림 | 1 즉시 |
| 4 | `ABS` | ABS 이상 | 1 즉시 |
| 5 | `MDPS` | 전동식 파워스티어링 이상 | 1 즉시 |
| 6 | `TPMS` | 타이어 공기압 경고 | 2 점검 |
| 7 | `LOW_VOLT_BATTERY` | 12V 배터리 충전 경고 | 2 점검 |
| 8 | `ESC` | 차체자세제어 작동 | 3 참고 |
| 9 | `ESC_OFF` | 차체자세제어 해제 | 3 참고 |
| 10 | `AIRBAG` | 에어백 이상 | 1 즉시 |
| 11 | `AFS` | 어댑티브 헤드램프 이상 | 2 점검 |
| 12 | `MASTER_WARNING` | 통합 경고 | 2 점검 |
| 13 | `AUTO_HOLD` | 오토홀드 | 3 참고 |

**램프 (14–19, 28)**

| ID | enum | 한국어 명칭(제안) | 긴급도(제안) |
| --- | --- | --- | --- |
| 14 | `AUTO_HIGH_BEAM` | 하이빔 어시스트 | 3 참고 |
| 15 | `HEADLAMP_LOW` | 전조등 하향 | 3 참고 |
| 16 | `HEADLAMP_HIGH` | 전조등 상향 | 3 참고 |
| 17 | `POSITION_LAMP` | 미등 | 3 참고 |
| 18 | `FRONT_FOG_LAMP` | 앞 안개등 | 3 참고 |
| 19 | `REAR_FOG_LAMP` | 뒤 안개등 | 3 참고 |
| 28 | `LED_HEADLAMP` | LED 헤드램프 이상 | 2 점검 |

**엔진 / 파워트레인 (20–27, 29–30, 102–103, 108)**

| ID | enum | 한국어 명칭(제안) | 긴급도(제안) |
| --- | --- | --- | --- |
| 20 | `ENGINE_OIL_PRESSURE` | 엔진오일 압력 경고 | **1 즉시** |
| 21 | `ENGINE_TEMPERATURE` | 냉각수 과열 | **1 즉시** |
| 24 | `POWER_DOWN` | 출력 제한 | **1 즉시** |
| 25 | `RBS` | 회생제동 시스템 | 2 점검 |
| 26 | `IMMOBILIZER` | 이모빌라이저 | 2 점검 |
| 27 | `ENGINE` | 엔진 체크 | 2 점검 |
| 29 | `PREHEAT` | 디젤 예열 플러그 | 3 참고 |
| 30 | `PARTICLE_FILTER` | DPF(매연포집필터) | 2 점검 |
| 102 | `WATER_SEPARATOR` | 연료 수분 분리기 | 2 점검 |
| 103 | `SCR` | 요소수(SCR) | 2 점검 |
| 108 | `LPI` | LPI 시스템 | 2 점검 |

**전동화 (23, 104–107, 109, 115–119)**

| ID | enum | 한국어 명칭(제안) | 긴급도(제안) |
| --- | --- | --- | --- |
| 23 | `EV_SYSTEM` | EV 시스템 이상 | **1 즉시** |
| 104 | `LOW_EV_BATTERY` | 구동 배터리 부족 | 2 점검 |
| 105 | `HEV_SYSTEM` | 하이브리드 시스템 이상 | **1 즉시** |
| 106 | `FCEV_SYSTEM` | 수소연료전지 시스템 이상 | **1 즉시** |
| 107 | `EREV_SYSTEM` | EREV 시스템 이상 | **1 즉시** |
| 109 | `EV_CHARGING_CONNECTOR` | 충전 커넥터 연결 | 3 참고 |
| 115 | `XEV_READY` | 주행 준비(READY) | 3 참고 |
| 116 | `I_PEDAL` | i-Pedal | 3 참고 |
| 117 | `LOW_FUEL` | 연료 부족 | 2 점검 |
| 118 | `LOW_FUEL_FCEV` | 수소 부족 | 2 점검 |
| 119 | `EV_BATTERY_CONDITIONING` | 배터리 컨디셔닝 | 3 참고 |

**승객 / 차체 (33–34, 120–121)**

| ID | enum | 한국어 명칭(제안) | 긴급도(제안) |
| --- | --- | --- | --- |
| 33 | `PASSENGER_AIRBAG_OFF` | 동승석 에어백 OFF | 2 점검 |
| 34 | `PASSENGER_AIRBAG_ON` | 동승석 에어백 ON | 3 참고 |
| 120 | `TRUNK_OPEN` | 트렁크 열림 | 1 즉시 |
| 121 | `CHILD_PRESENCE_DETECTION_OFF` | 후석 승객 알림 OFF | 2 점검 |

**방향지시 / 주행환경 (35–36, 100–101)**

| ID | enum | 한국어 명칭(제안) | 긴급도(제안) |
| --- | --- | --- | --- |
| 35 | `TURN_SIGNAL_LEFT` | 좌측 방향지시 | 3 참고 |
| 36 | `TURN_SIGNAL_RIGHT` | 우측 방향지시 | 3 참고 |
| 100 | `ICY_ROAD` | 노면 결빙 주의 | 2 점검 |
| 101 | `OVER_SPEED` | 과속 | 2 점검 |

**긴급 (22)**

| ID | enum | 한국어 명칭(제안) | 긴급도(제안) |
| --- | --- | --- | --- |
| 22 | `EMERGENCY_CALL` | 긴급 통화(e-Call) | **1 즉시** |

**주행모드 (13, 110–114, 200–236)** — *경고가 아닌 상태 표시. 긴급도 3(참고) 고정*

`AUTO_HOLD`(13), `AWD_LOCK`(110), `HEV_MODE`(111), `ISG`(112), `DBC`(113), `REV`(114),
`DRIVE_MODE`(200), `DRIVE_MODE_SPORT`(201), `DRIVE_MODE_ECO`(202), `DRIVE_MODE_COMFORT`(203), `DRIVE_MODE_TERRAIN`(204),
`FORCED_ALL_WHEEL_DRIVE`(210), `TOW_MODE`(219), `ESC_SPORT`(220), `CREEP_OFF`(221), `DRIFT_MODE`(222), `TRACK_MODE`(223), `UTILITY_MODE`(224), `GREEN_ZONE`(225),
`PHEV_MODE`(226), `PHEV_MODE_HEV`(227), `PHEV_MODE_AUTO`(228), `PHEV_MODE_EV_PLUS`(229), `EREV_MODE`(230),
`LAUNCH_MODE`(231), `ACCEL_RESPONSE_DASH`(232), `ACCEL_RESPONSE_ACCEL`(233),
`N_BATTERY_DRAG`(234), `N_BATTERY_SPRINT`(235), `N_BATTERY_ENDURANCE`(236)

**ADAS (300–320)**

| ID | enum | 한국어 명칭(제안) | 긴급도(제안) |
| --- | --- | --- | --- |
| 300 | `FCA` | 전방충돌방지보조 | 2 점검 |
| 301 | `LKA` | 차선유지보조 | 3 참고 |
| 302 | `LFA` | 차로유지보조 | 3 참고 |
| 303 | `LCA_LEFT` | 후측방 경고(좌) | 2 점검 |
| 304 | `LCA_RIGHT` | 후측방 경고(우) | 2 점검 |
| 305 | `ELK` | 차로이탈방지보조 | 3 참고 |
| 306 | `ICC` | 지능형 크루즈 | 3 참고 |
| 320 | `DRIVER_ATTENTION` | 운전자 주의 경고 | 2 점검 |

> 📌 **긴급도(제안) 컬럼은 본 문서가 제안하는 초안입니다.** 실제 앱에서는 ① 플랫폼이 제공하는 `color`/`priority` 값을 1순위로, ② 아래 표를 폴백 규칙으로 사용합니다. 분류 확정 근거와 안전 문구는 §1.2 공식 매뉴얼로 교차검증이 필요합니다.

---

## 3. 공공 데이터 · 정부 출처

### 3.1 ⭐ 국토교통부 자동차 리콜정보 OpenAPI — 채택

| 항목 | 내용 |
| --- | --- |
| 링크 | https://www.data.go.kr/data/15089863/openapi.do |
| 제공 내용 | 국내 출시 승용차의 **리콜 정보 및 리콜에 따른 무상수리 여부** |
| 형식 / 비용 | JSON / **무료** |
| 등록·수정일 | 2021-09-24 / 2021-09-28 |
| 담당 | 국토교통부 자동차정책과 |

**⚠️ 이용 조건 (반드시 준수 — 기획서·앱에 반영 필요)**

1. **출처표시 의무:** "자동차리콜센터" 표기
2. **딥링크 의무:** 리콜 상세내용 확인 시 **리콜센터로 연결**되어야 함 (상세 URL은 API가 함께 제공)
3. **사용량 보고 의무:** API 호출 건수, 서비스 사용자 수 등을 **주기적으로 제공**

### 3.2 한국교통안전공단 공공 API

| 데이터 | 내용 | 링크 |
| --- | --- | --- |
| 자동차 종합정보 **정비이력정보** 서비스 | 등록연월·차종·부품구분코드로 정비 관련 **통계 건수** 조회. 개발계정 트래픽 10,000건 | https://data.go.kr/data/15057856/openapi.do |
| **관리사업자정보** 서비스 | 등록연월·사업유형코드(자동차정비업 등)로 기간별 등록 통계 조회. **매월 자료는 익월 2일부터** 조회 가능 | https://www.data.go.kr/data/15057939/openapi.do |

> ⚠️ 두 API 모두 **개별 차량 단위가 아닌 통계 데이터**입니다. "내 차의 정비이력"에는 쓸 수 없습니다. 앱에는 §3.3을 안내 링크로 쓰는 편이 적합합니다.

### 3.3 자동차민원 대국민포털 (ecar.go.kr) — 안내 링크용

| 항목 | 내용 |
| --- | --- |
| 사이트 | **ecar.go.kr** |
| 제공 | 차량별 **정비이력 조회** (침수·사고 이력 확인 가능) |
| 모바일 앱 | "자동차 토털이력정보 조회" |
| 운영 | 한국교통안전공단(TS) |
| 용도 | **API 아님.** 앱에서 "정비이력 확인" 안내 링크로 사용 |

### 3.4 정비업체 위치 데이터 (지자체 파일데이터)

| 항목 | 내용 |
| --- | --- |
| 형태 | 지자체별 자동차정비업체 파일데이터 (종합정비업/전문정비업 등으로 분류) |
| API 변환 | 공공데이터활용지원센터가 3단계 이상 오픈포맷 파일데이터를 **RESTful JSON/XML OpenAPI로 자동 변환** 제공 |
| 예시 | 경기도 자동차정비업체 현황 (LINK 방식 API, XML) |
| 참고 | https://www.data.go.kr/data/15028204/standard.do (공공데이터 제공 표준) |

> ⚠️ **전국 통합 데이터셋은 확인되지 않았습니다.** 지자체별로 흩어져 있어 MVP에서는 **시연 지역만 선별 적재**하는 것이 현실적입니다 ([mvp-plan.md](mvp-plan.md) F3 참조).

---

## 4. 제조사 정비 네트워크 (블루핸즈 / 오토큐)

| 항목 | 확인된 내용 |
| --- | --- |
| 현대 블루핸즈 — 네트워크 소개 | https://www.hyundai.com/kr/ko/customer-support/service-network/service-network-information/bluehands |
| 현대 블루핸즈 — 검색/예약 | https://www.hyundai.com/kr/ko/customer-service/service-network/service-reservation-search |
| 규모 | 약 **1,200개** 네트워크. 유형: 전문 / 종합 / 하이테크센터 |
| 예약 | 온라인 예약 지원. **예약 내역 조회는 로그인 필요** |
| **공개 API** | ❌ **없음** (조사에서 확인되지 않음) |
| 기아 오토큐 | 기아 공식 정비 네트워크. 앱/온라인/전화 예약. **공개 API 및 공식 지점검색 페이지 URL 미확인** |

**→ MVP 대응 방안** (상세: [mvp-plan.md](mvp-plan.md) F3)

1. **1순위:** `NaviHelper.requestRoute` / `addWaypoint`로 내비게이션에 "정비소" 목적지를 넘김 → 차량 내장 내비의 POI 검색 결과 활용 (제휴·크롤링 불필요)
2. **2순위:** §3.4 공공데이터 정비업체 좌표를 시연 지역만 앱에 번들
3. **금지:** 블루핸즈 웹사이트 크롤링 — 약관 위반 및 심사 리스크

---

## 5. OBD-II DTC 고장코드 — 범위 제외

| 항목 | 내용 |
| --- | --- |
| 표준 | **SAE J2012** *Diagnostic Trouble Code Definitions*. 최신판 **J2012_202509** (J2012_201303 폐지) |
| 접근성 | **유료.** 코드 목록 본문이 유료 문서에 포함 |
| 공개 전체 목록 | ❌ 신뢰할 수 있는 오픈소스 전체 DB 확인되지 않음 |
| 링크 | https://saemobilus.sae.org/content/J2012_201612 |

**결론: 본 앱에서 제외합니다.**

- Pleos Vehicle SDK는 **DTC를 노출하지 않습니다** (19개 도메인 중 진단코드 도메인 없음)
- 앱이 다루는 단위는 DTC보다 상위인 **경고등(Telltale) / 소모품 상태 레벨**
- 따라서 유료 표준 구매도, DTC DB 구축도 불필요

---

## 6. ⭐ 채택한 DB 구축 방식

> **결정:** ISO 2575 기호·색상 체계를 분류 골격으로 삼고, 설명 본문은 자체 작성하며, "자세히"는 현대 공식 취급설명서로 딥링크 연결합니다. (저작권 안전 + 출처 신뢰도 확보)

### 6.1 3계층 구조

```
[1] 분류 키        → TelltaleType enum 78종 (§2)          — 플랫폼 체계 그대로
[2] 설명·조치 본문 → 자체 작성 (ISO 2575 색상 체계 기반)     — 저작권 안전
[3] 상세 근거       → ownersmanual.hyundai.com 딥링크 (§1.2) — 출처 신뢰도
```

### 6.2 데이터 스키마 (초안)

```json
{
  "schemaVersion": "1.0",
  "sourceNote": "분류 체계: Pleos Vehicle SDK TelltaleType / 색상 체계: ISO 2575:2021 참조",
  "items": [
    {
      "type": "ENGINE_TEMPERATURE",
      "typeId": 21,
      "nameKo": "냉각수 과열 경고",
      "color": "RED",
      "severity": 1,
      "severityLabel": "즉시 정차",
      "meaning": "엔진 냉각수 온도가 정상 범위를 넘었습니다.",
      "whatHappens": "그대로 주행하면 엔진이 손상될 수 있습니다.",
      "actionNow": [
        "안전한 곳에 차를 세우고 엔진을 끕니다.",
        "보닛을 열기 전 엔진이 식을 때까지 기다립니다.",
        "냉각수가 새는 흔적이 있는지 바닥을 확인합니다."
      ],
      "doNot": ["엔진이 뜨거울 때 라디에이터 캡을 열지 마세요."],
      "ttsScript": "냉각수 온도가 너무 높습니다. 안전한 곳에 차를 세우고 엔진을 꺼 주세요.",
      "manualRef": {
        "publisher": "현대자동차 취급설명서",
        "chapter": "경고등 및 표시등 확인하기",
        "urlTemplate": "https://ownersmanual.hyundai.com/full_pdf/{modelCode}/{year}/ko_KR"
      },
      "signalSource": "POWER_DOWN_WARNING_INDICATOR"
    }
  ]
}
```

### 6.3 저작권 처리 원칙

| 원칙 | 내용 |
| --- | --- |
| ✅ 허용 | enum 이름·ID(기능적 식별자), 사실 정보(색상·의미 분류), **자체 문장으로 재작성한 설명** |
| ✅ 허용 | 공식 매뉴얼 **링크 연결** |
| ⚠️ 주의 | 매뉴얼의 "경고/주의" 표기 규약 **차용**은 가능하나, 안전 문구를 그대로 복제하지 않음 |
| ❌ 금지 | 매뉴얼 본문·아이콘 이미지 복제, 블루핸즈 등 웹사이트 크롤링 |
| 📌 필수 | 리콜 API 사용 시 **"자동차리콜센터" 출처표시 + 상세 딥링크** (§3.1) |

### 6.4 작업 분량 추정

| 작업 | 분량 | 비고 |
| --- | --- | --- |
| 전체 78종 작성 | 약 3~4일 | 1차 접수 전엔 과함 |
| **MVP 범위: 감지 가능한 13종만** | **약 0.5일** | [mvp-plan.md](mvp-plan.md) 신호 매핑표 기준 — **권장** |
| 주행모드 31종 | 생략 | 경고가 아닌 상태 표시 |

---

## 7. 출처 신뢰도 · 라이선스 요약

| 출처 | 신뢰도 | 접근성 | 라이선스/조건 | MVP 채택 |
| --- | --- | --- | --- | --- |
| Pleos SDK `TelltaleType` enum | ★★★ 1차 | 무료 (Maven) | SDK 이용약관 범위 | ✅ 분류 골격 |
| ISO 2575:2021 | ★★★ 1차 | **유료** | 구매 필요 | ✅ 체계 참조 (전문 미구매) |
| 현대 취급설명서 | ★★★ 1차 | 무료 (PDF) | 링크만 사용 | ✅ 딥링크 |
| 국토부 리콜정보 OpenAPI | ★★★ 1차 | 무료 (신청) | **출처표시+딥링크+사용량보고** | ✅ |
| TS 정비이력/관리사업자 API | ★★★ 1차 | 무료 (신청) | 공공누리 | ❌ 통계뿐 |
| ecar.go.kr | ★★★ 1차 | 무료 (웹) | API 아님 | ✅ 안내 링크 |
| 지자체 정비업체 데이터 | ★★☆ | 무료 | 공공누리 | ⚠️ 시연 지역만 |
| 블루핸즈/오토큐 웹 | ★★★ 1차 | 공개 API 없음 | 크롤링 금지 | ❌ |
| SAE J2012 | ★★★ 1차 | **유료** | 구매 필요 | ❌ 범위 밖 |

---

## 8. 확인 필요 (미검증 항목)

- [ ] ISO 2575:2021 원문의 색상-의미 조문 (§1.1) — 유료 구매 또는 "참조" 수준 인용으로 마무리
- [ ] KS 부합화 표준(KS R ISO 2575 등) 및 UN ECE R121과의 관계
- [ ] 기아 · 제네시스 온라인 취급설명서 사이트 URL 패턴
- [ ] `ownersmanual.hyundai.com` 의 **차종코드 전체 목록** (현재 7종만 확인)
- [ ] 리콜 OpenAPI 의 실제 요청 변수 / 응답 필드 / 엔드포인트 — **활용신청 후 명세 확인 필수** (문서 최종수정 2021년으로 오래됨)
- [ ] 리콜 OpenAPI 의 "사용량 주기적 제공" 의무의 구체적 절차·주기
- [ ] 전국 통합 자동차정비업체 데이터셋 존재 여부
- [ ] `TelltaleData.color` 값의 인코딩 형식 (ARGB int인지 코드값인지) 및 `priority` 값 범위 — **에뮬레이터 실측 필요**

---

## 참고 링크 전체

**표준**
- [ISO 2575:2021](https://www.iso.org/standard/68409.html)
- [SAE J2012](https://saemobilus.sae.org/content/J2012_201612)

**제조사**
- [현대 취급설명서 (예: 아이오닉5 2026)](https://ownersmanual.hyundai.com/full_pdf/NE1/2026/ko_KR)
- [현대 블루핸즈 네트워크 소개](https://www.hyundai.com/kr/ko/customer-support/service-network/service-network-information/bluehands)
- [현대 서비스 네트워크 검색/예약](https://www.hyundai.com/kr/ko/customer-service/service-network/service-reservation-search)

**공공데이터**
- [국토교통부 자동차 리콜정보 API](https://www.data.go.kr/data/15089863/openapi.do)
- [한국교통안전공단 정비이력정보](https://data.go.kr/data/15057856/openapi.do)
- [한국교통안전공단 관리사업자정보](https://www.data.go.kr/data/15057939/openapi.do)
- [공공데이터 제공 표준](https://www.data.go.kr/data/15028204/standard.do)
- 자동차민원 대국민포털: `ecar.go.kr`
- 자동차리콜센터: `car.go.kr`

**Pleos (SDK 소스 — §2 출처)**
- [Pleos Connect SDK API 레퍼런스](https://document.pleos.ai/en/api-reference/connect-sdk-pleos/)
- [Vehicle SDK intro](https://document.pleos.ai/en/api-reference/connect-sdk-pleos/Vehicle/intro)
- [Vehicle 권한 정책](https://document.pleos.ai/en/api-reference/connect-sdk-pleos/Vehicle/vehicle-permission)
- [공식 Vehicle 샘플 앱](https://github.com/Pleos-Playground/pleos-sample-vehicle-app)
