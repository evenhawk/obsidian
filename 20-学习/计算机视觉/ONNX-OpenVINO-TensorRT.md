---
title: ONNX/OpenVINO/TensorRT
tags:
  - onnx
  - openvino
  - tensorrt
  - 模型部署
  - 计算机视觉
status: maintained
updated: 2026-11-15
---

# ONNX/OpenVINO/TensorRT

## 概念

使用不同的训练框架得到各自框架的模型

然后统一转换为 **ONNX** 格式

把这个中间格式放在硬件本地安装的推理框架上去运行

![[ONNX-部署架构图.png]]

**训练框架：**

1. **PyTorch**
2. **TensorFlow**
3. **keras**

不同的训练框架兼容的硬件设备不同

ONNX格式在这里起到了桥梁的作用

**TensorRT** 硬件为英伟达 GPU

**OpenVINO** 硬件为英特尔 CPU

## ONNX介绍

*详情见：YOLO26 模型的 ONNX 导出 | Ultralytics*

ONNX(*open neural network exchange*)

ONNX，即**开放神经网络交换**

是一种开放文件格式，用于表示机器学习模型，使这些模型能够在不同的 AI 框架和硬件上使用。

1. 开源机器学习的通用中间格式
2. 兼容各种深度学习框架
3. 兼容各种推理引擎
4. 兼容各种终端硬件
5. 兼容各种操作系统

ONNX模型可用于在不同框架之间**无缝迁移**

例如，在 PyTorch 中训练的深度学习模型可以导出为 ONNX 格式，然后导入 TensorFlow。

ONNX 模型还可以与 **ONNX Runtime** 配合使用

ONNX Runtime 是一款通用的跨平台机器学习模型加速器，兼容 PyTorch、TensorFlow 等框架。

通过 ultralytics 快速导出

```python
from ultralytics import YOLO
model = YOLO("yolov8n.pt")
model.export(
    format="onnx",
    imgsz=640,
    opset=14,        #算子集版本TensorRT推荐13~14；OpenVINO 14~17
    dynamic=False,   # True=动态batch+hw
    simplify=True,
)
```

验证onnx模型导出成功

```python
import onnx
onnx_model = onnx.load("yolo11n.onnx")
onnx.checker.check_model(onnx_model)
print("ONNX模型校验通过")
```

以可读的形式打印计算图

```python
import onnx
onnx_model = onnx.load("yolo11n.onnx")
# 上面两句和验证导出的重复了，一起的话可以不写

print(onnx.helper.printable_graph(onnx_model.graph)) #打印到终端

# 下面的可以保存为txt文档的格式
graph_str = onnx.helper.printable_graph(onnx_model.graph)
with open("yolo11n_graph.txt", "w", encoding="utf-8") as f:
    f.write(graph_str)
print("图信息已保存到 yolo11n_graph.txt")
```

**导出的ONNX有两种量化模式**

分别的导出命令如下：

```python
from ultralytics import YOLO

# 加载 YOLO26 模型
model = YOLO("yolo26n.pt")

# 将模型导出为 ONNX 格式
model.export(format="onnx")  # 创建 'yolo26n.onnx'

# 使用校准数据导出 INT8 量化的 ONNX 模型
model.export(format="onnx", quantize=8, data="coco8.yaml")  # 创建 'yolo26n_int8.onnx'
```

**两个不同量化的区别：**

最大区别在于量化后

模型的文件大小**减半**

但在Intel的cpu上运行速度**翻倍**

但是精度会**减少一点**

| 项目 | FP32 onnx（不加 quantize） | INT8 量化 onnx（quantize=8 + data） |
| --- | --- | --- |
| 张量数据类型 | 全部 float32 | 权重、激活走 int8，靠 QuantizeLinear / DequantizeLinear 转换 |
| 文件大小 | 大 | 大约减半 |
| 图节点 | 只有卷积、BN、SiLU 等算子 | 多了大量 QuantizeLinear / DequantizeLinear 节点 |
| 是否需要校准集 | 不需要 | **必须**，用 coco8.yaml 里的图片统计激活范围 |
| 推理速度 | 慢 | CPU / Intel NPU 明显更快；N 卡要看硬件 |
| 精度 | 最好，几乎和 PT 原模型一致 | 有小幅精度损失，校准集和真实业务越接近损失越小 |
| 适合后端 | TensorRT、OpenVINO、ORT 都友好 | 优先 OpenVINO / ORT CPU；原生 TensorRT 不太推荐 |

## OpenVINO介绍

*详情见：Intel OpenVINO 导出 | Ultralytics*

OpenVINO 是 *Open Visual Inference & Neural Network Optimization toolkit* 的简称

是一套用于优化和部署 AI 推理模型的综合工具包。

