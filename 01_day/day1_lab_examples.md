# Day 1 실습 예제 모음
## Android Framework 16 심화 교육 — 실습 가이드

> **환경:** Windows 11 + Android Studio + Emulator (API 36)  
> **범위:** Chapter 0~3 (아키텍처, Binder, AIDL, IPC 서비스)

---

## 📋 실습 목차

| # | 실습명 | 난이도 | 시간 | 챕터 |
|---|--------|--------|------|------|
| 1 | 시스템 탐험가 — adb로 Android 내부 들여다보기 | ★☆☆ | 20분 | Ch 0 |
| 2 | Binder 탐정 — 시스템 서비스 추적하기 | ★☆☆ | 20분 | Ch 1 |
| 3 | AIDL 계산기 — 기본 IPC 서비스 구현 | ★★☆ | 45분 | Ch 2 |
| 4 | Binder 생존 게임 — DeathRecipient & 복구 | ★★☆ | 30분 | Ch 2 |
| 5 | 비동기 주식 시세 서비스 — oneway + Callback | ★★★ | 40분 | Ch 2 |
| 6 | Messenger 채팅 앱 — 경량 IPC | ★★☆ | 30분 | Ch 3 |
| 7 | Content Provider 메모장 — 멀티프로세스 데이터 공유 | ★★★ | 45분 | Ch 3 |
| 8 | TransactionTooLargeException 재현 & 해결 | ★★☆ | 20분 | Ch 1 |

---

---

## 실습 1: 시스템 탐험가 — adb로 Android 내부 들여다보기

### 목표
- Android 시스템의 실제 프로세스 구조를 눈으로 확인
- Zygote, System Server, ServiceManager의 존재를 직접 확인
- 시스템 서비스 목록과 Binder 상태를 실시간 확인

### 난이도: ★☆☆ (20분)

### 실습 내용

#### Step 1: 프로세스 구조 확인
```bash
# 에뮬레이터 시작 후 adb 연결 확인
adb devices

# root 권한 획득 (Google APIs 이미지에서 가능)
adb root

# 에뮬레이터에 접속
adb shell

# 전체 프로세스 트리 확인 — Zygote가 앱의 부모임을 확인
ps -A -o pid,ppid,name | head -30

# Zygote 프로세스 찾기
ps -A | grep zygote
# 출력 예시:
# root  1234  1  zygote64
# root  1235  1  zygote

# System Server 찾기 (Zygote의 자식)
ps -A | grep system_server
# PPID가 zygote의 PID와 같은지 확인!

# init 프로세스 확인 (PID 1)
ps -A -o PID,PPID,NAME | awk '$1 == 1'
```

**✅ 확인 포인트:**
- Zygote의 PPID가 1(init)인지 확인
- system_server의 PPID가 Zygote의 PID인지 확인
- 다른 앱 프로세스의 PPID도 Zygote인지 확인 → **모든 앱은 Zygote에서 fork**

#### Step 2: 시스템 서비스 탐색
```bash
# 등록된 시스템 서비스 목록 (100개+)
service list
# 출력 예시:
# 0 sip: [android.net.sip.ISipService]
# 1 phone: [com.android.internal.telephony.ITelephony]
# 2 activity: [android.app.IActivityManager]

# 서비스 수 세기
service list | wc -l

# 특정 서비스 상태 덤프
dumpsys activity | head -50
dumpsys window displays | head -30
dumpsys package com.android.car.settings | head -30
```

#### Step 3: Binder 상태 확인
```bash
# Binder 전체 통계
cat /dev/binderfs/binder_logs/stats | head -20

# system_server의 Binder 정보
SYSPID=$(pidof system_server)
cat /dev/binderfs/binder_logs/proc/$SYSPID | head -30

# Binder 트랜잭션 수 확인
cat /dev/binderfs/binder_logs/stats | grep "BC_TRANSACTION"
```

#### Step 4: 부팅 시퀀스 확인
```bash
# 부팅 로그에서 주요 이벤트 확인
dmesg | grep -i "binder"
logcat -b events -d | grep "boot"

# init.rc 내용 확인 (Zygote 서비스 정의)
cat /system/etc/init/hw/init.zygote64_32.rc
```

### 🎯 결과물
수강생이 작성할 보고서:
1. Zygote PID와 system_server PID 기록
2. 시스템 서비스 총 개수
3. 가장 많은 Binder 트랜잭션을 처리하는 서비스 3개 추정

---

---

## 실습 2: Binder 탐정 — 시스템 서비스 추적하기

### 목표
- 실제 앱이 시스템 서비스를 호출할 때 Binder 트랜잭션이 발생하는 것을 실시간 관찰
- `service call` 명령으로 직접 시스템 서비스를 호출

### 난이도: ★☆☆ (20분)

### 실습 내용

#### Step 1: service call로 직접 서비스 호출
```bash
# clipboard 서비스에 텍스트 복사 (service call 사용)
# IClipboard.aidl의 setPrimaryClip은 transaction code 1
service call clipboard 1

# 현재 디스플레이 정보 확인
dumpsys display | grep "mBaseDisplayInfo"

# 배터리 상태 확인
dumpsys battery
```

#### Step 2: Binder 트랜잭션 실시간 관찰
```bash
# 1. 활성화 확인 (1이 나와야 함)
cat /sys/kernel/tracing/events/binder/binder_transaction/enable
# 0이면 활성화:
echo 1 > /sys/kernel/tracing/events/binder/binder_transaction/enable

# 2. tracing_on 확인 (1이어야 트레이싱 동작)
cat /sys/kernel/tracing/tracing_on
# 0이면 활성화:
echo 1 > /sys/kernel/tracing/tracing_on

# 3. 같은 경로에서 읽기
cat /sys/kernel/tracing/trace_pipe

# 에뮬레이터에서 앱을 열거나 설정을 변경하면
# 트랜잭션 로그가 실시간으로 출력됨!
# 0으로 세팅 : 트레이싱 멈춤
echo 0 > /sys/kernel/tracing/tracing_on

# 완료 후 비활성화
echo 0 > /sys/kernel/tracing/events/binder/binder_transaction/enable
```

#### Step 3: 특정 앱의 Binder 사용량 추적
```bash
# Settings 앱 PID 찾기
SETPID=$(pidof com.android.car.settings)

# 해당 프로세스의 Binder 상태
cat /dev/binderfs/binder_logs/proc/$SETPID

# Binder Thread 수 확인
cat /dev/binderfs/binder_logs/proc/$SETPID | grep "thread"
```

### 🎯 결과물
- 에뮬레이터에서 Settings 앱을 열 때 발생하는 Binder 트랜잭션 수 기록
- 어떤 서비스가 호출되는지 추정

---

---

## 실습 3: AIDL 계산기 — 기본 IPC 서비스 구현

### 목표
- AIDL 인터페이스를 정의하고 서비스/클라이언트를 구현
- Binder IPC의 동작을 직접 체험
- clearCallingIdentity 패턴을 적용

### 난이도: ★★☆ (45분)

### Step 1: 프로젝트 생성
Android Studio → New Project → **Empty Views Activity**
- Name: `BinderLab`
- Package: `com.example.binderlab`
- Min SDK: API 36

### Step 2: AIDL 인터페이스 정의

아래내용 추가
build.gradle.kts(Module :app)
```java
    buildFeatures {
        aidl = true
    }
```

**파일:** `app/src/main/aidl/com/example/binderlab/ICalculatorService.aidl`
```java
package com.example.binderlab;

interface ICalculatorService {
    // 기본 동기 호출
    int add(int a, int b);
    int subtract(int a, int b);
    int multiply(int a, int b);
    double divide(int a, int b);
    
    // 호출자 정보 반환 (Binder 보안 확인용)
    String getCallerInfo();
    
    // 계산 이력 조회
    List<String> getHistory();
}
```

빌드(Build → Make Project)하여 AIDL 자동 생성 확인.

### Step 3: 서비스 구현

**파일:** `app/src/main/java/com/example/binderlab/CalculatorService.java`
```java
package com.example.binderlab;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;
import android.os.Process;
import android.os.RemoteException;
import android.util.Log;
import java.util.ArrayList;
import java.util.List;

public class CalculatorService extends Service {
    private static final String TAG = "CalcService";
    
    // ★ 계산 이력 (스레드 안전하게 관리)
    private final List<String> history = new ArrayList<>();
    private final Object lock = new Object();
    
    private final ICalculatorService.Stub binder = 
        new ICalculatorService.Stub() {
        
        @Override
        public int add(int a, int b) throws RemoteException {
            // ★ 호출자 정보 로깅
            int callingUid = Binder.getCallingUid();
            int callingPid = Binder.getCallingPid();
            Log.d(TAG, "add() called by UID=" + callingUid 
                + " PID=" + callingPid);
            
            int result = a + b;
            addHistory(a + " + " + b + " = " + result);
            return result;
        }
        
        @Override
        public int subtract(int a, int b) throws RemoteException {
            int result = a - b;
            addHistory(a + " - " + b + " = " + result);
            return result;
        }
        
        @Override
        public int multiply(int a, int b) throws RemoteException {
            int result = a * b;
            addHistory(a + " × " + b + " = " + result);
            return result;
        }
        
        @Override
        public double divide(int a, int b) throws RemoteException {
            if (b == 0) throw new ArithmeticException("Division by zero");
            double result = (double) a / b;
            addHistory(a + " ÷ " + b + " = " + result);
            return result;
        }
        
        @Override
        public String getCallerInfo() throws RemoteException {
            // ★ Binder 보안 — 호출자 UID/PID 반환
            int uid = Binder.getCallingUid();
            int pid = Binder.getCallingPid();
            int myUid = Process.myUid();
            int myPid = Process.myPid();
            
            return String.format(
                "Caller: UID=%d, PID=%d | Service: UID=%d, PID=%d | Same process: %s",
                uid, pid, myUid, myPid, (pid == myPid) ? "YES" : "NO"
            );
        }
        
        @Override
        public List<String> getHistory() throws RemoteException {
            // ★ clearCallingIdentity 패턴 예시
            // (실제 시나리오: 이력을 DB에서 조회할 때 시스템 권한 필요)
            long token = Binder.clearCallingIdentity();
            try {
                synchronized (lock) {
                    return new ArrayList<>(history);
                }
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
    };
    
    private void addHistory(String entry) {
        synchronized (lock) {
            history.add(entry);
            if (history.size() > 50) history.remove(0);
        }
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "Service bound!");
        return binder;
    }
    
    @Override
    public void onDestroy() {
        Log.d(TAG, "Service destroyed!");
        super.onDestroy();
    }
}
```

### Step 4: AndroidManifest 등록

```xml
<!-- AndroidManifest.xml에 추가 -->
<service
    android:name=".CalculatorService"
    android:exported="true"
    android:process=":calc_remote">
    <intent-filter>
        <action android:name="com.example.binderlab.CALCULATOR" />
    </intent-filter>
</service>
```

> **핵심:** `android:process=":calc_remote"` — 서비스를 **별도 프로세스**에서 실행하여 실제 Binder IPC 발생을 강제합니다.

### Step 5: 클라이언트 Activity

**파일:** `MainActivity.java`
```java
package com.example.binderlab;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.os.RemoteException;
import android.util.Log;
import android.widget.Button;
import android.widget.EditText;
import android.widget.TextView;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;
import java.util.List;

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "CalcClient";
    private ICalculatorService service;
    private boolean bound = false;
    private TextView resultText;
    private TextView infoText;
    
    // ★ ServiceConnection — 바인딩 콜백
    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder binder) {
            // ★ 핵심: IBinder → AIDL Proxy 변환
            service = ICalculatorService.Stub.asInterface(binder);
            bound = true;
            Log.d(TAG, "Service connected!");
            
            // 호출자 정보 확인
            try {
                infoText.setText(service.getCallerInfo());
            } catch (RemoteException e) {
                Log.e(TAG, "Failed to get caller info", e);
            }
            
            // ★ DeathRecipient 등록
            try {
                binder.linkToDeath(deathRecipient, 0);
            } catch (RemoteException e) {
                Log.e(TAG, "linkToDeath failed", e);
            }
        }
        
        @Override
        public void onServiceDisconnected(ComponentName name) {
            service = null;
            bound = false;
            Log.d(TAG, "Service disconnected!");
            infoText.setText("⚠ Service disconnected!");
        }
    };
    
    // ★ 서비스 사망 감지
    private final IBinder.DeathRecipient deathRecipient = () -> {
        Log.w(TAG, "☠ Service process died!");
        runOnUiThread(() -> {
            bound = false;
            service = null;
            infoText.setText("☠ Service DIED! Rebinding...");
            // 자동 재연결 시도
            bindCalcService();
        });
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        resultText = findViewById(R.id.resultText);
        infoText = findViewById(R.id.infoText);
        EditText numA = findViewById(R.id.numA);
        EditText numB = findViewById(R.id.numB);
        
        // 연산 버튼들
        findViewById(R.id.btnAdd).setOnClickListener(v -> calculate(numA, numB, "add"));
        findViewById(R.id.btnSub).setOnClickListener(v -> calculate(numA, numB, "sub"));
        findViewById(R.id.btnMul).setOnClickListener(v -> calculate(numA, numB, "mul"));
        findViewById(R.id.btnDiv).setOnClickListener(v -> calculate(numA, numB, "div"));
        
        // 이력 버튼
        findViewById(R.id.btnHistory).setOnClickListener(v -> showHistory());
        
        // 서비스 바인딩
        bindCalcService();
    }
    
    private void bindCalcService() {
        Intent intent = new Intent("com.example.binderlab.CALCULATOR");
        intent.setPackage("com.example.binderlab");
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }
    
    private void calculate(EditText ea, EditText eb, String op) {
        if (!bound || service == null) {
            Toast.makeText(this, "Service not bound!", Toast.LENGTH_SHORT).show();
            return;
        }
        try {
            int a = Integer.parseInt(ea.getText().toString());
            int b = Integer.parseInt(eb.getText().toString());
            String result;
            switch (op) {
                case "add": result = a + " + " + b + " = " + service.add(a, b); break;
                case "sub": result = a + " - " + b + " = " + service.subtract(a, b); break;
                case "mul": result = a + " × " + b + " = " + service.multiply(a, b); break;
                case "div": result = a + " ÷ " + b + " = " + service.divide(a, b); break;
                default: result = "Unknown op";
            }
            resultText.setText(result);
        } catch (RemoteException e) {
            resultText.setText("❌ RemoteException: " + e.getMessage());
        } catch (Exception e) {
            resultText.setText("❌ Error: " + e.getMessage());
        }
    }
    
    private void showHistory() {
        if (!bound || service == null) return;
        try {
            List<String> history = service.getHistory();
            StringBuilder sb = new StringBuilder("📜 계산 이력:\n");
            for (String h : history) sb.append("  • ").append(h).append("\n");
            resultText.setText(sb.toString());
        } catch (RemoteException e) {
            resultText.setText("❌ Failed to get history");
        }
    }
    
    @Override
    protected void onDestroy() {
        if (bound) unbindService(connection);
        super.onDestroy();
    }
}
```

### Step 6: 레이아웃

**파일:** `res/layout/activity_main.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="24dp">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="🧮 Binder Calculator Lab"
        android:textSize="22sp"
        android:textStyle="bold"
        android:layout_marginBottom="8dp"/>

    <TextView
        android:id="@+id/infoText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Connecting..."
        android:textSize="11sp"
        android:textColor="#666"
        android:background="#F0F0F0"
        android:padding="8dp"
        android:layout_marginBottom="16dp"/>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="12dp">
        <EditText android:id="@+id/numA" android:layout_width="0dp" android:layout_weight="1"
            android:layout_height="wrap_content" android:inputType="number" android:hint="A" android:text="42"/>
        <EditText android:id="@+id/numB" android:layout_width="0dp" android:layout_weight="1"
            android:layout_height="wrap_content" android:inputType="number" android:hint="B" android:text="7" android:layout_marginStart="8dp"/>
    </LinearLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="12dp">
        <Button android:id="@+id/btnAdd" android:layout_width="0dp" android:layout_weight="1"
            android:layout_height="wrap_content" android:text="+" />
        <Button android:id="@+id/btnSub" android:layout_width="0dp" android:layout_weight="1"
            android:layout_height="wrap_content" android:text="-" />
        <Button android:id="@+id/btnMul" android:layout_width="0dp" android:layout_weight="1"
            android:layout_height="wrap_content" android:text="×" />
        <Button android:id="@+id/btnDiv" android:layout_width="0dp" android:layout_weight="1"
            android:layout_height="wrap_content" android:text="÷" />
    </LinearLayout>

    <Button android:id="@+id/btnHistory" android:layout_width="match_parent"
        android:layout_height="wrap_content" android:text="📜 계산 이력 보기"
        android:layout_marginBottom="16dp"/>

    <ScrollView android:layout_width="match_parent" android:layout_height="0dp" android:layout_weight="1">
        <TextView android:id="@+id/resultText" android:layout_width="match_parent"
            android:layout_height="wrap_content" android:text="결과가 여기에 표시됩니다."
            android:textSize="16sp" android:padding="12dp" android:background="#FAFAFA"/>
    </ScrollView>
</LinearLayout>
```

