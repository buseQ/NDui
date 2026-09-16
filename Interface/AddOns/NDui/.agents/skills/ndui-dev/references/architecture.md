# NDui 架构详解

所有路径相对于 `NDui/` 插件根目录。本文件展开 SKILL.md 提到的机制，需要"为什么/具体在哪"时读这里。

## 1. 加载顺序

`NDui_Mainline.toc`（真实清单；`NDui.toc` 是 CurseForge 占位空文件）：

```toc
## Interface: 120100
## SavedVariables: NDuiADB, NDuiPDB
## SavedVariablesPerCharacter: NDuiDB
Init.lua
Locales\Locales.xml
Core\Core.xml
Config\Config.xml
Libs\Libs.xml
Modules\Modules.xml
Plugins\Plugins.xml
```

- 无 `## Dependencies`。
- 各 XML 只做 `<Script file=.../>` 罗列，**新文件加进对应 XML**。
- `Core\Core.xml` 顺序固定：Database → Functions → Mover → DevTools → GUI → ProfileGUI → ExtraGUI → FramePreview → Tutorial → Changelog（GUI 必须在 Modules 之前，因为默认值表在 GUI.lua 里）。
- `Modules\Modules.xml`：ActionBar → Bags → Auras → Chat → Tooltip → Skins → Maps → UFs → Misc → Infobar（这同时决定 OnLogin 顺序）。
- 客户端分支不用 TOC，用 Lua：`DB.Client = GetLocale()`、`DB.isNewPatch = select(4, GetBuildInfo()) >= 120100`（`Core/Database.lua`）。

## 2. 命名空间（Init.lua）

```lua
local addonName, ns = ...
ns[1] = {}  -- B, Basement：共享函数库
ns[2] = {}  -- C, Config：静态默认表 + 之后挂上 C.db / C.mult
ns[3] = {}  -- L, Locales
ns[4] = {}  -- DB, Database：运行期常量（不持久化）
NDuiDB, NDuiADB, NDuiPDB = {}, {}, {}
_G[addonName] = ns   -- 全局 NDui 即 ns
```

### 全局事件枢纽

单个隐藏 host 帧按事件名分发到注册表；处理函数以 `xpcall(func, geterrorhandler())` 执行：

```lua
B:RegisterEvent("PLAYER_TALENT_UPDATE", CheckRole)   -- 支持匿名函数，多函数/事件
B:RegisterEvent("UNIT_HEALTH", func, "player")       -- 第3参起走 RegisterUnitEvent
B:UnregisterEvent(event, func)                       -- 最后一个函数被移除时自动反注册
```

`"CLEU"` 是 `COMBAT_LOG_EVENT_UNFILTERED` 的别名（遗留兼容）。Midnight 12.0 战斗日志已对插件关闭，新代码不要基于 CLEU 做功能。

### UI 缩放

`B:SetupUIScale(init)` 维护 `C.mult = 768/DB.ScreenHeight * NDuiADB["UIScale"]` 相关倍率；`UI_SCALE_CHANGED` 时重算。所有像素级偏移乘 `C.mult`。`C.margin = 3` 也是在此初始化的布局常量。

## 3. 模块系统

```lua
-- 模块核心文件（每模块一次）
local M = B:RegisterModule("Misc")     -- 注册名唯一，返回模块表
function M:OnLogin() ... end           -- PLAYER_LOGIN 时执行

-- 同模块的其他文件
local M = B:GetModule("Misc")          -- 拿同一张表，往上面挂方法
```

- 登录时按**注册顺序**遍历 `initQueue` 执行 `OnLogin`；因此 `Core` 里的模块（Mover、GUI、Settings）先于 `Modules` 里的跑。
- 已注册名：Actionbar, Cooldown, Auras, Bags, Infobar, Maps, Misc, Skins, Tooltip, UnitFrames, Chat（Modules/）+ Settings, Mover, GUI, AurasTable（Core/Config/）。
- 子功能清单模式（一个模块多个子文件各自贡献初始化）：
  - Misc：`M:RegisterMisc(name, func)` 存入 `MISC_LIST`，`M:OnLogin` 里逐个 xpcall。
  - Skins：三个注册表 `C.defaultThemes`（核心 FrameXML，无视开关）、`C.themes`（按 Blizzard 加载单元 ADDON_NAME 键控）、`C.otherSkins`（第三方，`S:RegisterSkin(addonName, func)`）。
  - Tooltip：`TT:RegisterTooltips(addon, func)`，ADDON_LOADED 时执行并清除。

## 4. 数据库分层

| 表 | TOC 声明 | 语义 | 默认值来源 |
|---|---|---|---|
| `NDuiDB` | SavedVariablesPerCharacter | 角色配置 | `G.DefaultSettings` |
| `NDuiADB` | SavedVariables | 账号配置+数据 | `G.AccountSettings` |
| `NDuiPDB[1..5]` | SavedVariables | 档案槽 | 同 DefaultSettings |

