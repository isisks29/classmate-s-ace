================================================================================
Ghidra 全面分析结果包 — ace -第四课-授权靶场.dylib
================================================================================

分析对象
--------------------------------------------------------------------------------
文件名    : ace -第四课-授权靶场.dylib
文件格式  : Mach-O 64-bit arm64 dynamically linked shared library (MH_DYLIB)
CPU       : arm64 (Apple Silicon / iOS)
大小      : 4,206,864 字节 (~4.0 MB)

分析工具  : Ghidra 12.1.4 (PUBLIC, 2026-09-21 build)
运行环境  : Ubuntu 22.04 x86_64 + OpenJDK 21 (Temurin)
分析模式  : Ghidra Headless (analyzeHeadless) 全量自动分析
           (分析总耗时约 240s；含 Objective-C Message Analyzer、Stack、
            Reference、Function ID、Mach-O Function Starts 等分析器)

关键概况
--------------------------------------------------------------------------------
- 函数数        : 3,209
- 定义符号数    : 12,780
- 导入符号数    : 557 (Ghidra 视角) / 358 (Mach-O symtab 权威视角)
- 导出符号(权威): 16 个
- 内存块(段)    : 39 个
- 字符串        : 2,089 个
- 代码引用      : 完整函数 XREF 列表
- 数据引用      : 完整数据 XREF 列表

【重要发现摘要】
--------------------------------------------------------------------------------
1. 该 dylib 是一个 iOS 注入/钩子框架（训练靶场），核心依赖 Dobby 内联钩子库：
   - 导出符号含 _DobbyDestroy / _DobbyInstrument / _DobbyCodePatch /
     _DobbySymbolResolver / _dobby_set_options / _dobby_set_near_trampoline /
     _dobby_register_alloc_near_code_callback
   - 含 closure bridge / trampoline asm (closure_bridge_asm, closure_trampoline_asm)
   - 含 SkyEye UI 钩子: _JZSkyEye_UpdateViewReplacement / _JZSkyEye_UpdateViewTrampoline
   - 含 _pm_poolmin_prepare (内存池)
2. 使用 SAMKeychain (com.samsoffes.samkeychain) 做 Keychain 凭据存取，
   并出现 encrypted_key 字符串 —— 疑似用于授权/凭证校验。
3. 内嵌完整 Metal 着色器源码 (metal_stdlib, vertex_main / fragment_main)，
   表明其以 Metal 渲染 UI 覆盖层。
4. 导入含 CC_SHA256、Security.framework、CFNetwork、SystemConfiguration、
   UIKit、Metal/MetalKit、CoreGraphics(PDF 上下文) 等 —— 典型 App 注入框架依赖。
5. 数据段含 __objc_classlist / __objc_nlclslist / __objc_catlist /
   __objc_protolist，字符串区含 __objc_methname / __objc_classname / __objc_methtype。

================================================================================
包内文件清单 (每个类别一个独立 txt)
================================================================================

00_overview.txt            分析总览 (Ghidra 版本、语言、导入导出/函数/段数量统计)
--------------------------------------------------------------------------------
一、符号与命名空间 (Symbol & Namespace)
01_symbols.txt             完整符号树 (定义符号, 地址排序)
02_imports.txt             导入 (Ghidra 外部符号 + 引用位置)
03_exports.txt             导出 (全局/入口符号)
04_functions.txt           函数列表 (地址/大小/符号名)
05_classes_namespaces.txt  类与命名空间 (按父命名空间统计)
06_macho_metadata.txt      [见二] 亦含 Ghidra 视角段/节
--------------------------------------------------------------------------------
二、Mach-O 元数据与内存布局
06_macho_metadata.txt      段与节 (Ghidra 内存块) + 原始 Mach-O 加载命令
                           (LC_SEGMENT_64 全部节、LC_SYMTAB、LC_DYSYMTAB、
                            LC_LOAD_DYLIB 依赖、符号表权威导入/导出)
07_linking_fixups.txt      链接与修复 (导入符号及其引用站点 / import sites)
--------------------------------------------------------------------------------
三、代码与数据引用 (XREF)
08_code_xrefs.txt          代码引用 (函数被谁调用)
09_data_xrefs.txt          数据引用 (数据/字符串/常量被谁引用)
--------------------------------------------------------------------------------
四、字符串与常量
10_strings.txt             全部字符串 (2,089 个)
11_objc_methods.txt        Objective-C 方法名/selector 字符串
12_ui_text_errors.txt      UI 文本与错误信息 (error/fail/invalid/denied/alert 等)
13_hardcoded_keys_paths.txt 硬编码密钥/路径/URL (keychain/key/secret/token/path/url 等)
--------------------------------------------------------------------------------
五、控制流图与函数细节
15_disassembly.txt         反汇编列表 (全部函数 arm64, 含机器码字节)
16_pseudocode.txt          反编译伪代码 (全部函数 C 伪码, 188k 行)
17_control_flow.txt        函数图细节 (签名/返回类型/参数/局部变量/调用约定/属性)
--------------------------------------------------------------------------------
六、语言特定元数据
18_language_metadata.txt   Objective-C (类/元类/分类/协议/selector 符号)
                           与 Swift (mangled 符号) 元数据

================================================================================
使用说明
--------------------------------------------------------------------------------
- 文件为 UTF-8 纯文本，可直接用文本编辑器打开；大文件 (15/16) 建议用支持
  大文件的编辑器或命令行 (grep/awk/less) 查看。
- 地址均为虚拟地址 (image base 0x00000000)。
- 伪代码中 "FUN_xxxxxxxx" 为 Ghidra 未识别的内部函数；有名称的为符号已识别
  (导出/ObjC selector/导入)。

================================================================================
补充说明 (数据来源与口径)
--------------------------------------------------------------------------------
- "导出符号(权威)" 直接解析 Mach-O LC_SYMTAB 中外部定义 (N_EXT|N_SECT) 符号，
  共 16 个，见 06_macho_metadata.txt。
- 导入有两份口径：Ghidra 外部符号 557 个 (02/07)，Mach-O 符号表未定义符号 358 个
  (06) —— 差异因 Ghidra 会将 ObjC 类引用/selector 等额外建符号。
- 加载命令原始解析见 06_macho_metadata.txt 后半段 (29 条命令)。
