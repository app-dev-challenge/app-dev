# MVP 기획 — 차량 이상징후 음성 브리핑 앱

> **작성일:** 2026-11-03
> **관련 문서:** [info.md](info.md) (공모전 요강) · [idea.md](idea.md) (아이디어 후보) · [sources.md](sources.md) (정보 출처) · [dev-environment.md](dev-environment.md) (개발환경·제약)

---

## 0. 전제 (확정된 결정사항)

| 항목 | 결정 | 근거 |
| --- | --- | --- |
| 아이디어 | [idea.md](idea.md) **A. 경고등 통역사** 계열 | 공모전 페인포인트 "차량 이상 징후의 조기 인지" 정면 대응 |
| 신호 소스 | **공개 API(`CAR_SAFETY`, protectionLevel=normal)만으로 축소 설계** | [dev-environment.md](dev-environment.md) §4.1 — Telltale 리스너는 `signature\|privileged` |
| 정보 DB | **ISO 2575 체계 + 자체 작성 본문 + 공식 매뉴얼 딥링크** | [sources.md](sources.md) §6 |
| 기능 범위 | **F1~F4 전부** (음성 브리핑 / 대시보드 / 정비소 경유지 / 이력·리콜) | — |
| 일정 | 1차 접수 **2026-11-15** (D-12) · 2차 PT **2026-12-03~04** | [info.md](info.md) §2 |

---

## 1. 제품 정의

### 1.1 한 줄 정의

> **차에 경고가 켜지는 순간, 차가 직접 "무슨 뜻인지 · 지금 멈춰야 하는지 · 어디로 가면 되는지"를 음성으로 알려주는 앱.**

### 1.2 페인포인트

| # | 페인포인트 | 근거 |
| --- | --- | --- |
| P1 | 경고등 의미를 모른다 | 78종의 경고등이 존재하고([sources.md](sources.md) §2), 설명은 차종·연식별로 분리된 수백 쪽 PDF에만 있다 |
| P2 | 주행 중에는 찾아볼 수 없다 | 폰 검색은 불가/위험. 계기판은 **아이콘만** 보여주고 조치는 알려주지 않는다 |
| P3 | 판단을 못 한다 | "지금 세워야 하나, 집까지 가도 되나"를 모름 → 무시하거나 과잉 공포 |
| P4 | 조치로 이어지지 않는다 | 의미를 알아도 어디서 어떻게 고칠지 모름 |

### 1.3 타깃

| 구분 | 대상 | 왜 |
| --- | --- | --- |
| 1차 | **초보·장년 운전자** | 경고등 지식 부족이 가장 큼 |
| 2차 | **렌터카·카셰어링 이용자** | 내 차가 아니라 매뉴얼이 없고, 차종도 매번 다름 |
| 3차 | **차량 관리 책임자** (법인·가족 차량) | 이력 리포트 수요 |

### 1.4 모바일 대비 차량특화성 (심사 핵심 항목)

| 요소 | 폰 앱으로 가능? | 이유 |
| --- | --- | --- |
| 경고 발생 순간의 자동 감지 | ❌ | OBD 동글 없이는 신호를 받을 수 없음 |
| 소모품 상태 실시간 조회 | ❌ | 차량 내부 신호 |
| 발생 즉시 음성 개입 | ❌ | 폰은 "열어서 봐야" 함. 차량 앱은 **차가 먼저 말을 건다** |
| 내비 경로에 정비소 경유지 삽입 | ❌ | 차량 내장 내비 제어 필요 |
| 좌석별 에어백·안전벨트 상태 | ❌ | 좌석 센서 |

---

## 2. 설계 원칙

| # | 원칙 | 구현 |
| --- | --- | --- |
| D1 | **주행 중 = 음성, 정차 중 = 화면** | 주행 중 화면은 큰 글씨 1줄 + 버튼 0~1개. 상세는 정차 시에만 |
| D2 | **운전자 상호작용 최소화** | 경고 발생 시 **터치 0회**로 브리핑 완료. 응답은 STT로 |
| D3 | **안전 정보는 오프라인** | 경고등 DB를 앱에 번들. 네트워크 없이 동작 ([dev-environment.md](dev-environment.md) §4.6) |
| D4 | **미지원은 조용히 숨긴다** | `checkXXXCapabilities` 실패 항목은 UI에서 제거. 모델명 분기 금지 |
| D5 | **읽기 전용** | 차량 제어 API(`CONTROL_CAR_*`)를 일절 사용하지 않음 → 심사 리스크 최소화 |
| D6 | **신호 소스 추상화** | `SignalSource` 인터페이스로 분리 → 공개 API / (검증 성공 시) Telltale / 시연용 Fake를 교체 가능 |

