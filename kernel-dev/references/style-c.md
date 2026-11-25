# C 代码风格基线

风格的第一原则是**与周围代码一致**，不是“符合某个规范”。审阅时先读目标文件，提取它已有的约定，再判断新代码是否破坏了这些约定。

## 冲突解决顺序

1. 目标文件内部一致的写法；
2. 同目录其他文件的写法；
3. 同子系统/模块的写法；
4. 无任何参照时，用本文默认值（= Linux 内核风格）。

只在**同一文件内**出现两种写法时判为 P2；跨文件差异不判，除非影响可读性。

## 命名

| 对象 | 规则 | 示例 |
|---|---|---|
| 函数 | `模块_动词_名词`，小写下划线 | `ddr_init_ctrl()`、`fip_verify_payload()` |
| 局部变量 | 小写下划线；作用域短可短名 | `i`、`len_bytes`、`timeout_ms` |
| 指针 | 不加 `p` 前缀，用 `const` 表达不可变 | `struct fip_toc *toc` |
| 结构体 / 枚举 | `struct foo`、`enum foo_state`，不新造 typedef | `enum bl2_state` |
| 宏 / 常量 | `UPPER_SNAKE` + 模块前缀 | `FIP_MAX_PAYLOAD`、`BL2_STACK_SIZE` |
| 布尔 | `is_` / `has_` / `can_` / `need_` | `is_secure_boot` |
| 静态符号 | 加文件或模块前缀，避免污染链接空间 | `static int bl2_load_image()` |
| 类型宽度 | 定宽类型 `u8/u16/u32/u64`（内核）或 `uint8_t` 等（固件） | 与目标代码保持一致 |

**单位后缀是强制的**：`_ms`、`_us`、`_hz`、`_bytes`、`_kb`、`_mhz`。无单位的时间/长度参数是 P2，因为它是内核里最常见的单位混用 bug 来源。

**禁止**：匈牙利命名法（`pFoo`、`dwSize`）；除循环变量外的单字母名；`tmp`、`data`、`buf` 这类无信息名（缓冲区除外，但要在注释里写长度约束）。

## 函数布局

```c
static int
fip_verify_payload(const struct fip_toc_entry *entry, void *dst, size_t len_bytes)
{
        int ret;

        if (!entry || !dst || !len_bytes)
                return -EINVAL;

        ret = fip_load(entry->offset, dst, len_bytes);
        if (ret)
                goto err_out;

        return fip_check(dst, len_bytes);

err_out:
        memset(dst, 0, len_bytes);
        return ret;
}
```

- 返回类型独占一行，函数名顶格；参数每行一个，与左括号对齐。
- 缩进：8 空格 tab（内核）；若目标文件用 4 空格，跟随。
- 行宽：80 列（内核）；目标文件为 100 列则跟随。
- 大括号：K&R，`else` 与 `}` 同行；单语句分支可省略大括号，但整个文件必须一致。
- 局部变量就近声明（C99），但**不能跳过初始化进入 goto 标签**；`-Wjump-misses-init` 应保持开启。
- 空行：逻辑段之间一个，函数之间一个；不留连续两个空行。

## 控制流

- 单资源早返回；多资源用 `goto` 逆序回退，标签命名 `err_free`、`err_unmap`、`err_put`、`out`。
- `switch` 覆盖所有枚举值，`default` 分支要么 `return` 要么写 `/* fall through */` 注释。
- 条件里不赋值，除非是 `while ((c = getc()) != EOF)` 这类惯用法，且必须加双括号。
- 不用 `do { } while (0)` 包裹普通语句块——它只用于宏。
- 循环上限来自外部输入时，先校验上限再循环。

## 注释

**必须写**：

- 文件头：`SPDX-License-Identifier` + 模块一句话职责。
- 对外函数或超过 15 行的函数：用途、参数、返回值、**上下文（可睡眠？持锁？ISR 安全？）**、副作用。
- 不变量与前置条件：`/* caller must hold foo->lock */`、`/* len_bytes excludes the header */`。
- 硬件相关：寄存器偏移来源（手册章节）、保留位、勘误、时序要求。
- 任何“看起来奇怪”的写法：为什么不能改。

**禁止写**：

- 复述代码（`i++; /* increment i */`）。
- 过期的注释（改了代码没改注释，比没注释更糟——P2）。
- 被注释掉的代码块（删掉，或用 `#if 0` 并注明原因与恢复条件）。

**TODO/FIXME** 必须带 owner 或 issue：`/* FIXME(jdoe): 临时绕过，见 BUG-1234 */`。

## 头文件与宏

- include 顺序：自身头文件 → 系统/标准库 → 平台/框架头 → 本地头；组间空行。
- 头文件用 guard，命名与路径一致：`#ifndef BL2_IMAGE_H` / `#define BL2_IMAGE_H`。
- 头文件里不放定义（除 `static inline` 与常量宏）；不放会变化的配置。
- 宏：参数加括号、整体加括号；多语句用 `do { } while (0)`；避免参数带副作用（`MIN(a++, b)`）。
- 优先 `static inline` 而不是宏，除非需要类型泛型或字符串化。

## 错误处理

- 统一错误码风格：内核返回负 errno；固件在 `int` 与自定义枚举间只能选一种，且全项目一致。
- 每层只做本层该做的清理，不假设上层会清理。
- 不吞错误：不能处理的错误必须往上返回，或明确记录并改变状态机。
- 失败路径也要保证状态一致（部分初始化必须回滚）。

## 工具门禁（能跑就跑）

| 工具 | 用途 | 命令 |
|---|---|---|
| `checkpatch.pl --strict` | 内核风格 | `scripts/checkpatch.pl --strict --no-tree -f file.c` |
| `sparse` | 地址空间标注 `__user`/`__iomem`/`__rcu` | `make C=1`、`sparse file.c` |
| `gcc -Wall -Wextra -Wshadow -Wpointer-arith` | 基础告警 | 编译时开启 |
| `cppcheck` | 裸机/无构建系统时兜底 | `cppcheck --enable=warning,style file.c` |
| `clang-tidy` | 有条件时 | 慎用，规则集需裁剪，避免噪声淹没真问题 |

工具报出的问题不直接当结论：先确认它是不是误报，再决定级别。
