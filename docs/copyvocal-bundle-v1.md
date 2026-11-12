# OpenTune × CopyVocal 交换格式 v1（`.cvf`）

本文档是 **OpenTune CopyVocal Bundle** 的权威格式定义。CopyVocal 负责把一次修正任务
导出成 `.cvf`，OpenTune 负责读取 `.cvf` 并生成一个**可继续编辑的工程**：原始学生
音频、音符、时间网格、原始 F0 与 CopyVocal 的目标 F0 都会进入 OpenTune 的编辑模型，
而不是简单导入 `student.corrected.wav`。

- 文件扩展名：`.cvf`（CopyVocal File）；旧扩展名 `.cvot` 仍可导入
- 媒体类型：`application/vnd.opentune.copyvocal+json;version=1`
- 文本编码：UTF-8，无 BOM
- 数字：JSON number，禁止 `NaN` / `Infinity` / `-Infinity`
- 字节序：所有二进制数组为 **little-endian**
- 生产者：CopyVocal `application.opentune_bundle`
- 消费者：OpenTune `Source/Utils/CopyVocalBundle.*`

---

## 1. 设计目标

1. **可编辑，而不是回放**：OpenTune 导入后得到的是 Note + PitchCurve + TimeGrid，
   用户可以继续改音符、改音高、改时间伸缩。
2. **不依赖 CopyVocal 内部文件**：OpenTune 只读 `.cvf`，不解析 `alignment.json`、
   `target_pitch.npz` 等实现细节；CopyVocal 内部结构变化不会直接破坏 OpenTune。
3. **坐标系显式化**：学生源时间、教师时间、输出时间、OpenTune 工程源时间四个时钟
   在文件里分开命名，禁止隐式假设。
4. **可验证**：所有数组长度、帧率、时长、时间网格端点都可在导入前校验。

---

## 2. 四个时钟

| 名称 | 字段 | 含义 |
| --- | --- | --- |
| 学生源时间 | `clocks.studentSeconds` | `student.reference.wav` / 原始学生录音的时间轴 |
| 教师时间 | `clocks.teacherSeconds` | `teacher.reference.wav` 的时间轴 |
| 输出时间 | `clocks.outputSeconds` | CopyVocal 渲染结果的时间轴；`timingChanged=true` 时等于教师时间 |
| 工程源时间 | `clocks.projectSourceSeconds` | OpenTune clip 的 source 时间轴，**恒等于输出时间** |

`.cvf` 中以下内容统一使用**工程源时间**：

- `f0.original` / `f0.target` 的帧原点
- `notes[].start` / `notes[].end`
- `timeGrid.handles[].sourceSeconds`
- `markers[].start/end`（当 `clock == "source"`）

以下内容保留各自原始时钟：

- `regions[].teacherStart/teacherEnd`、`regions[].teacherAnchors`：教师时间
- `regions[].studentStart/studentEnd`、`regions[].studentAnchors`：学生源时间
- `words.student`：学生源时间
- `words.teacher`：教师时间

> **为什么工程源时间 = 输出时间？**
> OpenTune 的 `TimeGrid` 有不变量：clip 的源总时长必须等于输出总时长。
> CopyVocal 的完整修正输出长度等于教师长度，通常不等于学生长度。因此导入时
> OpenTune 会把学生参考音频**补零/裁剪**到输出时长，形成“工程源音频”。
> 这个操作只发生在导入内存/工程媒体副本中，不修改 CopyVocal 的原始文件。
> 补零区间的 `f0.original` / `f0.target` 必须为 0。

---

## 3. 顶层结构

```json
{
  "format": "opentune.copyvocal-bundle",
  "formatVersion": 1,
  "producer": { ... },
  "project": { ... },
  "clocks": { ... },
  "media": { ... },
  "f0": { ... },
  "notes": [ ... ],
  "regions": [ ... ],
  "timeGrid": { ... },
  "words": { ... },
  "markers": [ ... ],
  "diagnostics": { ... }
}
```

### 3.1 `producer`（必需）

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `name` | string | 是 | 固定 `"CopyVocal"` |
| `version` | string | 是 | CopyVocal 版本 |
| `createdAt` | string | 是 | ISO-8601 UTC |
| `jobStatus` | string | 是 | `job.json.status` |
| `jobId` | string | 否 | 任务标识，建议用输入 SHA-256 组合 |
| `sourceJobDir` | string | 否 | 原始输出目录，仅用于溯源 |

