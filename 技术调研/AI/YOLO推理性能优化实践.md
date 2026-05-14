# YOLO推理性能优化实践

> 目标：优化宠物家庭监控视频推理端到端耗时，从 997ms 降至 497ms。

## 环境

- 模型：YOLOv8s（22MB），YOLOv8n（6.5MB）在本场景检测不到猫
- 设备：NVIDIA GPU (CUDA), WSL2
- 视频：720p H.264, 30s, 抽帧率 1fps → 30帧
- 任务：检测猫是否进入进食区（pet-zone overlap ≥ 0.2 + dwell ≥ 8s）

## 优化过程

### ① 换模型：YOLOv8n → YOLOv8s

| 模型 | 参数量 | 大小 | 猫检测帧数 | 事件 |
|------|--------|------|-----------|------|
| YOLOv8n (nano) | 3.2M | 6.5MB | 0/30 | 0 ❌ |
| YOLOv8s (small) | 11.2M | 22MB | 12/30 | 1 ✓ |

**结论**：YOLOv8n 在本场景完全检测不到猫，必须用 YOLOv8s 或更大型号。

**耗时**：YOLO推理 659ms，端到端 997ms。

### ② FP16 半精度推理

在 `pet_detector.py` 的 `detect_pets_batch()` 中加 `"half": True`：

```python
predict_kwargs = {
    "conf": conf_threshold, "imgsz": imgsz,
    "verbose": False, "batch": len(frame_paths),
    "half": True,  # 启用FP16
}
```

**效果**：YOLO 659ms → 504ms（↓24%），检测精度零损失（同为12帧检出猫）。

### ③ BGR raw pipe 直传 numpy 数组

**背景**：原流程先 ffmpeg 编码 JPEG 到磁盘 → YOLO 读文件解码 JPEG → 推理。JPEG 编解码来回浪费 CPU 时间。

**优化**：ffmpeg 直接输出 BGR24 原始像素到管道 → numpy 数组 → 直接喂给 YOLO。

```python
# ffmpeg输出raw BGR像素流
proc = subprocess.Popen([
    "ffmpeg", "-y", "-i", video_path,
    "-vf", f"fps={fps}", "-vsync", "vfr",
    "-f", "rawvideo", "-pix_fmt", "bgr24",
    "-loglevel", "error", "-"
], stdout=subprocess.PIPE)

# 零拷贝转numpy → 直接给YOLO
all_frames = np.frombuffer(raw_data, dtype=np.uint8).reshape(N, H, W, 3)
detections = model(frames_np, conf=0.25, imgsz=640, half=True)
```

**效果**：YOLO 504ms → 225ms（↓55%），检测帧数 12→14（BGR格式匹配OpenCV反而提升检出）。

### ④ 细粒度优化

- **cv2.VideoCapture 替代 ffprobe**：获取视频尺寸 ~1ms vs ~80ms
- **numpy 零拷贝 reshape**：`np.frombuffer(raw).reshape(N, H, W, 3)` 一次分配，逐帧 `[i]` 取视图

**效果**：端到端 748ms → 516ms（↓31%）。

### ⑤ 并行 JPEG 编码存盘

推理后的帧需存为 JPEG 供 replay 回放。用 `ThreadPoolExecutor(4)` 并行编码：

```python
with ThreadPoolExecutor(max_workers=4) as pool:
    futures = [pool.submit(cv2.imwrite, fp, arr, [cv2.IMWRITE_JPEG_QUALITY, 75])
               for arr, fp in zip(frames_np, frame_paths)]
```

JPEG 质量从 90 调到 75（肉眼无差别），配合 4 线程并行：157ms → 31ms（↓80%）。

## 最终效果

| 指标 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| 端到端耗时 | 997ms | **497ms** | ↓50% |
| YOLO推理 | 659ms | **147ms** | ↓78% |
| ffmpeg解码 | 335ms | 320ms | 持平 |
| JPEG存盘 | ~160ms | ~30ms | ↓81% |
| 检测事件数 | 1 | 1 | 不变 |

## 瓶颈分析（497ms 分布）

| 环节 | 耗时 | 占比 | 优化空间 |
|------|------|------|---------|
| ffmpeg H.264 解码 | ~320ms | 64% | CPU软解已近极限，NVDEC反更慢 |
| YOLO 推理 | ~150ms | 30% | 每帧5ms，fp16已到位 |
| JPEG编码+pipeline | ~30ms | 6% | 4线程并行已到位 |

## NVDEC 调研

测试了多种 NVDEC 方案（`-hwaccel cuda`、`-c:v h264_cuvid`、GPU滤镜链），**全部比 CPU 软解慢**。

原因：NVDEC 解码在 GPU 显存，720p 原始帧需从 GPU→CPU 走 PCIe 传输（30帧×2.76MB=83MB）。这个传输开销比 CPU 直接解码 H.264 还大。NVDEC 适合 4K/高帧率/多路并发场景，对短时低分辨率视频不适用。

## 踩坑记录

1. **YOLO 输入格式必须是 BGR**：YOLO 用 OpenCV 加载图像，内部期望 BGR。ffmpeg 输出 RGB 时检测数下降（12→8），改为 BGR24 后回升至 14。
2. **numpy reshape 返回视图是连续内存**：`arr.reshape(N,H,W,3)[i]` 返回的 `(H,W,3)` 视图 `C_CONTIGUOUS=True`，YOLO 可直接消费，无需 `ascontiguousarray` 拷贝。
3. **imgsz 不宜降低**：imgsz=480 时检测帧从 12 降到 8，且连续性断开导致 dwell 重置，事件无法触发。imgsz=640 是保证检测连续性的最低可用值。

## 相关文件

- `src/pet_home_monitor/services/video_processor.py` — BGR pipe + 并行JPEG
- `src/pet_home_monitor/services/pet_detector.py` — FP16半精度
- `src/pet_home_monitor/api/app.py` — CORS中间件

## 日期

2026-05-13
