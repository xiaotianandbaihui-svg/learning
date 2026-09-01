# GNU Make 4.3 完整执行流程（源码精确版）

---

## 总览

```
main()
  │
  ├─ 阶段一：初始化
  │     initialize_global_hash_tables()
  │     环境变量 / 命令行解析
  │     set_default_suffixes()
  │     install_default_suffix_rules()     ← 旧式后缀规则，读 makefile 之前装入
  │     define_automatic_variables()
  │     define_default_variables()         ← CC CFLAGS COMPILE.c 等默认变量
  │
  ├─ 阶段二：读取 makefile
  │     read_all_makefiles()
  │       └─ eval_makefile() → eval()      ← 词法/语法/语义全在这里
  │
  ├─ 阶段三：冻结依赖图
  │     define_makeflags(1, 0)
  │     snap_deps()                        ← 二次展开 + 特殊目标属性传播
  │     convert_to_pattern()               ← 后缀规则 → pattern rule
  │     install_default_implicit_rules()   ← 内置 pattern 规则在这里才装入
  │     snap_implicit_rules()              ← 统计 max_pattern_* 等限制值
  │     build_vpath_lists()
  │
  ├─ 阶段四（可选）：重建 makefile 自身
  │     update_goal_chain(read_files)      ← 若某个 included makefile 是目标
  │       → 若有更新 → exec() 重启 make 进程
  │
  ├─ 阶段五：确定构建目标
  │     命令行有目标 → goals 链表已在解析阶段填好
  │     命令行无目标 → 读取 .DEFAULT_GOAL 变量
  │                    → 取第一个通过过滤的非特殊目标
  │
  ├─ 阶段六：构建执行
  │     update_goal_chain(goals)
  │       └─ update_file()
  │            └─ update_file_1()          ← 核心递归
  │
  └─ 阶段七：收尾
        die()
          ├─ 等待所有子进程
          ├─ remove_intermediates(0)       ← 删中间文件（不是 notice_finished_file）
          └─ exit(status)
```

---

## 阶段一：初始化

```
main()
│
├─ initialize_global_hash_tables()
│    init_hash_global_variable_set()   ← 523 槽的全局变量哈希表
│    strcache_init()                   ← 字符串驻留池
│    init_hash_files()                 ← 1000 槽的全局文件哈希表
│    hash_init_directories()
│    hash_init_function_table()
│
├─ 设置特殊变量（直接 define_variable_cname）
│    .VARIABLES .RECIPEPREFIX .SHELLFLAGS .LOADED .FEATURES 等
│
├─ 从 envp[] 读取环境变量
│    for each "KEY=VALUE" in envp:
│        define_variable(KEY, len, VALUE, o_env, 1)
│        SHELL 特殊处理：保存副本，export = v_noexport（POSIX 要求）
│
├─ decode_env_switches("GNUMAKEFLAGS")
│   decode_env_switches("MAKEFLAGS")
│   decode_switches(argc, argv, 0)    ← 解析本次命令行 -f -j -k -n 等
│
├─ set_default_suffixes()
│    enter_file(".SUFFIXES")
│    suffix_file->deps = 内置后缀列表（.c .cc .o .h .y .l ...）
│    define_variable("SUFFIXES", ...)
│
├─ install_default_suffix_rules()
│    ★ 读 makefile 之前装入
│    把 default_suffix_rules[] 表里的旧式规则（如 .c.o:）
│    作为普通 struct file 写入 files 哈希表，cmds 字段填好
│    → 之后 convert_to_pattern() 会把它们转成 pattern rule
│
├─ define_automatic_variables()
│    $@ $< $^ $* $? $| $% 的 "说明" 变量（实际值在构建时注入）
│
├─ define_makeflags(0, 0)->export = v_export
│    初次设置 MAKEFLAGS / MFLAGS
│
└─ define_default_variables()
     CC CFLAGS CXX CXXFLAGS LD LDFLAGS COMPILE.c LINK.o AR 等
     来自 default_variables[] 数组，origin = o_default
```

---