### Step 7: 검증

```bash
# 실행 후 프로세스 확인 — 메인 앱과 서비스가 별도 프로세스인지
ps -A | grep binderlab
# 출력 예시:
# u0_a150  12345  1000  com.example.binderlab
# u0_a150  12346  1000  com.example.binderlab:calc_remote

# 서비스 바인딩 확인
dumpsys activity services | grep -A 5 "CalculatorService"

# Logcat에서 Binder 호출 로그 확인
logcat -s CalcService CalcClient
```

### 🎯 핵심 학습 포인트
1. `getCallerInfo()` 결과에서 **"Same process: NO"** 확인 → 실제 IPC 발생
2. `android:process` 제거 후 재실행 → **"Same process: YES"** → IPC 미발생 (직접 호출)
3. 계산 시 Logcat에서 **UID/PID 로그** 확인

---

---

## 실습 4: Binder 생존 게임 — DeathRecipient & 복구

### 목표
- 서비스 프로세스가 죽었을 때의 동작을 체험
- DeathRecipient 콜백을 통한 자동 복구 패턴 학습

### 난이도: ★★☆ (30분)

### 실습 내용

실습 3의 프로젝트에서 계속 진행합니다.

#### Step 1: 서비스 프로세스 강제 종료
```bash
# 서비스 프로세스 PID 확인
ps -A | grep calc_remote

# 강제 종료!
kill -9 <서비스_PID>
```

#### Step 2: 관찰할 것
1. 앱 화면에 **"☠ Service DIED! Rebinding..."** 메시지가 나타나는지 확인
2. Logcat에서 `"☠ Service process died!"` 로그 확인
3. 자동 재바인딩 후 다시 계산이 작동하는지 확인
4. `getCallerInfo()`의 PID가 바뀌었는지 확인 (새 프로세스이므로 PID 변경)

#### Step 3: 반복 Kill 테스트
```bash
# 3초마다 서비스 프로세스 kill (스트레스 테스트)
for i in 1 2 3 4 5; do
  PID=$(pidof com.example.binderlab:calc_remote)
  if [ -n "$PID" ]; then
    echo "Killing PID $PID (attempt $i)"
    kill -9 $PID
  fi
  sleep 3
done
```

앱이 매번 자동 복구되는지 확인합니다.

### 🎯 핵심 학습 포인트
1. **DeathRecipient**: Binder Thread에서 호출됨 → UI 업데이트는 `runOnUiThread()` 필수
2. **BIND_AUTO_CREATE**: 서비스 프로세스가 죽으면 시스템이 자동으로 재생성
3. **onServiceDisconnected vs DeathRecipient**: 둘 다 호출되지만 DeathRecipient가 더 빠름

---

---

## 실습 5: 비동기 주식 시세 서비스 — oneway + Callback

> **목표:** 동기 vs 비동기 Binder IPC의 차이를 직접 체험  
> **환경:** Windows 11 + Android Studio + Emulator (API 36)  
> **소요시간:** 약 40분

---

## 📋 이 실습에서 배우는 것

| 개념 | 체험 |
|------|------|
| 동기 IPC (`getPrice`) | 서버가 2초 블로킹 → 클라이언트도 2초 대기 |
| 비동기 IPC (`subscribe`, oneway) | 서버에 요청 후 **0ms**에 즉시 반환 |
| Callback 패턴 | 서비스 → 클라이언트 역방향 Binder IPC |
| RemoteCallbackList | 클라이언트 사망 시 자동 정리 |
| DeathRecipient | 서비스 프로세스 사망 감지 |

---

## Step 1: 프로젝트 생성

### 1-1. Android Studio에서 새 프로젝트

```
File → New → New Project
```

- 템플릿: **Empty Views Activity** 선택
- Next 클릭

### 1-2. 프로젝트 설정

| 항목 | 값 |
|------|-----|
| Name | `StockLab` |
| Package name | `com.example.stocklab` |
| Save location | 원하는 경로 |
| Language | **Java** |
| Minimum SDK | **API 28** 이상 |
| Build configuration language | Kotlin DSL (기본값) |

- **Finish** 클릭

### 1-3. 프로젝트 생성 완료 대기

Android Studio 하단 상태바에 "Gradle sync" 완료될 때까지 대기합니다.  
완료되면 `MainActivity.java`와 `activity_main.xml`이 자동 생성되어 있습니다.

---

## Step 2: AIDL 빌드 기능 활성화

### 2-1. build.gradle.kts 수정

**파일:** `app/build.gradle.kts`  
`android { }` 블록 안에 `buildFeatures`를 추가합니다:

```kotlin
android {
    namespace = "com.example.stocklab"
    compileSdk = 36

    // ★ 이 블록 추가!
    buildFeatures {
        aidl = true
    }

    defaultConfig {
        applicationId = "com.example.stocklab"
        minSdk = 28
        targetSdk = 36
        versionCode = 1
        versionName = "1.0"
    }

    // ... 나머지는 그대로 ...
}
```

### 2-2. Gradle Sync

파일 수정 후 상단에 노란색 바가 나타나면 **Sync Now** 클릭.  
또는 메뉴: `File → Sync Project with Gradle Files`

---

## Step 3: AIDL 파일 생성 (인터페이스 정의)

### 3-1. AIDL 디렉토리 생성

프로젝트 뷰(좌측)를 **Android** 모드로 설정한 상태에서:

```
app → 우클릭 → New → Folder → AIDL Folder
```

또는 수동으로 디렉토리를 생성합니다:

```
app/src/main/aidl/com/example/stocklab/
```

> ⚠ **패키지 경로가 Java 소스와 동일**해야 합니다 (`com/example/stocklab`)

### 3-2. IStockCallback.aidl 생성

```
app/src/main/aidl/com/example/stocklab/ → 우클릭 → New → File
파일명: IStockCallback.aidl
```

**파일 내용:**

```java
// IStockCallback.aidl
package com.example.stocklab;

// 서비스 → 클라이언트 방향의 콜백 인터페이스
// oneway: 비동기 (서비스가 콜백 호출 시 블로킹되지 않음)
interface IStockCallback {
    oneway void onPriceUpdate(String symbol, double price, long timestamp);
    oneway void onError(String symbol, String message);
}
```

### 3-3. IStockService.aidl 생성

같은 디렉토리에 `IStockService.aidl` 생성:

```java
// IStockService.aidl
package com.example.stocklab;

import com.example.stocklab.IStockCallback;

interface IStockService {
    // 동기 메서드: 결과가 올 때까지 클라이언트 블로킹 (2초 시뮬레이션)
    double getPrice(String symbol);

    // 비동기 메서드 (oneway): 즉시 반환, 결과는 콜백으로 나중에 수신
    oneway void subscribe(String symbol, IStockCallback callback);
    oneway void unsubscribe(String symbol);
}
```

### 3-4. 빌드하여 Stub/Proxy 자동 생성

```
Build → Make Project (Ctrl+F9)
```

하단 Build 탭에 에러 없이 `BUILD SUCCESSFUL`이 나와야 합니다.

### 3-5. 자동 생성 확인

빌드 성공 후, Java 파일 아무 곳에서 다음을 타이핑하여 자동완성이 되는지 확인:

```java
IStockService.Stub  // ← 자동완성 나오면 성공!
```

> 자동완성이 안 되면 빌드가 실패한 것. Build 탭의 에러 메시지를 확인하세요.

---

## Step 4: 레이아웃 작성 (activity_main.xml)

### 4-1. 기존 레이아웃 교체

**파일:** `app/src/main/res/layout/activity_main.xml`  
기존 내용을 **전부 삭제**하고 아래로 교체합니다:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="24dp">

    <!-- 제목 -->
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="📈 Stock Service Lab"
        android:textSize="22sp"
        android:textStyle="bold"
        android:layout_marginBottom="4dp"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="oneway + Callback 비동기 IPC 실습"
        android:textSize="13sp"
        android:textColor="#888"
        android:layout_marginBottom="12dp"/>

    <!-- 연결 상태 표시 -->
    <TextView
        android:id="@+id/statusText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="⏳ StockService 연결 중..."
        android:textSize="12sp"
        android:textColor="#555"
        android:background="#F0F0F0"
        android:padding="10dp"
        android:layout_marginBottom="16dp"/>

    <!-- 버튼 행 1: 동기 vs 비동기 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="8dp">

        <Button
            android:id="@+id/btnSync"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="동기 호출\n(2초 블로킹)"
            android:textSize="12sp"
            android:textAllCaps="false"/>

        <Button
            android:id="@+id/btnSubscribe"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="AAPL 구독\n(즉시 반환)"
            android:textSize="12sp"
            android:textAllCaps="false"
            android:layout_marginStart="8dp"/>
    </LinearLayout>

    <!-- 버튼 행 2: 전체 구독 / 로그 초기화 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="16dp">

        <Button
            android:id="@+id/btnSubscribeAll"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="전체 구독\nAAPL+GOOGL+MSFT"
            android:textSize="11sp"
            android:textAllCaps="false"/>

        <Button
            android:id="@+id/btnClear"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="로그 지우기"
            android:textSize="12sp"
            android:textAllCaps="false"
            android:layout_marginStart="8dp"/>
    </LinearLayout>

    <!-- 로그 출력 영역 -->
    <ScrollView
        android:id="@+id/scrollView"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1">

        <TextView
            android:id="@+id/logText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="로그가 여기에 표시됩니다.\n"
            android:textSize="12sp"
            android:fontFamily="monospace"
            android:padding="12dp"
            android:background="#FAFAFA"
            android:textColor="#333"/>
    </ScrollView>
</LinearLayout>
```

---

## Step 5: 서비스 구현 (StockService.java)

### 5-1. 파일 생성

```
app/src/main/java/com/example/stocklab/ → 우클릭 → New → Java Class
Name: StockService
```

### 5-2. 소스 코드

**파일:** `StockService.java`

```java
package com.example.stocklab;

import android.app.Service;
import android.content.Intent;
import android.os.Handler;
import android.os.HandlerThread;
import android.os.IBinder;
import android.os.RemoteCallbackList;
import android.os.RemoteException;
import android.util.Log;

import java.util.HashMap;
import java.util.Map;
import java.util.Random;

/**
 * 주식 시세 서비스 — 별도 프로세스에서 실행
 * 
 * 동기 메서드(getPrice): 2초 지연 후 가격 반환
 * 비동기 메서드(subscribe): 즉시 반환, 1초마다 콜백으로 가격 전송
 */
public class StockService extends Service {
    private static final String TAG = "StockService";

    // ★ RemoteCallbackList: 클라이언트가 죽으면 자동으로 콜백 제거
    private final Map<String, RemoteCallbackList<IStockCallback>> subscribers
            = new HashMap<>();

    // 종목별 현재 가격
    private final Map<String, Double> prices = new HashMap<>();
    private final Random random = new Random();

    // 가격 업데이트 워커 스레드
    private HandlerThread workerThread;
    private Handler workerHandler;
    private boolean running = true;

    // ★ AIDL Stub 구현 — 클라이언트의 요청을 처리
    private final IStockService.Stub binder = new IStockService.Stub() {

        @Override
        public double getPrice(String symbol) throws RemoteException {
            // ★ 동기 호출: 네트워크 요청을 시뮬레이션 (2초 블로킹!)
            Log.d(TAG, "getPrice(" + symbol + ") — SYNC call, blocking 2 seconds...");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            double price = prices.getOrDefault(symbol, 0.0);
            Log.d(TAG, "getPrice(" + symbol + ") — returning $" + price);
            return price;
        }

        @Override
        public void subscribe(String symbol, IStockCallback callback) {
            // ★ 비동기(oneway): 이 메서드는 즉시 반환됨!
            Log.d(TAG, "subscribe(" + symbol + ") — ASYNC, returning immediately!");
            synchronized (subscribers) {
                if (!subscribers.containsKey(symbol)) {
                    subscribers.put(symbol, new RemoteCallbackList<>());
                }
                subscribers.get(symbol).register(callback);
            }
            Log.d(TAG, "subscribe(" + symbol + ") — callback registered");
        }

        @Override
        public void unsubscribe(String symbol) {
            Log.d(TAG, "unsubscribe(" + symbol + ")");
            // 간단한 구현: 전체 콜백 리스트 정리는 생략
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        Log.d(TAG, "══════════════════════════════════");
        Log.d(TAG, "StockService CREATED (PID: " + android.os.Process.myPid() + ")");
        Log.d(TAG, "══════════════════════════════════");

        // 초기 주가 설정
        prices.put("AAPL", 185.50);
        prices.put("GOOGL", 140.20);
        prices.put("MSFT", 420.75);

        // 가격 업데이트 워커 스레드 시작
        workerThread = new HandlerThread("StockUpdater");
        workerThread.start();
        workerHandler = new Handler(workerThread.getLooper());
        workerHandler.post(this::updatePrices);
    }

    /**
     * 1초마다 가격을 랜덤 변동시키고 구독자에게 알림
     */
    private void updatePrices() {
        if (!running) return;

        for (Map.Entry<String, Double> entry : prices.entrySet()) {
            // ±$1.00 범위에서 랜덤 변동
            double change = (random.nextDouble() - 0.5) * 2.0;
            double newPrice = Math.max(1.0, entry.getValue() + change);
            newPrice = Math.round(newPrice * 100.0) / 100.0;
            entry.setValue(newPrice);

            // 구독자에게 콜백 전송
            notifySubscribers(entry.getKey(), newPrice);
        }

        // 1초 후 다시 실행
        workerHandler.postDelayed(this::updatePrices, 1000);
    }

    /**
     * ★ RemoteCallbackList 사용법
     * beginBroadcast() → getBroadcastItem(i) → finishBroadcast()
     * 반드시 try-finally로 finishBroadcast()를 보장해야 함!
     */
    private void notifySubscribers(String symbol, double price) {
        synchronized (subscribers) {
            RemoteCallbackList<IStockCallback> callbacks = subscribers.get(symbol);
            if (callbacks == null) return;

            int count = callbacks.beginBroadcast();
            try {
                for (int i = 0; i < count; i++) {
                    try {
                        callbacks.getBroadcastItem(i).onPriceUpdate(
                                symbol, price, System.currentTimeMillis()
                        );
                    } catch (RemoteException e) {
                        // 클라이언트 프로세스가 죽음
                        // RemoteCallbackList가 자동으로 정리해줌
                        Log.w(TAG, "Client dead for " + symbol + ", auto-removing");
                    }
                }
            } finally {
                callbacks.finishBroadcast();
            }
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "onBind() — client connected!");
        return binder;
    }

    @Override
    public boolean onUnbind(Intent intent) {
        Log.d(TAG, "onUnbind() — all clients disconnected");
        return super.onUnbind(intent);
    }

    @Override
    public void onDestroy() {
        Log.d(TAG, "StockService DESTROYED");
        running = false;
        workerThread.quitSafely();
        super.onDestroy();
    }
}
```

---

## Step 6: MainActivity.java 수정

### 6-1. 기존 내용 전부 교체

**파일:** `app/src/main/java/com/example/stocklab/MainActivity.java`

```java
package com.example.stocklab;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.os.RemoteException;
import android.util.Log;
import android.widget.Button;
import android.widget.ScrollView;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "StockClient";

    private IStockService stockService;
    private boolean bound = false;

    private TextView logText;
    private TextView statusText;
    private ScrollView scrollView;

    // ==============================
    // ServiceConnection — 서비스 바인딩 콜백
    // ==============================
    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // ★ IBinder → AIDL Proxy 변환
            stockService = IStockService.Stub.asInterface(service);
            bound = true;

            int myPid = android.os.Process.myPid();
            statusText.setText("✅ StockService 연결됨 (Client PID: " + myPid + ")");
            appendLog("✅ 서비스 연결 완료!");
            appendLog("   Client PID: " + myPid);

