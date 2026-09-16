# 扩展配方（How-to）

每条配方按步骤照做即可；代码片段均取自真实代码的简化形式。动手前先确认该功能归属哪个模块（见 modules.md）。

## 配方 1：新增一个 GUI 设置项

1. **登记默认值**（`Core/GUI.lua`）：角色级进 `G.DefaultSettings` 对应 section；账号级进 `G.AccountSettings`。漏掉这步会被 `InitialSettings` 清除。
   ```lua
   Misc = { ... , MyFeature = true, },
   ```
2. **加 locale**：`L["My Feature"]` 写进 `Locales/` 全部 6 个文件（enUS 必写；见配方 6）。
3. **加选项行**（`Core/GUI.lua` 的 `G.OptionList` 对应页签）：
   ```lua
   -- 即时生效型（标签带 *，带回调）：
   {1, "Misc", "MyFeature", L["My Feature"].."*", nil, nil, function(v) MyModule:UpdateMyFeature() end, L["MyFeatureTip"]},
   -- 需重载型（不带 *，无回调）：
   {1, "Misc", "MyFeature", L["My Feature"], nil},
   -- 账号级：
   {1, "ACCOUNT", "MyAccountKey", L["My Feature"].."*", ...},
   ```
   列语义：`{optType, section, option, label, horizon, data, callback, tooltip, disabled}`。
4. **消费它**：模块里 `if C.db["Misc"]["MyFeature"] then ... end`；即时生效型在回调里刷新状态，重载型在 OnLogin 里读取即可。
5. 需要用户"重置后恢复"无需额外工作；需要**转换旧用户数据**才走配方 10。

## 配方 2：新增 Infobar 区块

以 `Modules/Infobar/Gold.lua` 为模板：

1. 新建 `Modules/Infobar/MyInfo.lua`，挂进 `Modules/Infobar/Infobar.xml`：
   ```lua
   local _, ns = ...
   local B, C, L, DB = unpack(ns)
   if not C.Infobar.MyInfo then return end                 -- 硬开关（静态表）

   local module = B:GetModule("Infobar")
   local info = module:RegisterInfobar("MyInfo", C.Infobar.MyInfoPos)

   info.eventList = { "PLAYER_ENTERING_WORLD", "MY_EVENT", ... }
   info.onEvent = function(self, event, ...)
       self.text:SetText("...")
   end
   info.onEnter = function(self)
       GameTooltip:SetOwner(self, "ANCHOR_BOTTOMLEFT")      -- 锚点用 module:GetTooltipAnchor(info) 更稳
       GameTooltip:ClearLines()
       GameTooltip:AddLine(...)
       GameTooltip:Show()
   end
   info.onLeave = B.HideTooltip
   info.onMouseUp = function(self, btn) ... end            -- 可选；onUpdate 同理
   ```
2. **静态开关与锚点**（`Config/Modules.lua` 的 `C.Infobar` 表）：
   ```lua
   C.Infobar.MyInfo = true
   MyInfoPos = {"BOTTOM", UIParent, "BOTTOMRIGHT", -125, 6},
   ```
3. **默认布局串**（`Core/GUI.lua`，`"Misc"` section）：把 `[myinfo]` 追加进 `InfoStrLeft`/`InfoStrRight` 的默认值，例如 `InfoStrRight = "[spec][dura][gold][time]"` → `"[spec][dura][gold][myinfo][time]"`。区块名解析用 `strlower`，布局串里一律小写。
4. 若要 GUI 开关：`"Misc"` 页加一行 `{1, "Misc", "InfoMyInfo", L["My Info"].."*", true, nil, updateInfobarAnchor}`（布局改动回调统一是 `updateInfobarAnchor`）。

## 配方 3：新增皮肤

三种落点，选对注册表：

```lua
local _, ns = ...
local B, C, L, DB = unpack(ns)

-- A. Blizzard 加载单元（进 Modules/Skins/Blizzard/，挂 Blizzard.xml）
C.themes["Blizzard_SomePanel"] = function()
    local frame = SomePanelFrame
    B.StripTextures(frame)
    B.SetBD(frame)
    B.Reskin(frame.SomeButton)
    B.ReskinClose(frame.CloseButton)
    B.ReskinScroll(frame.ScrollBar)
    hooksecurefunc(frame, "Refresh", function() ... end)   -- 动态内容重刷样式
end

-- B. 核心 FrameXML 常驻（进 Modules/Skins/Blizzard/FrameXML/，挂 FrameXML.xml）
tinsert(C.defaultThemes, function()
    if not C.db["Skins"]["BlizzardSkins"] then return end
    ... -- StaticPopup 类：用 `if not x.styled then ... x.styled = true end` 防重复
end)

-- C. 第三方插件（进 Modules/Skins/AddOns/）
S:RegisterSkin("SomeAddon", function()
    if not IsAddOnLoaded("SomeAddon") then return end
    hooksecurefunc(SomeAddonFrame, "Update", function() ... end)
end)
```

- C 类在加载即完成注册；ADDON_LOADED 时由 `S:LoadAddOnSkins` 分发，懒加载的界面也能接到。
- 第三方 tooltip 皮肤走 Tooltip 模块的 `TT:RegisterTooltips("SomeAddon", fn)`，不要放 Skins。

## 配方 4：新增独立模块