同样也可以按照 **int8** 量化

把最后一条指令改为：`model.export(format="openvino", quantize=8, data="coco8.yaml")`

导出OpenVINO模型

```python
from ultralytics import YOLO

# 加载模型
model = YOLO("yolo26n.pt")

# 将模型导出为 OpenVINO 格式
model.export(format="openvino")
```

## TensorRT介绍

*详情见：YOLO26 模型的 TensorRT 导出 | Ultralytics*

TensorRT由 **NVIDIA** 开发，是一款先进的软件开发套件 (SDK)，专为高速深度学习推理而设计。

CLI导出tensorRT模型

```bash
trtexec --onnx=yolo26n.onnx --saveEngine=yolo26n_fp16.engine --fp16 #开启半精度

trtexec --onnx=yolo26n.onnx --saveEngine=yolo26n_fp32.engine #开启全精度
```

## 原生PyTorch模型/ONNX模型/OpenVINO推理时间对比

使用的是撞线计数的数据标注和模型训练(YOLO)里训练完的 `best.pt` 模型

**结果：**

![[YOLO-推理耗时对比.png]]

对比代码如下：

```python
import time
import cv2
import numpy as np
import torch
import onnxruntime as ort
import openvino as ov
from ultralytics import YOLO

IMG_PATH = "2026_10_14_15_50_31_145.jpg"
IMG_SIZE = 640
WARMUP = 5
TEST_ROUND = 20

# 共用预处理，和YOLO内部对齐
def preprocess(img_path, size=640):
    im = cv2.imread(img_path)
    im_rgb = cv2.cvtColor(im, cv2.COLOR_BGR2RGB)
    h0, w0 = im_rgb.shape[:2]
    r = min(size / h0, size / w0)
    h, w = round(h0 * r), round(w0 * r)
    im_res = cv2.resize(im_rgb, (w, h), interpolation=cv2.INTER_LINEAR)
    im_pad = np.full((size, size, 3), 114, dtype=np.uint8)
    im_pad[:h, :w] = im_res
    im_pad = im_pad.transpose(2, 0, 1) / 255.0
    im_tensor = np.expand_dims(im_pad, axis=0).astype(np.float32)
    return im_tensor, (h0,w0,r,h,w)

# ---------------- PT Torch推理 ----------------
def infer_pt(img_tensor):
    model = YOLO("best.pt")
    pt_model = model.model
    pt_model.eval()
    with torch.no_grad():
        x = torch.from_numpy(img_tensor)
        # warmup
        for _ in range(WARMUP):
            _ = pt_model(x)
        # 计时
        t0 = time.perf_counter()
        for _ in range(TEST_ROUND):
            out = pt_model(x)
        t1 = time.perf_counter()
        avg_ms = (t1 - t0) / TEST_ROUND * 1000
        print(f"[PT Torch] 平均推理耗时: {avg_ms:.2f} ms")
    return out

# ---------------- ONNX Runtime推理 ----------------
def infer_onnx(img_tensor):
    sess = ort.InferenceSession("best.onnx", providers=["CPUExecutionProvider"])
    input_name = sess.get_inputs()[0].name
    output_name = sess.get_outputs()[0].name
    # warmup
    for _ in range(WARMUP):
        _ = sess.run([output_name], {input_name: img_tensor})
    t0 = time.perf_counter()
    for _ in range(TEST_ROUND):
        out = sess.run([output_name], {input_name: img_tensor})
    t1 = time.perf_counter()
    avg_ms = (t1 - t0) / TEST_ROUND * 1000
    print(f"[ONNX Runtime CPU] 平均推理耗时: {avg_ms:.2f} ms")
    return out

# ---------------- OpenVINO推理----------------
def infer_openvino(img_tensor, ov_model_dir="best_openvino_model"):
    core = ov.Core()
    compiled_model = core.compile_model(f"{ov_model_dir}/best.xml", device_name="CPU")
    input_key = compiled_model.input(0)
    output_key = compiled_model.output(0)
    # warmup
    for _ in range(WARMUP):
        _ = compiled_model({input_key: img_tensor})
    t0 = time.perf_counter()
    for _ in range(TEST_ROUND):
        out = compiled_model({input_key: img_tensor})[output_key]
    t1 = time.perf_counter()
    avg_ms = (t1 - t0) / TEST_ROUND * 1000
    print(f"[OpenVINO CPU] 平均推理耗时: {avg_ms:.2f} ms")
    return out

if __name__ == "__main__":
    tensor, info = preprocess(IMG_PATH, IMG_SIZE)
    _pt_out = infer_pt(tensor)
    _onnx_out = infer_onnx(tensor)
    _ov_out = infer_openvino(tensor)
```
