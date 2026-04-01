# Android 16 기반 SRO 실습: Product Flavors로 리소스 오버레이 구현하기

## SRO와 RRO의 핵심 차이점

| 구분 | SRO (정적 오버레이) | RRO (동적 오버레이) |
|:-----|:-------------------|:-------------------|
| **적용 시점** | 앱 빌드 시점 | 앱 실행 중 (런타임) |
| **적용 방식** | Gradle 빌드 스크립트 설정 | `adb` 명령어 또는 코드로 활성화 |
| **변경 가능성** | 빌드 후 변경 불가능 | 언제든지 활성화/비활성화 가능 |
| **패키지** | 타겟 앱 APK에 포함됨 | 별도의 RRO APK 파일로 존재 |
| **주 용도** | 제조사별 커스텀 빌드, 고객사별 앱 분리 | SystemUI 테마 변경, A/B 테스트 |

> **Product Flavors란?**
> Gradle의 Product Flavors는 하나의 소스 코드에서 여러 버전의 앱을 빌드할 수 있게 해주는 기능입니다.
> `main` 소스 세트 위에 **계층적으로** 리소스를 추가하므로, 중복 리소스 충돌 없이 안전하게 오버레이할 수 있습니다.
> 이것이 단순 `sourceSets` 방식보다 권장되는 이유입니다.

---

## 실습 환경

- Android Studio Narwhal (2025.x) 이상
- Android 16 (API Level 36) SDK 설치
- 에뮬레이터: API 34~36 모두 가능 (SRO는 빌드 시점 적용이므로 API 버전 무관)

---

## 1단계: SROTarget 프로젝트 생성

### 1.1 새 프로젝트 만들기

1. Android 스튜디오에서 **File > New > New Project...** 선택
2. **Empty Views Activity** 템플릿 선택 → **Next**
3. 프로젝트 설정:
   - Application name: `SROTarget`
   - Package name: `com.example.srotarget`
   - Language: **Java**
   - Minimum SDK: **API 34** (에뮬레이터 호환을 위해)
4. **Finish** 클릭

### 1.2 레이아웃 파일 작성

**app/src/main/res/layout/activity_main.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/background_color"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/hello_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/hello_world"
        android:textColor="@color/text_color"
        android:textSize="28sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

### 1.3 기본 리소스 파일 작성

**app/src/main/res/values/strings.xml** (원본)

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">SROTarget</string>
    <string name="hello_world">Hello, Original!</string>
</resources>
```

**app/src/main/res/values/colors.xml** (원본)

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
    <color name="background_color">#FFFFFFFF</color>
    <color name="text_color">#FF000000</color>
</resources>
```

### 1.4 빌드 후 원본 상태 확인

앱을 실행하여 **흰색 배경에 검정색 텍스트 "Hello, Original!"**이 표시되는지 확인합니다.

---

## 2단계: Product Flavors 설정

### 2.1 build.gradle.kts 수정

**app/build.gradle.kts** 파일을 열고, `android { }` 블록 안에 `flavorDimensions`와 `productFlavors`를 추가합니다.

```kotlin
android {
    namespace = "com.example.srotarget"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.srotarget"
        minSdk = 34
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }

    // ──────────────────────────────────────────
    // ✅ 여기부터 Product Flavors 설정 추가
    // ──────────────────────────────────────────

    // 1. 버전의 종류(차원)를 정의합니다.
    flavorDimensions += "version"

    // 2. 각 버전(flavor)을 만듭니다.
    productFlavors {
        create("original") {
            dimension = "version"
            // 원본 버전: main 리소스만 사용 (추가 설정 불필요)
        }
        create("themed") {
            dimension = "version"
            // 테마 버전: themed 폴더의 리소스가 main 위에 오버레이됨
            // 필요 시 applicationIdSuffix로 별도 패키지 구분 가능
            // applicationIdSuffix = ".themed"
        }
    }

    // ──────────────────────────────────────────

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }
}
```

### 2.2 Gradle Sync 실행

파일 수정 후 Android 스튜디오 상단에 나타나는 **Sync Now** 배너를 클릭합니다.

> **Sync 후 변화**: Build Variants 탭에 `originalDebug`, `originalRelease`, `themedDebug`, `themedRelease` 4가지 빌드 변형이 생성됩니다.

---

## 3단계: themed 전용 리소스 폴더 생성

