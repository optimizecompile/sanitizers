# Sanitizers 工具链完全指南

> 本文档汇总自 [google/sanitizers](https://github.com/google/sanitizers) GitHub Wiki（共 78 页）及仓库本地文档，旨在为不同层次的开发者提供一份内容详尽、受众广泛的参考手册。
>
> **仓库状态**：该仓库已归档（Archived），核心代码已迁移至 [LLVM](http://llvm.org) 项目。本仓库保留用于存档历史文档、修复和辅助代码。

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. AddressSanitizer (ASan)](#2-addresssanitizer-asan)
  - [2.1 简介](#21-简介)
  - [2.2 检测的 bug 类型](#22-检测的-bug-类型)
  - [2.3 使用方法](#23-使用方法)
  - [2.4 算法原理](#24-算法原理)
  - [2.5 平台支持](#25-平台支持)
  - [2.6 编译标志](#26-编译标志)
  - [2.7 运行时标志](#27-运行时标志)
  - [2.8 性能数据](#28-性能数据)
- [3. LeakSanitizer (LSan)](#3-leaksanitizer-lsan)
- [4. ThreadSanitizer (TSan) — C/C++](#4-threadsanitizer-tsan--cc)
- [5. ThreadSanitizer (TSan) — Go](#5-threadsanitizer-tsan--go)
- [6. MemorySanitizer (MSan)](#6-memorysanitizer-msan)
- [7. HWAddressSanitizer (HWASan)](#7-hwaddresssanitizer-hwasan)
- [8. GWP-ASan](#8-gwp-asan)
- [9. MTE (Memory Tagging Extension)](#9-mte-memory-tagging-extension)
- [10. 通用运行时标志](#10-通用运行时标志)
- [11. 实践指南](#11-实践指南)
- [12. 常见问题 (FAQ)](#12-常见问题-faq)
- [13. 参考资源](#13-参考资源)

---

## 1. 项目概述

`sanitizers` 是 Google 开发的一组 C/C++（及 Go）内存错误检测工具的伞形项目。最初托管在 Google Code 上，包含三个独立项目：

- **address-sanitizer** — 内存地址错误检测
- **thread-sanitizer** — 数据竞争检测
- **memory-sanitizer** — 未初始化内存检测

随着项目发展，核心代码已迁移至 LLVM 编译器基础设施。当前仓库保留了部分基础设施和文档。

### 工具一览

| 工具 | 全称 | 检测目标 | 性能开销 |
|------|------|----------|----------|
| **ASan** | AddressSanitizer | 内存地址错误（越界、释放后使用等） | ~2x 减速 |
| **LSan** | LeakSanitizer | 内存泄漏 | 极低（仅退出时检测） |
| **TSan** | ThreadSanitizer | 数据竞争和死锁 | 2-20x 减速，5-10x 内存 |
| **MSan** | MemorySanitizer | 未初始化内存读取 | ~3x 减速 |
| **HWASan** | HWAddressSanitizer | 内存地址错误（基于硬件标签） | ~2x 减速，内存更少 |
| **GWP-ASan** | GWP-ASan | 堆内存错误（抽样检测） | 极低（抽样） |
| **UBSan** | UndefinedBehaviorSanitizer | 未定义行为 | 视启用检查项而定 |

---

## 2. AddressSanitizer (ASan)

### 2.1 简介

AddressSanitizer（简称 ASan）是一个快速的 C/C++ 内存错误检测器。它由两部分组成：

1. **编译器插桩模块**（LLVM pass）— 在编译时为每次内存访问插入检查代码
2. **运行时库** — 替换 `malloc`/`free` 函数，管理影子内存

### 2.2 检测的 bug 类型

ASan 能检测以下内存错误：

| Bug 类型 | 说明 |
|----------|------|
| **Use after free** | 释放后使用（悬空指针解引用） |
| **Heap buffer overflow** | 堆缓冲区溢出 |
| **Stack buffer overflow** | 栈缓冲区溢出 |
| **Global buffer overflow** | 全局缓冲区溢出 |
| **Use after return** | 函数返回后使用栈变量 |
| **Use after scope** | 变量作用域结束后使用 |
| **Initialization order bugs** | 初始化顺序问题 |
| **Memory leaks** | 内存泄漏（通过集成 LSan） |

### 2.3 使用方法

**编译命令：**

```bash
# 基本用法
clang -fsanitize=address -O1 -fno-omit-frame-pointer -g example.c -o example

# C++ 程序
clang++ -fsanitize=address -O1 -fno-omit-frame-pointer -g example.cpp -o example
```

**关键编译选项说明：**

| 标志 | 作用 |
|------|------|
| `-fsanitize=address` | 启用 AddressSanitizer |
| `-O1` 或更高 | 优化级别（推荐 O1 以获得合理性能） |
| `-fno-omit-frame-pointer` | 保留帧指针，获得更好的错误堆栈 |
| `-g` | 生成调试信息，错误报告中包含行号 |
| `-fno-common` | 不将 C 全局变量视为 common 变量（允许 ASan 插桩） |

**示例：Use-After-Free**

```c
// use-after-free.c
#include <stdlib.h>
int main() {
    char *x = (char*)malloc(10 * sizeof(char*));
    free(x);
    return x[5];  // BUG: 释放后使用
}
```

编译并运行：

```bash
clang -fsanitize=address -O1 -fno-omit-frame-pointer -g use-after-free.c
./a.out
```

**输出示例：**

```
==9901==ERROR: AddressSanitizer: heap-use-after-free on address 0x60700000dfb5
READ of size 1 at 0x60700000dfb5 thread T0
    #0 0x45917a in main use-after-free.c:5
    ...
0x60700000dfb5 is located 5 bytes inside of 80-byte region [0x60700000dfb0,0x60700000e000)
freed by thread T0 here:
    #0 0x4441ee in __interceptor_free
    #1 0x45914a in main use-after-free.c:4
```

### 2.4 算法原理

ASan 的核心是**影子内存（Shadow Memory）**机制：

**内存映射：**

- 虚拟地址空间分为两部分：**主应用内存**（Mem）和**影子内存**（Shadow）
- 每 8 字节应用内存映射到 1 字节影子内存
- 影子值含义：
  - `0`：8 字节全部可寻址（未中毒）
  - 负数：8 字节全部中毒（不可寻址）
  - `k`（1-7）：前 k 字节可寻址，其余中毒

**映射公式：**

```
# 64 位系统
Shadow = (Mem >> 3) + 0x7fff8000;

# 32 位系统
Shadow = (Mem >> 3) + 0x20000000;
```

**插桩原理：**

每次内存访问前，编译器插入检查代码：

```c
// 原始代码
*address = ...;

// 插桩后
shadow_address = MemToShadow(address);
if (ShadowIsPoisoned(shadow_address)) {
    ReportError(address, kAccessSize, kIsWrite);
}
*address = ...;
```

**栈保护：**

为检测栈缓冲区溢出，ASan 在栈变量周围插入红色区域（redzone）：

```
原始栈布局:          插桩后栈布局:
                     [redzone1 - 32 bytes]
char a[8];           [a - 8 bytes]
                     [redzone2 - 24 bytes]
                     [redzone3 - 32 bytes]
```

**堆保护：**

- `malloc` 分配的内存周围添加红色区域，对应影子值设为中毒
- `free` 的内存整体中毒并放入隔离队列（quarantine），防止被立即重新分配

### 2.5 平台支持

| 操作系统 | x86 | x86_64 | ARM | ARM64 | MIPS | MIPS64 | PowerPC | PowerPC64 |
|----------|-----|--------|-----|-------|------|--------|---------|-----------|
| Linux | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| OS X | ✅ | ✅ | | | | | | |
| iOS 模拟器 | ✅ | ✅ | | | | | | |
| FreeBSD | ✅ | ✅ | | | | | | |
| Android | ✅ | ✅ | ✅ | ✅ | | | | |

### 2.6 编译标志

| 标志 | 说明 |
|------|------|
| `-fsanitize=address` | 启用 ASan |
| `-fno-omit-frame-pointer` | 保留帧指针 |
| `-fsanitize-blacklist=path` | 传入黑名单文件，跳过指定函数的插桩 |
| `-fno-common` | 允许 ASan 插桩全局变量 |
| `-mllvm -asan-stack=1` | 检测栈对象溢出（默认开启） |
| `-mllvm -asan-globals=1` | 检测全局对象溢出（默认开启） |

**关闭特定函数的插桩：**

```c
// 使用属性禁用特定函数的检测
__attribute__((no_sanitize("address")))
void skip_asan_check() {
    // 第三方库调用或特殊代码
}
```

### 2.7 运行时标志

通过 `ASAN_OPTIONS` 环境变量配置：

```bash
ASAN_OPTIONS=verbosity=1:malloc_context_size=20 ./a.out
```

也可在源码中嵌入默认选项：

```c
const char *__asan_default_options() {
    return "verbosity=1:malloc_context_size=20";
}
```

**关键运行时标志：**

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `quarantine_size_mb` | 256 (移动端 16) | 隔离区大小（MB），用于检测 use-after-free |
| `redzone` | 16 | 堆对象周围红色区域最小字节数 |
| `max_redzone` | 2048 | 堆对象周围红色区域最大字节数 |
| `detect_stack_use_after_return` | false | 启用栈返回后使用检测 |
| `check_initialization_order` | false | 检测初始化顺序问题 |
| `detect_container_overflow` | true | 检测容器溢出 |
| `detect_odr_violation` | 2 | 检测 ODR（单一定义规则）违反 |
| `halt_on_error` | true | 首次错误后崩溃（需配合 `-fsanitize-recover=address`） |
| `alloc_dealloc_mismatch` | true | 报告 malloc/delete, new/free 不匹配 |
| `strict_string_checks` | false | 检查字符串参数是否正确以 null 结尾 |
| `replace_str` | true | 替换 libc 字符串函数以发现更多错误 |
| `replace_intrin` | true | 替换 memset/memcpy/memmove 内联函数 |
| `start_deactivated` | false | 降低初始内存消耗（主要用于 Android） |
| `sleep_before_dying` | 0 | 打印错误后等待的秒数（便于附加 gdb） |

查看所有支持的标志：

```bash
ASAN_OPTIONS=help=1 ./a.out
```

### 2.8 性能数据

- 平均减速：**~2x**
- 内存开销：**~2x-3x**
- 对比 Valgrind/Memcheck：ASan 快 **10-20 倍**

---

## 3. LeakSanitizer (LSan)

### 3.1 简介

LeakSanitizer（简称 LSan）是集成在 ASan 中的内存泄漏检测器。它在进程退出时执行额外的泄漏检测阶段。

- **x86_64 Linux**：默认启用
- **x86_64 OS X**：通过 `ASAN_OPTIONS=detect_leaks=1` 启用
- 也可独立使用（`-fsanitize=leak`），无需 ASan 插桩

### 3.2 使用方法

```c
// memory-leak.c
#include <stdlib.h>
void *p;
int main() {
    p = malloc(7);
    p = 0;  // 内存泄漏
    return 0;
}
```

```bash
clang -fsanitize=address -g memory-leak.c
./a.out
```

**输出示例：**

```
==7829==ERROR: LeakSanitizer: detected memory leaks
Direct leak of 7 byte(s) in 1 object(s) allocated from:
    #0 0x42c0c5 in __interceptor_malloc
    #1 0x43ef81 in main memory-leak.c:6
SUMMARY: AddressSanitizer: 7 byte(s) leaked in 1 allocation(s).
```

### 3.3 独立模式

如果只需要泄漏检测，不需要 ASan 的完整开销：

```bash
clang -fsanitize=leak -g memory-leak.c
```

### 3.4 LSan 标志

通过 `LSAN_OPTIONS` 环境变量配置：

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `exitcode` | 23 | 检测到泄漏时的退出码 |
| `max_leaks` | 0 | 报告的泄漏数量上限（0 = 全部） |
| `suppressions` | (无) | 抑制规则文件路径 |
| `print_suppressions` | 1 | 打印匹配的抑制统计 |
| `report_objects` | 0 | 报告泄漏对象地址 |
| `use_unaligned` | 0 | 查找指针时是否考虑非对齐模式 |

### 3.5 抑制已知泄漏

创建抑制文件：

```
# suppr.txt
# 这是一个已知的泄漏
leak:FooBar
```

运行时指定：

```bash
ASAN_OPTIONS=detect_leaks=1 LSAN_OPTIONS=suppressions=suppr.txt ./a.out
```

`^` 和 `$` 可匹配字符串的开头和结尾。

---

## 4. ThreadSanitizer (TSan) — C/C++

### 4.1 简介

ThreadSanitizer（简称 TSan）是一个 C/C++ 数据竞争检测器。数据竞争是并发系统中最常见且最难调试的 bug 类型之一——当两个线程同时访问同一变量且至少有一个是写操作时，即发生数据竞争。C++11 标准将数据竞争定义为**未定义行为**。

### 4.2 数据竞争示例

```cpp
// simple_race.cc
#include <pthread.h>
int Global;

void *Thread1(void *x) {
    Global++;
    return NULL;
}

void *Thread2(void *x) {
    Global--;
    return NULL;
}

int main() {
    pthread_t t[2];
    pthread_create(&t[0], NULL, Thread1, NULL);
    pthread_create(&t[1], NULL, Thread2, NULL);
    pthread_join(t[0], NULL);
    pthread_join(t[1], NULL);
}
```

### 4.3 使用方法

```bash
# 编译并链接
clang++ -fsanitize=thread -O2 -g simple_race.cc -o simple_race

# 运行
./simple_race
```

**输出示例：**

```
==================
WARNING: ThreadSanitizer: data race (pid=26327)
  Write of size 4 at 0x7f89554701d0 by thread T1:
    #0 Thread1(void*) simple_race.cc:8

  Previous write of size 4 at 0x7f89554701d0 by thread T2:
    #0 Thread2(void*) simple_race.cc:13

  Thread T1 (tid=26328, running) created at:
    #0 pthread_create
    #1 main simple_race.cc:19
==================
```

### 4.4 平台支持

| 操作系统 | 支持的架构 |
|----------|-----------|
| Linux | x86_64, mips64, aarch64, powerpc64 |
| macOS | x86_64, aarch64 |
| FreeBSD | x86_64 |
| NetBSD | x86_64 |

### 4.5 算法原理

TSan（v2）由插桩模块和运行时库组成：

**插桩：**
- 对每次内存访问插入函数调用（如 `__tsan_read4(addr)`）
- 跳过已知无竞争的访问（如常量全局变量读取、不逃逸的局部变量访问）

**影子状态：**
- 每个对齐的 8 字节应用内存映射到 N 个影子字（N = 2/4/8）
- 每个影子字（64 位）包含：
  - TID（线程 ID）：16 位
  - 标量时钟：42 位
  - IsWrite：1 位
  - 访问大小（1/2/4/8）：2 位
  - 地址偏移（0..7）：3 位

**状态机：**
- 每次内存访问时，线程时钟递增
- 遍历所有影子字，检查是否存在竞争
- 若两个不同线程的访问之间没有 happens-before 关系，则报告竞争

### 4.6 运行时开销

| 指标 | 开销 |
|------|------|
| 内存使用 | 5-10x |
| 执行时间 | 2-20x |

### 4.7 抑制报告

当无法修复某些竞争（如第三方代码）时：

1. **抑制文件**（运行时机制）— 参见 [ThreadSanitizerSuppressions](https://github.com/google/sanitizers/wiki/ThreadSanitizerSuppressions)
2. **黑名单文件**（编译时机制）
3. **条件编译**：

```cpp
#if defined(__has_feature) && __has_feature(thread_sanitizer)
// TSan 下跳过此代码
#endif
```

### 4.8 重要限制

- **所有代码必须用 `-fsanitize=thread` 编译**，否则可能产生误报/漏报
- 不支持 libc/libstdc++ 静态链接
- 不支持 C++ 异常
- TSan 映射大量虚拟地址空间（不是实际占用），`ulimit` 可能表现异常
- **不能与 ASan 同时使用**

---

## 5. ThreadSanitizer (TSan) — Go

### 5.1 简介

Go 内置了数据竞争检测器，使用非常简单——只需在 go 命令后添加 `-race` 标志。

### 5.2 使用方法

```bash
# 测试包
go test -race mypkg

# 运行源文件
go run -race mysrc.go

# 构建命令
go build -race mycmd

# 安装包
go install -race mypkg
```

### 5.3 数据竞争示例

```go
func main() {
    c := make(chan bool)
    m := make(map[string]string)
    go func() {
        m["1"] = "a"  // 第一个冲突访问
        c <- true
    }()
    m["2"] = "b"  // 第二个冲突访问
    <-c
    for k, v := range m {
        fmt.Println(k, v)
    }
}
```

### 5.4 运行时开销

| 指标 | 开销 |
|------|------|
| 内存使用 | 5-10x |
| 执行时间 | 2-20x |

---

## 6. MemorySanitizer (MSan)

### 6.1 简介

MemorySanitizer（简称 MSan）是一个 C/C++ 未初始化内存读取检测器。它检测栈或堆分配的内存在写入之前被读取的情况。

**核心特点：**
- **位精确**：可以跟踪位字段中未初始化的位
- 容忍未初始化内存的复制和简单逻辑/算术运算
- 当未初始化值影响程序执行时才报告警告

### 6.2 报告条件

MSan 在以下情况报告警告：
1. 未初始化值用于条件分支
2. 未初始化指针用于内存访问
3. 未初始化值作为函数参数传递或返回（可通过 `-fno-sanitize-memory-param-retval` 禁用）
4. 未初始化数据传入某些 libc 调用

### 6.3 使用方法

```cpp
// umr.cc
#include <stdio.h>
int main(int argc, char** argv) {
    int* a = new int[10];
    a[5] = 0;
    if (a[argc])    // BUG: a[argc] 可能未初始化
        printf("xx\n");
    return 0;
}
```

```bash
clang -fsanitize=memory -fPIE -pie -fno-omit-frame-pointer -g -O2 umr.cc
./a.out
```

**输出示例：**

```
==6726== WARNING: MemorySanitizer: UMR (uninitialized-memory-read)
    #0 0x7fd1c2944171 in main umr.cc:6
```

### 6.4 来源追踪

使用 `-fsanitize-memory-track-origins` 可追踪未初始化值的来源：

```bash
clang -fsanitize=memory -fsanitize-memory-track-origins -fPIE -pie \
      -fno-omit-frame-pointer -g -O2 umr.cc
```

输出会包含来源信息：

```
==6726== WARNING: MemorySanitizer: UMR (uninitialized-memory-read)
    #0 0x7fd1c2944171 in main umr.cc:6
  ORIGIN: heap allocation:
    #0 0x7f5872b6a31b in operator new[](unsigned long)
    #1 0x7f5872b62151 in main umr.cc:4
```

> 注意：来源追踪会带来额外的 1.5x-2.5x 减速。

### 6.5 平台支持

- x86_64、AArch64、PPC64、MIPS64
- 自 Clang 4.0 起广泛可用

### 6.6 关键要求

**所有代码（包括使用的库，特别是 C++ 标准库）都必须用 MSan 编译。** 参见 [MemorySanitizerLibcxxHowTo](https://github.com/google/sanitizers/wiki/MemorySanitizerLibcxxHowTo)。

### 6.7 编程接口

```c
#include <sanitizer/msan_interface.h>

// 打印内存范围的影子状态（0=已初始化，1=未初始化）
__msan_print_shadow(ptr, size);

// 将内存范围标记为已初始化（不改变实际内存内容）
__msan_unpoison(ptr, size);
```

### 6.8 符号化

设置 `MSAN_SYMBOLIZER_PATH` 环境变量指向 `llvm-symbolizer` 路径，或将其放入 `$PATH`。

---

## 7. HWAddressSanitizer (HWASan)

### 7.1 简介

HWASan（Hardware-assisted AddressSanitizer）是 ASan 的新变体，利用硬件内存标签技术（如 ARM 的 Top-Byte-Ignore / Memory Tagging Extension）实现更低内存消耗的内存错误检测。

**与 ASan 的对比：**

| 特性 | ASan | HWASan |
|------|------|--------|
| 检测机制 | 影子内存 + 红色区域 | 硬件内存标签 |
| 内存开销 | ~2-3x | 显著更低 |
| 堆 bug 检测 | ✅ | ✅ |
| 栈 bug 检测 | ✅ | 有限 |
| 全局变量检测 | ✅ | 有限 |
| 平台 | 广泛 | ARM64 为主 |

### 7.2 仓库中的相关资源

- **check_registers** — x86 硬件指针标签支持测试套件（测试 Intel LAM / AMD UAI）
- **QEMU 支持** — `run_in_qemu_with_lam.sh` 脚本用于在 QEMU 中运行 LAM
- **相关论文** — 仓库中包含多篇 HWASan 和内存标签相关的论文和演示文稿

### 7.3 参考文档

- [HWASan 设计文档](https://clang.llvm.org/docs/HardwareAssistedAddressSanitizerDesign.html)
- [Android NDK HWASan 指南](https://developer.android.com/ndk/guides/hwasan)

---

## 8. GWP-ASan

### 8.1 简介

GWP-ASan 是一个基于抽样的堆内存错误检测工具。它以极低的性能开销运行，适合在生产环境中使用。

**工作原理：**
- 在每次 `malloc` 时以极低概率（通常 1/5000）选择一个分配进行特殊监控
- 被选中的分配放置在单独的页中，两侧为不可访问的页（Guard Pages）
- 当访问这些被监控的分配时发生错误，会触发 SIGSEGV 并提供详细的错误报告

### 8.2 检测的 bug 类型

- Heap-use-after-free
- Heap-buffer-overflow
- Double-free
- Use-after-scope（部分情况）

### 8.3 仓库中的资源

仓库包含 GWP-ASan 的 ICSE 2024 论文（位于 `gwp-asan/icse2024/`），包括：
- 论文 PDF 和 LaTeX 源码
- 演示幻灯片

### 8.4 参考文档

- [Android NDK GWP-ASan 指南](https://developer.android.com/ndk/guides/gwp-asan)

---

## 9. MTE (Memory Tagging Extension)

### 9.1 简介

ARM MTE（Memory Tagging Extension）是 ARMv8.5 引入的硬件内存标签扩展。它为每个内存分配分配一个随机标签，并在指针的高位存储匹配标签，硬件在每次内存访问时自动检查标签是否匹配。

### 9.2 两种模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **同步模式 (SYNC)** | 标签不匹配时立即触发异常 | 开发/调试 |
| **异步模式 (ASYNC)** | 标签不匹配时记录，稍后报告 | 生产环境 |

### 9.3 仓库中的资源

仓库包含 MTE 动态划分（Dynamic Carveout）的原型实现：

- **硬件需求和操作系统设计草图** — [spec.md](file:///workspace/mte-dynamic-carveout/spec.md)
- **Linux 内核原型补丁** — [pcc/linux/tree/mte-dynamic](https://github.com/pcc/linux/tree/mte-dynamic)
- **QEMU 原型补丁** — [pcc/qemu/tree/mte-dynamic](https://github.com/pcc/qemu/tree/mte-dynamic)

**QEMU 使用：**

```bash
# 启用 MTE 共享分配
-machine virt,virtualization=on,mte=on,mte-shared-alloc=on
```

### 9.4 已知限制

- 不兼容 HW 标签 KASAN
- 不兼容 KVM 的 MTE 虚拟机
- 标记内存和非标记内存之间的自动页面迁移尚未实现

### 9.5 Android 示例应用

仓库包含一个 Android 示例应用（位于 `android/app/`），展示了如何使用以下工具构建应用：

1. HWASan
2. GWP-ASan
3. MTE（同步和异步模式）
4. 以上都不使用（对照组）

**安装预构建应用：**

```bash
adb install prebuilt-apks/app-<variant>-release.apk
```

**自行构建：**

```bash
cd src && ./gradlew build
```

---

## 10. 通用运行时标志

以下标志适用于所有 sanitizer 工具，通过对应的环境变量（`ASAN_OPTIONS`、`TSAN_OPTIONS`、`MSAN_OPTIONS`、`LSAN_OPTIONS`）配置：

### 10.1 符号化和堆栈

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `symbolize` | true | 将虚拟地址转换为文件/行位置 |
| `external_symbolizer_path` | "" | 外部符号化器路径 |
| `strip_path_prefix` | "" | 从错误报告的文件路径中去除此前缀 |
| `malloc_context_size` | 30 | 每次分配/释放保留的最大堆栈帧数 |
| `symbolize_inline_frames` | true | 打印堆栈中的内联帧 |

### 10.2 信号处理

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `handle_segv` | true | 自定义 SEGV 处理程序 |
| `handle_sigbus` | true | 自定义 SIGBUS 处理程序 |
| `handle_sigill` | true | 自定义 SIGILL 处理程序 |
| `handle_sigfpe` | true | 自定义 SIGFPE 处理程序 |
| `handle_abort` | false | 自定义 SIGABRT 处理程序 |
| `use_sigaltstack` | true | 使用备用栈处理信号 |

### 10.3 日志和输出

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `log_path` | stderr | 日志写入路径（支持 "stdout"、"stderr"） |
| `log_to_syslog` | false (Android/Darwin: true) | 同时写入 syslog |
| `verbosity` | 0 | 详细程度（0=静默，1+ 更多输出） |
| `color` | auto | 报告着色（always/never/auto） |
| `print_summary` | true | 打印错误摘要 |
| `exitcode` | 1 | 检测到错误时的退出码 |
| `abort_on_error` | false (Darwin: true) | 错误后调用 abort() 而非 _exit() |

### 10.4 内存限制

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `hard_rss_limit_mb` | 0 | 硬 RSS 限制（MB），超出则终止 |
| `soft_rss_limit_mb` | 0 | 软 RSS 限制，超出则 malloc 返回 NULL |
| `mmap_limit_mb` | 0 | mmap 内存限制（不含影子内存） |
| `allocator_may_return_null` | false | 内存不足时是否返回 NULL 而非崩溃 |

### 10.5 代码覆盖率

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `coverage` | false | 程序退出时转储覆盖率信息 |
| `coverage_dir` | "." | 覆盖率转储目录 |
| `coverage_direct` | false (Android: true) | 直接转储到内存映射文件 |
| `html_cov_report` | false | 生成 HTML 覆盖率报告 |

### 10.6 其他常用标志

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `detect_leaks` | true | 启用内存泄漏检测 |
| `leak_check_at_exit` | true | 退出时执行泄漏检查 |
| `detect_deadlocks` | true | 启用死锁检测 |
| `strict_string_checks` | false | 检查字符串参数是否正确 null 终止 |
| `intercept_strstr` | true | 自定义 strstr/strcasestr 包装器 |
| `intercept_memcmp` | true | 自定义 memcmp 包装器 |
| `disable_coredump` | true (64 位) | 禁用 core dump（避免 16T+ core 文件） |
| `help` | false | 打印所有可用标志 |

---

## 11. 实践指南

### 11.1 选择合适的工具

| 场景 | 推荐工具 |
|------|----------|
| 日常开发调试内存错误 | ASan |
| 检测内存泄漏 | LSan（独立模式或集成 ASan） |
| 多线程数据竞争 | TSan |
| 未初始化内存问题 | MSan |
| 生产环境极低开销检测 | GWP-ASan |
| ARM64 移动设备 | HWASan / MTE |
| 未定义行为检测 | UBSan |

### 11.2 构建系统集成

**CMake 集成示例：**

```cmake
# 创建专门的 sanitizer 构建选项
option(SANITIZE_ADDRESS "Enable AddressSanitizer" OFF)
option(SANITIZE_THREAD "Enable ThreadSanitizer" OFF)
option(SANITIZE_MEMORY "Enable MemorySanitizer" OFF)

if(SANITIZE_ADDRESS)
    add_compile_options(-fsanitize=address -fno-omit-frame-pointer -g)
    add_link_options(-fsanitize=address)
elseif(SANITIZE_THREAD)
    add_compile_options(-fsanitize=thread -fno-omit-frame-pointer -g)
    add_link_options(-fsanitize=thread)
elseif(SANITIZE_MEMORY)
    add_compile_options(-fsanitize=memory -fPIE -pie -fno-omit-frame-pointer -g)
    add_link_options(-fsanitize=memory -fPIE -pie)
endif()
```

**使用方式：**

```bash
# 启用 ASan 构建
cmake -DCMAKE_BUILD_TYPE=Debug -DSANITIZE_ADDRESS=ON ..
make

# 启用 TSan 构建
cmake -DCMAKE_BUILD_TYPE=Debug -DSANITIZE_THREAD=ON ..
make
```

### 11.3 CI/CD 集成

```yaml
# .github/workflows/sanitizer.yml 示例
jobs:
  asan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install Clang
        run: sudo apt-get install -y clang
      - name: Build with AddressSanitizer
        run: |
          cmake -DCMAKE_BUILD_TYPE=Debug -DSANITIZE_ADDRESS=ON .
          make
      - name: Run tests with ASan
        run: ./test_suite
        env:
          ASAN_OPTIONS: detect_leaks=1:halt_on_error=1
```

### 11.4 性能优化策略

1. **增量检测**：只对修改的模块启用 sanitizer
2. **合理设置隔离区大小**：降低 `quarantine_size_mb` 可减少内存使用
3. **关闭不需要的检测**：如 `detect_stack_use_after_return=false`（默认）
4. **使用 `-O1` 或更高优化级别**：平衡检测精度和性能
5. **生产环境使用 GWP-ASan**：以极低开销进行抽样检测

### 11.5 误报处理

```c
// 方法 1：使用属性禁用特定函数的检测
__attribute__((no_sanitize("address")))
void skip_asan_check() {
    // 第三方库调用
}

// 方法 2：使用抑制文件
// ASAN_OPTIONS=suppressions=suppr.txt ./a.out

// 方法 3：手动取消内存中毒
#include <sanitizer/asan_interface.h>
__asan_unpoison_memory_region(ptr, size);
```

---

## 12. 常见问题 (FAQ)

### Q: ASan 和 TSan 能同时使用吗？

**不能。** 它们使用不同的影子内存布局，互不兼容。需要分别构建和运行。

### Q: TSan 报告 "failed to restore the stack" 怎么办？

尝试增大 `history_size` 标志：

```bash
TSAN_OPTIONS=history_size=7 ./a.out
```

### Q: TSan 报告 "can not mmap the shadow memory" 怎么办？

启用 ASLR：

```bash
echo 2 >/proc/sys/kernel/randomize_va_space
```

在 GDB 下运行时：

```bash
gdb -ex 'set disable-randomization off' --args ./a.out
```

### Q: TSan 在 top 中显示使用数百 GB 内存？

不是实际占用。TSan 映射（但不保留）大量虚拟地址空间。`ulimit` 可能表现异常。

### Q: MSan 要求所有库都用 MSan 编译吗？

**是的。** 所有代码（包括 C++ 标准库）都必须用 MSan 编译，否则会产生误报。参见 [MemorySanitizerLibcxxHowTo](https://github.com/google/sanitizers/wiki/MemorySanitizerLibcxxHowTo)。

### Q: 如何在 GDB 下调试 MSan 问题？

```
set disable-randomization off
set overload-resolution off
br __msan_warning
br __msan_warning_noreturn
run
...
call __msan_print_shadow(&x, sizeof(x))
```

### Q: ASan 检测到 ODR 违反怎么办？

ODR（One-Definition-Rule）违反表示同一符号在不同编译单元中有不同定义。可以通过 `detect_odr_violation=0` 关闭检测，但建议修复根本原因。

### Q: 如何获取符号化的堆栈跟踪？

1. 编译时添加 `-g` 标志
2. 确保 `llvm-symbolizer` 在 `$PATH` 中
3. 或设置 `external_symbolizer_path` 指向符号化器路径
4. 使用 `strip_path_prefix` 去除路径前缀以获得更简洁的输出

### Q: 在哪里报告 bug？

该仓库已归档，不接受新的 bug 报告。请到对应的项目报告：

| 组件 | Bug 跟踪器 |
|------|-----------|
| LLVM (sanitizer 运行时和插桩) | [LLVM Bug Tracker](https://github.com/llvm/llvm-project/issues/) |
| GCC (sanitizer 移植) | [GCC Bugzilla](https://gcc.gnu.org/bugzilla/) |
| Linux 内核 (KASAN/KMSAN/KCSAN) | [Linux 内核邮件列表](https://vger.kernel.org/vger-lists.html#linux-kernel) |
| Android NDK | [NDK Issue Tracker](https://github.com/android/ndk) |
| Apple (Xcode) | Apple 开发者渠道 |
| Microsoft (Visual Studio) | Microsoft 开发者渠道 |

---

## 13. 参考资源

### 13.1 官方文档

| 资源 | 链接 |
|------|------|
| Sanitizers Wiki 首页 | [github.com/google/sanitizers/wiki](https://github.com/google/sanitizers/wiki) |
| AddressSanitizer | [wiki/AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) |
| ASan 算法 | [wiki/AddressSanitizerAlgorithm](https://github.com/google/sanitizers/wiki/AddressSanitizerAlgorithm) |
| ASan 标志 | [wiki/AddressSanitizerFlags](https://github.com/google/sanitizers/wiki/AddressSanitizerFlags) |
| LeakSanitizer | [wiki/AddressSanitizerLeakSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizerLeakSanitizer) |
| ThreadSanitizer (C++) | [wiki/ThreadSanitizerCppManual](https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual) |
| ThreadSanitizer (Go) | [wiki/ThreadSanitizerGoManual](https://github.com/google/sanitizers/wiki/ThreadSanitizerGoManual) |
| TSan 算法 | [wiki/ThreadSanitizerAlgorithm](https://github.com/google/sanitizers/wiki/ThreadSanitizerAlgorithm) |
| MemorySanitizer | [wiki/MemorySanitizer](https://github.com/google/sanitizers/wiki/MemorySanitizer) |
| 通用标志 | [wiki/SanitizerCommonFlags](https://github.com/google/sanitizers/wiki/SanitizerCommonFlags) |
| HWASan 设计 | [clang.llvm.org/docs/HardwareAssistedAddressSanitizerDesign](https://clang.llvm.org/docs/HardwareAssistedAddressSanitizerDesign.html) |
| UBSan | [clang.llvm.org/docs/UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html) |

### 13.2 内核 Sanitizer 文档

| 工具 | 链接 |
|------|------|
| KASAN | [kernel.org/doc/html/v4.12/dev-tools/kasan](https://www.kernel.org/doc/html/v4.12/dev-tools/kasan.html) |
| KMSAN | [github.com/google/kmsan](https://github.com/google/kmsan) |
| KCSAN | [github.com/google/kernel-sanitizers](https://github.com/google/kernel-sanitizers/blob/master/KCSAN.md) |

### 13.3 论文和演示

| 资源 | 链接 |
|------|------|
| ASan 论文 (USENIX ATC 2012) | [research.google.com/pub37752](https://research.google.com/pubs/pub37752.html) |
| MSan 论文 | [static.googleusercontent.com/43308.pdf](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43308.pdf) |
| ASan LLVM Dev Meeting 2011 | [Video](https://www.youtube.com/watch?v=CPnRS1nv3_s) / [Slides](http://llvm.org/devmtg/2011-11/Serebryany_FindingRacesMemoryErrors.pdf) |
| MSan LLVM Dev Meeting 2013 | [Slides](http://llvm.org/devmtg/2013-04/stepanov-slides.pdf) / [Video](http://llvm.org/devmtg/2013-04/videos/stepanov-hires.mov) |
| GWP-ASan 论文 (ICSE 2024) | 仓库内 `gwp-asan/icse2024/paper.pdf` |
| HWASan / 内存标签演示 | 仓库内 `hwaddress-sanitizer/` 目录 |

### 13.4 仓库本地文档

| 文档 | 路径 |
|------|------|
| 仓库主 README | [README.md](file:///workspace/README.md) |
| Android 示例应用 | [android/app/README.md](file:///workspace/android/app/README.md) |
| Dashboard 工具 | [dashboard/README.md](file:///workspace/dashboard/README.md) |
| check_registers 测试套件 | [hwaddress-sanitizer/check_registers/README.md](file:///workspace/hwaddress-sanitizer/check_registers/README.md) |
| MarkUs-GC | [hwaddress-sanitizer/MarkUs-GC.md](file:///workspace/hwaddress-sanitizer/MarkUs-GC.md) |
| MTE 动态划分 | [mte-dynamic-carveout/README.md](file:///workspace/mte-dynamic-carveout/README.md) |
| MTE 动态划分规格 | [mte-dynamic-carveout/spec.md](file:///workspace/mte-dynamic-carveout/spec.md) |

### 13.5 邮件列表

| 工具 | 邮件列表 |
|------|----------|
| AddressSanitizer | address-sanitizer@googlegroups.com |
| ThreadSanitizer | thread-sanitizer@googlegroups.com |
| MemorySanitizer | memory-sanitizer@googlegroups.com |

---

> **文档说明**：本文档基于 google/sanitizers GitHub Wiki（78 页）和仓库本地文档整理而成。由于仓库已归档，部分信息可能已过时。最新文档请参考 [LLVM 官方文档](https://clang.llvm.org/docs/)。
