# SR4R_I18N — Saints Row IV 外挂汉化 DLL 技术方案

## 1. 项目目标

为 Saints Row IV（Steam EOS 2024+ 版）实现运行时外挂式中文汉化，架构参照已交付的 SR3R_I18N（黑道圣徒3重制版）。

**核心原则**：不动任何游戏资源文件，DLL 卸载即完全恢复原版。

---

## 2. 引擎与版本背景

| 项目 | SR3R（已交付） | SR4（本工程） |
|------|----------------|---------------|
| 游戏名 | Saints Row: The Third Remastered | Saints Row IV |
| 主程序 | SRTTR.exe (x64) | sr_hv.exe (x64) |
| 引擎分支 | SR35_GGP (2020 重制版) | SR35_GGP (2013 原版) |
| imagebase | 0x140000000 | 0x140000000（相同） |
| .text 段大小 | — | 0x013FEE9C (~20MB) |
| 主程序大小 | — | 28.8 MB |
| vpp 格式版本 | version 6 (LZ4, 0x1000 对齐) | **version 10** (zlib 无 adler, 无对齐) |
| le_strings 格式 | 12B 头 + 16B bucket + 8B 条目 | **完全相同** |
| vf3 字体格式 | TNFV v4 | **完全相同** |
| ASI Loader | binkw64.dll 3.6MB 自带 | 原版 binkw64.dll 无 loader（用户已解决注入） |
| CJK 原生支持 | 无 | **有**：jap.vf3_pc + charlist_jp.dat (1893 汉字) |

**关键结论**：同引擎同架构，le_strings/字体格式完全兼容，SR3R 的成熟代码可以大规模移植。

---

## 3. SR4 vpp_pc 格式（已逆向验证）

```
Header (0x28 字节):
  +0x00  u32  magic    = 0x51890ACE
  +0x04  u32  version  = 10
  +0x08  u32  crc?
  +0x0C  u32  ?
  +0x10  u32  flags?
  +0x14  u32  count    (文件数)
  +0x18  u32  dirSize  (目录区字节数 = count * 24)
  +0x1C  u32  nameSize (名字区字节数)

Directory (count × 24 字节):
  +0x00  u32  nameOff    (名字区偏移)
  +0x04  u32  (0)
  +0x08  u32  dataOff    (数据区偏移)
  +0x0C  u32  usz        (解压后大小)
  +0x10  u32  csz        (压缩后大小)
  +0x14  u32  flags

Names: 紧跟目录，每个 \0 结尾
Data:  紧跟名字区（无 0x1000 对齐），每个文件 = zlib 流（无 adler32 尾部）

解压方法: zlib.decompressobj().decompress(blob)  (eof=False 属正常)
未压缩特例: csz == usz 且数据非 0x78 开头 → 直接存储
```

工具：`Tools/sr4_vpp.py`（已验证：misc.vpp_pc 317 文件全部解包成功）

---

## 4. 资源概况

### 4.1 misc.vpp_pc 内容（317 文件）

| 类别 | 数量 | 说明 |
|------|------|------|
| le_strings | 210 | 多语言 cz/de/es/fr/it/jp/nl/pl/ru/us，无 zh |
| 字体 (vf3_pc) | 6 | editor-default/font_body/font_header/font_header_pc/font_sk/jap |
| 字体图集 (gvbm_pc) | 12 | 上述字体 + 对应 _nobdr 版本 |
| charlist (dat) | 12 | 各语言字符映射表 |
| cpeg/gpeg | ~30 | UI 纹理 |
| lua | 2 | game_lib.lua, sr3_city.lua |

### 4.2 le_strings 分类（us 版本）

| 文件 | 条目数 | 说明 |
|------|--------|------|
| menu_us | 900 | 菜单文本 |
| subtitle_us | 603 | 字幕 |
| mission_us | — | 任务文本（最大，196KB） |
| hud_us | — | HUD |
| voice_us | — | 语音文本（1.7MB） |
| customize_us | — | 自定义 |
| new_sr35_us | — | SR35 新增文本 |
| platform_pc_us | — | PC 平台文本 |
| static_us | — | 静态文本 |
| 等 | — | 另有 dlc1-7, multiplayer, diversion 等 |

