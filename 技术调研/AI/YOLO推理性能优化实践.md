# YOLO推理性能优化实践

> 目标：优化宠物家庭监控视频推理端到端耗时，从 997ms 降至 ~460ms。

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

**原理**：GPU 的 Tensor Core 对 FP16（半精度浮点）的吞吐量是 FP32 的 2 倍。YOLO 推理中大部分计算是矩阵乘法（卷积），这些运算在 FP16 下精度损失极小（检测数不变），但速度显著提升。

ultralytics 的 `predict()` 方法支持 `half=True` 参数，内部会将模型权重和输入张量转为 FP16，推理完成后再转回 FP32 输出。模型本身仍以 FP32 存储在磁盘上。

```python
predict_kwargs = {
    "conf": conf_threshold, "imgsz": imgsz,
    "verbose": False, "batch": len(frame_paths),
    "half": True,  # 启用FP16
}
results = model(frame_paths, **predict_kwargs)
```

**效果**：YOLO 659ms → 504ms（↓24%），检测精度零损失（同为12帧检出猫）。

### ③ BGR raw pipe 直传 numpy 数组

**原理**：原流程的 JPEG 编解码是纯浪费——ffmpeg 把 H.264 解码后的原始像素编码为 JPEG 写入磁盘，YOLO 再从磁盘读取 JPEG 解码还原为像素。这个"像素→JPEG→像素"的往返浪费了大量 CPU 时间（编码约100ms + 解码约250ms）。

改为 ffmpeg 直接通过管道输出 BGR24 原始像素（零压缩），Python 端 `np.frombuffer` 零拷贝转为 numpy 数组，直接喂给 YOLO。消除了两次 JPEG 编解码和磁盘 I/O。

```
原流程：  ffmpeg → JPEG编码 → 磁盘I/O → JPEG解码 → YOLO     = 335 + 504 = 839ms
优化后：  ffmpeg → raw BGR pipe → numpy reshape → YOLO       = 320 + 225 = 545ms
```

```python
proc = subprocess.Popen([
    "ffmpeg", "-y", "-i", video_path,
    "-vf", f"fps={fps}", "-vsync", "vfr",
    "-f", "rawvideo", "-pix_fmt", "bgr24",
    "-loglevel", "error", "-"
], stdout=subprocess.PIPE)

raw_data = proc.stdout.read()
all_frames = np.frombuffer(raw_data, dtype=np.uint8).reshape(N, H, W, 3)
```

**效果**：YOLO 504ms → 225ms（↓55%），检测帧数 12→14（BGR格式匹配OpenCV反而提升检出）。

### ④ 细粒度优化

**原理**：

- **cv2.VideoCapture 替代 ffprobe**：ffprobe 是独立进程（fork+exec），需要进程创建+管道通信开销。cv2.VideoCapture 在进程内通过 FFmpeg 共享库读取视频元数据，无需进程切换，从 ~80ms 降到 ~1ms。

- **numpy 零拷贝 reshape**：`np.frombuffer(raw_data).reshape(N, H, W, 3)` 不复制数据，返回原始内存的视图。每个 `all_frames[i]` 也是零拷贝视图（`C_CONTIGUOUS=True`），YOLO 可以直接消费。

**效果**：端到端 748ms → 516ms（↓31%）。

### ⑤ 并行 JPEG 编码存盘

**原理**：推理后的帧需存为 JPEG 供 replay 回放。cv2.imwrite 是 CPU 密集型操作（DCT变换+Huffman编码），单线程编码30帧约 160ms。利用 Python 的 ThreadPoolExecutor（GIL 对 cv2 的 C 代码不阻塞），4个线程并行编码，JPEG 编码吞吐量提升 ~4x。

JPEG 质量从 90 调到 75：DCT 量化表更粗糙，编码更快（约快15%），文件更小，肉眼无明显差别。

```python
with ThreadPoolExecutor(max_workers=4) as pool:
    futures = [pool.submit(cv2.imwrite, fp, arr, [cv2.IMWRITE_JPEG_QUALITY, 75])
               for arr, fp in zip(frames_np, frame_paths)]
```

**效果**：JPEG 存盘 160ms → ~30ms（↓81%）。

### ⑥ 流水线重叠：CPU/GPU 并行

**原理**：ffmpeg 解码（CPU 密集）和 YOLO 推理（GPU 密集）原本严格串行。CPU 解码时 GPU 空闲，GPU 推理时 CPU 空闲。改为生产者-消费者模式：

