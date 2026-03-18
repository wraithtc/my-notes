# 宠语翻译 & 宠物前世今生 & 宠物AI表情包 技术调研

## 背景

1. **宠语翻译**：将猫狗的叫声转译成文字，让用户能理解宠物的意图，同时能将人类的语言转换成宠物的叫声，从而让用户可以跟宠物沟通
2. **宠物前世今生**：根据用户上传的宠物照片，生成宠物前世的人物照片，更多是情绪价值提供，没有科学依据
3. **宠物AI表情包**：根据用户上传的宠物照片，生成宠物专属表情包

---

## 一、宠语翻译技术（纯声音识别）

> **核心定位**：双向翻译 - 宠物叫声→文字 + 文字/人声→宠物叫声

### 1.1 技术架构

```
┌─────────────────────────────────────────────────────────────┐
│                      宠语翻译系统                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   宠物叫声 ──→ 声学特征提取 ──→ 音频大模型 ──→ 文字输出     │
│      ↑                                          │          │
│      │                                          ↓          │
│      └──── 宠物声音合成 ←── 声音克隆/VC ←── 文字/人声输入   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**两个核心方向**：
1. **宠物→人类**：宠物叫声识别 → 情绪/意图分析 → 自然语言输出
2. **人类→宠物**：文字/人声输入 → 声音合成/转换 → 宠物风格叫声

---

### 1.2 宠物叫声识别（宠物→人类）

#### 技术流程

```
宠物声音采集 → 声学特征提取 → 音频大模型/分类模型 → 情绪/意图识别 → 自然语言输出
```

#### 核心声学特征
| 特征类型 | 说明 |
|---------|------|
| **基频 (F0)** | 声音的基本频率，反映音高 |
| **抖动 (Jitter)** | 频率周期变化，反映声音稳定性 |
| **闪烁 (Shimmer)** | 振幅变化，反映声音能量波动 |
| **MFCC** | 梅尔频率倒谱系数，核心语音特征 |
| **频谱质心** | 声音"亮度"指标 |
| **Mel频谱** | 模拟人耳感知的频谱表示 |

### 1.2 国内开源音频大模型

#### 🔥 推荐模型

| 模型 | 团队 | 特点 | GitHub |
|-----|------|------|--------|
| **Qwen2-Audio** | 阿里通义 | 支持语音/音乐/自然声音（含动物声音），开源可商用 | [QwenLM/Qwen2-Audio](https://github.com/QwenLM/Qwen2-Audio) |
| **SenseVoice** | 阿里 | 多语言语音识别，情感识别能力强 | [FunAudioLLM/SenseVoice](https://github.com/FunAudioLLM/SenseVoice) |
| **FunASR** | 阿里达摩院 | 语音识别工具链，支持自定义训练 | [modelscope/FunASR](https://github.com/modelscope/FunASR) |
| **Whisper** | OpenAI | 开源语音识别，可微调用于动物声音 | [openai/whisper](https://github.com/openai/whisper) |
| **Audio Flamingo** | - | 开源音频大语言模型，多任务能力强 | [lahaudio/Audio-Flamingo](https://github.com/lahaudio/Audio-Flamingo) |

#### Qwen2-Audio 详情

```
GitHub: https://github.com/QwenLM/Qwen2-Audio
HuggingFace: https://huggingface.co/Qwen/Qwen2-Audio-7B-Instruct

能力：
├── 语音识别与理解
├── 音乐分析
├── 自然声音识别（包含动物叫声）
└── 音频问答（输入音频 → 输出文本描述）

模型规模：
├── Qwen2-Audio-7B-Instruct（推荐）
└── 支持 vLLM 加速推理

部署方式：
├── transformers 直接加载
├── vLLM 高性能推理
└── ModelScope 国内镜像
```

#### SenseVoice 详情

```
GitHub: https://github.com/FunAudioLLM/SenseVoice
特点：
├── 50+ 语言语音识别
├── 情感识别（高兴、悲伤、愤怒等）
├── 音频事件检测
└── 极快推理速度（比Whisper快15倍）