## 阶段二：读取 makefile

```
read_all_makefiles(makefiles)
│
├─ 展开 $(MAKEFILES) 环境变量，对每个文件调：
│    eval_makefile(name, RM_NO_DEFAULT_GOAL|RM_INCLUDED|RM_DONTCARE)
│
├─ 若命令行有 -f file：
│    eval_makefile(*makefiles, 0)   ← 对每个 -f 依次读
│
└─ 若无 -f：
     按顺序尝试 GNUmakefile / makefile / Makefile
     找到第一个存在的，调 eval_makefile(p, 0)
     全都不存在：把它们加入 read_files 链表（后续 update_goal_chain 再试）


eval_makefile(filename, flags)
│
├─ alloc_goaldep()：为此文件建 goaldep 节点，链入 read_files
├─ enter_file(filename)：在 files 哈希表中注册此 makefile 自身
├─ fopen(filename, "r")
├─ do_variable_definition("MAKEFILE_LIST", filename, f_append_value)
└─ eval(&ebuf, set_default)        ← 真正的解析在这里


eval(ebuf, set_default)
│  这是词法+语法+语义的核心循环，按行驱动：
│
├─ 变量定义行  VAR = value / VAR := value / VAR += value / VAR ?= value
│    parse_var_assignment() 识别操作符
│    do_variable_definition() 写入当前作用域的 variable_set
│
├─ define ... endef 块
│    读取多行，调 do_variable_definition(f_recursive)
│
├─ include / -include / sinclude
│    eval_makefile(file, RM_INCLUDED | RM_NO_TILDE | [RM_DONTCARE])
│    ★ 递归！include 嵌套在当前 eval 调用栈上
│
├─ $(eval ...) 函数
│    展开参数后调 eval_buffer()，递归再次进入 eval
│
├─ ifeq / ifneq / ifdef / ifndef / else / endif
│    条件分支：满足则继续读，不满足则跳到对应 else/endif
│
├─ 规则行  target: prereqs
│    识别冒号位置，分析是否含 %（隐式规则）
│    是否含第二个冒号（静态模式规则）
│    收集后续 TAB 开头的食谱行
│    → record_files() 或 create_pattern_rule()
│
│    record_files()（显式规则）：
│      for each target name:
│          f = enter_file(target)              ← 创建/查找 struct file
│          f->is_target = 1
│          f->cmds = ...                       ← 挂食谱
│          deps 链 = enter_prereqs(split_prereqs(depstr), NULL)
│              for each prereq:
│                  d->file = enter_file(prereq) ← 建立 file 节点
│          f->deps 链上挂 dep 节点
│          ★ 此时 dep->file 已指向 struct file（因为是显式规则）
│
│          默认目标逻辑：
│            若 default_goal_var 值为空 && set_default：
│              跳过含 % 的目标
│              跳过以 . 开头且不含 / 的目标
│              跳过与后缀规则匹配的名字
│              把第一个通过的目标名写入 .DEFAULT_GOAL 变量
│
│    create_pattern_rule()（隐式规则）：
│      alloc struct rule
│      rule->targets = targets[]
│      rule->deps = dep 链，dep->name = "%.c"（% 原样保留！）
│      dep->file = NULL                        ← ★ 不指向 struct file
│      new_pattern_rule() → 链入 pattern_rules 链表
│
├─ target: VARNAME = value（target-specific 变量）
│    record_target_var()
│      若 target 含 %：create_pattern_var() → 链入 pattern_vars 链表
│      若 target 是具体名：enter_file(target)，
│                           define_variable_in_set(f->variables->set, ...)
│
└─ .PHONY .PRECIOUS .SUFFIXES .DEFAULT 等特殊目标
     作为普通规则记录，deps 链挂好
     snap_deps() 阶段才真正传播属性
```

---

## 阶段二结束时内存状态

