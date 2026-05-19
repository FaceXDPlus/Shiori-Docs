# Shiori 项目介绍

## 一、软件简介

**Shiori** 是一款基于 **ARKit** 的 iOS 端面部动作捕捉
应用。它通过 iPhone / iPad 的前置 TrueDepth 摄像头获取 52 项 ARKit
`BlendShape`、头部姿态、眼球朝向等数据，转换为 Live2D 模型可消费的参数，并实时
通过 TCP / WebSocket 推送给 PC 端的桌面虚拟形象或直播软件，让手机充当一台专用
的面捕推流设备。

---

## 二、数据包总览

所有数据包均为 **UTF-8 编码的 JSON 文本**，每一帧（目标 60Hz）发送一条，没有
附加长度头和分隔符。

- **WebSocket**：一条 message 即一个完整 JSON。
- **TCP**：接收端需自行根据 JSON 边界（`{` / `}` 配对，或换行）拆包。

Shiori 在「数据模式」按钮中提供三种数据包格式，可在 App 内切换：

| 模式 | 简介 | 适用场景 |
| --- | --- | --- | --- |
| **NekoDice** | Live2D 2D 通道映射，短键名 + 字符串数值 | 普通 Live2D 模型驱动 |
| **CuteNova** | 52 项 ARKit BlendShape 短键 + 头部 6DoF | Perfect Sync 模型 / 需要全脸细节 |
| **Raw** | 直接透出 ARKit 原始字段、4×4 变换矩阵、双眼矩阵、lookAt 点 | 上位机自行做二次处理 / 调试 |

> 三种模式的字段集是**不相交**的（键名 / 结构都不一样），上位机可以通过判断
> 关键字段（如 `xs` / `transform` / `headPitch`）来识别当前来源模式。

数值精度：
- **NekoDice**：字符串浮点，保留 3 位小数；`isTracked` 为 Bool；`xs` 固定为 `true`。
- **CuteNova**：JSON 数字（BlendShape 0–1 之间，头部角度 / 位置为浮点）；`iT` 为 Bool。
- **Raw**：JSON 数字；`isTracked` 为 Bool。

---

## 三、NekoDice 模式

### 3.1 字段说明

| Key | 类型 | 含义 |
| --- | --- | --- |
| `hP` / `hY` / `hR` | String(float) | 头部 pitch / yaw / roll（度） |
| `bX` / `bY` / `bZ` | String(float) | 身体角度（联动头部） |
| `bMX` / `bMY` / `bMZ` | String(float) | 身体位移 |
| `eLO` / `eRO` | String(float) | 左 / 右眼开合，0–1 |
| `eLS` / `eRS` | String(float) | 左 / 右眼笑，0–1 |
| `eBYL` / `eBYR` | String(float) | 左 / 右眉上下 |
| `eBLF` / `eBRF` | String(float) | 左 / 右眉形变 |
| `eBAL` / `eBAR` | String(float) | 左 / 右眉角度 |
| `eX` / `eY` | String(float) | 眼球朝向，-1–1 |
| `mO` | String(float) | 嘴张开 |
| `mF` | String(float) | 嘴形 |
| `mU` / `mT` / `mX` | String(float) | 嘴 U / T / X 方向形变 |
| `mPU` / `mP` / `mC` | String(float) | 撅嘴 / 鼓嘴 / 歪嘴 |
| `iT` | Bool | 是否检测到面部 |
| `xs` | Bool（`true`） | NekoDice 模式标识 |
| `uuid` | String | 设备唯一标识 |

### 3.2 完整示例

```json
{
  "hP": "5.231",
  "hY": "-12.480",
  "hR": "1.022",
  "bX": "-6.240",
  "bY": "2.616",
  "bZ": "0.511",
  "bMX": "0.120",
  "bMY": "-0.030",
  "bMZ": "0.000",
  "eLO": "0.870",
  "eRO": "0.910",
  "eLS": "0.120",
  "eRS": "0.130",
  "eBYL": "0.250",
  "eBYR": "0.260",
  "eBLF": "0.000",
  "eBRF": "0.000",
  "eBAL": "0.100",
  "eBAR": "0.100",
  "eX": "-0.230",
  "eY": "0.110",
  "mO": "0.180",
  "mF": "0.450",
  "mU": "0.000",
  "mT": "0.000",
  "mX": "0.020",
  "mPU": "0.000",
  "mP": "0.040",
  "mC": "0.000",
  "iT": true,
  "xs": true,
  "uuid": "B1F2C3D4-E5F6-4A7B-8C9D-0123456789AB"
}
```

---

## 四、CuteNova 模式

### 4.1 字段对照

#### 眼部

