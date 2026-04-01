# Android 16 간단한 APEX 빌드 실습 가이드 (aosp_car_x86_64)

## 1. 개발 환경 준비

### 필수 요구사항
- Android 16 (API Level 36, 코드명 Baklava) AOSP 소스 코드
- aosp_car_x86_64 타겟 환경 (또는 Cuttlefish 가상 디바이스)
- Linux 개발 환경 (Ubuntu 22.04 이상 권장)
- 최소 400GB 이상 디스크 공간, 64GB RAM 권장

### AOSP 소스 다운로드
```bash
# Android 16 소스 다운로드 (android-latest-release 권장)
# 참고: 2026년부터 AOSP는 Q2/Q4에 소스 코드를 공개하며,
#       android-latest-release 매니페스트 브랜치 사용을 권장합니다.
mkdir aosp-android16 && cd aosp-android16

repo init -u https://android.googlesource.com/platform/manifest \
    -b android-latest-release --partial-clone
repo sync -c -j$(nproc)
```

### 환경 설정
```bash
# AOSP 빌드 환경 설정
source build/envsetup.sh

# Android 16에서 변경된 lunch 명령어 형식:
#   lunch <product>-<release>-<variant>
# Car 에뮬레이터 타겟 설정
lunch aosp_car_x86_64-aosp_current-userdebug

# 참고: Cuttlefish 가상 디바이스를 사용할 경우
# lunch aosp_cf_x86_64_auto-aosp_current-userdebug
```

> **Android 16 변경사항 (lunch 명령어)**
> - Android 14: `lunch sdk_car_x86_64` (2-파트 형식)
> - Android 16: `lunch aosp_car_x86_64-aosp_current-userdebug` (3-파트 형식: product-release-variant)
> - `sdk_car_x86_64` 타겟은 더 이상 제공되지 않을 수 있으며, `aosp_car_x86_64` 또는 Cuttlefish(`aosp_cf_x86_64_auto`) 사용을 권장합니다.

## 2. 프로젝트 구조

```
$ANDROID_BUILD_TOP/
├── simple_apex/                          ← APEX 모듈 디렉토리
│   ├── Android.bp
│   ├── apex_manifest.json
│   ├── binary/
│   │   ├── Android.bp
│   │   └── main.cpp
│   ├── com.example.simple.pem
│   ├── com.example.simple.x509.pem
│   ├── com.example.simple.pk8
│   └── com.example.simple.avbpubkey
│
└── system/sepolicy/apex/                 ← file_contexts는 반드시 여기에 위치
    └── com.example.simple-file_contexts
```

> **⚠️ 중요: `file_contexts` 배치 규칙**
> - AOSP 빌드 시스템(Soong)은 `file_contexts`가 반드시 `system/sepolicy/` 하위에 존재하도록 강제합니다.
> - APEX 모듈 디렉토리 내부에 `file_contexts`를 배치하면 빌드 오류가 발생합니다:
>   ```
>   error: file_contexts: should be under system/sepolicy, but found in "simple_apex/"
>   ```
> - 이 규칙은 Android 14~16 모두 동일하게 적용됩니다.

## 3. 소스 코드 작성

### 3.1 간단한 실행 파일

**binary/main.cpp**
```cpp
#include <iostream>
#include <string>

int main() {
    std::cout << "Hello from Simple APEX on Android 16 (Baklava)!" << std::endl;
    std::cout << "API Level: 36" << std::endl;
    std::cout << "This is a minimal APEX example" << std::endl;
    return 0;
}
```

### 3.2 바이너리 빌드 설정

**binary/Android.bp**
```json
cc_binary {
    name: "simple_apex_binary",
    srcs: ["main.cpp"],
    apex_available: ["com.example.simple"],
    // Android 16 = API Level 36
    min_sdk_version: "36",
}
```

### 3.3 APEX 매니페스트

**apex_manifest.json**
```json
{
    "name": "com.example.simple",
    "version": 1,
    "versionName": "1.0.0"
}
```

### 3.4 메인 APEX 빌드 설정

**Android.bp**
```json
apex {
    name: "com.example.simple",
    manifest: "apex_manifest.json",
    // ⚠️ file_contexts 속성은 명시하지 않음
    // → 빌드 시스템이 자동으로 아래 경로를 탐색:
    //   system/sepolicy/apex/com.example.simple-file_contexts
    binaries: ["simple_apex_binary"],
    key: "com.example.simple.key",
    certificate: ":com.example.simple.certificate",
    // Android 16 = API Level 36
    min_sdk_version: "36",
    // 커스텀 APEX는 Mainline 업데이트 대상이 아니므로 false
    updatable: false,
    // ⚠️ compressible 속성은 사용하지 않음
    // updatable: false + compressible: true → 빌드 오류 발생:
    //   "error: compressible: do not compress non-updatable APEX"
    // APEX 압축은 updatable: true인 Mainline 모듈에서만 사용 가능
}

apex_key {
    name: "com.example.simple.key",
    public_key: "com.example.simple.avbpubkey",
    private_key: "com.example.simple.pem",
}

android_app_certificate {
    name: "com.example.simple.certificate",
    certificate: "com.example.simple",
}
```