```
files 哈希表（1000 槽，开放寻址）
  "app"    → struct file { deps→[foo.o,bar.o链], cmds→gcc -o..., is_target=1 }
  "foo.o"  → struct file { deps→[foo.c,foo.h链], cmds→gcc -c..., is_target=1 }
  "foo.c"  → struct file { deps=NULL, cmds=NULL, is_target=0 }
  ".c.o"   → struct file { cmds→旧式食谱, builtin=1 }   ← suffix rule
  ".PHONY" → struct file { deps→[clean链], is_target=1 }
  ...

pattern_rules 链表（struct rule *）
  → rule{ targets=["%.o"], deps→dep("%.c"), cmds=NULL }  ← 用户定义
  ★ 内置 pattern rule 尚未装入

pattern_vars 链表（struct pattern_var *，按 len 升序）
  → pv{ target="%.o", variable{name="CFLAGS",value="-Wall",flavor=f_append} }

global_variable_set（523 槽哈希表）
  CC="gcc"  CFLAGS="-O2"  MAKE="make"  ...
```

---

## 阶段三：冻结依赖图

```
define_makeflags(1, 0)
  重新计算 MAKEFLAGS 字符串（此时所有开关都确定了）


snap_deps()                         ← main.c:2121
│
├─ snapped_deps = 1
│    此后 enter_file 里若检测到新目标加入会报内部错误
│    （guard 用途：防止迭代哈希表时表被扩容）
│
├─ 二次展开（仅当 .SECONDEXPANSION 出现过）
│    hash_dump(&files) 拍快照
│    for each file:
│        expand_deps(file)
│          对 dep->need_2nd_expansion==1 的依赖：
│            若 dep->staticpattern：把 % 替换成 $*（防止二次展开时 % 被错误处理）
│            initialize_file_variables(file)
│            set_file_variables(file)          ← 注入 $* 等自动变量
│            variable_expand_for_file(dep->name, file)
│            重新 enter_prereqs(split_prereqs(expanded))
│
└─ 特殊目标属性传播（按 lookup_file 查找特殊目标，遍历其 deps 链）
     .PRECIOUS   → f2->precious = 1
     .PHONY      → f2->phony = 1
                    f2->is_target = 1
                    f2->last_mtime = NONEXISTENT_MTIME   ← 强制总重建
     .INTERMEDIATE → f2->intermediate = 1
     .SECONDARY  → f2->intermediate = 1, f2->secondary = 1
                    无 deps 则 all_secondary = 1
     .IGNORE     → 无 deps → ignore_errors_flag = 1
                    有 deps → f2->command_flags |= COMMANDS_NOERROR
     .SILENT     → 无 deps → run_silent = 1
                    有 deps → f2->command_flags |= COMMANDS_SILENT
     .NOTPARALLEL→ not_parallel = 1
     .EXPORT_ALL_VARIABLES → export_all_variables = 1


convert_to_pattern()                ← main.c:2128
  遍历 files 哈希表，找形如 ".X.Y" 的文件节点（旧式后缀规则）
  调 convert_suffix_rule()：
    构造等价的 pattern rule（%.Y: %.X）
    create_pattern_rule() → 链入 pattern_rules
  ★ 在内置 pattern rule 装入之前执行，保证用户后缀规则优先


install_default_implicit_rules()    ← main.c:2135
  ★ 在 read_all_makefiles 之后才执行（关键！）
  遍历 default_pattern_rules[] 表（%.o: %.c 等约 100 条）
  install_pattern_rule() → create_pattern_rule() → new_pattern_rule()
  链入 pattern_rules 链尾
  → 用户定义的规则在链头，内置规则在链尾
  → pattern_search 从链头扫，用户规则优先匹配


snap_implicit_rules()               ← main.c:2139
  遍历 pattern_rules 链表，计算：
    num_pattern_rules       → 规则总数
    max_pattern_targets     → 单条规则最多目标数（pattern_search 分配 tryrules[]）
    max_pattern_deps        → 单条规则最多依赖数
    max_pattern_dep_length  → 最长依赖 pattern 字符串长度（pattern_search 分配 depname[]）
  处理依赖 pattern 含 / 的情况：dep->changed = !dir_file_exists_p(dir, "")
  追加 .EXTRA_PREREQS 到所有规则


build_vpath_lists()
  解析所有 vpath 指令和 VPATH 变量，构建搜索路径数组
```

