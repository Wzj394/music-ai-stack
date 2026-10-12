# 06 · DAW 集成与自动化

> 这篇离你最近——你本来就在 REAPER 里干活。
> 音乐 AI 最尴尬的一点是：**模型很聪明，但它住在命令行里，你的作品住在 DAW 里。** 这一篇讲怎么把这最后一公里打通。

---

## 一、为什么这件事值得做

模型输出一个 `output.wav` 或 `output.mid`，然后呢？

手动导入、对齐、切片、命名、导出……做十次你就烦了。**而这一整套流程，脚本三十行就能干完。**

真正的价值不在"跑一次模型"，在于：

- 批量处理一整个文件夹的素材
- 按时间码精确落位，不用手动对齐
- 分析结果（节拍、调性、音高）直接写回工程
- 一条命令跑完全流程：分离 → 转 MIDI → 生成 → 混音

---

## 二、REAPER 为什么是自动化的最佳宿主

这不是信仰问题，是事实：

| 特性 | REAPER | 多数其他 DAW |
|---|---|---|
| 脚本语言 | **Lua / EEL2 / Python 三种** | 通常只有一种，或没有 |
| API 覆盖面 | 几乎能操作一切：轨道、片段、MIDI、渲染、FX | 有限 |
| 脚本分发 | 一个 `.lua` 文件，扔进 Scripts 目录 | 要编译插件 |
| 命令行渲染 | 支持（`reaper -renderproject`） | 部分支持 |
| 便携安装 | 支持，整个 DAW 装 U 盘里 | 少见 |

**结论**：如果你的目标是"音乐 + AI + 自动化"，REAPER 不是妥协，是最优解。

---

## 三、ReaScript 三种语言怎么选

| 语言 | 适合 | 不适合 |
|---|---|---|
| **Lua** | 首选。生态最大，ReaPack 上 90% 的脚本是 Lua，启动快，跨平台 | 没有 pip 生态 |
| **EEL2** | 极致性能的 DSP 循环 | 语法古怪，写业务逻辑很痛苦 |
| **Python** | 需要 numpy / torch / librosa 的时候 | 启动慢，REAPER 内置的 Python 版本可能和你系统的不一致 |

**给你的建议**：

```
纯 REAPER 内部操作（移动片段、改 MIDI、批量命名） → Lua
要调 AI 模型、要 numpy                           → Python
```

你机器上装了 `luaparser`——如果你在写工具去**解析别人的 Lua 脚本**，那是另一回事。

### Python 版本这个坑要注意

REAPER 调用 Python 时，用的是它自己找到的解释器。你机器上有 4 个 Python，很容易对不上：

> **在 REAPER 里跑 `reaper.ShowConsoleMsg(sys.version)`，先确认它用的是哪个。**

否则你会遇到"我明明装了 librosa，脚本里却 import 不到"。

---

## 四、ReaScript 的核心概念

不用背 API，理解这几个对象的关系就够了：

```
Project（工程）
 └── Track（轨道）
      └── Item（片段，音频或 MIDI）
           └── Take（片段里的一个 take）
                └── MIDI 数据 / 音频源
```

**几乎所有脚本都是在这棵树上爬。**

几个一定会用到的操作类型：

- 遍历：拿到选中的轨道 / 片段
- 读：取位置、长度、名称、MIDI 音符
- 写：改位置、改名、插音符、加 FX
- 渲染：把结果导出成文件

> API 大全在 REAPER 安装目录的 `reaper_plugin_functions.html`，或者运行 Action `ReaScript: 显示 API 帮助`。**这是唯一权威的参考，别只依赖教程。**

---

## 五、两种集成方向

### 方向 A：脚本住在 REAPER 里

适合：操作工程本体（移动片段、批量改名、MIDI 处理）

```
REAPER → Actions → 加载脚本 → 执行
```

Lua 脚本直接跑。Python 脚本也能跑，但受限于 REAPER 的解释器。

### 方向 B：外部 Python 遥控 REAPER

适合：要跑 AI 模型（需要 torch / librosa 等重依赖）

```
你的 Python 脚本（跑模型）
    ↓
  操作 REAPER（通过 socket / 文件交换 / 命令行）
```

外部控制 REAPER 的思路：

| 方式 | 说明 |
|---|---|
| **借道脚本** | 外部脚本生成一个 `.lua` 临时文件，REAPER 里一个 Action 去执行它 |
| **reapy** | 第三方 Python 库，提供 `reapy` 包直接遥控 REAPER，需要在 REAPER 里装对应组件 |
| **文件交换** | 最土但最稳：Python 写一个 JSON / MIDI 文件 → Lua 脚本读取并应用 |
| **命令行渲染** | 用 `reaper -renderproject xxx.rpp` 做无头渲染 |

