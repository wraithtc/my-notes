---
title: YOLO-World 调研报告
date: 2026-04-01
tags:
  - AI
  - 目标检测
  - 开放词汇检测
  - YOLO
  - 宠物监控
  - 边缘部署
category: 技术调研
---

# YOLO-World：实时开放词汇目标检测

> [!info] 核心定位
> YOLO-World 是由腾讯 AI Lab、华中科技大学等联合提出的**实时开放词汇目标检测**模型（CVPR 2024）。它在保持 YOLO 系列实时推理速度的同时，支持通过**自然语言文本**指定任意类别进行检测，无需重新训练。

> [!tip] 对宠物监控场景的价值
> 传统 YOLO 只能检测预训练的固定类别（如 COCO 80 类中的 "cat"/"dog"）。YOLO-World 允许你在推理时**动态指定检测类别**，例如：
> - `"cat, dog, pet bowl, pet toy, litter box, scratching post"`
> - `"cat vomiting, cat limping, dog chewing furniture"`
> - 无需为每新增一个类别就重新收集标注数据、重新训练模型。

---

## 1. 技术背景与动机

### 1.1 传统目标检测的局限

| 方法 | 词汇范围 | 增加新类别 | 速度 |
|------|----------|-----------|------|
| YOLOv5/v8/v11 (闭词汇) | 固定 80 类 (COCO) | 需重新训练 | ⚡ 实时 |
| Grounding DINO | 开放词汇 | 仅需文本 | 🐢 5-10 FPS |
| GLIP | 开放词汇 | 仅需文本 | 🐢 慢 |
| **YOLO-World** | **开放词汇** | **仅需文本** | **⚡ 实时 35+ FPS** |

### 1.2 核心问题

> 如何让 YOLO 既能实时运行，又能像 CLIP/Grounding DINO 一样检测任意文本描述的物体？

YOLO-World 的答案：**Prompt-then-Detect** —— 先用文本编码器生成类别嵌入，再通过跨模态融合注入 YOLO 检测流水线，推理时通过**重参数化**消除语言分支的计算开销。

---