1. 后台线程从 ffmpeg 管道逐帧读取（CPU）
2. 前一半帧就绪后，立即启动 YOLO batch1（GPU）
3. YOLO batch1 在 GPU 运行时，后台线程继续读取后半帧（CPU）
4. 读取完成后，启动 YOLO batch2

```
原流程（串行）：  [ffmpeg 320ms] → [YOLO 150ms] = 470ms
流水线重叠：      [ffmpeg前半 160ms] [ffmpeg后半 160ms]
                          ↓ batch1就绪
                          [YOLO batch1 ~80ms]  [YOLO batch2 ~80ms]
                 总计：160 + max(160,80) + 80 ≈ 400ms
```

代价是分 batch 带来的 GPU kernel launch 开销（每次 ~50ms），以及逐帧 `.copy()` 的内存拷贝开销（不能用零拷贝 reshape，因为数据在后台线程中逐步到达）。

```python
# 后台线程逐帧读取
def _read_frames():
    while True:
        raw = proc.stdout.read(frame_size)
        if len(raw) < frame_size:
            break
        frames_np.append(np.frombuffer(raw, dtype=np.uint8).reshape(H, W, 3).copy())
        if len(frames_np) == mid:
            half_ready.set()  # 通知主线程：前半就绪
    read_done.set()

# 主线程：等前半就绪 → 启动YOLO batch1（线程）→ 等读取完成 → YOLO batch2
half_ready.wait()
yolo_thread = Thread(target=lambda: detect_pets_batch(model, frames_np[:mid], ...))
yolo_thread.start()
read_done.wait()
yolo_thread.join()
batch2 = detect_pets_batch(model, frames_np[mid:], ...)
```

**效果**：497ms → ~460ms（↓7%）。提升幅度受限于 batch 分拆开销和逐帧内存拷贝，但 CPU/GPU 资源利用更充分。

## 最终效果

| 指标 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| 端到端耗时 | 997ms | **~460ms** | ↓54% |
| YOLO推理 | 659ms | **~230ms** | ↓65% |
| ffmpeg解码 | 335ms | ~330ms | 持平 |
| JPEG存盘 | ~160ms | ~30ms | ↓81% |
| 检测事件数 | 1 | 1 | 不变 |

## 瓶颈分析（~460ms 分布）

| 环节 | 耗时 | 占比 | 优化空间 |
|------|------|------|---------|
| ffmpeg H.264 解码 | ~330ms | 72% | CPU软解已近极限，NVDEC反更慢 |
| YOLO 推理 | ~230ms | 25% | 含batch分拆开销，纯推理~150ms |
| JPEG编码+pipeline | ~30ms | 3% | 4线程并行已到位 |

## NVDEC 调研

测试了多种 NVDEC 方案（`-hwaccel cuda`、`-c:v h264_cuvid`、GPU滤镜链），**全部比 CPU 软解慢**。

**原理**：NVDEC 在 GPU 显存中解码 H.264，但后续处理（fps滤镜、rawvideo输出）需要像素数据在 CPU 内存。GPU→CPU 需走 PCIe 传输，720p×30帧 = 83MB 数据。对于短时低分辨率视频，PCIe 传输开销 > CPU 直接解码 H.264 的时间。

NVDEC 适合：4K/高帧率/多路并发/长时间视频。对 720p 30帧短视频不适用。

## 踩坑记录

1. **YOLO 输入格式必须是 BGR**：YOLO 用 OpenCV 加载图像，内部期望 BGR。ffmpeg 输出 RGB 时检测数下降（12→8），改为 BGR24 后回升至 14。这是因为 RGB→BGR 通道顺序错误会改变像素值，导致特征提取异常。

2. **numpy reshape 返回视图是连续内存**：`arr.reshape(N,H,W,3)[i]` 返回的 `(H,W,3)` 视图 `C_CONTIGUOUS=True`，YOLO 可直接消费，无需 `ascontiguousarray` 拷贝。但流水线重叠模式下逐帧读取需要 `.copy()`，因为 pipe 数据是逐步到达的。

3. **imgsz 不宜降低**：imgsz=480 时检测帧从 12 降到 8，且连续性断开导致 dwell 重置，事件无法触发。原因：更小的推理分辨率降低了对小目标的敏感度，导致部分帧漏检。imgsz=640 是保证检测连续性的最低可用值。

4. **流水线重叠的分 batch 开销**：分 2 个 batch 会引入 ~50ms GPU kernel launch 开销 + ~78ms 逐帧 copy 开销，总计 ~128ms 额外开销。overlap 省的是 batch1 的推理时间（~80ms），净收益约 37ms。对于 GPU 推理占比更高的场景（更大模型、更多帧），收益会更明显。