适用场景：
└── 宠物声音情感分析
```

### 1.3 专用生物声学模型

| 模型 | 特点 |
|-----|------|
| **Perch 2.0** (Google DeepMind) | 生物声学分类SOTA，专为物种识别设计 |
| **Audio-Language Model for Bioacoustics** | 首个生物声学专用音频-语言模型 |
| **密歇根大学狗声解码模型** | 基于人类语音预训练，跨物种迁移学习 |

### 1.4 技术实现方案

#### 方案一：音频大模型（推荐）

```python
# 使用 Qwen2-Audio 进行宠物声音理解
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2-Audio-7B-Instruct")
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2-Audio-7B-Instruct")

# 输入：宠物叫声音频
# 输出：情绪/意图的自然语言描述
```

**优势**：
- 端到端处理，直接输出自然语言
- 支持多种音频格式
- 可进行音频问答交互

#### 方案二：传统分类模型 + 规则引擎

```
音频 → MFCC/Mel特征提取 → CNN/ResNet分类 → 情绪类别 → 规则映射 → 文案生成
```

**技术栈**：
- **特征提取**: librosa、torchaudio
- **分类模型**: Audio Spectrogram Transformer (AST)、PANNs
- **情绪类别**: 快乐、恐惧、疑惑、饥饿、愤怒、焦虑等

**优势**：
- 部署轻量，可端侧运行
- 响应速度快
- 可控性强

#### 方案三：微调开源模型

```
基础模型 (Whisper/Qwen-Audio) + 宠物声音数据集 → 微调 → 专用宠物翻译模型
```

**数据需求**：
- 宠物叫声音频 + 情绪/意图标签
- 建议每个类别 500+ 样本

### 1.5 代表产品

| 产品 | 特点 | 准确率 |
|-----|------|-------|
| **Traini** | 全球首个宠物翻译器，人狗双向语音对话 | 81.5% |
| **宠智灵科技** | 识别10+种叫声类型（咆哮、呻吟、吠叫、呜咽等） | - |
| **密歇根大学研究** | 识别狗的年龄、性别、品种 | - |

### 1.6 技术局限性

- 当前技术更多是**声学模式匹配**而非真正的语义翻译
- 翻译结果反映**情绪状态**（快乐/恐惧/饥饿等）而非具体含义
- 不同宠物个体差异大，通用性有限
- 需要大量**标注数据**进行模型训练/微调

### 1.7 参考资源

- [Qwen2-Audio GitHub](https://github.com/QwenLM/Qwen2-Audio)
- [FunASR GitHub](https://github.com/modelscope/FunASR)
- [密歇根大学狗声解码研究](https://developer.aliyun.com/article/1540710)
- [Wild Animal Initiative - AI Animal Translation](https://www.wildanimalinitiative.org/blog/ai-animal-translation)
- [Nature - AI Animal Communication](https://www.nature.com/articles/d41586-025-02917-9)

---

## 二、宠物前世今生技术

### 2.1 技术原理

核心是将宠物照片转换为"人类"形象，保留宠物的特征（眼睛颜色、表情、毛发纹理等），营造出"前世是人类"的创意效果。

**本质是 Pet-to-Human AI 生成技术**。

### 2.2 国内开源多模态视觉模型

#### 🔥 推荐模型

| 模型 | 团队 | 特点 | GitHub |
|-----|------|------|--------|
| **Qwen2.5-VL** | 阿里通义 | 图像+视频理解，部分基准超越GPT-4o | [QwenLM/Qwen2-VL](https://github.com/QwenLM/Qwen2-VL) |
| **InternVL 3.5** | 上海AI实验室 | CVPR 2024 Oral，推理能力强 | [OpenGVLab/InternVL](https://github.com/OpenGVLab/InternVL) |
| **DeepSeek-VL2** | 深度求索 | 推理效率高，MIT许可证完全开源 | [deepseek-ai/DeepSeek-VL2](https://github.com/deepseek-ai/DeepSeek-VL2) |
| **CogVLM2** | 智谱AI | 视觉理解+对话，支持多模态 | [THUDM/CogVLM2](https://github.com/THUDM/CogVLM2) |

#### Qwen2.5-VL 详情

```
GitHub: https://github.com/QwenLM/Qwen2-VL
HuggingFace: https://huggingface.co/Qwen/Qwen2.5-VL-7B-Instruct

模型规模：
├── Qwen2.5-VL-2B（轻量级）
├── Qwen2.5-VL-7B（推荐）
└── Qwen2.5-VL-72B（旗舰）

能力：
├── 图像理解与描述
├── 长视频理解
├── 任意分辨率图像处理
├── OCR 文字识别
└── 多模态推理

