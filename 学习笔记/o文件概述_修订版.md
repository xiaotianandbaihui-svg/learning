# .o 文件概述与 ELF 结构完整笔记

本文介绍 Linux 与 GNU 工具链中常见的 **ELF 可重定位目标文件**，并区分 ELF32、ELF64 和 ARM32 的相关细节。内容按“整体结构 → ELF 文件头 → 节头表 → 符号表 → 绑定与可见性 → 重定位”组织。

`.o` 是常用扩展名，不是文件格式的充分判据；其他平台或特殊编译选项也可能生成其他格式的 `.o`。以下讨论以普通 ELF 目标文件为前提。

## 1 什么是 .o 文件

典型生成命令：

```bash
gcc -c test.c -o test.o
```

`gcc` 驱动编译工具链完成预处理、编译和汇编，生成目标文件；`-c` 表示不进行最终链接。

普通 `.o` 文件通常已经含有机器代码，但仍可能存在尚未解析的外部符号和需要修补的地址或指令字段。链接器将多个目标文件及所需库组合起来，确定布局、解析符号并处理重定位。

```bash
gcc test.o other.o -o program
```

ELF 可重定位目标文件的 `e_type` 是 `ET_REL`。它通常不能像最终可执行程序一样直接运行。

## 2 ELF 文件的整体结构

ELF 文件中常见的组成如下。**这是内容组成示意，不是固定的磁盘排列顺序。**

```text
ELF 文件
├── ELF Header                 文件头
├── Program Header Table       程序头表，普通 .o 通常没有
├── 各节的实际内容
│   ├── .text                  代码及相关内容
│   ├── .data                  已初始化的可写数据
│   ├── .rodata                只读数据
│   ├── .symtab                符号表
│   ├── .strtab                符号名称字符串表
│   ├── .shstrtab              节名称字符串表
│   ├── .rel.* 或 .rela.*      重定位条目
│   └── 其他节
└── Section Header Table       节头表，其位置由 e_shoff 指定
```

`.bss` 通常也有节头表项，但类型为 `SHT_NOBITS`，其对应的数据字节不实际存储在文件中。

需要区分三个概念：

| 概念 | 主要作用 |
|---|---|
| 节 Section | 从链接角度组织内容，如代码、数据、符号和重定位信息 |
| 节头表 Section Header Table | 记录每个节的类型、大小、文件偏移和其他属性 |
| 段 Segment | 从装载角度组织内容，由程序头表描述；一个段可以包含多个节 |

程序头表主要用于可执行文件和共享对象的装载。节头表主要服务于链接和分析，但不是“只有 `.o` 才有”；可执行文件和共享库也常带节头表。

**符号表和重定位表本身也是节。**它们的节头表项保存位置及属性，实际条目位于对应节内容中。不要把“节头表中的记录”与“这个节的内容”混在一起。

## 3 ELF32 与 ELF64 的区别

文件类别由 `e_ident[EI_CLASS]` 指定，不能仅凭文件扩展名判断。

| 标准结构或字段 | ELF32 | ELF64 |
|---|---:|---:|
| ELF 文件头 | 52 字节 | 64 字节 |
| 程序头表项 | 32 字节 | 56 字节 |
| 节头表项 | 40 字节 | 64 字节 |
| 符号表项 | 16 字节 | 24 字节 |
| `Rel` 重定位表项 | 8 字节 | 16 字节 |
| `Rela` 重定位表项 | 12 字节 | 24 字节 |
| `Addr`、`Off` 类型 | 32 位无符号 | 64 位无符号 |
| `Word` 类型 | 32 位无符号 | 32 位无符号 |

ELF64 的 `Word` 仍为 32 位；需要 64 位整数的字段使用 `Xword` 或其他适当类型。

原笔记节表截图中，`.symtab` 的 `EntSize` 为 `0x18`，即 24 字节，因此那份示例是 **ELF64**。后面列出的 `Elf32_*` 结构体则用于 ELF32。两者的字段含义可以对照，但大小、布局和位运算不能直接混用。具体处理器架构还要查看 `e_machine`。

ARM32 通常对应 ELF32、`EM_ARM`；AArch64 通常对应 ELF64、`EM_AARCH64`，不能把二者的重定位规则混用。

## 4 ELF 文件头

查看命令：

```bash
readelf -h test.o
```

### 4.1 ELF32 文件头结构

以下为字段布局示意，类型含义遵循 ELF 定义：

