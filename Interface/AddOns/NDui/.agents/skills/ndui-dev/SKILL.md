---
name: ndui-dev
description: NDui（魔兽世界 Midnight 12.x 全功能 UI 整合插件）项目专属开发技能。凡涉及本 NDui 代码库的任何改动都应使用它：新增模块/功能/设置项/皮肤/Infobar 区块/locale 字符串、修复 bug、扩展 oUF 单位框体或 cargBags 背包、接入事件与 SavedVariables。Use whenever editing files under the NDui addon folder (Core/, Config/, Modules/, Plugins/, Locales/, Libs/, Init.lua, *.toc) — even for small tweaks — to follow NDui's B/C/L/DB namespace, module OnLogin lifecycle, settings DB layering, and GUI option conventions.
---

# NDui 开发技能

NDui 是魔兽世界 Midnight（12.x）全功能 UI 整合插件：动作条、单位框体(oUF)、背包(cargBags)、姓名板、聊天、美化(Skins)、Infobar、地图、Misc 玩意儿集合。本仓库为 mainline 单端（`## Interface: 120100`，无经典分支 TOC），版本号见 `NDui_Mainline.toc` 的 `## Version`。

本文件是地图与施工规范；细节按需读 references（都在本 skill 目录下）：

| 你要做的事 | 读 |
|---|---|
| 弄清加载顺序、ns 命名空间、事件枢纽、模块系统、设置数据库分层、GUI 选项系统的完整机制 | `references/architecture.md` |
| 找某个功能在哪个文件、各模块实现了什么、每个模块的扩展点在哪 | `references/modules.md` |
| "怎么新增 X"的分步配方（新设置项 / 新 Infobar 区块 / 新皮肤 / 新模块 / 新 locale 字符串 / 新 bag 过滤器…） | `references/recipes.md` |
| 多语言工作流、Media 资产、内嵌库(oUF/cargBags/LAB)、Plugins、调试命令 | `references/locales-media.md` |

## 铁律

1. **改元数据用 `NDui_Mainline.toc`**；根目录 `NDui.toc` 是 CurseForge 占位空文件（没有文件列表）。新增 .lua 文件要挂到所在目录的 `.xml`（如 `Modules\Misc\Misc.xml`），不要写进 TOC。
2. **每个文件以固定头部开始**，不新建任何全局（仅 SavedVariables 三个全局除外）：
   ```lua
   local _, ns = ...
   local B, C, L, DB = unpack(ns)
   ```
3. **新增持久化设置必须同步登记默认值**：写进 `Core/GUI.lua` 的 `G.DefaultSettings`（角色配置）或 `G.AccountSettings`（账号数据）。否则 `InitialSettings` 在下次登录会把它当"过期键"删掉。
4. **模块必须有 `OnLogin`**：PLAYER_LOGIN 时按注册顺序 xpcall 执行；漏写会打印 `Module <X> does not loaded.`。没有 OnEnable/OnInitialize。
5. **GUI 标签 `*` 后缀约定**：`L["xxx"].."*"` = 改动经回调即时生效；不带 `*` = 保存后弹 ReloadUI 提示。回调行才需要写 callback 函数。
6. **Midnight 12.0 Secret Values**：血量/蓝量/战斗数值可能是 secret——参与运算前用 `B:IsSecretValue()` / `B:NotSecretTable()` 防护（封装在 `Core/Functions.lua`）。CLEU 在 12.0 已对插件移除，`B:RegisterEvent("CLEU", ...)` 别名是遗留物，新代码不要依赖。
7. **像素完美**：偏移量乘 `C.mult`（由 `B:SetupUIScale` 维护）。可移动框体一律 `B:Mover(frame, L["标题"], "唯一Mover名", 默认锚点)`。
8. **文本一律 `L["English key"]`**：key 就是英文原文。改 locale 需同步 `Locales/` 下全部 6 个文件（enUS 守卫被注释、总是先加载，是兜底；各文件行数保持对齐是本项目惯例）。

## 命名空间

`Init.lua` 构造 `ns` 并发布为全局 `NDui`（`_G[addonName] = ns`）：

| 索引 | 名称 | 职责 | 持久化 |
|---|---|---|---|
| `ns[1]` | **B** (Basement) | 共享函数库：widget 构造、Reskin 系列、格式化、事件枢纽、模块注册 | 否 |
| `ns[2]` | **C** (Config) | `C.db`（活动设置表句柄）+ `Config/*.lua` 里的静态默认锚点/开关表 | `C.db` → SavedVariables |
| `ns[3]` | **L** (Locales) | 字符串表，`L["key"]` 直查 | 否 |
| `ns[4]` | **DB** (Database) | 运行期常量：颜色、字体、纹理路径、客户端信息、角色信息 | 否（名字有迷惑性） |