---

## 阶段四（可选）：重建 makefile 自身

```
若 read_files 链表非空（有读取过 makefile）：

  update_goal_chain(read_files)
  ★ 把 makefile 文件本身作为目标尝试更新
  ★ rebuilding_makefiles = 1 模式下运行

  → 若某个 makefile 被更新（mtime 改变）：
       remove_intermediates(0)
       exec() 用原始 argv 重启 make 进程
       ★ 新进程重新走全部流程，读到更新后的 makefile

  → 若 makefile 无需更新：继续往下走
```

---

## 阶段五：确定构建目标

```
命令行有目标（make app clean）：
  decode_switches 阶段已把目标名加入 goals 链表
  → 直接使用

命令行无目标：
  读 default_goal_var->value（即 .DEFAULT_GOAL 变量）
  → 在 eval() 阶段读第一条规则时已写入
  
  过滤规则（读取阶段就用这些规则，不是在这里过滤）：
    含 % 的目标：跳过（不是具体目标）
    以 . 开头且不含 / 的目标：跳过（特殊目标）
    名字匹配旧式后缀组合（如 .c.o）的目标：跳过

  f = lookup_file(.DEFAULT_GOAL->value)
  goals = alloc_goaldep(); goals->file = f

若 goals 仍为空：
  fatal("No targets specified and no makefile found")
```

---

## 阶段六：构建执行

### 6.1 update_goal_chain(goals)

```
update_goal_chain(goals)
│
│  goals 是 struct goaldep * 链表
│  每个节点有 .file 指向要构建的 struct file
│
│  事件驱动循环（支持并行 make）：
│
└─ while (goals 未全部完成):
       start_waiting_jobs()     ← 若有等待 token 的 job，尝试启动
       reap_children(1, 0)      ← 等待并收割至少一个子进程
       
       for each g in goals:
           for f in (g->file 的双冒号链):
               fail = update_file(f, depth=0)
               → 若 f->command_state == cs_running/cs_deps_running：
                    stop=1, break   ← 并行模式：此 target 还在跑，下次再检查
               → 若 fail：记录失败，依 -k 决定是否继续
           
           若此 goal 的所有双冒号链全部 cs_finished：
               从 goals 链中摘除
```

### 6.2 update_file(file, depth) — DAG 遍历层

```
update_file(file, depth)
│
├─ f = file->double_colon ? file->double_colon : file
│       ★ 双冒号规则从链头（首条）开始遍历
│
├─ 剪枝检查：
│    if f->considered == considered:  ← considered 是全局单调递增计数器
│        return f->command_state==cs_finished ? f->update_status : us_success
│        ★ 同一 pass 内不重复处理同一节点（有向图去重）
│
└─ for f in 双冒号链（f, f->prev, f->prev->prev ...）:
       f->considered = considered
       new = update_file_1(f, depth)
       
       if f->command_state == cs_running || cs_deps_running:
           return us_success   ← 并行模式：命令在运行，本次 pass 到此为止
       
       if new && !keep_going_flag:
           return new          ← 失败且不 -k：立即返回
```

### 6.3 update_file_1(file, depth) — 核心逻辑

