# Android 16 기반 RRO(런타임 리소스 오버레이) 실습 예제

Android 16(API Level 36, 코드명 Baklava)에서 런타임 리소스 오버레이(Runtime Resource Overlay, RRO)를 사용하여 앱의 리소스를 동적으로 변경하는 방법을 단계별로 안내합니다. 이 예제에서는 간단한 앱을 만들고, RRO를 통해 문자열과 색상 리소스를 변경해보겠습니다.

---

## 실습 1: 문자열 리소스 오버레이

---

### 1단계: 실습 준비

Android 16(API Level 36)을 지원하는 최신 버전의 Android 스튜디오를 준비합니다.

- **Android Studio Narwhal (2025.x)** 이상 권장
- Android SDK Platform API 36 설치
- Android 16 에뮬레이터 시스템 이미지 설치

> **Android 16 변경사항**: Android 16에서는 `targetSdkVersion 36`이 적용되며, 대형 화면(600dp 이상)에서 화면 방향·리사이즈 제한이 해제됩니다. UI가 다양한 화면 비율에 적응하도록 구현해야 합니다.

---

### 2단계: 타겟 앱 만들기

RRO를 적용할 대상이 되는 간단한 앱을 만듭니다.

#### 1. 새 프로젝트 생성

- Android 스튜디오에서 **File > New > New Project...**를 선택합니다.
- **Empty Views Activity** 템플릿을 선택하고 **Next**를 클릭합니다.
- 애플리케이션 이름: `RROTarget`
- 패키지 이름: `com.example.rrotarget`
- Language: **Java**
- **Minimum SDK를 API 36: Android 16**으로 설정하고 **Finish**를 클릭합니다.

#### 2. 레이아웃 및 리소스 정의

**app/res/layout/activity_main.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/hello_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/hello_world"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

**app/res/values/strings.xml** — 문자열 리소스 정의

```xml
<resources>
    <string name="app_name">RROTarget</string>
    <string name="hello_world">how are you</string>
</resources>
```

**app/res/values/colors.xml** — 색상 리소스 정의

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
    <color name="background_color">#FFFFFF</color>
    <color name="text_color">#000000</color>
</resources>
```

#### 3. 오버레이 가능한 리소스 지정 (중요)

RRO가 리소스를 변경하기 위해서는 타겟 앱에서 해당 리소스가 **오버레이 가능**하다고 명시해야 합니다.

`app/res/values/` 디렉터리에 **overlayable.xml** 파일을 새로 만듭니다.

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <overlayable name="RROTargetTheme">
        <policy type="public">
            <item type="string" name="hello_world" />
        </policy>
    </overlayable>
</resources>
```

- `<overlayable>` 태그의 `name`은 오버레이를 식별하는 데 사용됩니다.
- `<policy type="public">`: RRO가 접근할 수 있도록 공개 정책을 설정합니다.
- `<item>`: 오버레이를 허용할 리소스의 `type`(종류)과 `name`(이름)을 지정합니다.

> **Android 16 참고**: Android 11부터 `<overlayable>` 선언이 의무화되었으며, Android 16에서도 동일하게 적용됩니다. 선언되지 않은 리소스에 대한 오버레이 시도는 무시됩니다.

#### 4. build.gradle 설정 확인

**app/build.gradle.kts** (Android 16 기준)

```kotlin
android {
    namespace = "com.example.rrotarget"
    compileSdk = 36

    defaultConfig {
        applicationId = "com.example.rrotarget"
        minSdk = 36
        targetSdk = 36
        versionCode = 1
        versionName = "1.0"
    }
    // ...
}
```

> **Android 16 변경사항**: Android 스튜디오 최신 버전에서는 `build.gradle.kts` (Kotlin DSL)가 기본이며, `compileSdk`, `minSdk`, `targetSdk`를 모두 **36**으로 설정합니다.

#### 5. 타겟 앱 빌드

Android 스튜디오에서 **Build > Generate App Bundles or APKs > Generate APKs**를 선택하여 RROTarget 패키지의 APK 파일을 생성합니다.

---

### 3단계: RRO 패키지 만들기

타겟 앱의 리소스를 덮어쓸 RRO 패키지를 만듭니다.

#### 1. 새 프로젝트 생성

- Android 스튜디오에서 **File > New > New Project...**를 선택합니다.
- **Empty Views Activity** 템플릿을 선택하고 **Next**를 클릭합니다.
- 애플리케이션 이름: `RROOverlay`
- 패키지 이름: `com.example.rrooverlay`
- Language: **Java**
- **Minimum SDK를 API 36: Android 16**으로 설정하고 **Finish**를 클릭합니다.

#### 2. AndroidManifest.xml 수정

RRO 패키지는 자바 코드가 필요 없으며, `<overlay>` 태그를 추가하는 것이 핵심입니다.

