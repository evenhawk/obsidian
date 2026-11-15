---
title: ONNX / OpenVINO / TensorRT 模型部署笔记
tags:
  - onnx
  - openvino
  - tensorrt
  - 模型部署
  - 推理优化
  - 计算机视觉
status: maintained
updated: 2026-11-15
---

# ONNX / OpenVINO / TensorRT 模型部署笔记

> [!abstract] 一句话总结
> 各训练框架各产各的模型 → 统一导出成 **ONNX** 这个中间格式 → 交给目标硬件上的推理引擎运行：NVIDIA GPU 用 **TensorRT**，Intel CPU 用 **OpenVINO**，通用平台用 **ONNX Runtime**。再转成 INT8 量化，能换来体积约减半、Intel CPU 上速度约翻倍，代价是精度略降。

## 一、整体思路

![[ONNX-部署架构图.png]]

1. 用不同的训练框架训练 → 得到各自框架的模型
2. 统一转换为 ONNX 格式（中间格式）
3. 把这个中间格式放到硬件本地安装的推理框架上去运行

| 推理引擎 | 面向硬件 |
| --- | --- |
| TensorRT | NVIDIA GPU |
| OpenVINO | Intel CPU（及 Intel NPU） |
| ONNX Runtime | 通用 CPU / 跨平台 |

不同的训练框架兼容的硬件设备不同，ONNX 格式在这里起到了**桥梁**的作用。

## 二、ONNX 是什么

- 全称 **Open Neural Network Exchange**（开放神经网络交换）。原文写的 "open neurl network exchange" 拼写有误，是 Neural 不是 neurl。
- 是一种开放的文件格式，用于表示机器学习模型，让这些模型能在不同的 AI 框架和硬件上使用。
- 四个「兼容」：
  1. 开源机器学习的通用中间格式
  2. 兼容各种深度学习框架
  3. 兼容各种推理引擎
  4. 兼容各种终端硬件、各种操作系统
- **跨框架迁移**：在 PyTorch 中训练的模型可以导出为 ONNX，再导入 TensorFlow 使用。
- 可与 **ONNX Runtime** 配合：ONNX Runtime 是一款通用的跨平台机器学习模型加速器，兼容 PyTorch、TensorFlow 等框架。

## 三、用 Ultralytics 导出 ONNX

```python
from ultralytics import YOLO

model = YOLO("yolo26n.pt")

model.export(
    format="onnx",
    imgsz=640,
    opset=14,        # 算子集版本：TensorRT 推荐 13~14；OpenVINO 14~17
    dynamic=False,   # True = 动态 batch + 动态 H/W
    simplify=True,
)
```

### 校验导出是否成功

```python
import onnx

onnx_model = onnx.load("yolo26n.onnx")
onnx.checker.check_model(onnx_model)
print("ONNX 模型校验通过")
```

### 以可读形式打印计算图

```python
import onnx

onnx_model = onnx.load("yolo26n.onnx")   # 和上面校验代码重复，连在一起写时不必重复加载

print(onnx.helper.printable_graph(onnx_model.graph))   # 打印到终端

# 也可以保存成 txt 文件
graph_str = onnx.helper.printable_graph(onnx_model.graph)
with open("yolo26n_graph.txt", "w", encoding="utf-8") as f:
    f.write(graph_str)
print("图信息已保存到 yolo26n_graph.txt")
```

## 四、两种精度：FP32 与 INT8 量化

原文说「两种量化模式」，更准确的说法是**两种精度**：普通 FP32，以及 INT8 量化（只有后者才是量化）。

```python
from ultralytics import YOLO

# 加载模型
model = YOLO("yolo26n.pt")

# 导出普通 FP32 ONNX
model.export(format="onnx")                                  # 生成 yolo26n.onnx

# 用校准数据导出 INT8 量化的 ONNX
model.export(format="onnx", quantize=8, data="coco8.yaml")   # 生成 yolo26n_int8.onnx
```

| 项目 | FP32 onnx（不加 quantize） | INT8 量化 onnx（quantize=8 + data） |
| --- | --- | --- |
| 张量数据类型 | 全部 float32 | 权重、激活走 int8，靠 QuantizeLinear / DequantizeLinear 转换 |
| 文件大小 | 大 | 大约减半 |
| 图节点 | 只有卷积、BN、SiLU 等算子 | 多了大量 QuantizeLinear / DequantizeLinear 节点 |
| 是否需要校准集 | 不需要 | **必须**，用 coco8.yaml 里的图片统计激活范围 |
| 推理速度 | 慢 | CPU / Intel NPU 明显更快；N 卡要看硬件 |
| 精度 | 最好，几乎和 PT 原模型一致 | 有小幅精度损失，校准集和真实业务越接近损失越小 |
| 适合后端 | TensorRT、OpenVINO、ORT 都友好 | 优先 OpenVINO / ORT CPU；原生 TensorRT 不太推荐 |

> [!tip] 用一句话记住差别
> 位宽从 FP32 砍到 INT8（位宽减半）后：**体积约减半、Intel CPU 上速度约翻倍、精度略降**。
> 注意原文「文件大小减少了一倍 / 运行速度增加了一倍」的说法不严谨——「减少一倍」等于消失，应说「约减半 / 约翻倍」。

## 五、OpenVINO

- 全称 **Open Visual Inference & Neural Network Optimization toolkit**。
- 是一套用于优化和部署 AI 推理模型的综合工具包，主打 Intel CPU / NPU。
- 同样支持 INT8 量化：把最后一条导出指令改成 `model.export(format="openvino", quantize=8, data="coco8.yaml")`。