部署：
├── transformers
├── vLLM
└── mlx-lm（Apple Silicon）
```

#### InternVL 3.5 详情

```
GitHub: https://github.com/OpenGVLab/InternVL
HuggingFace: https://huggingface.co/OpenGVLab/InternVL

模型规模：
├── InternVL3-1B
├── InternVL3-8B
└── InternVL3-78B

特点：
├── CVPR 2024 Oral 论文
├── 原生多模态训练
├── 强推理能力
└── InternLM-XComposer 支持流式视频+音频
```

#### DeepSeek-VL2 详情

```
GitHub: https://github.com/deepseek-ai/DeepSeek-VL2

模型规模：
├── DeepSeek-VL2-Tiny
├── DeepSeek-VL2-Small
└── DeepSeek-VL2

特点：
├── MoE 架构，推理效率高
├── MIT 许可证，完全开源
├── 更少训练数据达到相同效果
└── 适合资源受限场景
```

### 2.3 技术实现方案

```
宠物照片上传 → 特征提取 → 风格迁移 → 人像生成 → 后处理优化
```

**关键技术**：

| 技术模块 | 实现方式 |
|---------|---------|
| **特征提取** | 提取眼睛颜色、表情、毛发纹理等特征 |
| **风格迁移** | Stable Diffusion + LoRA/ControlNet |
| **人像生成** | 基于文本引导的图像生成 |
| **特征保留** | IP-Adapter 保持宠物特征映射到人像 |

### 2.4 开源图像生成工具

| 工具 | 说明 | GitHub |
|-----|------|--------|
| **Stable Diffusion WebUI** | 最流行的SD图形界面 | [AUTOMATIC1111/stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui) |
| **ComfyUI** | 节点式工作流，灵活强大 | [comfyanonymous/ComfyUI](https://github.com/comfyanonymous/ComfyUI) |
| **Fooocus** | 简化版SD，易于使用 | [lllyasviel/Fooocus](https://github.com/lllyasviel/Fooocus) |
| **IP-Adapter** | 图像特征保持适配器 | [tencent-ailab/IP-Adapter](https://github.com/tencent-ailab/IP-Adapter) |
| **InstantID** | 零样本身份保持生成 | [instantX-research/InstantID](https://github.com/instantX-research/InstantID) |
| **FaceID Plus V2** | 高质量换脸模型 | [Civitai](https://civitai.com/models) |

### 2.5 推荐技术栈

**方案一：基于 Stable Diffusion + ComfyUI**（推荐）

```
ComfyUI 工作流：
├── 加载宠物图片
├── Qwen2-VL 分析宠物特征 → 生成描述Prompt
├── IP-Adapter FaceID 保持特征
├── ControlNet 控制姿态
├── LoRA 风格微调（古风/现代/动漫等）
└── Stable Diffusion 生成人像
```

**方案二：基于 GPT-4V + DALL-E 3**

```
GPT-4V 分析宠物特征 → 生成描述性Prompt → DALL-E 3 生成人像
```

**方案三：轻量级方案**

```
Fooocus + IP-Adapter → 一键生成
```

### 2.6 代表商业工具

| 工具 | 特点 | 链接 |
|-----|------|-----|
| **EaseMate AI** | 使用ChatGPT-4o和Gemini转换 | [easemate.ai](https://www.easemate.ai/pet-to-human) |
| **Kaze.ai** | 保留毛色、眼睛颜色、表情特征 | [kaze.ai](https://kaze.ai/ai-pet-to-human) |
| **Fotor** | 一键生成，支持多种宠物 | [fotor.com](https://www.fotor.com/features/pet-to-human/) |
| **insMind** | 即时生成人像版本 | [insmind.com](https://www.insmind.com/ai-pet-to-human-generator/) |

### 2.7 参考资源

- [Qwen2-VL GitHub](https://github.com/QwenLM/Qwen2-VL)
- [InternVL GitHub](https://github.com/OpenGVLab/InternVL)
- [DeepSeek-VL2 GitHub](https://github.com/deepseek-ai/DeepSeek-VL2)
- [ComfyUI GitHub](https://github.com/comfyanonymous/ComfyUI)
- [IP-Adapter GitHub](https://github.com/tencent-ailab/IP-Adapter)
- [Stable Diffusion 换脸教程](https://www.bilibili.com/video/BV1nZDBYLEnL/)
- [Pet-to-Human.com](https://pet-to-human.com/)

---

## 三、宠物AI表情包技术

### 3.1 技术原理

宠物表情包生成涉及三个核心技术：
1. **换脸技术**：将宠物脸部替换到表情包模板
2. **表情迁移**：让静态宠物照片动起来
3. **风格化生成**：将宠物照片转为卡通/手绘风格

### 3.2 开源表情迁移模型

#### 🔥 LivePortrait（快手可灵）⭐ 强烈推荐

```
GitHub: https://github.com/KwaiVGI/LivePortrait
Star: 6.5K+
团队: 快手可灵大模型团队