### 3.1 폴더 구조 만들기

Android 스튜디오의 **Project** 뷰(좌측 상단 드롭다운에서 "Project" 선택)로 전환한 뒤, 다음 폴더 구조를 생성합니다.

```
SROTarget/
└── app/
    └── src/
        ├── main/                         ← 공통 리소스 (모든 flavor에서 사용)
        │   ├── java/
        │   ├── res/
        │   │   ├── layout/
        │   │   │   └── activity_main.xml
        │   │   └── values/
        │   │       ├── colors.xml        ← 원본 색상
        │   │       └── strings.xml       ← 원본 문자열
        │   └── AndroidManifest.xml
        │
        ├── original/                     ← original flavor (비어 있어도 됨)
        │   └── (비어 있음 — main 리소스를 그대로 사용)
        │
        └── themed/                       ← themed flavor 전용 리소스
            └── res/
                └── values/
                    ├── colors.xml        ← 오버레이할 색상
                    └── strings.xml       ← 오버레이할 문자열
```

### 3.2 폴더 생성 방법 (Android 스튜디오에서)

1. `app/src/` 폴더를 우클릭 → **New > Directory**
2. `themed/res/values` 입력 → **Enter** (중간 폴더가 자동 생성됨)

### 3.3 themed 전용 리소스 파일 작성

**app/src/themed/res/values/strings.xml** — 문자열 오버레이

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="hello_world">Hello, SRO Themed! 🎨</string>
</resources>
```

> **중요**: `app_name`은 여기에 넣지 않습니다. `main`의 `app_name`이 그대로 유지됩니다.
> themed 폴더에는 **덮어쓰고 싶은 리소스만** 넣으면 됩니다.

**app/src/themed/res/values/colors.xml** — 색상 오버레이

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="background_color">#4A148C</color>
    <color name="text_color">#F3E5F5</color>
</resources>
```

> **Product Flavors의 장점**: `main`의 `colors.xml`을 수정하거나 주석 처리할 필요가 **없습니다**.
> Gradle이 자동으로 themed 폴더의 리소스를 main 위에 오버레이합니다.
> 단순 `sourceSets` 방식에서는 중복 리소스를 주석 처리해야 했지만, Product Flavors에서는 불필요합니다.

---

## 4단계: Build Variant 선택 및 실행

### 4.1 Build Variant 변경 방법

1. Android 스튜디오 메뉴: **Build > Select Build Variant...**
   (또는 좌측 하단 **Build Variants** 탭 클릭)

2. 드롭다운에서 **Active Build Variant**를 변경합니다:

| Build Variant | 결과 |
|:-------------|:-----|
| `originalDebug` | 흰색 배경, 검정 텍스트, "Hello, Original!" |
| `themedDebug` | **보라색 배경, 연보라 텍스트, "Hello, SRO Themed! 🎨"** |

### 4.2 original 버전으로 실행

1. Build Variant를 **`originalDebug`**로 선택
2. **Shift + F10** (또는 Run 버튼)으로 앱 실행
3. 확인: 흰색 배경, 검정 텍스트, **"Hello, Original!"**

### 4.3 themed 버전으로 실행

1. Build Variant를 **`themedDebug`**로 변경
2. **Shift + F10** 으로 앱 실행
3. 확인: **보라색 배경, 연보라 텍스트, "Hello, SRO Themed! 🎨"**

> **핵심 포인트**: `main` 폴더의 원본 리소스는 전혀 수정하지 않았습니다.
> Gradle이 빌드 시 themed 폴더의 리소스를 main 위에 자동으로 덮어씌웁니다.

---

## 5단계: 결과 비교 및 원리 이해

### 5.1 리소스 병합 우선순위

Gradle은 빌드 시 다음 순서로 리소스를 병합합니다 (아래로 갈수록 우선순위 높음):

```
라이브러리(AAR) 리소스       ← 가장 낮은 우선순위
      ↑
main 소스 세트 리소스
      ↑
Product Flavor 리소스        ← 더 높은 우선순위
      ↑
Build Type 리소스            ← 가장 높은 우선순위
```

따라서 `themed` flavor의 `colors.xml`에 정의된 `background_color`가 `main`의 동일 리소스를 덮어씁니다.

### 5.2 APK 내용 비교