---

## 3. ⭐ 신호 매핑 — 공개 API로 감지 가능한 경고 13종

**이것이 MVP의 실제 기반입니다.** 모든 항목이 `pleos.car.permission.CAR_SAFETY` 등 **공개 권한(normal)** 으로 접근 가능합니다.

| # | 경고 | 공개 API | 리스너 | `TelltaleType` 대응 | 긴급도 |
| --- | --- | --- | --- | --- | --- |
| 1 | 엔진 출력 제한 | `Safety.hasPowerDownWarningIndicator` | ✅ `registerPowerDownWarningIndicator` | `POWER_DOWN`(24) | **1 즉시** |
| 2 | 에어백 이상 | `Safety.hasSeatAirbagEnabled(area)` | ✅ `registerSeatAirbagEnabled` | `AIRBAG`(10) | **1 즉시** |
| 3 | 에어백 전개(사고) | `Safety.hasSeatAirbagBeenDeployed(area)` | ✅ `registerSeatAirbagDeployed` | `EMERGENCY_CALL`(22) | **1 즉시** |
| 4 | 브레이크액 부족 | `Safety.getBrakeFluidLevelState` | ✅ `registerBrakeFluidLevelState` | — (제동계) | **1 즉시** ※`CRITICALLY_LOW` |
| 5 | 엔진오일 부족 | `Safety.getEngineOilLevelState` | ✅ `registerEngineOilLevelState` | `ENGINE_OIL_PRESSURE`(20) | 1 또는 2 ※레벨별 |
| 6 | 변속기오일 이상 | `Safety.getTransmissionOilLevelState` | ✅ `registerTransmissionOilLevelState` | — (구동계) | 2 점검 |
| 7 | 타이어 공기압 부족 | `Tire.getLowTirePressure(area)` | ❌ **polling** | `TPMS`(6) | 2 점검 |
| 8 | 워셔액 부족 | `Safety.hasLowWasherFluidWarning` | ✅ `registerLowWasherFluidWaringState` | — (소모품) | 3 참고 |
| 9 | 안전벨트 미체결 | `Safety.hasSeatBeltNotBuckledWarning(area)` | ✅ `registerSeatBeltNotBuckledWarning` | `SEAT_BELT`(2) | **1 즉시** |
| 10 | 연료 부족 | `Fuel.getFuelPercent` | ✅ `registerFuelPercent` | `LOW_FUEL`(117) | 2 점검 |
| 11 | 구동 배터리 부족 | `EvBattery.getEvBatteryStateOfCharge` | ✅ `registerEvBatteryStateOfCharge` | `LOW_EV_BATTERY`(104) | 2 점검 |
| 12 | 도어 열림 | `Door.getCurrentDoorPosition(area)` | ✅ `registerDoorPosition` | `DOOR_OPEN`(3) | **1 즉시** |
| 13 | 창문 열림(주차 시) | `Window.getCurrentWindowPosition(area)` | ✅ register 있음 | — | 3 참고 |

**보조 컨텍스트 신호** (브리핑 톤과 조치 결정에 사용)

| 용도 | API |
| --- | --- |
| 정차/주행 판별 | `Gear.getCurrentGearSelectionState` + `Brake.getCurrentBrakePedalStatus` |
| 주행거리·트립 | `Odometer.getCurrentDistanceDriven`, `getTripCurrentDrivingDistance`, `getTripCurrentDrivingTime` |
| 주행가능거리 | `Odometer.getRangeRemainingDistance` |
| 차량 식별 | `CarInfo.getVINInfo`, `getCRNInfo` |
| 우천(가시성) | `Wiper.getCurrentWiperState` |

> ⚠️ **`OilLevelState` enum:** `CRITICALLY_LOW` / `LOW` / `NORMAL` / `HIGH` / `ERROR` → 긴급도 분기의 핵심 값입니다.
> ⚠️ **#7 타이어는 리스너가 없어 polling 필요** (`getLowTirePressure`). 주행 중 30초 간격 polling으로 설계.

### 3.1 🔓 상위 경로 (검증 성공 시 자동 확장)

