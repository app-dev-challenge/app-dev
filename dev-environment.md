# 개발환경 조사 — Pleos Connect 인앱 구현 (제한사항 중심)

> **조사일:** 2026-11-03
> **검증 방법:** 공식 문서 + 공식 샘플 앱 소스 + **Maven 저장소에서 SDK 소스 직접 취득·분석**
> **관련 문서:** [info.md](info.md) · [idea.md](idea.md) · [sources.md](sources.md) · [mvp-plan.md](mvp-plan.md)

---

## 0. 한 장 요약

| 항목 | 값 | 근거 |
| --- | --- | --- |
| 플랫폼 | Android Automotive OS (AAOS) 기반 Pleos Connect | 공식 문서 |
| 에뮬레이터 이미지 | **Pleos Connect v2.1.0 / API 34** | 공식 셋업 가이드 |
| minSdk | **28 권장** (SDK 자체는 23) | §1.1 불일치 정리 |
| compileSdk / targetSdk | **35** | 공식 샘플 앱 |
| JDK | **17** | 공식 샘플 앱 |
| Gradle | **8.0 이상** | 공식 문서 |
| 언어 | Kotlin (stdlib 1.9.25) 또는 Java | Vehicle POM |
| Maven 저장소 | `https://nexus-playground.pleos.ai/repository/maven-releases/` | 공식 샘플 README |
| Vehicle SDK 최신 | **2.3.2** (필요 Vehicle Service ≥ **2.3.13**) | Maven metadata + AAR manifest |
| 🚨 최대 제약 | **경고등(Telltale)·타이어 공기압 API = `signature\|privileged` 권한** → 서드파티 불가 | §4.1 (SDK 소스 확인) |
| 🚨 두 번째 제약 | **ADAS SDK는 Maven 저장소에 없음** (미제공) | §4.2 |

---

## 1. 확정된 요구사항

### 1.1 ⚠️ API 레벨 — 출처별 불일치 정리

조사 중 **세 가지 다른 값**이 확인되었습니다.

| 출처 | 명시된 값 |
| --- | --- |
| 공모전 안내 ([info.md](info.md) §5) | Android API **28 이상** |
| Pleos 에뮬레이터 셋업 문서 | "Minimum Android SDK는 **API 28 이상**, 차량 사이버보안(서명키 rotation·lineage 지원) 때문" |
| Pleos 개발환경 설정 문서(일부 페이지) | Android API **26 이상** |
| 공식 샘플 앱 `build.gradle` | `minSdk = 23` |
| `Vehicle-2.3.2.aar` 매니페스트 | `<uses-sdk android:minSdkVersion="23" />` |

**→ 권고: `minSdk = 28`**
이유: ① 공모전 안내가 28로 명시 ② 사이버보안(서명키 rotation) 근거가 구체적 ③ SDK 자체가 23이므로 28은 상위 호환으로 안전. 샘플의 23은 SDK 하한을 그대로 쓴 것일 뿐 심사 기준이 아닙니다.

### 1.2 빌드 환경 (공식 샘플 앱 검증값)

| 항목 | 값 |
| --- | --- |
| JDK | 17 (`JAVA_HOME=/path/to/jdk-17 ./gradlew ...`) |
| Android SDK Platform | 35 |
| `minSdk` | 23 (샘플) → **28 권장** |
| `targetSdk` | 35 |
| Gradle | 8.0 이상 |
| Kotlin stdlib | 1.9.25 (Vehicle 2.3.2 의존성) |
| 버전 규약 (샘플) | `versionName = 1.0.0-sdk35`, `versionCode = 35010000` (`targetSdk*1,000,000 + major*10,000 + minor*100 + patch`) |

### 1.3 에뮬레이터 셋업 절차

