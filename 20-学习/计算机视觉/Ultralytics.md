---
title: Ultralytics 学习笔记
tags:
  - yolo
  - ultralytics
  - 目标检测
  - 计算机视觉
status: maintained
updated: 2026-11-15
---

# Ultralytics 学习笔记

> [!abstract] 一句话总结
> Ultralytics 是基于 **PyTorch** 的 YOLO 全流程开发库，把训练、验证、推理、导出、跟踪、基准测试全都封装成 `model.xxx()` 一行调用。它用两个维度组织能力：**Task（做什么任务）** × **模式（处在哪个阶段）**。

## 一、它是什么

- 一套基于 PyTorch 的 YOLO 全流程开发库，相当于把 PyTorch 的底层流程封装好、直接调用。
- 不用自己写训练循环、数据加载、后处理和指标计算。

### 1. 七种 Task

| Task | 中文 | 输出什么 |
| --- | --- | --- |
| `detect` | 目标检测 | 矩形框 + 类别 |
| `segment` | 实例分割 | 每个实例的像素掩码 |
| `OBB` | 旋转框检测 | 带角度的框（航拍、遥感常用） |
| `classify` | 图像分类 | 整张图的类别 |
| `pose` | 姿势检测 | 关键点 |
| `semantic` | 语义分割 | 逐像素类别（不区分实例） |
| `depth` | 深度估计 | 每个像素的深度值 |

### 2. 六种模式

| 模式 | 作用 | 用法 |
| --- | --- | --- |
| `train` | 训练 | `model.train(...)` |
| `val` | 验证 | `model.val(...)` |
| `predict` | 预测 | `model.predict(...)` |
| `export` | 导出 | `model.export(...)` |
| `track` | 多目标跟踪 | `model.track(...)` |
| `benchmark` | 基准测试 | `model.benchmark(...)` |

> [!tip] 不写模式时默认是推理
> 直接调用模型对象 `model(source)`，等价于 `model.predict(source)`。

## 二、下载与准备

1. 打开 GitHub 发布页：https://github.com/ultralytics/assets/releases
2. 下载 **Source code** 压缩包，以及你要用的模型文件（`.pt` 后缀的那些）。
3. 解压后用 VSCode 打开整个文件夹，把模型和待检测的图片/视频放进这个文件夹。
4. 用源码方式运行前先装依赖（在解压目录下执行）：`pip install -e .`；也可以直接 `pip install ultralytics` 用 pip 版。

## 三、推理模式（predict）

```python
from ultralytics import YOLO

model = YOLO("/var/train_run/Router_ALL/train7/weights/best.pt")

res = model.predict(
    source="/var/datasets/mixed_datasets/Router_ALL/images/_2026_09_18_16_27_06_877.jpg",
    imgsz=640,
    device=0,
    conf=0.5,
    save_txt=True,
    save_conf=True,
    save=True,
)
```

| 参数 | 含义 |
| --- | --- |
| `source` | 输入：单张图 / 图片目录 / 视频 / 摄像头编号 / URL |
| `imgsz` | 推理输入尺寸 |
| `device` | `0` = 0 号 GPU；`"cpu"` = 用 CPU |
| `conf` | 置信度阈值，低于它的框直接丢弃 |
| `save` | 保存可视化结果图 |
| `save_txt` | 保存检测结果 txt |
| `save_conf` | 在 txt 里带上置信度 |

> [!warning] 两个小坑
> - 结果默认落在 **`runs/detect/predict*/`** 目录（不是 `run`），重复推理会自动递增成 `predict2`、`predict3`…
> - 只开 `save_txt=True` 时 txt 里只有坐标和类别；要带置信度必须同时开 `save_conf=True`。

## 四、训练模式（train）

### 1. 先写 dataset.yaml

```yaml
path: /home/datasets/mixed_datasets/Router_ALL/dataset_Router_ALL
train: # train images
  - train/images
val: # val images
  - val/images
test: # test images (optional)

# Classes
names:
  0: product
```

| 字段 | 含义 | 要点 |
| --- | --- | --- |
| `path` | 数据集根目录 | 后面的 train / val / test 都相对它拼接 |
| `train` | 训练集 | 支持三种写法，见下 |
| `val` | 验证集 | 每个 epoch 后自动验证、算 mAP |
| `test` | 测试集 | **训练时不会自动跑**，需要 `split='test'` 手动评估 |
| `names` | 类别字典 `id: 名字` | 必须和 labels 里 txt 的类别 id 对齐 |

`train` 支持的三种写法：

1. **文件夹**：相对 `path` 的路径，如 `train/images`
2. **txt 文件**：如 `train.txt`，txt 里每行一个图片路径
3. **数组**：`[路径1, 路径2]`，用来合并多个文件夹一起训练