`Safety.getTelltalesInfo`는 어노테이션이 **공개 권한 `CAR_SAFETY`** 입니다([dev-environment.md](dev-environment.md) §4.1.1). 호출이 성공하면:

- 감지 범위가 **13종 → `TelltaleType` 78종 전체**로 확대
- 플랫폼이 제공하는 **`color`·`priority` 값으로 긴급도를 직접 산출** (자체 추정 불필요)

**→ D6의 `SignalSource` 추상화 덕분에 이 경우 UI·DB 변경 없이 소스만 교체하면 됩니다. 실패해도 13종으로 정상 동작합니다.**

---

## 4. 기능 명세

### F1. 경고 감지 → 3단계 긴급도 음성 브리핑 ⭐ 코어

**트리거:** 공개 리스너 9종 콜백 + 타이어/연료 polling

**브리핑 3요소** — 항상 이 순서, 터치 0회

```
① 무슨 일인가   (1문장, 전문용어 없이)
② 지금 어떻게 할까 (긴급도별 1문장)
③ 다음 행동 제안   (Level 1~2만, "안내해 드릴까요?")
```

**긴급도 판정 규칙**

| 레벨 | 판정 조건 | TTS 톤 | 화면 | 후속 |
| --- | --- | --- | --- | --- |
| **1 즉시 정차** | 출력제한 true / 에어백 이상·전개 / 브레이크액 `CRITICALLY_LOW` / 엔진오일 `CRITICALLY_LOW` / 안전벨트 미체결 / 주행 중 도어 열림 | 즉시 개입, 단문, 반복 1회 | 적색 전체 배너 | "가까운 정비소로 안내해 드릴까요?" |
| **2 오늘 중 점검** | 엔진오일 `LOW` / 변속기오일 `LOW`·`ERROR` / 브레이크액 `LOW` / 타이어 공기압 부족 / 연료 15% 이하 / SOC 15% 이하 | 차분한 안내 | 황색 상단 토스트 | "경로에 정비소를 추가할까요?" |
| **3 참고** | 워셔액 부족 / 창문 열림 / 상태 표시 계열 | 알림음 없이 | 대시보드 배지만 | 없음 |

**TTS 스크립트 예시**

| 경고 | Level | 스크립트 |
| --- | --- | --- |
| 출력 제한 | 1 | "엔진 출력이 제한되고 있습니다. 비상등을 켜고 안전한 곳에 차를 세워 주세요. 가까운 정비소로 안내해 드릴까요?" |
| 브레이크액 `CRITICALLY_LOW` | 1 | "브레이크액이 매우 부족합니다. 제동력이 떨어질 수 있으니 속도를 줄이고 바로 차를 세워 주세요." |
| 엔진오일 `LOW` | 2 | "엔진오일이 부족합니다. 당장 위험하진 않지만 오늘 중 점검을 권합니다. 경로에 정비소를 추가할까요?" |
| 타이어 공기압 부족 | 2 | "뒤 오른쪽 타이어 공기압이 낮습니다. 가까운 정비소에서 확인해 주세요." |
| 워셔액 부족 | 3 | *(음성 없음 — 대시보드 배지만)* |

**주의분산 대응** ([dev-environment.md](dev-environment.md) §4.7)
- 주행 중 화면: **최대 2줄 텍스트 + 버튼 0개.** 응답은 STT로만 받음
- Level 3는 주행 중 **음성·시각 알림 모두 억제** → 정차 시 묶어서 안내
- 모든 액티비티에 Distraction Optimized 메타데이터 선언

---

### F2. 정차 시 차량 컨디션 대시보드

**트리거:** `Gear` = P (정차 판별)

| 영역 | 내용 |
| --- | --- |
| 상단 | **종합 상태 1줄** — "지금 점검이 필요한 항목 2개" + 긴급도 색상 |
| 중앙 | 신호 13종 카드 그리드. 각 카드 = 아이콘 / 상태 / 한 줄 설명 |
| 카드 탭 | 상세 시트 — 의미 / 그대로 두면 / 조치 단계 / **하지 말 것** / "공식 설명서에서 보기" (딥링크) |
| 하단 | 트립 요약 (`Odometer`) + 주행가능거리 |

**규칙**
- `checkXXXCapabilities` 실패 항목은 **카드를 만들지 않음** (원칙 D4)
- `getXXX`가 `null` 반환 → "정보 없음" 카드 (오류 아님) ([dev-environment.md](dev-environment.md) §4.8)
- 주행 중에는 이 화면으로 진입 불가

