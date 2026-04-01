# 실습: SystemUI 커스터마이징 실습 (Android 16)

Android 16(API Level 36) AOSP 환경에서 SystemUI를 분석하고 커스터마이징하는 5가지 실습을 단계별로 진행합니다.

> **실습 환경**
> - Android AOSP Car 에뮬레이터 (`emulator_car64_x86_64`, userdebug 빌드)
> - 에뮬레이터 API Level: 35 (SDK 환경에 따라 34~36)
> - ADB가 연결된 상태
> - 호스트 PC: Windows 또는 Linux

---

## 실습 1: `adb shell dumpsys statusbar`로 현재 상태 확인

### 학습 목표
- `dumpsys` 명령어를 사용하여 상태바(StatusBar)의 현재 상태를 실시간으로 확인하는 방법을 익힙니다.
- SystemUI의 내부 구조와 상태 관리 방식을 이해합니다.

### 1.1 기본 상태바 정보 확인

```shell
# 상태바 전체 정보 덤프
adb shell dumpsys statusbar
```

출력 예시 (주요 섹션):

```
StatusBarService state:
  mDisabled1=0x0
  mDisabled2=0x0
  ...
  Notifications:
    ...
  Quick Settings Tiles:
    ...
```

### 1.2 주요 섹션별 분석

#### (1) 알림(Notification) 상태 확인

```shell
# 현재 활성 알림 목록 확인
adb shell dumpsys statusbar | grep -A 5 "Notifications"
```

#### (2) Quick Settings 타일 목록 확인

```shell
# QS 타일 목록과 상태 확인
adb shell dumpsys statusbar | grep -i "tile"
```

#### (3) 상태바 아이콘 목록 확인

```shell
# 현재 표시 중인 상태바 아이콘 확인
adb shell dumpsys statusbar | grep -i "icon"
```

### 1.3 추가 유용한 dumpsys 명령어

```shell
# SystemUI 전체 정보 (메모리 포함)
adb shell dumpsys activity provider com.android.systemui

# 알림 서비스 상세 정보
adb shell dumpsys notification

# 활성 알림 요약
adb shell dumpsys notification --noredact

# SystemUI 프로세스의 메모리 사용량
adb shell dumpsys meminfo com.android.systemui

# 프레임 렌더링 통계 (jank 분석)
adb shell dumpsys gfxinfo com.android.systemui

# 현재 포커스된 Activity 확인
adb shell dumpsys activity activities | grep mResumedActivity
```

### 1.4 실습 과제

1. 에뮬레이터에서 알림 하나를 발생시킨 뒤 `dumpsys notification`의 출력 변화를 관찰하세요.
2. QS 패널을 펼친 상태와 접은 상태에서 `dumpsys statusbar` 출력을 비교해보세요.
3. Wi-Fi를 켜고 끌 때 상태바 아이콘 변화를 `dumpsys`로 확인하세요.

```shell
# 테스트 알림 발생시키기
adb shell am start -a android.intent.action.VIEW -d "https://example.com"

# Wi-Fi 토글
adb shell svc wifi disable
adb shell svc wifi enable
```

---

## 실습 2: RRO로 SystemUI 색상 변경

### 학습 목표
- 런타임 리소스 오버레이(RRO)를 사용하여 SystemUI의 색상 테마를 동적으로 변경합니다.
- Car SystemUI 환경에서 RRO를 시스템 앱으로 설치하는 방법을 익힙니다.
- `aapt`로 실제 리소스 이름을 추출하여 올바른 오버레이를 구성하는 방법을 배웁니다.

### 2.1 Car SystemUI의 오버레이 구조 파악

먼저 현재 에뮬레이터에서 SystemUI에 적용된 오버레이 목록을 확인합니다.

```shell
adb shell cmd overlay list | grep systemui
```

출력 예시:

```
com.android.systemui
[x] com.android.systemui.car.rro
[x] com.android.systemui.auto_generated_rro_product__
[x] com.android.car.aaesystemui.rro
--- com.android.car.ui.overlay.sdk.carsystemui
[x] com.android.systemui.rro.landscape
[ ] com.android.systemui.googlecarui.theme.orange.rro
[ ] com.android.systemui.googlecarui.theme.pink.rro
...
```

상태 표시의 의미:

| 표시 | 의미 |
|:-----|:-----|
| `[x]` | 활성화된 오버레이 |
| `[ ]` | 비활성화 상태 (활성화 가능) |
| `---` | **매칭 실패** (활성화 불가) |

### 2.2 기존 Car RRO의 구조 확인

```shell
# 활성화된 Car SystemUI RRO의 상세 정보 확인
adb shell dumpsys overlay | grep -A 5 "com.android.systemui.car.rro"
```

출력에서 핵심 정보:

```
mTargetPackageName.....: com.android.systemui
mTargetOverlayableName.: null        ← overlayable 미선언
mBaseCodePath..........: /system/app/CarSystemUIRRO/CarSystemUIRRO.apk
```