## 2. 架构详解

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────┐
│                   YOLO-World 架构                     │
│                                                       │
│  ┌──────────────┐     ┌──────────────────────────┐   │
│  │  Text Prompt  │     │      Input Image          │   │
│  │ "cat, dog,    │     │                            │   │
│  │  pet bowl"    │     │                            │   │
│  └──────┬───────┘     └────────────┬───────────────┘   │
│         │                          │                    │
│         ▼                          ▼                    │
│  ┌──────────────┐     ┌──────────────────────────┐   │
│  │ CLIP Text     │     │  YOLOv8 CSP Backbone     │   │
│  │ Encoder       │     │  (多尺度特征提取)          │   │
│  └──────┬───────┘     └────────────┬───────────────┘   │
│         │                          │                    │
│         │    Text Embeddings        │  Visual Features   │
│         │         │                 │       │           │
│         └─────────┼─────────────────┘       │           │
│                   ▼                         │           │
│         ┌─────────────────────┐             │           │
│         │   RepVL-PAN Neck    │◄────────────┘           │
│         │  (跨模态特征融合)     │                         │
│         └─────────┬───────────┘                          │
│                   │                                      │
│                   ▼                                      │
│         ┌─────────────────────┐                          │
│         │   Detection Head    │                          │
│         │  (对比匹配分类)      │                          │
│         └─────────┬───────────┘                          │
│                   │                                      │
│                   ▼                                      │
│         Bounding Boxes + Categories                      │
└─────────────────────────────────────────────────────┘
```

### 2.2 核心组件

#### 2.2.1 CLIP Text Encoder

- 使用 **CLIP ViT-L/14** 或 **CLIP ViT-B/32** 作为文本编码器
- 将类别文本（如 `"cat"`）编码为**固定维度向量**（如 512 维或 768 维）
- 支持一次性编码所有候选类别，作为动态分类权重

#### 2.2.2 YOLOv8 CSP Backbone

- 基于 **YOLOv8 的 CSPDarknet** 骨干网络
- 提取多尺度特征：P3 (80×80)、P4 (40×40)、P5 (20×20)
- 保持 YOLO 的**高效特征提取**能力

#### 2.2.3 RepVL-PAN（核心创新 ⭐）

**Re-parameterizable Vision-Language Path Aggregation Network** 是 YOLO-World 的核心创新：

```
特征金字塔层级 (P3/P4/P5):
┌─────────────────────────────────────────────┐
│                                              │
│  Visual Features ──► Vision-Language Fusion  │
│                         │          ▲         │
│                    Cross-Modal    Cross-Modal │
│                    Attention      Attention   │
│                         │          │         │
│  Text Embeddings ──► VL-Fuse ◄── LV-Fuse    │
│                                              │
│  训练时: 双向跨模态注意力融合                    │
│  推理时: 重参数化为标准卷积 (零额外开销!)        │
│                                              │
└─────────────────────────────────────────────┘
```

**关键设计：**
- **VL-Fuse (Vision→Language)**：图像特征增强文本嵌入的空间感知能力
- **LV-Fuse (Language→Vision)**：文本语义引导图像特征关注相关区域
- **重参数化**：训练完成后，融合模块可折叠为标准卷积参数，推理时**无额外计算成本**

#### 2.2.4 Detection Head

- 采用**区域-文本对比学习**进行分类
- 检测头的分类分支将 ROI 特征与文本嵌入进行**余弦相似度匹配**
- 回归分支与标准 YOLO 一致（CIoU Loss + DFL）

### 2.3 训练策略（三阶段）

| 阶段 | 数据 | 目标 |
|------|------|------|
| **Stage 1: 预训练** | 大规模 region-text 数据（Objects365, GoldG, CC3M 等，共数百万图文对） | 对齐视觉-语言特征空间 |
| **Stage 2: 检测微调** | COCO, LVIS 等标注数据集 | 精炼定位能力 |
| **Stage 3: 推理部署** | 用户提供文本 prompt → 编码 → 重参数化 → 单次前向推理 | 零样本检测新类别 |

---

## 3. 性能对比

### 3.1 与其他开放词汇检测器对比

| 方法 | 骨干网络 | LVIS mAP | COCO mAP | 速度 (V100) | 参数量 |
|------|---------|----------|----------|------------|--------|
| GLIP-T | Swin-T | 26.5 | 50.2 | ~8 FPS | ~150M |
| Grounding DINO-T | Swin-T | 27.4 | 52.5 | ~6 FPS | ~170M |
| Grounding DINO-L | Swin-L | 33.2 | 57.2 | ~3 FPS | ~690M |
| **YOLO-World-L** | **CSPDarknet-L** | **~35** | **~52** | **~35 FPS** | **~100M** |
| **YOLO-World-X** | **CSPDarknet-X** | **~37** | **~54** | **~25 FPS** | **~140M** |

> [!note] 关键结论
> YOLO-World 在速度上领先 Grounding DINO **5-10 倍**，精度相当甚至更优。这是在边缘设备上部署开放词汇检测的**唯一可行方案**。

### 3.2 不同模型尺寸

| 模型变体 | 参数量 | FLOPs | 推理速度 (T4) | 适用场景 |
|---------|--------|-------|--------------|---------|
| YOLO-World-S | ~30M | ~50G | ~60 FPS | 高实时性边缘设备 |
| YOLO-World-M | ~60M | ~100G | ~45 FPS | 平衡型 |
| YOLO-World-L | ~100M | ~180G | ~35 FPS | 高精度 |
| YOLO-World-X | ~140M | ~280G | ~25 FPS | 最高精度 |

---

## 4. 宠物家用监控场景分析

### 4.1 应用场景需求

| 需求 | 描述 | YOLO-World 适配性 |
|------|------|-----------------|
| 宠物识别 | 识别猫、狗、特定品种 | ✅ 原生支持，零样本检测 |
| 行为检测 | 呕吐、拆家、进食、排泄 | ⚠️ 需要结合行为分类器或描述性文本 |
| 物品检测 | 食盆、猫砂盆、玩具、窝 | ✅ 开放词汇，直接指定类别名 |
| 异常事件 | 陌生人闯入、宠物打架 | ✅ 灵活添加新类别 |
| 实时性 | 家用 IPC/NAS/边缘盒子上运行 | ✅ 核心优势，实时推理 |
| 多宠物区分 | 区分家里不同的宠物 | ⚖️ 需配合 ReID 模块 |

### 4.2 推荐技术方案

```
┌─────────────────────────────────────────────────────────────┐
│                 宠物家用监控系统架构                           │
│                                                              │
│  ┌─────────┐    ┌──────────────┐    ┌───────────────────┐   │
│  │ RTSP/   │───►│  YOLO-World  │───►│  后处理 & 业务逻辑  │   │
│  │ USB Cam │    │  检测引擎     │    │                   │   │
│  └─────────┘    └──────┬───────┘    └───────────────────┘   │
│                        │                                     │
│          ┌─────────────┼─────────────┐                      │
│          ▼             ▼             ▼                      │
│   ┌────────────┐ ┌──────────┐ ┌──────────────┐            │
│   │ 宠物追踪    │ │ 行为分类  │ │ 异常事件告警   │            │
│   │ (DeepSORT) │ │ (分类头)  │ │ (规则引擎)    │            │
│   └────────────┘ └──────────┘ └──────────────┘            │
│                                                              │
│  Prompt 示例:                                                │
│  "cat, dog, pet food bowl, water bowl, litter box,          │
│   pet toy, sofa, person, cat scratching furniture,          │
│   dog chewing, pet vomit, broken object"                    │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 Prompt 工程（类别设计）