核心功能：
├── 表情迁移：驱动视频表情 → 目标图片
├── 姿态迁移：头部姿态精准控制
├── 静态转动态：一张照片生成动态视频
├── 多人物支持：多人面部表情驱动
├── 动物模式 (V2)：2024年8月更新，支持动物表情迁移 ⭐
└── 对口型：唇形同步功能

技术特点：
├── 69M 高质量训练帧
├── 混合训练策略
├── 升级网络结构
└── 强泛化性、可控性

一键启动：
├── https://github.com/aidayang/LivePortrait-OneClick
└── 整合包，无需配置环境
```

#### 其他开源表情迁移模型

| 模型 | 说明 | GitHub |
|-----|------|--------|
| **SadTalker** | 音频驱动人脸动画 | [OpenTalker/SadTalker](https://github.com/OpenTalker/SadTalker) |
| **Wav2Lip** | 音频驱动唇形同步 | [Rudrabha/Wav2Lip](https://github.com/Rudrabha/Wav2Lip) |
| **First Order Motion** | 单张图片生成动画 | [AliaksandrSiarohin/first-order-model](https://github.com/AliaksandrSiarohin/first-order-model) |
| **Thin-Plate Spline** | 运动估计 | [amin-berjawi/TPS](https://github.com/amon-berjawi/tps) |

### 3.3 开源换脸模型

| 模型 | 说明 | GitHub |
|-----|------|--------|
| **InsightFace** | 人脸识别与换脸工具链 | [deepinsight/insightface](https://github.com/deepinsight/insightface) |
| **FaceShifter** | 高质量换脸 | [taotaonice/FaceShifter](https://github.com/taotaonice/FaceShifter) |
| **Ghost** | 高保真换脸 | [ai-hypercomputer/Ghost](https://github.com/ai-hypercomputer/Ghost) |
| **SimSwap** | 高效换脸 | [neuralchen/SimSwap](https://github.com/neuralchen/SimSwap) |
| **Roop** | 一键换脸工具 | [s0md3v/roop](https://github.com/s0md3v/roop) |

### 3.4 开源卡通化/风格化模型

| 模型 | 说明 | GitHub |
|-----|------|--------|
| **White-box Cartoonization** | 高质量卡通化 | [SystemErrorWang/White-box-Cartoonization](https://github.com/SystemErrorWang/White-box-Cartoonization) |
| **AnimeGAN** | 动漫风格迁移 | [TachibanaYoshino/AnimeGANv2](https://github.com/TachibanaYoshino/AnimeGANv2) |
| **CartoonGAN** | 卡通风格生成 | [Yijunmaverick/CartoonGAN-Test-Pytorch-Torch](https://github.com/Yijunmaverick/CartoonGAN-Test-Pytorch-Torch) |
| **U-GAT-IT** | 自适应风格迁移 | [znxlwm/UGATIT-pytorch](https://github.com/znxlwm/UGATIT-pytorch) |

### 3.5 技术实现方案

#### 方案一：换脸生成表情包

```
宠物照片 → 人脸/宠物脸检测 → 特征点提取 → 表情包模板匹配 → 融合生成
```

**技术栈**：
- **人脸检测**: OpenCV、dlib、MediaPipe
- **特征点定位**: 68/468点人脸关键点检测
- **换脸算法**: InsightFace、Ghost、FaceShifter
- **融合优化**: Poisson Blending、色彩校正

#### 方案二：表情迁移（让宠物说话/动起来）⭐ 推荐

```
静态宠物照片 → LivePortrait → 驱动视频/音频 → 动态表情包
```

**LivePortrait 使用示例**：
```python
# LivePortrait 推理
python inference.py \
    --source_image pet_photo.jpg \
    --driving_video template.mp4 \
    --output_dir ./output