> **⚠️ 핵심 발견: Car SystemUI는 `<overlayable>`을 선언하지 않습니다.**
>
> 이것이 의미하는 바:
> - `adb install`로 설치한 사용자 오버레이는 `---` 상태가 되어 활성화할 수 없습니다.
> - Android 11부터 사용자 오버레이는 타겟 앱이 `<overlayable>`을 선언한 리소스만 오버레이할 수 있습니다.
> - 시스템 파티션(`/system/app/`)에 설치된 오버레이만 이 제한을 우회합니다.
>
> | 설치 위치 | policy | overlayable 필요 여부 | 결과 |
> |-----------|--------|---------------------|------|
> | `/system/app/` | system | **불필요** | `[x]` 또는 `[ ]` (활성화 가능) |
> | `adb install` (사용자) | public | **필수** (타겟이 선언해야 함) | `---` (매칭 실패) |

### 2.3 Car SystemUI의 실제 리소스 이름 추출

Car SystemUI는 Phone SystemUI와 리소스 이름이 다릅니다. 오버레이에 잘못된 리소스 이름을 넣으면 색상이 변경되지 않습니다. 따라서 **APK에서 실제 리소스 이름을 추출**하는 과정이 필요합니다.

#### (1) Car SystemUI APK 가져오기

```shell
# APK 경로 확인
adb shell pm path com.android.systemui
# 출력: package:/system_ext/priv-app/CarSystemUI/CarSystemUI.apk

# APK를 PC로 복사
adb pull /system_ext/priv-app/CarSystemUI/CarSystemUI.apk .
```

#### (2) aapt로 색상 리소스 목록 추출

```shell
# aapt 경로 (Android SDK build-tools 내부)
# Windows 예시:
C:\Users\사용자\AppData\Local\Android\Sdk\build-tools\35.0.0\aapt.exe dump resources CarSystemUI.apk | findstr /i "color"

# Linux/Mac 예시:
aapt dump resources CarSystemUI.apk | grep "color/"
```

> **aapt 경로를 모를 경우:**
> ```shell
> # Windows
> dir /s /b C:\Users\%USERNAME%\AppData\Local\Android\Sdk\build-tools\aapt.exe
>
> # 또는 PATH에 추가
> set PATH=%PATH%;C:\Users\%USERNAME%\AppData\Local\Android\Sdk\build-tools\35.0.0
> ```

#### (3) 추출 결과에서 주요 Car SystemUI 리소스 확인

실제 추출한 결과에서 확인된 Car SystemUI 주요 색상 리소스:

| 리소스 이름 | 적용 영역 | 실제 적용 여부 |
|:-----------|:---------|:-------------|
| `system_bar_clock_text_color` | 상단바 시계 텍스트 | ✅ 적용됨 |
| `system_bar_icon_color` | 상단바 아이콘 (블루투스, Wi-Fi 등) | ✅ 적용됨 |
| `system_bar_text_color` | 상단바 "Driver" 등 텍스트 | ✅ 적용됨 |
| `car_status_icon_color` | 상태바 아이콘 보조 색상 | ✅ 적용됨 |
| `car_nav_icon_fill_color` | 하단 내비바 아이콘 채움색 | ✅ 적용됨 |
| `car_nav_icon_fill_color_selected` | 하단 내비바 선택된 아이콘 | ✅ 적용됨 |
| `car_nav_icon_background_color` | 하단 내비바 아이콘 배경 | ✅ 적용됨 |
| `system_bar_background_opaque` | 상단바 배경 | ⚠️ 투명 설정으로 미적용 |
| `control_bar_background_color` | 하단바 배경 | ⚠️ 투명 설정으로 미적용 |

> **⚠️ Car SystemUI 배경색이 변경되지 않는 이유**
>
> Car SystemUI의 상단/하단 바는 **투명(transparent) 배경**으로 설정되어 있고, 뒤에 Car Launcher(홈 화면)의 배경이 비치는 구조입니다. `system_bar_background_opaque`를 변경해도 실제로는 `system_bar_background_transparent` 리소스가 사용될 수 있습니다. 바 배경을 변경하려면 투명 리소스까지 함께 오버레이해야 합니다.

### 2.4 RRO 오버레이 프로젝트 생성

Android 스튜디오에서 새 프로젝트를 생성합니다.

- 프로젝트 이름: `SystemUIOverlay`
- 패키지 이름: `com.example.systemuioverlay`
- Minimum SDK: **API 34** (에뮬레이터 호환을 위해 34로 설정)

> **⚠️ minSdk 주의**: API 36으로 설정하면 API 35 에뮬레이터에서 `INSTALL_FAILED_OLDER_SDK` 오류가 발생합니다. RRO 기능은 API 34에서도 동일하게 동작하므로 34로 설정하세요.

#### (1) AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.systemuioverlay">

    <application
        android:hasCode="false"
        android:label="SystemUI Color Overlay">
    </application>

    <!-- ⚠️ Car SystemUI는 overlayable을 선언하지 않으므로
         targetName을 제거해야 합니다.
         targetName을 지정하면 매칭 실패(---) 발생 -->
    <overlay
        android:targetPackage="com.android.systemui"
        android:priority="1"
        android:isStatic="false" />