빌드 후 APK 내부를 확인하면, 선택한 flavor에 따라 최종 리소스가 다릅니다.

```shell
# originalDebug APK 분석
# Build > Analyze APK... > app-original-debug.apk 선택
# resources.arsc에서 background_color = #FFFFFFFF (흰색)

# themedDebug APK 분석
# Build > Analyze APK... > app-themed-debug.apk 선택
# resources.arsc에서 background_color = #4A148C (보라색)
```

### 5.3 SRO vs RRO 비교 정리

| 항목 | SRO (이 실습) | RRO (이전 실습) |
|:-----|:-------------|:---------------|
| 리소스 변경 시점 | **빌드 시** (APK에 포함) | **런타임** (별도 APK 설치 후 활성화) |
| main 리소스 수정 필요 | **불필요** (Product Flavors 사용 시) | **불필요** (overlayable 선언만 필요) |
| 중복 충돌 위험 | **없음** (Gradle이 자동 병합) | **없음** (OverlayManager가 관리) |
| 되돌리기 | Build Variant 변경 후 재빌드 | `cmd overlay disable` 즉시 복원 |
| 활성화 명령 | 없음 (빌드에 포함) | `adb shell cmd overlay enable` |
| 대표 사용 사례 | 고객사별 앱 브랜딩, White Label 앱 | 런타임 테마 변경, SystemUI 커스터마이징 |

---

## 심화: Flavor별 기능 분기 (선택)

Product Flavors는 리소스뿐 아니라 **Java/Kotlin 코드**도 분리할 수 있습니다.

### 코드 분기 예시

```
app/src/
├── main/java/com/example/srotarget/
│   └── MainActivity.java              ← 공통 코드
├── original/java/com/example/srotarget/
│   └── BrandConfig.java               ← original 전용 코드
└── themed/java/com/example/srotarget/
    └── BrandConfig.java               ← themed 전용 코드
```

**app/src/original/java/.../BrandConfig.java**

```java
package com.example.srotarget;

public class BrandConfig {
    public static final String BRAND_NAME = "Original App";
    public static final boolean SHOW_LOGO = false;
}
```

**app/src/themed/java/.../BrandConfig.java**

```java
package com.example.srotarget;

public class BrandConfig {
    public static final String BRAND_NAME = "Themed App";
    public static final boolean SHOW_LOGO = true;
}
```

**main의 MainActivity.java에서 사용:**

```java
// BrandConfig는 선택된 flavor에 따라 자동으로 결정됨
Log.d("SRO", "Brand: " + BrandConfig.BRAND_NAME);
```

> **주의**: 코드 분기 시 해당 클래스는 `main`에 존재하면 안 됩니다.
> `main`에 공통 인터페이스를 두고, 각 flavor에 구현체를 두는 패턴이 일반적입니다.

---

## 트러블슈팅

### Q1: Sync 후 Build Variant가 보이지 않습니다

`flavorDimensions`와 `productFlavors`가 `android { }` 블록 **안에** 있는지 확인하세요. 블록 바깥에 있으면 인식되지 않습니다.

### Q2: `Duplicate resources` 에러가 발생합니다

themed 폴더에 `app_name` 같은 리소스를 넣었는데, main에도 동일한 리소스가 있으면 발생할 수 있습니다. Product Flavors에서는 보통 자동 병합되지만, **동일 파일 내 동일 키**가 양쪽에 있으면 충돌합니다.

해결: themed 폴더에는 **덮어쓸 리소스만** 포함하세요. main과 겹치는 키만 themed에 넣으면 Gradle이 자동으로 themed의 값을 우선 적용합니다.

### Q3: original과 themed 앱을 동시에 설치하고 싶습니다

`build.gradle.kts`에서 `applicationIdSuffix`를 설정하면 별도 앱으로 동시 설치 가능합니다:

```kotlin
productFlavors {
    create("original") {
        dimension = "version"
    }
    create("themed") {
        dimension = "version"
        applicationIdSuffix = ".themed"   // 패키지: com.example.srotarget.themed
    }
}
```

---

## 참고 자료

- [Android Product Flavors 공식 문서](https://developer.android.com/build/build-variants)
- [Gradle 리소스 병합 규칙](https://developer.android.com/build/manage-manifests)
- [Android 16 동작 변경사항](https://developer.android.com/about/versions/16/behavior-changes-all)