### 3.2 `project`（必需）

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `name` | string | 是 | 工程名，OpenTune 用作默认工程名 |
| `language` | string | 是 | `zh` / `en` / `ja` |
| `sampleRate` | int | 是 | 固定 44100 |

### 3.3 `clocks`（必需）

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `studentSeconds` | number | 是 | 学生参考音频时长 |
| `teacherSeconds` | number | 是 | 教师参考音频时长 |
| `outputSeconds` | number | 是 | CopyVocal 渲染输出时长 |
| `projectSourceSeconds` | number | 是 | 必须等于 `outputSeconds` |
| `outputClock` | string | 是 | `"teacher"` 或 `"student"` |
| `timingChanged` | bool | 是 | 与 `target_pitch.npz["timing_changed"]` 一致 |
| `pitchPolicy` | string | 是 | 例如 `teacher_contour_student_register` |
| `registerOffsetCents` | number | 是 | 目标音高的整体寄存器偏移，0 表示无 |

### 3.4 `media`（必需）

```json
"media": {
  "student": {
    "role": "student",
    "path": "student.reference.wav",
    "originalPath": "D:/songs/student.wav",
    "sha256": "...",
    "sampleRate": 44100,
    "channels": 1,
    "sampleCount": 329466,
    "durationSeconds": 7.470884353741496
  },
  "teacher": { ... },
  "corrected": { ... }
}
```

- `path`：相对于 `.cvf` 文件所在目录；允许绝对路径。推荐使用相对路径，便于整体移动。
- `student` 必需；`teacher` / `corrected` 可选。
- `channels` 允许 1 或 2；OpenTune 只接受 1/2 声道。
- `sampleCount` / `durationSeconds` 必须来自实际音频文件，而不是分析 JSON 的估算。

### 3.5 `f0`（必需）

```json
"f0": {
  "hopSize": 160,
  "sampleRate": 16000,
  "frameStepSeconds": 0.01,
  "frameCount": 733,
  "original": {
    "encoding": "base64-f32le",
    "values": "AAAAAA...",
    "voiced": "AAAB..."
  },
  "target": {
    "encoding": "base64-f32le",
    "values": "AAAAAA...",
    "voiced": "AAAB..."
  }
}
```

- `hopSize` 必须为 160，`sampleRate` 必须为 16000；即 10 ms 一帧，与 OpenTune
  的 F0 域一致。v1 不接受其他帧率。
- `frameCount` 是 `values` 的 float32 数量，也是 `voiced` 的字节数。
- `values`：base64 后的 little-endian `float32[frameCount]`，单位 Hz；无声帧写 0。
- `voiced`：base64 后的 `uint8[frameCount]`，0/1。
- 第 i 帧时间 = `i * hopSize / sampleRate`。
- `original`：学生原始 F0（RMVPE），映射到工程源时间；补零区间为 0。
- `target`：CopyVocal 的目标 F0，映射回工程源时间；无目标/未渲染区间为 0。
  OpenTune 会把它作为 `PitchCurve` 的修正段（HandDraw）。
- `target` 只有在 `original > 0` 的帧上才会真正生效；这是 OpenTune 渲染器的限制。

### 3.6 `notes`（必需，可为空数组）

```json
{
  "start": 0.74,
  "end": 0.90,
  "pitchHz": 392.438,
  "midi": 67.106,
  "lyric": "我",
  "source": "teacher",
  "confidence": 1.0
}
```

- 时间使用工程源时间，按 `start` 升序，互不重叠。
- `pitchHz` 是**目标音高**（已包含寄存器偏移），OpenTune 直接写入 `Note::pitch`。
- `midi` 仅用于诊断/显示，OpenTune 不依赖它。
- `source`：`"teacher"` / `"student"` / `"target"`，说明音符来源。
- `lyric` 可为空；v1 的 OpenTune 不显示歌词，但保留给后续版本。

### 3.7 `regions`（必需，可为空数组）