```c
typedef struct {
    unsigned char e_ident[16];
    Elf32_Half    e_type;
    Elf32_Half    e_machine;
    Elf32_Word    e_version;
    Elf32_Addr    e_entry;
    Elf32_Off     e_phoff;
    Elf32_Off     e_shoff;
    Elf32_Word    e_flags;
    Elf32_Half    e_ehsize;
    Elf32_Half    e_phentsize;
    Elf32_Half    e_phnum;
    Elf32_Half    e_shentsize;
    Elf32_Half    e_shnum;
    Elf32_Half    e_shstrndx;
} Elf32_Ehdr;
```

`Elf32_Half` 是 **16 位无符号整数**，不是有符号整数。`Elf32_Word`、`Elf32_Addr`、`Elf32_Off` 均为 32 位无符号类型，名称反映用途；`Elf32_Sword` 才是 32 位有符号整数。

自行解析二进制文件时还必须处理文件字节序，不能只把文件内容强制转换为本机结构体就认为一定正确。

### 4.2 e_ident 的 16 个字节

| 下标 | 含义 |
|---|---|
| `[0..3]` | ELF 魔数：`0x7F 0x45 0x4C 0x46`，即 `0x7F 'E' 'L' 'F'` |
| `[4]`，`EI_CLASS` | 文件类别：`1 = ELF32`，`2 = ELF64`；不是版本号 |
| `[5]`，`EI_DATA` | 数据编码：`1 = 小端`，`2 = 大端` |
| `[6]`，`EI_VERSION` | ELF 标识版本，当前通常为 `1` |
| `[7]`，`EI_OSABI` | OS/ABI 标识 |
| `[8]`，`EI_ABIVERSION` | ABI 版本 |
| `[9..15]` | 填充字节 |

只有前四个字节是固定魔数，整个 `e_ident` 并不是固定为四个字符；也不能把 `[7..15]` 全部称为未使用字段。

### 4.3 其他文件头字段

| 字段 | 含义 |
|---|---|
| `e_type` | ELF 文件类型 |
| `e_machine` | 目标处理器架构 |
| `e_version` | ELF 格式版本，通常为 `EV_CURRENT = 1` |
| `e_entry` | 程序入口地址；普通可重定位 `.o` 通常为 0 |
| `e_phoff` | 程序头表相对于文件开头的字节偏移；没有该表时为 0 |
| `e_shoff` | 节头表相对于文件开头的字节偏移；没有该表时为 0 |
| `e_flags` | 处理器相关标志，必须按对应架构 ABI 解释 |
| `e_ehsize` | ELF 文件头大小 |
| `e_phentsize` | 每个程序头表项的大小 |
| `e_phnum` | 程序头表项数量，注意扩展计数情况 |
| `e_shentsize` | 每个节头表项的大小 |
| `e_shnum` | 节头表项数量，注意扩展计数情况 |
| `e_shstrndx` | 节名称字符串表的节索引 |

常见 `e_type`：

| 数值 | 常量 | 含义 |
|---:|---|---|
| 0 | `ET_NONE` | 未指定文件类型 |
| 1 | `ET_REL` | 可重定位文件，通常是 `.o` |
| 2 | `ET_EXEC` | 可执行文件 |
| 3 | `ET_DYN` | 共享对象；PIE 可执行程序也常使用此类型 |
| 4 | `ET_CORE` | 核心转储文件 |

常见 `e_machine`：

| 数值 | 架构 |
|---:|---|
| 3 | i386 |
| 8 | MIPS |
| `0x28`，40 | ARM |
| `0x3E`，62 | x86-64 |
| `0xB7`，183 | AArch64 |

### 4.4 扩展计数

字段宽度有限，ELF 定义了扩展编码：

- `e_shnum = 0` 不一定表示没有节头表；使用扩展节计数时，真实数量保存在第 0 项节头的 `sh_size` 中。
- `e_shstrndx = SHN_XINDEX` 时，真实的节名字符串表索引保存在第 0 项节头的 `sh_link` 中。
- `e_phnum = PN_XNUM` 时，真实的程序头数量保存在第 0 项节头的 `sh_info` 中。

因此，应结合其他字段和对应规则判断，不能仅凭一个零值下结论。

### 4.5 ARM32 的 e_flags

`0x05000000` 表示 **ARM EABI 第 5 版**，不是“硬件浮点支持”。

`EF_ARM_ABI_FLOAT_HARD = 0x400` 是与硬浮点过程调用约定有关的标志。调用约定与“机器支持浮点指令”或“代码是否使用浮点指令”并非同一个问题。