---

### F3. STT 음성 명령 → 가까운 정비소 경유지 추가

**흐름**

```
[F1 Level 1~2 브리핑 종료]
   ↓ TTS: "가까운 정비소로 안내해 드릴까요?"
   ↓ STT 수신 ("네" / "경로에 추가해줘" / "아니요")
   ↓ 긍정 시
NaviHelper.addWaypoint(RequestWaypointInfo)      ← 경로 중이면 경유지 추가
NaviHelper.requestRoute(RouteInfo)               ← 경로 없으면 목적지 설정
   ↓ TTS: "경로에 정비소를 추가했습니다. 약 N분 뒤 도착합니다."
   ↑ NaviHelper.getRouteStateInfo / getTBTInfo 로 ETA 획득
```

| 항목 | 내용 |
| --- | --- |
| 필요 권한 | `pleos.car.permission.NAVI_ROUTE_SEARCH` (addWaypoint·requestRoute) + `NAVI_ROUTE` (ETA 조회) |
| 시스템 앱 제약 | ✅ **없음** — `addWaypoint`는 서드파티 호출 가능 ([dev-environment.md](dev-environment.md) §3.6) |
| 정비소 데이터 | **1순위** 차량 내장 내비의 POI 검색 결과 활용 / **2순위** 공공데이터 정비업체 좌표를 시연 지역만 번들 ([sources.md](sources.md) §3.4) |
| 제약 | NaviHelper는 **내비게이션 탑재 차량에서만 전체 호환** → 미탑재 시 F3을 숨기고 F1·F2만 동작 |
| 금지 | 블루핸즈 등 웹사이트 크롤링 ([sources.md](sources.md) §4) |

> ⚠️ **미검증:** `RequestWaypointInfo`가 좌표를 받는지 POI ID를 받는지. **SDK 소스 확인 또는 에뮬레이터 실측 필요.**

---

### F4. 경고 이력 리포트 + 리콜 조회

| 기능 | 내용 |
| --- | --- |
| 이력 기록 | 경고 발생 시각 / 종류 / 긴급도 / 당시 주행거리(`Odometer`)를 로컬(Room)에 저장 |
| 리포트 | 정차 시 "최근 30일 경고 요약" — 반복 경고 강조 ("엔진오일 부족이 3회 반복됨") |
| 공유 | 정비소 방문용 요약을 **QR 또는 텍스트**로 출력 |
| 리콜 조회 | `CarInfo.getVINInfo` → **국토교통부 자동차 리콜정보 OpenAPI** 조회 → 해당 차량 리콜·무상수리 여부 표시 |

**🚨 리콜 API 준수사항** ([sources.md](sources.md) §3.1 — 반드시 구현)
1. **"자동차리콜센터" 출처표시**를 화면에 노출
2. 상세내용은 **API가 제공하는 상세 URL로 리콜센터에 연결** (앱 내 재구성 금지)
3. **API 호출 건수·사용자 수를 주기적으로 제공**

> ⚠️ 리콜 OpenAPI 문서가 2021년 최종수정이므로 **활용신청 후 실제 명세(요청 변수·응답 필드·엔드포인트) 확인 필수.**
> ⚠️ VIN은 개인정보에 준해 취급. 앱 제출 시 **개인정보 신고 + 처리방침 URL 필수** ([dev-environment.md](dev-environment.md) §4.11).

---

## 5. 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│ UI (Compose)                                            │
│  DrivingBriefScreen (주행)  │  DashboardScreen (정차)   │
│  HistoryScreen              │  DetailSheet              │
└──────────────┬──────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────┐
│ Domain                                                  │
│  SeverityClassifier   경고 → 1/2/3 판정                  │
│  BriefComposer        경고 + DB → TTS 스크립트 생성       │
│  DriveStateDetector   Gear/Brake → 주행 vs 정차          │
│  ActionRouter         Level 1~2 → F3 제안 트리거          │
└──────────────┬──────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────┐
│ Data                                                    │
│  ⭐ SignalSource (interface)         ← 원칙 D6           │
│     ├ PublicApiSignalSource  Safety/Tire/Fuel/EvBattery │
│     ├ TelltaleSignalSource   §3.1 검증 성공 시 사용       │
│     └ FakeSignalSource       시연·단위테스트용            │
│  TelltaleInfoRepository  번들 JSON (오프라인)            │
│  WarningHistoryDao       Room                            │
│  RecallApiClient         국토부 OpenAPI (+ 캐싱·재시도)    │
└──────────────┬──────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────┐
│ Platform                                                │
│  VehicleSession  Vehicle 2.3.2 (initialize/release)     │
│  NaviGateway     NaviHelper 3.0.5                       │
│  SpeechGateway   TextToSpeech 3.3.1 / SpeechToText 3.3.1│
└─────────────────────────────────────────────────────────┘
```

### 5.1 `SignalSource` 인터페이스 (설계 핵심)

```kotlin
data class VehicleWarning(
    val type: String,          // TelltaleType 이름과 동일한 키 (예: "POWER_DOWN")
    val active: Boolean,
    val severity: Int,         // 1 | 2 | 3
    val area: String? = null,  // 좌석/휠/도어 영역
    val rawLevel: String? = null,  // OilLevelState 등
    val occurredAt: Long
)

