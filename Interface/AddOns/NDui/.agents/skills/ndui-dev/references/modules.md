# NDui 功能模块图鉴与扩展点

每模块：文件构成 → 已实现功能 → 扩展点（往哪儿加东西）。加载顺序即 `Modules\Modules.xml`：ActionBar → Bags → Auras → Chat → Tooltip → Skins → Maps → UFs → Misc → Infobar。

## ActionBar（`Modules/ActionBar/`）

文件：`Bars/Bars.lua`（核心，LibActionButton 构 8 条主条）、`Bars/Stancebar|Petbar|Extrabar|Leave_vehicle|Hide_blizzart|MicroMenu|StylePreset.lua`、`ButtonStyle.lua`（快捷键缩写/宏文本）、`Cooldown.lua`（独立模块 "Cooldown"：接管冷却数字，`Cooldown:IgnoreCooldown(cd)` 供其他模块豁免）、`Keybind.lua`（点击绑定模式）。
功能：8 主条 + 态度条/宠物条/额外条/下车按钮；页码状态驱动（`RegisterStateDriver(frame, "page", ...)`）、可见性驱动、微菜单替换、隐藏暴雪条（`DisableAllScripts`）、条布局预设导入导出。
扩展点：新按钮样式进 `ButtonStyle.lua`；新条/新可见性规则在 `Bars/Bars.lua` 的 `BAR_DATA` + `Bar:UpdateVisibility()`；按键缩写映射在 `ButtonStyle.lua` 的 `replaces` 表。

## Bags（`Modules/Bags/Core.lua` + `Filters.lua`，cargBags）

`cargBags:NewImplementation("NDui_Backpack")`；容器分 背包/银行/账号银行 三大区，内置 ~30 个命名容器（消耗品、装备、垃圾、任务、自定义组 bagCustom1..5…）。按钮类重绘 iLvl/新建图标/收藏星标/任务标记；模式按钮（拆分/收藏/垃圾/Ctrl+Alt 删除）；`SpawnPlugin` 挂搜索条/钱袋/背包条；排序模式 1/2/3（暴雪/暴雪+倒序/关闭）。
扩展点：**新过滤器** = `Filters.lua` 加谓词函数 + `module:GetFilters()` 组合；**新容器** = `Backpack:OnInit()` 里 `AddNewContainer`；过滤关键词进 `NDuiADB` 自定义组（GUI 齿轮 → `G:SetupBagFilter`）。

## Auras（`Modules/Auras/`）

`Auras.lua` 玩家 buff/debuff 条（隐藏暴雪 BuffFrame/DebuffFrame，`CreateAuraHeader` + `RegisterStateDriver`）；`Reminder.lua` buff 提醒（数据源 `DB.ReminderBuffs[DB.MyClass]`，条件：天赋/专精/战斗/装备/宝石）；`Totems.lua` 图腾条。
扩展点：新提醒规则加进 `Core/Database.lua` 的 `DB.ReminderBuffs`；时长格式化用 `C_StringUtil.CreateNumericRuleFormatter()`。

## Chat（`Modules/Chat/`）

`Core.lua` 聊天框皮肤/锁定/底框编辑框/Tab 频道循环/快速滚动/密语自动邀请/密语置顶与音效；`ChannelRename.lua` 时间戳+频道改名；`Chatcopy.lua` 复制聊天；`Chatbar.lua` 频道切换条（NDui_ChatBar）；`Filter.lua` 垃圾信息过滤（关键词/相似度/白名单，列表存 `NDuiADB["ChatFilterList"/"ChatFilterWhiteList"]`）；`Hyperlink.lua` 链接增强（Alt 邀请/Ctrl 工会邀请）。
扩展点：新聊天增强做成 `module:Feature()` 并在 `OnLogin` 的元素清单里追加。

## Tooltip（`Modules/Tooltip/`）

`Tip.lua` 单位提示重写（战斗隐藏、角色图标、名字着色、目标的目标、NPC ID、锚点 4 模式 + Mover）；`TooltipIcons.lua` 标题内嵌图标；`TooltipID.lua` 各类 ID 行（SpellID/ItemID/…）与背包计数；`ItemLevel.lua` 装等引擎（延迟节流+缓存）；`PetInfo.lua` 宠物提示；`Hovertips.lua` 聊天链接悬浮；`AzeriteArmor.lua` 已知天赋标记。
扩展点：单位提示追加信息写在 `TT:OnTooltipSetUnit()` 尾部的调用链（如 `TT.ShowUnitMaxGold` 统计 334 历史最高金币）；皮肤注册走 `TT:RegisterTooltips(addon, func)`；事件挂接用 `TooltipDataProcessor.AddTooltipPostCall(Enum.TooltipDataType.Unit, fn)`。

## Skins（`Modules/Skins/`）

引擎在 `Skins.lua`（三个注册表 + ADDON_LOADED 按需执行）。皮肤按目标分三处：
- `Blizzard/Blizzard_*.lua`（65 个）：`C.themes["Blizzard_XXX"] = function() ... end`
- `Blizzard/FrameXML/*.lua`（50+）：`tinsert(C.defaultThemes, function() ... end)`
- `AddOns/*.lua`：第三方（BigWigs/DBM/Details/WeakAuras…），`S:RegisterSkin("AddonName", func)`
扩展点见 recipes.md"新增皮肤"。辅助 API：`S:CreateToggle(frame)` 大面板侧边开合按钮。
- 滑条美化二选一：`B:ReskinSlider` 仅适用裸滑条（OptionsSliderTemplate）；`MinimalSliderWithSteppers` 容器须用 `B:ReskinStepperSlider`——`ReskinSlider` 入口已做形状检测自动转发，并对无拇指滑条安全跳过（Baganator 曾因传容器报 nil thumb 错误）。