```
update_file_1(file, depth)
│
├─ ① 快速返回检查
│    if file->updated:
│        if file->update_status > us_none: 返回失败状态（已失败过）
│        else: 返回 0（已成功，无需再处理）
│    switch file->command_state:
│        cs_running    → 返回 0（正在运行）
│        cs_finished   → 返回 file->update_status
│
├─ ② 读取当前 mtime
│    this_mtime = file_mtime(file)    ← stat() 系统调用，结果缓存到 file->last_mtime
│    noexist = (this_mtime == NONEXISTENT_MTIME)
│    must_make = noexist              ← 文件不存在 → 初始 must_make = 1
│
├─ ③ 搜索隐式规则（仅当 cmds==NULL 且未试过）
│    if (!file->phony && file->cmds==0 && !file->tried_implicit):
│        try_implicit_rule(file, depth)
│            └─ pattern_search(file, 0, depth, 0, 0)
│                 → 成功则填入：
│                     file->cmds = rule->cmds
│                     file->deps = 展开 % 后的 dep 链（enter_file 调用）
│                     file->stem = stem 字符串（$* 的来源）
│        file->tried_implicit = 1
│    
│    if file->cmds==0 && !file->is_target && default_file->cmds!=0:
│        file->cmds = default_file->cmds   ← .DEFAULT 规则兜底
│
├─ ④ 递归更新所有依赖（非中间文件）
│
│    ★ 遍历 also_make 链再加上 file 自身（处理多目标规则的依赖）
│    ad = &amake_sentinel; while (ad != NULL):
│        d = ad->file->deps
│        while (d != NULL):
│            if d->file->intermediate: 跳过（中间文件在步骤⑤单独处理）
│
│            start_updating(d->file)     ← 设 updating 标志，检测循环依赖
│            if is_updating(d->file):
│                error("Circular dependency"); 从链中移除此 dep; continue
│
│            d->file->parent = file      ← ★ 记录"谁需要我"（用于错误消息和变量作用域）
│
│            mtime = file_mtime(d->file) ← stat 缓存
│            new = check_dep(d->file, depth, this_mtime, &maybe_make)
│                 └─ 内部调 update_file(d->file, depth+1)  ← 递归！
│                    比较 d->file mtime vs this_mtime
│                    若 d->file 更新或不存在：*must_make_ptr = 1
│            
│            d->changed = (file_mtime(d->file) != mtime || mtime==NONEXISTENT)
│
│    if (running):   ← 并行：某个依赖的命令还在跑
│        set_command_state(file, cs_deps_running)
│        return 0   ★ 不阻塞！返回事件循环，下次 pass 再来
│
├─ ⑤ 递归更新中间文件依赖（must_make 为真时）
│    if must_make || always_make_flag:
│        for d in file->deps:
│            if d->file->intermediate:
│                d->file->parent = file
│                new = update_file(d->file, depth)  ← 对中间文件也递归
│
├─ ⑥ 计算 deps_changed 和最终 must_make
│    for d in file->deps:
│        d_mtime = file_mtime(d->file)
│        if !d->ignore_mtime:
│            if d_mtime==NONEXISTENT && !d->file->intermediate: must_make=1
│            deps_changed |= d->changed
│        d->changed |= (noexist || d_mtime > this_mtime)
│                       ★ d->changed 决定该 dep 是否进入 $?
│
│    最终 must_make 判定（三个 else-if 分支）：
│    ┌─ if file->double_colon && file->deps==0:
│    │      must_make = 1   ← 双冒号无依赖，永远重建
│    ├─ else if !noexist && file->is_target && !deps_changed
│    │         && file->cmds==0 && !always_make_flag:
│    │      must_make = 0   ← 文件存在+有规则+依赖未变+无命令+非-B → 不重建
│    └─ else if !must_make && file->cmds!=0 && always_make_flag:
│           must_make = 1   ← -B 强制重建
│
├─ ⑦ 不需重建
│    if !must_make:
│        notice_finished_file(file)   ← 标记 cs_finished，传播 also_make
│        file->name = file->hname     ← 使用 VPATH 找到的真实路径
│        return 0
│
└─ ⑧ 需要重建 → remake_file(file)
       if file->cmds == 0:
           if file->phony:    us_success（伪目标无命令，成功）
           if file->is_target: us_success（有规则但无命令，成功）
           else:
               complain(file)         ← 输出 "No rule to make target 'X'"
               ★ 完整错误消息在 complain() 里，
                 若 file->parent 非 NULL 则附上 "needed by 'parent'"
               file->update_status = us_failed
       else:
           chop_commands(file->cmds)  ← 分割命令行为数组
           if !touch_flag || cmds->any_recurse:
               execute_file_commands(file)
                   ├─ initialize_file_variables(file)  ← 构建作用域链
                   ├─ set_file_variables(file)          ← 注入 $@ $* $< $^ $? $| $+
                   ├─ 串行模式：fork() + exec() 或 spawn()
                   └─ 并行模式：向 jobserver 申请 token，获得后 fork()
               return 0  ← 并行模式：命令发出但未完成，返回事件循环
           else (touch 模式):
               file->update_status = us_success
               notice_finished_file(file)
```