# 动物模式（V2新增）
python inference.py \
    --source_image cat.jpg \
    --driving_video expression.mp4 \
    --animal_mode
```

#### 方案三：风格化卡通生成

```
宠物照片 → 图像分割 → 风格迁移 → 卡通化 → 表情包合成
```

**技术栈**：
- **Stable Diffusion + ControlNet**: 保持轮廓的风格化
- **CartoonGAN / White-box Cartoonization**: 卡通化
- **Line Drawing**: 线稿提取

### 3.6 完整实现流程

```python
# 宠物表情包生成器示例
from liveportrait import LivePortrait
from insightface.app import FaceAnalysis

class PetMemeGenerator:
    def __init__(self):
        # 表情迁移
        self.live_portrait = LivePortrait(
            model_path="KwaiVGI/LivePortrait",
            animal_mode=True  # V2 动物模式
        )
        # 换脸
        self.face_analyzer = FaceAnalysis(name="buffalo_l")
        # 风格化（可选）
        self.cartoonizer = WhiteBoxCartoonization()

    def generate_animated_meme(self, pet_photo, driving_video):
        """生成动态表情包"""
        return self.live_portrait.generate(
            source=pet_photo,
            driving=driving_video
        )

    def generate_face_swap(self, pet_photo, template):
        """生成换脸表情包"""
        # 检测脸部
        faces = self.face_analyzer.get(pet_photo)
        # 换脸融合
        return self.face_swapper.swap(pet_photo, template, faces[0])

    def generate_cartoon(self, pet_photo):
        """生成卡通风格表情包"""
        return self.cartoonizer.cartoonize(pet_photo)
```

### 3.7 表情迁移技术前沿

**网易伏羲技术方案**（CVPR 2024 & ECCV 2024 双料冠军）：
- 高精度表情迁移
- 已应用于NPC等场景
- SIGGRAPH Asia 发表

### 3.8 参考资源

**开源项目**：
- [LivePortrait GitHub](https://github.com/KwaiVGI/LivePortrait)
- [LivePortrait 一键启动包](https://github.com/aidayang/LivePortrait-OneClick)
- [InsightFace GitHub](https://github.com/deepinsight/insightface)
- [SadTalker GitHub](https://github.com/OpenTalker/SadTalker)
- [White-box Cartoonization](https://github.com/SystemErrorWang/White-box-Cartoonization)

**教程**：
- [LivePortrait 全功能升级指南](https://blog.csdn.net/gitblog_00628/article/details/151472155)
- [快手开源LivePortrait详解](https://juejin.cn/post/7392143759098413108)
- [LivePortrait V2 动物模式](https://cloud.tencent.com/developer/article/2442012)

**商业工具**：
- [AI Face Swap](https://aifaceswap.io/ai-pet-talking/)
- [Live3D AI Talking Photo](https://live3d.io/ai-talking-photo)

---

## 四、技术选型建议

### 4.1 宠语翻译（纯声音）

| 方案 | 优点 | 缺点 | 适用场景 |
|-----|------|------|---------|
| **Qwen2-Audio** | 开源可商用、端到端输出自然语言 | 模型较大(7B)、需GPU | ⭐ 推荐首选 |
| **SenseVoice** | 情感识别强、推理快 | 需结合其他模型生成文案 | 情感分析场景 |
| **分类模型+规则** | 轻量、响应快、可端侧部署 | 需自建数据和规则 | 资源受限场景 |
| **微调Whisper/Qwen** | 效果可定制 | 需标注数据、训练成本 | 追求效果优化 |
| **第三方API** | 快速上线 | 成本高、不可控 | MVP验证 |

**推荐路径**:
```
MVP阶段：Qwen2-Audio-7B-Instruct（开箱即用）
    ↓
优化阶段：收集宠物声音数据 → 微调 Qwen2-Audio
    ↓
