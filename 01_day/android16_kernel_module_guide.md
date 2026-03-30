# Android 16 에뮬레이터 커널 모듈(.ko) 빌드 및 실행 가이드

> **전제 조건:** 에뮬레이터용 커널이 이미 빌드된 상태  
> **빌드 시스템:** Kleaf DDK (ddk_module) — Android 16에서 `kernel_module()`은 deprecated  
> **커널 타겟:** `//common-modules/virtual-device:virtual_device_x86_64`

---

## 📋 목차

1. [커스텀 커널 모듈 빌드 (ddk_module)](#1-커스텀-커널-모듈-빌드)
2. [에뮬레이터에서 커널 모듈 로드 및 실행](#2-에뮬레이터에서-커널-모듈-로드-및-실행)
3. [실전 예제: Binder 모니터링 모듈](#3-실전-예제-binder-모니터링-모듈)
4. [실전 예제: 프로세스 감시 모듈](#4-실전-예제-프로세스-감시-모듈)
5. [복수 모듈 동시 빌드](#5-복수-모듈-동시-빌드)
6. [트러블슈팅](#6-트러블슈팅)
7. [자동화 스크립트](#7-자동화-스크립트)
8. [핵심 명령어 요약](#8-핵심-명령어-요약)

---

## ⚠ Android 16 빌드 시스템 변경사항

```
kernel_module()  →  ❌ deprecated (Android 16에서 경고 + 빌드 실패)
ddk_module()     →  ✅ 신규 표준 (DDK: Driver Development Kit)
```

**`ddk_module()`의 장점:**
- **Kbuild/Makefile 자동 생성** — 수동 Kbuild 파일 불필요
- **아키텍처 헤더 자동 처리** — x86_64/arm64 혼동 방지
- **의존성 관리 개선** — `deps`로 다른 모듈/헤더 참조

---

## 1. 커스텀 커널 모듈 빌드

### 1-1. 모듈 소스 작성

```bash
cd ~/android-kernel

# 모듈 디렉토리 생성
mkdir -p my_modules/hello
```

**파일: `my_modules/hello/hello_android.c`**
```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/proc_fs.h>
#include <linux/uaccess.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Android Framework Training");
MODULE_DESCRIPTION("Hello Android 16 Kernel Module");
MODULE_VERSION("1.0");

static char msg_buf[256] = "Hello from Android 16 Kernel Module!\n";

/* /proc/hello_android 읽기 핸들러 */
static ssize_t hello_proc_read(struct file *file, char __user *buf,
                                size_t count, loff_t *pos)
{
    return simple_read_from_buffer(buf, count, pos,
                                   msg_buf, strlen(msg_buf));
}

static const struct proc_ops hello_proc_ops = {
    .proc_read = hello_proc_read,
};

static struct proc_dir_entry *proc_entry;

static int __init hello_android_init(void)
{
    pr_info("hello_android: Module loaded! (Android 16 Kernel)\n");

    /* /proc/hello_android 파일 생성 */
    proc_entry = proc_create("hello_android", 0444, NULL, &hello_proc_ops);
    if (!proc_entry) {
        pr_err("hello_android: Failed to create /proc entry\n");
        return -ENOMEM;
    }

    pr_info("hello_android: /proc/hello_android created successfully\n");
    return 0;
}

static void __exit hello_android_exit(void)
{
    if (proc_entry)
        proc_remove(proc_entry);

    pr_info("hello_android: Module unloaded. Goodbye!\n");
}

module_init(hello_android_init);
module_exit(hello_android_exit);
```

### 1-2. BUILD.bazel 작성 (ddk_module)

**파일: `my_modules/hello/BUILD.bazel`**
```python
load("//build/kernel/kleaf:kernel.bzl", "ddk_module")

ddk_module(
    name = "hello_android",
    srcs = ["hello_android.c"],
    out = "hello_android.ko",
    kernel_build = "//common-modules/virtual-device:virtual_device_x86_64",
    deps = [],
)
```

> ⚠ **핵심 포인트:**
> - `ddk_module`을 사용 (`kernel_module`이 아님!)
> - `outs` (리스트)가 아닌 `out` (단일 문자열)
> - **Kbuild, Makefile 파일을 만들지 마세요** — `ddk_module`이 자동 생성합니다
> - `kernel_build`는 반드시 `virtual_device_x86_64` 사용 (x86_64 아키텍처 헤더 정확히 적용)

### 1-3. 디렉토리 구조 확인

```bash
find my_modules/hello/ -type f
# my_modules/hello/BUILD.bazel       ← Bazel 빌드 정의
# my_modules/hello/hello_android.c   ← 소스 코드
# (총 2개 파일만! Kbuild/Makefile 없음!)
```

> ⚠ **Kbuild나 Makefile이 있으면 삭제하세요.** ddk_module과 충돌합니다.

### 1-4. 빌드

```bash
cd ~/android-kernel

tools/bazel build //my_modules/hello:hello_android \
    --check_visibility=false \
    --verbose_failures
```

> `--check_visibility=false`: `virtual_device_x86_64` 타겟 접근을 허용  
> `--verbose_failures`: 실패 시 상세 에러 출력

**영구적으로 visibility 체크를 비활성화하려면:**
```bash
echo "build --check_visibility=false" >> ~/android16-kernel/user.bazelrc
# 이후부터 플래그 없이:
tools/bazel build //my_modules/hello:hello_android
```

### 1-5. 빌드 결과물 확인

```bash
# .ko 파일 찾기
find bazel-bin/ -name "hello_android.ko"

# 모듈 정보 확인
$ sudo apt install kmod

KO=$(find bazel-bin/ -name "hello_android.ko" | head -1)

modinfo "$KO"
# filename:       hello_android.ko
# license:        GPL
# description:    Hello Android 16 Kernel Module
# author:         Android Framework Training
# vermagic:       6.12.x-android16-... SMP preempt mod_unload

$ sudo apt install file
file "$KO"
# hello_android.ko: ELF 64-bit LSB relocatable, x86-64, ...
```

> ⚠ **`vermagic`이 에뮬레이터 커널과 정확히 일치해야** `insmod`가 성공합니다. 동일 소스에서 커널과 모듈을 빌드했다면 자동으로 일치합니다.

---

## 2. 에뮬레이터에서 커널 모듈 로드 및 실행

### 2-1. 모듈 파일 전송

```bash
# 에뮬레이터가 실행 중인지 확인
adb devices
# emulator-5554   device

# root 권한 획득 (Google APIs 이미지)
adb root

# .ko 파일 경로 찾기 & 윈도우로 전송
KO=$(find ~/android-kernel/bazel-bin/ -name "hello_android.ko" | head -1)


# 에뮬레이터에 업로드
adb push "$KO" /data

# 전송 확인
adb shell ls -la /data/hello_android.ko
```

### 2-2. 모듈 로드 (insmod)

```bash
# 모듈 로드
adb shell insmod /data/hello_android.ko

# 로드 확인
adb shell lsmod
# Module                  Size  Used by
# hello_android           16384  0

# 커널 로그에서 모듈 메시지 확인
adb shell dmesg | grep hello_android
# [  xxx.xxxxxx] hello_android: Module loaded! (Android 16 Kernel)
# [  xxx.xxxxxx] hello_android: /proc/hello_android created successfully
```

### 2-3. 모듈 동작 테스트

```bash
# /proc/hello_android 읽기
adb shell cat /proc/hello_android
# Hello from Android 16 Kernel Module!
```

### 2-4. 모듈 언로드 (rmmod)

```bash
# 모듈 언로드
adb shell rmmod hello_android

# 언로드 확인
adb shell lsmod | grep hello
# (출력 없음 → 정상 언로드됨)

# 커널 로그 확인
adb shell dmesg | tail -3
# [  xxx.xxxxxx] hello_android: Module unloaded. Goodbye!

# /proc 엔트리도 제거됨
adb shell cat /proc/hello_android
# cat: /proc/hello_android: No such file or directory
```

---

## 3. 실전 예제: Binder 모니터링 모듈

Binder 관련 시스템 정보를 `/proc` 파일시스템에 노출하는 모듈입니다.

### 3-1. 소스 작성

```bash
mkdir -p my_modules/binder_mon
```

**파일: `my_modules/binder_mon/binder_monitor.c`**
```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/sched.h>
#include <linux/ktime.h>
#include <linux/utsname.h>

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Binder Transaction Monitor for Android 16");

static atomic_long_t read_count = ATOMIC_LONG_INIT(0);
static ktime_t load_time;

static int binder_mon_show(struct seq_file *m, void *v)
{
    ktime_t now = ktime_get();
    s64 uptime_ms = ktime_to_ms(ktime_sub(now, load_time));

    seq_printf(m, "=== Binder Monitor Module ===\n");
    seq_printf(m, "Kernel: %s %s\n",
               utsname()->release, utsname()->version);
    seq_printf(m, "Module uptime: %lld ms\n", uptime_ms);
    seq_printf(m, "Read count: %ld\n",
               atomic_long_inc_return(&read_count));
    seq_printf(m, "Current process: %s (PID %d, UID %d)\n",
               current->comm, current->pid,
               from_kuid_munged(current_user_ns(), current_cred()->uid));
    seq_printf(m, "\n[Tip] Binder 상태 확인:\n");
    seq_printf(m, "  cat /sys/kernel/debug/binder/state\n");
    seq_printf(m, "  cat /sys/kernel/debug/binder/stats\n");

    return 0;
}

static int binder_mon_open(struct inode *inode, struct file *file)
{
    return single_open(file, binder_mon_show, NULL);
}

static const struct proc_ops binder_mon_ops = {
    .proc_open    = binder_mon_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};

static int __init binder_mon_init(void)
{
    load_time = ktime_get();

    if (!proc_create("binder_monitor", 0444, NULL, &binder_mon_ops)) {
        pr_err("binder_monitor: Failed to create /proc entry\n");
        return -ENOMEM;
    }

    pr_info("binder_monitor: Module loaded. Read /proc/binder_monitor\n");
    return 0;
}

static void __exit binder_mon_exit(void)
{
    remove_proc_entry("binder_monitor", NULL);
    pr_info("binder_monitor: Module unloaded\n");
}

module_init(binder_mon_init);
module_exit(binder_mon_exit);
```

### 3-2. BUILD.bazel

**파일: `my_modules/binder_mon/BUILD.bazel`**
```python
load("//build/kernel/kleaf:kernel.bzl", "ddk_module")

ddk_module(
    name = "binder_monitor",
    srcs = ["binder_monitor.c"],
    out = "binder_monitor.ko",
    kernel_build = "//common-modules/virtual-device:virtual_device_x86_64",
    deps = [],
)
```

### 3-3. 빌드 및 테스트

```bash
# 빌드
tools/bazel build //my_modules/binder_mon:binder_monitor \
    --check_visibility=false

# 아래 파일 윈도우로 전송
KO=$(find bazel-bin/ -name "binder_monitor.ko" | head -1)

# 에뮬레이터로 전송
adb root
adb push binder_monitor.ko /data
adb shell insmod /data/binder_monitor.ko

# 테스트
adb shell cat /proc/binder_monitor
# === Binder Monitor Module ===
# Kernel: 6.12.x-android16-...
# Module uptime: 1234 ms
# Read count: 1
# Current process: cat (PID 12345, UID 0)
#
# [Tip] Binder 상태 확인:
#   cat /sys/kernel/debug/binder/state
#   cat /sys/kernel/debug/binder/stats

# 정리
adb shell rmmod binder_monitor
```

---

## 4. 실전 예제: 프로세스 감시 모듈

특정 프로세스의 시작/종료를 커널 레벨에서 감지하는 모듈입니다.

### 4-1. 소스 작성

```bash
mkdir -p my_modules/proc_watch
```

**파일: `my_modules/proc_watch/proc_watcher.c`**
```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/sched/signal.h>
#include <linux/sched/task.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/mm.h>
#include <linux/utsname.h>
#include <linux/ktime.h>

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Process Inspector for Android 16 Emulator");

static ktime_t load_time;
static atomic_long_t read_count = ATOMIC_LONG_INIT(0);

static int proc_watch_show(struct seq_file *m, void *v)
{
    struct task_struct *task;
    int total = 0, running = 0, sleeping = 0, stopped = 0, zombie = 0;
    int binder_threads = 0, render_threads = 0;
    ktime_t now = ktime_get();
    s64 uptime_ms = ktime_to_ms(ktime_sub(now, load_time));

    /* 헤더 */
    seq_printf(m, "=== Android Process Inspector ===\n");
    seq_printf(m, "Kernel: %s\n", utsname()->release);
    seq_printf(m, "Module uptime: %lld.%03lld s\n",
               uptime_ms / 1000, uptime_ms % 1000);
    seq_printf(m, "Read count: %ld\n\n",
               atomic_long_inc_return(&read_count));

    /* 프로세스 스캔 */
    rcu_read_lock();
    for_each_process(task) {
        total++;
        switch (task->__state) {
        case TASK_RUNNING:
            running++;
            break;
        case TASK_INTERRUPTIBLE:
        case TASK_UNINTERRUPTIBLE:
            sleeping++;
            break;
        case __TASK_STOPPED:
            stopped++;
            break;
        default:
            if (task->exit_state == EXIT_ZOMBIE)
                zombie++;
            break;
        }

        /* Binder/Render 스레드 카운트 */
        if (strncmp(task->comm, "Binder:", 7) == 0)
            binder_threads++;
        if (strcmp(task->comm, "RenderThread") == 0)
            render_threads++;
    }
    rcu_read_unlock();

    /* 통계 출력 */
    seq_printf(m, "--- Process Statistics ---\n");
    seq_printf(m, "  Total processes:  %d\n", total);
    seq_printf(m, "  Running:          %d\n", running);
    seq_printf(m, "  Sleeping:         %d\n", sleeping);
    seq_printf(m, "  Stopped:          %d\n", stopped);
    seq_printf(m, "  Zombie:           %d\n", zombie);
    seq_printf(m, "\n");
    seq_printf(m, "--- Android-specific ---\n");
    seq_printf(m, "  Binder threads:   %d\n", binder_threads);
    seq_printf(m, "  RenderThreads:    %d\n", render_threads);
    seq_printf(m, "\n");

    /* 앱 프로세스 목록 (com. 으로 시작하는 것만) */
    seq_printf(m, "--- App Processes ---\n");
    rcu_read_lock();
    for_each_process(task) {
        /* 메인 스레드만 (PID == TGID) */
        if (task->pid != task->tgid)
            continue;
        /* com. 으로 시작하는 Android 앱 */
        if (strncmp(task->comm, "com.", 4) == 0 ||
            strncmp(task->comm, "android.", 8) == 0) {
            seq_printf(m, "  PID %5d  UID %5d  %s\n",
                       task->pid,
                       from_kuid_munged(current_user_ns(),
                                        task_uid(task)),
                       task->comm);
        }
    }
    rcu_read_unlock();

    seq_printf(m, "\n--- Reader Info ---\n");
    seq_printf(m, "  Your process: %s (PID %d)\n",
               current->comm, current->pid);

    return 0;
}

static int proc_watch_open(struct inode *inode, struct file *file)
{
    return single_open(file, proc_watch_show, NULL);
}

static const struct proc_ops proc_watch_ops = {
    .proc_open    = proc_watch_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};

static int __init proc_watcher_init(void)
{
    load_time = ktime_get();

    if (!proc_create("proc_watcher", 0444, NULL, &proc_watch_ops)) {
        pr_err("proc_watcher: Failed to create /proc entry\n");
        return -ENOMEM;
    }

    pr_info("proc_watcher: Module loaded (process inspector mode)\n");
    return 0;
}

static void __exit proc_watcher_exit(void)
{
    remove_proc_entry("proc_watcher", NULL);
    pr_info("proc_watcher: Module unloaded\n");
}

module_init(proc_watcher_init);
module_exit(proc_watcher_exit);
```

### 4-2. BUILD.bazel

**파일: `my_modules/proc_watch/BUILD.bazel`**
```python
load("//build/kernel/kleaf:kernel.bzl", "ddk_module")

ddk_module(
    name = "proc_watcher",
    srcs = ["proc_watcher.c"],
    out = "proc_watcher.ko",
    kernel_build = "//common-modules/virtual-device:virtual_device_x86_64",
    deps = [],
)
```

### 4-3. 빌드 및 테스트

```bash
# 빌드
tools/bazel build //my_modules/proc_watch:proc_watcher \
    --check_visibility=false

# 윈도우 전송 
KO=$(find bazel-bin/ -name "proc_watcher.ko" | head -1)

# 에뮬레이터 업로드
adb root
adb push "$KO" /data/local/tmp/
adb shell insmod /data/local/tmp/proc_watcher.ko

# 에뮬레이터에서 앱을 열고 닫으면서 이벤트 관찰
adb shell cat /proc/proc_watcher
# === Process Watcher ===
# Total exits detected: 42
#
# --- Recent Events (latest 32) ---
#   [EXIT] com.android.settings (PID 3456)
#   [EXIT] sh (PID 3500)
#   ...

# 정리
adb shell rmmod proc_watcher
```

> ⚠ kprobe가 커널 설정에서 비활성화되어 있으면 등록 실패 로그가 나옵니다. 이 경우 커널 빌드 시 `CONFIG_KPROBES=y` defconfig fragment를 추가하세요.

---