检查 ARM32 `.o` 时，应结合 `.ARM.attributes` 中的 ABI 属性，例如 `Tag_ABI_VFP_args`，不要仅凭一个 `e_flags` 数值判断浮点兼容性：

```bash
arm-none-eabi-readelf -h test.o
arm-none-eabi-readelf -A test.o
```

交叉工具链前缀也可能是 `arm-linux-gnueabihf-`、`arm-linux-gnueabi-` 等，应使用实际安装的工具。

## 5 节头表

查看命令：

```bash
readelf -S test.o
readelf -SW test.o
```

`-W` 使用宽输出，减少一条记录被拆成多行的情况。输出分成几行只是显示方式，不代表文件中存在两种节表项。

### 5.1 ELF32 节头结构

```c
typedef struct {
    Elf32_Word sh_name;
    Elf32_Word sh_type;
    Elf32_Word sh_flags;
    Elf32_Addr sh_addr;
    Elf32_Off  sh_offset;
    Elf32_Word sh_size;
    Elf32_Word sh_link;
    Elf32_Word sh_info;
    Elf32_Word sh_addralign;
    Elf32_Word sh_entsize;
} Elf32_Shdr;
```

### 5.2 readelf 各列的含义

| 显示列 | 相关字段 | 含义 |
|---|---|---|
| `[Nr]` | 表项索引 | 节索引，从 0 开始 |
| `Name` | `sh_name` | 工具从节名字符串表中读出的名称；底层字段保存字节偏移 |
| `Type` | `sh_type` | 节类型，如 `PROGBITS`、`NOBITS`、`SYMTAB` |
| `Address` | `sh_addr` | 该节在执行映像中的地址；普通 `.o` 中通常尚未分配，常为 0 |
| `Offset` | `sh_offset` | 节在文件中的起始字节偏移；`NOBITS` 的含义需单独处理 |
| `Size` | `sh_size` | 节大小；`NOBITS` 也可以有非零大小 |
| `EntSize` | `sh_entsize` | 若节由固定大小条目组成，表示每条大小；不适用时通常为 0 |
| `Flags` | `sh_flags` | 节属性的可读显示 |
| `Link` | `sh_link` | 与节类型相关的关联信息 |
| `Info` | `sh_info` | 与节类型相关的附加信息，不总是节索引 |
| `Align` | `sh_addralign` | 地址对齐约束；0 或 1 表示无特殊对齐要求，其他有效值通常为 2 的幂 |

`Address` 和 `Offset` 分别属于地址空间和文件位置两个概念，不可互换。

例如：

```text
There are 14 section headers, starting at offset 0x4d8:
```

准确含义是：有 **14 个节头表项**，节头表从文件偏移 `0x4d8` 开始。第 0 项为保留的空项，不是一个普通内容节。

### 5.3 Flags

这些字母是 `readelf` 对标志位的显示，文件中实际保存的是数值位掩码。

| 字母 | 标志 | 含义 |
|---|---|---|
| `W` | `SHF_WRITE` | 对应内容在执行时应允许写入 |
| `A` | `SHF_ALLOC` | 对应内容应占据执行期间的内存空间 |
| `X` | `SHF_EXECINSTR` | 含有可执行机器指令 |
| `M` | `SHF_MERGE` | 节中的合适条目允许链接器合并 |
| `S` | `SHF_STRINGS` | 包含以零字符结尾的字符串条目 |
| `I` | `SHF_INFO_LINK` | `sh_info` 字段包含一个节索引 |

`M` 不表示链接器可以随意合并整个节；`I` 也不是笼统的“Link 和 Info 有信息”。

### 5.4 Link 与 Info 的具体例子

| 节类型 | `sh_link` | `sh_info` |
|---|---|---|
| `SHT_SYMTAB` / `SHT_DYNSYM` | 关联字符串表的节索引 | 第一个非 `STB_LOCAL` 符号的索引 |
| 可重定位文件中的 `SHT_REL` / `SHT_RELA` | 关联符号表的节索引 | 被修补的目标节索引 |

原示例中的 `.rela.text`：

- `Link = 11`，关联第 11 节 `.symtab`。
- `Info = 1`，表示重定位目标是第 1 节 `.text`。

`Link` 负责找到符号表；每个重定位条目再用自身的符号索引，从那个符号表中选中具体符号。

## 6 常见节逐项说明

### 6.1 .text