## 相关文件

- `src/pet_home_monitor/services/video_processor.py` — BGR pipe + 并行JPEG + 流水线重叠
- `src/pet_home_monitor/services/pet_detector.py` — FP16半精度
- `src/pet_home_monitor/api/app.py` — CORS中间件

## TensorRT 调研

使用 ultralytics 内置 `model.export(format="engine")` 尝试 TensorRT 优化。

**ONNX 导出成功**：`yolov8s.onnx`（42.8MB），可通过 ONNX Runtime 加载。

**TensorRT engine 创建失败**：
```
[TRT] [E] createInferBuilder: Error Code 6: API Usage Error (CUDA initialization failure with error: 35)
```
原因：WSL2 环境下 TensorRT 调用 `trt.Builder(logger)` 时 CUDA 初始化失败（error code 35）。这是 WSL2 的已知限制，TensorRT 10.16.1.11 与 CUDA 12.8 / driver 573.22 在 WSL2 下存在兼容性问题。`trtexec` 命令行工具在此环境也不可用（仅 pip 安装的 TensorRT，非系统级 deb 包）。

**解决方向**：在原生 Linux 或 Docker 容器（NVIDIA Container Toolkit）中执行 TensorRT 导出，然后将 `.engine` 文件拷贝回 WSL2 推理。

## ONNX Runtime 调研

安装 `onnxruntime-gpu`（含 CUDAExecutionProvider + TensorrtExecutionProvider），对比 ultralytics 原生推理：

| 方案 | 15帧耗时 | 单帧耗时 |
|------|---------|---------|
| Ultralytics batch FP16 | **95.6ms** | 6.4ms |
| ONNX Runtime sequential (batch=1) | 133.6ms | 8.9ms |
| ONNX Runtime IOBinding | — | 8.1ms |

**结论**：ONNX Runtime 在本场景无优势。

- 单帧推理速度相当（~9ms），但 ultralytics 支持 batch 推理（15帧并行），ONNX 模型导出时 batch=1 固定，只能逐帧串行
- batch 并行让 ultralytics 15帧快 40%（95ms vs 133ms）
- 要用 ONNX Runtime batch 推理需要重新导出动态 batch ONNX（`dynamic=True`），但 ultralytics 的 batch FP16 已经够快，不值得额外引入 ONNX Runtime 依赖

## 日期

2026-05-13（初版），2026-05-14（TensorRT/ONNX Runtime 调研补充），2026-05-15（多路并发 + NVDEC HEVC 补充）

---

## 多路并发推理优化（2026-05-15）

### 问题

多路摄像头视频并发上报时，每个视频独立调用 YOLO 推理，GPU 利用率低。两个视频串行处理 976ms，可合并为一次 GPU batch 调用降至 ~515ms。

### InferenceScheduler

新增 `InferenceScheduler` 服务，将多路视频的解码帧合并为一次 GPU batch 调用：

```
Camera A: cv2 decode → submit(frames_a) ─┐
                                           ├→ detect_pets_batch(60帧) → 分发结果
Camera B: cv2 decode → submit(frames_b) ─┘
```

- 线程安全：`threading.Lock` + `threading.Event` + `Future`
- `batch_window_ms=50`：收到首个 submit 后等 50ms 收集更多请求，超时立即 flush
- `max_batch_frames=120`：防 GPU OOM

### DECODE_MAX_DIM 解码降分辨率

ffmpeg/cv2 解码时缩放到 max_dim 以内，减少 CPU 解码和 GPU 推理开销：

```
DECODE_MAX_DIM=640: 1280x720 → 640x360
```

坐标自动缩放回原始分辨率（`scale_x = orig_w / nw`），不影响检测结果。

### 进食判定阈值调整

`min_dwell_seconds` 从 8s 降至 5s，提高进食事件灵敏度。

### 并发 Benchmark（Tesla T4, 8核 CPU）

#### H.264 1280x720（cv2 CPU 解码）

| 场景 | wall时间 | v/s | 300s容量 |
|------|---------|-----|---------|
| 单视频 | ~500ms | — | — |
| N=100 fps=0.5 | 26.7s | 3.7 | **1124路** |
| N=100 fps=1 | 29.3s | 3.4 | 1023路 |

---

## NVDEC HEVC 硬件解码（2026-05-15）

### 问题