**app/manifests/AndroidManifest.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.rrooverlay">

    <application
        android:hasCode="false"
        android:label="RRO Overlay">
    </application>

    <overlay
        android:targetPackage="com.example.rrotarget"
        android:targetName="RROTargetTheme"
        android:priority="1"
        android:isStatic="false" />
</manifest>
```

각 속성의 의미:

- `android:hasCode="false"`: 이 패키지에 실행 가능한 코드가 없음을 나타냅니다.
- `<overlay>`: 이 패키지가 RRO임을 명시합니다.
  - `android:targetPackage`: 오버레이할 타겟 앱의 패키지 이름(`com.example.rrotarget`)을 정확히 입력합니다.
  - `android:targetName`: 타겟 앱의 `overlayable.xml`에 정의한 `<overlayable>`의 `name`(`RROTargetTheme`)을 입력합니다.
  - `android:isStatic="false"`: 런타임에 동적으로 활성화/비활성화할 수 있는 RRO임을 의미합니다.

#### 3. 변경할 리소스 정의

RRO 패키지에서 타겟 앱의 리소스와 **동일한 이름**으로 새로운 값을 정의합니다.

**app/res/values/strings.xml**

```xml
<resources>
    <string name="app_name">RROOverlay</string>
    <string name="hello_world">Hello, RRO on Android 16! 🚀</string>
</resources>
```

**app/res/values/colors.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
    <color name="background_color">#FFFFFF</color>
    <color name="text_color">#000000</color>
</resources>
```

#### 4. RRO 패키지 빌드

Android 스튜디오에서 **Build > Generate App Bundles or APKs > Generate APKs**를 선택하여 RROOverlay 패키지의 APK 파일을 생성합니다.

---

### 4단계: RRO 설치 및 활성화

ADB(Android Debug Bridge)를 사용하여 RRO를 설치하고 활성화합니다.

#### 1. RROTarget APK 설치

생성된 APK 파일(`app-debug.apk`)의 경로를 확인합니다.
(보통 `AndroidStudioProjects/RROTarget/app/build/outputs/apk/debug/` 경로에 있습니다)

해당 디렉토리의 `app-debug.apk` 파일을 작업 디렉토리의 `rrotarget.apk`로 복사한 뒤 설치합니다.

```shell
adb install rrotarget.apk
```

#### 2. RROOverlay APK 설치

생성된 APK 파일(`app-debug.apk`)의 경로를 확인합니다.
(보통 `AndroidStudioProjects/RROOverlay/app/build/outputs/apk/debug/` 경로에 있습니다)

해당 디렉토리의 `app-debug.apk` 파일을 작업 디렉토리의 `rrooverlay.apk`로 복사한 뒤 설치합니다.

```shell
adb install rrooverlay.apk
```

#### 3. RRO 활성화

설치 후, RRO를 활성화해야 변경 사항이 적용됩니다.

```shell
# adb shell 접속
adb shell

# RRO 오버레이 활성화
cmd overlay enable com.example.rrooverlay
```

#### 4. RRO 상태 모니터링

```shell
# 오버레이 목록에서 확인
cmd overlay list | grep com.example.rrooverlay

# 오버레이 상세 정보 확인
cmd overlay dump com.example.rrooverlay
```

#### 5. 변경 사항 확인

RROTarget 앱을 다시 실행하거나, 이미 실행 중이었다면 앱을 재시작합니다.
문구가 **"Hello, RRO on Android 16! 🚀"**로 변경된 것을 확인할 수 있습니다.

---

### 5단계: RRO 비활성화 및 제거

RRO를 비활성화하거나 시스템에서 제거할 수 있습니다.

**RRO 비활성화:**

```shell
adb shell cmd overlay disable com.example.rrooverlay
```

**RRO 패키지 제거:**

```shell
adb uninstall com.example.rrooverlay
```

---

## 실습 2: 드로어블(Drawable) 리소스 오버레이

이전 예제에서 만든 RROTarget 앱에 아이콘과 배경 이미지를 추가하고, 새로운 RRO 패키지를 통해 이 이미지들을 변경해보겠습니다.

---

### 1단계: 기존 RROTarget 앱 수정

RROTarget 앱의 코드를 수정하여 ImageView를 추가하고, 배경을 색상 대신 드로어블로 설정합니다.

#### 1. 배경 드로어블 만들기

RROTarget 프로젝트의 `app/res/drawable/` 디렉터리에 **background_shape.xml** 파일을 새로 만듭니다.

```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <solid android:color="#E0E0E0" />
    <corners android:radius="16dp" />
    <stroke android:width="4dp" android:color="#CCCCCC" />
</shape>
```

#### 2. 레이아웃(activity_main.xml) 수정