            // ★ DeathRecipient 등록 — 서비스 프로세스 사망 감지
            try {
                service.linkToDeath(deathRecipient, 0);
                appendLog("   DeathRecipient 등록됨");
            } catch (RemoteException e) {
                Log.e(TAG, "linkToDeath failed", e);
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            stockService = null;
            bound = false;
            statusText.setText("❌ Service disconnected (Main Thread 콜백)");
            appendLog("❌ onServiceDisconnected() 호출됨 (Main Thread)");
        }
    };

    // ==============================
    // DeathRecipient — Binder Thread에서 호출됨 (더 빠름!)
    // ==============================
    private final IBinder.DeathRecipient deathRecipient = () -> {
        // ★ 이 콜백은 Binder Thread에서 실행됨!
        runOnUiThread(() -> {
            bound = false;
            stockService = null;
            statusText.setText("☠ Service DIED! (DeathRecipient — Binder Thread)");
            appendLog("☠ binderDied() 호출됨 (Binder Thread — 더 빠름!)");
            appendLog("   → 서비스 자동 재시작 대기 중...");
        });
    };

    // ==============================
    // 비동기 콜백 — 서비스에서 가격 업데이트 수신
    // ==============================
    private final IStockCallback.Stub stockCallback = new IStockCallback.Stub() {
        @Override
        public void onPriceUpdate(String symbol, double price, long timestamp) {
            // ★ 이 메서드는 Binder Thread에서 호출됨!
            // UI 업데이트는 반드시 runOnUiThread()
            runOnUiThread(() -> {
                String time = new SimpleDateFormat("HH:mm:ss.SSS", Locale.getDefault())
                        .format(new Date(timestamp));
                appendLog(String.format("  [%s] %s: $%.2f", time, symbol, price));
            });
        }

        @Override
        public void onError(String symbol, String message) {
            runOnUiThread(() -> appendLog("  ❌ ERROR [" + symbol + "]: " + message));
        }
    };

    // ==============================
    // Activity 생명주기
    // ==============================
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // View 바인딩
        logText = findViewById(R.id.logText);
        statusText = findViewById(R.id.statusText);
        scrollView = findViewById(R.id.scrollView);

        Button btnSync = findViewById(R.id.btnSync);
        Button btnSubscribe = findViewById(R.id.btnSubscribe);
        Button btnSubscribeAll = findViewById(R.id.btnSubscribeAll);
        Button btnClear = findViewById(R.id.btnClear);

        // 버튼 이벤트 연결
        btnSync.setOnClickListener(v -> testSyncCall());
        btnSubscribe.setOnClickListener(v -> testAsyncSubscribe("AAPL"));
        btnSubscribeAll.setOnClickListener(v -> {
            testAsyncSubscribe("AAPL");
            testAsyncSubscribe("GOOGL");
            testAsyncSubscribe("MSFT");
        });
        btnClear.setOnClickListener(v -> {
            logText.setText("로그 초기화됨\n");
        });

        // 서비스 바인딩 시작
        appendLog("═══ Stock Service Lab 시작 ═══");
        appendLog("서비스 바인딩 시도...");
        bindStockService();
    }

    @Override
    protected void onDestroy() {
        if (bound) {
            unbindService(connection);
            appendLog("서비스 언바인딩");
        }
        super.onDestroy();
    }

    // ==============================
    // 서비스 바인딩
    // ==============================
    private void bindStockService() {
        Intent intent = new Intent("com.example.stocklab.STOCK_SERVICE");
        intent.setPackage("com.example.stocklab");
        boolean result = bindService(intent, connection, Context.BIND_AUTO_CREATE);
        appendLog("bindService() 반환: " + result);
    }

    // ==============================
    // ★ 동기 호출 테스트
    // ==============================
    private void testSyncCall() {
        if (!bound || stockService == null) {
            appendLog("❌ 서비스 미연결!");
            return;
        }

        appendLog("");
        appendLog("═══ 동기 호출 테스트 (getPrice) ═══");
        appendLog("⏳ 서버에서 2초 블로킹 예상...");

        // ★ 반드시 별도 스레드에서! (UI 스레드에서 하면 ANR)
        new Thread(() -> {
            try {
                long start = System.currentTimeMillis();

                // ★ 이 호출은 서버가 응답할 때까지 블로킹됨!
                double price = stockService.getPrice("AAPL");

                long elapsed = System.currentTimeMillis() - start;

                runOnUiThread(() -> {
                    appendLog(String.format("✅ 결과: AAPL = $%.2f", price));
                    appendLog(String.format("⏱ 소요시간: %d ms (블로킹!)", elapsed));
                    appendLog("─────────────────────────");
                });
            } catch (RemoteException e) {
                runOnUiThread(() -> appendLog("❌ RemoteException: " + e.getMessage()));
            }
        }, "SyncCallThread").start();

        // 이 줄은 즉시 실행됨 (위의 호출은 별도 스레드)
        appendLog("(UI 스레드는 즉시 반환됨 — 별도 스레드에서 대기 중)");
    }

    // ==============================
    // ★ 비동기 구독 테스트
    // ==============================
    private void testAsyncSubscribe(String symbol) {
        if (!bound || stockService == null) {
            appendLog("❌ 서비스 미연결!");
            return;
        }

        try {
            long start = System.currentTimeMillis();

            // ★ oneway 메서드: 즉시 반환! 결과는 콜백으로 나중에 수신
            stockService.subscribe(symbol, stockCallback);

            long elapsed = System.currentTimeMillis() - start;

            appendLog(String.format("📡 %s 구독 완료! (소요: %d ms — 즉시!)", symbol, elapsed));
            appendLog("   → 1초마다 콜백으로 가격 수신 시작...");
        } catch (RemoteException e) {
            appendLog("❌ 구독 실패: " + e.getMessage());
        }
    }

    // ==============================
    // 로그 출력 헬퍼
    // ==============================
    private void appendLog(String msg) {
        logText.append(msg + "\n");
        // 스크롤을 맨 아래로
        scrollView.post(() -> scrollView.fullScroll(ScrollView.FOCUS_DOWN));
    }
}
```

---

## Step 7: AndroidManifest.xml 수정

**파일:** `app/src/main/AndroidManifest.xml`

`<application>` 태그 안에 **StockService**를 추가합니다:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="StockLab"
        android:supportsRtl="true"
        android:theme="@style/Theme.StockLab"
        tools:targetApi="36">

        <!-- ★ MainActivity — Launcher에서 실행 -->
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- ★ StockService — 별도 프로세스에서 실행! -->
        <service
            android:name=".StockService"
            android:exported="true"
            android:process=":stock_remote">
            <intent-filter>
                <action android:name="com.example.stocklab.STOCK_SERVICE" />
            </intent-filter>
        </service>

    </application>
</manifest>
```

> ⚠ **`android:process=":stock_remote"`** — 서비스가 별도 프로세스에서 실행됩니다. 이것이 있어야 **실제 Binder IPC**가 발생합니다. 없으면 같은 프로세스 내 직접 호출이 되어 IPC를 체험할 수 없습니다.

---

## Step 8: 최종 파일 구조 확인

프로젝트 뷰를 **Project** 모드로 변경하여 확인합니다:

```
app/src/main/
├── aidl/
│   └── com/example/stocklab/
│       ├── IStockCallback.aidl      ← 콜백 인터페이스
│       └── IStockService.aidl       ← 서비스 인터페이스
├── java/
│   └── com/example/stocklab/
│       ├── MainActivity.java        ← 클라이언트 (UI)
│       └── StockService.java        ← 서비스 (별도 프로세스)
├── res/
│   └── layout/
│       └── activity_main.xml        ← UI 레이아웃
└── AndroidManifest.xml              ← 서비스 등록
```

총 **6개 파일**을 생성/수정했습니다.

---

## Step 9: 빌드 및 실행

### 9-1. 빌드

```
Build → Make Project (Ctrl+F9)
```

에러가 없어야 합니다. 흔한 에러와 해결:

| 에러 | 원인 | 해결 |
|------|------|------|
| `cannot find symbol: IStockService` | AIDL 빌드 안 됨 | `Build → Clean + Rebuild` |
| `cannot find symbol: IStockCallback` | AIDL 패키지 경로 불일치 | `aidl/` 폴더 패키지 구조 확인 |
| `cannot find symbol: List` | import 누락 | `import java.util.List;` 추가 |
| `AIDL File requires buildFeatures.aidl` | gradle 설정 누락 | Step 2 확인 |

### 9-2. 에뮬레이터에서 실행

```
Run → Run 'app' (Shift+F10)
```

에뮬레이터를 선택하고 실행합니다.

---

## Step 10: 테스트 시나리오

앱이 실행되면 다음 순서로 테스트합니다:

### 테스트 1: 연결 확인

앱 시작 시 자동으로 서비스에 바인딩됩니다. 로그에서 확인:

```
═══ Stock Service Lab 시작 ═══
서비스 바인딩 시도...
bindService() 반환: true
✅ 서비스 연결 완료!
   Client PID: 12345
   DeathRecipient 등록됨
```

### 테스트 2: 동기 호출 (2초 블로킹)

**"동기 호출 (2초 블로킹)"** 버튼을 누릅니다.

```
═══ 동기 호출 테스트 (getPrice) ═══
⏳ 서버에서 2초 블로킹 예상...
(UI 스레드는 즉시 반환됨 — 별도 스레드에서 대기 중)

... 2초 대기 ...

✅ 결과: AAPL = $185.50
⏱ 소요시간: 2005 ms (블로킹!)
─────────────────────────
```

> **관찰:** 동기 호출은 서버가 응답할 때까지 **클라이언트 스레드가 2초간 블로킹**됩니다.

### 테스트 3: 비동기 구독 (즉시 반환)

**"AAPL 구독 (즉시 반환)"** 버튼을 누릅니다.

```
📡 AAPL 구독 완료! (소요: 0 ms — 즉시!)
   → 1초마다 콜백으로 가격 수신 시작...

  [14:30:01.123] AAPL: $186.23
  [14:30:02.125] AAPL: $185.89
  [14:30:03.127] AAPL: $186.45
  [14:30:04.129] AAPL: $187.01
  ...
```

> **관찰:** `oneway` 메서드이므로 **0ms에 즉시 반환**되고, 가격은 **콜백으로 1초마다** 자동 수신됩니다.

### 테스트 4: 전체 구독

**"전체 구독 AAPL+GOOGL+MSFT"** 버튼을 누릅니다.

```
📡 AAPL 구독 완료! (소요: 0 ms — 즉시!)
📡 GOOGL 구독 완료! (소요: 0 ms — 즉시!)
📡 MSFT 구독 완료! (소요: 1 ms — 즉시!)

  [14:31:01.100] AAPL: $186.78
  [14:31:01.101] GOOGL: $141.05
  [14:31:01.102] MSFT: $419.33
  [14:31:02.100] AAPL: $187.12
  [14:31:02.101] GOOGL: $140.88
  [14:31:02.102] MSFT: $420.11
  ...
```

> **관찰:** 3개 종목이 **동시에 1초 간격으로** 실시간 업데이트됩니다.

### 테스트 5: 서비스 프로세스 Kill

CMD 또는 터미널에서:

```bash
adb shell "kill -9 $(pidof com.example.stocklab:stock_remote)"
```

앱 화면에서 확인:

```
☠ binderDied() 호출됨 (Binder Thread — 더 빠름!)
   → 서비스 자동 재시작 대기 중...
❌ onServiceDisconnected() 호출됨 (Main Thread)
```

> **관찰:** `binderDied()`가 `onServiceDisconnected()`보다 **먼저** 호출됩니다.
> `BIND_AUTO_CREATE` 덕분에 서비스가 자동 재생성되어, 잠시 후 `onServiceConnected()`가 다시 호출됩니다.

### 테스트 6: 프로세스 분리 확인

```bash
adb shell "ps -A -o PID,NAME | grep stocklab"
```

```
12345 com.example.stocklab              ← 앱 프로세스 (Client)
12346 com.example.stocklab:stock_remote  ← 서비스 프로세스 (Server)
```

> **관찰:** 2개의 별도 프로세스 → 이들 사이의 모든 통신이 **Binder IPC**입니다.

---

## Step 11: Logcat으로 서비스 측 로그 확인

Android Studio 하단 **Logcat** 탭에서 필터를 설정합니다:

```
tag:StockService OR tag:StockClient
```

서비스 측 로그:

```
D/StockService: ══════════════════════════════════
D/StockService: StockService CREATED (PID: 12346)
D/StockService: ══════════════════════════════════
D/StockService: onBind() — client connected!
D/StockService: subscribe(AAPL) — ASYNC, returning immediately!
D/StockService: subscribe(AAPL) — callback registered
D/StockService: getPrice(AAPL) — SYNC call, blocking 2 seconds...
D/StockService: getPrice(AAPL) — returning $185.50
```

> **PID가 앱과 다른 것**을 확인하세요 — 서비스는 별도 프로세스입니다.

---

## 핵심 학습 정리

| 관찰 | Binder 개념 |
|------|------------|
| 동기 호출 2000ms vs 비동기 0ms | `oneway`는 Driver에서 즉시 반환 — reply 대기 없음 |
| 구독 후 자동 업데이트 | 서비스 → 클라이언트 역방향 Binder IPC (콜백) |
| 콜백에서 `runOnUiThread` 필수 | 콜백은 **Binder Thread**에서 호출됨 |
| `binderDied()`가 먼저 호출됨 | Driver가 직접 감지 (Binder Thread) vs AMS 경유 (Main Thread) |
| 2개 프로세스 확인 | `android:process` 지정 → 실제 IPC 발생 증명 |
| 서비스 자동 재시작 | `BIND_AUTO_CREATE` 플래그의 효과 |

---

*이 실습을 완료하면 Binder IPC의 동기/비동기 동작, 콜백 패턴, DeathRecipient의 원리를 코드 레벨에서 이해할 수 있습니다.*

---

---

## 실습 6: Messenger 채팅 앱 — 경량 IPC

> **목표:** AIDL 없이 Messenger로 프로세스 간 양방향 통신 구현  
> **환경:** Windows 11 + Android Studio + Emulator (API 36)  
> **소요시간:** 약 30분

---

## 📋 이 실습에서 배우는 것

| 개념 | 체험 |
|------|------|
| Messenger IPC | AIDL 파일 없이 프로세스 간 통신 |
| Handler 기반 처리 | 단일 스레드에서 순차 처리 → 자동 스레드 안전 |
| replyTo 패턴 | 서비스 → 클라이언트 역방향 메시지 전달 |
| Message + Bundle | 약한 타입의 데이터 전달 방식 |
| Messenger vs AIDL | 같은 Binder 위에서 동작하지만 사용법이 다름 |

---

## Step 1: 프로젝트 생성

### 1-1. Android Studio에서 새 프로젝트

```
File → New → New Project
```

- 템플릿: **Empty Views Activity**
- Next 클릭

### 1-2. 프로젝트 설정

| 항목 | 값 |
|------|-----|
| Name | `MessengerChat` |
| Package name | `com.example.messengerchat` |
| Language | **Java** |
| Minimum SDK | **API 28** 이상 |
| Build configuration language | Kotlin DSL (기본값) |

- **Finish** 클릭
- Gradle sync 완료까지 대기

> ⚠ 이 실습에서는 **AIDL 파일을 생성하지 않습니다!** `buildFeatures { aidl = true }`도 불필요합니다. Messenger는 Framework에 내장된 IPC 메커니즘입니다.

---

## Step 2: 메시지 상수 정의 (ChatConstants.java)

서비스와 클라이언트가 공유하는 메시지 코드를 별도 클래스로 정의합니다.

```
app/src/main/java/com/example/messengerchat/ → 우클릭 → New → Java Class
Name: ChatConstants
```

**파일:** `ChatConstants.java`

```java
package com.example.messengerchat;

/**
 * 서비스와 클라이언트가 공유하는 메시지 상수
 * 
 * Messenger IPC에서는 AIDL처럼 메서드 이름이 없고,
 * Message.what 정수값으로 요청 종류를 구분합니다.
 */
public class ChatConstants {

    // 클라이언트 → 서비스 (요청)
    public static final int MSG_SEND_TEXT = 1;      // 텍스트 메시지 전송
    public static final int MSG_REGISTER = 2;       // 클라이언트 등록 (브로드캐스트용)
    public static final int MSG_UNREGISTER = 3;     // 클라이언트 등록 해제
    public static final int MSG_REQUEST_TIME = 4;   // 서버 시간 요청

    // 서비스 → 클라이언트 (응답)
    public static final int MSG_REPLY_TEXT = 10;     // 텍스트 응답
    public static final int MSG_REPLY_TIME = 11;     // 서버 시간 응답
    public static final int MSG_BROADCAST = 12;      // 전체 브로드캐스트

    // Bundle 키
    public static final String KEY_TEXT = "text";
    public static final String KEY_SENDER = "sender";
    public static final String KEY_TIME = "time";
}
```

---

## Step 3: 서비스 구현 (ChatService.java)

```
우클릭 → New → Java Class → Name: ChatService
```

**파일:** `ChatService.java`

