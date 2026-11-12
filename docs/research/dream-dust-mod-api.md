# 一键合成梦尘的库存、基础价值与原版执行接口

研究范围为 Windows Steam Build `26525552`。以下程序集结论来自只读 IL 检查，游戏与存档未启动或修改。公开文档与游戏程序集各自提供接口约定和当前版本实现。

## 证据来源

- 开发者在 Steam 置顶帖指向官方 Mod 指南。[开发者置顶帖](https://steamcommunity.com/app/4348910/discussions/0/573805072991083000/)
- 本机 Steam 清单 `steamapps/appmanifest_4348910.acf` 的 `buildid` 与 `TargetBuildID` 均为 `26525552`。下文记作「程序集」的证据来自 `Probably Stolen_Data/Managed/Assembly-CSharp.dll`，SHA-256 为 `725CB999D019DFF1FF31BED353C7F9BB8D81A7605A15DE1F80F5A3A6CF7B1D05`。可用同目录的 `DseCecil.dll` 和 `ModuleDefinition.ReadModule` 静态核对所列方法。此方法用于开发调查，游戏内 Mod 的接入路径见下文。

## 官方支持与版本边界

- 官方支持独立游戏内 UI、机器窗口与按钮，以及通过 `MachineHelper.GetBodyStorage(item, i)` 或 `MachineryPanelRow.GetPanel(role)` 读取机器面板。文档把改造现有游戏界面列为需要自行维护的路径。[机器窗口](https://github.com/dragonpan2/probably-stolen-modding/blob/main/ModdingGuide.md#machine-windows) · [支持范围](https://github.com/dragonpan2/probably-stolen-modding/blob/main/ModdingGuide.md#supported--unsupported-modding-features)
- 开发者文档允许 Harmony 补丁和受约束的游戏类型访问。程序集也有 `ModReflect`。以下店铺字段、机器构造和原版回调均属于版本相关的游戏实现。官方变更日志记载，12 月 10 日更新计划拒绝外部反射等能力，且 Mod 调用 `DseCecil` 会被归为文件能力。[反射说明](https://github.com/dragonpan2/probably-stolen-modding/blob/main/ModdingGuide.md#reflection) · [变更日志](https://github.com/dragonpan2/probably-stolen-modding/blob/main/APIChanges.md)

## 库存读取和可移动性

| 范围 | 当前版本入口及边界 |
| --- | --- |
| 化学设备 | 读取设备 `GameItem.contentWindow` 的面板及 `GameInventory.childItems`。原版搅拌机含 `input` 和 `container`，加热板、净化机和蒸发器均有 `container`，蒸发器另有 `distillate`，成型机有 `output`。电池、模块与输出格有不同职责。来源为程序集各 `Machine*.CreateMachineInventoryWindow`。 |
| 店内库存与柜台 | `EmporiumEntry.Instance.invElement` 是 `ModHelper.MoveItemToInventory` 的目标，`backInvinvElement` 是后柜台目标。前柜台 `frontInvinvElement` 独立。原版快捷移动从柜台取用时有所有权和场景判断。来源为程序集 `EmporiumEntry.GetInvItems/GetBackInvItems/GetCountersItems`、`ModHelper.MoveItemToInventory/MoveItemToBackCounter`、`ItemQuickTransferHandler.OnEventRelease`。 |
| 背包 | 实体背包是有子库存的 `GameItem`，例如 `ContainerItemDirectory.BackpackSmall` 创建库存窗口并调用 `ContainerHelper.InitBackpackItem`。外出模式还有独立 `afterhourPocketSlotInvBackpack`、`afterhourInventory` 和 `selectedBackpack`，店内主库存接口只返回主库存的直接物品。应在店内验证玩家选定的实体背包和可访问条件，外出场景需单独确认。来源为程序集 `EmporiumEntry.SetupAfterhourInv/TransferBackpackToInv/GetInvItems`。 |
| 储藏区 | 普通和化学储藏舱有自己的窗口和子库存。化学储藏舱用标签筛选。已扩容的容器使用 `storage` 角色下的 `MachineryTabs`，可以按 `count` 和 `GetTabInventory(i)` 读取当前及非当前页。来源为程序集 `ShipSystemDirectory.StorageBay/ChemistStorageBay`、`StorageExpansionHelper.GetTabs`、`MachineryTabs.GetTabInventory/CollectStorage`。 |

建议从明确授权的根遍历容器和分页，以对象引用去重。读取已有的 `uniqueId` 可辅助识别，调用 `GetUniqueID()` 可能分配新 ID。库存快照应包含父库存、`unitCount`、容器液体成分与体积、相关设备状态。液体成分可读取 `WaterHelper.GetLiquidParts` 并按 `Liquid.Liquids` 对应，或根据 `Liquid.GetCurrentPartFromId` 取对应只读标签。来源为程序集 `GameItem.GetUniqueID`、`WaterHelper.GetLiquidParts/GetTotalVolume` 和 `Liquid`。

原版快速移动先检查 `GameItem.MayRemove()`。该方法检查父库存锁、非自有容器和移出回调，`MaxNumRemove()` 还限制堆叠的可取数量。用户指定的物品锁应独立检查 `GeneralHelper.IsItemLocked(item)`。玩家拖拽可读 `ItemMouseDragHandler.current.IsDraggingItem` 及 `GetCurrentItem()`，运行中的设备可读 `MachineryHelper.IsProcessingActive(machine)` 并结合设备槽状态判断。当前证据覆盖这些分立信号，通用的「正在使用」判定仍需按机器和场景实测。来源为程序集同名方法及 `ItemQuickTransferHandler.OnEventRelease`。

移动可沿原版 `GraphUtils.TryAcceptOnce/TryAcceptAll` 和槽位规则进行，移动前先重读权限、数量和目的格。槽位查找可能穿入子容器，应明确目的库存并验证最终父库存。有效参数下 `TryAcceptAll` 返回未移入的数量，空参数早退也返回零。官方示例中的 `ModHelper.MoveItemToInventory` 等包装丢弃结果。确认成功应比较实际来源和目标库存的数量或位置变化。来源为程序集 `GraphUtils.TryAcceptAll`、`SlotMarker.TryAcceptOnce`、`GameGridInventory.TryFindOneValidInventorySlot` 与 `ModHelper.MoveItemToInventory`。

## 基础价值

- 固体原料有 `GameItem.unitValue` 和 `unitCount`，例如梦帽菇工厂写入 `unitValue=23`。`GameItem.GetBaseValue()` 计算 `unitValue`、复杂项价值、部分附加项和容器价的组合值，因此需要核对参与计价的项目是单件、整容器还是其内容物。`unitBaseValue` 字段也用于电源价值，不能凭字段名称认定它就是所有材料的统一底价。来源为程序集 `HydroponicDirectory.DreamCap`、`GameItem.GetBaseValue/GetItemMiscValue/GetComplexeItemValue` 和 `PowerHelper.GetPowerSourceItemValue`。
- 小瓶酒精的预制方法从 `mini_bottle` 取壳并灌入 175,000 μL 酒精和 75,000 μL 水。瓶壳价值有 `LIQUID_CONTAINER_BASE_PRICE_TAG`，`Liquid` 定义本身未发现散装成分的单位价。`rubbing_alcohol` 是不同物品，不能用其 `unitValue=35` 直接为工艺用散装酒精计价。来源为程序集 `PreBuildChemHelper.CreateMiniAlcoholBottle`、`MedsItemDirectory.RubbingAlcohol`、`WaterHelper.GetContainerPrice` 和 `Liquid` 字段。
- 决策票应明确可回收液体、散装试剂和半成品的折算来源，以及可重复使用的容器是否进入净成本。可用同一容器满、半、空的读数验证基础价值是否随溶液变化。当前证据能确认接口和无直接成分价的缺口，尚未得到可通用于所有候选配方的逐毫升价。

## 原版执行与确认

| 阶段 | 当前版本的原版路径 |
| --- | --- |
| 搅拌与投料 | `MachineBlender.<Blender>b__2` 对已装有未满容器的设备逐件处理 `BLENDABLE_TAG`，调用 `GetLoss` 和 `ChemicalHelper.BlendInto`。原版容器倒液的 `WaterHelper.TransferLiquid` 处理容量、按比例分配、过滤及温度传递。直接写成分标签会绕过这些动作。 |
| 加热萃取 | `MachineHeatingPlate` 每秒处理机器状态，`HandleHeating` 更新容器温度，容器的 `ChemicalHelper.HandleTemperatureReaction` 独立处理梦帽菇反应和降解。用原版刻度或控件设置目标并等待实际反应，确认温度、成分与进度。 |
| 净化与蒸发 | 净化机循环结束回调先检查电力，再执行 `PurifyContainer` 并按原版消耗电力。蒸发机循环回调检查容器和供电、减少挥发成分，可根据模块和蒸馏接收瓶状态回收。观察 `MachineryHelper.IsProcessingActive`、电力与实际成分变化，避免重复触发已在运行的周期。 |
| 成型 | 原版 `ChemFinisherUIManager.OnChemRecipeSelected` 给当前机器写选择标签。成型按钮 `MachineChemFinisher.<ChemFinisher>b__4` 先检查 `CanOutput(machine, containerSlot)`，按所选配方抽取材料、应用设备损耗、尝试放入输出格，随后调用 `ChemicalProductHelper.GradeProduct`。配方 ID 为 `dream_dust`。 |

来源为程序集同名方法和设备构造中注册的原版回调。当前版本 `MachineryButton.OnSelect` 调用该按钮回调，继承的 `SelectableElement.Select()` 只检查是否已选中。直接调用 `Select()` 可能绕过 UI 的禁用与隐藏判断，接入时须检查 `isDisabled`、`isHidden` 和原版前置条件。按钮控件的确切定位、配方切换、回调触发与时序需在实际装载的 Mod 上验证。来源为程序集 `SelectableElement.Select`、`MachineryButton.OnSelect` 及 `MachineChemFinisher.ChemFinisher`。

`CanOutput` 校验配方和原料容器，成型回调在抽走原料后才调用 `GraphUtils.TryAcceptAll`，且忽略返回值。先确认输出可放且重查状态。随后用输出库存中本次实际进入的 `dream_dust` 瓶数，以及 `ChemicalProductHelper.GetPurity` 和品级状态确认结果。按钮选择返回值或原料消耗量均不足以确认产出。来源为程序集 `MachineChemFinisher.CanOutput/<ChemFinisher>b__4` 和 `ChemicalProductHelper`。

## 后续规格验收需要的实机证据

1. 在店内用主库存、实体背包、储藏舱非当前页和各工作设备各放一份可区分材料。分别验证锁定物品、非自有容器、正在拖动、库存锁和运行设备。
2. 在独立测试存档中验证一条原版路径，记录投料前后体积、部件状态、电力、各步骤完成时机和蒸馏瓶回收体积。再验证输入变化、断电、输出格满及低于目标品级时的结果。
3. 校验正式目标版本的 Mod 扫描和实际加载。对无直接基础价的试剂使用经规格确认的折算口径，再验证同一试剂不同容器间的成本一致性。