interface SignalSource {
    suspend fun start(onWarning: (VehicleWarning) -> Unit)
    suspend fun snapshot(): List<VehicleWarning>   // F2 대시보드용
    suspend fun stop()
}
```

**→ 경고 타입 키를 `TelltaleType` enum 이름과 **동일하게** 두는 것이 핵심입니다.** DB([sources.md](sources.md) §6.2)와 UI가 그대로 재사용되므로, §3.1 검증이 성공하면 `PublicApiSignalSource`를 `TelltaleSignalSource`로 바꾸는 것만으로 13종 → 78종이 됩니다.

### 5.2 안정성 규칙 (필수 준수)

| 규칙 | 근거 |
| --- | --- |
| `Vehicle.initialize()` 완료 후에만 property API 호출. 실패 시 중단 | [dev-environment.md](dev-environment.md) §2.4 |
| 모든 도메인에서 `checkXXXCapabilities` 선행, 보고된 범위만 사용 | §4.8 |
| `getXXX`의 `null` 반환을 정상 경로로 처리 | §4.8 |
| `register`/`unregister`에 **동일 리스너 인스턴스**, `release()` 전 전부 해제 | §2.4 |
| `SecurityException`·미지원 도메인에서 **앱이 죽지 않게** 처리 | §3.1 |
| 비동기 체인은 coroutine으로 순차화 | §2.4 |

---

## 6. 경고등 정보 DB

| 항목 | 내용 |
| --- | --- |
| 형식 | 앱 번들 JSON (오프라인, 원칙 D3) |
| 스키마 | [sources.md](sources.md) §6.2 |
| 키 | `TelltaleType` enum 이름 |
| **MVP 범위** | **§3의 13종만 작성** (약 0.5일). 78종 전체는 PT 심사 전까지 확장 |
| 본문 | 자체 작성 (ISO 2575 색상 체계 기반) |
| "자세히" | `ownersmanual.hyundai.com` 딥링크 |
| 금지 | 매뉴얼 본문·아이콘 복제 ([sources.md](sources.md) §6.3) |

---

## 7. 에뮬레이터 시연 시나리오 (가산점·PT 심사용)

**사전 준비:** Pleos Connect v2.1.0 (cold boot) + CRN 주입 + Vehicle Service ≥ 2.3.13 ([dev-environment.md](dev-environment.md) §1)

| # | 장면 | 조작 | 기대 결과 | 길이 |
| --- | --- | --- | --- | --- |
| 1 | 정상 주행 | — | 대시보드 전체 녹색 | 10s |
| 2 | **Level 2 발생** | VHAL로 엔진오일 → `LOW` | 황색 토스트 + TTS "엔진오일이 부족합니다…" | 15s |
| 3 | **음성 응답** | "경로에 정비소 추가해줘" (STT) | `addWaypoint` 실행 + TTS ETA 안내 | 15s |
| 4 | **Level 1 발생** | VHAL로 출력제한 → `true` | 적색 전체 배너 + 즉시 정차 TTS | 15s |
| 5 | **정차 전환** | Gear → P | 대시보드 자동 전환, 상세 시트 | 15s |
| 6 | **이력·리콜** | 이력 탭 | 경고 3건 타임라인 + 리콜 조회 결과 | 15s |

> ⚠️ **미검증:** Virtual Console의 VHAL 주입 UI로 Safety/Tire 프로퍼티를 직접 조작 가능한지. **불가하면 `FakeSignalSource`로 시연**하고, 영상에 "에뮬레이터 신호 주입 한계로 일부는 시뮬레이션"을 명시 (정직성 확보).
> 📌 **숏폼 챌린지**(세로 30초~1분, `#HMG차량용앱개발공모전`)용으로 장면 2·3·4만 재편집해 재활용 ([info.md](info.md) §3).