```python
from ultralytics import YOLO

# 加载模型
model = YOLO("yolo26n.pt")

# 导出 OpenVINO 格式
model.export(format="openvino")
```

> [!note] 导出产物是个目录
> 会生成 `xxx_openvino_model/` 目录，里面是 `.xml`（网络结构）+ `.bin`（权重），所以下面推理代码里加载的是 `best_openvino_model/best.xml`。

## 六、TensorRT

- 由 NVIDIA 开发，是一款先进的软件开发套件（SDK），专为高速深度学习推理而设计。
- 用 CLI 转换 TensorRT 模型：

```bash
# 半精度 FP16
trtexec --onnx=yolo26n.onnx --saveEngine=yolo26n_fp16.engine --fp16

# 全精度 FP32（默认精度就是 FP32，不需要额外加精度参数）
trtexec --onnx=yolo26n.onnx --saveEngine=yolo26n_fp32.engine
```

> [!warning] 关于原文里的 `--fp32`
> 多数 trtexec 版本**没有** `--fp32` 这个选项——不加 `--fp16` / `--int8` 时默认就是 FP32。照原文执行可能报 unknown option，把 `--fp32` 删掉即可。
> 另外要做 INT8 的 engine，需要额外提供校准集/校准缓存，不是加个开关就行。

## 七、实测：PyTorch / ONNX / OpenVINO 推理耗时对比

背景：用的是「撞线计数」项目里数据标注和模型训练（YOLO）跑完得到的 `best.pt` 模型。

结果（作者截图）：

![[YOLO-推理耗时对比.png]]

> [!note] 看数字前先看测量口径
> - 只测**网络前向**（`pt_model(x)` / `sess.run` / `compiled_model(...)`），**不含 NMS 和后处理**，所以是横向比较用的相对值，不等于端到端延迟。
> - 先 `WARMUP = 5` 次热身，再 `TEST_ROUND = 20` 轮取平均，排除首次加载 / 编译开销。
> - 三者都在 **CPU** 上测（ORT 用 `CPUExecutionProvider`，OpenVINO 用 `device_name="CPU"`）。PyTorch 若改用 GPU，结论会完全不同。

### 对比代码

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


# 共用预处理，和 YOLO 内部对齐
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
    return im_tensor, (h0, w0, r, h, w)


# ---------------- PT Torch 推理 ----------------
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


# ---------------- ONNX Runtime 推理 ----------------
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


# ---------------- OpenVINO 推理 ----------------
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

## 八、速查

| 想做的事 | 做法 |
| --- | --- |
| 导出 FP32 ONNX | `model.export(format="onnx")` |
| 导出 INT8 ONNX | `model.export(format="onnx", quantize=8, data="校准集.yaml")` |
| 导出 OpenVINO | `model.export(format="openvino")` |
| 导出 INT8 OpenVINO | `model.export(format="openvino", quantize=8, data="校准集.yaml")` |
| ONNX → TensorRT engine | `trtexec --onnx=x.onnx --saveEngine=x.engine --fp16` |
| 校验 ONNX 文件 | `onnx.checker.check_model(onnx.load("x.onnx"))` |
| 打印计算图 | `onnx.helper.printable_graph(m.graph)` |

## 九、复习清单

- [ ] 能说出「训练框架 → ONNX → 推理引擎 → 硬件」这条链路
- [ ] 记得 TensorRT / OpenVINO / ONNX Runtime 分别对应什么硬件
- [ ] 记得 INT8 量化必须提供 `data` 校准集
- [ ] 记得 `opset` 的选择范围（TRT 13~14，OpenVINO 14~17）
- [ ] 记得 INT8 的三个效应：体积减半、CPU 提速、精度略降
- [ ] 说得清那次耗时对比测的是「前向」而不是端到端

## 十、自测（先自己想，再展开）

> [!question]- 1. 为什么不直接把 PyTorch 模型丢给 TensorRT？
> 各家框架的模型格式和算子实现都不同。ONNX 作为公开的中间格式，各框架都能导出、各推理引擎都能读，N 个框架对接 M 个硬件只需要 N + M 套适配，而不是 N × M 套。

> [!question]- 2. INT8 量化为什么必须有校准数据？
> 需要拿真实数据跑一遍、统计每层激活值的范围，才能算出缩放系数和零点。没有校准集就只能瞎猜范围，精度会崩。

> [!question]- 3. 体积减半、CPU 变快了，代价是什么？
> 精度会小幅下降；校准集与真实业务分布越接近，损失越小。另外不是所有硬件都吃得饱 INT8 加速（N 卡要看具体型号）。

> [!question]- 4. 导出 ONNX 时 `opset` 怎么选？
> 按目标推理引擎推荐的区间选：TensorRT 推荐 13~14，OpenVINO 14~17。不确定就取目标引擎支持区间内的偏新版本。

> [!question]- 5. `onnx.checker.check_model` 通过，就说明模型能正确推理吗？
> 不能。它只校验文件结构和算子是否符合 ONNX 规范，不检查数值精度、也不保证推理结果正确。

> [!question]- 6. TensorRT 的 `.engine` 文件能直接拷到另一台机器用吗？
> 一般不能。engine 和 GPU 架构、TensorRT / CUDA 版本强绑定，换环境要重新构建。

> [!question]- 7. 只看那张截图就说「OpenVINO 比 PyTorch 快 X 倍」，严谨吗？
> 不严谨：只覆盖了单张图、CPU、网络前向这三层限定。batch、分辨率、CPU 型号、是否含后处理都会改变结论。