轻量化：蒸馏/量化模型 → 端侧部署
```

**开源资源汇总**：
| 资源 | 链接 |
|-----|------|
| Qwen2-Audio | [github.com/QwenLM/Qwen2-Audio](https://github.com/QwenLM/Qwen2-Audio) |
| SenseVoice | [github.com/FunAudioLLM/SenseVoice](https://github.com/FunAudioLLM/SenseVoice) |
| FunASR | [github.com/modelscope/FunASR](https://github.com/modelscope/FunASR) |
| Whisper | [github.com/openai/whisper](https://github.com/openai/whisper) |

### 4.2 宠物前世今生

| 方案 | 优点 | 缺点 | 适用场景 |
|-----|------|------|---------|
| **Stable Diffusion + ComfyUI** | 开源可控、成本低 | 需部署GPU服务器 | ⭐ 大规模商用 |
| **Qwen2.5-VL + SD** | 国产化、效果好 | 需多模型组合 | 中期方案 |
| **GPT-4V + DALL-E 3** | 效果好、简单 | API成本高 | 快速验证 |
| **第三方API** | 最简单 | 依赖第三方、成本不可控 | 初期测试 |

**推荐路径**:
```
MVP阶段：第三方API（Fotor/insMind）验证需求
    ↓
中期阶段：ComfyUI + IP-Adapter + InstantID
    ↓
规模化：自建 SD 服务 + Qwen2.5-VL 特征分析
```

**开源资源汇总**：
| 资源 | 链接 |
|-----|------|
| ComfyUI | [github.com/comfyanonymous/ComfyUI](https://github.com/comfyanonymous/ComfyUI) |
| Stable Diffusion WebUI | [github.com/AUTOMATIC1111/stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui) |
| IP-Adapter | [github.com/tencent-ailab/IP-Adapter](https://github.com/tencent-ailab/IP-Adapter) |
| InstantID | [github.com/instantX-research/InstantID](https://github.com/instantX-research/InstantID) |
| Qwen2.5-VL | [github.com/QwenLM/Qwen2-VL](https://github.com/QwenLM/Qwen2-VL) |
| InternVL | [github.com/OpenGVLab/InternVL](https://github.com/OpenGVLab/InternVL) |
| DeepSeek-VL2 | [github.com/deepseek-ai/DeepSeek-VL2](https://github.com/deepseek-ai/DeepSeek-VL2) |

### 4.3 宠物AI表情包

| 方案 | 优点 | 缺点 | 适用场景 |
|-----|------|------|---------|
| **LivePortrait** | 支持动物、效果自然、开源 | 需GPU | ⭐ 动态表情包首选 |
| **InsightFace 换脸** | 成熟稳定、效果好 | 模板有限 | 换脸表情包 |
| **卡通化模型** | 创意空间大 | 可能失真 | 静态表情包 |
| **第三方API** | 快速上线 | 成本高 | MVP验证 |

**推荐路径**:
```
MVP阶段：LivePortrait 一键启动包（支持动物模式）
    ↓
扩展阶段：InsightFace 换脸 + 多模板库
    ↓
差异化：卡通化风格 + 自定义文案
```

**开源资源汇总**：
| 资源 | 链接 |
|-----|------|
| LivePortrait | [github.com/KwaiVGI/LivePortrait](https://github.com/KwaiVGI/LivePortrait) |
| LivePortrait 一键包 | [github.com/aidayang/LivePortrait-OneClick](https://github.com/aidayang/LivePortrait-OneClick) |
| InsightFace | [github.com/deepinsight/insightface](https://github.com/deepinsight/insightface) |
| SadTalker | [github.com/OpenTalker/SadTalker](https://github.com/OpenTalker/SadTalker) |
| Wav2Lip | [github.com/Rudrabha/Wav2Lip](https://github.com/Rudrabha/Wav2Lip) |
| Roop | [github.com/s0md3v/roop](https://github.com/s0md3v/roop) |
| White-box Cartoonization | [github.com/SystemErrorWang/White-box-Cartoonization](https://github.com/SystemErrorWang/White-box-Cartoonization) |
| AnimeGAN | [github.com/TachibanaYoshino/AnimeGANv2](https://github.com/TachibanaYoshino/AnimeGANv2) |

---

## 五、市场参考

- **Traini**: 宠物翻译器，iOS下载量超75万，获千万级融资
- **"猫狗翻译官"赛道**: 整体融资超5千万
- **谷歌 DolphinGemma**: 海豚语言翻译，实现水下实时交流

---

## 更新日志

- 2026-03-18: 初始技术调研完成
- 2026-03-18: 补充开源模型详细信息（Qwen2-Audio、Qwen2.5-VL、InternVL、DeepSeek-VL2、LivePortrait等）
- 2026-03-18: 宠语翻译调整为纯声音识别方案
