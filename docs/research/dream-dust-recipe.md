# 梦尘配方、工序和品级判定

核验对象为《深空当铺：可能是偷的》Steam App 4348910、本机 Build `26525552`。下文按本机 `Probably Stolen_Data/Managed/Assembly-CSharp.dll` 的静态分析记录。程序集 SHA-256 为 `725CB999D019DFF1FF31BED353C7F9BB8D81A7605A15DE1F80F5A3A6CF7B1D05`。可用同目录 `DseCecil.dll` 的 `DseCecil.ModuleDefinition.ReadModule` 读取对应方法的 `Body.Instructions`。数值均使用游戏内部的整数 μL，注明百分比时除外。以下例子是按静态规则手算的校验数据。

## 投入和加工

| 工序 | 已核实规则 | 一手证据 |
| --- | --- | --- |
| 识别原料 | 真梦帽菇按大小或实际物品状态提供 `dream_cap_mush`。默认小、普通、大对应的体积参数为 20.5、32.5、75，糊量为 `27.333 × 参数 / 32.5` mL，游戏以单精度执行乘除后转成整数 μL。普通尺寸初始约 27.333 mL 糊，另有水、蛋白质、矿物、微生物和物理杂质。同属 `dream_cap` 的食用菇变体可只有植物残渣，故要读组成。 | `PreBuiltItemHelper.CreateSmallDreamCap`、`CreateDreamCap`、`CreateLargeDreamCap`、`ChemicalCompositionHelper.SetDreamCapComposition`、`ChemicalHelper.SetChemContentML` |
| 搅拌 | 需要有空间的输出容器，原料须满足搅拌标签。机器数 0、1、2、3、4 个升级模块时，损耗为 35%、20%、10%、5%、0%。每个原料的每种组分分别乘保留率并向最近整数取整，正好一半时远离零。容器容量不足还会触发超容量移除。 | `MachineBlender.<Blender>b__2`、`GetUpgradeCount`、`GetLoss`、`ChemicalHelper.BlendInto` |
| 配酒精、加热萃取 | 小瓶酒精预制品含 175,000 μL 酒精和 75,000 μL 水，酒精用量按有效体积计。萃取温度门槛为 40°C。每次反应处理 `min(M, max(2000, round(M × 0.1)))` 的当前糊量 `M`。若酒精为 `A`，转成萃取液的比例为 `0.5 × min(1, A / (5 × M))`，对本次转化量四舍六入、半数取偶。其余糊量转为植物残渣。反应逐次重算比例，萃取自身未扣酒精，因此 5:1 是**单次满产率**所需比例，实际整批最优投料仍须模拟各次反应。 | `PreBuildChemHelper.CreateMiniAlcoholBottle`、`ChemicalHelper.HandleDreamCapMushReaction` |
| 控温 | 40–50°C 是同时满足萃取且避开降解分支的区间，下限可取 40°C，上限不含 50°C。糊的降解分支从 50°C 起，萃取液的降解分支从 60°C 起，损失由温度和现存量决定。加热板煮沸酒精需超过 78.37°C，故应使用蒸发器去除溶剂。 | `ChemicalHelper.HandleTemperatureReaction`、`HandleDreamCapMushDegradation`、`HandleDreamCapExtractDegradation`、`HandleBoilingReaction`、`Liquid.InitLiquid` |
| 沉淀和过滤 | 每次蛋白沉淀先算有效试剂量 `abs(酸 − 碱)` 与剩余蛋白质的较小值，按 20% 截断，至少转换 1 μL 且最多转换到该较小值。等量耗去对应试剂并把蛋白质转成不溶碎屑。预制酸或碱每瓶 250,000 μL。标准或高级滤芯的粒径阈值为 22，可截留粒径 200 的植物残渣和碎屑，以及粒径 100 的物理杂质和粒径 30 的矿物。粒径 5 的萃取液和未沉淀蛋白通过。实际过滤量受每种组分的残余下限、容器中分出的体积以及滤芯剩余耐久约束。过量试剂的粒径也是 5。 | `ChemicalHelper.HandleProteinPrecipitation`、`PreBuildChemHelper.CreateAcidBottle`、`CreateBaseBottle`、`AmenitiesItemDirectory.WaterFilter`、`WaterFilterAdvanced`、`WaterHelper.HandleFilter`、`Liquid.InitLiquid` |
| 蒸发和回收 | 蒸发器需要电源。每次按机器性能调整 300,000 μL 的上限，按各挥发组分占比分配移除量。萃取液未标为挥发组分。回收量等于各组分实际移除量乘机器质量百分比后取最近整数，半数取偶，且须有蒸馏接收容器。溶剂回收模块的质量加成是 50 个百分点，并将蒸馏回收物限制为酒精。其他回收配置可能把水一起装进接收容器，后续仍应读组分。 | `MachineEvaporator.<Evaporator>b__7`、`ModuleDirectory.EvaporatorSolventRecovery`、`Liquid.InitLiquid` |
| 成型 | 每瓶从输入容器移出总计 2,000 μL，挥发性组分合计最多 100 μL 才通过原版干燥判定。配方指定 `dream_cap_extract` 为有效成分，但成型的门槛代码只校验总量和干燥条件。原版之后根据成分判级，因此仅靠成型机亮起不能证明达到目标。出料放进成型机的输出格。 | `ChemFinisherRecipe.InitRecipes`、`ChemFinisherRecipe.CanOutput`、`WaterHelper.IsWet`、`MachineChemFinisher.<ChemFinisher>b__4` |