## Maps（`Modules/Maps/`）

`WorldMap.lua` 坐标/比例/地图 ID/去战争迷雾；`Minimap.lua` 小地图全家桶（脉冲边框、区域按钮重排、时钟、日历、音量滚轮、右键菜单、谁 ping 了地图）；`RawMapData.lua` 迷雾覆盖数据表。
扩展点：小地图新按钮进 `Minimap.lua` 的 `ReskinRegions`；新地图数据处理加在 `RawMapData.lua`。

## UFs（`Modules/UFs/`，oUF）

- `Tags.lua`：`oUF.Tags.Methods/Events` 自定义标签（VariousHP/VariousMP/color/DDG…），类色用 `NDui_ScanTooltip` 扫描。
- `Functions.lua`：元素构造库 `UF:CreateHeader/HealthBar/PowerBar/Portrait/CastBar/Prediction/ClassPower/Auras/ClickSets/ThreatBorder...`；`CreateAuraElement + AddAuraGroup` 包装 oUF aura 分组。
- `Spawns.lua`：`oUF:RegisterStyle` → `oUF:Spawn`，玩家/目标/ToT/焦点/宠物/Boss1-10/竞技场1-5；团队框架 `oUF:SpawnHeader`（按职责排序、可见性字符串动态拼接）；姓名板 `oUF:SpawnNamePlates("oUF_NPs")`。
- `Raid.lua` 团队元素/角标法术；`Nameplates.lua` 姓名板样式+CVars；`Elements/` Castbar（时长绑定）、SpellsIndicator（团队角标指示）。
扩展点：**新元素** = `Functions.lua` 加 `UF:CreateXxx(self)`，在对应 style 函数里调用；**新单位样式** = `Spawns.lua` 注册并 Spawn + Mover；**新标签** = `Tags.lua` 加 Methods/Events 对。oUF 位于 `Libs/oUF`（9.5.7，NDui 补丁点标 `-- NDui mod`），经 `ns.oUF` 取用。

## Misc（`Modules/Misc/`，20+ 子文件的杂烩）

`Misc.lua` 注册 `M:RegisterMisc(name, func)` 清单机制并内联：截图、快速拾取、交易目标信息、屏蔽陌生人邀请、Boss 横幅开关、电影快进、右键菜单加项、增强颜色选择器、快速删除确认等。子文件：`BlizzFix`（暴雪 bug 规避）、`AlertFrames`（警报重排）、`AttachedMeters`（停靠 Details/Skada）、`EnhancedCDM`（暴雪冷却管理器重排）、`ExpRep`（经验/声望条）、`Focuser`（Shift 点击聚焦）、`IgnoreNote`（黑名单备注，存 `NDuiADB["IgnoreNotes"]`）、`ItemLevel`（提示装集）、`Mail`（一键收件）、`MDGuildBest`（M+ 工会成绩）、`MissingStats`、`Notifications`（打断/CD 警报 + 插件通讯）、`PetFilterTab`、`QuestNotification`、`QuestTool`、`QuickJoin`、`RaidTool`（悬浮团队工具条）、`TradeTabs`（专业侧边签）、`SingingSockets`（宝石插座）。
扩展点：新小功能 = 子文件里 `M:RegisterMisc("Name", M.Feature)` + `function M:Feature()`；需要独立开关则在 GUI "Misc" 页加选项行（见 recipes）。

## Infobar（`Modules/Infobar/`）

底部长条信息区。`Core.lua` 提供 `INFO:RegisterInfobar(name, point)` / `INFO:LoadInfobar(info)`；左右背景条 `NDuiLeftInfobar`/`NDuiRightInfobar`（450×18，`B.Mover`）。布局串在 `C.db["Misc"]["InfoStrLeft"/"InfoStrRight"]`，如 `"[gold][dura] [fps][ping]"`，`gmatch("%[(%w+)%]")` 解析后从中心向两侧链式锚定。
现有区块（注册名 → 文件）：`Guild`、`Friend`、`Ping`、`Fps`、`Zone`、`Time`、`Gold`（今日收益/分角色金币存 `NDuiADB["totalGold"]`/自动卖灰）、`Dura`（耐久+修理）、`Spec`（天赋/拾取专精切换）。
扩展点见 recipes.md"新增 Infobar 区块"。硬开关与默认锚点在 `Config/Modules.lua` 的 `C.Infobar` 表。

## Core/ 辅件一览

`Database.lua`（DB 常量/材质/字体/BuffList）、`Functions.lua`（B API ~100 项）、`Mover.lua`、`DevTools.lua`（开发者命令）、`GUI.lua`（设置中枢）、`ProfileGUI.lua`（档案导出导入，LibBase64）、`ExtraGUI.lua`（子面板）、`FramePreview.lua`（框体预览）、`Tutorial.lua`（首启教程 + `/ndui`）、`Changelog.lua`（zhCN 专用更新弹窗）。

## Plugins/（`Plugins/`，非模块）

自带第三方小脚本，头注释保留原作者 + `-- NDui MOD`：TaintLess、AlreadyKnown、DragEmAll、ExtraQuestButton、QuickQuest、yClassColors。不参与模块生命周期。扩展点见 recipes.md"新增 Plugin"。