```json
{
  "teacherStart": 0.52,
  "teacherEnd": 0.70,
  "studentStart": 0.74,
  "studentEnd": 0.90,
  "status": "matched",
  "mode": "synthesize",
  "quality": 1.0,
  "unsupported": false,
  "reason": "matched",
  "teacherAnchors": [0.52, 0.55, 0.70],
  "studentAnchors": [0.74, 0.77, 0.90],
  "stretchRatio": 1.125
}
```

- `mode`：`synthesize` / `mute` / `silence` / `acoustic_fallback`。
- 只有 `mode == "synthesize"`（或允许的 `acoustic_fallback`）且 `unsupported == false`
  的区域 anchors 会参与 TimeGrid。其余区域只提供边界。
- `teacherAnchors` / `studentAnchors` 必须等长、严格递增。
- 时间域见第 2 节。

### 3.8 `timeGrid`（必需）

```json
"timeGrid": {
  "totalDurationSeconds": 7.321473922902494,
  "handles": [
    { "sourceSeconds": 0.0, "outputSeconds": 0.0, "kind": "clipStart", "confidence": "default" },
    { "sourceSeconds": 0.74, "outputSeconds": 0.52, "kind": "onsetVoiced", "confidence": "high" },
    { "sourceSeconds": 0.77, "outputSeconds": 0.55, "kind": "internalOnset", "confidence": "high" },
    { "sourceSeconds": 7.321473922902494, "outputSeconds": 7.321473922902494, "kind": "clipEnd", "confidence": "default" }
  ]
}
```

- 第一个必须是 `clipStart`，`(0, 0)`，`confidence = "default"`。
- 最后一个必须是 `clipEnd`，`(D, D)`，`confidence = "default"`。
- `sourceSeconds` 与 `outputSeconds` 都必须严格递增。
- `D = clocks.projectSourceSeconds = clocks.outputSeconds`。
- `kind`：`clipStart` / `clipEnd` / `onsetVoiced` / `onsetSibilant` / `onsetSilence` /
  `internalOnset` / `userAdded`。
- `confidence`：`default` / `high`。
- v1 允许 CopyVocal 的高置信度锚点**密于 150 ms**（例如音节/音素边界）。OpenTune
  只在 `high` + `high` 的相邻 handle 上放宽 150 ms 限制；用户后来手动插入的 handle
  仍遵守 150 ms 规则。

### 3.9 `words`（可选）

```json
"words": {
  "student": [
    { "text": "我", "start": 0.74, "end": 0.90,
      "phonemes": [{ "text": "w", "start": 0.74, "end": 0.77 }] }
  ],
  "teacher": [ ... ]
}
```

- 仅用于溯源和后续歌词功能；OpenTune v1 不消费。
- 学生词用学生源时间，教师词用教师时间。

### 3.10 `markers`（可选）

```json
"markers": [
  {
    "start": 0.51, "end": 0.52,
    "clock": "output",
    "kind": "unresolved",
    "label": "voiced_teacher_region_without_lyric_correspondence"
  }
]
```

- `clock`：`source` 或 `output`。
- v1 OpenTune 不显示 marker，仅用于诊断/未来功能。

### 3.11 `diagnostics`（可选）

```json
"diagnostics": {
  "warnings": ["Possible lyric mismatch or ASR error: ..."],
  "unresolvedRegionCount": 1,
  "unsupportedDurationSeconds": 0.12
}
```

---

## 4. CopyVocal → `.cvf` 的映射规则

### 4.1 输出时长与工程源时长

```
timingChanged = target_pitch.npz["timing_changed"]
outputSeconds = render.json["experiments"]["full"]["sample_count"] / sample_rate
projectSourceSeconds = outputSeconds
```

若 `timingChanged == false`，`outputSeconds = studentSeconds`，TimeGrid 为恒等映射。

### 4.2 原始 F0

取 `student.analysis.json["pitch"]`（10 ms 网格），按工程源帧数截断/补零：

```
frameCount = round(projectSourceSeconds / 0.01) + 1
original[i] = student_f0[i]  (i < student_frame_count else 0)
```

若学生分析不是 10 ms 网格，导出器必须重采样到 160/16000；v1 的 OpenTune 只接受
160/16000。

### 4.3 目标 F0