> ⚠️ **reapy 的维护状态请自行核实**（去 GitHub 看最后提交时间）。
> 「文件交换」这种方式看着笨，但**永远能用、永远不坏**，新手我建议从这里开始。

---

## 六、Python 这边的 MIDI 三件套

你的环境里已经装了这三个：

| 库 | 层次 | 适合 | 特点 |
|---|---|---|---|
| **mido** | 底层 | 实时 MIDI 消息、端口通信 | 精确控制每个 message，能接 MIDI 硬件 |
| **pretty_midi** | 高层 | 分析、处理现成 MIDI 文件 | 有 `get_piano_roll()`、和弦识别等音乐学工具 |
| **MIDIUtil** | 生成 | 从零写一个 MIDI 文件 | 写起来最直白，适合"把数据变成 MIDI" |

**怎么选**：

```python
# 要从零生成一段 MIDI → MIDIUtil
from midiutil import MIDIFile
mf = MIDIFile(1)
mf.addTempo(0, 0, 120)
for i, pitch in enumerate([60, 62, 64, 65]):
    mf.addNote(0, 0, pitch, i, 1, 100)
with open("out.mid", "wb") as f:
    mf.writeFile(f)
```

```python
# 要读进来分析 → pretty_midi
import pretty_midi
pm = pretty_midi.PrettyMIDI("in.mid")
piano_roll = pm.get_piano_roll(fs=100)   # [128, time] 的矩阵，可以直接喂 numpy
```

```python
# 要实时收发 MIDI 消息 → mido
import mido
with mido.open_output() as out:
    out.send(mido.Message("note_on", note=60, velocity=100))
```

---

## 七、一个真实的工作流长什么样

假设你想做：**把一段音频变成 MIDI，然后自动铺到 REAPER 工程里**

```
1. Python: 读音频 → 重采样 → 送进转写模型 → 得到 MIDI 文件
2. Python: 把 MIDI 转成 REAPER 能读的格式（或写个 JSON 描述音符）
3. 触发 REAPER：
     - 方案 A：生成一个 .lua，手动/快捷键执行
     - 方案 B：用命令行让 REAPER 打开工程并跑 Action
4. Lua: 读 JSON → 在指定轨道上建 MIDI item → 插入音符
5. REAPER: 你听到结果
```

**新手建议**：先把 1 和 5 手动跑通，确认模型输出是对的；**再**去折腾 2-4 的自动化。一上来就全自动，出错时你根本不知道是哪一环。

---

## 八、坑清单

| 坑 | 表现 | 解法 |
|---|---|---|
| Python 解释器不是你以为的那个 | `import` 失败 | 在 REAPER 里打印 `sys.executable` |
| 时间单位混用 | 音符位置全错 | REAPER 内部用**秒**，MIDI 用 **tick / beat**，转换要显式做 |
| 采样率不一致 | 音高错乱 | 见 [01-音频基础.md](01-音频基础.md) |
| 脚本改了没生效 | 还是旧行为 | REAPER 缓存脚本，重新加载 Action 或重启 |
| 编码问题 | 中文路径/文件名乱码 | Lua 和 Python 对 UTF-8 处理不同，尽量用英文路径测试 |
| 硬编码绝对路径 | 换机器就崩 | 用相对路径或工程目录变量 |
| 忘了 `Undo` | 脚本改错无法回退 | 脚本开头调 `reaper.Undo_BeginBlock()`，结尾 `Undo_EndBlock()` |

---

## 九、从哪开始

按这个顺序，每一步都能独立看到结果：

1. **写一个最简单的 Lua 脚本**：遍历选中轨道，把名字前面加上 `[AI]`
   → 熟悉"加载脚本 → 执行 → 看效果"这个循环
2. **读一段 MIDI**：用 pretty_midi 打印出所有音符的音高和时长
   → 熟悉 MIDI 数据结构
3. **生成一段 MIDI**：用 MIDIUtil 写一个 C 大调音阶，导入 REAPER
   → 打通"代码 → DAW"
4. **接到真实模型**：跑一次音频转 MIDI，把结果导入
   → 完整链路
5. **再做自动化**：把上面手工的步骤写成脚本
   → 这时候你才知道哪里该自动化

---

## 相关

- 转 MIDI 的模型 → [05-音频转MIDI.md](05-音频转MIDI.md)
- 音频基础概念 → [01-音频基础.md](01-音频基础.md)
