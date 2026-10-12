# 06 · DAW 集成与自动化

> 这篇离你最近——你本来就在 REAPER 里干活。
> 音乐 AI 最尴尬的一点是：**模型住在命令行里，你的作品住在 DAW 里。** 这一篇讲怎么把最后一公里打通。
>
> **核实时间：2026-10-12**

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

## 二、REAPER 为什么适合自动化

| 特性 | REAPER | 多数其他 DAW |
|---|---|---|
| 脚本语言 | **Lua / EEL2 / Python 三种** | 通常只有一种，或没有 |
| API 覆盖面 | 几乎能操作一切：轨道、片段、MIDI、渲染、FX | 有限 |
| 脚本分发 | 一个 `.lua` 文件，扔进 Scripts 目录 | 要编译插件 |
| 命令行渲染 | 支持（`reaper -renderproject`） | 部分支持 |
| 便携安装 | 支持 | 少见 |

**结论**：目标如果是"音乐 + AI + 自动化"，REAPER 不是妥协，是**最优解**。

---

## 三、ReaScript 三种语言怎么选

| 语言 | 适合 | 不适合 |
|---|---|---|
| **Lua** | 首选。生态最大，ReaPack 上绝大多数脚本是 Lua，启动快 | 没有 pip 生态 |
| **EEL2** | 极致性能的 DSP 循环 | 语法古怪，写业务逻辑很痛苦 |
| **Python** | 需要 numpy / librosa 的时候 | ⚠️ 见下面那个坑 |

### ⚠️ 重要：ReaScript 的 Python 环境很麻烦

REAPER 调用的是**它自己找到的**那个 Python 解释器。你机器上有 4 个 Python，很容易对不上。而且**在里面装 numpy / torch 这类重依赖非常折腾**。

**正确做法**：

```
REAPER 侧的脚本（Lua）  ←→  外部 Python 进程（跑模型）
        简单操作                  重依赖都放这里
        通过文件 / 命令行交换数据
```

> **重活永远放外部进程。** 别试图在 ReaScript 里 import torch。

调试前先确认它用的是哪个解释器：在 REAPER 里跑 `reaper.ShowConsoleMsg(sys.executable)`。

---

## 四、REAPER 生态工具