---

## 8. 일정 (D-12 → PT)

### Phase 1 — 1차 접수 (11/03 ~ 11/15)

| 기간 | 작업 | 산출물 |
| --- | --- | --- |
| 11/03 ~ 11/04 | Playground 가입 → My Project → **CRN 발급**, 에뮬레이터 구축, 공식 샘플 빌드 | 동작하는 개발환경 |
| 11/05 | 🔑 **`getTelltalesInfo` 공개 권한 검증** + 공개 Safety 9종 실측 | 신호 가용성 확정표 |
| 11/06 ~ 11/07 | `SignalSource` + `VehicleSession` + 13종 DB 작성 | 신호 수신 동작 |
| 11/08 ~ 11/09 | **F1 코어** (판정 + TTS 브리핑) | 음성 브리핑 동작 |
| 11/10 ~ 11/11 | **F2 대시보드** | 정차 화면 |
| 11/12 | **F3** STT + `addWaypoint` | 경유지 추가 동작 |
| 11/13 | 시연 영상 촬영·편집 + release APK 서명 | `팀명_앱이름_release.apk` |
| 11/14 | GitHub 정리 (`apk/` 폴더 + README 4항목), 접수양식 작성 | 제출 패키지 |
| **11/15** | **이메일 접수** (팀원 전원 CC, 서명 동의서 첨부) | ✅ 1차 제출 |

> 📌 F4는 1차 접수 범위에서 제외하고 **기획서에 계획으로만 기술**합니다. 개발 산출물은 선택 제출이고, **PT 심사 전까지 자유롭게 고도화 가능**하기 때문입니다 ([info.md](info.md) §4).

### Phase 2 — PT 심사 (11/23 결과 발표 ~ 12/03)

| 기간 | 작업 |
| --- | --- |
| 11/24 ~ 11/26 | **F4** 이력 리포트 + 리콜 API 연동 (활용신청 선행) |
| 11/27 ~ 11/29 | 경고등 DB **78종 전체**로 확장 ([sources.md](sources.md) §2) |
| 11/30 ~ 12/01 | Distraction Optimized 검증, 안정화, 엣지케이스(null·SecurityException·미지원) |
| 12/02 | PT 자료 + 라이브 시연 리허설 |
| **12/03 ~ 04** | **오프라인 PT 심사** (현대차 강남대로사옥 UX스튜디오 서울) |

---

## 9. 심사 기준 매핑 ([info.md](info.md) §6)

| 심사 항목 | 본 기획의 답 |
| --- | --- |
| **차량특화성** (필수) | 경고 신호·소모품 상태·좌석 센서는 **폰으로 취득 불가**. 주행/정차를 `Gear`·`Brake`로 판별해 **주행 중엔 음성, 정차 중엔 화면**으로 분리 → 차량 맥락과 유기적 결합 (§1.4, §2) |
| **창의성** (필수) | 가이드 예시 3종(음성메모·안전운전점수·배달동선)과 무관. "계기판은 **아이콘만** 보여주고 조치는 알려주지 않는다"는 미해결 페인포인트를 **차가 먼저 말을 거는 방식**으로 해결 (§1.2) |
| **기술 활용성** (필수) | **Vehicle SDK**(Safety·Tire·Fuel·EvBattery·Odometer·Gear·Door·Window·Brake 9개 도메인) + **TTS**(브리핑) + **STT**(응답) + **NaviHelper**(경유지) = **4개 SDK를 핵심 기능에** 사용. 권한 정책·capability 체크·세션 라이프사이클까지 공식 규약 준수 (§3, §5.2) |
| **사업성** (필수) | 타깃 3계층 명확(§1.3). Retention = **경고는 반복 발생**하고 이력이 쌓일수록 가치 증가. 확장: 정비 네트워크 예약 연계, 렌터카·카셰어링 B2B, 법인 차량 관리 |
| **[가산점] 개발 산출물** | 1차 접수 시 **시연 영상 + GitHub(소스+APK)** 동시 제출 (§8 Phase 1) |

---

## 10. 범위 제외 (Out of Scope)