```java
package com.example.messengerchat;

import android.app.Service;
import android.content.Intent;
import android.os.Bundle;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.os.Message;
import android.os.Messenger;
import android.os.RemoteException;
import android.util.Log;

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Locale;

/**
 * Messenger 기반 채팅 서비스
 * 
 * ★ Handler 기반이므로 단일 스레드에서 순차 처리
 *   → synchronized 불필요 (자동 스레드 안전!)
 * ★ AIDL 파일 불필요 — Messenger가 내부적으로 Binder 사용
 */
public class ChatService extends Service {
    private static final String TAG = "ChatService";

    // 등록된 클라이언트 목록 (브로드캐스트용)
    private final List<Messenger> clients = new ArrayList<>();

    // 메시지 이력
    private final List<String> messageHistory = new ArrayList<>();
    private int messageCount = 0;

    /**
     * ★ 핵심: Handler가 모든 메시지를 단일 스레드에서 처리
     * AIDL은 여러 Binder Thread에서 동시 호출되지만,
     * Messenger는 Handler Looper가 메시지를 큐에 넣고 순차 처리
     */
    class IncomingHandler extends Handler {
        IncomingHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            Log.d(TAG, "Received message: what=" + msg.what
                    + " from PID=" + android.os.Binder.getCallingPid());

            switch (msg.what) {

                case ChatConstants.MSG_SEND_TEXT: {
                    // ★ 텍스트 메시지 수신
                    String text = msg.getData().getString(ChatConstants.KEY_TEXT, "");
                    messageCount++;
                    String logEntry = "#" + messageCount + " " + text;
                    messageHistory.add(logEntry);
                    Log.d(TAG, "Message received: " + text);

                    // ★ replyTo가 설정되어 있으면 응답 전송
                    if (msg.replyTo != null) {
                        sendReply(msg.replyTo, text);
                    }

                    // 모든 등록된 클라이언트에게 브로드캐스트
                    broadcastToAll("📢 [서버] 새 메시지: " + text);
                    break;
                }

                case ChatConstants.MSG_REGISTER: {
                    // ★ 클라이언트 등록
                    if (msg.replyTo != null) {
                        clients.add(msg.replyTo);
                        Log.d(TAG, "Client registered. Total: " + clients.size());

                        // 환영 메시지 전송
                        sendTextToClient(msg.replyTo,
                                "👋 서버에 연결되었습니다! (등록 클라이언트: "
                                        + clients.size() + "명)");
                    }
                    break;
                }

                case ChatConstants.MSG_UNREGISTER: {
                    // 클라이언트 등록 해제
                    if (msg.replyTo != null) {
                        clients.remove(msg.replyTo);
                        Log.d(TAG, "Client unregistered. Remaining: " + clients.size());
                    }
                    break;
                }

                case ChatConstants.MSG_REQUEST_TIME: {
                    // ★ 서버 시간 요청 → 응답
                    if (msg.replyTo != null) {
                        String time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS",
                                Locale.getDefault()).format(new Date());

                        Message reply = Message.obtain(null, ChatConstants.MSG_REPLY_TIME);
                        Bundle data = new Bundle();
                        data.putString(ChatConstants.KEY_TIME, time);
                        data.putInt("messageCount", messageCount);
                        data.putInt("clientCount", clients.size());
                        reply.setData(data);

                        try {
                            msg.replyTo.send(reply);
                        } catch (RemoteException e) {
                            Log.w(TAG, "Failed to send time reply");
                        }
                    }
                    break;
                }

                default:
                    Log.w(TAG, "Unknown message type: " + msg.what);
                    super.handleMessage(msg);
            }
        }
    }

    /**
     * 특정 클라이언트에게 텍스트 응답 전송
     */
    private void sendReply(Messenger replyTo, String originalText) {
        Message reply = Message.obtain(null, ChatConstants.MSG_REPLY_TEXT);
        Bundle data = new Bundle();
        data.putString(ChatConstants.KEY_TEXT,
                "✅ Echo: " + originalText.toUpperCase());
        data.putString(ChatConstants.KEY_SENDER, "Server");
        data.putLong(ChatConstants.KEY_TIME, System.currentTimeMillis());
        reply.setData(data);

        try {
            replyTo.send(reply);
            Log.d(TAG, "Reply sent: " + originalText.toUpperCase());
        } catch (RemoteException e) {
            Log.w(TAG, "Client dead, cannot send reply");
        }
    }

    /**
     * 특정 클라이언트에게 텍스트 메시지 전송
     */
    private void sendTextToClient(Messenger client, String text) {
        Message msg = Message.obtain(null, ChatConstants.MSG_BROADCAST);
        Bundle data = new Bundle();
        data.putString(ChatConstants.KEY_TEXT, text);
        msg.setData(data);

        try {
            client.send(msg);
        } catch (RemoteException e) {
            Log.w(TAG, "Failed to send to client");
        }
    }

    /**
     * ★ 등록된 모든 클라이언트에게 메시지 브로드캐스트
     * 죽은 클라이언트는 자동 제거
     */
    private void broadcastToAll(String text) {
        List<Messenger> deadClients = new ArrayList<>();

        for (Messenger client : clients) {
            Message msg = Message.obtain(null, ChatConstants.MSG_BROADCAST);
            Bundle data = new Bundle();
            data.putString(ChatConstants.KEY_TEXT, text);
            msg.setData(data);

            try {
                client.send(msg);
            } catch (RemoteException e) {
                // 클라이언트가 죽었으면 제거 대상에 추가
                deadClients.add(client);
                Log.w(TAG, "Dead client detected, removing");
            }
        }

        // 죽은 클라이언트 정리
        clients.removeAll(deadClients);
    }

    // ★ Messenger 객체 — 이것의 getBinder()가 IBinder를 반환
    private Messenger messenger;

    @Override
    public void onCreate() {
        super.onCreate();
        messenger = new Messenger(new IncomingHandler(getMainLooper()));
        Log.d(TAG, "═══════════════════════════════");
        Log.d(TAG, "ChatService CREATED (PID: " + android.os.Process.myPid() + ")");
        Log.d(TAG, "═══════════════════════════════");
    }

    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "onBind() — returning Messenger's Binder");
        // ★ Messenger.getBinder()가 내부적으로 IMessenger.Stub을 반환
        //    즉, Messenger도 결국 Binder IPC를 사용!
        return messenger.getBinder();
    }

    @Override
    public void onDestroy() {
        Log.d(TAG, "ChatService DESTROYED");
        super.onDestroy();
    }
}
```

---

## Step 4: 레이아웃 작성 (activity_main.xml)

**파일:** `app/src/main/res/layout/activity_main.xml`  
기존 내용을 **전부 삭제**하고 아래로 교체:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="20dp">

    <!-- 제목 -->
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="💬 Messenger Chat Lab"
        android:textSize="22sp"
        android:textStyle="bold"
        android:layout_marginBottom="4dp"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="AIDL 없이 Messenger로 IPC 통신"
        android:textSize="13sp"
        android:textColor="#888"
        android:layout_marginBottom="12dp"/>

    <!-- 연결 상태 -->
    <TextView
        android:id="@+id/statusText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="⏳ 서비스 연결 중..."
        android:textSize="12sp"
        android:textColor="#555"
        android:background="#F0F0F0"
        android:padding="10dp"
        android:layout_marginBottom="12dp"/>

    <!-- 메시지 입력 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="8dp">

        <EditText
            android:id="@+id/inputText"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:hint="메시지를 입력하세요"
            android:text="Hello Android!"
            android:inputType="text"
            android:singleLine="true"/>

        <Button
            android:id="@+id/btnSend"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="전송"
            android:textAllCaps="false"
            android:layout_marginStart="8dp"/>
    </LinearLayout>

    <!-- 기능 버튼들 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="8dp">

        <Button
            android:id="@+id/btnTime"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="서버 시간\n요청"
            android:textSize="11sp"
            android:textAllCaps="false"/>

        <Button
            android:id="@+id/btnMulti"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="연속 전송\n(5개)"
            android:textSize="11sp"
            android:textAllCaps="false"
            android:layout_marginStart="8dp"/>

        <Button
            android:id="@+id/btnClear"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="로그\n지우기"
            android:textSize="11sp"
            android:textAllCaps="false"
            android:layout_marginStart="8dp"/>
    </LinearLayout>

    <!-- 채팅 로그 -->
    <ScrollView
        android:id="@+id/scrollView"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:background="#FAFAFA">

        <TextView
            android:id="@+id/logText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text=""
            android:textSize="12sp"
            android:fontFamily="monospace"
            android:padding="12dp"
            android:textColor="#333"/>
    </ScrollView>
</LinearLayout>
```

---

## Step 5: MainActivity.java 수정

기존 내용을 **전부 삭제**하고 아래로 교체:

**파일:** `MainActivity.java`

```java
package com.example.messengerchat;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.os.Message;
import android.os.Messenger;
import android.os.RemoteException;
import android.util.Log;
import android.widget.Button;
import android.widget.EditText;
import android.widget.ScrollView;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "ChatClient";

    // 서비스 측 Messenger (서비스에 메시지를 보낼 때 사용)
    private Messenger serviceMessenger = null;
    private boolean bound = false;

    private EditText inputText;
    private TextView logText;
    private TextView statusText;
    private ScrollView scrollView;

    // ==============================
    // ★ 클라이언트 측 Handler — 서비스로부터 응답을 수신
    // ==============================
    class ClientHandler extends Handler {
        ClientHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {

                case ChatConstants.MSG_REPLY_TEXT: {
                    // ★ 서비스로부터 에코 응답 수신
                    String text = msg.getData().getString(ChatConstants.KEY_TEXT, "");
                    String sender = msg.getData().getString(ChatConstants.KEY_SENDER, "?");
                    long time = msg.getData().getLong(ChatConstants.KEY_TIME, 0);
                    String timeStr = new SimpleDateFormat("HH:mm:ss.SSS",
                            Locale.getDefault()).format(new Date(time));
                    appendLog("  ← [" + timeStr + "] " + sender + ": " + text);
                    break;
                }

                case ChatConstants.MSG_REPLY_TIME: {
                    // ★ 서버 시간 응답 수신
                    String time = msg.getData().getString(ChatConstants.KEY_TIME, "?");
                    int msgCount = msg.getData().getInt("messageCount", 0);
                    int clientCount = msg.getData().getInt("clientCount", 0);
                    appendLog("  ← 🕐 서버 시간: " + time);
                    appendLog("       메시지 수: " + msgCount + " | 클라이언트: " + clientCount);
                    break;
                }

                case ChatConstants.MSG_BROADCAST: {
                    // ★ 브로드캐스트 메시지 수신
                    String text = msg.getData().getString(ChatConstants.KEY_TEXT, "");
                    appendLog("  ← " + text);
                    break;
                }

                default:
                    appendLog("  ← 알 수 없는 메시지: what=" + msg.what);
                    super.handleMessage(msg);
            }
        }
    }

    // ★ 클라이언트 Messenger — 서비스가 이것으로 응답을 보냄
    private Messenger clientMessenger;

    // ==============================
    // ServiceConnection
    // ==============================
    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // ★ IBinder → Messenger 변환 (AIDL의 Stub.asInterface에 해당)
            serviceMessenger = new Messenger(service);
            bound = true;

            int myPid = android.os.Process.myPid();
            statusText.setText("✅ ChatService 연결됨 (PID: " + myPid + ")");
            appendLog("✅ 서비스 연결 완료!");

            // ★ 서비스에 클라이언트 등록 (브로드캐스트 수신을 위해)
            registerToService();
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            serviceMessenger = null;
            bound = false;
            statusText.setText("❌ Service disconnected");
            appendLog("❌ 서비스 연결 해제됨");
        }
    };

    // ==============================
    // Activity 생명주기
    // ==============================
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // ★ 클라이언트 Messenger 생성 (응답 수신용)
        clientMessenger = new Messenger(new ClientHandler(getMainLooper()));

        // View 바인딩
        inputText = findViewById(R.id.inputText);
        logText = findViewById(R.id.logText);
        statusText = findViewById(R.id.statusText);
        scrollView = findViewById(R.id.scrollView);

        Button btnSend = findViewById(R.id.btnSend);
        Button btnTime = findViewById(R.id.btnTime);
        Button btnMulti = findViewById(R.id.btnMulti);
        Button btnClear = findViewById(R.id.btnClear);

        // 버튼 이벤트
        btnSend.setOnClickListener(v -> sendMessage());
        btnTime.setOnClickListener(v -> requestServerTime());
        btnMulti.setOnClickListener(v -> sendMultipleMessages());
        btnClear.setOnClickListener(v -> logText.setText(""));

        // 서비스 바인딩
        appendLog("═══ Messenger Chat Lab 시작 ═══");
        appendLog("서비스 바인딩 시도...");
        bindChatService();
    }

    @Override
    protected void onDestroy() {
        // ★ 서비스에서 클라이언트 등록 해제
        unregisterFromService();

        if (bound) {
            unbindService(connection);
            bound = false;
        }
        super.onDestroy();
    }

    // ==============================
    // 서비스 바인딩
    // ==============================
    private void bindChatService() {
        Intent intent = new Intent("com.example.messengerchat.CHAT_SERVICE");
        intent.setPackage("com.example.messengerchat");
        boolean result = bindService(intent, connection, Context.BIND_AUTO_CREATE);
        appendLog("bindService() 반환: " + result);
    }

    // ==============================
    // 서비스에 클라이언트 등록
    // ==============================
    private void registerToService() {
        if (!bound || serviceMessenger == null) return;

        Message msg = Message.obtain(null, ChatConstants.MSG_REGISTER);
        // ★ replyTo에 클라이언트 Messenger를 설정
        //    서비스가 이것을 저장하여 나중에 브로드캐스트에 사용
        msg.replyTo = clientMessenger;

        try {
            serviceMessenger.send(msg);
            appendLog("📋 서비스에 클라이언트 등록 완료");
        } catch (RemoteException e) {
            appendLog("❌ 등록 실패: " + e.getMessage());
        }
    }

    private void unregisterFromService() {
        if (!bound || serviceMessenger == null) return;

        Message msg = Message.obtain(null, ChatConstants.MSG_UNREGISTER);
        msg.replyTo = clientMessenger;

        try {
            serviceMessenger.send(msg);
        } catch (RemoteException e) {
            Log.w(TAG, "Unregister failed", e);
        }
    }

    // ==============================
    // ★ 메시지 전송 (핵심 동작)
    // ==============================
    private void sendMessage() {
        if (!bound || serviceMessenger == null) {
            appendLog("❌ 서비스 미연결!");
            return;
        }

        String text = inputText.getText().toString().trim();
        if (text.isEmpty()) {
            appendLog("❌ 메시지를 입력하세요!");
            return;
        }

        // ★ Message 객체 생성
        Message msg = Message.obtain(null, ChatConstants.MSG_SEND_TEXT);

        // ★ Bundle에 데이터 담기 (AIDL은 Parcel, Messenger는 Bundle)
        Bundle data = new Bundle();
        data.putString(ChatConstants.KEY_TEXT, text);
        msg.setData(data);

        // ★ 핵심! replyTo에 클라이언트 Messenger 설정
        //    서비스가 이 Messenger로 응답을 보내줌
        msg.replyTo = clientMessenger;

        long start = System.currentTimeMillis();
        try {
            serviceMessenger.send(msg);
            long elapsed = System.currentTimeMillis() - start;
            appendLog("→ [전송] \"" + text + "\" (소요: " + elapsed + "ms)");
        } catch (RemoteException e) {
            appendLog("❌ 전송 실패: " + e.getMessage());
        }
    }

    // ==============================
    // 서버 시간 요청
    // ==============================
    private void requestServerTime() {
        if (!bound || serviceMessenger == null) {
            appendLog("❌ 서비스 미연결!");
            return;
        }

        Message msg = Message.obtain(null, ChatConstants.MSG_REQUEST_TIME);
        msg.replyTo = clientMessenger;

        try {
            serviceMessenger.send(msg);
            appendLog("→ [요청] 서버 시간 조회");
        } catch (RemoteException e) {
            appendLog("❌ 요청 실패: " + e.getMessage());
        }
    }

    // ==============================
    // 연속 메시지 전송 테스트
    // ==============================
    private void sendMultipleMessages() {
        if (!bound || serviceMessenger == null) {
            appendLog("❌ 서비스 미연결!");
            return;
        }

        appendLog("═══ 연속 5개 메시지 전송 ═══");

        String[] messages = {"Hello!", "How are you?", "Binder IPC", "Messenger 테스트", "완료!"};

        for (int i = 0; i < messages.length; i++) {
            Message msg = Message.obtain(null, ChatConstants.MSG_SEND_TEXT);
            Bundle data = new Bundle();
            data.putString(ChatConstants.KEY_TEXT, messages[i]);
            msg.setData(data);
            msg.replyTo = clientMessenger;

            try {
                serviceMessenger.send(msg);
                appendLog("→ [" + (i + 1) + "/5] \"" + messages[i] + "\"");
            } catch (RemoteException e) {
                appendLog("❌ 전송 실패: " + e.getMessage());
            }
        }

        appendLog("★ Handler 큐에 순서대로 처리됨 (단일 스레드!)");
    }

    // ==============================
    // 로그 출력 헬퍼
    // ==============================
    private void appendLog(String msg) {
        logText.append(msg + "\n");
        scrollView.post(() -> scrollView.fullScroll(ScrollView.FOCUS_DOWN));
    }
}
```

---

## Step 6: AndroidManifest.xml 수정

**파일:** `app/src/main/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="MessengerChat"
        android:supportsRtl="true"
        android:theme="@style/Theme.MessengerChat"
        tools:targetApi="36">

        <!-- MainActivity -->
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- ★ ChatService — 별도 프로세스에서 실행! -->
        <service
            android:name=".ChatService"
            android:exported="true"
            android:process=":chat_remote">
            <intent-filter>
                <action android:name="com.example.messengerchat.CHAT_SERVICE" />
            </intent-filter>
        </service>

    </application>