</manifest>
```

> **일반 SystemUI vs Car SystemUI**
>
> | 환경 | `targetName` 설정 |
> |:-----|:----------------|
> | 일반 Phone/Tablet SystemUI | `android:targetName="SystemUIOverlayableTheme"` (필요) |
> | **Car SystemUI (이 실습)** | **`targetName` 제거** (overlayable 미선언) |

#### (2) 변경할 색상 리소스 정의

**res/values/colors.xml** — Car SystemUI 전용 리소스 이름 사용

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>

    <!-- ===== 상단 상태바 영역 (✅ 적용 확인됨) ===== -->

    <!-- 시계 텍스트: 흰색 → 핑크 -->
    <color name="system_bar_clock_text_color">#FFFF4081</color>
    <color name="status_bar_clock_color">#FFFF4081</color>

    <!-- 상태바 아이콘 (블루투스, Wi-Fi 등): 흰색 → 노란색 -->
    <color name="system_bar_icon_color">#FFFFEB3B</color>
    <color name="car_status_icon_color">#FFFFEB3B</color>

    <!-- 상태바 텍스트 ("Driver" 등): 흰색 → 시안 -->
    <color name="system_bar_text_color">#FF00E5FF</color>


    <!-- ===== 하단 내비게이션 바 영역 (✅ 적용 확인됨) ===== -->

    <!-- 내비 아이콘 채움색: 기본 → 주황 -->
    <color name="car_nav_icon_fill_color">#FFFFAB40</color>
    <color name="car_nav_icon_fill_color_selected">#FFFFFF00</color>

    <!-- 내비 아이콘 배경: 기본 → 딥 네이비 -->
    <color name="car_nav_icon_background_color">#FF1A237E</color>
    <color name="car_nav_icon_background_color_selected">#FFFF6D00</color>


    <!-- ===== 바 배경 (투명→불투명 전환 시도) ===== -->

    <!-- 상단바 배경: 투명 → 딥 네이비 (불투명) -->
    <color name="system_bar_background_opaque">#FF1A237E</color>
    <color name="system_bar_background_transparent">#FF1A237E</color>
    <color name="status_icon_panel_bg_color">#FF1A237E</color>

    <!-- 하단바 배경: 투명 → 블루 -->
    <color name="control_bar_background_color">#FF0D47A1</color>
    <color name="control_bar_button_background_color">#FF1565C0</color>
    <color name="minimized_control_bar_background_color">#FF0D47A1</color>


    <!-- ===== 테마 색상 (Car Design System) ===== -->

    <color name="car_primary">#FF651FFF</color>
    <color name="car_primary_container">#FF311B92</color>
    <color name="car_on_primary">#FFFFFFFF</color>
    <color name="car_secondary">#FF00BFA5</color>
    <color name="car_secondary_container">#FF004D40</color>
    <color name="car_tertiary">#FFFF4081</color>

    <!-- 전체 배경/표면 색상 -->
    <color name="car_background">#FF0A0E2A</color>
    <color name="car_surface">#FF121640</color>
    <color name="car_surface_1">#FF1A1F4E</color>
    <color name="car_surface_2">#FF1A1F4E</color>
    <color name="car_surface_3">#FF1A1F4E</color>
    <color name="car_on_surface">#FFE0E0FF</color>
    <color name="car_on_background">#FFE0E0FF</color>
    <color name="car_scrim">#FF0A0E2A</color>
    <color name="car_neutral_6">#FF0A0E2A</color>
    <color name="car_neutral_10">#FF121640</color>


    <!-- ===== 알림 영역 ===== -->

    <color name="notification_shade_background_color">#FF0A0E2A</color>
    <color name="notification_background_color">#FF1A1F4E</color>
    <color name="overlay_panel_bg_color">#FF0A0E2A</color>


    <!-- ===== 텍스트 색상 ===== -->

    <color name="car_text_primary">#FFE0E0FF</color>
    <color name="car_text_secondary">#FFB0B0D0</color>
    <color name="car_ui_text_color_primary">#FFE0E0FF</color>
    <color name="car_ui_text_color_secondary">#FFB0B0D0</color>
    <color name="primary_text_color">#FFE0E0FF</color>
    <color name="secondary_text_color">#FFB0B0D0</color>

</resources>
```

#### (3) APK 빌드

```shell
# Android 스튜디오에서 빌드
# Build > Generate App Bundles or APKs > Generate APKs

# 빌드된 APK를 작업 디렉토리로 복사
# 보통 경로: AndroidStudioProjects/SystemUIOverlay/app/build/outputs/apk/debug/app-debug.apk
# → systemuioverlay.apk 로 이름 변경
```

### 2.5 시스템 파티션에 오버레이 설치 (Car SystemUI 전용)

Car SystemUI는 `<overlayable>`을 선언하지 않으므로, 시스템 앱으로 설치해야 합니다.
이 과정은 **2단계 remount**가 필요합니다.

#### Step 1: Verity 비활성화 및 OverlayFS 설정