YOLO-World 对 prompt 设计比较敏感，以下给出宠物场景的推荐类别设计：

#### 基础检测层（高置信度）

```python
base_categories = [
    "cat", "dog", "person",
    "food bowl", "water bowl",
    "pet bed", "cat tree",
    "litter box", "pet toy",
    "couch", "table", "floor"
]
```

#### 行为/状态检测层（需验证）

```python
behavior_categories = [
    "eating cat food",          # 进食
    "drinking water",           # 喝水
    "sleeping on couch",        # 沙发睡觉
    "scratching sofa",          # 抓沙发
    "chewing furniture",        # 咬家具
    "vomiting on floor",        # 呕吐
    "knocking over objects",    # 打翻物品
    "pet waste on floor",       # 排泄物
    "climbing counter",         # 爬台面
    "playing with toy",         # 玩玩具
]
```

> [!warning] 注意事项
> YOLO-World 的开放词汇能力**对简单名词/物体类别效果最好**，对复杂行为描述（动词+名词组合）的效果可能有限。建议：
> 1. 先用物体检测定位关键区域
> 2. 再用姿态估计或时序模型判断行为
> 3. 或微调添加行为类别

### 4.4 部署方案建议

| 方案 | 硬件 | 模型选择 | 预期 FPS | 适用场景 |
|------|------|---------|---------|---------|
| **云端推理** | GPU 服务器 (T4/A10) | YOLO-World-X | 25-40 | 多摄像头聚合 |
| **边缘盒子** | Jetson Orin NX / RK3588 | YOLO-World-S/M | 15-30 | 单摄像头本地推理 |
| **NAS 部署** | Synology+外接 GPU / NPU | YOLO-World-S | 10-20 | 家庭 NAS |
| **IPC 内置** | 瑞芯微 RK3588 / 地平线 J5 | YOLO-World-S (INT8) | 10-15 | 摄像头内置 |

#### TensorRT 部署流程

```bash
# 1. 导出 ONNX (含重参数化)
yolo export model=yolov8s-worldv2.pt format=onnx

# 2. 转换为 TensorRT
trtexec --onnx=yolov8s-worldv2.onnx \
        --saveEngine=yolov8s-worldv2.engine \
        --fp16

# 3. 推理时动态设置类别
# Python API 中通过 custom={"classes": [...]} 指定
```

---

## 5. 实践指南

### 5.1 Ultralytics 快速上手

```python
from ultralytics import YOLO

# 加载预训练 YOLO-World 模型
model = YOLO("yolov8s-worldv2.pt")

# 设置自定义类别 (宠物监控场景)
model.set_classes([
    "cat", "dog", "person",
    "food bowl", "water bowl", "litter box",
    "pet toy", "pet bed"
])

# 推理
results = model.predict(
    source="rtsp://camera-stream-url",
    stream=True,       # 流式处理
    conf=0.25,         # 置信度阈值
    iou=0.45,          # NMS IoU 阈值
    imgsz=640,         # 输入尺寸
)

for result in results:
    boxes = result.boxes
    for box in boxes:
        cls = int(box.cls)
        conf = float(box.conf)
        xyxy = box.xyxy.cpu().numpy()
        label = model.names[cls]
        print(f"检测到: {label}, 置信度: {conf:.2f}, 位置: {xyxy}")
```

### 5.2 微调自定义数据（可选）

如果零样本效果不理想（如特定品种的猫/狗、特定家用物品），可以进行微调：

