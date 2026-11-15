---
title: Ultralytics
tags:
  - yolo
  - ultralytics
  - 计算机视觉
status: maintained
updated: 2026-11-15
---

# Ultralytics

介绍:

ultralytics是一套基于PyTorch的YOLO全流程开发库

相当于帮你封装了PyTorch,直接使用

Task:

1. detect 目标检测
2. segment 实例分割
3. OBB 旋转框检测
4. classify 图像分类
5. pose 姿势检测
6. semantic 语义分割
7. depth 深度估计

模式:

1. train: 训练
2. val: 验证
3. predict: 预测
4. export: 导出
5. track: 多目标跟踪
6. benchmark: 基准测试

不写模式默认推理模式

## 下载

GitHub: https://github.com/ultralytics/assets/releases

下载 Source code 的压缩包

还有你想使用的模型(.pt后缀的那些)

## 使用

### 1.推理模式

解压后使用VScode打开整个文件夹

将你的模型和你要检测的图片或视频等源放到文件夹中

```python
from ultralytics import YOLO
model = YOLO("/var/train_run/Router_ALL/train7/weights/best.pt")
res = model.predict(source="/var/datasets/mixed_datasets/Router_ALL/images/_2026_09_18_16_27_06_877.jpg",
                    imgsz=640, device=0, conf=0.5, save_txt=True, save_conf=True, save=True)
```

运行即可用官方模型检测图片或者视频

运行结果在runs文件夹下

### 2.训练模式

需要先编写dataset.yaml文件

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

path：数据集根目录，后面的train/val/test都是相对于path去拼接的

train：训练集，支持三种格式

1. 文件夹：相对path的路径
2. txt文件：train.txt，txt里每一行的图片路径
3. 数组：[文件路径1, 文件路径2] 用来合并多个文件夹训练

val：验证集，每轮epoch后自动验证，算mAP，格式和训练集一样

test：测试集，训练过程不会自动跑测试集，需要加命令 split = test 才加载评估

names:字典，类别id:类别名字；需要和labels中txt文档中的顺序一致

训练代码如下

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
    patience=50
)
```