- `C.db` 是**句柄**：`NDuiADB["ProfileIndex"][DB.MyFullName] == 1` 时指向 `NDuiDB`，否则指向 `NDuiPDB[n]`。档案切换后模块读 `C.db` 无感知。
- 合并器 `InitialSettings(source, target, fullClean)`（GUI.lua）：补缺失默认值 + **删除 defaults 里已不存在的过期键** —— 这就是铁律 3 的原因。
- 单发迁移：`G.DefaultSettings` 里的 `Reset4/5/6/...` 标志位，`ADDON_LOADED` 里 `if not C.db["ResetN"] then <变换旧值> C.db["ResetN"] = true end`。
- `ADDON_LOADED("NDui")` 还负责：解析状态条材质（`NDuiADB["CustomTex"]` 或 `G.TextureList[NDuiADB["TexStyle"]]` → `DB.normTex`）、初始化 UI 缩放。

## 5. GUI 设置系统（Core/GUI.lua）

- 模块 `G = B:RegisterModule("GUI")`。面板不注册 Blizzard Settings 分类，而是 `hooksecurefunc(GameMenuFrame, "InitButtons", ...)` 注入按钮打开；战斗中隐藏。
- `G.TabList`：15 个页签（动作条/背包/头像/团队框架/姓名板/玩家板/Auras/团队工具/聊天/地图/美化/鼠标提示/杂项/UI设置/档案）。前缀 `IsNew = "ISNEW"` 会渲染"新"徽标；`HeaderTag = "|cff00cc4c"` 绿色区块头。
- `G.OptionList[页签号] = {行, 行, ...}`，每行：

```lua
{optType, section, option, label, horizon, data, callback, tooltip, disabled}
-- 例（复选框+齿轮子面板）：
{1, "Actionbar", "Enable", HeaderTag..L["Enable Actionbar"], nil, setupActionBar}
-- 例（下拉）：
{4, "Actionbar", "CDFormat", L["Show Cooldown"].."*", nil, {L["ColorTenth"], ...}, updateCooldown}
-- 例（滑条 {min,max,step}）：
{3, "Actionbar", "CDFontSize", L["GeneralCDFontSize"].."*", true, {5, 30, 1}, updateCDText, L["GeneralCDFontSizeTip"]}
-- 例（空行 {} = 分隔线）
```

| 列 | 含义 |
|---|---|
| optType | 1 复选框、2 编辑框、3 滑条、4 下拉（保存索引）、5 颜色板、`{}` 空行 |
| section | `C.db` / `G.DefaultSettings` 的 section 名；**`"ACCOUNT"` 表示读写 `NDuiADB`** |
| option | section 内的键名 |
| label | 显示文本；尾缀 `*` = 即时生效，否则提示 ReloadUI |
| horizon | true = 同行右侧排布 |
| data | 类型相关：复选框=齿轮按钮 OnClick（打开 ExtraGUI 子面板）；滑条={min,max,step}；下拉=选项表 |
| callback | `function(newValue)`，改动时调用 |
| tooltip | 悬停提示 |
| disabled | true = 禁用置灰 |

- 读写统一走 `CheckUIOption(key, value, newValue)`：`"ACCOUNT"` → `NDuiADB[value]`，否则 `C.db[key][value]`。
- 每个控件挂 `__key/__value/__name/__callback/__default`；`G.needUIReload` 置位后 OKAY 弹 `StaticPopupDialogs["RELOAD_NDUI"]`。

## 6. 附加 GUI 子面板（Core/ExtraGUI.lua）

主面板齿轮按钮打开的复杂编辑器：`G:SetupClickCast`（点击施法 → `NDuiADB["ClickSets"][DB.MyClass]`）、`G:SetupSpellsIndicator`（角标指示法术 → `NDuiADB["CornerSpells"]`）、`G:SetupUnitFrame / SetupRaidFrame`（UF 精细布局）、`G:SetupBagFilter`（背包自定义分组）等。弹出窗通用模式 `createExtraGUI(parent, name, title, bgFrame)`，通用控件构造器 `G:CreateEditbox/CheckBox/Dropdown/Scroll/BarWidgets`。

## 7. 位置系统（Core/Mover.lua）

- `B:Mover(frame, text, value, anchor, width, height)`：创建拖拽框，位置持久化到 `C.db["Mover"][value]`，并把 frame 锚上去。**value 全局唯一**（"Bar1"、"PlayerUF"、"GameTooltip"…）。
- `B:CreateMF(parent, saved)`：轻量跟随移动（持久化到 `C.db["TempAnchor"][frameName]`）；`B:BlizzFrameMover(frame)` 移动暴雪面板。
- `/mm`（SlashCmdList["NDUI_MOVER"]）打开调试控制台（x/y 微调、箭头），`M:UnlockElements()` 解锁；网格为 SlashCmdList["TOGGLEGRID"]（`/ng`）。战斗中自动锁定（PLAYER_REGEN_DISABLED/ENABLED）。
- Mover 模块还 stub 了 `EditModeManagerFrame` 的若干刷新函数（`M:DisableBlizzardMover()`）。