</manifest>
```

> ⚠ `android:process=":chat_remote"` — 서비스가 별도 프로세스에서 실행됩니다. 이것이 **실제 Binder IPC**가 발생하는 핵심 설정입니다.

---

## Step 7: 최종 파일 구조 확인

```
app/src/main/
├── java/
│   └── com/example/messengerchat/
│       ├── ChatConstants.java      ← 메시지 상수 정의
│       ├── ChatService.java        ← 서비스 (별도 프로세스)
│       └── MainActivity.java       ← 클라이언트 (UI)
├── res/
│   └── layout/
│       └── activity_main.xml       ← UI 레이아웃
└── AndroidManifest.xml             ← 서비스 등록
```

총 **5개 파일**. AIDL 파일이 없습니다!

---

## Step 8: 빌드 및 실행

### 8-1. 빌드

```
Build → Make Project (Ctrl+F9)
```

### 8-2. 흔한 에러

| 에러 | 원인 | 해결 |
|------|------|------|
| `Handler()` deprecated 경고 | Looper 미지정 | `new Handler(Looper)` 사용 (코드에 이미 적용됨) |
| `Cannot resolve symbol ChatConstants` | 파일 누락 또는 패키지 불일치 | 패키지명 `com.example.messengerchat` 확인 |
| 서비스 연결 안 됨 | Intent action 불일치 | Manifest의 action과 코드의 Intent가 일치하는지 확인 |

### 8-3. 에뮬레이터에서 실행

```
Run → Run 'app' (Shift+F10)
```

---

## Step 9: 테스트 시나리오

### 테스트 1: 기본 메시지 전송

1. 앱 시작 → 자동 연결 확인
2. 입력란에 `Hello Android!` 입력
3. **전송** 버튼 클릭

예상 로그:

```
═══ Messenger Chat Lab 시작 ═══
서비스 바인딩 시도...
bindService() 반환: true
✅ 서비스 연결 완료!
📋 서비스에 클라이언트 등록 완료
  ← 👋 서버에 연결되었습니다! (등록 클라이언트: 1명)
→ [전송] "Hello Android!" (소요: 1ms)
  ← ✅ Echo: HELLO ANDROID!
  ← 📢 [서버] 새 메시지: Hello Android!
```

> **관찰:** 전송 후 **에코 응답**과 **브로드캐스트** 두 가지가 돌아옵니다.

### 테스트 2: 서버 시간 요청

**"서버 시간 요청"** 버튼 클릭:

```
→ [요청] 서버 시간 조회
  ← 🕐 서버 시간: 2026-03-29 14:30:15.123
       메시지 수: 1 | 클라이언트: 1
```

> **관찰:** 서비스가 자신의 상태 정보를 Bundle에 담아 응답합니다.

### 테스트 3: 연속 메시지 (순서 보장 확인)

**"연속 전송 (5개)"** 버튼 클릭:

```
═══ 연속 5개 메시지 전송 ═══
→ [1/5] "Hello!"
→ [2/5] "How are you?"
→ [3/5] "Binder IPC"
→ [4/5] "Messenger 테스트"
→ [5/5] "완료!"
★ Handler 큐에 순서대로 처리됨 (단일 스레드!)
  ← ✅ Echo: HELLO!
  ← 📢 [서버] 새 메시지: Hello!
  ← ✅ Echo: HOW ARE YOU?
  ← 📢 [서버] 새 메시지: How are you?
  ← ✅ Echo: BINDER IPC
  ← 📢 [서버] 새 메시지: Binder IPC
  ← ✅ Echo: MESSENGER 테스트
  ← 📢 [서버] 새 메시지: Messenger 테스트
  ← ✅ Echo: 완료!
  ← 📢 [서버] 새 메시지: 완료!
```

> **핵심 관찰:** 메시지가 **전송 순서 그대로** 처리됩니다. Handler가 단일 스레드에서 큐를 순차 처리하기 때문입니다. AIDL은 멀티스레드이므로 순서가 보장되지 않을 수 있습니다.

### 테스트 4: 프로세스 분리 확인

```bash
adb shell "ps -A -o PID,NAME | grep messenger"
```

```
12345 com.example.messengerchat              ← 앱 (Client)
12346 com.example.messengerchat:chat_remote   ← 서비스 (Server)
```

### 테스트 5: 서비스 프로세스 Kill

```bash
adb shell "kill -9 $(pidof com.example.messengerchat:chat_remote)"
```

앱 로그:

```
❌ 서비스 연결 해제됨
```

> `BIND_AUTO_CREATE` 덕분에 서비스가 자동 재시작됩니다.

---

## Step 10: AIDL과의 비교 실험

이전 실습(StockLab)과 이번 실습을 비교합니다:

### 코드 비교

| 항목 | Messenger (이번 실습) | AIDL (StockLab) |
|------|----------------------|-----------------|
| 인터페이스 정의 | 없음 (Message.what 상수만) | `.aidl` 파일 2개 |
| 서비스 구현 | Handler.handleMessage() | IStockService.Stub 상속 |
| 클라이언트 호출 | `messenger.send(msg)` | `service.getPrice("AAPL")` |
| 데이터 전달 | Bundle (키-값) | 메서드 파라미터 (강타입) |
| 역방향 통신 | `msg.replyTo` | Callback AIDL |
| 스레드 모델 | **단일** (Handler Looper) | **멀티** (Binder Thread Pool) |
| 타입 안전성 | 약함 (문자열 키) | 강함 (컴파일 타임 체크) |
| 동시성 | 순차 처리 | 동시 처리 |

### 동작 비교

| 관찰 | Messenger | AIDL |
|------|-----------|------|
| 연속 5개 전송 | **순서 보장** (Handler 큐) | 순서 보장 안 될 수 있음 |
| 동기 호출 | 불가 (항상 비동기) | 가능 (`getPrice` 블로킹) |
| 스레드 안전 | **자동** (단일 스레드) | 수동 (`synchronized` 필요) |
| 성능 | 단일 스레드 → 처리량 제한 | 멀티스레드 → 높은 처리량 |

---

## 핵심 학습 정리

| 관찰 | Binder 개념 |
|------|------------|
| AIDL 없이 IPC 동작 | Messenger 내부에서 `IMessenger.aidl` Binder를 자동 사용 |
| `messenger.getBinder()` | 서비스의 IBinder를 반환 — Binder IPC 진입점 |
| `new Messenger(service)` | IBinder → Messenger 변환 (AIDL의 `Stub.asInterface`에 해당) |
| `msg.replyTo` | 역방향 Binder IPC를 위한 클라이언트 Messenger 전달 |
| 5개 메시지 순서 보장 | Handler가 **MessageQueue**에서 FIFO 순서로 처리 |
| `synchronized` 불필요 | 단일 스레드 → 동시 접근 자체가 불가능 |
| 프로세스 2개 확인 | `android:process` 지정 → 실제 Binder IPC 발생 |

---

## Messenger 내부 구조 (이해를 위한 참고)

```
Client                          Service
  │                               │
  │  Messenger.send(msg)          │
  │    → IMessenger.Stub.Proxy    │
  │       .send(msg)              │
  │         → Binder IPC ─────►  │  IMessenger.Stub
  │                               │    .onTransact()
  │                               │      → Handler.sendMessage(msg)
  │                               │        → handleMessage(msg)
  │                               │
  │  msg.replyTo.send(reply)      │
  │    ◄── Binder IPC ──────────  │  replyTo = Client의 Messenger
  │  ClientHandler                │
  │    .handleMessage(reply)      │
```

> Messenger는 결국 **AIDL(`IMessenger.aidl`)의 래퍼**입니다. 차이는 개발자가 `.aidl` 파일을 직접 작성하지 않아도 되고, Handler 기반으로 자동 직렬화된다는 것입니다.

---

*이 실습을 완료하면 Messenger IPC의 동작 원리와 AIDL과의 차이를 코드 레벨에서 이해할 수 있습니다.*

---

---

## 실습 7: Content Provider 메모장 — 멀티프로세스 데이터 공유

> **목표:** Content Provider를 통한 구조화된 데이터 공유와 Binder IPC 동작 확인  
> **환경:** Windows 11 + Android Studio + Emulator (API 36)  
> **소요시간:** 약 45분

---

## 📋 이 실습에서 배우는 것

| 개념 | 체험 |
|------|------|
| Content Provider | 프로세스 간 구조화된 데이터 CRUD 공유 |
| Binder IPC (내부) | ContentResolver가 내부적으로 Binder Transaction 사용 |
| CursorWindow | 공유 메모리(FD) 기반 대용량 결과 전달 |
| ContentObserver | 데이터 변경 시 Binder 콜백으로 실시간 알림 |
| adb content 명령 | 외부에서 Provider에 직접 CRUD 요청 |

---

## Step 1: 프로젝트 생성

### 1-1. Android Studio에서 새 프로젝트

```
File → New → New Project
```

- 템플릿: **Empty Views Activity**
- Next 클릭

### 1-2. 프로젝트 설정

| 항목 | 값 |
|------|-----|
| Name | `MemoProvider` |
| Package name | `com.example.memoprovider` |
| Language | **Java** |
| Minimum SDK | **API 28** 이상 |
| Build configuration language | Kotlin DSL (기본값) |

- **Finish** 클릭
- Gradle sync 완료까지 대기

> ⚠ 이 실습에서는 AIDL 파일이 불필요합니다. Content Provider는 Framework 내장 Binder IPC를 자동으로 사용합니다.

---

## Step 2: 상수 정의 (MemoContract.java)

Provider의 URI, 테이블명, 컬럼명 등을 정의하는 계약(Contract) 클래스입니다.

```
app/src/main/java/com/example/memoprovider/ → 우클릭 → New → Java Class
Name: MemoContract
```

**파일:** `MemoContract.java`

```java
package com.example.memoprovider;

import android.net.Uri;

/**
 * Content Provider 계약 클래스
 * 
 * Provider와 클라이언트가 공유하는 상수 정의
 * URI 구조: content://com.example.memoprovider.memo/memos
 */
public final class MemoContract {

    // Provider Authority (AndroidManifest와 일치해야 함)
    public static final String AUTHORITY = "com.example.memoprovider.memo";

    // 기본 Content URI
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/memos");

    // 테이블명
    public static final String TABLE_NAME = "memos";

    // 컬럼명
    public static final String COL_ID = "_id";
    public static final String COL_TITLE = "title";
    public static final String COL_CONTENT = "content";
    public static final String COL_CREATED_AT = "created_at";

    // MIME 타입
    public static final String CONTENT_TYPE_DIR =
            "vnd.android.cursor.dir/vnd." + AUTHORITY + ".memo";
    public static final String CONTENT_TYPE_ITEM =
            "vnd.android.cursor.item/vnd." + AUTHORITY + ".memo";

    private MemoContract() {} // 인스턴스 생성 방지
}
```

---

## Step 3: Content Provider 구현 (MemoProvider.java)

```
우클릭 → New → Java Class → Name: MemoProvider
```

**파일:** `MemoProvider.java`

```java
package com.example.memoprovider;

import android.content.ContentProvider;
import android.content.ContentUris;
import android.content.ContentValues;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;
import android.net.Uri;
import android.os.Binder;
import android.util.Log;

import androidx.annotation.NonNull;
import androidx.annotation.Nullable;

/**
 * 메모 Content Provider — 별도 프로세스에서 실행
 * 
 * ★ ContentResolver.query/insert/update/delete 호출 시
 *   내부적으로 Binder IPC가 발생하여 이 Provider의 메서드가 호출됨
 * ★ query() 결과는 CursorWindow(공유 메모리)를 통해 전달
 */
public class MemoProvider extends ContentProvider {
    private static final String TAG = "MemoProvider";

    // URI 매칭 상수
    private static final int MEMOS = 1;      // content://authority/memos
    private static final int MEMO_ID = 2;     // content://authority/memos/123

    private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    static {
        uriMatcher.addURI(MemoContract.AUTHORITY, "memos", MEMOS);
        uriMatcher.addURI(MemoContract.AUTHORITY, "memos/#", MEMO_ID);
    }

    private SQLiteDatabase db;

    // ==============================
    // SQLite 헬퍼 (내부 클래스)
    // ==============================
    private static class MemoDbHelper extends SQLiteOpenHelper {
        private static final String DB_NAME = "memo.db";
        private static final int DB_VERSION = 1;

        MemoDbHelper(android.content.Context context) {
            super(context, DB_NAME, null, DB_VERSION);
        }

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL("CREATE TABLE " + MemoContract.TABLE_NAME + " ("
                    + MemoContract.COL_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, "
                    + MemoContract.COL_TITLE + " TEXT NOT NULL, "
                    + MemoContract.COL_CONTENT + " TEXT, "
                    + MemoContract.COL_CREATED_AT + " INTEGER NOT NULL"
                    + ")");
            Log.d("MemoProvider", "Database created!");
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
            db.execSQL("DROP TABLE IF EXISTS " + MemoContract.TABLE_NAME);
            onCreate(db);
        }
    }

    // ==============================
    // Provider 생명주기
    // ==============================
    @Override
    public boolean onCreate() {
        Log.d(TAG, "═══════════════════════════════════");
        Log.d(TAG, "MemoProvider CREATED (PID: " + android.os.Process.myPid() + ")");
        Log.d(TAG, "═══════════════════════════════════");

        MemoDbHelper helper = new MemoDbHelper(getContext());
        db = helper.getWritableDatabase();
        return db != null;
    }

    // ==============================
    // ★ query — SELECT (Binder IPC로 호출됨)
    // ==============================
    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection,
                        @Nullable String selection, @Nullable String[] selectionArgs,
                        @Nullable String sortOrder) {

        int callingUid = Binder.getCallingUid();
        int callingPid = Binder.getCallingPid();
        Log.d(TAG, "query() called by UID=" + callingUid + " PID=" + callingPid);

        Cursor cursor;
        switch (uriMatcher.match(uri)) {
            case MEMOS:
                // 전체 메모 조회
                cursor = db.query(
                        MemoContract.TABLE_NAME,
                        projection,
                        selection,
                        selectionArgs,
                        null, null,
                        sortOrder != null ? sortOrder : MemoContract.COL_CREATED_AT + " DESC"
                );
                break;

            case MEMO_ID:
                // 특정 메모 조회 (URI의 마지막 세그먼트가 ID)
                String id = uri.getLastPathSegment();
                cursor = db.query(
                        MemoContract.TABLE_NAME,
                        projection,
                        MemoContract.COL_ID + "=?",
                        new String[]{id},
                        null, null, null
                );
                break;

            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }

        // ★ ContentObserver 연동 — 데이터 변경 시 이 URI를 감시하는 Observer에게 알림
        if (getContext() != null) {
            cursor.setNotificationUri(getContext().getContentResolver(), uri);
        }

        Log.d(TAG, "query() returning " + cursor.getCount() + " rows");
        return cursor;
    }

    // ==============================
    // ★ insert — INSERT (Binder IPC로 호출됨)
    // ==============================
    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
        if (uriMatcher.match(uri) != MEMOS) {
            throw new IllegalArgumentException("Invalid URI for insert: " + uri);
        }

        int callingUid = Binder.getCallingUid();
        Log.d(TAG, "insert() called by UID=" + callingUid);

        // 생성 시간 자동 추가
        if (values != null && !values.containsKey(MemoContract.COL_CREATED_AT)) {
            values.put(MemoContract.COL_CREATED_AT, System.currentTimeMillis());
        }

        long id = db.insert(MemoContract.TABLE_NAME, null, values);
        if (id == -1) {
            Log.e(TAG, "insert() FAILED");
            return null;
        }

        Uri resultUri = ContentUris.withAppendedId(MemoContract.CONTENT_URI, id);
        Log.d(TAG, "insert() SUCCESS → " + resultUri);

        // ★ 변경 알림 → ContentObserver에게 Binder 콜백 전송!
        if (getContext() != null) {
            getContext().getContentResolver().notifyChange(MemoContract.CONTENT_URI, null);
        }

        return resultUri;
    }

    // ==============================
    // ★ update — UPDATE
    // ==============================
    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues values,
                      @Nullable String selection, @Nullable String[] selectionArgs) {

        int count;
        switch (uriMatcher.match(uri)) {
            case MEMOS:
                count = db.update(MemoContract.TABLE_NAME, values, selection, selectionArgs);
                break;
            case MEMO_ID:
                String id = uri.getLastPathSegment();
                count = db.update(MemoContract.TABLE_NAME, values,
                        MemoContract.COL_ID + "=?", new String[]{id});
                break;
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }

        Log.d(TAG, "update() affected " + count + " rows");

        if (count > 0 && getContext() != null) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return count;
    }

    // ==============================
    // ★ delete — DELETE
    // ==============================
    @Override
    public int delete(@NonNull Uri uri, @Nullable String selection,
                      @Nullable String[] selectionArgs) {

        int count;
        switch (uriMatcher.match(uri)) {
            case MEMOS:
                count = db.delete(MemoContract.TABLE_NAME, selection, selectionArgs);
                break;
            case MEMO_ID:
                String id = uri.getLastPathSegment();
                count = db.delete(MemoContract.TABLE_NAME,
                        MemoContract.COL_ID + "=?", new String[]{id});
                break;
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }

        Log.d(TAG, "delete() removed " + count + " rows");

        if (count > 0 && getContext() != null) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return count;
    }

    // ==============================
    // getType — MIME 타입 반환
    // ==============================
    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        switch (uriMatcher.match(uri)) {
            case MEMOS:
                return MemoContract.CONTENT_TYPE_DIR;
            case MEMO_ID:
                return MemoContract.CONTENT_TYPE_ITEM;
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }
}
```

---

## Step 4: 레이아웃 작성 (activity_main.xml)

**파일:** `app/src/main/res/layout/activity_main.xml`  
기존 내용을 **전부 삭제**하고 아래로 교체:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="20dp">

    <!-- 제목 -->
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="📝 Content Provider 메모장"
        android:textSize="22sp"
        android:textStyle="bold"
        android:layout_marginBottom="4dp"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="멀티프로세스 Binder IPC 데이터 공유"
        android:textSize="13sp"
        android:textColor="#888"
        android:layout_marginBottom="12dp"/>

    <!-- 상태 표시 -->
    <TextView
        android:id="@+id/statusText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="준비됨"
        android:textSize="12sp"
        android:textColor="#555"
        android:background="#F0F0F0"
        android:padding="10dp"
        android:layout_marginBottom="12dp"/>

    <!-- 입력 영역 -->
    <EditText
        android:id="@+id/inputTitle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="제목"
        android:text="첫 번째 메모"
        android:inputType="text"
        android:singleLine="true"
        android:layout_marginBottom="4dp"/>

    <EditText
        android:id="@+id/inputContent"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="내용"
        android:text="Content Provider를 통한 IPC 테스트!"
        android:inputType="text"
        android:singleLine="true"
        android:layout_marginBottom="8dp"/>

    <!-- CRUD 버튼 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="8dp">

        <Button
            android:id="@+id/btnInsert"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="추가\n(INSERT)"
            android:textSize="11sp"
            android:textAllCaps="false"/>

        <Button
            android:id="@+id/btnQuery"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="조회\n(SELECT)"
            android:textSize="11sp"
            android:textAllCaps="false"
            android:layout_marginStart="6dp"/>

        <Button
            android:id="@+id/btnDeleteAll"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="전체삭제\n(DELETE)"
            android:textSize="11sp"
            android:textAllCaps="false"
            android:layout_marginStart="6dp"/>

        <Button
            android:id="@+id/btnClear"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="로그\n지우기"
            android:textSize="11sp"
            android:textAllCaps="false"
            android:layout_marginStart="6dp"/>
    </LinearLayout>

    <!-- 대량 테스트 버튼 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="12dp">

        <Button
            android:id="@+id/btnBulkInsert"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="대량 삽입 (50개)"
            android:textSize="11sp"
            android:textAllCaps="false"/>

        <Button
            android:id="@+id/btnObserver"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="Observer 등록/해제"
            android:textSize="11sp"
            android:textAllCaps="false"
            android:layout_marginStart="6dp"/>
    </LinearLayout>

    <!-- 로그 출력 -->
    <ScrollView
        android:id="@+id/scrollView"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:background="#FAFAFA">

        <TextView
            android:id="@+id/logText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text=""
            android:textSize="12sp"
            android:fontFamily="monospace"
            android:padding="12dp"
            android:textColor="#333"/>
    </ScrollView>
</LinearLayout>
```