```shell
# root 권한 획득
adb root

# remount 요청 (첫 번째: verity 비활성화 + overlayfs 설정)
adb remount
```

출력 예시:

```
Successfully disabled verity
Using overlayfs for /system
Using overlayfs for /vendor
Using overlayfs for /product
Using overlayfs for /system_ext
Verity disabled; overlayfs enabled.
Now reboot your device for settings to take effect
```

#### Step 2: 재부팅

```shell
adb reboot

# 에뮬레이터가 완전히 부팅될 때까지 대기
adb wait-for-device
```

#### Step 3: 다시 root + remount (실제 마운트)

```shell
# 재부팅 후 반드시 다시 실행해야 /system이 쓰기 가능해짐
adb root
adb remount
```

이번에는 `remount succeeded` 메시지가 나와야 합니다.

> **⚠️ 왜 remount가 2번 필요한가?**
>
> | 단계 | 역할 |
> |:-----|:-----|
> | 1차 `adb remount` | dm-verity 비활성화 + overlayfs 설정 (재부팅 필요) |
> | 재부팅 | 변경사항 적용 |
> | 2차 `adb remount` | **실제로 /system을 쓰기 가능으로 마운트** |
>
> 2차 remount를 하지 않으면 `Read-only file system` 오류가 발생합니다.

#### Step 4: 오버레이 APK 설치

```shell
# 시스템 앱 폴더 생성
adb shell mkdir -p /system/app/SystemUIOverlay

# APK를 시스템 앱으로 설치
adb push systemuioverlay.apk /system/app/SystemUIOverlay/SystemUIOverlay.apk

# 재부팅 (시스템 앱은 재부팅해야 인식됨)
adb reboot
```

### 2.6 오버레이 활성화 및 확인

```shell
# 오버레이 상태 확인 ([ ] 로 표시되어야 성공)
adb shell cmd overlay list | grep systemuioverlay

# 활성화
adb shell cmd overlay enable com.example.systemuioverlay

# SystemUI 재시작하여 반영
adb shell killall com.android.systemui
```

> **상태 확인 체크리스트**
>
> | 표시 | 의미 | 다음 단계 |
> |:-----|:-----|:---------|
> | `[ ]` | 매칭 성공, 비활성 상태 | `cmd overlay enable`로 활성화 |
> | `[x]` | 활성화됨 | 성공! 색상 변경 확인 |
> | `---` | 매칭 실패 | AndroidManifest.xml에서 `targetName` 제거 후 재빌드 |

### 2.7 오버레이 적용 결과 확인

```shell
# 오버레이 상세 정보 확인
adb shell cmd overlay dump com.example.systemuioverlay

# 스크린샷 캡처하여 비교
adb shell screencap -p /sdcard/after_overlay.png
adb pull /sdcard/after_overlay.png .
```

실제 적용 결과 (Car 에뮬레이터에서 확인됨):