通常存放机器代码，常见类型为 `PROGBITS`，标志为 `AX`。但不要把 `.text` 每一个字节都当作指令：例如 ARM 代码节中也可能有文字池等数据。

使用 `-ffunction-sections` 等选项时，还可能出现 `.text.main`、`.text.func` 等多个代码节。

### 6.2 .data

通常存放有初始内容的可写全局变量和静态存储期变量，常见类型为 `PROGBITS`，标志为 `WA`。

初始化为零的变量也可能被放进 `.bss`，因此“写了初始化表达式”不保证进入 `.data`。

### 6.3 .bss

通常存放需要零初始化的全局变量和静态存储期变量，常见类型为 `NOBITS`，标志为 `WA`。

它的关键性质是：**有逻辑大小，但文件中不存放对应的初始化数据字节。**装载器或裸机启动代码负责在运行前建立所需的零初始化状态。

原截图的 `.bss`：

```text
Type   = NOBITS
Offset = 0xbc
Size   = 0x4
```

这表示需要 4 字节，而不是 `Size=0`。紧随其后的 `.rodata` 可以同样从文件偏移 `0xbc` 开始，因为 `.bss` 不消耗这 4 个文件数据字节。

### 6.4 .rodata

通常存放字符串字面量等只读数据，常见标志为 `A`，没有 `W`。

不是每个 `const` 对象都一定占据 `.rodata`：它可能是局部自动对象、被优化消除，或因重定位等要求采用其他安排。是否只读取决于具体存储安排和运行时映射，不能仅凭源码中的一个关键字判断。

### 6.5 .symtab

普通符号表，常见类型为 `SYMTAB`。记录链接或分析所需的符号，包括函数、对象、文件、节等。

它**不保证包含源程序中的所有变量名和函数名**。普通自动局部变量通常不以独立符号出现；优化也会消除对象。源代码级局部变量信息通常需要查阅调试信息。

共享对象中还常有 `.dynsym`，用于动态链接。`.dynsym` 与 `.symtab` 不是同一张表。

### 6.6 .strtab 与 .shstrtab

- `.strtab`：通常为 `.symtab` 提供符号名称字符串。
- `.shstrtab`：为节头表中的 `sh_name` 提供节名称字符串。

二者类型都通常为 `STRTAB`。关联关系应通过对应字段确定，而不是只靠约定名称猜测。

### 6.7 .rel.* 与 .rela.*

它们保存重定位条目。例如：

- `.rel.text` / `.rela.text`：修补 `.text` 中的位置。
- `.rel.data` / `.rela.data`：修补 `.data` 中的位置，例如全局指针初始值。
- `.rela.eh_frame`：修补 `.eh_frame` 中的位置。

重定位不只由外部函数引用产生。本文件内其他节中的对象、地址常量等，也可能需要重定位。

### 6.8 其他常见节

| 节 | 作用 |
|---|---|
| 第 0 项 | 类型为 `SHT_NULL` 的保留节头表项；某些字段可承载扩展计数信息 |
| `.comment` | 常用于记录编译器版本等信息，具体内容和标志取决于工具链 |
| `.note.GNU-stack` | 向链接器传达该输入对象的栈执行权限要求；常以是否有执行标志区分 |
| `.note.gnu.property` | GNU 属性记录，例如某些处理器特性；名称区分大小写，以实际文件为准 |
| `.eh_frame` | 栈展开相关信息，可用于异常处理、回溯等 |
| `.debug_*` | 调试信息，如源文件、行号、类型和局部变量描述 |
| `.ARM.attributes` | ARM 架构和 ABI 属性 |
| `.ARM.exidx` / `.ARM.extab` | ARM 异常处理 ABI 中常见的栈展开相关节 |

仅凭存在 `.note.GNU-stack`，不能一概断言“不需要可执行栈”；还要查看它的标志。该输入节主要由链接器处理，最终装载要求通常通过输出文件的相关程序头表达。

## 7 符号表

查看命令：

```bash
readelf -sW test.o
nm test.o
```

`readelf` 展示 ELF 的字段及其解释；`nm` 将多种属性归纳成 `T`、`D`、`B`、`U` 等类型字母。符号表内部不是直接存储这些 `nm` 字母。

### 7.1 ELF32 符号表项

```c
typedef struct {
    Elf32_Word    st_name;   /* 名称在关联字符串表中的字节偏移 */
    Elf32_Addr    st_value;  /* 值，具体含义取决于符号及文件类型 */
    Elf32_Word    st_size;   /* 符号大小，单位为字节 */
    unsigned char st_info;   /* 绑定与类型 */
    unsigned char st_other;  /* 可见性及其他约定的位 */
    Elf32_Half    st_shndx;  /* 节索引或特殊索引 */
} Elf32_Sym;
```