> **⚠️ Android.bp 작성 시 주의사항 (빌드 오류 방지)**
>
> | 속성 | 올바른 설정 | 잘못된 설정 | 빌드 오류 메시지 |
> |------|-----------|-----------|----------------|
> | `file_contexts` | **생략** (자동 탐색) | APEX 내부 파일 참조 | `should be under system/sepolicy, but found in "simple_apex/"` |
> | `compressible` | **생략** | `updatable: false`와 함께 `true` 설정 | `do not compress non-updatable APEX` |

## 4. 인증서 생성

```bash
cd simple_apex

# 1. 4096비트 RSA 개인키 생성 (중요: 4096비트 필수!)
openssl genrsa -out com.example.simple.pem 4096

# 2. X.509 인증서 생성 (유효기간 10년으로 설정)
openssl req -new -x509 -key com.example.simple.pem \
    -out com.example.simple.x509.pem -days 3650 \
    -subj "/C=KR/ST=Seoul/L=Seoul/O=SimpleApex/OU=Development/CN=com.example.simple"

# 3. 개인키를 PKCS#8 DER 형식으로 변환
openssl pkcs8 -topk8 -outform DER \
    -in com.example.simple.pem -inform PEM \
    -out com.example.simple.pk8 -nocrypt

# 4. AVB 공개키 생성
#    avbtool은 AOSP 빌드 환경 설정 후 사용 가능
avbtool extract_public_key \
    --key com.example.simple.pem \
    --output com.example.simple.avbpubkey

# 5. 파일 및 키 크기 확인
ls -la com.example.simple.*
echo "=== Checking key size ==="
openssl rsa -in com.example.simple.pem -text -noout | grep "Private-Key"
echo "=== Verifying certificate ==="
openssl x509 -in com.example.simple.x509.pem -text -noout | head -15
```

> **참고**: 인증서 생성 절차는 Android 14와 동일합니다. 다만 프로덕션 용도에서는 유효기간을 충분히 길게 설정하세요.

## 5. SELinux 정책 설정

AOSP 빌드 시스템은 APEX의 `file_contexts`를 **반드시 `system/sepolicy/apex/` 디렉토리**에서 찾습니다.
파일명 규칙은 `{APEX 모듈명}-file_contexts` 형식입니다.

```bash
# AOSP 루트 디렉토리에서 실행
cd $ANDROID_BUILD_TOP

# SELinux 정책 디렉토리 생성
mkdir -p system/sepolicy/apex

# file_contexts 파일 생성
# 파일명은 반드시 "{APEX name}-file_contexts" 형식을 따라야 함
cat > system/sepolicy/apex/com.example.simple-file_contexts << 'EOF'
(/.*)?                                          u:object_r:system_file:s0
/bin(/.*)?                                      u:object_r:system_file:s0
/bin/simple_apex_binary                         u:object_r:system_file:s0
EOF
```

> **file_contexts 자동 탐색 규칙**
> - `Android.bp`에서 `file_contexts` 속성을 **생략**하면, 빌드 시스템이 자동으로 다음 경로를 탐색합니다:
>   ```
>   system/sepolicy/apex/{APEX_NAME}-file_contexts
>   ```
> - 예시: APEX 이름이 `com.example.simple`이면
>   ```
>   system/sepolicy/apex/com.example.simple-file_contexts
>   ```
> - 커스텀 SELinux 타입이 필요한 경우에만 별도 `.te` 파일을 추가합니다 (단순 실습에서는 `system_file` 타입으로 충분).

## 6. 빌드 및 테스트

### 6.1 빌드

```bash
# AOSP 루트 디렉토리에서 실행
cd $ANDROID_BUILD_TOP

# 빌드 환경 설정 (세션당 1회)
source build/envsetup.sh
lunch aosp_car_x86_64-aosp_current-userdebug

# simple_apex 디렉토리로 이동하여 개별 빌드
cd simple_apex
mm

# 또는 AOSP 루트에서 모듈명으로 빌드
# cd $ANDROID_BUILD_TOP
# m com.example.simple

# 빌드 결과 확인
ls -la $OUT/system/apex/com.example.simple.apex

# APEX 내용 확인
deapexer list $OUT/system/apex/com.example.simple.apex
```

### 6.2 에뮬레이터에서 테스트

```bash
# 방법 1: AOSP 빌드 전체를 에뮬레이터로 실행
emulator &

# 방법 2: Cuttlefish 가상 디바이스 사용 (Android 16 권장)
# launch_cvd

# APEX 설치 및 테스트
adb root
adb remount

# APEX 파일 푸시
adb push $OUT/system/apex/com.example.simple.apex /system/apex/

# 재부팅
adb reboot

# 재부팅 후 APEX 활성화 확인
adb shell ls -la /apex/com.example.simple/

# 바이너리 실행 테스트
adb shell /apex/com.example.simple/bin/simple_apex_binary

# APEX 상태 확인 (apexd 로그)
adb logcat -s apexd
```