### 4.3 字体信息

| 字体 | vf3 大小 | gvbm 大小 | 说明 |
|------|----------|-----------|------|
| font_body | 16KB | 256KB | 正文字体 |
| font_header_pc | 19KB | 1.3MB | 标题字体 |
| font_sk | 39KB | 2.7MB | 大标题 |
| jap | 58KB | 2.7MB | 日文字体（含 CJK） |
| editor-default | 6KB | 64KB | 编辑器字体 |
| ug-debug | 6KB | 85KB | 调试字体 |

vf3 头格式：`TNFV` (magic) + version 4 + 变长布局（与 SR3R 完全相同）

### 4.4 charlist 分析

| 文件 | 映射项数 | CJK 汉字数 | 说明 |
|------|----------|-----------|------|
| charlist_jp.dat | 2377 | **1893** | 完整日文汉字集 |
| charlist_sk.dat | 1514 | 0 | 韩文 |
| charlist_us.dat | 284 | 0 | ASCII + 欧洲字符 |

**注意**：虽然 SR4 引擎原生支持 CJK（jap 字库），但用户选择了外挂 DLL 自建图集方案（与 SR3R 一致），不复用 jap 字库。

---

## 5. 方案架构

采用与 SR3R 相同的混合架构：

```
┌─────────────────────────────────────────────┐
│              SR4R_I18N.asi                  │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │  文本层 (Text Layer)                │    │
│  │  - 词典加载 (le_strings txt 格式)   │    │
│  │  - CRC32 开链哈希 (65536 桶)        │    │
│  │  - 128MB 字符串 arena               │    │
│  │  - Hook A: DrawWide   (文本绘制)    │    │
│  │  - Hook B: Format     (格式化)      │    │
│  │  - Hook J: Subtitle   (字幕入口)    │    │
│  │  - Hook F/G: LangCur/LangTxt        │    │
│  │           (语言服务整句替换)        │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │  字形层 (Glyph Layer)               │    │
│  │  - stb_truetype 光栅化思源黑体      │    │
│  │  - 伪字体对象 (count 扩为 0xFFE0)   │    │
│  │  - 自建图集 D3D11 纹理              │    │
│  │  - 读回官方图集 BC 解码 + 拼接      │    │
│  │  - Hook C: FontLookup (字体查询)    │    │
│  │  - Hook D: TexObj    (纹理查询)     │    │
│  │  - Hook E: SrvResolve (SRV 解析)    │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │  安全机制                           │    │
│  │  - 16 字节特征码校验 (防更新错位)   │    │
│  │  - 词典空 → idle 模式               │    │
│  │  - 字体构建失败 → 回退官方          │    │
│  │  - kernStart=-1 铁律                │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

---

## 6. Hook 定位状态（全部完成）

### 6.1 函数调用拓扑桥接法

SR3R 和 SR4 使用不同编译器版本/选项，寄存器分配和指令选择完全不同，字节模式匹配不可靠。
但函数调用拓扑（谁调用谁）完全一致。通过 SR3R→SR4 的函数对应关系链式推导，精确定位目标函数。

### 6.2 全部 7 个 Hook 定位结果

| Hook | 名称 | SR3R VA | SR4 VA | 入口特征码 | 状态 |
|------|------|---------|--------|-----------|------|
| A | DrawWide | `0x1408B5FF0` | `0x140DC36A0` | `40 53 56 57 48 81 EC 10 01 00 00 8B BC 24 60 01` | ✅ |
| B | Format | `0x140812610` | `0x140CF9A00` | `40 55 56 57 41 57 48 8D AC 24 78 D0 FF FF` | ✅ |
| C | FontLookup | `0x140859B10` | `0x140BF8550` | `83 F9 FF 7D 3E 8D 81 FF FF FF 7F 83 F8 FF 7E 2E` | ✅ |
| D | TexObj | `0x14085DB30` | `0x140B7AF30` | `4C 63 C1 85 C9 78 77 44 3B 05 8E A3 D9 05 7D 6E` | ✅ |
| E | SrvResolve | `0x140915D60` | `0x140E2C9E0` | `40 53 48 83 EC 20 0F B6 DA 83 F9 FF` | ✅ |
| F | LangCur | `0x140812060` | `0x140CF1980` | `48 8B 05 B1 DD 4C 06 48 85 C0 74 0C 48 8B 50 08` | ✅ |
| G | LangTxt | `0x140812040` | `0x140CF1960` | `48 8B 05 D1 DD 4C 06 48 85 C0 74 0B 48 8B 10` | ✅ |
| J | Subtitle | `0x1402D2BC0` | `0x140476D80` | `40 53 55 56 57 41 54 41 55 41 56 41 57 48 81 EC F8 00 00 00 48 8B 05 0D 85 56 01 48 33 C4 48 89` | ✅ |

### 6.3 SR4 架构变更（重要适配点）

#### Subtitle (Hook J) 架构变更
- SR4 字幕文本从 UTF-16 改为 **UTF-8**（`char*` vs `wchar_t*`）
- SR4 Subtitle (975B) 比 SR3R (1094B) 小 — 不含 `\n`+毫秒时长解析逻辑和 `0x2711` 计数器
- SR4 Subtitle 不直接调 DrawText，绘制逻辑移到独立的渲染函数
- SR4 不再引用 `byte_1415AD4B0`（字幕激活标志），改用 `dword_1417E3E6C` 状态机

#### SrvResolve (Hook E) 架构变更
- SR3R 从 SRV 全局表（`qword_1415456F0[+0x28]`，64字节步长 `shl rax, 6`）查找 SRV
- SR4 改为直接从 `texObj+0x38` 读取 QWORD 指针，跳过全局表索引
- 返回值偏移：SR3R `[entry+0x30]`/`[entry+0x38]` → SR4 `[ptr+8]`/`[ptr+0x10]`（ptr = entry+0x28）

#### 字体对象布局差异
- SR3R font 对象 texId 在 +184 偏移，SR4 改为 **+568** 偏移

### 6.4 完整函数映射表

| 功能 | SR3R | SR4 |
|------|------|-----|
| FontLookup (C) | `sub_140859B10` | `sub_140BF8550` |
| DrawWide (A) | `sub_1408B5FF0` (1333B) | `sub_140DC36A0` (1333B) |
| TexObj (D) | `sub_14085DB30` | `sub_140B7AF30` |
| TexObj wrapper | `sub_14085D930` (39B) | `sub_140B7AC00` (39B) |
| TexObj query (no anim) | — | `sub_140B7AC30` (0x7c) |
| Streaming tex resolver | `sub_1408FE2F0` | `sub_140D98D30` |
| SrvResolve (E) | `sub_140915D60` (259B) | `sub_140E2C9E0` (101B) |
| Tex flag query | `sub_140D313C0` (195B) | `sub_140DFD410` (75B) |
| Format (B) | `sub_140812610` (0x6d2) | `sub_140CF9A00` (0x78c) |
| LangCur (F) | `sub_140812060` | `sub_140CF1980` |
| LangTxt (G) | `sub_140812040` | `sub_140CF1960` |
| SrvResolve cache | `sub_140916DD0` | `sub_140E0F7D0`/`E0FCD0`/`E0FF30` (3变体) |
| Subtitle (J) | `sub_1402D2BC0` (0x446) | `sub_140476D80` (0x3cf) |
| 折行 wrapper | `sub_14085A1F0` (83B) | `sub_140BF8D40` (80B) |
| 折行核心 | `sub_140859620` | `sub_140BE8830` |
| 测量回调 | `sub_140859C80` | `sub_140BF8760` (27B) |
| 测量函数 | — | `sub_140BE7D60` |
| 测量 wrapper | — | `sub_140BF8740` (14B) |
| D3D11 创建 | `sub_140921C80` | `sub_140E4F1A0` |

### 6.5 全局变量映射表

| SR3R 全局变量 | SR4 全局变量 | 用途 | 状态 |
|---------------|-------------|------|------|
| `qword_142998650` | `qword_1469152D8` | 纹理表 (24字节步长) | ✅ |
| `dword_142998660` | `dword_1469152CC` | 纹理计数 | ✅ |
| `dword_14295B650` | `dword_146AEA4A0` | 随机种子 | ✅ |
| `qword_143070F02` | `qword_1473F0992` | streaming tex 计数 (u16) | ✅ |
| `xmmword_143070F10` | `xmmword_1473F09A0` | streaming tex 表 (8字节步长) | ✅ |
| `qword_142906D28` | `qword_1471BF738` | 语言服务对象 (vtable[0]=LangTxt, [1]=LangCur, [3]=Symbol) | ✅ |
| `qword_1430784F8` | `qword_147667C70` | D3D11Device* | ✅ |
| `qword_143078500` | `qword_147667C78` | D3D11DeviceContext* | ✅ |
| `qword_1415456F0` | **不再需要** | SRV全局表 — SR4 SrvResolve 改用 `[texObj+0x38]` 直接指针 | ✅ |
| FONTTAB | `qword_146AE7060` | 字体表 | ✅ |
| FONTCOUNT | `dword_146AE504C` | 字体计数 | ✅ |
| `qword_142998658` | ? | SRV cache (8字节步长) | ❌ 待定位 |

---

### 7.1 可直接复用（引擎无关）

| 模块 | 说明 |
|------|------|
| 词典层 | CRC32 哈希 + arena + 查表逻辑（与游戏无关） |
| 配置解析 | ini 手工解析（UTF-8 → UTF-16） |
| 字符集管理 | g_charSet bitmap + 频率排序 |
| DumpText | 未命中收集 + 批量落盘 |
| 日志系统 | CRITICAL_SECTION 保护的 fprintf |
| 主线程框架 | DllMain → MainThread → 安装 hook 的流程 |
| 安全机制 | 16 字节特征码校验 + idle 回退 |
| 折行重组状态机 | v7.3 WrapState（备用，如果 Hook J 足够则可省略） |

### 7.2 需要适配（SR4 地址/布局不同）

| 模块 | 改动 |
|------|------|
| 所有 VA 常量 | 已重新计算（见 §6.2） |
| 所有特征码 | 已提取（见 §6.2，7/7 完成） |
| MAGIC texId 安全区 | 理论可复用（0x60000000 段），需验证 SR4 纹理注册数 |
| 字体对象布局 | texId 偏移 +184 → **+568** |
| 纹理对象布局 | SrvResolve 改用 `[texObj+0x38]` 直接指针 |
| D3D11 全局指针 | 已定位：`qword_147667C70` (device) / `qword_147667C78` (context) |
| FONTTAB / FONTCOUNT | 已定位：`qword_146AE7060` / `dword_146AE504C` |
| Subtitle 文本编码 | UTF-16 → **UTF-8**（`char*` vs `wchar_t*`） |
| SrvResolve 架构 | 不含 SRV 全局表，改用 `[texObj+0x38]` 直接指针 |

### 7.3 需要从 SR3R 移植的文件

| 文件 | 来源 | 说明 |
|------|------|------|
| dllmain.cpp | SR3R_I18N/dllmain.cpp | 核心代码（~2800 行），改 VA + 特征码 |
| stb_truetype.h | SR3R_I18N/ | 字体光栅化库（不改动） |
| charset_data.h | SR3R_I18N/ | 内置字符集（font-only 模式备用） |
| pch.h / pch.cpp | SR3R_I18N/ | 预编译头 |
| framework.h | SR3R_I18N/ | Windows 头 |

### 7.4 已完成/待完成的工具

| 工具 | 说明 | 状态 |
|------|------|------|
| sr4_vpp.py | VPP v10 解包器 | ✅ 已验证 |
| sr4le_extract.py | le_strings → txt 解包 | ✅ 已验证（210文件/273028条） |
| sr4le_repack.py | txt → le_strings 回写 | ❌ 待建 |
| sr4_vpp_pack.py | txt → vpp_pc 打包 | ❌ 待建 |
| charlist 生成 | SR4 简体用字 charlist.txt | ❌ 待建 |
| exe_hardcoded 提取 | exe 内硬编码字符串 | ❌ 待做 |

---

## 8. 实施计划

### 阶段 1：文本工具链（无 DLL，纯 Python）

**目标**：建立 le_strings 解包/翻译/回写工具链

1. 移植 `sr3le_extract.py` → `sr4le_extract.py`
   - le_strings 格式完全相同，主要改动是 charlist 文件路径和 xtbl 目录
   - 用 SR4 misc.vpp_pc 解包后的 le_strings + misc_tables.vpp_pc 的 xtbl 测试
2. 移植 `le_strings_repack.py` → `sr4le_repack.py`
   - 验证回写后的 le_strings 二进制与原版结构一致
3. 从 misc.vpp_pc 中提取全部 us 版 le_strings → txt
4. 准备翻译工作流（KEY 翻译 → 回写 → 打包 vpp_pc）
5. 移植 `vpp_pack.py` → `sr4_vpp_pack.py`（SR4 v10 格式打包）

### 阶段 2：Hook 定位（IDA 逆向）

**目标**：在 sr_hv.exe 中定位全部 10 个 hook 点

1. 用 IDA 打开 sr_hv.exe（run_auto_analysis=false 加速）
2. 确认 DrawWide / Format / FontLookup 函数体与 SR3R 一致
3. 从 FontLookup 调用链定位 TexObj / SrvResolve
4. 筛选 Subtitle / LangCur / LangTxt 正确候选
5. 定位 FONTTAB / FONTCOUNT / D3D_DEVICE / D3D_CONTEXT 全局变量
6. 确认字体对象 / 纹理对象内存布局与 SR3R 是否一致
7. 提取所有 hook 的 16 字节特征码

### 阶段 3：DLL 代码移植与编译

**目标**：完成 SR4R_I18N.asi 编译

1. 复制 SR3R dllmain.cpp → SR4R dllmain.cpp
2. 替换所有 VA 常量（§6 + §7.2）
3. 替换所有特征码
4. 适配字体对象布局差异（如有）
5. 修改日志名/配置名/版本标识为 SR4R
6. 配置 vcxproj（v141_xp + /MT + C++17/20 + MinHook NuGet）
7. 编译 Release|x64

### 阶段 4：部署与测试

**目标**：实测汉化效果

1. 准备测试词典（少量菜单+字幕文本翻译）
2. 部署 DLL + 字体 + 词典到游戏目录
3. 用户本地测试：
   - 词典命中 → 文本替换是否正常
   - CJK 文本 → 字体图集是否正确构建
   - 字幕 → 整句替换是否正常
   - 稳定性 → 长时间运行是否崩溃
4. 检查日志（SR4R_I18N.log）排查问题
5. 修复迭代

### 阶段 5：完整翻译

**目标**：全部 le_strings 汉化

1. 提取全部 us 版 le_strings → txt
2. 翻译全部文本（机器翻译初稿 + 人工校对）
3. 软化敏感内容（如有）
4. 回写 le_strings → 打包 misc.vpp_pc
5. 与外挂 DLL 词典合并部署
6. 最终实测

---

## 9. 风险与对策

| 风险 | 概率 | 影响 | 对策 |
|------|------|------|------|
| 字体对象布局不同 | 中 | 高 | IDA 逆向确认；如有差异则适配代码 |
| MAGIC texId 不安全 | 低 | 高 | 验证 SR4 纹理注册数；必要时换安全区段 |
| TexObj/SrvResolve 函数不存在 | 低 | 高 | 从 FontLookup 调用链定位等效函数 |
| 引擎全局变量地址变化 | 高 | 中 | IDA 重新定位 |
| vpp 打包格式差异 | 低 | 中 | 已逆向验证，回写测试覆盖 |
| le_strings 回写哈希不一致 | 低 | 低 | 验证格式完全相同，可复用 SR3R repack 逻辑 |
| 注入方式兼容性 | 低 | 高 | 用户已解决 |
| 游戏更新导致特征码失效 | 低 | 中 | 16 字节特征码 + idle 回退（设计已考虑） |

---

## 10. 配置规格

### SR4R_I18N.ini（计划）

```ini
[settings]
dict_dir = dict
font_file = SourceHanSansHWSC-VF.ttf
dump_enabled = 0
lang_early = 1
subtitle_early = 1
charlist_file = charlist.txt
early_diag = 0
```

### 部署目录结构（计划）

```
游戏根目录/
├── sr_hv.exe
├── binkw64.dll          (原版或用户自备 loader)
└── scripts/
    ├── SR4R_I18N.asi
    ├── SR4R_I18N.ini
    ├── SR4R_I18N.log
    ├── dict/
    │   └── *.txt        (词典)
    ├── charlist.txt
    └── SourceHanSansHWSC-VF.ttf
