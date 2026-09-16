# 多语言、资源、内嵌库与调试工具

## 1. Locales（`Locales/`）

文件：`Locales.xml` 按 enUS → deDE → zhCN → zhTW → ruRU → frFR 加载。机制是**自研**的，不是 AceLocale：

```lua
-- zhCN.lua 等
local _, ns = ...
local _, _, L = unpack(ns)
if GetLocale() ~= "zhCN" then return end
L["From"] = "来自"
```

- `enUS.lua` 的守卫被注释掉，**总是加载**，充当兜底；后加载的同 key 覆盖。
- 键 = 英文原文；值支持 `|n` 换行、`|cffff0000..|r` 颜色、`%s` 格式化。
- 6 个文件当前各 861 行、条目位置一一对齐——这是刻意的维护惯例，新增条目放到各文件相同相对位置。
- 翻译者署名在文件头注释（deDE: DlargeX；zhTW: EK；ruRU: unw1s3, hacktivist；frFR: Zeddicus40）。
- 客户端语言判断用 `DB.Client = GetLocale()`（zhCN/zhTW 有专属分支，如 Changelog 弹窗仅 zhCN）。

## 2. Media（`Media/`）

只有纹理，无字体文件（字体用暴雪全局：`DB.Font = {STANDARD_TEXT_FONT, 12, "OUTLINE"}`）。

- 根目录：`normTex / gradTex / flatTex / bgTex / glowTex / pushed.blp / Inner-Shadow / TargetArrow / arrows.png` 等。
- `Media/Hutu/`（插画）：logo、star、Flag、Afdian/Patreon/Curseforge 徽标、`Menu/`（微菜单图标）。
- **引用方式**：全部在 `Core/Database.lua` 里挂到 DB 键（`DB.normTex`、`DB.starTex = Media.."Hutu\\star"`、`DB.MicroTex = Media.."Hutu\\Menu\\"`…），代码只消费 `DB.*Tex`，不写死路径。新资产照此登记。
- 唯一的外部媒体注册：Skins.lua 把 `normTex` 注册进 LibSharedMedia 供第三方读取。

## 3. 内嵌库（`Libs/`，Libs.xml 顺序加载）

| 库 | 状态 | NDui 接入点 |
|---|---|---|
| LibStub / CallbackHandler-1.0 | 原版 | 基础设施 |
| LibCustomGlow-1.0-**NDui** | 分叉 | PLAYER_LOGIN 时挂为 `B.ShowOverlayGlow / B.HideOverlayGlow` |
| LibActionButton-1.0-**NDui** | 分叉 | ActionBar 模块 `LAB:CreateButton(...)` |
| LibBase64 | 已改（直接 unpack ns） | ProfileGUI 档案导入导出 |
| cargBags | 核心+`base-add`/`mixins-add` NDui 扩展 | `ns.cargBags` → Bags 模块 `NDui_Backpack`；插件：searchBar/bagBar/bagTabs/tagDisplay |
| oUF 9.5.7 | 补丁级修改（`-- NDui mod` 标记） | `ns.oUF` → UFs 模块；`ns.oUF.Private`；禁用暴雪框体逻辑已内建 |

注意：改 Libs 里的分叉库时确认是 NDui 分叉（MAJOR 带 `-NDui` 或有 `-- NDui mod` 标记），上游更新需手工合并。`Libs/oUF/Plugins/RaidAuras.lua` 未被 oUF.xml 引用（活代码在 `Modules/UFs/Elements/RaidAuras.lua`）。

## 4. 调试工具（`Core/DevTools.lua`）

- 前置：角色名在 `DB.Devs` 表内 → `DB.isDeveloper`。
- `/rl` 重载；`/ng` 网格；`/mm` 布局调试器（Mover 模块）；`/nt` 枚举鼠标下提示信息；`/nf` 鼠标下框架；`/ns` 法术信息；`/getid` `/getnpc` `/getenc` `/getfont` `/gettiersets` 等查 ID 工具。
- `TOGGLEGRID` 为网格开关的 SlashCmdList 名。
- 错误展示靠随包分发的 `!BaudErrorFrame`（独立插件，不在 NDui TOC 内）。
- `/ndui` 重新打开首启教程/帮助（Tutorial 模块 `SlashCmdList["NDUI"]`）。

## 5. 工程环境

- `.vscode/settings.json`：Lua 5.1 + ketho wow-api 注解库，globals 白名单——新全局要加进去才不报红线（本插件基本不产生新全局）。
- 语言服务器不能解析游戏内 API 语义，存在误报属正常；以游戏内 `/rl` 实测为准。
- 仓库根 `D:\git_repo\NDui`，插件目录 `Interface/AddOns/NDui`；打 release 包时 CurseForge 读根目录 `NDui.toc` 的元数据。