`target_pitch.npz` 的 `times` 是输出时间，`source_times` 是每个输出帧对应的学生源时间。
CopyVocal 的目标音高位于声码器帧网格（约 11.6 ms），**不能**用
`round(source_time / 0.01)` 直接散射到 10 ms 工程源网格——那样每约 6 帧就会留下一个
空帧，OpenTune 的音高线会表现为周期性掉点。

正确做法是把目标 F0 视为 `source_times` 上的函数，在每个 10 ms 工程源帧处插值：

```
valid = rendered & finite(source_times) & finite(f0_hz) & f0_hz > 0
把 (source_times[valid], f0_hz[valid]) 按源时间排序，并对重复源时间取对数域均值
for s in range(frameCount):
    t = s * 0.01
    找到源时间上 bracketing 的两个有效帧 t0 <= t <= t1
    if t1 - t0 <= 0.030:
        target[s] = exp(lerp(log(f0_0), log(f0_1), (t - t0) / (t1 - t0)))
    elif t 距首个/末个有效帧不足一个目标帧步长:
        target[s] = 最近有效帧的值
    else:
        target[s] = 0
```

- `0.030` 秒的阈值允许跨过一个丢失的声码器帧（11.6 ms × 2 ≈ 23 ms），但不会把真正
  的无声段（气口、辅音、未渲染区）填成虚假音高。
- 对数域插值避免算术平均在音高上产生偏置。
- 导出器还应对每个连续有声段做 **5 帧中值滤波**，抑制 1–3 帧的音高尖峰。这类尖峰
  低于声码器的时间分辨率，在 CopyVocal 渲染音频中不可闻，但会原样进入 OpenTune 的
  曲线和渲染，在音符接缝处表现为异常颤音。
- 要求：`target[s] > 0` 仅当 `original[s] > 0`。若原始帧无声，导出器把 target 置 0，
  避免 OpenTune 把修正段切碎后留下空洞。

### 4.4 音符

优先使用 `teacher.analysis.json["notes"]`（教师 GAME 音符）：

1. 对每个教师音符 `[ts, te]`，取输出帧 `times ∈ [ts, te]` 且 `rendered` 的帧。
2. `sourceStart = min(source_times)`，`sourceEnd = max(source_times)`。
3. `pitchHz = median(target_f0_hz)`（仅取 >0 的帧）；没有则用
   `midi_to_hz(midi + registerOffsetCents/100)`。
4. 裁剪到 `[0, projectSourceSeconds]`，丢弃时长 < 20 ms 的音符。
5. 排序并消除重叠：后一个音符的 `start` 不得早于前一个的 `end`。

如果教师没有音符，使用学生 GAME 音符；如果仍没有，从 `target` F0 做简单的
稳定段切分。

### 4.5 TimeGrid

1. 加入 `(0, 0)` 和 `(D, D)`。
2. 对每个 `mode ∈ {"synthesize", "acoustic_fallback"}` 且 `unsupported == false` 的区域：
   把 `studentAnchors[i] → teacherAnchors[i]` 全部加入。
3. 对其余区域（`mute` / `silence` / 不支持）：
   若 `studentEnd > studentStart`，加入 `(studentStart, teacherStart)` 和
   `(studentEnd, teacherEnd)` 作为边界。
4. 只保留 `0 ≤ source ≤ D`、`0 ≤ output ≤ D` 的对。
5. 按 source 排序；跳过 source 不递增或 output 不递增的冲突项。
6. 内部 handle 全部标 `confidence = "high"`；端点标 `"default"`。
7. 若最终没有内部 handle，输出恒等 TimeGrid。

### 4.6 媒体

- `student` 优先 `student.reference.wav`（已 downmix/重采样），否则 `job.json.student`。
- `teacher` 优先 `teacher.reference.wav`，否则 `job.json.teacher`。
- `corrected` 若 `student.corrected.wav` 存在则包含。
- `sha256` 可选但推荐；导出器默认计算 `student` / `teacher`。

---

## 5. OpenTune 侧的导入语义

1. 读取 `.cvf`，校验格式版本、帧率、数组长度、时间网格不变量。
2. 加载 `media.student`，把音频补零/裁剪到 `clocks.projectSourceSeconds`。
3. `prepareImport` 生成 source/clip；`sourceWindow = [0, projectSourceSeconds]`。
4. 写入：
   - `notes` → `ContentState.notes`
   - `f0.original` → `PitchCurve.originalF0`
   - `f0.target` → `PitchCurve` 修正段（`PitchCorrectionSegment::Source::HandDraw`）
   - `timeGrid.handles` → `TimeGridSnapshot`
   - `analysis.originalF0State = Ready`（跳过 F0 重新提取）