设置载体速查：

| 载体 | 默认值定义处 | 读取方式 | 典型内容 |
|---|---|---|---|
| `C.db[Section][Key]` | `G.DefaultSettings`（Core/GUI.lua） | 各模块功能开关与数值 | `"Actionbar"."Bar1Size"` |
| `NDuiADB[Key]` | `G.AccountSettings` | 账号级数据/设置（GUI 里 section 写 `"ACCOUNT"`） | `totalGold`, `ClickSets`, `ChatFilterList` |
| `C.Section.*`（非 db） | `Config/Modules.lua` 等静态表 | 直接读，不持久 | `C.Infobar.Gold`, `C.UFs.PlayerPos` |
| `NDuiPDB[1..5]` | 档案系统 | `NDuiADB["ProfileIndex"]` 指向其一；`C.db` 会指向它 | 档案槽 |

## 生命周期（一行版）

文件加载（建帧、注册事件、定义 OnLogin）→ `ADDON_LOADED("NDui")` 合并默认值/迁移/定 `C.db` 指向 → `PLAYER_LOGIN`：CVars、UI 缩放、依次执行所有 `module:OnLogin()`。

## 常用 B API 速查

- 事件：`B:RegisterEvent(event, func, unit1, unit2)` / `B:UnregisterEvent`（全局枢纽帧分发，xpcall 包裹；支持 unit event；`"CLEU"` 别名遗留）。
- 模块：`local M = B:RegisterModule("Name")`（核心文件）/ `local M = B:GetModule("Name")`（同模块其他文件）；子功能清单模式 `M:RegisterMisc(name, func)`（Misc）、`S:RegisterSkin(addon, func)`（Skins）、`TT:RegisterTooltips(addon, func)`（Tooltip）。
- 布局：`B.Mover`（可移动+持久化）、`B:CreateMF`（轻量跟随移动）、`B:SetBD` / `B:CreateBDFrame` / `B:CreateSD`（背板/阴影）、`B:CreateFS`（字体串）、`B:PixelIcon` / `B:AuraIcon`。
- 美化：`B:StripTextures`、`B:Reskin` / `B:ReskinClose` / `B:ReskinScroll` / `B:ReskinInput` / `B:ReskinDropDown`。
- 控件：`B:CreateButton / CreateCheckBox / CreateEditBox / CreateDropDown / CreateSlider / CreateColorSwatch / CreateGear`，`B:AddTooltip`。
- 工具：`B.HexRGB` / `B.ClassColor` / `B:RGBColorGradient`、`B:Round` / `B.Numb` / `B.FormatTime`、`B:GetNPCID` / `B.CopyTable` / `B.SplitList`、`B:SendChatMessage`（12.0 secret 安全包装）。
- 纹理/字体常量：`DB.normTex / gradTex / flatTex / bgTex / glowTex`，`DB.Font = {字体, 12, "OUTLINE"}`（数组式，取 `DB.Font[2]`）。资产路径集中在 `Core/Database.lua:69` 附近定义（`DB.starTex`、`DB.MicroTex`…）。

## 代码规范

- 方法用冒号定义在 B/模块表上（需要 self），纯函数用点（`B.HexRGB`）；函数 PascalCase，变量 camelCase。
- 每个"可开关"的功能入口先查开关再干活：`if not C.db["Misc"]["Xxx"] then return end` 或静态 `if not C.Infobar.Gold then return end`，早退。
- 受保护调用统一 `xpcall(func, geterrorhandler())`。
- 注释中英混排（数据表多为中文注释）；GUI 区块头用 `HeaderTag..L["..."]`（绿色），新功能徽标用 `IsNew..L["..."]`（"ISNEW" 魔法前缀）。
- 配方、完整机制、逐模块功能图见 references；动手前先读对应条目。

## 调试与测试

- `/rl` 重载；`/ndui` 打开设置/教程；`/mm` 布局调试器（配 `/ng` 网格）；`/nt` `/nf` `/ns` `/getid` `/getnpc` `/getenc` 为开发者命令（需在 `Core/DevTools.lua` 的 `DB.Devs` 名单内，`DB.isDeveloper`）。
- 改动后走一遍：重载 → 触发相关界面 → 看 `!BaudErrorFrame`（随仓库分发的报错框）→ 涉及 GUI 设置项时在 `/ndui` 里切换开关验证回调。
- Lua 语言服务器配置见 `.vscode/settings.json`（ketho wow-api 注解库，Lua 5.1）。