梦尘批次的净消耗应跟踪原料、投入和回收后的酒精、沉淀用掉的试剂，以及能折成成本的滤芯耐久和供电消耗。成型后容器中的剩余有效干料和未用尽且可继续投入的溶剂可留待下一批。蒸发器可回收的液体要按实际收到的酒精和水分别记录。原版过滤函数扣滤芯耐久并返回通过后的体积，过滤掉的固体没有可回收的输出。搅拌和成型损失也没有独立的回收物。容器作为可复用设备记状态和容量，重复使用时不应重复计算购入成本。基础价值如何换算到一份溶液、滤芯耐久或电池电量，交由材料成本口径的后续决策。

## 装瓶、取整与品级

1. 从输入容器移出 2,000 μL 时，`WaterHelper.SplitProportionally` 先对每种组分的份额向下取整，再按余数从大到小逐个补 1 μL。余数相同优先原组分较多者，其后保留 `Liquid.Liquids` 顺序。`ChemicalHelper.TransferLiquidToChem` 调用这个路径。计算多瓶时须逐瓶更新剩余组分。
2. 成型机的 0、1、2、3、4 个升级模块对应 12%、6%、3%、1%、0% 杂质替换。先按同一比例分配法移除 `floor(2000 × 损失率)`，再加回等量 `physical_contaminant`。因此最终瓶量仍为 2,000 μL，但有效成分可能降低。证据见 `MachineChemFinisher.GetLoss`、`IntroduceImpurity`、`ChemicalHelper.RemoveVolume`。
3. `ChemicalProductHelper.GradeProduct` 按 `100 × 有效成分 / 成品总量` 的双精度值直接比较。等级计算使用实际纯度，小数不先取整，也不以显示文字或伪装标签作输入。下表中文为便于阅读的品级描述，界面实现以游戏本地化键为准。证据见 `GradeProduct`、`ItemConditionList.Create*Grade` 等方法。

| 品级 | 游戏品级标识 | 实际纯度区间 | 2,000 μL 成品最低有效成分 |
| --- | --- | --- | ---: |
| 淀粉级 | `starch_grade` | <30% | 0 μL |
| 垃圾级 | `junk_grade` | [30%, 45%) | 600 μL |
| 掺兑级 | `cut_product` | [45%, 60%) | 900 μL |
| 街头标准 | `street_standard` | [60%, 80%) | 1,200 μL |
| 接近药用级 | `near_pharma` | [80%, 90%) | 1,600 μL |
| 药用级 | `pharma_grade` | [90%, 99%) | 1,800 μL |
| 实验室纯品级 | `lab_pure` | ≥99% | 1,980 μL |

淀粉级的 0 μL 是原版判级和成型门槛的静态结果。是否允许自动方案把完全没有梦帽菇萃取液的成品计入目标，属于产品规则；如要求具有有效成分，应在规划时明确最低量。

## 可复算示例

**准备好的半成品。** 容器内含 6,000 μL 梦帽菇糊、30,000 μL 实际酒精，其他组分为零。假设一直保持 45°C 且有充足滤芯耐久、容器空间和电量。萃取三次，每次处理 2,000 μL 糊并生成 1,000 μL 萃取液和 1,000 μL 植物残渣。每次酒精与剩余糊之比均至少为 5:1，萃取结束有 3,000 μL 萃取液、3,000 μL 植物残渣、30,000 μL 酒精。过滤移除植物残渣，然后用仅有溶剂回收模块的蒸发器及有余量的接收容器，移除 30,000 μL 酒精并收回 15,000 μL。若用 3 模块成型机装一瓶，输入容器保留 1,000 μL 萃取液。输出 1,980 μL 萃取液与 20 μL 杂质，正好 99%，达到实验室纯品级。这里没有断言整菇可制得这个理想输入；整菇路线还需按其蛋白质、杂质和处理后的实测组分核验。

**边界。** 若 3 模块成型机的单瓶 2,000 μL 干料含 1,999 μL 萃取液和 1 μL 淀粉，20 μL 的比例移除份额中，萃取液配额为 19.99、淀粉为 0.01，余数使第 20 μL 仍从萃取液扣除。最终仅 1,979 μL 有效成分，实际纯度 98.95%，为药用级。优化器需要按成型机档位和原版取整预测每瓶，不能只比较入料纯度或按平均值验收。

## 后续实机核验

上述规则均来自静态分析。后续应在专用测试存档中，确认本机中文界面品级名称、整菇批次的实际反应触发与温度控制、材料容器的转移与余料、输出格不足时的行为，以及 3 模块成型机在 99% 边界的实机结果。还需根据具体设备核对滤芯耐久和电池的扣量、可用回收物及其基础价值，避免用名义容量冒充实际可用量。文档不承诺跨版本公式一致。

[开发者置顶的官方 Mod 文档入口](https://steamcommunity.com/app/4348910/discussions/0/573805072991083000/) · [官方 Mod 指南的机器窗口](https://github.com/dragonpan2/probably-stolen-modding/blob/main/ModdingGuide.md#machinery-windows)