```

---

## 11. 编译环境

| 项 | 值 |
|-----|-----|
| IDE | VS2017 |
| 工具集 | v141 |
| 平台 | x64 only |
| 运行库 | /MT (MultiThreaded) |
| 标准 | C++17 (或 C++20) |
| 字符集 | Unicode |
| 预编译头 | Use/Create (pch.h) |
| 依赖 | MinHook 1.3.3 (NuGet), stb_truetype (内置), d3d11 (SDK) |
| 输出 | SR4R_I18N.asi (DLL 改后缀) |

---

## 12. 当前进度

### 已完成

- [x] 摸清 SR4 游戏目录、主进程（sr_hv.exe, x64）
- [x] 逆向 vpp 格式 (v10: zlib 无 adler, 无对齐)
- [x] 编写并验证 vpp 解包器 (sr4_vpp.py, 317 文件 0 失败)
- [x] 确认 le_strings 格式与 SR3R 完全相同
- [x] 确认 vf3 字体格式与 SR3R 完全相同
- [x] 分析 charlist (jap 有 1893 CJK 汉字，用户选择外挂字库路线)
- [x] 编写本方案文档
- [x] **阶段 1：le_strings 提取工具 sr4le_extract.py 完成**（210文件/273028条字符串/18281个xtbl键名反查）
- [x] **阶段 2：全部 7 个 hook 定位完成**（A/B/C/D/E/F/G/J）
- [x] 全部 7 个特征码提取完成
- [x] 全局变量定位：纹理表、语言服务对象、字体表、D3D设备/上下文（SRV_CACHE 待定位）
- [x] SR4 架构变更分析：Subtitle 编码改 UTF-8、SrvResolve 改直接指针、字体 texId 偏移 +568
- [x] le_strings 回写工具（sr4le_repack.py）
- [x] vpp 打包工具（sr4_vpp_pack.py）
- [x] exe_hardcoded 硬编码字符串提取（5528 条，已译 1436 条）
- [x] **阶段 3：DLL 代码移植与编译**（Release|x64，vs2017 v141）
- [x] **阶段 4：部署实测通过**（词典命中、CJK 字形注入、字幕整句替换均正常）
- [x] **阶段 5：完整翻译**（voice 100%、le_data_supplement 94%、exe_hardcoded 26%）
- [x] 字幕 Hook J 修正（sub_1403D18F0 正确入口）

### 待处理

- [ ] SRV cache 全局变量定位（低优先级，可能不需要）
- [ ] le_data_supplement 剩余 ~6% 技术性条目（单字符/格式标签/品牌名）
- [ ] exe_hardcoded 剩余技术字符串（动画状态/调试标志/硬件品牌，暂保留英文）