| 영역 | 변경 전 | 변경 후 | 적용 여부 |
|:-----|:--------|:--------|:---------|
| 상단바 시계 | 흰색 | **핑크** (#FF4081) | ✅ |
| 상태바 아이콘 (BT, WiFi) | 흰색 | **노란색** (#FFEB3B) | ✅ |
| "Driver" 텍스트 | 흰색 | **시안** (#00E5FF) | ✅ |
| 알림 벨 아이콘 | 흰색 | **노란색** (#FFEB3B) | ✅ |
| 하단바 숫자 (63) | 흰색 | **노란색** (#FFEB3B) | ✅ |
| 하단 내비 아이콘 | 기본색 | **주황색** (#FFAB40) | ✅ |
| 상단/하단 바 배경 | 검정 | 변경 안됨 | ⚠️ 투명 구조 |

### 2.8 APK 업데이트 절차 (색상 수정 후 재배포)

색상을 수정한 뒤 APK를 업데이트할 때는 재부팅 없이도 가능합니다:

```shell
# remount가 이미 설정된 상태에서
adb root
adb remount

# 수정된 APK 덮어쓰기
adb push systemuioverlay.apk /system/app/SystemUIOverlay/SystemUIOverlay.apk

# 재부팅
adb reboot

# 재부팅 후 오버레이 활성화 (최초 설치 이후에는 자동 활성화될 수 있음)
adb shell cmd overlay enable com.example.systemuioverlay
adb shell killall com.android.systemui
```

### 2.9 오버레이 비활성화 및 제거

```shell
# 비활성화 (원래 색상으로 복원)
adb shell cmd overlay disable com.example.systemuioverlay

# 시스템 앱 제거 (remount 필요)
adb root
adb remount
adb shell rm -rf /system/app/SystemUIOverlay
adb reboot
```

### 2.10 AOSP 빌드 시스템에서의 RRO (심화)

AOSP 소스 트리 내에서 RRO를 빌드하면 시스템 파티션에 자동으로 포함됩니다.

**Android.bp**
```json
runtime_resource_overlay {
    name: "CustomSystemUIOverlay",
    resource_dirs: ["res"],
    certificate: "platform",
    product_specific: true,
    sdk_version: "current",
}
```

```shell
# AOSP 빌드
cd $ANDROID_BUILD_TOP
m CustomSystemUIOverlay
```

### 2.11 트러블슈팅 정리

| 문제 | 원인 | 해결 |
|:-----|:-----|:-----|
| `INSTALL_FAILED_OLDER_SDK` | `minSdk`가 에뮬레이터 API보다 높음 | `minSdk`를 34로 낮추기 |
| `cmd overlay enable` 시 `SecurityException` | `adb install`로 설치 + 타겟이 overlayable 미선언 | 시스템 앱으로 설치 (`/system/app/`) |
| 오버레이가 `---`로 표시됨 | `targetName` 불일치 또는 overlayable 미선언 | `targetName` 제거 + 시스템 앱으로 설치 |
| `Read-only file system` | `adb remount`를 재부팅 후 다시 실행하지 않음 | `adb root` → `adb remount` 재실행 |
| `mkdir failed` | 재부팅 후 2차 remount 누락 | 위와 동일 |
| 오버레이 활성화했는데 색상 변화 없음 | 리소스 이름이 Car SystemUI와 불일치 | `aapt dump resources`로 실제 리소스 이름 확인 |
| 바 배경색이 변경 안됨 | Car SystemUI 바가 투명 배경 사용 | `system_bar_background_transparent` 등 투명 리소스도 함께 오버레이 |

---

## 실습 3: AOSP SystemUI 빌드 — 커스텀 QS 타일 추가

### 학습 목표
- AOSP SystemUI 소스 코드를 직접 수정하여 새로운 Quick Settings(QS) 타일을 추가합니다.
- QSTileImpl의 구조와 타일 등록 과정을 이해합니다.

### 3.1 SystemUI 소스 구조 파악

```
frameworks/base/packages/SystemUI/
├── src/com/android/systemui/
│   ├── qs/                           ← QS 관련 코드
│   │   ├── tiles/                    ← 개별 타일 구현
│   │   │   ├── AirplaneModeTile.java
│   │   │   ├── BluetoothTile.java
│   │   │   ├── FlashlightTile.java
│   │   │   ├── WifiTile.java
│   │   │   └── ...
│   │   ├── QSTileHost.java           ← 타일 등록 및 관리
│   │   └── ...
│   └── ...
├── res/
│   ├── drawable/                     ← 아이콘 리소스
│   ├── values/
│   │   ├── strings.xml               ← 문자열 리소스
│   │   └── config.xml                ← 기본 타일 목록 설정
│   └── ...
└── Android.bp
```

### 3.2 커스텀 QS 타일 구현: CpuInfoTile

CPU 정보를 보여주는 간단한 QS 타일을 만들어보겠습니다.

#### (1) 타일 클래스 생성

**파일 경로**: `frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tiles/CpuInfoTile.java`

```java
package com.android.systemui.qs.tiles;

import android.content.Intent;
import android.os.Handler;
import android.os.Looper;
import android.service.quicksettings.Tile;
import android.widget.Toast;

import com.android.internal.logging.MetricsLogger;
import com.android.systemui.R;
import com.android.systemui.dagger.qualifiers.Background;
import com.android.systemui.dagger.qualifiers.Main;
import com.android.systemui.plugins.ActivityStarter;
import com.android.systemui.plugins.FalsingManager;
import com.android.systemui.plugins.qs.QSTile.BooleanState;
import com.android.systemui.plugins.statusbar.StatusBarStateController;
import com.android.systemui.qs.QSHost;
import com.android.systemui.qs.QsEventLogger;
import com.android.systemui.qs.logging.QSLogger;
import com.android.systemui.qs.tileimpl.QSTileImpl;

import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

import javax.inject.Inject;

public class CpuInfoTile extends QSTileImpl<BooleanState> {

    private boolean mShowingInfo = false;

    @Inject
    public CpuInfoTile(
            QSHost host,
            QsEventLogger uiEventLogger,
            @Background Looper backgroundLooper,
            @Main Handler mainHandler,
            FalsingManager falsingManager,
            MetricsLogger metricsLogger,
            StatusBarStateController statusBarStateController,
            ActivityStarter activityStarter,
            QSLogger qsLogger
    ) {
        super(host, uiEventLogger, backgroundLooper, mainHandler,
                falsingManager, metricsLogger, statusBarStateController,
                activityStarter, qsLogger);
    }

    @Override
    public BooleanState newTileState() {
        BooleanState state = new BooleanState();
        state.handlesLongClick = false;
        return state;
    }

    @Override
    protected void handleClick(/* View view */) {
        mShowingInfo = !mShowingInfo;
        refreshState();
    }

    @Override
    protected void handleUpdateState(BooleanState state, Object arg) {
        state.label = "CPU Info";
        state.icon = ResourceIcon.get(R.drawable.ic_qs_cpu_info);

        if (mShowingInfo) {
            state.state = Tile.STATE_ACTIVE;
            state.secondaryLabel = getCpuTemperature();
        } else {
            state.state = Tile.STATE_INACTIVE;
            state.secondaryLabel = "Tap to show";
        }
    }

    @Override
    public int getMetricsCategory() {
        return 0;
    }

    @Override
    public Intent getLongClickIntent() {
        return new Intent(android.provider.Settings.ACTION_DEVICE_INFO_SETTINGS);
    }

    @Override
    public CharSequence getTileLabel() {
        return "CPU Info";
    }

    private String getCpuTemperature() {
        try {
            BufferedReader reader = new BufferedReader(
                    new FileReader("/sys/class/thermal/thermal_zone0/temp"));
            String line = reader.readLine();
            reader.close();
            if (line != null) {
                float temp = Float.parseFloat(line.trim()) / 1000.0f;
                return String.format("%.1f°C", temp);
            }
        } catch (IOException | NumberFormatException e) {
            // 에뮬레이터에서는 thermal zone이 없을 수 있음
        }
        return "N/A";
    }
}
```

#### (2) 아이콘 리소스 추가

**파일 경로**: `frameworks/base/packages/SystemUI/res/drawable/ic_qs_cpu_info.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24"
    android:tint="?android:attr/colorControlNormal">
    <path
        android:fillColor="#FFFFFFFF"
        android:pathData="M15,21h-2v-2h2V21z M13,14h-2v5h2V14z M21,
        12h-2v4h2V12z M17,14h-2v3h2V14z M7,14H5v5h2V14z M9,
        12H7v6h2V12z M11,10H9v8h2V10z M19,10h-2v7h2V10z M3,
        14H1v5h2V14z M15,7h-2v2h2V7z M11,3H9v2h2V3z M13,
        3h-2v6h2V3z M5,10H3v3h2V10z" />
</vector>
```

#### (3) 문자열 리소스 추가

**frameworks/base/packages/SystemUI/res/values/strings.xml** 에 추가:

```xml
<string name="quick_settings_cpu_info_label">CPU Info</string>
<string name="quick_settings_cpu_info_detail">Show CPU temperature</string>
```

#### (4) 타일을 QSTileHost에 등록

**파일 경로**: `frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QSTileHost.java`

```java
// QSTileHost.java 의 createTileInternal() 메서드 내부에 추가

case "cpuinfo":
    return new CpuInfoTile(
        mHost, mUiEventLogger, mBgLooper, mMainHandler,
        mFalsingManager, mMetricsLogger, mStatusBarStateController,
        mActivityStarter, mQSLogger
    );
```

> **Android 16 참고**: 최신 AOSP에서는 Dagger/Hilt 기반의 의존성 주입을 사용합니다. 실제 등록 방식은 `QSTileHost`의 `createTile()` 메서드와 `@Binds @IntoMap` 애노테이션 기반 팩토리 패턴을 확인하세요.

#### (5) 기본 타일 목록에 추가

**frameworks/base/packages/SystemUI/res/values/config.xml** 에서:

```xml
<string name="quick_settings_tiles_default" translatable="false">
    wifi,cell,battery,dnd,flashlight,rotation,bt,airplane,cpuinfo
</string>
```

### 3.3 SystemUI 빌드

```shell
cd $ANDROID_BUILD_TOP
source build/envsetup.sh
lunch aosp_car_x86_64-aosp_current-userdebug

# SystemUI만 빌드
m SystemUI
```

### 3.4 테스트

```shell
adb root
adb remount
adb push $OUT/system/priv-app/SystemUI/SystemUI.apk /system/priv-app/SystemUI/
adb shell killall com.android.systemui
```

> **참고**: 실습 2에서 이미 remount 설정을 완료했다면, `adb root` → `adb remount` 만으로 바로 push할 수 있습니다.

---

## 실습 4: 상태바에 커스텀 아이콘 (배터리 온도) 추가

### 학습 목표
- SystemUI의 상태바(StatusBar)에 커스텀 정보(배터리 온도)를 표시하는 아이콘/텍스트를 추가합니다.
- StatusBar의 레이아웃 구조와 View 바인딩 메커니즘을 이해합니다.

### 4.1 상태바 레이아웃 구조 파악

```
SystemUI 상태바 레이아웃 파일:
frameworks/base/packages/SystemUI/res/layout/status_bar.xml

주요 영역:
┌──────────────────────────────────────────────────┐
│ [시계] [알림아이콘...]     [시스템아이콘] [배터리] │
│  (start)                    (end)                │
└──────────────────────────────────────────────────┘
```

### 4.2 배터리 온도 커스텀 뷰 생성

**파일 경로**: `frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/BatteryTemperatureView.java`

```java
package com.android.systemui.statusbar;

import android.content.Context;
import android.os.BatteryManager;
import android.os.Handler;
import android.os.Looper;
import android.util.AttributeSet;
import android.widget.TextView;

public class BatteryTemperatureView extends TextView {

    private static final long UPDATE_INTERVAL_MS = 10_000;
    private final Handler mHandler = new Handler(Looper.getMainLooper());
    private boolean mAttached = false;

    private final Runnable mUpdateRunnable = new Runnable() {
        @Override
        public void run() {
            updateTemperature();
            if (mAttached) {
                mHandler.postDelayed(this, UPDATE_INTERVAL_MS);
            }
        }
    };

    public BatteryTemperatureView(Context context) { this(context, null); }
    public BatteryTemperatureView(Context context, AttributeSet attrs) { this(context, attrs, 0); }
    public BatteryTemperatureView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        mAttached = true;
        updateTemperature();
        mHandler.postDelayed(mUpdateRunnable, UPDATE_INTERVAL_MS);
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        mAttached = false;
        mHandler.removeCallbacks(mUpdateRunnable);
    }

    private void updateTemperature() {
        float temp = getBatteryTemperature();
        if (temp > 0) {
            setText(String.format("%.0f°C", temp));
            if (temp >= 45) setTextColor(0xFFFF5252);
            else if (temp >= 38) setTextColor(0xFFFFAB40);
            else setTextColor(0xFFFFFFFF);
        } else {
            setText("--°C");
        }
    }

    private float getBatteryTemperature() {
        try {
            BatteryManager bm = getContext().getSystemService(BatteryManager.class);
            if (bm != null) {
                int temp = bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_TEMPERATURE);
                if (temp > 0) return temp / 10.0f;
            }
            java.io.BufferedReader reader = new java.io.BufferedReader(
                    new java.io.FileReader("/sys/class/thermal/thermal_zone0/temp"));
            String line = reader.readLine();
            reader.close();
            if (line != null) return Float.parseFloat(line.trim()) / 1000.0f;
        } catch (Exception e) { }
        return -1;
    }
}
```

### 4.3 레이아웃에 커스텀 뷰 추가

**frameworks/base/packages/SystemUI/res/layout/status_bar.xml** 에서 배터리 아이콘 앞에 추가:

```xml
<com.android.systemui.statusbar.BatteryTemperatureView
    android:id="@+id/battery_temperature"
    android:layout_width="wrap_content"
    android:layout_height="match_parent"
    android:gravity="center_vertical"
    android:textSize="12sp"
    android:textColor="#FFFFFFFF"
    android:paddingStart="4dp"
    android:paddingEnd="4dp"
    android:text="--°C" />
```

### 4.4 빌드 및 테스트

```shell
cd $ANDROID_BUILD_TOP
m SystemUI

adb root
adb remount
adb push $OUT/system/priv-app/SystemUI/SystemUI.apk /system/priv-app/SystemUI/
adb shell killall com.android.systemui
```

### 4.5 dumpsys로 결과 확인

```shell
adb shell dumpsys statusbar | grep -i "temperature"
adb shell dumpsys activity top | grep -A 3 "battery_temperature"
```

---

## 실습 5: Perfetto로 SystemUI 렌더링 성능 프로파일링

### 학습 목표
- Perfetto를 사용하여 SystemUI의 프레임 렌더링 성능을 측정하고 분석합니다.
- UI Thread, RenderThread, SurfaceFlinger의 동작을 이해하고 jank(끊김)의 원인을 파악합니다.

### 5.1 Perfetto 기본 개념

```
프레임 렌더링 파이프라인:
┌─────────────┐    ┌──────────────┐    ┌────────────────┐
│  UI Thread   │ →  │ RenderThread  │ →  │ SurfaceFlinger  │
│ (Measure,    │    │ (GPU 명령    │    │ (화면 합성)     │
│  Layout,     │    │  실행)        │    │                 │
│  Draw)       │    │              │    │                 │
└─────────────┘    └──────────────┘    └────────────────┘
       16.67ms 이내 (60Hz 기준)
```

- **Jank(끊김)**: 한 프레임이 16.67ms(60Hz) 또는 11.11ms(90Hz)를 초과하면 발생
- **UI Thread**: 레이아웃 계산, 이벤트 처리
- **RenderThread**: GPU 렌더링 명령 실행
- **SurfaceFlinger**: 최종 화면 합성

### 5.2 Perfetto 트레이스 캡처

#### 방법 1: 명령줄에서 직접 캡처

```shell
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/systemui_trace.perfetto-trace \
<<EOF

buffers: {
  size_kb: 63488
  fill_policy: RING_BUFFER
}
buffers: {
  size_kb: 2048
  fill_policy: RING_BUFFER
}
data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "power/suspend_resume"
      ftrace_events: "sched/sched_wakeup"
      ftrace_events: "sched/sched_wakeup_new"
      ftrace_events: "sched/sched_process_exit"
      ftrace_events: "sched/sched_process_free"
      ftrace_events: "task/task_newtask"
      ftrace_events: "task/task_rename"
      ftrace_events: "power/cpu_frequency"
      ftrace_events: "power/cpu_idle"
      atrace_categories: "gfx"
      atrace_categories: "view"
      atrace_categories: "wm"
      atrace_categories: "am"
      atrace_categories: "input"
      atrace_apps: "com.android.systemui"
    }
  }
}
data_sources: {
  config {
    name: "linux.process_stats"
    process_stats_config {
      scan_all_processes_on_start: true
      proc_stats_poll_ms: 1000
    }
  }
}
data_sources: {
  config {
    name: "android.surfaceflinger.frametimeline"
  }
}
duration_ms: 10000

EOF
```

#### 방법 2: Perfetto UI에서 캡처 설정

1. 브라우저에서 https://ui.perfetto.dev 접속
2. **Record new trace** 클릭
3. **Probes** 설정:
   - CPU: `Coarse CPU usage counter` 활성화
   - Android apps & svcs: `gfx`, `view`, `wm`, `am`, `input` 체크, Per-app: `com.android.systemui`
   - Display: `Frame timeline` 활성화
4. Duration 10s, Buffer 64MB → **Start recording**

### 5.3 트레이스 파일 가져오기

```shell
adb pull /data/misc/perfetto-traces/systemui_trace.perfetto-trace .
# https://ui.perfetto.dev 에서 파일 드래그 & 드롭
```

### 5.4 Perfetto UI 기본 조작

| 키/동작 | 기능 |
|---------|------|
| `W` / `S` | 타임라인 확대 / 축소 |
| `A` / `D` | 좌 / 우 이동 |
| `M` | 두 지점 사이 시간 측정 |
| 클릭 | 특정 이벤트 선택 및 상세 정보 확인 |
| `F` | 선택 영역에 맞춰 확대 |

### 5.5 실습: SystemUI 조작하며 트레이스 캡처

```shell
# 트레이스 캡처 시작 (10초, 백그라운드)
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/qs_scroll.perfetto-trace \
<<EOF
buffers: { size_kb: 63488 fill_policy: RING_BUFFER }
data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      atrace_categories: "gfx"
      atrace_categories: "view"
      atrace_categories: "wm"
      atrace_categories: "input"
      atrace_apps: "com.android.systemui"
    }
  }
}
data_sources: {
  config { name: "android.surfaceflinger.frametimeline" }
}
duration_ms: 10000
EOF &

# SystemUI 조작
sleep 2
adb shell input swipe 500 0 500 1500 300    # QS 열기
sleep 1
adb shell input swipe 500 1500 500 0 300    # QS 닫기
sleep 1
adb shell input swipe 500 0 500 1500 300    # 다시 열기
sleep 5

# 트레이스 가져오기
adb pull /data/misc/perfetto-traces/qs_scroll.perfetto-trace .
echo "https://ui.perfetto.dev 에서 열어 분석하세요"
```

### 5.6 gfxinfo를 활용한 간이 프레임 분석

```shell
adb shell dumpsys gfxinfo com.android.systemui reset
adb shell input swipe 500 0 500 1500 300
sleep 2
adb shell dumpsys gfxinfo com.android.systemui
```

확인 항목: **Janky frames 비율 5% 미만**이면 양호, **90th percentile 16.67ms 미만**이면 대부분 정상

### 5.7 분석 결과 해석

| 증상 | Perfetto 확인 위치 | 가능한 원인 |
|------|-------------------|-----------|
| UI Thread가 느림 | `Choreographer#doFrame` > 16ms | 복잡한 레이아웃 |
| RenderThread가 느림 | `DrawFrame` > 8ms | GPU 오버드로우 |
| 입력 지연 | `deliverInputEvent` > 8ms | 메인 스레드 블로킹 |
| Binder 지연 | `binder transaction` 장기 대기 | IPC 호출 병목 |
| SurfaceFlinger 지연 | `onMessageReceived` > 4ms | 합성 레이어 과다 |

---

## 부록: 실습 환경 빠른 설정 가이드

### 에뮬레이터 실행

```shell
cd $ANDROID_BUILD_TOP
source build/envsetup.sh
lunch aosp_car_x86_64-aosp_current-userdebug
emulator &
```

### ADB 연결 및 시스템 파티션 쓰기 활성화 (최초 1회)

```shell
# 최초 설정 (verity 비활성화)
adb root
adb remount          # → "reboot required" 메시지 확인
adb reboot

# 재부팅 후 실제 마운트
adb root
adb remount          # → "remount succeeded" 확인
```

### SystemUI 빠른 재시작

```shell
adb shell killall com.android.systemui
```

### 스크린샷 및 화면 녹화

```shell
adb shell screencap -p /sdcard/screenshot.png
adb pull /sdcard/screenshot.png .

adb shell screenrecord --time-limit 10 /sdcard/recording.mp4
adb pull /sdcard/recording.mp4 .
```

---

## 참고 자료

- [Android dumpsys 공식 문서](https://developer.android.com/tools/dumpsys)
- [Android RRO 리소스 오버레이](https://developer.android.com/guide/topics/resources/overlays)
- [AOSP QS 타일 구현 가이드](https://android.googlesource.com/platform/frameworks/base/+/master/packages/SystemUI/docs/qs-tiles.md)
- [QS 타일 개발자 API](https://developer.android.com/develop/ui/views/quicksettings-tiles)
- [Perfetto 공식 사이트](https://perfetto.dev/)
- [Perfetto 시작하기](https://perfetto.dev/docs/getting-started/start-using-perfetto)
- [Android 16 릴리스 노트](https://source.android.com/docs/whatsnew/android-16-release)