> **Android 16 변경사항 (테스트 환경)**
> - Google은 Pixel 디바이스 트리를 AOSP에서 제거하는 방향으로 전환했으며, **Cuttlefish** 가상 디바이스를 AOSP 레퍼런스 디바이스로 권장합니다.
> - `adb install --apex` 명령으로 런타임 APEX 설치도 가능합니다 (개발 빌드에 한함).

## 7. Android 14 → 16 마이그레이션 체크리스트

| 항목 | Android 14 | Android 16 |
|------|-----------|-----------|
| API Level | 34 | **36** |
| 코드명 | UpsideDownCake | **Baklava** |
| `min_sdk_version` | `"34"` | `"36"` |
| lunch 형식 | `sdk_car_x86_64` | `aosp_car_x86_64-aosp_current-userdebug` |
| 레퍼런스 디바이스 | Pixel / 에뮬레이터 | **Cuttlefish 권장** |
| AOSP 브랜치 | `android-14.0.0_rXX` | `android-latest-release` 또는 `android-16.0.0_rXX` |
| 커널 | GKI (Linux 5.15/6.1) | **GKI (Linux 6.12)** |
| file_contexts 위치 | `system/sepolicy/apex/` | `system/sepolicy/apex/` **(동일, 변경 없음)** |
| APEX 압축 | `updatable: true`인 경우만 가능 | **동일** (`updatable: false`와 혼용 불가) |
| KeyMint 버전 | 3.0 | **4.0** (APEX moduleHash 필드 추가) |

## 8. 트러블슈팅

### 8.1 `file_contexts: should be under system/sepolicy` 오류

```
error: module "com.example.simple": file_contexts: should be under system/sepolicy,
but found in "simple_apex/"
```

**원인**: `file_contexts` 파일을 APEX 모듈 디렉토리 내부에 배치하거나, `Android.bp`에서 `filegroup`으로 직접 참조한 경우 발생합니다.

**해결**:
```bash
# 1. APEX 디렉토리 내의 file_contexts 삭제
rm -f simple_apex/file_contexts

# 2. Android.bp에서 file_contexts 및 filegroup 관련 내용 모두 제거

# 3. system/sepolicy/apex/ 에 올바른 파일 생성
mkdir -p system/sepolicy/apex
cat > system/sepolicy/apex/com.example.simple-file_contexts << 'EOF'
(/.*)?                                          u:object_r:system_file:s0
/bin(/.*)?                                      u:object_r:system_file:s0
/bin/simple_apex_binary                         u:object_r:system_file:s0
EOF
```

### 8.2 `compressible: do not compress non-updatable APEX` 오류

```
error: module "com.example.simple": compressible: do not compress non-updatable APEX
```

**원인**: `updatable: false`와 `compressible: true`를 동시에 설정한 경우 발생합니다. APEX 압축은 Mainline 업데이트를 통해 `/data` 파티션에 새 버전이 설치되는 APEX에만 의미가 있기 때문입니다.

**해결**: `Android.bp`에서 `compressible: true` 줄을 제거합니다.
```json
apex {
    name: "com.example.simple",
    // ...
    updatable: false,
    // compressible: true,  ← 이 줄 제거
}
```

### 8.3 `lunch` 타겟을 찾을 수 없는 경우
```bash
# 사용 가능한 타겟 목록 확인
lunch --print-all-targets 2>/dev/null || lunch

# Android 16에서 Car 관련 타겟 필터링
lunch --print-all-targets 2>/dev/null | grep car
```

### 8.4 APEX 서명 검증 실패
```bash
# 키 크기 확인 (반드시 4096비트)
openssl rsa -in com.example.simple.pem -text -noout | grep "Private-Key"

# avbpubkey 재생성
avbtool extract_public_key --key com.example.simple.pem --output com.example.simple.avbpubkey
```

### 8.5 커널 버전 불일치 문제
```bash
# Android 16은 GKI 커널 Linux 6.12 기반
# 에뮬레이터 커널 버전 확인
adb shell uname -r

# 필요한 커널 기능 확인
adb shell zcat /proc/config.gz | grep -E "(CONFIG_BLK_DEV_LOOP|CONFIG_DM_VERITY)"
```

### 8.6 SELinux 정책 오류
```bash
# SELinux 거부 로그 확인
adb logcat | grep -i "avc.*denied"

# 개발 중 Permissive 모드로 전환 (디버깅용)
adb shell setenforce 0
```

## 참고 자료

- [AOSP APEX 공식 문서](https://source.android.com/docs/core/ota/apex)
- [Vendor APEX 가이드](https://source.android.com/docs/core/ota/vendor-apex)
- [Android 16 릴리스 노트](https://source.android.com/docs/whatsnew/android-16-release)
- [AOSP 빌드 가이드](https://source.android.com/docs/setup/build/building)
- [Cuttlefish 가상 디바이스](https://source.android.com/docs/setup/create/cuttlefish)