这是 ELF 符号表项的二进制布局描述，不应称为“只在内核中的表示”。链接器、装载器、调试器和分析工具都可能处理它。

### 7.2 ELF64 符号表项

```c
typedef struct {
    Elf64_Word    st_name;
    unsigned char st_info;
    unsigned char st_other;
    Elf64_Half    st_shndx;
    Elf64_Addr    st_value;
    Elf64_Xword   st_size;
} Elf64_Sym;
```

ELF64 的字段顺序与 ELF32 不完全相同，因此不仅要改字段宽度，还要按对应布局读取。

### 7.3 readelf 输出列

| 列 | 含义 |
|---|---|
| `Num` | 该符号在当前这张符号表中的索引 |
| `Value` | `st_value` |
| `Size` | `st_size` |
| `Type` | 从 `st_info` 中提取的符号类型 |
| `Bind` | 从 `st_info` 中提取的绑定属性 |
| `Vis` | 从 `st_other` 中提取的可见性 |
| `Ndx` | `st_shndx` 对应的节索引或特殊名称 |
| `Name` | 根据 `st_name` 从关联字符串表取得的名称 |

**`st_name` 不是 `Num`。**前者是字符串表字节偏移，后者是符号表条目索引。

第 0 个符号表项通常是规定的空符号项，不是普通用户定义符号。

### 7.4 st_info 与符号类型

`st_info` 的高 4 位保存绑定属性，低 4 位保存类型：

```c
bind = st_info >> 4;
type = st_info & 0x0f;
```

| 类型 | 常见含义 |
|---|---|
| `STT_NOTYPE` | 未指定更具体类型 |
| `STT_OBJECT` | 数据对象 |
| `STT_FUNC` | 函数或其他可执行代码对象 |
| `STT_SECTION` | 节符号 |
| `STT_FILE` | 源文件名称符号 |
| `STT_TLS` | 线程局部存储对象 |

符号类型与绑定属性是两个独立维度。例如函数既可以是 `LOCAL`，也可以是 `GLOBAL` 或 `WEAK`。

### 7.5 st_value 与 Ndx 必须结合看

| 情形 | `st_value` 的典型含义 |
|---|---|
| `ET_REL` 中普通已定义符号，`Ndx` 为普通节索引 | 相对于所属节的偏移，需考虑架构特殊约定 |
| `Ndx = UND`，即 `SHN_UNDEF` | 当前文件未定义该符号；值通常为 0，不能直接当目标地址 |
| `Ndx = ABS`，即 `SHN_ABS` | 绝对值，不随普通节布局调整 |
| `Ndx = COM`，即 `SHN_COMMON` | 通常表示对齐要求；`st_size` 表示所需大小 |
| 最终可执行文件中的普通已定义符号 | 通常为链接后的虚拟地址 |
| 共享对象或 PIE 中的普通已定义符号 | 通常为链接时虚拟地址，运行时还需结合装载偏移 |
| TLS 符号 | 按线程局部存储规则解释，不能套用普通地址规则 |

因此，“`.o` 中是偏移，链接后全部变为绝对地址”过于简单。

当 `st_shndx = SHN_XINDEX` 时，真实节索引通过关联的 `SHT_SYMTAB_SHNDX` 节取得，不能把该值当成普通节编号。

原截图中的几个例子：

| 符号 | Value | Size | Bind | Ndx | 解释 |
|---|---:|---:|---|---|---|
| `nInitData` | `0x0` | 4 | GLOBAL | 3 | 定义在第 3 节 `.data` 的起始位置 |
| `nStaticInit.2320` | `0x4` | 4 | LOCAL | 3 | 定义在同一 `.data` 节的偏移 4 处 |
| `nStaticUninit.2321` | `0x0` | 4 | LOCAL | 4 | 定义在第 4 节 `.bss` 的起始位置 |
| `nUninitData` | `0x4` | 4 | GLOBAL | COM | 需要 4 字节空间、4 字节对齐，不能把 Value 当作地址 4 |
| `func` | `0x0` | 40 | GLOBAL | 1 | 定义在第 1 节 `.text`，大小 40 字节 |
| `printf` | `0x0` | 0 | GLOBAL | UND | 当前文件引用但未定义，由后续链接解析 |
| `main` | `0x28` | 73 | GLOBAL | 1 | 在 `.text` 中偏移 `0x28`，大小 73 字节 |