| 名称 | 用途 | 链接 | 状态 | 备注 |
|---|---|---|---|---|
| **ReaScript API 文档** | 官方 API 参考 | [reaper.fm/sdk/reascript](https://www.reaper.fm/sdk/reascript/reascripthelp.html) | 随版本更新 | ⭐ **唯一权威参考**，别只依赖教程 |
| **ReaPack** | REAPER 包管理器 | [codeberg.org/cfillion/reapack](https://codeberg.org/cfillion/reapack) | 🟢 | ⚠️ **已从 GitHub 迁到 Codeberg**，旧 GitHub 仓已归档 |
| **SWS Extension** | 大量动作与扩展 API | [reaper-oss/sws](https://github.com/reaper-oss/sws) | 🟢 2026-05 | **脚本能力的放大器** |
| **ReaImGui** | 在 REAPER 里画 Dear ImGui 界面 | [cfillion/reaimgui](https://github.com/cfillion/reaimgui) | 🟢 | 复杂脚本 UI 首选 |
| ~~**reapy**~~ | ~~外部 Python 控制 REAPER~~ | [RomeoDespres/reapy](https://github.com/RomeoDespres/reapy) | 🔴 **停止维护 2021-02** | ❌ **别用**，很多老教程还指向它 |

### ReaScript 的核心概念

```
Project（工程）
 └── Track（轨道）
      └── Item（片段，音频或 MIDI）
           └── Take（片段里的一个 take）
                └── MIDI 数据 / 音频源
```

**几乎所有脚本都是在这棵树上爬。**

> **脚本开头别忘了** `reaper.Undo_BeginBlock()`，结尾 `Undo_EndBlock()`。
> 不然脚本改错了，用户按 Ctrl+Z 退不回去。

---

## 五、跨宿主的 Python 方案

> 如果你不想被 REAPER 绑死，或者需要**离线批量渲染**，这两条路更好走。

| 名称 | 用途 | 链接 | 状态 | 备注 |
|---|---|---|---|---|
| **DawDreamer** | Python 驱动 DAW 图（**含 VST 渲染**） | [DBraun/DawDreamer](https://github.com/DBraun/DawDreamer) | 🟢 2026-10 | ⭐ **离线批量渲染与音频 ML 数据管线** |
| **pedalboard**（Spotify） | Python 音频效果链（VST3/AU 宿主） | [spotify/pedalboard](https://github.com/spotify/pedalboard) | 🟢 2026-10 | **与 librosa / torch 配合做数据增广很顺手** |

**为什么推荐这两条**：纯 Python 环境，没有宿主依赖，能塞进训练管线。

---

## 六、其他 DAW 的自动化

| 名称 | 用途 | 链接 | 状态 | 备注 |
|---|---|---|---|---|
| **ableton-js** | 用 JS/TS 控制 Ableton Live | [leolabs/ableton-js](https://github.com/leolabs/ableton-js) | 🟢 2026-10 | 依赖 Live 的 MIDI Remote Script |
| **AbletonOSC** | 用 OSC 控制 Live（任意语言） | [ideoforms/AbletonOSC](https://github.com/ideoforms/AbletonOSC) | 🟡 2025-11 | **跨语言自动化首选路径** |
| **DrivenByMoss** | Bitwig 控制器/自动化扩展 | [git-moss/DrivenByMoss](https://github.com/git-moss/DrivenByMoss) | 🟢 2026-10 | Bitwig 生态最活跃项目 |
| **Ardour** | 开源 DAW（自带 Lua 脚本） | [Ardour/ardour](https://github.com/Ardour/ardour) | 🟢 2026-10 | GPL；脚本能力不如 REAPER |
| **Tidal / Strudel** | Live coding 图案语言 | [Codeberg: uzu/tidal](https://codeberg.org/uzu/tidal) · [strudel](https://codeberg.org/uzu/strudel) | 🟢 | ⚠️ **已迁 Codeberg**，GitHub 原仓归档 |
| **Faust** | 音频 DSP 函数式语言 | [grame-cncm/faust](https://github.com/grame-cncm/faust) | 🟢 2026-10 | 算法级音效开发 |
| **JUCE / iPlug2 / nih-plug / DPF** | 音频插件框架 | [JUCE](https://github.com/juce-framework/JUCE) · [iPlug2](https://github.com/iPlug2/iPlug2) · [nih-plug](https://github.com/robbert-vdh/nih-plug) · [DPF](https://github.com/DISTRHO/DPF) | 🟢 | 做 VST3/AU 的正路；校验用 [pluginval](https://github.com/Tracktion/pluginval) |
| **Logic Pro** | — | — | — | ❌ **无官方脚本/API**，任何"自动化方案"都很脆 |

---

## 七、Python 这边的 MIDI 生态

| 库 | 层次 | 适合 | 状态 |
|---|---|---|---|
| **mido** | 底层 | 实时 MIDI 消息、端口通信 | 🟢 MIT |
| **pretty_midi** | 高层 | 分析、处理现成 MIDI 文件 | 🟢 MIT |
| **music21** | 乐理层 | 分析、转换、乐理运算 | 🟢 BSD（速度慢） |
| **MidiTok** | 分词 | 给 Transformer 用 | 🟢 MIT |
| **MIDIUtil** | 生成 | 从零写 MIDI 文件 | 🔴 停止维护 2018 |
| **python-rtmidi** | 实时端口 | mido 的实时后端 | 🟢 |

**怎么选**：

```python
# 要从零生成一段 MIDI → MIDIUtil（简单）或 mido（更可控）
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
piano_roll = pm.get_piano_roll(fs=100)   # [128, time] 矩阵，可直接喂 numpy
```

```python
# 要实时收发 MIDI 消息 → mido + python-rtmidi
import mido
with mido.open_output() as out:
    out.send(mido.Message("note_on", note=60, velocity=100))
```

---

## 八、一个真实的工作流长什么样

假设你想做：**把一段音频变成 MIDI，然后自动铺到 REAPER 工程里**

```
1. 外部 Python: 读音频 → 重采样 → 分离 → 转写模型 → 得到 MIDI 文件
2. 外部 Python: 把 MIDI 转成 REAPER 能读的格式（或写个 JSON 描述音符）
3. 触发 REAPER：
     - 方案 A：生成一个 .lua，手动/快捷键执行
     - 方案 B：用命令行让 REAPER 打开工程并跑 Action
4. Lua: 读 JSON → 在指定轨道上建 MIDI item → 插入音符
5. REAPER: 你听到结果
```

**新手建议**：先把 1 和 5 手动跑通，确认模型输出是对的；**再**去折腾 2-4 的自动化。一上来就全自动，出错时你根本不知道是哪一环。

---

## 九、坑清单

| 坑 | 表现 | 解法 |
|---|---|---|
| **ReaScript 的 Python 环境** | `import` 失败 | 打印 `sys.executable`；**重活放外部进程** |
| **reapy 已死** | 老教程教你装 reapy | **别用**，2021 年就停了 |
| **ReaPack 链接失效** | GitHub 仓已归档 | 去 **Codeberg** |
| **时间单位混用** | 音符位置全错 | REAPER 内部用**秒**，MIDI 用 **tick / beat**，转换要显式做 |
| **采样率不一致** | 音高错乱 | 见 [01-音频基础.md](01-音频基础.md) |
| **脚本改了没生效** | 还是旧行为 | REAPER 缓存脚本，重新加载 Action 或重启 |
| **忘了 Undo** | 脚本改错无法回退 | `Undo_BeginBlock()` / `Undo_EndBlock()` |
| **宿主大版本升级** | 脚本失效 | DAW 自动化的稳定性远低于纯 Python |

---

## 十、从哪开始

按这个顺序，每一步都能独立看到结果：

1. **写一个最简单的 Lua 脚本**：遍历选中轨道，把名字前面加上 `[AI]`
   → 熟悉"加载脚本 → 执行 → 看效果"这个循环
2. **读一段 MIDI**：用 `pretty_midi` 打印所有音符的音高和时长
3. **生成一段 MIDI**：用 `MIDIUtil` 写一个 C 大调音阶，导入 REAPER
   → 打通"代码 → DAW"
4. **试 pedalboard**：用 Python 给一段音频加效果，导出
   → 打通"Python → 音频处理"，不需要宿主
5. **接到真实模型**：跑一次音频转 MIDI，把结果导入
   → 完整链路
6. **再做自动化**：把上面手工的步骤写成脚本
   → 这时候你才知道哪里该自动化

---

## 相关

- 转 MIDI 的模型 → [05-音频转MIDI.md](05-音频转MIDI.md)
- 音频基础概念 → [01-音频基础.md](01-音频基础.md)