| 제외 항목 | 이유 |
| --- | --- |
| 차량 제어 (`CONTROL_CAR_*`) | 원칙 D5 — 심사 리스크. 읽기 전용 유지 |
| OBD-II DTC 고장코드 | SDK가 노출하지 않고, 표준이 유료 ([sources.md](sources.md) §5) |
| 타이어 공기압 **실측값** | `getTirePressure`는 `PRIVATE_CAR_SAFETY` ([dev-environment.md](dev-environment.md) §4.1) |
| 블루핸즈 실시간 예약 | 공개 API 없음, 크롤링 금지 ([sources.md](sources.md) §4) |
| LLM 기반 대화형 설명 | 공모전 안내에 LLM SDK 미기재 + 안전 정보 환각 리스크 |
| ADAS 신호 활용 | ADAS SDK 미제공 ([dev-environment.md](dev-environment.md) §4.2) |
| 서버 백엔드 | 리콜 API 외 외부 의존 없음. 오프라인 우선 (원칙 D3) |

---

## 11. 리스크 및 대응

| # | 리스크 | 영향 | 대응 |
| --- | --- | --- | --- |
| R1 | `getTelltalesInfo`가 `SecurityException` | 중 | **이미 대응 완료** — 13종 공개 API만으로 전 기능 동작 (§3) |
| R2 | VHAL 주입으로 Safety 프로퍼티 변경 불가 | **고** | `FakeSignalSource`로 시연 + 영상에 명시 (§7) |
| R3 | 에뮬레이터에 NaviHelper 미탑재 | 중 | F3을 조건부 기능으로 설계, 미탑재 시 숨김. 시연은 F1·F2 중심 |
| R4 | 영역별 가용성 차이 (Door Lock `ROW_1_LEFT` 실패 사례) | 중 | `checkXXXCapabilities`로 영역 선별, 미지원 영역 숨김 (원칙 D4) |
| R5 | STT 인식률 (에뮬레이터 마이크) | 중 | 짧은 긍정어("네") 중심 설계 + 터치 폴백 1버튼 제공 |
| R6 | Distraction Optimized 심사 거절 | **고** | 주행 중 2줄·버튼 0개 원칙 엄수, UI Design Guide 확보 후 재검토 |
| R7 | 리콜 API 명세가 문서와 다름 | 저 | F4를 Phase 2로 미룸. 실패 시 ecar.go.kr 안내 링크로 대체 |
| R8 | 경고등 설명의 안전성 오류 | **고** | 공식 매뉴얼 교차검증 + "하지 말 것" 항목 명시 + 딥링크 병기 ([sources.md](sources.md) §6) |
| R9 | 12일 일정 부족 | 중 | F4를 Phase 2로 분리. **기획서만으로도 접수 가능**([info.md](info.md) FAQ)하므로 F1 하나만 되면 가산점 확보 |

---

## 12. 확인 필요 (결정 대기 / 미검증)

**개발 착수 즉시 검증**
- [ ] `getTelltalesInfo`를 `CAR_SAFETY`만으로 호출 성공 여부 (§3.1) — **최우선**
- [ ] Virtual Console에서 Safety/Tire 프로퍼티 VHAL 주입 가능 여부 (R2)
- [ ] `RequestWaypointInfo`의 입력 형식 (좌표 vs POI ID) (F3)
- [ ] 에뮬레이터의 NaviHelper 탑재 여부 (R3)
- [ ] `TelltaleData.color` 인코딩 / `priority` 범위

**기획 확정 필요 (사용자 컨펌 대기)**
- [ ] **앱 이름** — 현재 미정. "경고등 통역사", "Telltale Talk" 등은 작업명
- [ ] **팀명 / 참가 부문** (대학생 / 일반)
- [ ] **숏폼 챌린지 참여 여부** (선택, 가산점 없음, 별도 상금 50만원)
- [ ] F2 대시보드에 **EV 전용 항목**(충전 상태 등)을 넣을지 — 시연 차종이 EV인지에 따라 결정
- [ ] 긴급도 판정 임계값: 연료/SOC **15%** 기준이 적절한지 (현재 임의 설정)

**외부 절차**
- [ ] 국토부 리콜 OpenAPI **활용신청** → 실제 명세 확보 (F4)
- [ ] Pleos Connect **UI Design Guide** 원문 확보 (R6)
- [ ] 온라인 설명회 다시보기 시청 — 개발환경 관련 추가 제약 확인 ([info.md](info.md) §2)
