# Smoke Test 输出结果查看说明

本目录保存两条路线在同一批 40 张小测试集图片上的推理结果：

- `trans4pass_synpass/`：Trans4PASS + SynPASS checkpoint，全景专用模型路线。
- `segformer_cityscapes/`：SegFormer B5 + Cityscapes checkpoint，全景切片 baseline 路线。

小测试集原图位置：

```text
images/
```

## 推荐查看顺序

优先打开两张总览图：

```text
outputs/smoke_test/segformer_cityscapes_report.png
outputs/smoke_test/trans4pass_synpass_report.png
```

如果要逐张细看，再进入对应模型目录的 `overlay/` 和 `risk_map/`：

```text
outputs/smoke_test/segformer_cityscapes/overlay/
outputs/smoke_test/segformer_cityscapes/risk_map/

outputs/smoke_test/trans4pass_synpass/overlay/
outputs/smoke_test/trans4pass_synpass/risk_map/
```

## 文件夹含义

每个模型目录下面都有相同的输出结构：

```text
raw_mask/   原始类别编号 mask
color/      彩色语义分割图
overlay/    原图 + 彩色语义分割叠加图
task_map/   盲杖任务类别图
risk_map/   可通行 / 风险区域图
summary.csv 每张图的路径和左/中/右区域统计
```

### raw_mask

这是单通道类别编号图，每个像素值代表模型预测的类别 ID。

它主要给程序使用，不适合直接肉眼判断。看起来可能是黑色或灰度块，这不代表模型没有输出。

### color

这是把语义类别 ID 映射成颜色后的结果。

适合看模型大致把图像分成了哪些语义区域，例如道路、天空、建筑、车辆、行人等。但不同模型的颜色体系不完全一样，因此不要只凭颜色判断类别，要结合 overlay 和任务图。

### overlay

这是最适合判断“模型分割效果”的图。

它把彩色分割结果半透明叠加在原图上，可以直接看：

- 道路、人行道是否覆盖在真实地面区域上；
- 天空、建筑、树木是否大致分对；
- 车辆、行人、杆、墙等风险物体是否被识别出来；
- 分割边界是否大体贴合原图。

### task_map

这是把原始语义类别进一步合并成盲杖任务类别后的图。

颜色含义：

```text
绿色：passable，可通行区域，例如 road、sidewalk、ground、terrain、parking
红色：risk，风险区域，例如 person、car、bus、bicycle、pole、fence、wall、traffic sign
灰色：background，背景或暂不参与盲杖判断的区域，例如 sky、building、vegetation、unknown
```

### risk_map

这是最适合判断“盲杖提示是否合理”的图。

它和 `task_map` 类似，但背景更暗，突出绿色可通行区域和红色风险区域：

```text
绿色越连续：越可能表示该方向可通行
红色越集中：越可能表示该方向有障碍或风险
深灰色：背景或当前规则未纳入判断的区域
```

### summary.csv

每个模型目录下都有一个 `summary.csv`，记录每张图对应的文件路径和方向统计分数。

其中 `scores` 包含三个方向：

```text
left
center
right
```

每个方向会统计：

```text
passable_ratio：该方向中可通行像素比例
risk_ratio：该方向中风险像素比例
```

简单判断逻辑：

```text
center 的 passable_ratio 高，risk_ratio 低：前方较可能可通行
center 风险较高，但 left 或 right 更安全：建议向左或向右
三个方向风险都高：提示谨慎、减速或停止
```

## 文件名对应关系

同一张图使用相同的 `sample_id`。

例如 `cvrg_pano_001`：

```text
原图：
images/cvrg_pano_001.png

SegFormer overlay：
outputs/smoke_test/segformer_cityscapes/overlay/cvrg_pano_001_overlay.png

SegFormer risk_map：
outputs/smoke_test/segformer_cityscapes/risk_map/cvrg_pano_001_risk_map.png

Trans4PASS overlay：
outputs/smoke_test/trans4pass_synpass/overlay/cvrg_pano_001_overlay.png

Trans4PASS risk_map：
outputs/smoke_test/trans4pass_synpass/risk_map/cvrg_pano_001_risk_map.png
```

## 怎么判断结果是否良好

本阶段是 smoke test，主要做流程验证和主观可视化判断，还不是正式 mIoU 评估。建议按下面几个标准看。

### 1. 语义是否基本正确

看 `overlay/`：

- 道路和人行道应主要落在真实地面区域；
- 天空应主要在图像上方；
- 建筑、树木、墙面等大区域不能明显乱分；
- 车辆、行人、杆、交通标志等风险目标如果明显存在，最好能被分出来。

### 2. 可通行区域是否连续

看 `risk_map/`：

- 真实道路、人行道区域应呈现较连续的绿色；
- 如果绿色区域被切得很碎，说明后续盲杖方向判断会不稳定；
- 如果大量天空、建筑被误判成绿色，说明可通行判断不可信。

### 3. 风险区域是否合理

看 `risk_map/`：

- 车辆、行人、杆、墙、护栏等位置应尽量出现红色；
- 如果前方有明显障碍但完全没有红色，需要记录为失败案例；
- 如果大面积安全地面被误标成红色，可能导致系统过度保守。

### 4. 方向提示是否符合直觉

结合 `risk_map/` 和 `summary.csv`：

- 中间区域绿色多、红色少，通常可以输出“前方可通行”；
- 中间红色明显，左侧绿色更多，则可以输出“前方存在障碍，建议向左”；
- 左右都不好，但中间还有连续绿色，则可以继续前行但提示谨慎；
- 三个方向都缺少绿色或红色很多，则提示“前方环境复杂，请减速或停止”。

### 5. 两条路线如何对比

建议同一张图同时看两个模型的 `overlay/` 和 `risk_map/`：

- `Trans4PASS + SynPASS`：重点说明它是全景专用模型，适合作为项目主线和研究点。
- `SegFormer + Cityscapes`：重点观察 road、sidewalk、car、person 是否更清晰，适合作为展示效果更直观的 baseline。

如果某张图中 SegFormer 的道路、人行道更稳定，可以在展示中说明：普通街景数据训练的模型在户外道路场景上有更强类别先验；但它不是专门为全景图设计，所以边缘、畸变和拼接区域仍可能出错。

## 当前结果的合理预期

这批结果只用于回答三个问题：

```text
1. 两条路线是否都能批量推理成功？
2. 是否能输出语义 mask、overlay、task_map、risk_map？
3. 是否能初步转化成左/中/右方向的可通行与风险判断？
```

如果要做正式实验，还需要进一步：

```text
1. 人工筛选更适合盲杖场景的户外全景图；
2. 统一记录每张图的人工判断结果；
3. 统计提示是否合理，而不是只看单张图是否好看；
4. 对失败案例分类，例如道路误分、风险漏检、全景畸变导致错误等。
```
