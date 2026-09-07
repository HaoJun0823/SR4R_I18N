---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: 'e67f28ca-178c-4c15-9557-580018354d6a'
  PropagateID: 'e67f28ca-178c-4c15-9557-580018354d6a'
  ReservedCode1: '77c01c9e-267a-4891-bebc-6c1e7ecfd1f4'
  ReservedCode2: '77c01c9e-267a-4891-bebc-6c1e7ecfd1f4'
---

# SR4R_I18N — 黑道圣徒4（Saints Row IV）外挂式简体中文汉化

基于 dinput8/binkw64 代理注入的运行时汉化 DLL。**不动任何游戏资源文件**，卸载 DLL 即完全恢复原版。

> 项目代号：SR4R（Saints Row IV Re-Elected / EOS 2024+ 版）
> 架构参照已交付的 SR3R_I18N（黑道圣徒3重制版 v7.5.2）。
> 姊妹项目：**[SR3R_I18N](https://github.com/HaoJun0823/SR3R_I18N)**（黑道圣徒3重制版汉化，同作者同架构）。两仓库共享同一套外挂注入/字形注入/词典替换方法论，互可对照阅读。

---

## 功能特性

- **外挂式注入**：DLL 以 `SR4R_I18N.asi` 形式加载（binkw64 代理），游戏资源零改动
- **词典命中替换**：多级查表（CRC32 哈希 + 原始键 + trim + 空格规范化），命中即换中文
- **整句替换**：语言服务返回层（`lang_early`）+ 字幕绘制入口（`subtitle_early`）双通道，折行前换整句
- **运行时 CJK 字形注入**：stb_truetype 光栅化思源黑体 → 自建 D3D11 图集，无需替换游戏字体文件
- **自动字符集**：按 `charlist.txt` 收集用字，动态构建字形缓存
- **安全回退**：16 字节特征码校验 + 词典为空自动进入 font-only 模式 + 字体构建失败回退官方字体
- **未命中收集**：`dump_enabled=1` 时把引擎送达的未命中文本写入 `DumpText.dtxt`，便于补译

## 翻译覆盖

| 词典 | 条目 | 翻译率 | 内容 |
|------|------|--------|------|
| `voice_001~017.txt` | 15797 | 100% | 语音台词（含字幕） |
| `le_data_supplement.txt` | 12306 | ~94% | le_data 全部文本（菜单/HUD/任务/电台/字幕等） |
| `exe_hardcoded.txt` | 5528 | ~26% | 引擎硬编码字符串（面向玩家的已译，技术/调试/动画状态保留英文） |

## 注入机制（binkw64 代理 / ASI Loader）

与 SR3R 相同，采用 **binkw64 代理 DLL 注入**，不改 EXE、无第三方注入器：

1. 游戏根目录原版 `binkw64.dll` 重命名为 `binkw64Hooked.dll`
2. 放入同名 `binkw64.dll`（ASI Loader，代理 DLL）顶替原版
3. 代理在 `DllMain` 中 `LoadLibrary("binkw64Hooked.dll")` 转发原版函数，同时扫描 `scripts\` 目录加载所有 `*.asi`（即 `SR4R_I18N.asi`）
4. `SR4R_I18N.asi` 的 `DllMain` 安装各 hook（MinHook），文本替换与字形注入生效；删除代理 DLL 即完全恢复原版

> 注意：`binkw64.dll` 是**注入器**，不是汉化本体；汉化本体是 `scripts\SR4R_I18N.asi`。

## 安装

1. 将游戏目录下原有 `binkw64.dll` 改名备份为 `binkw64Hooked.dll`（若 Steam 版缺失则跳过）
2. 将 `SR4R_I18N.asi` 放到游戏根目录 `scripts/` 下
3. 将 `SR4R_I18N.ini`、`SourceHanSansHWSC-VF.ttf`、`charlist.txt` 放到 `scripts/` 下
4. 将 `Dict_CHS/` 下全部 `*.txt` 放到 `scripts/dict/` 下
5. 启动游戏

### 目录结构

```
游戏根目录/
├── sr_hv.exe
├── binkw64.dll             (代理 loader，原版改名 binkw64Hooked.dll)
└── scripts/
    ├── SR4R_I18N.asi       (汉化 DLL 本体)
    ├── SR4R_I18N.ini       (配置)
    ├── SR4R_I18N.log       (运行日志)
    ├── SourceHanSansHWSC-VF.ttf  (中文字体)
    ├── charlist.txt        (字符清单)
    ├── DumpText.dtxt       (未命中文本收集, ini 可关)
    └── dict/               (词典)
        ├── voice_001~017.txt
        ├── le_data_supplement.txt
        └── exe_hardcoded.txt
```

### 配置项（SR4R_I18N.ini）

| 键 | 默认 | 说明 |
|----|------|------|
| `dict_dir` | `dict` | 词典文件夹（相对 asi 目录） |
| `origin_dir` | `origin` | 联表文件夹（已弃用，可留空） |
| `font_file` | `SourceHanSansHWSC-VF.ttf` | 中文字体 TTF 文件名 |
| `charlist_file` | `charlist.txt` | 字符清单文件名 |
| `dump_enabled` | `1` | 收集未命中文本到 DumpText.dtxt |
| `lang_early` | `1` | 语言服务返回层整句替换 |
| `early_diag` | `1` | 早期替换诊断日志 |
| `subtitle_early` | `1` | 字幕绘制入口整句替换 |

## 构建

需要 VS2017（v141 工具集）+ MinHook 1.3.3（NuGet 自动还原，许可证见 `LICENSE-MinHook.txt`）。

```
MSBuild.exe SR4R_I18N/SR4R_I18N.vcxproj /p:Configuration=Release /p:Platform=x64 /t:Rebuild
```

产物：`SR4R_I18N/x64/Release/SR4R_I18N.dll` → 复制为 `SR4R_I18N.asi` 部署。

## 技术文档

详细逆向方案与 Hook 定位见 **[SR4R_I18N_TECH_SPEC.md](./SR4R_I18N_TECH_SPEC.md)**（vpp 格式、le_strings 格式、7 个 Hook 的 VA/特征码、函数/全局变量映射表）。

## 仓库结构

```
├── SR4R_I18N/           # DLL 源码（VS2017 工程）
│   ├── dllmain.cpp      # 主逻辑（词典/字形/Hook/安全机制 ~2600 行）
│   ├── stb_truetype.h   # 字体光栅化库
│   └── charset_data.h   # 内置字符集（font-only 模式备用）
├── Dict_CHS/            # 最终词典（部署到 scripts/dict/）
│   ├── voice_001~017.txt
│   ├── le_data_supplement.txt
│   └── exe_hardcoded.txt
├── le_data/             # le_strings 解包原文/汉化对照
│   ├── original/        # 英文原文（解包自 misc.vpp_pc）
│   └── schinese/        # 简体中文对照（译文桥接源）
├── Tools/               # Python 工具链
│   ├── sr4_vpp.py       # vpp_pc v10 解包器
│   ├── sr4_vpp_pack.py  # vpp_pc v10 打包器
│   ├── sr4le_extract.py # .le_strings → txt 解包器
│   └── sr4le_repack.py  # txt → .le_strings 回写器
├── Docs/glossary_voice.md  # 语音术语表（角色/名词统一命名）
├── LICENSE-MinHook.txt     # MinHook 第三方依赖许可证（BSD-2-Clause）
└── SR4R_I18N_TECH_SPEC.md  # 完整逆向技术方案
```

## 注意事项

- 仅支持 **Steam EOS 2024+（Re-Elected）x64** 版，主程序 `sr_hv.exe`
- 词典中的格式占位符（`[format]`、`{0}`、`%d` 等）需保持原样
- 引擎侧部分文本在送达时被折行/压缩空格，DLL 已做 trim + 规范化键匹配
- 汉化中黄赌毒等敏感内容已全部柔化替换

## License

本工程仅供学习交流使用，游戏资源版权归原厂商所有。

第三方依赖：
- **MinHook**（[TsudaKageyu/minhook](https://github.com/TsudaKageyu/minhook)，BSD-2-Clause）——见 `LICENSE-MinHook.txt`
- **stb_truetype**（Public Domain / MIT，单头文件随源码分发）