1. 최신 Android Studio 설치 · [Android Emulator 시스템 요구사항](https://developer.android.com/studio/run/emulator) 충족 확인
2. **기존 Pleos Connect Emulator 제거**
3. **Tools > SDK Manager > SDK Update Sites** → 사이트 추가 (이름: `Pleos Connect System Image`)
   - macOS: `...crp-arm64-sys-img.xml`
   - Windows / Linux: `...crp-sys-img.xml`
4. **SDK Platforms** 탭 → `Show Package Details` 체크 → **API 34용 Pleos Connect 시스템 이미지** 설치 → Android Studio 재시작
5. **Tools > Device Manager > Create device** → Form Factor **Automotive** → **Pleos Connect v2.1.0**
6. **Advanced Settings** → `Startup > Default boot` = **Cold** ⚠️ **필수**
7. `Emulated Performance` → **RAM 4 GB** (구버전 Android Studio는 `Graphics = Hardware`도 설정)
8. Device 탭 → **API 34** + **Pleos Connect v2.1.0 APIs** → Finish → 실행

### 1.4 🔑 CRN 주입 (앱 실행 전 필수)

앱을 구동하려면 에뮬레이터에 **CRN(Connected Resource Name)** 을 주입해야 합니다. CRN은 [Pleos Playground](https://pleos.ai/playground/console/projects)에서 **My Project**를 생성하면 발급됩니다.

```bash
adb root
# propId: 554696961 / areaId: 0 / values: {CRN} 를
# /data/vendor/vsomeip/vhal_fifo 에 기록
# → 주입 후 에뮬레이터 재시작
```

> ⚠️ **AVD의 Boot option이 quick boot면 VHAL을 사용할 수 없습니다.** 반드시 **cold boot**. 실행 시 `-no-snapshot` 옵션이 함께 사용됩니다.
> ⚠️ 위 명령은 문서 버전에 따라 표기가 다릅니다. **실제 명령은 사용 중인 버전의 공식 문서에서 재확인하세요.**

### 1.5 🔑 SDK ↔ Vehicle Service 버전 매트릭스

`Vehicle-2.3.2.aar` 매니페스트에서 직접 확인한 값입니다. **에뮬레이터의 Vehicle Service 버전이 낮으면 SDK가 동작하지 않습니다.**

| 메타데이터 | 값 |
| --- | --- |
| `service.vehicle.min_version_name` | **2.3.13** |
| `service.vehicle.min_version_code` | 65 |
| `sdk.vehicle.current_version_name` | 2.3.2 |
| `sdk.vehicle.current_version_code` | 73 |

공식 샘플 검증 환경: **`Pleos_Connect_v2.1` (Android 14) + Vehicle SDK 2.3.2 + Vehicle Service 2.3.22**
※ Virtual Console 사용에는 **에뮬레이터 2.1.0 이상** 필요.

---

## 2. 의존성 설정 (바로 사용 가능)

### 2.1 Gradle

```kotlin
// settings.gradle.kts 또는 build.gradle.kts
repositories {
    maven("https://nexus-playground.pleos.ai/repository/maven-releases/")
}

dependencies {
    implementation("ai.pleos.playground:Vehicle:2.3.2")
    implementation("ai.pleos.playground:NaviHelper:3.0.5")
    implementation("ai.pleos.playground:TextToSpeech:3.3.1")
    implementation("ai.pleos.playground:SpeechToText:3.3.1")
    // implementation("ai.pleos.playground:FusedLocation:1.1.3")
    // implementation("ai.pleos.playground:LLM:2.1.3.2")
}
```

### 2.2 ✅ Maven 저장소에서 실제 확인한 아티팩트 (2026-11-03)

| SDK | 제공 버전 | 최신 | 소스 jar | 비고 |
| --- | --- | --- | --- | --- |
| `Vehicle` | 2.0.3, **2.3.2** | 2.3.2 | ✅ | 공모전 안내의 "Vehicle SDK" |
| `NaviHelper` | 2.0.2, 2.0.3, **3.0.5** | 3.0.5 | ✅ | 메이저 버전이 3으로 올라감 |
| `TextToSpeech` | 2.1.2 ~ 2.1.5.1, 3.3.0, **3.3.1** | 3.3.1 | ❌ | |
| `SpeechToText` | 2.1.2 ~ 2.1.3.2, 3.3.0, **3.3.1** | 3.3.1 | ❌ | |
| `FusedLocation` | 1.0.4, **1.1.3** | 1.1.3 | ✅ | 문서상 "준비 중"이나 **아티팩트는 존재** |
| `LLM` | 2.1.2, 2.1.3.1, **2.1.3.2** | 2.1.3.2 | ❌ | 공모전 안내엔 미기재이나 **배포되고 있음** |
| `Adas` / `ADAS` | — | — | — | ❌ **저장소에 없음** |
| `Sync` / `Gleo` | — | — | — | ❌ **저장소에 없음** |

> 📌 **저장소는 인증 없이 접근 가능**하며, `maven-metadata.xml`과 `-sources.jar`를 내려받아 **API를 코드 레벨에서 사전 검증할 수 있습니다.** 본 문서의 §4 제약사항 대부분이 이 방법으로 확인되었습니다.
> ```bash
> curl -sfL "https://nexus-playground.pleos.ai/repository/maven-releases/ai/pleos/playground/Vehicle/maven-metadata.xml"
> curl -sfLO "https://nexus-playground.pleos.ai/repository/maven-releases/ai/pleos/playground/Vehicle/2.3.2/Vehicle-2.3.2-sources.jar"
> ```

### 2.3 Vehicle SDK 런타임 의존성 (POM 확인)

`kotlin-stdlib 1.9.25`, `ai.pleos.playground:vehicle-aidl:1.1.1`, `ai.pleos.playground:vehicle-data:1.1.11`, `androidx.core:core-ktx:1.8.0`

### 2.4 세션 라이프사이클 (공식 샘플 기준)

```kotlin
val vehicle = Vehicle(context.applicationContext)
vehicle.initialize()           // 반드시 완료 후에만 property API 호출. 실패하면 중단.

val safety = vehicle.getSafety()
// checkXXXCapabilities → getXXX / registerXXX / unregisterXXX

vehicle.release()              // 소유 컴포넌트 destroy 시 호출
```

**리스너 규칙 (반드시 준수)**
- `register`와 `unregister`에 **같은 리스너 인스턴스**를 전달
- 세션 `release()` **전에** 모두 unregister — 리스너가 세션보다 오래 살면 안 됨
- 모든 호출이 비동기 → 다음 단계는 이전 콜백의 성공 지점에서 시작 (또는 coroutine / state machine)

---

## 3. 권한 체계 (가장 중요)

### 3.1 기본 규칙

| 항목 | 내용 |
| --- | --- |
| 네임스페이스 | `pleos.car.permission.{상수명}` |
| 선언 위치 | `AndroidManifest.xml` 의 `<uses-permission>` |
| 명명 규약 | 읽기 = `CAR_*` / 제어 = `CONTROL_CAR_*` / **비공개 = `PRIVATE_CAR_*`** |
| 공개 권한의 보호 수준 | **`protectionLevel="normal"`** → **런타임 권한 다이얼로그 없음** |
| 비공개 권한의 보호 수준 | **`signature\|privileged`** → **서드파티 앱 획득 불가** |
| 사용 범위 | **Pleos App Market과 사전 협의·승인된 범위 내에서만** 사용 가능 |
| 🚨 위반 결과 | **미승인 권한을 선언하거나 해당 API를 호출하면 앱 심사에서 탈락** |
| 승인 절차 | 앱 제출 시 **Playground 프로젝트 ID**(My Project > Project Information > Project ID) 제출 |

### 3.2 공개 읽기 권한 (`protectionLevel = normal`) — 사용 가능

`PleosPermission.kt` 소스에서 확인한 전체 목록입니다.

```
CAR_CLIMATE   CAR_DOORS    CAR_DRIVE    CAR_ENERGY   CAR_INFO
CAR_LIGHTS    CAR_MIRRORS  CAR_SAFETY   CAR_SEATS    CAR_TOLL     CAR_WINDOWS
```

공식 샘플 매니페스트가 추가로 선언하는 권한:
```
CAR_MILEAGE   CAR_POWERTRAIN   CAR_SPEED   CAR_TIRES
READ_CAR_DISPLAY_UNITS   READ_CAR_STEERING
READ_ADAS_STATES   READ_WINDSHIELD_WIPERS   CONTROL_WINDSHIELD_WIPERS
CAR_CONNECT_VENDOR_CONTROL
```

### 3.3 공개 제어 권한 (`CONTROL_CAR_*`)

```
CONTROL_CAR_CLIMATE  CONTROL_CAR_DOORS   CONTROL_CAR_DRIVE   CONTROL_CAR_ENERGY
CONTROL_CAR_INFO     CONTROL_CAR_LIGHTS  CONTROL_CAR_MIRRORS CONTROL_CAR_SAFETY
CONTROL_CAR_SEATS    CONTROL_CAR_TOLL    CONTROL_CAR_WINDOWS
```

### 3.4 🚨 비공개 권한 (`PRIVATE_CAR_*`, `signature|privileged`) — **서드파티 획득 불가**

```
PRIVATE_CAR_CLIMATE  PRIVATE_CAR_DOORS   PRIVATE_CAR_DRIVE   PRIVATE_CAR_ENERGY
PRIVATE_CAR_INFO     PRIVATE_CAR_LIGHTS  PRIVATE_CAR_MIRRORS PRIVATE_CAR_SAFETY
PRIVATE_CAR_SEATS    PRIVATE_CAR_TOLL    PRIVATE_CAR_WINDOWS
```

> 📌 공식 샘플 README도 명시: *"It does not declare any `PRIVATE_*` permission."*

### 3.5 도메인별 비공개 API 분포 (SDK 2.3.2 소스 전수 조사)

| 도메인 | 전체 함수 | `PRIVATE_*` 요구 | 영향 |
| --- | --- | --- | --- |
| **Tire** | 7 | **8** (실질 4개 함수) | 🚨 **타이어 공기압 실측값 불가** |
| **Safety** | 39 | **7** (실질 3개 함수) | 🚨 **경고등 리스너 불가** |
| CarInfo | 5 | 1 (`getCapabilityInfo`) | 경미 |
| Brake / Display / Door / DrivingMode / EvBattery / Fuel / Gear / HVAC / Light / Odometer / Seat / SideMirror / SteeringWheel / Toll / TurnSignal / Window / Wiper | 합계 460+ | **0** | ✅ 전부 공개 |

### 3.6 NaviHelper 권한

| 권한 | 용도 |
| --- | --- |
| `pleos.car.permission.NAVI_ROUTE_SEARCH` | `requestRoute`, `requestRouteWithWaypoints`, **`addWaypoint`**, `removeWaypoint`, `changeRouteOption`, `requestReRoute`, `cancelRoute` |
| `pleos.car.permission.NAVI_ROUTE` | `getTBTInfo`, `getRouteStateInfo`, `getDestinationInfo`, `getWaypointInfo`, `getCurrentLocationInfo`, `getBookmarkInfo`, `getRecentDestinationInfo`, `getChargerOperatorInfo`, `requestTrip` |
| `pleos.car.permission.NAVI_CUSTOM_MAP` | `requestNavigationScreen` |
| `pleos.car.permission.NAVI_CUSTOM_ETC` / `NAVI_CUSTOM_ROUTE` | 기타 커스텀 |

> ✅ **`addWaypoint`는 시스템 앱 전용이 아닙니다.** SDK 내부에 `checkSystemApplication()`(시스템 앱이 아니면 `SecurityException`)이 존재하지만, `NaviHelper.kt`의 공개 함수들에는 적용되지 않고 `InternalApi` / `PrivateApi` 경로에만 쓰입니다. → **"정비소 경유지 추가" 기능은 구현 가능.**

### 3.7 TTS / STT 권한

| SDK | AAR 매니페스트 선언 권한 |
| --- | --- |
| TextToSpeech 3.3.1 | `android.permission.INTERNET` |
| SpeechToText 3.3.1 | `android.permission.INTERNET` |

> 📌 Pleos 전용 권한을 요구하지 않습니다 → **TTS/STT는 권한 리스크가 가장 낮은 SDK**. STT용 `android.permission.RECORD_AUDIO`는 앱이 직접 선언해야 합니다.
> 📌 두 SDK 모두 `constant/Mode` 클래스를 가지며, TTS에는 `api/PrivilegedApi`가 존재 → **일부 TTS API는 특권 전용일 수 있음. 검증 필요.**

---

## 4. 🚨 제한사항 (핵심)

### 4.1 [치명적] 경고등 · 타이어 공기압 API = 비공개

`Vehicle-2.3.2-sources.jar`의 `Safety.kt` / `Tire.kt`에서 확인한 사실입니다.

| API | 요구 권한 (`@RequiresPermission`) | 서드파티 |
| --- | --- | --- |
| `Safety.checkTelltalesCapabilities` | `PRIVATE_CAR_SAFETY` | ❌ |
| `Safety.registerTelltales(TelltaleInfoListener)` | `PRIVATE_CAR_SAFETY` | ❌ |
| `Safety.unregisterTelltales` | `PRIVATE_CAR_SAFETY` | ❌ |
| **`Safety.getTelltalesInfo`** | **`CAR_SAFETY`** ⬅ 공개 | ⚠️ **§4.1.1 참조** |
| `Tire.checkTirePressureCapabilities` | `PRIVATE_CAR_SAFETY` | ❌ |
| `Tire.getTirePressure` | `PRIVATE_CAR_SAFETY` | ❌ |
| `Tire.registerTirePressure` / `unregisterTirePressure` | `PRIVATE_CAR_SAFETY` | ❌ |
| `CarInfo.getCapabilityInfo` | `PRIVATE_CAR_INFO` | ❌ |

**교차 근거:** 공식 샘플 앱 `PublicApiPolicy.kt`가 위 9개 API를 `blockedApis`에 넣고, README에 *"Private, hidden, and deprecated APIs are intentionally excluded"* 라고 명시합니다.

```kotlin
// Pleos-Playground/pleos-sample-vehicle-app
private val blockedApis = setOf(
    "CarInfo.getCapabilityInfo",
    "Safety.checkTelltalesCapabilities", "Safety.getTelltalesInfo",
    "Safety.registerTelltales",          "Safety.unregisterTelltales",
    "Tire.checkTirePressureCapabilities","Tire.getTirePressure",
    "Tire.registerTirePressure",         "Tire.unregisterTirePressure"
)
```

#### 4.1.1 ⭐ `getTelltalesInfo` — SDK 내 불일치 (검증 1순위)

`getTelltalesInfo`에는 **어노테이션과 KDoc이 서로 다릅니다.**

```kotlin
/**
 * Gets telltales info.
 * **Permission:** [PleosPermission.PRIVATE_CAR_SAFETY] (protection level: signature|privileged)   ← 문서
 */
@RequiresPermission(PleosPermission.CAR_SAFETY)                                                    ← 실제 어노테이션 (공개!)
fun getTelltalesInfo(onComplete: (telltales: Telltales) -> Unit, onFailed: (Exception) -> Unit)
```

**→ 공개 권한 `CAR_SAFETY`만으로 호출이 성공할 가능성이 있습니다.** 성공하면 `TelltaleType` **78종 전체 + `color` + `priority`** 를 polling 방식으로 얻을 수 있어 제품 가치가 크게 올라갑니다.

| 검증 방법 | 내용 |
| --- | --- |
| 1 | 에뮬레이터에서 `CAR_SAFETY`만 선언한 앱으로 `getTelltalesInfo()` 호출 → `SecurityException` 여부 확인 |
| 2 | 성공 시 **polling 기반**(리스너는 비공개이므로 사용 불가)으로 구현 |
| 3 | 단, **문서가 PRIVATE이라고 명시**하므로 App Market 심사에서 미승인 권한 호출로 판정될 리스크 존재 → **공모전 시연 용도로만 사용하고 폴백 경로를 반드시 유지** |

**✅ 공개 권한(`CAR_SAFETY`)으로 확정 사용 가능한 경고 신호** — MVP의 실제 기반

| API | 반환 | 리스너 | 의미 |
| --- | --- | --- | --- |
| `hasPowerDownWarningIndicator` | `Boolean?` | `registerPowerDownWarningIndicator` | *"true면 엔진에 문제가 있음"* (출력 제한) |
| `getEngineOilLevelState` | `OilLevelState?` | `registerEngineOilLevelState` | 엔진오일 레벨 |
| `getBrakeFluidLevelState` | `OilLevelState?` | `registerBrakeFluidLevelState` | 브레이크액 레벨 |
| `getTransmissionOilLevelState` | `OilLevelState?` | `registerTransmissionOilLevelState` | 변속기오일 레벨 |
| `hasLowWasherFluidWarning` | `Boolean?` | `registerLowWasherFluidWaringState` | 워셔액 부족 |
| `hasSeatAirbagEnabled(area)` | `Boolean?` | `registerSeatAirbagEnabled` | *"true면 해당 좌석 에어백에 문제가 있음"* |
| `hasSeatAirbagBeenDeployed(area)` | `Boolean?` | `registerSeatAirbagDeployed` | 에어백 전개(사고) |
| `hasSeatBeltNotBuckledWarning(area)` | `Boolean?` | `registerSeatBeltNotBuckledWarning` | 안전벨트 미체결 |
| **`Tire.getLowTirePressure(area)`** | `Float?` | — (리스너 없음) | ⭐ **타이어 공기압 부족 경고 — 공개!** |

`OilLevelState` enum: `CRITICALLY_LOW` / `LOW` / `NORMAL` / `HIGH` / `ERROR`

> 🔑 **결론:** 타이어 공기압은 **실측값(`getTirePressure`)만 비공개**이고 **부족 경고(`getLowTirePressure`)는 공개**입니다. 즉 "TPMS 경고등" 수준의 기능은 구현 가능합니다.
> ⚠️ 단, `getLowTirePressure`에는 **리스너가 없어 polling이 필요**합니다.

### 4.2 [중대] SDK 제공 상태

| SDK | 상태 | 근거 |
| --- | --- | --- |
| Vehicle | ✅ 제공 (2.3.2) | Maven 확인 |
| NaviHelper | ⚠️ 제공 (3.0.5) — **내비게이션이 탑재된 차량에서만 전체 호환** | 공식 호환성 문서 |
| TextToSpeech / SpeechToText | ✅ 제공 (3.3.1) — **전체 호환** | 공식 호환성 문서 |
| FusedLocation | ⚠️ 제공 (1.1.3) — **모델별 지원 상이**, 문서에는 "준비 중"으로도 기재 | Maven + 문서 (불일치) |
| **ADAS** | ❌ **Maven 저장소에 아티팩트 없음.** 문서상 "준비 중" | Maven 확인 |
| LLM | ⚠️ 제공 (2.1.3.2). **공모전 안내엔 미기재.** 문서에 *"No backward compatibility support"* | Maven + 문서 |

> ⚠️ **[idea.md](idea.md)의 Proximity Alert 기반 아이디어(E·F·H)는 FusedLocation 의존 → 모델별 가용성 리스크.** `isSupported` API로 확인 필요.
> ⚠️ **LLM SDK는 공모전 안내에 없으므로 심사에서 "제공 SDK"로 인정될지 불확실.** 핵심 기능을 LLM에 의존시키지 않을 것을 권장.

### 4.3 [중대] AAOS Alignment 정책

| 규칙 | 내용 |
| --- | --- |
| 원칙 | AAOS 표준에 동일/유사 API가 추가되면 **플랫폼 파편화를 줄이기 위해 표준 API 사용을 권장**하며, **중복 기능은 Pleos SDK가 제공하지 않음** |
| 하위 호환 | SDK 전용 기능이 Android Automotive에 채택되면 **최소 2개 메이저 버전**까지 하위 호환 제공 |
| 표준 데이터 | **AAOS Car API** 경로 — *"속도, 기어 상태, 외기 온도, 연료 레벨 등"* |
| 특화 기능 | **Pleos Connect SDK** 경로 — *"제조사 전용 시트 제어, 특화 라이팅, 고유 보안 인증 등"* |
| 가용성 확인 | `VehiclePropertyIds`로 정의·ID 확인 → **`CarPropertyManager`로 실제 지원 여부 및 접근 가능 상태 확인** |

> 🔑 **차량 속도**는 Pleos Vehicle SDK 도메인에 없습니다 ([idea.md](idea.md) §5 미확인 항목에 대한 답). **AAOS `CarPropertyManager` + `PERF_VEHICLE_SPEED`** 경로를 써야 하며, 이 경우 **AAOS 표준 권한(`android.car.permission.*`)의 보호 수준이 별도 제약**이 됩니다.
> ⚠️ **미검증:** AAOS 표준 권한 중 서드파티가 획득 가능한 범위. **에뮬레이터 실측 필요.**

### 4.4 [중대] 하드웨어 접근 정책

| 규칙 | 내용 |
| --- | --- |
| 금지 | 앱은 차량의 **물리적 제어나 센서에 직접 접근할 수 없음** |
| 필수 | 차량 상태는 **Pleos Connect SDK가 제공하는 표준 플랫폼 API를 통해서만** 조회·제어 |
| 특수 프로퍼티 | SDK가 제공하지 않는 프로퍼티가 필요하면 **직접 구현 전에 기술지원팀에 검토 요청** |
| 모델 분기 금지 | **모델명으로 분기하지 말고** `checkXXXCapabilities` / `isSupported` / `CarPropertyManager`로 실제 지원 여부 확인 |

### 4.5 [중대] 모바일 전용 API 제약

차량 플랫폼에서 **제공되지 않거나 기능이 제한될 수 있는** 항목:

- **Telephony(전화), SMS**
- **일부 센서** (자이로, 근접 등)
- **캘린더, 주소록**

**대응:** 필수가 아닌 기능은 제거하거나, **API 지원 여부를 확인한 뒤 동작하는 Optional 구조**로 설계.

### 4.6 [중대] 네트워크 — 주행 중 연결 불안정

권장 대응: **로컬 캐싱**, **재시도(Retry) + 타임아웃 설정**, **오프라인 모드 안내 UI**

> 🔑 [mvp-plan.md](mvp-plan.md)에서 경고등 설명 DB를 **앱에 번들(오프라인)** 하기로 한 근거입니다. 안전 정보가 네트워크에 의존하면 안 됩니다.

### 4.7 [중대] Driver Distraction (주의분산) 제약

| 항목 | 내용 |
| --- | --- |
| 선언 방식 | `AndroidManifest.xml` 의 `<activity>`에 **Distraction Optimized** 메타데이터 추가 |
| 앱 단위 인정 | **모든 액티비티가 가이드라인을 준수**해야 앱 전체가 Distraction Optimized로 지정됨 |
| 미선언 시 동작 | UX 제약이 활성화되면 **미선언 앱은 OS 소유 액티비티에 의해 가려짐(차단됨)** |
| 플랫폼 검증 | 플랫폼은 **준수 여부를 검증하지 않고 매니페스트 선언만 확인** → 실제 준수는 **심사에서 판정** |
| Pleos 심사 기준 | **운전자 상호작용을 최소화**해야 하며, 제출 전 **Pleos Connect UI Design Guide** 검토 필수 |
| 🚨 거절 리스크 | **가이드를 따랐더라도 심사 중 주의분산 위험이 확인되면 거절될 수 있음** |
| 참고 | [AOSP Driver Distraction Guidelines](https://source.android.com/docs/automotive/driver_distraction/guidelines) · [Car app quality (DD-1/2/3)](https://developer.android.com/docs/quality-guidelines/car-app-quality) |

> 🔑 **설계 결론:** 주행 중은 **TTS 음성 중심 + 최소 터치**, 화면 기반 상세 정보는 **정차 시**로 분리. ([mvp-plan.md](mvp-plan.md) 설계 원칙)
> ⚠️ **미검증:** Pleos Connect UI Design Guide 원문 (문서 사이트에서 해당 페이지를 찾지 못함) → **온라인 설명회 자료 또는 문의 필요.**

### 4.8 [중] API 동작 특성 (공식 샘플이 실측한 주의점)

| 특성 | 내용 |
| --- | --- |
| capability 선행 | `checkXXXCapabilities(onComplete, onFailed)` → `config.accessType` 및 영역별 `minValue`/`maxValue` 획득. **보편적 범위를 가정하지 말고 보고된 범위를 사용** |
| 미지원 영역 | 미지원 영역은 **범위를 보고하지 않을 수 있음** |
| null 반환 | `getXXX`는 **`null`을 반환할 수 있음** |
| SET ≠ GET | **SET 성공이 GET을 트리거하지 않음.** 상태 확인은 **다시 읽어야 함** |
| 실측 사례 | Door Position `SET=1` 성공 후 재읽기가 `0` 반환 → README는 "요청 완료와 별개의 읽기 동작"으로 해석 (영속적 상태 변경 미확정) |
| 실측 사례 | 테스트 AVD에서 **Door Lock의 `ROW_1_LEFT` 읽기 실패** → 가용성은 영역별로 다름 |
| 제어 API | 값을 범위 검증하고, 제어 접근을 확인하고, **명시적 사용자 동작을 선행**시켜야 함 |

### 4.9 [중] 에뮬레이터 제약

| 항목 | 제약 |
| --- | --- |
| Boot | **cold boot 필수** (quick boot면 VHAL 불가) |
| RAM | 4 GB |
| 기본 언어 | **한국어** |
| 마이크 | **STT 테스트 시 에뮬레이터 설정에서 마이크 입력을 허용**해야 함 |
| Bluetooth | **제한적 지원.** 테스트에 **USB 동글 필요** — 검증된 동글: Linux(x86) **BT-400**, macOS(arm64) **BT-500**. 옵션: `-packet-streamer-endpoint` 또는 `-usb-passthrough` |
| 실차 | Pleos 디바이스/에뮬레이터 + 호환 Vehicle Service 필요. **일반 Android 폰 이미지로는 차량 프로퍼티를 전혀 쓸 수 없음** |
| 알려진 이슈 | Android Emulator / Android Studio / AGP 자체의 알려진 이슈 존재 |

### 4.10 [중] 공식 샘플 앱의 알려진 결함

| 결함 | 내용 |
| --- | --- |
| 테스트 실패 | `PublicApiPolicyTest`가 SDK 2.2.10 기준 **441개**로 고정되어 있으나 현재 2.3.2는 **442개** 노출 → **단정문이 완화되지 않아 전체 유닛 스위트가 실패** |
| 범위 | **Door Position만이 유일한 typed 참조 플로우.** 카탈로그는 메서드명·파라미터 탐색용이며 **프로덕션 용도가 아님** |
| 제외 | **Gear는 샘플 카탈로그에 없음** (단, SDK에는 `Gear.kt`가 존재: `getCurrentGearSelectionState`, `registerGearSelection`, `unregisterGearSelection`) |

### 4.11 [중] 앱 제출 · 심사 제약

| 항목 | 요구사항 |
| --- | --- |
| APK 용량 | **최대 600 MB** |
| 심사 자료 | **모든 기능·콘텐츠에 접근 가능한 계정** 제공, 외부 서버 컴포넌트 **사전 활성화**, 리뷰 노트에 **기능 사용 가이드** 첨부 (최대 60 MB) |
| 권한 신고 | 앱 권한 + **SDK 권한 승인 필요 시 Playground 프로젝트 ID** 제출 |
| 개인정보 | 처리하는 **모든 개인정보 신고**, GPS·지도·위치 사용 시 **위치 신고 + 위치서비스 이용약관 URL** |
| 정책 URL | **개인정보 처리방침 URL 필수** — **서비스 국가의 모든 언어로 접근 가능**해야 함 |
| 연령등급 | **IARC 연령등급 취득 + 증빙 제출** |
| 거절 사유 | 주행 안전 위험, 오동작/성능 저하, 부적절 콘텐츠, 개인정보·보안·사이버보안 위반, **기만적 메타데이터(→ 즉시 삭제 및 영구 제재)** |
| 금지 | **개발용 SDK 오용**, **승인 없는 사용자 데이터 사용** |
| 사후 관리 | 마켓이 **주기적으로 샘플링·재평가**하며 기준 미달 앱을 제거할 수 있음 |
| 수상팀 지원 | *"Pleos App Market 탑재 준비 지원"* — **별도 검증·심사 및 지원 차종·지역 조건을 거치며 최종 입점/배포를 보장하지 않음** ([info.md](info.md) §6) |

### 4.12 [중] 문서 품질 리스크

| 항목 | 내용 |
| --- | --- |
| 공식 경고 | API 레퍼런스에 *"문서가 사용 중인 SDK 및 에뮬레이터 버전과 일치하지 않을 수 있음"* 명시 |
| 버전 혼재 | 문서 페이지별로 상이한 버전 표기 (`connect.v2.0.5` 이미지 URL, Vehicle SDK v2.0.3, Emulator v2.1.0 등) |
| 한/영 차이 | 영문 번역본과 한국어판의 내용이 다를 수 있음 |
| 도메인 이전 | `pleos.ai/playground/*` → `developers.pleos.ai/*` → `document.pleos.ai/*` 로 **301/308 리다이렉트 중**. 오래된 링크 다수 |
| SDK 내 불일치 | §4.1.1의 `getTelltalesInfo` 어노테이션 vs KDoc 불일치 |

**→ 대응: 문서보다 `-sources.jar`의 `@RequiresPermission` 어노테이션을 1차 근거로 삼고, 최종 판단은 에뮬레이터 실측으로 확정.**

---

## 5. 공모전 관점 체크리스트

| # | 항목 | 상태 |
| --- | --- | --- |
| 1 | Pleos Playground 가입 + **My Project 생성 → CRN 발급** | ⬜ **최우선** |
| 2 | Pleos Connect v2.1.0 에뮬레이터 설치 (cold boot, RAM 4GB) | ⬜ |
| 3 | CRN 주입 + 에뮬레이터 재시작 | ⬜ |
| 4 | 공식 샘플 앱(`pleos-sample-vehicle-app`) 빌드·설치로 환경 검증 | ⬜ |
| 5 | **§4.1.1 `getTelltalesInfo` 공개 권한 호출 검증** | ⬜ **핵심 분기** |
| 6 | 공개 Safety 9종 신호 실측 (VHAL 주입으로 값 변화 확인) | ⬜ |
| 7 | `Tire.getLowTirePressure` 동작 확인 (polling) | ⬜ |
| 8 | TTS 3.3.1 발화 확인 | ⬜ |
| 9 | STT 3.3.1 — 에뮬레이터 마이크 허용 후 인식 확인 | ⬜ |
| 10 | NaviHelper 3.0.5 `addWaypoint` 동작 확인 (내비 탑재 전제) | ⬜ |
| 11 | Distraction Optimized 메타데이터 선언 + 주행 중 동작 확인 | ⬜ |
| 12 | release APK 서명 + `팀명_앱이름_release.apk` 명명 ([info.md](info.md) §4) | ⬜ |
| 13 | GitHub 저장소 + `apk/` 폴더 + README 필수 4항목 | ⬜ |

---

## 6. 확인 필요 (미검증 항목)

- [ ] **§4.1.1** `getTelltalesInfo`를 `CAR_SAFETY`만으로 호출 시 성공 여부 — **최우선**
- [ ] `TelltaleData.color` 인코딩 형식 및 `priority` 값 범위
- [ ] Pleos Connect **UI Design Guide** 원문 위치 (주행 중 터치 횟수·텍스트 길이 제한 등 정량 기준)
- [ ] Distraction Optimized 메타데이터의 **Pleos 고유 키 이름** (AAOS 표준 키와 동일한지)
- [ ] AAOS `CarPropertyManager` 사용 시 서드파티가 획득 가능한 `android.car.permission.*` 범위
- [ ] 공모전 참가자에게 **LLM SDK 사용이 허용되는지** (안내문엔 5개 SDK만 명시)
- [ ] FusedLocation 의 **문서상 "준비 중" vs Maven 1.1.3 배포** 불일치의 실제 의미
- [ ] 에뮬레이터 **Virtual Console**의 VHAL 값 주입 UI로 Safety/Tire 프로퍼티를 조작 가능한지
- [ ] TTS `api/PrivilegedApi` 에 포함된 API 목록 (특권 전용 범위)
- [ ] CRN 주입 명령의 현행 버전 정확한 형식

---

## 7. 근거 링크

**공식 문서**
- [에뮬레이터 셋업](https://document.pleos.ai/en/docs/connect/guide/getting-started/application-development/setup-connect-sdk-emulator)
- [개발환경 설정](https://document.pleos.ai/en/docs/connect/guide/getting-started/application-development/setup-application-development-environment)
- [Vehicle SDK intro](https://document.pleos.ai/en/api-reference/connect-sdk-pleos/Vehicle/intro)
- [Vehicle 권한 정책](https://document.pleos.ai/en/api-reference/connect-sdk-pleos/Vehicle/vehicle-permission)
- [SDK API 정책 (AAOS Alignment)](https://document.pleos.ai/en/docs/pleos-only/vehicle-app-planning-guide/pleos-connect-sdk-api)
- [플랫폼 API 아키텍처 (하드웨어 접근 정책)](https://document.pleos.ai/docs/pleos-only/vehicle-app-planning-guide/platform-api-architecture)
- [API 호환성 및 가용성](https://document.pleos.ai/docs/pleos-only/vehicle-app-planning-guide/api-compatibility-availability)
- [App Submission](https://document.pleos.ai/en/docs/pleos-only/distribute/app-submission-guide)
- [App Review Guidelines](https://document.pleos.ai/en/docs/pleos-only/distribute/app-review-guide)

**공식 샘플**
- [pleos-sample-vehicle-app](https://github.com/Pleos-Playground/pleos-sample-vehicle-app)
- [pleos-sample-navi-app](https://github.com/Pleos-Playground/pleos-sample-navi-app)
- [pleos-sample-fusedlocation-app](https://github.com/Pleos-Playground/pleos-sample-fusedlocation-app)

**AAOS**
- [Driver Distraction Guidelines](https://source.android.com/docs/automotive/driver_distraction/guidelines)
- [Car app quality](https://developer.android.com/docs/quality-guidelines/car-app-quality)
- [Parked app support](https://developer.android.com/training/cars/parked/automotive-os)