---

## Step 5: MainActivity.java 수정

기존 내용을 **전부 삭제**하고 아래로 교체:

**파일:** `MainActivity.java`

```java
package com.example.memoprovider;

import android.content.ContentValues;
import android.database.ContentObserver;
import android.database.Cursor;
import android.net.Uri;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.util.Log;
import android.widget.Button;
import android.widget.EditText;
import android.widget.ScrollView;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MemoClient";

    private EditText inputTitle;
    private EditText inputContent;
    private TextView logText;
    private TextView statusText;
    private ScrollView scrollView;

    private boolean observerRegistered = false;

    // ==============================
    // ★ ContentObserver — 데이터 변경 감지
    // Provider가 notifyChange()를 호출하면
    // Binder IPC 콜백으로 이 Observer의 onChange()가 호출됨
    // ==============================
    private final ContentObserver memoObserver = new ContentObserver(new Handler(Looper.getMainLooper())) {
        @Override
        public void onChange(boolean selfChange) {
            onChange(selfChange, null);
        }

        @Override
        public void onChange(boolean selfChange, Uri uri) {
            // ★ 데이터 변경 감지! (다른 프로세스에서 변경해도 알림 받음)
            appendLog("🔔 [Observer] 데이터 변경 감지! (selfChange=" + selfChange + ")");
            if (uri != null) {
                appendLog("   변경 URI: " + uri);
            }
            // 자동으로 목록 갱신
            queryMemos();
        }
    };

    // ==============================
    // Activity 생명주기
    // ==============================
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // View 바인딩
        inputTitle = findViewById(R.id.inputTitle);
        inputContent = findViewById(R.id.inputContent);
        logText = findViewById(R.id.logText);
        statusText = findViewById(R.id.statusText);
        scrollView = findViewById(R.id.scrollView);

        Button btnInsert = findViewById(R.id.btnInsert);
        Button btnQuery = findViewById(R.id.btnQuery);
        Button btnDeleteAll = findViewById(R.id.btnDeleteAll);
        Button btnClear = findViewById(R.id.btnClear);
        Button btnBulkInsert = findViewById(R.id.btnBulkInsert);
        Button btnObserver = findViewById(R.id.btnObserver);

        // 버튼 이벤트
        btnInsert.setOnClickListener(v -> insertMemo());
        btnQuery.setOnClickListener(v -> queryMemos());
        btnDeleteAll.setOnClickListener(v -> deleteAllMemos());
        btnClear.setOnClickListener(v -> logText.setText(""));
        btnBulkInsert.setOnClickListener(v -> bulkInsert());
        btnObserver.setOnClickListener(v -> toggleObserver());

        // 시작
        appendLog("═══ Content Provider 메모장 시작 ═══");
        appendLog("Client PID: " + android.os.Process.myPid());
        appendLog("");

        statusText.setText("✅ 준비됨 (PID: " + android.os.Process.myPid() + ")");

        // 초기 조회
        queryMemos();
    }

    @Override
    protected void onDestroy() {
        // Observer 해제
        if (observerRegistered) {
            getContentResolver().unregisterContentObserver(memoObserver);
        }
        super.onDestroy();
    }

    // ==============================
    // ★ INSERT — 메모 추가
    // ==============================
    private void insertMemo() {
        String title = inputTitle.getText().toString().trim();
        String content = inputContent.getText().toString().trim();

        if (title.isEmpty()) {
            appendLog("❌ 제목을 입력하세요!");
            return;
        }

        appendLog("═══ INSERT ═══");

        // ★ ContentValues에 데이터 담기
        ContentValues values = new ContentValues();
        values.put(MemoContract.COL_TITLE, title);
        values.put(MemoContract.COL_CONTENT, content);
        // created_at은 Provider에서 자동 추가

        long start = System.currentTimeMillis();

        // ★ 이 호출이 Binder IPC를 통해 Provider 프로세스에 전달됨!
        Uri resultUri = getContentResolver().insert(MemoContract.CONTENT_URI, values);

        long elapsed = System.currentTimeMillis() - start;

        if (resultUri != null) {
            appendLog("✅ 추가 성공! URI: " + resultUri);
            appendLog("   소요시간: " + elapsed + "ms (Binder IPC 포함)");
        } else {
            appendLog("❌ 추가 실패!");
        }
        appendLog("");
    }

    // ==============================
    // ★ SELECT — 메모 조회
    // ==============================
    private void queryMemos() {
        appendLog("═══ SELECT ═══");

        long start = System.currentTimeMillis();

        // ★ 이 호출이 Binder IPC를 통해 Provider 프로세스에 전달됨!
        // 결과는 CursorWindow(공유 메모리 FD)를 통해 반환됨
        Cursor cursor = getContentResolver().query(
                MemoContract.CONTENT_URI,
                null,     // 모든 컬럼
                null,     // WHERE 없음
                null,     // WHERE 인자 없음
                null      // 기본 정렬 (created_at DESC)
        );

        long elapsed = System.currentTimeMillis() - start;

        if (cursor == null) {
            appendLog("❌ 조회 실패 (cursor null)");
            return;
        }

        int count = cursor.getCount();
        appendLog("📋 총 " + count + "개 메모 (소요: " + elapsed + "ms)");
        statusText.setText("📋 메모 " + count + "개 | PID: " + android.os.Process.myPid());

        // 결과 출력
        SimpleDateFormat sdf = new SimpleDateFormat("MM/dd HH:mm:ss", Locale.getDefault());
        int num = 0;
        while (cursor.moveToNext()) {
            num++;
            int id = cursor.getInt(cursor.getColumnIndexOrThrow(MemoContract.COL_ID));
            String title = cursor.getString(cursor.getColumnIndexOrThrow(MemoContract.COL_TITLE));
            String content = cursor.getString(cursor.getColumnIndexOrThrow(MemoContract.COL_CONTENT));
            long createdAt = cursor.getLong(cursor.getColumnIndexOrThrow(MemoContract.COL_CREATED_AT));
            String time = sdf.format(new Date(createdAt));

            appendLog(String.format("  #%d [ID:%d] %s — %s (%s)", num, id, title, content, time));

            // 처음 10개만 출력
            if (num >= 10 && count > 10) {
                appendLog("  ... 외 " + (count - 10) + "개 생략");
                break;
            }
        }

        cursor.close();

        if (count == 0) {
            appendLog("  (메모가 없습니다)");
        }
        appendLog("");
    }

    // ==============================
    // ★ DELETE — 전체 삭제
    // ==============================
    private void deleteAllMemos() {
        appendLog("═══ DELETE ALL ═══");

        long start = System.currentTimeMillis();

        // ★ Binder IPC를 통해 Provider의 delete() 호출
        int deleted = getContentResolver().delete(
                MemoContract.CONTENT_URI,
                null,    // WHERE 없음 = 전체 삭제
                null
        );

        long elapsed = System.currentTimeMillis() - start;
        appendLog("🗑 " + deleted + "개 삭제됨 (소요: " + elapsed + "ms)");
        statusText.setText("🗑 전체 삭제 완료");
        appendLog("");
    }

    // ==============================
    // ★ BULK INSERT — 대량 삽입 (성능 테스트)
    // ==============================
    private void bulkInsert() {
        appendLog("═══ BULK INSERT (50개) ═══");

        long start = System.currentTimeMillis();

        for (int i = 1; i <= 50; i++) {
            ContentValues values = new ContentValues();
            values.put(MemoContract.COL_TITLE, "Memo #" + i);
            values.put(MemoContract.COL_CONTENT, "Bulk insert test item " + i);
            getContentResolver().insert(MemoContract.CONTENT_URI, values);
        }

        long elapsed = System.currentTimeMillis() - start;
        appendLog("✅ 50개 삽입 완료! (총 소요: " + elapsed + "ms)");
        appendLog("   평균: " + (elapsed / 50) + "ms/건 (각각 Binder IPC)");
        appendLog("");

        // 결과 확인
        queryMemos();
    }

    // ==============================
    // ★ ContentObserver 등록/해제
    // ==============================
    private void toggleObserver() {
        if (observerRegistered) {
            getContentResolver().unregisterContentObserver(memoObserver);
            observerRegistered = false;
            appendLog("🔕 ContentObserver 해제됨");
            appendLog("   → 외부 변경 알림을 더 이상 받지 않음");
        } else {
            // ★ ContentObserver 등록
            // notifyForDescendants=true: 하위 URI 변경도 감지
            getContentResolver().registerContentObserver(
                    MemoContract.CONTENT_URI,
                    true,  // notifyForDescendants
                    memoObserver
            );
            observerRegistered = true;
            appendLog("🔔 ContentObserver 등록됨!");
            appendLog("   → 다른 프로세스(adb 포함)에서 변경하면 알림 수신");
            appendLog("   → 테스트: adb shell content insert 명령 실행해보세요");
        }
        appendLog("");
    }

    // ==============================
    // 로그 출력 헬퍼
    // ==============================
    private void appendLog(String msg) {
        logText.append(msg + "\n");
        scrollView.post(() -> scrollView.fullScroll(ScrollView.FOCUS_DOWN));
    }
}
```

---

## Step 6: AndroidManifest.xml 수정

**파일:** `app/src/main/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="MemoProvider"
        android:supportsRtl="true"
        android:theme="@style/Theme.MemoProvider"
        tools:targetApi="36">

        <!-- MainActivity -->
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- ★ Content Provider — 별도 프로세스에서 실행! -->
        <provider
            android:name=".MemoProvider"
            android:authorities="com.example.memoprovider.memo"
            android:exported="false"
            android:process=":provider" />

    </application>
</manifest>
```

> ⚠ **`android:process=":provider"`** — Provider가 별도 프로세스에서 실행됩니다. ContentResolver의 모든 호출이 **실제 Binder IPC**를 통해 전달됩니다.  
> ⚠ **`android:authorities`** 값이 `MemoContract.AUTHORITY`와 **정확히 일치**해야 합니다.

---

## Step 7: 최종 파일 구조 확인

```
app/src/main/
├── java/
│   └── com/example/memoprovider/
│       ├── MemoContract.java       ← URI/컬럼 상수
│       ├── MemoProvider.java       ← Content Provider (별도 프로세스)
│       └── MainActivity.java       ← 클라이언트 (UI)
├── res/
│   └── layout/
│       └── activity_main.xml       ← UI 레이아웃
└── AndroidManifest.xml             ← Provider 등록
```

총 **5개 파일**. AIDL 파일 없음!

---

## Step 8: 빌드 및 실행

### 8-1. 빌드

```
Build → Make Project (Ctrl+F9)
```

### 8-2. 흔한 에러

| 에러 | 원인 | 해결 |
|------|------|------|
| `Failed to find provider info` | authorities 불일치 | Manifest와 MemoContract.AUTHORITY 값 비교 |
| `SecurityException: Permission Denial` | exported 설정 문제 | `exported="false"` 확인 (같은 앱이므로 접근 가능) |
| `NullPointerException in query` | cursor 컬럼 이름 오타 | `getColumnIndexOrThrow()` 결과 확인 |

### 8-3. 에뮬레이터에서 실행

```
Run → Run 'app' (Shift+F10)
```

---

## Step 9: 테스트 시나리오

### 테스트 1: 메모 추가 (INSERT)

1. 제목: `첫 번째 메모`, 내용: `Content Provider 테스트!`
2. **추가(INSERT)** 버튼 클릭

```
═══ INSERT ═══
✅ 추가 성공! URI: content://com.example.memoprovider.memo/memos/1
   소요시간: 5ms (Binder IPC 포함)
```

### 테스트 2: 메모 조회 (SELECT)

**조회(SELECT)** 버튼 클릭:

```
═══ SELECT ═══
📋 총 1개 메모 (소요: 3ms)
  #1 [ID:1] 첫 번째 메모 — Content Provider 테스트! (03/29 14:30:15)
```

### 테스트 3: 대량 삽입 성능 테스트

**대량 삽입 (50개)** 버튼 클릭:

```
═══ BULK INSERT (50개) ═══
✅ 50개 삽입 완료! (총 소요: 320ms)
   평균: 6ms/건 (각각 Binder IPC)

═══ SELECT ═══
📋 총 51개 메모 (소요: 4ms)
  #1 [ID:51] Memo #50 — Bulk insert test item 50 (03/29 14:31:02)
  #2 [ID:50] Memo #49 — Bulk insert test item 49 (03/29 14:31:02)
  ...
  #10 [ID:42] Memo #41 — Bulk insert test item 41 (03/29 14:31:02)
  ... 외 41개 생략
```

> **관찰:** 50건 삽입에 각각 Binder IPC가 발생하여 총 ~300ms. 건당 ~6ms는 Binder Transaction 오버헤드입니다.

### 테스트 4: ContentObserver 등록 + adb에서 외부 변경

1. 앱에서 **Observer 등록/해제** 버튼 클릭 (등록 상태)
2. **별도 터미널**에서 adb 명령으로 메모를 추가:

```bash

am get-current-user

adb shell content insert --user 10 --uri content://com.example.memoprovider.memo/memos \
    --bind title:s:"Memo added in ADB" \
    --bind content:s:"Binder IPC in external processes!" \
    --bind created_at:l:$(date +%s000)
```

앱 화면에 즉시 알림이 나타납니다:

```
🔔 [Observer] 데이터 변경 감지! (selfChange=false)
   변경 URI: content://com.example.memoprovider.memo/memos

═══ SELECT ═══
📋 총 52개 메모 (소요: 3ms)
  #1 [ID:52] ADB에서 추가한 메모 — 외부 프로세스에서 Binder IPC! (03/29 14:32:00)
  ...
```

> **핵심 관찰:** adb는 **완전히 다른 프로세스**입니다. 그런데도 ContentObserver가 알림을 받았습니다. 이것이 **Binder 콜백 IPC**입니다.