ImageView를 추가하고, 배경을 방금 만든 드로어블로 변경합니다.

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@drawable/background_shape"
    android:gravity="center"
    android:padding="32dp"
    tools:context=".MainActivity">

    <ImageView
        android:id="@+id/icon_view"
        android:layout_width="96dp"
        android:layout_height="96dp"
        android:layout_centerHorizontal="true"
        android:src="@drawable/ic_target_icon"
        android:contentDescription="@string/app_name" />

    <TextView
        android:id="@+id/hello_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@id/icon_view"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="24dp"
        android:text="@string/hello_world"
        android:textColor="@color/text_color"
        android:textSize="28sp" />

</RelativeLayout>
```

> **Android 16 참고**: `ImageView`에 `android:contentDescription` 속성을 추가하여 접근성을 개선하는 것이 권장됩니다. Android 16에서는 접근성 관련 Lint 검사가 더 강화되었습니다.

#### 3. 아이콘 추가

- **File > New > Vector Asset**을 선택하여 아이콘을 추가합니다.
- Clip Art에서 원하는 아이콘(예: `android`)을 선택합니다.
- 이름을 `ic_target_icon`으로 지정하고 **Next**, **Finish**를 클릭합니다.
- 이 파일은 `app/res/drawable/`에 생성됩니다.

#### 4. overlayable.xml 업데이트 (중요)

새로 추가한 드로어블 리소스들을 RRO가 덮어쓸 수 있도록 `app/res/values/overlayable.xml` 파일에 추가합니다.

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <overlayable name="RROTargetTheme">
        <policy type="public">
            <item type="color" name="background_color" />
            <item type="color" name="text_color" />
            <item type="string" name="hello_world" />
            <item type="drawable" name="background_shape" />
            <item type="drawable" name="ic_target_icon" />
        </policy>
    </overlayable>
</resources>
```

#### 5. 타겟 앱 다시 빌드 및 설치

Android 스튜디오에서 **Build > Generate App Bundles or APKs > Generate APKs**를 선택하여 RROTarget 패키지의 APK 파일을 재생성 후 설치합니다.

```shell
adb install -r rrotarget.apk
```

---

### 2단계: RROOverlay 패키지 수정

아이콘과 배경 드로어블을 교체할 RROOverlay 패키지를 수정합니다.

#### 1. 교체할 드로어블 리소스 추가

타겟 앱의 리소스와 **반드시 동일한 파일 이름**을 사용해야 합니다.

**새로운 배경 드로어블** — `app/res/drawable/background_shape.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <gradient
        android:angle="90"
        android:endColor="#673AB7"
        android:startColor="#B39DDB" />
    <corners android:radius="32dp" />
</shape>
```

#### 2. 새로운 아이콘 추가

- **File > New > Vector Asset**을 선택합니다.
- 다른 아이콘(예: `rocket_launch`)을 선택하고, 파일 이름을 **`ic_target_icon`으로 정확하게 지정**합니다.

#### 3. RRO 패키지 빌드

Android 스튜디오에서 **Build > Generate App Bundles or APKs > Generate APKs**를 선택하여 RROOverlay 패키지의 APK 파일을 생성합니다.

---

### 3단계: 테스트

이후 테스트 과정은 실습 1의 4단계~5단계와 동일합니다.

```shell
# RROOverlay APK 재설치
adb install -r rrooverlay.apk

# RRO 활성화
adb shell cmd overlay enable com.example.rrooverlay

# RROTarget 앱 재시작 후 변경 확인
# - 배경이 회색 단색 → 보라색 그래디언트로 변경
# - 아이콘이 Android → 로켓으로 변경
```

---

## Android 14 → 16 RRO 마이그레이션 요약

| 항목 | Android 14 (API 34) | Android 16 (API 36) |
|------|---------------------|---------------------|
| `compileSdk` | 34 | **36** |
| `minSdk` / `targetSdk` | 34 | **36** |
| `<overlayable>` 필수 여부 | 필수 (Android 11+) | **필수** (동일) |
| `<overlay>` 속성 | 동일 | **동일** |
| 에뮬레이터 이미지 | API 34 | **API 36** |
| build.gradle 형식 | Groovy / Kotlin DSL | **Kotlin DSL 기본** |
| 대형 화면 적응 | 선택 | **필수** (600dp 이상) |
| `cmd overlay` 명령어 | 동일 | **동일** |

> **핵심 포인트**: RRO의 기본 메커니즘(overlayable 선언, overlay 매니페스트, cmd overlay 명령어)은 Android 14와 16에서 동일합니다. 주요 변경 사항은 API Level(34→36), SDK 빌드 도구 버전, 그리고 대형 화면 적응형 UI 정책입니다.

---

## 참고 자료

- [Android 리소스 오버레이 공식 문서](https://developer.android.com/guide/topics/resources/overlays)
- [Android 16 동작 변경사항](https://developer.android.com/about/versions/16/behavior-changes-all)
- [Android 16 API Level 36 개발자 가이드](https://developer.android.com/about/versions/16)
- [RRO 오버레이 매니페스트 레퍼런스](https://developer.android.com/guide/topics/manifest/overlay-element)