| Short | ARKit BlendShape | 含义 |
| --- | --- | --- |
| `eBL` / `eBR` | `eyeBlinkLeft` / `eyeBlinkRight` | 眨眼 |
| `eLUL` / `eLUR` | `eyeLookUpLeft` / `eyeLookUpRight` | 向上看 |
| `eLDL` / `eLDR` | `eyeLookDownLeft` / `eyeLookDownRight` | 向下看 |
| `eLIL` / `eLIR` | `eyeLookInLeft` / `eyeLookInRight` | 向内看 |
| `eLOL` / `eLOR` | `eyeLookOutLeft` / `eyeLookOutRight` | 向外看 |
| `eWL` / `eWR` | `eyeWideLeft` / `eyeWideRight` | 睁大 |
| `eSL` / `eSR` | `eyeSquintLeft` / `eyeSquintRight` | 眯眼 |

#### 嘴部

| Short | ARKit BlendShape | 含义 |
| --- | --- | --- |
| `mL` / `mR` | `mouthLeft` / `mouthRight` | 嘴左 / 右移 |
| `mSL` / `mSR` | `mouthSmileLeft/Right` | 微笑 |
| `mFL` / `mFR` | `mouthFrownLeft/Right` | 撇嘴 |
| `mPL` / `mPR` | `mouthPressLeft/Right` | 抿嘴 |
| `mUUL` / `mUUR` | `mouthUpperUpLeft/Right` | 上唇上扬 |
| `mLDL` / `mLDR` | `mouthLowerDownLeft/Right` | 下唇下拉 |
| `mSTL` / `mSTR` | `mouthStretchLeft/Right` | 嘴角拉伸 |
| `mDL` / `mDR` | `mouthDimpleLeft/Right` | 酒窝 |
| `mC` | `mouthClose` | 闭嘴 |
| `mF` | `mouthFunnel` | 漏斗形 |
| `mP` | `mouthPucker` | 撅嘴 |
| `mSSU` / `mSSL` | `mouthShrugUpper/Lower` | 上 / 下唇耸起 |
| `mRU` / `mRL` | `mouthRollUpper/Lower` | 上 / 下唇内卷 |

#### 下颌 / 鼻 / 颊 / 舌 / 眉

| Short | ARKit BlendShape | 含义 |
| --- | --- | --- |
| `jO` / `jF` / `jL` / `jR` | `jawOpen` / `jawForward` / `jawLeft` / `jawRight` | 下颌 |
| `nSL` / `nSR` | `noseSneerLeft/Right` | 皱鼻 |
| `cP` | `cheekPuff` | 鼓腮 |
| `cSL` / `cSR` | `cheekSquintLeft/Right` | 颊紧 |
| `tO` | `tongueOut` | 吐舌 |
| `bDL` / `bDR` | `browDownLeft/Right` | 眉下压 |
| `bOUL` / `bOUR` | `browOuterUpLeft/Right` | 眉外上扬 |
| `bIU` | `browInnerUp` | 眉内上扬 |

#### 头部 / 位置 / 视线 / 状态

| Short | 含义 |
| --- | --- |
| `R` / `Y` / `P` | Roll / Yaw / Pitch |
| `PX` / `PY` / `PZ` | 头部位置 |
| `d` | 设备到面部距离 |
| `eX` / `eY` | 视线方向，-1–1 |
| `iT` | 是否追踪到面部 |

> 当 `iT` 缺失（值为 `nil`）时兜底为 `true`；BlendShape / 头部数值若缺失则
> 兜底为 `0`，避免向 JSON 字典插入 `nil` 导致崩溃。

### 4.2 完整示例

```json
{
  "iT": true,

  "eBL": 0.13, "eLUL": 0.00, "eLDL": 0.02, "eLIL": 0.10, "eLOL": 0.00,
  "eWL": 0.05, "eSL": 0.01,
  "eBR": 0.09, "eLUR": 0.00, "eLDR": 0.02, "eLIR": 0.10, "eLOR": 0.00,
  "eWR": 0.05, "eSR": 0.01,

  "mL": 0.02, "mSL": 0.12, "mFL": 0.00, "mPL": 0.00,
  "mUUL": 0.04, "mLDL": 0.03, "mSTL": 0.00, "mDL": 0.01,
  "mR": 0.02, "mSR": 0.13, "mFR": 0.00, "mPR": 0.00,
  "mUUR": 0.04, "mLDR": 0.03, "mSTR": 0.00, "mDR": 0.01,

  "mC": 0.00, "mF": 0.00, "mP": 0.00,
  "mSSU": 0.00, "mSSL": 0.00, "mRU": 0.00, "mRL": 0.00,

  "jO": 0.18, "jF": 0.00, "jL": 0.00, "jR": 0.00,

  "nSL": 0.00, "nSR": 0.00,
  "cP": 0.00, "cSL": 0.00, "cSR": 0.00,
  "tO": 0.00,

  "bDL": 0.00, "bOUL": 0.20,
  "bDR": 0.00, "bOUR": 0.20,
  "bIU": 0.05,

  "R": 1.022, "Y": -12.480, "P": 5.231,
  "PX": 0.012, "PY": -0.034, "PZ": -0.420,
  "d": 0.420,

  "eX": -0.230, "eY": 0.110
}
```

---

## 五、Raw 模式

### 5.1 字段说明