### 6.4 叶子节点的三种归宿

```
① 磁盘上存在的源文件（foo.c），无 deps，无 cmds，非 target
   update_file_1:
     noexist=0, must_make=0
     cmds==NULL → try_implicit_rule → 不命中（没有以 .c 为目标的规则）
     deps==NULL → 无依赖递归
     最终 must_make=0 → notice_finished_file → return 0   ✓ 叶子节点

② 不存在的源文件（missing.h），无 deps，无 cmds，非 target
   update_file_1:
     noexist=1, must_make=1
     cmds==NULL → try_implicit_rule → 不命中
     deps==NULL
     最终 must_make=1 → remake_file → cmds==0 && !is_target
       → complain() → "No rule to make target 'missing.h'"
       → update_status = us_failed   ✗

③ .PHONY 目标（clean），有 cmds，phony=1
   update_file_1:
     this_mtime = NONEXISTENT_MTIME（snap_deps 阶段强制设置）
     noexist=1, must_make=1
     phony=1 → 跳过隐式规则搜索
     deps 若有：递归处理（clean: distclean 时递归进 distclean）
     最终 must_make=1 → remake_file → execute_file_commands   ✓
```

---

## 阶段七：收尾

```
update_goal_chain 返回 update_status 后：

die(makefile_status)
│
├─ while job_slots_used > 0: reap_children(1, err)
│     ★ 等待所有并行子进程完成
│
├─ remote_cleanup()
│
├─ remove_intermediates(0)
│    ★ 不是在 notice_finished_file 里！是在这里统一删除
│    遍历 files 哈希表：
│    条件：f->intermediate
│          && (f->dontcare || !f->precious)
│          && !f->secondary
│          && !f->cmd_target          ← 命令行指定的目标不删
│          && f->update_status != us_none  ← 必须真正被构建过才删
│    unlink(f->name)
│
├─ print_data_base()   （若 -p）
│
├─ clean_jobserver(status)
│
└─ exit(status)
     MAKE_SUCCESS = 0
     MAKE_TROUBLE = 1  （-q 模式，需要更新）
     MAKE_FAILURE = 2  （构建失败）
```

---

## 核心数据流总结

```
envp[] + argv[]
     │
     ▼
main() 初始化
  install_default_suffix_rules()    ← 旧式后缀规则（读前装入）
  define_default_variables()        ← 内置变量（读前装入）
     │
     ▼
read_all_makefiles()
  eval_makefile() → eval()
     │
     ├──────────────┬──────────────┐
     ▼              ▼              ▼
  variable.c      rule.c         file.c
  变量哈希表     pattern_rules   files 哈希表
  (global_set)   链表             (struct file)
     │              │              │
     └──────────────┴──────────────┘
                    │
                    ▼
            snap_deps()                 ← 特殊属性传播
            convert_to_pattern()        ← 后缀规则转换
            install_default_implicit_rules()  ← 内置 pattern（读后装入）
            snap_implicit_rules()       ← 统计 max_pattern_*
                    │
                    ▼
          update_goal_chain(goals)
                    │
                    ▼
          update_file()               ← DAG 剪枝（considered）
            └─ update_file_1()        ← 核心：mtime/deps/commands
                  ├─ try_implicit_rule() / pattern_search()
                  ├─ check_dep() → update_file()  ← 递归
                  └─ remake_file()
                        ├─ complain()             ← "No rule" 错误
                        └─ execute_file_commands()
                              ├─ initialize_file_variables()
                              ├─ set_file_variables()  ← $@ $* $< $^ 等
                              └─ fork() / exec()
                    │
                    ▼
              die()
                remove_intermediates()
                exit()
```