1. 建目录 `Modules/MyThing/`：`MyThing.lua`（核心）+ 其余文件 + `MyThing.xml`：
   ```lua
   local _, ns = ...
   local B, C, L, DB = unpack(ns)

   local M = B:RegisterModule("MyThing")        -- 唯一注册名

   function M:OnLogin()                          -- 必须有，否则登录报 not loaded
       if not C.db["MyThing"]["Enable"] then return end
       ...
   end
   ```
   其余文件用 `local M = B:GetModule("MyThing")` 挂方法。
2. `MyThing.xml` 罗列文件（核心文件在前）；`Modules/Modules.xml` 追加 `<Include file="MyThing\MyThing.xml"/>`。
3. GUI 开关：`G.DefaultSettings` 加 `MyThing = { Enable = true, ... }`；`G.TabList` 与 `G.OptionList` 加页签和行；需要子面板走配方 11。
4. 位置类功能一律 `B:Mover(frame, L["MyThing"], "MyThing", {"BOTTOM", UIParent, "BOTTOM", 0, 200})`。

## 配方 5：新增 Misc 子功能（轻量）

```lua
-- Modules/Misc/MyFeature.lua（挂进 Misc/Misc.xml）
local _, ns = ...
local B, C, L, DB = unpack(ns)
local M = B:GetModule("Misc")

function M:MyFeature()
    if not C.db["Misc"]["MyFeature"] then return end
    B:RegisterEvent("SOME_EVENT", function() ... end)
end
M:RegisterMisc("MyFeature", M.MyFeature)
```

`M:OnLogin()` 会自动 xpcall 清单里所有项；GUI 开关按配方 1 加在 "Misc" 页。

## 配方 6：新增 locale 字符串

1. 代码里直接写 `L["English key"]`（key 即英文原文，可含 `|cff...|r`、`%s`、`|n`）。
2. 同步改 `Locales/` 全部 6 个文件，**加在各自相同相对位置**（项目惯例：文件行数保持对齐，当前每文件 861 行）：
   - `enUS.lua`：必须加（守卫被注释、总是加载，是兜底）——键值相同即可，如 `L["My Feature"] = "My Feature"`；含变量的给完整英文句。
   - `zhCN/zhTW/deDE/frFR/ruRU.lua`：翻译；来不及翻译就先复制英文值。各文件头部是 `if GetLocale() ~= "zhCN" then return end` 形式。
3. 未翻译的 key 显示为 key 本身（英文），不会报错。

## 配方 7：新增 UF 元素 / oUF 标签

- **标签**（`Modules/UFs/Tags.lua`）：
  ```lua
  oUF.Tags.Events["MyTag"] = "UNIT_HEALTH UNIT_MAX_HEALTH"
  oUF.Tags.Methods["MyTag"] = function(unit)
      local hp = UnitHealth(unit)
      if B:IsSecretValue(hp) then return "" end            -- 12.0 secret 防护
      return ...
  end
  ```
- **元素**（`Modules/UFs/Functions.lua`）：`function UF:CreateMyElement(self) ... end`；在 `Spawns.lua` 对应 style 函数（`CreatePlayerStyle` 等）里追加调用。注意元素尺寸参与 `SetUnitFrameSize` 布局的要在 `UF:Configure*` 系列里响应 `C.db["UFs"]` 选项变化。
- **团队角标指示法术**：数据在 `Config/CornerSpells.lua`（`C.CornerBuffs`）+ `NDuiADB["CornerSpells"]`（AurasTable 模块负责同步），UI 在 `UFs/Elements/SpellsIndicator.lua`。

## 配方 8：新增背包过滤器/容器

1. `Modules/Bags/Filters.lua` 写谓词：`local function isMyThing(item) ... end`（item 是 cargBags itemkey）。
2. `module:GetFilters()` 里组合：新命名容器加 `filters.bagMyThing = function(i) return ... end`；自定义组走 `C.db["Bags"]["CustomItems"][itemID] == index`。
3. `Core.lua` 的 `Backpack:OnInit()` 里 `AddNewContainer(...)` 挂上；标题映射在 `MyContainer:OnCreate` 的 header label 表里补条目（走 L 键）。

## 配方 9：新增 Plugin（不参与模块生命周期的独立脚本）

1. `Plugins/MyPlugin.lua`，头注释保留原作者信息；内部仍可用 `local B, C, L, DB = unpack(ns)`。
2. `Plugins/Plugins.xml` 加 `<Script file="MyPlugin.lua"/>`。
3. 需要开关时在 Skins 或 Misc 的设置页挂选项行，插件内读 `C.db`。

## 配方 10：设置迁移（老数据变换）

1. `G.DefaultSettings` 顶部加 `Reset7 = false`（沿用下一个未用的编号）。
2. `ADDON_LOADED` 处理器（GUI.lua）里：
   ```lua
   if not C.db["Reset7"] then
       -- 变换 C.db 里旧结构 → 新结构
       C.db["Reset7"] = true
   end
   ```
   只增删键不需要迁移（`InitialSettings` 自动补/清）。

## 配方 11：新增 ExtraGUI 子面板（复杂编辑器）

1. `Core/ExtraGUI.lua` 写 `G:SetupMyPanel(parent)`，复用 `G:CreateEditbox/CheckBox/Dropdown/Scroll/BarWidgets`，数据落 `NDuiADB["MyPanelSets"]` 或 `C.db`。
2. 主面板选项行第 6 列传齿轮回调：`{1, "Misc", "MyPanel", L["My Panel"].."*", true, G.SetupMyPanel}`——checkbox 的 `data` 为函数时自动渲染齿轮按钮。