```python
from ultralytics import YOLO

# 加载 YOLO-World 预训练权重
model = YOLO("yolov8s-worldv2.pt")

# 准备数据集 (YOLO 格式)
# dataset.yaml:
# path: ./pet_dataset
# train: images/train
# val: images/val
# names:
#   0: cat
#   1: dog
#   2: persian_cat
#   3: corgi
#   4: auto_feeder
#   5: cat_tower

# 微调训练
model.train(
    data="pet_dataset.yaml",
    epochs=50,
    imgsz=640,
    batch=16,
    lr0=0.001,
    name="yolo-world-pet-monitor",
)
```

### 5.3 与跟踪算法集成

```python
from ultralytics import YOLO

model = YOLO("yolov8s-worldv2.pt")
model.set_classes(["cat", "dog", "person"])

# 使用内置 BoT-SORT 跟踪
results = model.track(
    source="rtsp://camera-url",
    stream=True,
    tracker="botsort.yaml",  # 或 "bytetrack.yaml"
    persist=True,            # 保持跟踪状态
)

for result in results:
    if result.boxes.id is not None:
        track_ids = result.boxes.id.cpu().numpy()
        # 实现宠物行为轨迹分析
        for track_id, box in zip(track_ids, result.boxes):
            print(f"宠物 #{track_id}: {model.names[int(box.cls)]}")
```

---

## 6. 优势与局限

### 6.1 优势

| 优势 | 说明 |
|------|------|
| **实时性** | 35+ FPS (V100)，唯一能在边缘设备实时运行的开放词汇检测器 |
| **零样本能力** | 无需标注数据即可检测新类别，大幅降低数据成本 |
| **灵活扩展** | 推理时动态更换检测类别，一套模型覆盖多种场景 |
| **部署友好** | 重参数化后与标准 YOLO 推理开销一致 |
| **生态完善** | 已集成 Ultralytics，支持 ONNX/TensorRT/OpenVINO 导出 |

### 6.2 局限性

| 局限 | 说明 | 应对策略 |
|------|------|---------|
| 小目标检测偏弱 | 远距离、小尺寸宠物可能漏检 | 使用更高分辨率输入 (imgsz=1280) |
| 行为/动作检测有限 | 开放词汇更擅长物体而非动作 | 结合姿态估计 + 时序模型 |
| 特定品种区分 | 对细粒度品种区分能力有限 | 用特定品种数据微调 |
| 中文 prompt 效果 | CLIP 文本编码器以英文为主 | 使用英文类别名，或微调中文编码器 |
| 复杂场景遮挡 | 宠物被家具遮挡时检测率下降 | 多摄像头互补 + 跟踪算法 |

---

## 7. 与竞品对比（宠物场景）

| 维度 | YOLO-World | YOLOv8 (闭词汇) | Grounding DINO | Florence-2 |
|------|-----------|----------------|----------------|-----------|
| **开放词汇** | ✅ | ❌ | ✅ | ✅ |
| **实时性** | ⚡ 35 FPS | ⚡ 50 FPS | 🐢 5 FPS | 🐢 10 FPS |
| **边缘部署** | ✅ | ✅ | ❌ | ⚠️ |
| **自定义类别** | 即时切换 | 需重训 | 即时切换 | 即时切换 |
| **宠物检测精度** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ (同类) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **家用物品检测** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **集成难度** | 低 (Ultralytics) | 低 | 中 | 中 |
| **综合推荐** | **⭐⭐⭐⭐⭐** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |

> [!success] 推荐结论
> 对于宠物家用监控场景，**YOLO-World 是最佳选择**：
> 1. 开放词汇能力让你无需为每种家用物品和宠物状态收集标注数据
> 2. 实时性保证在边缘设备上流畅运行
> 3. Ultralytics 生态完善，部署和迭代成本低
> 4. 可通过微调进一步提升特定场景精度

---

## 8. 参考资料

- **论文**: *YOLO-World: Real-Time Open-Vocabulary Object Detection* (Cheng et al., CVPR 2024)
- **GitHub**: [AILab-CVC/YOLO-World](https://github.com/AILab-CVC/YOLO-World)
- **Ultralytics 文档**: [docs.ultralytics.com/models/yolo-world](https://docs.ultralytics.com/models/yolo-world/)
- **arXiv**: [2401.17270](https://arxiv.org/abs/2401.17270)
- **视频解读**: YOLO-World 原理详解（B站可搜索）