| Key | 类型 | 含义 |
| --- | --- | --- |
| `isTracked` | Bool | 是否检测到面部 |
| `blendShapes` | Object<String, Number> | **完整** ARKit BlendShape 字典，键为 ARKit 官方名（如 `eyeBlinkLeft`、`jawOpen`），值为 0–1 浮点 |
| `headPosition` | Object `{x, y, z}` | 头部位置；若启用「归中」则减去基线 `centerPositionX/Y/Z` |
| `transform` | Number[16] | 头部 4×4 变换矩阵（ARKit `ARFaceAnchor.transform`），按行展开：第 1 行为各列的 `.x`、第 2 行为 `.y`、第 3 行为 `.z`、第 4 行为 `.w` |
| `leftEyeTransform` | Number[16] | 左眼 4×4 变换矩阵，展开方式同 `transform` |
| `rightEyeTransform` | Number[16] | 右眼 4×4 变换矩阵 |
| `lookAtPoint` | Object `{x, y, z}` | ARKit 注视点（人脸坐标系，米） |

> 与 NekoDice / CuteNova 不同，Raw 模式**不发送** `uuid`、`xs`、`iT`；判断当前模式可以靠 `transform` / `headPosition` 等字段是否存在。

#### `transform` 展开顺序

源代码中按以下顺序写入（`t.columns[c]` 表示第 c 列）：

```
[ c0.x, c1.x, c2.x, c3.x,
  c0.y, c1.y, c2.y, c3.y,
  c0.z, c1.z, c2.z, c3.z,
  c0.w, c1.w, c2.w, c3.w ]
```

即 **行主序的 4×4 矩阵**（数组中前 4 个数是矩阵第一行）。`leftEyeTransform`
/ `rightEyeTransform` 完全一致。

### 5.2 完整示例

```json
{
  "isTracked": true,
  "blendShapes": {
    "eyeBlinkLeft": 0.13,
    "eyeBlinkRight": 0.09,
    "eyeLookUpLeft": 0.00,
    "eyeLookUpRight": 0.00,
    "eyeLookDownLeft": 0.02,
    "eyeLookDownRight": 0.02,
    "eyeLookInLeft": 0.10,
    "eyeLookInRight": 0.10,
    "eyeLookOutLeft": 0.00,
    "eyeLookOutRight": 0.00,
    "eyeWideLeft": 0.05,
    "eyeWideRight": 0.05,
    "eyeSquintLeft": 0.01,
    "eyeSquintRight": 0.01,
    "mouthLeft": 0.02,
    "mouthRight": 0.02,
    "mouthSmileLeft": 0.12,
    "mouthSmileRight": 0.13,
    "mouthFrownLeft": 0.00,
    "mouthFrownRight": 0.00,
    "mouthPressLeft": 0.00,
    "mouthPressRight": 0.00,
    "mouthUpperUpLeft": 0.04,
    "mouthUpperUpRight": 0.04,
    "mouthLowerDownLeft": 0.03,
    "mouthLowerDownRight": 0.03,
    "mouthStretchLeft": 0.00,
    "mouthStretchRight": 0.00,
    "mouthDimpleLeft": 0.01,
    "mouthDimpleRight": 0.01,
    "mouthClose": 0.00,
    "mouthFunnel": 0.00,
    "mouthPucker": 0.00,
    "mouthShrugUpper": 0.00,
    "mouthShrugLower": 0.00,
    "mouthRollUpper": 0.00,
    "mouthRollLower": 0.00,
    "jawOpen": 0.18,
    "jawForward": 0.00,
    "jawLeft": 0.00,
    "jawRight": 0.00,
    "noseSneerLeft": 0.00,
    "noseSneerRight": 0.00,
    "cheekPuff": 0.00,
    "cheekSquintLeft": 0.00,
    "cheekSquintRight": 0.00,
    "tongueOut": 0.00,
    "browDownLeft": 0.00,
    "browDownRight": 0.00,
    "browOuterUpLeft": 0.20,
    "browOuterUpRight": 0.20,
    "browInnerUp": 0.05
  },
  "headPosition": { "x": 0.012, "y": -0.034, "z": -0.420 },
  "transform": [
    0.9781, -0.0213,  0.2065,  0.012,
    0.0185,  0.9996,  0.0212, -0.034,
   -0.2068, -0.0184,  0.9782, -0.420,
    0.0000,  0.0000,  0.0000,  1.000
  ],
  "leftEyeTransform": [
    0.9988, -0.0150,  0.0463,  0.032,
    0.0156,  0.9998, -0.0123,  0.018,
   -0.0461,  0.0130,  0.9988, -0.410,
    0.0000,  0.0000,  0.0000,  1.000
  ],
  "rightEyeTransform": [
    0.9988,  0.0150, -0.0463, -0.032,
   -0.0156,  0.9998,  0.0123,  0.018,
    0.0461, -0.0130,  0.9988, -0.410,
    0.0000,  0.0000,  0.0000,  1.000
  ],
  "lookAtPoint": { "x": -0.020, "y": 0.005, "z": -0.600 }
}
```

---