### 테스트 5: adb에서 직접 CRUD

```bash
# 조회
adb shell content query --user 10 --uri content://com.example.memoprovider.memo/memos

# 특정 ID 조회
adb shell content query --user 10 --uri content://com.example.memoprovider.memo/memos/1

# 삽입
adb shell content insert --user 10 --uri content://com.example.memoprovider.memo/memos \
    --bind title:s:"터미널 메모" \
    --bind content:s:"adb에서 직접 삽입" \
    --bind created_at:l:1711700000000

# 삭제
adb shell content delete --user 10 --uri content://com.example.memoprovider.memo/memos/1

# 전체 삭제
adb shell content delete --user 10 --uri content://com.example.memoprovider.memo/memos
```

### 테스트 6: 프로세스 분리 확인

```bash
adb shell "ps -A -o PID,NAME | grep memoprovider"
```

```
12345 com.example.memoprovider              ← 앱 프로세스 (Client)
12346 com.example.memoprovider:provider      ← Provider 프로세스 (Server)
```

> 2개 프로세스 → ContentResolver의 모든 CRUD 호출이 **Binder IPC**입니다.

### 테스트 7: 전체 삭제

**전체삭제(DELETE)** 버튼 클릭:

```
═══ DELETE ALL ═══
🗑 52개 삭제됨 (소요: 8ms)
```

---

## Step 10: Logcat으로 Provider 측 로그 확인

Android Studio 하단 **Logcat** 탭에서 필터:

```
tag:MemoProvider OR tag:MemoClient
```

Provider 측 로그:

```
D/MemoProvider: ═══════════════════════════════════
D/MemoProvider: MemoProvider CREATED (PID: 12346)
D/MemoProvider: ═══════════════════════════════════
D/MemoProvider: insert() called by UID=10150
D/MemoProvider: insert() SUCCESS → content://com.example.memoprovider.memo/memos/1
D/MemoProvider: query() called by UID=10150 PID=12345
D/MemoProvider: query() returning 1 rows
```

> **PID가 앱과 다른 것**을 확인하세요. Provider는 별도 프로세스(`:provider`)에서 실행 중입니다. `Binder.getCallingUid()`로 호출자를 확인할 수 있습니다.

---

## 핵심 학습 정리

| 관찰 | Binder 개념 |
|------|------------|
| insert/query 소요시간 ~5ms | Binder Transaction 오버헤드 (직렬화 + 커널 경유 + 역직렬화) |
| 50건 대량 삽입 ~300ms | 건당 Binder IPC → 대량 처리 시 `bulkInsert()` 사용 권장 |
| adb content 명령이 동작 | Provider는 **어떤 프로세스**에서든 URI로 접근 가능 |
| ContentObserver가 adb 변경 감지 | `notifyChange()` → Binder 콜백 → Observer.onChange() |
| CursorWindow | query() 결과를 공유 메모리(FD)로 전달 → 1MB 제한 우회 |
| `Binder.getCallingUid()` | Provider에서 호출자의 UID를 확인 → 접근 제어 가능 |
| 프로세스 2개 확인 | `android:process=":provider"` → 실제 IPC 발생 |

---

## Messenger / AIDL / Content Provider 비교

| 항목 | Messenger | AIDL | Content Provider |
|------|-----------|------|-----------------|
| 용도 | 단순 메시지 교환 | 복잡한 API 호출 | 구조화된 데이터 공유 |
| 인터페이스 | Message.what | .aidl 메서드 | URI + CRUD |
| 데이터 | Bundle | Parcelable | ContentValues/Cursor |
| 스레드 | 단일 (Handler) | 멀티 (Pool) | 멀티 (Pool) |
| 외부 접근 | 불가 | 불가 | **가능** (adb, 다른 앱) |
| 실시간 알림 | replyTo 콜백 | Callback AIDL | **ContentObserver** |

---

*이 실습을 완료하면 Content Provider의 CRUD 동작, Binder IPC 구조, ContentObserver의 원리를 코드 레벨에서 이해할 수 있습니다.*

---

---

## 실습 8: TransactionTooLargeException 재현 & 해결

> **목표:** Binder 1MB 제한을 직접 체험하고, ParcelFileDescriptor로 우회하는 방법 학습  
> **환경:** Windows 11 + Android Studio + Emulator (API 36)  
> **소요시간:** 약 20분

---

## 📋 이 실습에서 배우는 것

| 개념 | 체험 |
|------|------|
| Binder 버퍼 1MB 제한 | 크기를 점진적으로 늘려 정확히 어디서 터지는지 확인 |
| TransactionTooLargeException | 실제 예외를 의도적으로 발생시켜 로그 관찰 |
| ParcelFileDescriptor (Pipe) | FD만 Binder로 전달하여 크기 제한 우회 |
| 크기별 전송 시간 비교 | Binder 직접 vs Pipe 전달 성능 차이 측정 |

---

## Step 1: 프로젝트 생성

### 1-1. Android Studio에서 새 프로젝트

```
File → New → New Project
```

- 템플릿: **Empty Views Activity**
- Next 클릭

### 1-2. 프로젝트 설정

| 항목 | 값 |
|------|-----|
| Name | `BinderLimitLab` |
| Package name | `com.example.binderlimitlab` |
| Language | **Java** |
| Minimum SDK | **API 28** 이상 |

- **Finish** 클릭
- Gradle sync 완료 대기

---

## Step 2: AIDL 빌드 기능 활성화

**파일:** `app/build.gradle.kts`

`android { }` 블록 안에 추가:

```kotlin
android {
    namespace = "com.example.binderlimitlab"
    compileSdk = 36

    // ★ 추가!
    buildFeatures {
        aidl = true
    }

    // ... 나머지 그대로 ...
}
```

**Sync Now** 클릭.

---

## Step 3: AIDL 인터페이스 정의

### 3-1. AIDL 디렉토리 생성

```
app/src/main/aidl/com/example/binderlimitlab/
```

### 3-2. IDataService.aidl 생성

**파일:** `app/src/main/aidl/com/example/binderlimitlab/IDataService.aidl`

```java
package com.example.binderlimitlab;

import android.os.ParcelFileDescriptor;

interface IDataService {
    // ★ 방법 1: Binder 직접 전달 (1MB 제한에 걸림!)
    byte[] getLargeData(int sizeInKB);

    // ★ 방법 2: ParcelFileDescriptor로 전달 (크기 제한 없음!)
    ParcelFileDescriptor getLargeDataSafe(int sizeInKB);

    // 서비스 정보 조회
    String getServiceInfo();
}
```

### 3-3. 빌드하여 Stub/Proxy 생성

```
Build → Make Project (Ctrl+F9)
```

---

## Step 4: 서비스 구현 (DataService.java)

```
app/src/main/java/com/example/binderlimitlab/ → 우클릭 → New → Java Class
Name: DataService
```

**파일:** `DataService.java`

```java
package com.example.binderlimitlab;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.os.ParcelFileDescriptor;
import android.os.RemoteException;
import android.util.Log;

import java.io.IOException;
import java.io.OutputStream;
import java.util.Random;

/**
 * 대용량 데이터 전송 서비스 — 별도 프로세스에서 실행
 * 
 * 두 가지 전송 방식을 비교:
 * 1) getLargeData(): Binder 직접 전달 → 1MB 초과 시 크래시!
 * 2) getLargeDataSafe(): Pipe(FD)로 전달 → 크기 제한 없음
 */
public class DataService extends Service {
    private static final String TAG = "DataService";
    private final Random random = new Random();

    private final IDataService.Stub binder = new IDataService.Stub() {

        /**
         * ★ 방법 1: Binder 버퍼로 직접 전달
         * Binder Transaction 버퍼는 프로세스당 약 1MB
         * 이 크기를 초과하면 TransactionTooLargeException 발생!
         */
        @Override
        public byte[] getLargeData(int sizeInKB) throws RemoteException {
            Log.d(TAG, "getLargeData(" + sizeInKB + "KB) — Binder 직접 전달 시도...");

            byte[] data = new byte[sizeInKB * 1024];
            random.nextBytes(data);

            Log.d(TAG, "getLargeData() — " + sizeInKB + "KB 데이터 생성 완료, 반환 중...");
            // ★ 이 return에서 Parcel에 직렬화 → Binder Transaction 발생
            //    크기가 ~1MB를 넘으면 TransactionTooLargeException!
            return data;
        }

        /**
         * ★ 방법 2: ParcelFileDescriptor (Pipe)로 전달
         * FD(파일 디스크립터, 4바이트)만 Binder로 전달
         * 실제 데이터는 Pipe를 통해 별도로 전송 → 크기 제한 없음!
         */
        @Override
        public ParcelFileDescriptor getLargeDataSafe(int sizeInKB) throws RemoteException {
            Log.d(TAG, "getLargeDataSafe(" + sizeInKB + "KB) — Pipe 전달 시도...");

            try {
                // ★ Pipe 생성: [0]=읽기 쪽(클라이언트), [1]=쓰기 쪽(서비스)
                ParcelFileDescriptor[] pipe = ParcelFileDescriptor.createPipe();
                ParcelFileDescriptor readSide = pipe[0];
                ParcelFileDescriptor writeSide = pipe[1];

                // ★ 백그라운드 스레드에서 데이터 쓰기
                //    Binder 호출은 즉시 반환, 데이터는 Pipe로 비동기 전달
                new Thread(() -> {
                    try (OutputStream os =
                             new ParcelFileDescriptor.AutoCloseOutputStream(writeSide)) {

                        byte[] data = new byte[sizeInKB * 1024];
                        random.nextBytes(data);

                        // 청크 단위로 쓰기 (대용량 안정성)
                        int chunkSize = 64 * 1024; // 64KB씩
                        int offset = 0;
                        while (offset < data.length) {
                            int len = Math.min(chunkSize, data.length - offset);
                            os.write(data, offset, len);
                            offset += len;
                        }

                        Log.d(TAG, "getLargeDataSafe() — " + sizeInKB + "KB Pipe 전송 완료");
                    } catch (IOException e) {
                        Log.e(TAG, "Pipe write failed: " + e.getMessage());
                    }
                }, "PipeWriter-" + sizeInKB + "KB").start();

                // ★ readSide(FD)만 Binder로 반환 — 4바이트!
                return readSide;

            } catch (IOException e) {
                throw new RemoteException("Pipe creation failed: " + e.getMessage());
            }
        }

        @Override
        public String getServiceInfo() throws RemoteException {
            return "DataService PID=" + android.os.Process.myPid()
                    + " | Binder limit: ~1MB per transaction";
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        Log.d(TAG, "═══════════════════════════════════");
        Log.d(TAG, "DataService CREATED (PID: " + android.os.Process.myPid() + ")");
        Log.d(TAG, "═══════════════════════════════════");
    }

    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "onBind()");
        return binder;
    }

    @Override
    public void onDestroy() {
        Log.d(TAG, "DataService DESTROYED");
        super.onDestroy();
    }
}
```

---

## Step 5: 레이아웃 작성 (activity_main.xml)

**파일:** `app/src/main/res/layout/activity_main.xml`  
기존 내용을 **전부 삭제**하고 아래로 교체:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="20dp">

    <!-- 제목 -->
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="💥 Binder 1MB Limit Lab"
        android:textSize="22sp"
        android:textStyle="bold"
        android:layout_marginBottom="4dp"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="TransactionTooLargeException 재현 및 해결"
        android:textSize="13sp"
        android:textColor="#888"
        android:layout_marginBottom="12dp"/>

    <!-- 상태 표시 -->
    <TextView
        android:id="@+id/statusText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="⏳ 서비스 연결 중..."
        android:textSize="12sp"
        android:textColor="#555"
        android:background="#F0F0F0"
        android:padding="10dp"
        android:layout_marginBottom="12dp"/>

    <!-- 테스트 버튼 행 1 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="8dp">

        <Button
            android:id="@+id/btnTestBinder"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="Binder 직접 전달\n(크기 증가 테스트)"
            android:textSize="11sp"
            android:textAllCaps="false"/>

        <Button
            android:id="@+id/btnTestPipe"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="Pipe(FD) 전달\n(크기 제한 없음)"
            android:textSize="11sp"
            android:textAllCaps="false"
            android:layout_marginStart="8dp"/>
    </LinearLayout>

    <!-- 테스트 버튼 행 2 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginBottom="12dp">

        <Button
            android:id="@+id/btnCompare"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="비교 테스트\n(Binder vs Pipe)"
            android:textSize="11sp"
            android:textAllCaps="false"/>

        <Button
            android:id="@+id/btnClear"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="로그 지우기"
            android:textSize="12sp"
            android:textAllCaps="false"
            android:layout_marginStart="8dp"/>
    </LinearLayout>

    <!-- 로그 출력 -->
    <ScrollView
        android:id="@+id/scrollView"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:background="#FAFAFA">

        <TextView
            android:id="@+id/logText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text=""
            android:textSize="12sp"
            android:fontFamily="monospace"
            android:padding="12dp"
            android:textColor="#333"/>
    </ScrollView>
</LinearLayout>
```

---

## Step 6: MainActivity.java 수정

기존 내용을 **전부 삭제**하고 아래로 교체:

**파일:** `MainActivity.java`

```java
package com.example.binderlimitlab;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.os.ParcelFileDescriptor;
import android.os.RemoteException;
import android.util.Log;
import android.widget.Button;
import android.widget.ScrollView;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;