源码中的暂定定义是否生成 COMMON 符号，与语言及 `-fcommon` / `-fno-common` 等选项有关，不能认定所有未显式初始化的全局变量都是 COMMON。

### 7.6 ARM32 的 Thumb 特例

对 ARM ELF 的某些函数符号，尤其 Thumb 函数符号，`st_value` 的最低位用于指示 Thumb 状态。分析实际指令地址时必须按 ARM ABI 解释，不能把奇数值简单视为指令真的从奇数字节地址开始。

ARM 还可能有 `$a`、`$t`、`$d` 等映射符号，用于区分 ARM 指令、Thumb 指令和数据区域。因此代码节中出现数据并不一定是格式错误。

## 8 字符串表与 st_name 的查找过程

假设符号表通过自身节头的 `sh_link` 关联到下面的字符串表：

```text
偏移   00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
字节   00 6D 61 69 6E 00 70 72 69 6E 74 66 00 66 6F 6F
字符   \0  m  a  i  n \0  p  r  i  n  t  f \0  f  o  o

偏移   10 11 12 13
字节   00 67 5F 00
字符   \0  g  _ \0
```

查找规则：

```text
名称开始位置 = 关联字符串表的起始位置 + st_name
```

从该位置读取，直到遇到零字节 `0x00` 为止。这里的“起始位置”可指文件中的节偏移，也可指工具已读取的字符串表缓冲区起始位置，不能混用两种基准。

| st_name | 从哪里读 | 结果 |
|---:|---|---|
| 1 | 字节偏移 `0x01` | `main` |
| 6 | 字节偏移 `0x06` | `printf` |
| 13 | 字节偏移 `0x0D` | `foo` |
| 17 | 字节偏移 `0x11` | `g_` |

字符串表是字节数组，不是固定宽度的“名称行表”。`st_name=13` 不是“第 13 个名称”。

## 9 Bind 与 Vis

### 9.1 绑定属性 Bind

| 值 | 含义 |
|---|---|
| `STB_LOCAL` | 局部符号，仅在所属目标文件的链接范围内使用，不作为供其他目标文件按名称解析的全局定义 |
| `STB_GLOBAL` | 全局符号，可参与跨目标文件的符号解析；是否对动态链接导出还受其他条件影响 |
| `STB_WEAK` | 弱符号，具有特殊解析规则，通常允许同名的普通全局定义优先 |

局部符号仍会参与本文件内的重定位等链接工作；“LOCAL”不等于“不参与链接”。

**未定义和弱绑定是两个不同属性。**例如：

```c
extern int value;
```

当代码实际引用 `value` 时，通常产生的是 `GLOBAL + UND`，并不会因为当前文件没有定义它就自动成为 `WEAK`。

弱符号通常来自显式的弱符号声明、汇编指令或编译器特定机制。对未解析弱引用的处理，也要按具体链接方式和 ABI 判断。

### 9.2 可见性 Vis

`st_other` 不是完全未使用的保留字段。通用 ELF 可见性通常由其低 2 位取得：

```c
visibility = st_other & 0x03;
```

其余位可能有架构或平台约定，不宜一概称为永远未使用。

| 值 | 含义 |
|---|---|
| `STV_DEFAULT` | 默认可见性，具体行为与绑定属性、文件类型及链接规则共同决定 |
| `STV_INTERNAL` | 处理器或平台相关的内部可见性语义 |
| `STV_HIDDEN` | 符号不作为对组件外部可见的普通动态接口；链接器还可能将其局部化 |
| `STV_PROTECTED` | 对外可见，但定义所在组件内部的引用绑定自身定义，不被外部同名定义抢占 |

`PROTECTED` **不是内存只读权限**，不会把一个可写变量变成“外部只能读取，不能修改”。符号抢占是名称解析问题，内存写入是另一件事。

`HIDDEN` 也不等于隐藏机器代码或阻止别人分析文件，不能作为防止读取文件内容的安全边界。

### 9.3 为什么 GLOBAL 还需要 Vis

例如，一个共享库中有很多需要跨源文件调用的内部函数，它们在编译阶段可能具有全局绑定，但不希望全部成为共享库的外部接口。可见性可用于控制这一点。

常见做法是默认隐藏，再将公共 API 标为默认可见。但最终导出情况还受版本脚本、链接选项、符号是否被保留等影响，不能说“所有 GLOBAL 都必然导出”或“只有显式 DEFAULT 的才必然导出”。