HEVC 2304x1296 视频在 Tesla T4 上 CPU 解码耗时 ~1200ms，是瓶颈。H.264 已验证 NVDEC 无优势（GPU→CPU PCIe 传输开销抵消加速），但 HEVC 解码量大，值得尝试。

### 方案：PyNvVideoCodec 零拷贝 GPU 解码

使用 NVIDIA PyNvVideoCodec 库（v2.1.0），NVDEC 硬件解码 HEVC 直接到 GPU 显存：

```
HEVC video → cv2 probe codec → PyNvVideoCodec NVDEC decode (GPU)
  → batch GPU ops (stack + RGB→BGR flip + permute)
  → 一次 GPU→CPU 传输
  → cv2 resize (CPU)
  → detect_pets_batch / InferenceScheduler (现有路径)
H.264 video → cv2 路径（不改动）
```

### 关键设计

1. **按编码自动选择**：cv2 探测 FOURCC，HEVC（`hevc`/`hvc1`/`hev1`/`HEVC`/`H265`/`h265`）走 GPU，H.264 走 cv2
2. **批量 GPU 操作**：30 帧 stack → RGB→BGR flip → permute → **一次** GPU→CPU 传输（252ms），比逐帧传输+转换（960ms）快 3.8x
3. **CPU resize**：先传 uint8 到 CPU（快），再用 cv2.resize（直接处理 uint8，不需要 float 转换）
4. **优雅降级**：PyNvVideoCodec 未安装或解码失败 → 自动 fallback 到 cv2

### 优化过程

| 尝试 | decode 耗时 | 问题 |
|------|-----------|------|
| cv2 软解（baseline） | 1569ms | CPU 瓶颈 |
| `get_batch_frames_by_index()`（随机访问） | 2332ms | HEVC 随机 seek keyframe 很慢 |
| 顺序迭代 + 逐帧 float32 resize | 3332ms | float32 全分辨率 tensor 太大 |
| 顺序迭代 + 逐帧 uint8 传 CPU + cv2 resize | 2660ms | 逐帧 GPU→CPU 传输开销 |
| **顺序迭代 + batch GPU ops + 一次传输** | **761ms** | **最终方案** |

### HEVC Benchmark

| 场景 | decode | infer | total | 300s容量 |
|------|--------|-------|-------|---------|
| HEVC 单视频 GPU decode | 1034ms | 109ms | 1157ms | — |
| HEVC 单视频 cv2（对比） | 1569ms | 110ms | 1779ms | — |
| HEVC N=20 fps=0.5 GPU decode | 2637ms | 619ms | — | **933路** |
| **混合 50H+10E fps=0.5** | H:2527 E:1325 | H:478 E:789 | — | **1164路** |

### 配置

| 参数 | 默认值 | Env Var |
|------|--------|---------|
| gpu_decode_enabled | false | `GPU_DECODE_ENABLED` |
| decode_max_dim | 0 | `DECODE_MAX_DIM` |
| batch_window_ms | 50 | `INFERENCE_BATCH_WINDOW_MS` |
| max_batch_frames | 120 | `INFERENCE_MAX_BATCH_FRAMES` |

Docker 需 `NVIDIA_DRIVER_CAPABILITIES=compute,utility,video` 启用 NVDEC。

### 新增/修改文件

- `src/pet_home_monitor/services/gpu_decoder.py` — NVDEC decode + batch GPU ops
- `src/pet_home_monitor/services/inference_scheduler.py` — 多路 batch 合并推理
- `src/pet_home_monitor/services/video_processor.py` — FOURCC 检测 + HEVC GPU/cv2 自动选择
- `src/pet_home_monitor/config.py` — `gpu_decode_enabled`, `decode_max_dim`, scheduler 配置

### 踩坑

1. **FOURCC 大小写敏感**：cv2 报告的 `hevc`（小写，值 1668703592）与 `HEVC`（大写，值 1129727304）是不同的 fourcc int，必须都加入匹配集合
2. **SimpleDecoder 顺序比随机快**：`get_batch_frames_by_index()` 做 random access seek，HEVC keyframe 间隔大导致 seek 开销巨大。顺序迭代全 450 帧选 30 帧（611ms）比随机选 30 帧（539ms + seek 开销）更快
3. **float32 全分辨率 resize 慢**：2304×1296 uint8 转 float32 = 36MB/帧，F.interpolate 在这样大的 float tensor 上很慢。改为 uint8 传 CPU 再 cv2.resize
4. **batch GPU→CPU 传输**：30 帧 stack + flip + permute → 一次传输（252ms），比 30 次单独传输+numpy 转换（960ms）快 3.8x
