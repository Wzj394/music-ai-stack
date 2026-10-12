# 05 · 音频转 MIDI

> 把声音变成音符。**对做音乐的人来说，这可能是整个音乐 AI 里最实用的一类** —— 因为 MIDI 你能改，波形你改不动。

---

## 这类任务的难点在哪

转 MIDI 听起来简单（有音高就记下来），实际很难，原因是：

| 难点 | 说明 |
|---|---|
| **复音** | 一个瞬间有多个音同时响，要全部分出来 |
| **音色干扰** | 钢琴、吉他、人声、鼓，算法差别很大 |
| **混响** | 前一个音还没衰减完，下一个音开始了，音高重叠 |
| **颤音和滑音** | 一个音内部音高在变，算一个音还是多个 |
| **节拍对齐** | 检测出时间点了，还要量化到网格上 |
| **演奏表情** | 力度、踏板、连奏，MIDI 里怎么表达 |

**所以**：没有"完美"的转写工具，只有"够用"的。**期望值要放对。**

---

## 工具表

> 状态标记见 [00-怎么用这个仓库.md](00-怎么用这个仓库.md)。**最后核实：2026-10**

### 复音转写（音频 → MIDI）

| 名称 | 用途 | 语言/框架 | 链接 | 状态 | 备注 |
|---|---|---|---|---|---|
| **Basic Pitch** | 通用复音转写，含弯音检测 | Python / TF | [spotify/basic-pitch](https://github.com/spotify/basic-pitch) | 🟢 | **新手首选**：轻量、安装简单、效果稳定 |
| **MT3** | 多任务转写（多乐器） | Python / JAX | [magenta/mt3](https://github.com/magenta/mt3) | 🔴 | Magenta 归档，但思路和权重仍有参考价值 |
| **Onsets and Frames** | 钢琴转写经典方案 | Python / TF | [magenta/onsets-frames](https://github.com/magenta/onsets-frames) | 🔴 | 同上，钢琴专用 |
| **Omnizart** | 多任务音乐转写工具箱 | Python | [MCTY/omnizart](https://github.com/MCTY/omnizart) | 🟡 | 涵盖人声、鼓、钢琴、和弦 |
| **ByteDance Piano Transcription** | 高精度钢琴转写 | Python | 搜索最新仓库 | ⚪ | 钢琴专项，效果指标领先 |

### 单音高检测（Pitch Tracking）

| 名称 | 用途 | 语言/框架 | 链接 | 状态 | 备注 |
|---|---|---|---|---|---|
| **CREPE** | 深度学习音高跟踪 | Python / TF | [marl/crepe](https://github.com/marl/crepe) | 🟡 | 单音、人声、旋律乐器效果好 |
| **torchcrepe** | CREPE 的 PyTorch 重实现 | Python | [maxrmorrison/torchcrepe](https://github.com/maxrmorrison/torchcrepe) | 🟢 | 更容易集成进现代流程 |
| **RMVPE** | 高精度音高提取 | Python | 集成在 RVC 生态里 | 🟢 | 本来是给声音转换用的，做音高分析也很准 |
| **librosa.pyin** | 传统方法音高估计 | Python | [librosa/librosa](https://github.com/librosa/librosa) | 🟢 | 不用装模型，几行代码，够快 |

### 鼓点 / 节奏转写

| 名称 | 用途 | 链接 | 状态 |
|---|---|---|---|
| **Omnizart** | 鼓点转写 | [MCTY/omnizart](https://github.com/MCTY/omnizart) | 🟡 |
| **librosa onset 检测** | 起点检测，再做分类 | [librosa/librosa](https://github.com/librosa/librosa) | 🟢 |
| **ADTOF** | 自动鼓转写 | [MZehren/ADTOF](https://github.com/MZehren/ADTOF) | 🟡 |

### 编码 / 解码（转出来之后怎么用）

| 名称 | 用途 | 链接 | 状态 |
|---|---|---|---|
| **mido** | MIDI 消息层读写 | [mido/mido](https://github.com/mido/mido) | 🟢 |
| **pretty_midi** | MIDI 高层分析处理 | [craffel/pretty-midi](https://github.com/craffel/pretty-midi) | 🟢 |
| **MIDIUtil** | 从零生成 MIDI | [MarkCWirt/MIDIUtil](https://github.com/MarkCWirt/MIDIUtil) | 🟢 |

> 这三个你机器上都装了，用法见 [06-DAW集成与自动化.md](06-DAW集成与自动化.md)

---

## 实际情况

### 新手就从 Basic Pitch 开始

理由很实际：

- **装得动** —— 不像大模型那样要几十 GB
- **跑得快** —— CPU 也能忍
- **有官方命令行和 Python API** —— 不用自己写一堆胶水
- **输出带弯音** —— 比只给音高的工具实用

```python
from basic_pitch.inference import predict_and_save
from basic_pitch import ICASSP_2022_MODEL_PATH

predict_and_save(
    ["input.wav"],
    "output_dir",
    save_midi=True,
    sonify_midi=False,
    save_model_outputs=False,
    save_notes=False,
    model_or_model_path=ICASSP_2022_MODEL_PATH,
)
```

> ⚠️ API 可能随版本变化，以官方 README 为准。

### 一个关键技巧：先分离，再转写

直接拿混音去转 MIDI，效果通常很差。**先分离出你要的那个声部，再转写**：

```
混音 → 分离（Demucs / UVR5）→ 取出人声或贝斯 → 转 MIDI → 导入 DAW
```

这一步能显著提升质量，代价是多一道工序。

### 关于"量化"

模型输出的是**时间点**，不是**音符网格**。你想要的通常是后者。

```
原始检测：音符起点在 1.237 秒、1.481 秒、1.729 秒……
你想要： 1/8 音符网格上
```

量化会损失人性化演奏的细微差别，但让结果可用。推荐用 DAW 的量化功能，而不是自己写。

**注意**：量化前**先确认 BPM 检测是对的**，否则全盘皆错。

### 对做音乐的人，这条路的价值

| 场景 | 价值 |
|---|---|
| **扒歌** | 听不出和弦/旋律？转出来看谱 |
| **采样** | 把别人的 loop 转成 MIDI，换成自己的音色 |
| **修演奏** | 弹错的地方，转 MIDI 后手工改 |
| **教学** | 分析大师演奏的音符选择 |
| **老录音抢救** | 磁带/录像里的演奏转成可编辑素材 |

---

## 新手路线

```
第 1 步：装 Basic Pitch，转一段钢琴独奏
         → 建立"能跑通"的信心

第 2 步：导入 DAW，听一下
         → 你会立刻发现"能听"和"能用"的差距

第 3 步：加分离这一步
         → 混音 → 分离 → 转写，看质量提升多少

第 4 步：量化 + 手工修
         → 这一步省不掉。机器给的是毛坯

第 5 步：写个脚本把这几步串起来
         → 见 06
```

---

## 坑清单

| 坑 | 表现 | 说明 |
|---|---|---|
| **BPM 没对上** | 量化后全乱 | 先测 BPM，人工确认，再量化 |
| **复音过载** | 输出一堆碎音符 | 和弦复杂时模型会"猜"，人手修不可避免 |
| **踏板丢失** | 钢琴听起来很干 | 多数模型不输出踏板信息 |
| **时间戳精度** | 音符位置差几个 tick | 正常现象，靠量化解决 |
| **音色假设** | 吉他当钢琴转 | 模型的训练数据决定它"想当然"的方向 |
| **混响干扰** | 长尾音被切成多个音 | 预处理做去混响有帮助 |
| **采样率不对** | 音高整体偏移 | 见 [01-音频基础.md](01-音频基础.md) |

---

## 期望值管理

**说句实话**：

> 现在的工具能把 80% 的音符抓对，但要让结果**真正能用**，
> 你还得手工修剩下的 20% —— 而且这 20% 往往是最花时间的部分。

**什么时候值得用**：

- 素材量大，手工扒不现实
- 只需要个大概，后面要重新编曲
- 作为参考，不是最终产物

**什么时候不值得**：

- 就几个和弦，自己听更快
- 要求 100% 准确
- 素材本身质量差（现场、低码率、混响重）

---

## 下一步

- 分离人声/乐器 → [02-音频理解.md](02-音频理解.md)
- 接进 DAW 流程 → [06-DAW集成与自动化.md](06-DAW集成与自动化.md)
- 音频格式基础 → [01-音频基础.md](01-音频基础.md)