5. 触发一次完整渲染。用户随后可以像普通 OpenTune 工程一样编辑、保存为 `.otproj`。
6. 导入后工程默认标记为“未保存”，工程路径建议为 `.cvf` 同目录下的同名 `.otproj`；
   用户按 Ctrl+S 即可固化为普通 OpenTune 工程。

---

## 6. 校验清单（导入器必须执行）

- `format == "opentune.copyvocal-bundle"` 且 `formatVersion == 1`
- `clocks.projectSourceSeconds == clocks.outputSeconds > 0`
- `f0.hopSize == 160`、`f0.sampleRate == 16000`
- `values` 解码后字节数 == `frameCount * 4`
- `voiced` 解码后字节数 == `frameCount`
- `clocks.timingChanged == false` 时 TimeGrid 必须是恒等映射
- TimeGrid 端点、严格递增、总时长守恒
- `notes` 时间在 `[0, projectSourceSeconds]` 内
- 媒体路径存在且可读，声道数 1/2，采样率 > 0
- 所有 number 有限

---

## 7. 版本策略

- `formatVersion` 只在**不兼容**变化时递增。
- 新增可选字段不递增版本；旧导入器忽略未知字段。
- CopyVocal 和 OpenTune 都必须拒绝高于自己支持版本的 bundle，并给出明确错误。
- v1 的 OpenTune 导入器不依赖 `words` / `markers` / `diagnostics`；它们可以缺失。

---

## 8. 最小示例

```json
{
  "format": "opentune.copyvocal-bundle",
  "formatVersion": 1,
  "producer": {
    "name": "CopyVocal",
    "version": "1.0",
    "createdAt": "2026-11-10T10:00:00+00:00",
    "jobStatus": "completed_with_unresolved"
  },
  "project": { "name": "demo", "language": "zh", "sampleRate": 44100 },
  "clocks": {
    "studentSeconds": 7.470884353741496,
    "teacherSeconds": 7.321473922902494,
    "outputSeconds": 7.321473922902494,
    "projectSourceSeconds": 7.321473922902494,
    "outputClock": "teacher",
    "timingChanged": true,
    "pitchPolicy": "teacher_contour_student_register",
    "registerOffsetCents": 0.0
  },
  "media": {
    "student": {
      "role": "student",
      "path": "student.reference.wav",
      "sampleRate": 44100,
      "channels": 1,
      "sampleCount": 329466,
      "durationSeconds": 7.470884353741496
    }
  },
  "f0": {
    "hopSize": 160,
    "sampleRate": 16000,
    "frameStepSeconds": 0.01,
    "frameCount": 733,
    "original": { "encoding": "base64-f32le", "values": "...", "voiced": "..." },
    "target": { "encoding": "base64-f32le", "values": "...", "voiced": "..." }
  },
  "notes": [
    { "start": 0.74, "end": 0.90, "pitchHz": 392.438, "midi": 67.106,
      "lyric": "我", "source": "teacher", "confidence": 1.0 }
  ],
  "regions": [
    { "teacherStart": 0.52, "teacherEnd": 0.70,
      "studentStart": 0.74, "studentEnd": 0.90,
      "status": "matched", "mode": "synthesize", "quality": 1.0,
      "unsupported": false, "reason": "matched",
      "teacherAnchors": [0.52, 0.55, 0.70],
      "studentAnchors": [0.74, 0.77, 0.90],
      "stretchRatio": 1.125 }
  ],
  "timeGrid": {
    "totalDurationSeconds": 7.321473922902494,
    "handles": [
      { "sourceSeconds": 0.0, "outputSeconds": 0.0, "kind": "clipStart", "confidence": "default" },
      { "sourceSeconds": 0.74, "outputSeconds": 0.52, "kind": "onsetVoiced", "confidence": "high" },
      { "sourceSeconds": 7.321473922902494, "outputSeconds": 7.321473922902494, "kind": "clipEnd", "confidence": "default" }
    ]
  }
}
```