可见性在 `.o` 中就能记录，并能影响静态链接阶段的处理和优化，不是只到运行时才起作用。

| 比较项 | Bind | Vis |
|---|---|---|
| 主要关注 | 局部、全局、弱绑定及符号解析规则 | 对组件外部的可见性与是否允许抢占 |
| 所在字段 | `st_info` 高 4 位 | `st_other` 的可见性位 |
| 常见值 | LOCAL / GLOBAL / WEAK | DEFAULT / INTERNAL / HIDDEN / PROTECTED |
| 需要避免的误解 | 未定义符号不自动是弱符号 | 可见性不是内存读写权限，也不是保密机制 |

## 10 重定位表

这里应使用术语 **重定位 relocation**，不是“重定向”。

重定位条目告诉链接器或动态装载器：**在哪里，根据哪个符号和哪种规则，计算并修补一个值或指令字段。**

查看命令：

```bash
readelf -rW test.o
objdump -dr test.o
```

`objdump -dr` 将反汇编与重定位信息放在一起显示，便于观察某条指令或数据位置为何需要修补。

### 10.1 ELF32 重定位条目

```c
typedef struct {
    Elf32_Addr r_offset;
    Elf32_Word r_info;
} Elf32_Rel;

typedef struct {
    Elf32_Addr  r_offset;
    Elf32_Word  r_info;
    Elf32_Sword r_addend;
} Elf32_Rela;
```

### 10.2 ELF64 重定位条目

```c
typedef struct {
    Elf64_Addr  r_offset;
    Elf64_Xword r_info;
} Elf64_Rel;

typedef struct {
    Elf64_Addr   r_offset;
    Elf64_Xword  r_info;
    Elf64_Sxword r_addend;
} Elf64_Rela;
```

### 10.3 字段含义

| 字段 | 含义 |
|---|---|
| `r_offset` | 被修补的位置；在 `ET_REL` 中通常为目标节内偏移，在可执行文件或共享对象中通常为虚拟地址 |
| `r_info` | 符号索引和重定位类型的组合编码 |
| `r_addend` | `Rela` 中显式保存的附加数 |

`Rel` **没有显式的附加数字段**，但不代表计算中没有附加数。其附加数通常隐含在目标位置的原内容中；对指令重定位，可能需要按指令格式提取和解释。

`Rela` 将附加数显式保存在 `r_addend` 中。

### 10.4 r_info 的拆分

通常的 ELF32 编码：

```c
symbol_index = r_info >> 8;
reloc_type   = r_info & 0xff;
```

即高 24 位为符号索引，低 8 位为重定位类型。

通常的 ELF64 编码：

```c
symbol_index = r_info >> 32;
reloc_type   = r_info & 0xffffffff;
```

即高 32 位为符号索引，低 32 位为重定位类型。具体架构可能有扩展规则；分析某架构时应以对应 ABI 为准。

### 10.5 两条不同的查找路径

对普通 `.o` 中的重定位，需要区分“修补哪里”和“引用谁”。

```text
修补哪里
重定位节的 sh_info
    → 找到目标节
    → 用 r_offset 找到目标节内的修补位置

引用谁
重定位节的 sh_link
    → 找到关联符号表
    → 从 r_info 提取符号索引
    → 找到对应符号表项
    → 解析其定义、最终位置及其他属性
```

符号索引对应的是 **关联符号表中的条目编号**，通常就是那张表在 `readelf -s` 中显示的 `Num`。它不是字符串表偏移，也不是节索引，更不是终端输出的任意物理行号。

### 10.6 为什么不能只靠节号得到准确地址

对于普通已定义符号，需要结合所属输入节、节内偏移和链接后的布局等信息来确定符号值。

对于 `SHN_UNDEF` 符号，还必须进行全局符号解析，去其他目标文件或库中寻找适用定义；对于 COMMON、TLS、Thumb 函数等情况，则还要使用对应规则。

而且“确定符号值”之后，仍要按重定位类型计算最终修补内容。例如，常见形式包括：

```text
绝对形式：    S + A
PC 相对形式： S + A - P
```

其中：

- `S` 是按 ABI 规则确定的符号值。
- `A` 是附加数。
- `P` 是重定位位置的地址。

这些只是常见形式示意，不是所有重定位的统一公式。实际规则可能还涉及 GOT、PLT、TLS、符号的指令集状态、范围检查，以及指令位域编码。