`val` 的格式和 `train` 完全一样。

> [!warning] 易错点
> - 原文 "`spilt = test`" 是笔误，正确参数名是 **`split`**，即 `model.val(split='test')`。
> - `names` 里的 id 顺序必须和 label txt 里写的类别 id 一致，否则不会报错，但模型学到的是错的类别。
> - `train` / `val` 里若写绝对路径，就不再相对 `path` 拼接。

### 2. 训练代码

```python
from ultralytics import YOLO

model = YOLO("/var/assets/yolo26n.pt")

model.train(
    data="/home/datasets/mixed_datasets/Router_ALL/new/dataset.yaml",
    epochs=1000,
    project="/var/train_run/Router_ALL",
    imgsz=640,
    batch=16,
    degrees=45,
    shear=0.02,
    perspective=0.001,
    flipud=0.5,
    fliplr=0.5,
    mosaic=0.75,
    mixup=0.1,
    copy_paste=0.1,
    patience=50,
)
```

### 3. 参数速查

| 参数 | 作用 |
| --- | --- |
| `data` | dataset.yaml 路径 |
| `epochs` | 总训练轮数 |
| `imgsz` | 输入尺寸 |
| `batch` | 批大小 |
| `project` | 输出根目录 |
| `name` | 输出子目录名；不写默认 `exp`，多次训练递增为 `exp2`、`exp3`… |
| `patience` | 早停：连续 N 轮指标无提升就停止 |
| `degrees` | 随机旋转，范围 ±N 度 |
| `shear` | 随机错切 |
| `perspective` | 随机透视变换 |
| `flipud` / `fliplr` | 上下 / 左右翻转的概率 |
| `mosaic` | 马赛克（四图拼接）增强概率 |
| `mixup` | 两图混叠增强概率 |
| `copy_paste` | 实例复制粘贴增强概率 |

> [!note] 训练产物在哪
> 在 `{project}/{name}/weights/` 下：`best.pt`（最优）和 `last.pt`（最后一轮）。
> 上面推理代码用的是 `.../Router_ALL/train7/weights/best.pt`，说明那次训练设过 `name="train7"`——原文的训练代码里少了这个参数，不写就会落到默认的 `exp` 目录。

## 五、命令速查

| 想做的事 | 代码 |
| --- | --- |
| 载入模型 | `model = YOLO("yolo26n.pt")` |
| 推理 | `model.predict(source="a.jpg", imgsz=640, device=0, conf=0.5)` |
| 训练 | `model.train(data="dataset.yaml", epochs=100, imgsz=640)` |
| 验证 | `model.val()` |
| 测试集评估 | `model.val(split="test")` |
| 导出 | `model.export(format="onnx")` |

## 六、复习清单

- [ ] 能默写 7 个 Task 和 6 种模式
- [ ] 能不看笔记写出 dataset.yaml 的 5 个字段
- [ ] 说得清 `train` 的三种写法分别用在什么情况
- [ ] 记得 `test` 集要手动 `split='test'` 才会评估
- [ ] 记得推理结果落在 `runs/detect/predict*`
- [ ] 记得用 `name` 参数控制训练输出子目录

## 七、自测（先自己想，再展开）

> [!question]- 1. Ultralytics 和 PyTorch 是什么关系？
> 它是一套基于 PyTorch 的 YOLO 全流程库：底层还是 PyTorch，但把训练循环、数据加载、增强、后处理、指标计算都封装成了 API。

> [!question]- 2. `detect`、`segment`、`semantic` 三者的区别？
> `detect` 只出矩形框；`segment` 是实例分割，每个实例有自己的掩码；`semantic` 是语义分割，逐像素分类但不区分实例。

> [!question]- 3. dataset.yaml 里 `path` 起什么作用？
> `path` 是数据集根目录，`train` / `val` / `test` 里写的相对路径都相对它拼接；如果那里写的是绝对路径，就不再拼接。

> [!question]- 4. 为什么 `test` 集在训练时没有被评估？
> Ultralytics 训练结束只跑 `val`；要用测试集得显式调用 `model.val(split='test')`。

> [!question]- 5. 想把这次训练固定输出到 `train7` 目录，加什么参数？
> `project="/var/train_run/Router_ALL"` 配合 `name="train7"`。

> [!question]- 6. `save_txt=True` 了，但 txt 里没有置信度，为什么？
> 少开了 `save_conf=True`。

> [!question]- 7. 训练设了 `epochs=1000`，为什么常常跑不满？
> 因为 `patience=50`：连续 50 轮指标没有提升就提前停止。