import java.io.ByteArrayOutputStream;
import java.io.InputStream;

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "LimitClient";

    private IDataService dataService;
    private boolean bound = false;

    private TextView logText;
    private TextView statusText;
    private ScrollView scrollView;

    // ==============================
    // ServiceConnection
    // ==============================
    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            dataService = IDataService.Stub.asInterface(service);
            bound = true;

            try {
                String info = dataService.getServiceInfo();
                statusText.setText("✅ " + info);
                appendLog("✅ 서비스 연결 완료!");
                appendLog("   " + info);
                appendLog("");
            } catch (RemoteException e) {
                statusText.setText("✅ 연결됨");
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            dataService = null;
            bound = false;
            statusText.setText("❌ Service disconnected");
            appendLog("❌ 서비스 연결 해제됨");
        }
    };

    // ==============================
    // Activity 생명주기
    // ==============================
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        logText = findViewById(R.id.logText);
        statusText = findViewById(R.id.statusText);
        scrollView = findViewById(R.id.scrollView);

        Button btnTestBinder = findViewById(R.id.btnTestBinder);
        Button btnTestPipe = findViewById(R.id.btnTestPipe);
        Button btnCompare = findViewById(R.id.btnCompare);
        Button btnClear = findViewById(R.id.btnClear);

        btnTestBinder.setOnClickListener(v -> testBinderDirect());
        btnTestPipe.setOnClickListener(v -> testPipeTransfer());
        btnCompare.setOnClickListener(v -> testComparison());
        btnClear.setOnClickListener(v -> logText.setText(""));

        appendLog("═══ Binder 1MB Limit Lab 시작 ═══");
        appendLog("");

        // 서비스 바인딩
        bindDataService();
    }

    @Override
    protected void onDestroy() {
        if (bound) unbindService(connection);
        super.onDestroy();
    }

    private void bindDataService() {
        Intent intent = new Intent("com.example.binderlimitlab.DATA_SERVICE");
        intent.setPackage("com.example.binderlimitlab");
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }

    // ==============================
    // ★ 테스트 1: Binder 직접 전달 (크기 증가)
    // ==============================
    private void testBinderDirect() {
        if (!bound || dataService == null) {
            appendLog("❌ 서비스 미연결!");
            return;
        }

        appendLog("═══════════════════════════════════");
        appendLog("★ Binder 직접 전달 테스트");
        appendLog("  크기를 점진적으로 증가시켜 한계를 찾습니다");
        appendLog("═══════════════════════════════════");

        // ★ 별도 스레드에서 실행 (UI 블로킹 방지)
        new Thread(() -> {
            int[] sizes = {10, 50, 100, 200, 400, 600, 800, 900, 1000, 1024, 2048};

            for (int kb : sizes) {
                try {
                    long start = System.currentTimeMillis();

                    // ★ 이 호출에서 TransactionTooLargeException 발생 가능!
                    byte[] data = dataService.getLargeData(kb);

                    long elapsed = System.currentTimeMillis() - start;
                    int received = data != null ? data.length / 1024 : 0;

                    String msg = String.format("  ✅ %4dKB: 성공! (%dKB 수신, %dms)",
                            kb, received, elapsed);
                    runOnUiThread(() -> appendLog(msg));

                } catch (Exception e) {
                    String errorName = e.getClass().getSimpleName();
                    String msg = String.format("  💥 %4dKB: 실패! → %s", kb, errorName);
                    runOnUiThread(() -> {
                        appendLog(msg);
                        appendLog("     └ " + e.getMessage());
                    });

                    // 실패 후에도 계속 테스트 (더 큰 크기도 확인)
                }
            }

            runOnUiThread(() -> {
                appendLog("");
                appendLog("결론: Binder 버퍼 ~1MB 초과 시 예외 발생!");
                appendLog("  → 대용량 데이터는 Pipe/FD로 전달해야 함");
                appendLog("");
            });
        }, "BinderTestThread").start();
    }

    // ==============================
    // ★ 테스트 2: Pipe(FD) 전달 (크기 제한 없음)
    // ==============================
    private void testPipeTransfer() {
        if (!bound || dataService == null) {
            appendLog("❌ 서비스 미연결!");
            return;
        }

        appendLog("═══════════════════════════════════");
        appendLog("★ Pipe(FD) 전달 테스트");
        appendLog("  FD만 Binder로 전달 → 크기 제한 없음!");
        appendLog("═══════════════════════════════════");

        new Thread(() -> {
            // Binder에서 실패했던 크기 + 훨씬 더 큰 크기도 테스트
            int[] sizes = {100, 500, 1024, 2048, 4096, 8192};

            for (int kb : sizes) {
                try {
                    long start = System.currentTimeMillis();

                    // ★ FD만 Binder로 전달 (4바이트!)
                    ParcelFileDescriptor pfd = dataService.getLargeDataSafe(kb);

                    // ★ Pipe에서 데이터 읽기 (Binder와 무관)
                    InputStream is = new ParcelFileDescriptor.AutoCloseInputStream(pfd);
                    ByteArrayOutputStream bos = new ByteArrayOutputStream();
                    byte[] buf = new byte[64 * 1024]; // 64KB 버퍼
                    int len;
                    while ((len = is.read(buf)) != -1) {
                        bos.write(buf, 0, len);
                    }
                    is.close();

                    long elapsed = System.currentTimeMillis() - start;
                    int received = bos.size() / 1024;

                    String msg = String.format("  ✅ %4dKB: 성공! (%dKB 수신, %dms)",
                            kb, received, elapsed);
                    runOnUiThread(() -> appendLog(msg));

                } catch (Exception e) {
                    String msg = String.format("  ❌ %4dKB: 실패! → %s",
                            kb, e.getMessage());
                    runOnUiThread(() -> appendLog(msg));
                }
            }

            runOnUiThread(() -> {
                appendLog("");
                appendLog("결론: Pipe(FD)는 8MB도 문제없이 전달!");
                appendLog("  → Binder로는 FD(4바이트)만 전달");
                appendLog("  → 실제 데이터는 커널 Pipe 버퍼로 전송");
                appendLog("");
            });
        }, "PipeTestThread").start();
    }

    // ==============================
    // ★ 테스트 3: 동일 크기에서 Binder vs Pipe 비교
    // ==============================
    private void testComparison() {
        if (!bound || dataService == null) {
            appendLog("❌ 서비스 미연결!");
            return;
        }

        appendLog("═══════════════════════════════════");
        appendLog("★ 성능 비교: Binder 직접 vs Pipe(FD)");
        appendLog("  동일 크기(500KB)로 비교합니다");
        appendLog("═══════════════════════════════════");

        new Thread(() -> {
            int testSizeKB = 500;
            int trials = 3;

            // === Binder 직접 ===
            runOnUiThread(() -> appendLog("\n[Binder 직접 전달 — " + testSizeKB + "KB × " + trials + "회]"));

            long binderTotal = 0;
            for (int i = 0; i < trials; i++) {
                try {
                    long start = System.currentTimeMillis();
                    byte[] data = dataService.getLargeData(testSizeKB);
                    long elapsed = System.currentTimeMillis() - start;
                    binderTotal += elapsed;

                    int n = i + 1;
                    runOnUiThread(() ->
                            appendLog(String.format("  시도 %d: %dms (%dKB)",
                                    n, elapsed, data.length / 1024)));
                } catch (Exception e) {
                    runOnUiThread(() ->
                            appendLog("  시도 실패: " + e.getClass().getSimpleName()));
                }
            }

            long binderAvg = binderTotal / trials;
            long bAvg = binderAvg;
            runOnUiThread(() -> appendLog("  → 평균: " + bAvg + "ms"));

            // === Pipe(FD) ===
            runOnUiThread(() -> appendLog("\n[Pipe(FD) 전달 — " + testSizeKB + "KB × " + trials + "회]"));

            long pipeTotal = 0;
            for (int i = 0; i < trials; i++) {
                try {
                    long start = System.currentTimeMillis();

                    ParcelFileDescriptor pfd = dataService.getLargeDataSafe(testSizeKB);
                    InputStream is = new ParcelFileDescriptor.AutoCloseInputStream(pfd);
                    ByteArrayOutputStream bos = new ByteArrayOutputStream();
                    byte[] buf = new byte[64 * 1024];
                    int len;
                    while ((len = is.read(buf)) != -1) {
                        bos.write(buf, 0, len);
                    }
                    is.close();

                    long elapsed = System.currentTimeMillis() - start;
                    pipeTotal += elapsed;

                    int n = i + 1;
                    int received = bos.size() / 1024;
                    runOnUiThread(() ->
                            appendLog(String.format("  시도 %d: %dms (%dKB)",
                                    n, elapsed, received)));
                } catch (Exception e) {
                    runOnUiThread(() ->
                            appendLog("  시도 실패: " + e.getMessage()));
                }
            }

            long pipeAvg = pipeTotal / trials;
            long pAvg = pipeAvg;

            runOnUiThread(() -> {
                appendLog("  → 평균: " + pAvg + "ms");
                appendLog("");
                appendLog("═══ 비교 결과 ═══");
                appendLog("  Binder 직접: 평균 " + bAvg + "ms (500KB 이하만 가능)");
                appendLog("  Pipe(FD):    평균 " + pAvg + "ms (크기 제한 없음)");
                appendLog("");

                if (bAvg < pAvg) {
                    appendLog("  → 소량 데이터는 Binder가 더 빠름");
                    appendLog("  → 하지만 Binder는 ~1MB 초과 불가!");
                } else {
                    appendLog("  → Pipe가 더 빠르거나 비슷함");
                }
                appendLog("  → 대용량 데이터는 항상 Pipe/FD를 사용하세요");
                appendLog("");
            });
        }, "CompareThread").start();
    }

    // ==============================
    // 로그 출력 헬퍼
    // ==============================
    private void appendLog(String msg) {
        logText.append(msg + "\n");
        scrollView.post(() -> scrollView.fullScroll(ScrollView.FOCUS_DOWN));
    }
}
```

---

## Step 7: AndroidManifest.xml 수정

**파일:** `app/src/main/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="BinderLimitLab"
        android:supportsRtl="true"
        android:theme="@style/Theme.BinderLimitLab"
        tools:targetApi="36">

        <!-- MainActivity -->
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- ★ DataService — 별도 프로세스에서 실행! -->
        <service
            android:name=".DataService"
            android:exported="true"
            android:process=":data_remote">
            <intent-filter>
                <action android:name="com.example.binderlimitlab.DATA_SERVICE" />
            </intent-filter>
        </service>

    </application>
</manifest>
```

---

## Step 8: 최종 파일 구조 확인

```
app/src/main/
├── aidl/
│   └── com/example/binderlimitlab/
│       └── IDataService.aidl        ← 인터페이스 (byte[] + PFD)
├── java/
│   └── com/example/binderlimitlab/
│       ├── DataService.java         ← 서비스 (별도 프로세스)
│       └── MainActivity.java        ← 클라이언트 (UI)
├── res/
│   └── layout/
│       └── activity_main.xml        ← UI 레이아웃
└── AndroidManifest.xml              ← 서비스 등록
```

총 **5개 파일**.

---

## Step 9: 빌드 및 실행

### 9-1. 빌드

```
Build → Make Project (Ctrl+F9)
```

### 9-2. 흔한 에러

| 에러 | 원인 | 해결 |
|------|------|------|
| `cannot find symbol: IDataService` | AIDL 빌드 안 됨 | `Build → Clean + Rebuild` |
| `cannot find symbol: ParcelFileDescriptor` | import 누락 | `import android.os.ParcelFileDescriptor;` |
| `cannot find symbol: ByteArrayOutputStream` | import 누락 | `import java.io.ByteArrayOutputStream;` |
| `AIDL File requires buildFeatures.aidl` | gradle 설정 누락 | Step 2 확인 |

### 9-3. 에뮬레이터에서 실행

```
Run → Run 'app' (Shift+F10)
```

---

## Step 10: 테스트 시나리오

### 테스트 1: Binder 직접 전달 (한계 찾기)

**"Binder 직접 전달 (크기 증가 테스트)"** 버튼 클릭.

예상 결과:

```
═══════════════════════════════════
★ Binder 직접 전달 테스트
  크기를 점진적으로 증가시켜 한계를 찾습니다
═══════════════════════════════════
  ✅   10KB: 성공! (10KB 수신, 2ms)
  ✅   50KB: 성공! (50KB 수신, 3ms)
  ✅  100KB: 성공! (100KB 수신, 4ms)
  ✅  200KB: 성공! (200KB 수신, 5ms)
  ✅  400KB: 성공! (400KB 수신, 8ms)
  ✅  600KB: 성공! (600KB 수신, 12ms)
  ✅  800KB: 성공! (800KB 수신, 15ms)
  💥  900KB: 실패! → TransactionTooLargeException
     └ data parcel size 921600 bytes
  💥 1000KB: 실패! → TransactionTooLargeException
     └ data parcel size 1024000 bytes
  💥 1024KB: 실패! → TransactionTooLargeException
     └ data parcel size 1048576 bytes
  💥 2048KB: 실패! → TransactionTooLargeException
     └ data parcel size 2097152 bytes

결론: Binder 버퍼 ~1MB 초과 시 예외 발생!
  → 대용량 데이터는 Pipe/FD로 전달해야 함
```

> ⚠ 정확한 한계점은 **~800KB~900KB** 사이입니다. Binder 버퍼는 프로세스당 약 1MB이지만, Parcel 헤더와 다른 동시 트랜잭션이 공유하므로 순수 데이터는 ~800KB 정도에서 실패할 수 있습니다.

### 테스트 2: Pipe(FD) 전달 (크기 제한 없음)

**"Pipe(FD) 전달 (크기 제한 없음)"** 버튼 클릭.

예상 결과:

```
═══════════════════════════════════
★ Pipe(FD) 전달 테스트
  FD만 Binder로 전달 → 크기 제한 없음!
═══════════════════════════════════
  ✅  100KB: 성공! (100KB 수신, 5ms)
  ✅  500KB: 성공! (500KB 수신, 10ms)
  ✅ 1024KB: 성공! (1024KB 수신, 18ms)    ← Binder에서는 실패!
  ✅ 2048KB: 성공! (2048KB 수신, 30ms)    ← 2MB도 성공!
  ✅ 4096KB: 성공! (4096KB 수신, 55ms)    ← 4MB도 성공!
  ✅ 8192KB: 성공! (8192KB 수신, 105ms)   ← 8MB도 성공!

결론: Pipe(FD)는 8MB도 문제없이 전달!
  → Binder로는 FD(4바이트)만 전달
  → 실제 데이터는 커널 Pipe 버퍼로 전송
```

> **핵심:** Binder에서 **1MB**조차 전달 못했는데, Pipe로는 **8MB**도 문제없습니다!

### 테스트 3: 성능 비교 (동일 크기)

**"비교 테스트 (Binder vs Pipe)"** 버튼 클릭.

예상 결과:

```
═══════════════════════════════════
★ 성능 비교: Binder 직접 vs Pipe(FD)
  동일 크기(500KB)로 비교합니다
═══════════════════════════════════

[Binder 직접 전달 — 500KB × 3회]
  시도 1: 10ms (500KB)
  시도 2: 8ms (500KB)
  시도 3: 9ms (500KB)
  → 평균: 9ms

[Pipe(FD) 전달 — 500KB × 3회]
  시도 1: 12ms (500KB)
  시도 2: 11ms (500KB)
  시도 3: 10ms (500KB)
  → 평균: 11ms

═══ 비교 결과 ═══
  Binder 직접: 평균 9ms (500KB 이하만 가능)
  Pipe(FD):    평균 11ms (크기 제한 없음)

  → 소량 데이터는 Binder가 더 빠름
  → 하지만 Binder는 ~1MB 초과 불가!
  → 대용량 데이터는 항상 Pipe/FD를 사용하세요
```

> **관찰:** 500KB에서는 Binder가 약간 더 빠릅니다 (mmap 기반 1회 복사 vs Pipe의 커널 버퍼 경유). 하지만 Binder는 1MB를 넘을 수 없으므로, **대용량은 무조건 Pipe/FD**입니다.

### 테스트 4: 프로세스 분리 확인

```bash
adb shell "ps -A -o PID,NAME | grep binderlimit"
```

```
12345 com.example.binderlimitlab
12346 com.example.binderlimitlab:data_remote
```

### 테스트 5: Logcat 확인

Android Studio Logcat에서 필터: `tag:DataService OR tag:LimitClient`

```
D/DataService: ═══════════════════════════════════
D/DataService: DataService CREATED (PID: 12346)
D/DataService: ═══════════════════════════════════
D/DataService: getLargeData(100KB) — Binder 직접 전달 시도...
D/DataService: getLargeData() — 100KB 데이터 생성 완료, 반환 중...
D/DataService: getLargeData(900KB) — Binder 직접 전달 시도...
D/DataService: getLargeData() — 900KB 데이터 생성 완료, 반환 중...
    ← 이 시점에서 Binder Transaction 실패!

D/DataService: getLargeDataSafe(8192KB) — Pipe 전달 시도...
D/DataService: getLargeDataSafe() — 8192KB Pipe 전송 완료
    ← Pipe는 성공!
```

---

## 핵심 학습 정리

| 관찰 | Binder 개념 |
|------|------------|
| ~800KB에서 실패 시작 | Binder 버퍼 ~1MB를 헤더 + 동시 트랜잭션과 공유 |
| `TransactionTooLargeException` | Parcel 직렬화 크기가 버퍼 초과 시 발생 |
| Pipe(FD)는 8MB도 성공 | **FD(4바이트)**만 Binder로 전달, 데이터는 커널 Pipe |
| 500KB에서 Binder가 약간 더 빠름 | mmap 1회 복사 vs Pipe 커널 버퍼 경유 |
| 대용량은 항상 Pipe/FD | ContentProvider의 CursorWindow도 동일 원리 (FD 전달) |

---

## 현업에서의 활용

| 상황 | 잘못된 방법 | 올바른 방법 |
|------|-----------|-----------|
| Bitmap을 Intent로 전달 | `intent.putExtra("img", bitmap)` | FileProvider URI 전달 |
| 대용량 리스트 반환 | AIDL `List<Item> getAll()` | ContentProvider + 페이징 |
| 파일 공유 | `intent.putExtra("data", bytes)` | `ParcelFileDescriptor` |
| onSaveInstanceState | Bundle에 대량 데이터 저장 | ViewModel에 보관, ID만 저장 |
| 대량 Parcelable | 한 번에 수천 개 전달 | 청크 분할 또는 DB + URI |

---

## 전송 방식 비교 요약

```
[방법 1: Binder 직접]
Client → Parcel 직렬화 → Binder Driver → copy_from_user()
  → 커널 버퍼 (≤1MB!) → mmap → Server Parcel 역직렬화
  
  장점: 빠름 (1회 복사)
  단점: ★ 1MB 제한!

[방법 2: Pipe/FD]
Client → Binder Driver → FD 전달 (4바이트!)
Server → Pipe에 데이터 write (크기 무제한)
Client → Pipe에서 데이터 read

  장점: ★ 크기 제한 없음!
  단점: 약간 느림 (커널 Pipe 버퍼 경유)

[방법 3: ContentProvider + CursorWindow]
Client → ContentResolver.query() → Binder IPC
Server → Cursor + CursorWindow(공유 메모리 FD) 반환
Client → FD를 mmap하여 직접 읽기

  장점: 대량 구조화 데이터에 최적
  단점: CRUD API만 지원
```

---

*이 실습을 완료하면 Binder 1MB 제한의 실체와 ParcelFileDescriptor를 활용한 대용량 전송 패턴을 코드 레벨에서 이해할 수 있습니다.*

---

---

## 📋 실습 체크리스트

### 수강생 확인 사항

| 실습 | 완료 기준 | 체크 |
|------|-----------|------|
| 실습 1 | Zygote PID, system_server PID, 서비스 개수 기록 | ☐ |
| 실습 2 | Binder 트랜잭션 실시간 로그 관찰 완료 | ☐ |
| 실습 3 | 별도 프로세스 서비스 바인딩 + 계산 동작 확인 | ☐ |
| 실습 4 | 서비스 kill → DeathRecipient 콜백 → 자동 복구 확인 | ☐ |
| 실습 5 | 동기(2초 블로킹) vs 비동기(0ms 반환) 시간 비교 | ☐ |
| 실습 6 | Messenger 양방향 메시지 교환 동작 확인 | ☐ |
| 실습 7 | adb content 명령으로 Provider CRUD 확인 | ☐ |
| 실습 8 | TransactionTooLargeException 재현 + FD로 해결 | ☐ |

---

*시간이 부족할 경우 실습 1~4는 필수, 5~8은 선택으로 운영하세요.*