### 10.7 一个简化的绝对地址重定位例子

以下为说明流程而设定的例子，不对应原截图中的实际二进制文件。假设处理普通数据符号，并且该重定位的计算规则为 `S + A`。

```text
目标节 .text 的文件偏移       = 0x40
r_offset                    = 0x10
符号最终解析值 S             = 0x20000008
附加数 A                    = 4
```

那么，在输入 `.o` 中，需要查看或修补的位置为：

```text
0x40 + 0x10 = 文件偏移 0x50
```

计算结果为：

```text
S + A = 0x2000000c
```

**文件偏移 `0x50` 是修补位置，`0x2000000c` 是按规则算出的值，二者不是一回事。**链接器生成输出文件时还会将输入节放入新的布局，不必在输出文件中继续使用输入文件的偏移 `0x50`。

### 10.8 ARM32 的重定位补充

ARM32 工具链常见 `.rel.*` 形式，但仍应以文件中的实际节类型为准。

常见 ARM 重定位名称包括 `R_ARM_ABS32`、`R_ARM_REL32`、`R_ARM_CALL`、`R_ARM_THM_CALL` 等。处理调用和跳转时，要考虑 ARM/Thumb 指令格式、状态切换、位移编码和范围限制；不能把“重定位”统一理解为向一个位置写入四字节函数地址。

## 11 常用检查命令

| 命令 | 主要用途 |
|---|---|
| `file test.o` | 初步识别文件类型、位数和架构 |
| `readelf -h test.o` | ELF 文件头 |
| `readelf -SW test.o` | 节头表，宽输出 |
| `readelf -sW test.o` | 符号表 |
| `readelf -rW test.o` | 重定位表 |
| `readelf -x .text test.o` | 以十六进制查看 `.text` 的原始字节 |
| `readelf -p .strtab test.o` | 按字符串形式查看 `.strtab` |
| `readelf -A test.o` | 架构专用属性，例如 ARM 属性 |
| `objdump -dr test.o` | 反汇编并标出重定位 |
| `nm test.o` | 符号的简洁显示 |
| `strings -a test.o` | 扫描整个文件中的可打印字符串片段，不依赖符号表 |

对 ARM 目标文件，优先使用对应交叉工具链中的工具，尤其是反汇编工具。例如：

```bash
arm-none-eabi-readelf -h -SW -sW -rW test.o
arm-none-eabi-objdump -dr test.o
arm-none-eabi-nm test.o
```

普通 `.o` 中的反汇编地址往往是节内偏移，并非最终运行地址；某些操作数还要等待重定位完成后才有最终意义。

## 12 阅读时应始终区分的几组概念

1. **文件偏移与运行地址**：`sh_offset` 不是 `sh_addr`。
2. **节与段**：节主要服务链接，段主要服务装载。
3. **节头表与节内容**：节头记录位置及属性，表项内容在对应节内。
4. **符号索引与名称偏移**：`Num` 不是 `st_name`。
5. **符号类型与绑定属性**：`FUNC` 不是 `GLOBAL`，它们来自不同位。
6. **未定义与弱绑定**：`UND` 不等于 `WEAK`。
7. **符号可见性与内存权限**：`PROTECTED` 不等于只读。
8. **节大小与文件占用**：`.bss` 可有非零大小而不存储对应数据字节。
9. **修补位置与被引用对象的位置**：`r_offset` 定位前者，符号解析参与确定后者。
10. **REL 与 RELA**：前者没有显式附加数字段，不能说它没有附加数。
11. **ELF32 与 ELF64**：不仅宽度不同，部分结构体字段顺序也不同。
12. **符号名与实际代码内容**：符号表、字符串扫描、反汇编分别观察不同层面的信息。

## 13 进一步核对的规范

ELF 的结构和通用字段应以通用 ABI 为准，处理器相关标志和重定位以对应处理器 ABI 为准，工具显示方式则参考工具文档。

- [System V ELF 通用 ABI](https://gabi.xinuos.com/elf/)
- [Arm ABI 官方文档仓库](https://github.com/ARM-software/abi-aa)，其中 `aaelf32` 说明 ARM32 ELF，`addenda32` 包含相关补充属性说明。
- [GNU Binutils 文档](https://sourceware.org/binutils/docs/)，可查阅 `readelf`、`objdump`、`nm` 和 `strings`。

以上链接用于继续查阅。本文中的示例字段和概念说明不替代特定架构 ABI 的完整定义。
