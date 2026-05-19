# Shiori — Project Introduction

## 1. Overview

**Shiori** is an iOS face-tracking application built on the **ARKit**. It captures the 52 ARKit `BlendShape` values, head pose
and eye gaze from the TrueDepth front camera of an iPhone / iPad, converts
them into parameters consumable by Live2D models, and streams them in real
time over TCP / WebSocket to desktop avatar or live-streaming software on a
PC — effectively turning the phone into a dedicated face-capture transmitter.

---

## 2. Packet overview

Every frame (target 60 Hz) Shiori sends one **UTF-8 JSON text** payload, with
**no length prefix and no framing delimiter**.

- **WebSocket**: one message equals one complete JSON document.
- **TCP**: the receiver must split JSON documents itself (e.g. by matching
  `{` / `}` or by newline).

Shiori exposes three switchable packet formats via the "Data Mode" button:

| Mode | Summary | Use case |
| --- | --- | --- | --- |
| **NekoDice** | Live2D 2D channel mapping; short keys, string-encoded floats | Standard Live2D models |
| **CuteNova** | 52 ARKit BlendShape short keys + head 6-DoF | Perfect Sync models / full-face detail |
| **Raw** | Raw ARKit field names, 4×4 transforms, eye transforms, lookAt point | Custom post-processing / debugging |

> The three modes use **disjoint** key sets / structures. A host can sniff a
> distinguishing field (e.g. `xs` / `transform` / `headPitch`) to identify the
> incoming mode.

Numeric precision:
- **NekoDice**: string-encoded floats, 3 decimal places; `isTracked` is Bool;
  `xs` is fixed to `true`.
- **CuteNova**: JSON numbers (BlendShapes are 0–1 floats; head angles /
  positions are floats); `iT` is Bool.
- **Raw**: JSON numbers; `isTracked` is Bool.

---

## 3. NekoDice 

### 3.1 Fields

| Key | Type | Meaning |
| --- | --- | --- |
| `hP` / `hY` / `hR` | String(float) | Head pitch / yaw / roll |
| `bX` / `bY` / `bZ` | String(float) | Body angle (follows the head) |
| `bMX` / `bMY` / `bMZ` | String(float) | Body translation |
| `eLO` / `eRO` | String(float) | Left / right eye openness, 0–1 |
| `eLS` / `eRS` | String(float) | Left / right eye smile, 0–1 |
| `eBYL` / `eBYR` | String(float) | Brow vertical |
| `eBLF` / `eBRF` | String(float) | Brow shape |
| `eBAL` / `eBAR` | String(float) | Brow angle |
| `eX` / `eY` | String(float) | Gaze, -1–1 |
| `mO` | String(float) | Mouth open |
| `mF` | String(float) | Mouth form |
| `mU` / `mT` / `mX` | String(float) | Mouth deformation U / T / X |
| `mPU` / `mP` / `mC` | String(float) | Pucker / puff / crook |
| `iT` | Bool | Whether a face is detected |
| `xs` | Bool (`true`) | NekoDice mode marker |
| `uuid` | String | Device identifier |

### 3.2 Full example

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

## 4. CuteNova

### 4.1 Key mapping

#### Eyes

| Short | ARKit BlendShape | Meaning |
| --- | --- | --- |
| `eBL` / `eBR` | `eyeBlinkLeft` / `eyeBlinkRight` | Blink |
| `eLUL` / `eLUR` | `eyeLookUpLeft` / `eyeLookUpRight` | Look up |
| `eLDL` / `eLDR` | `eyeLookDownLeft` / `eyeLookDownRight` | Look down |
| `eLIL` / `eLIR` | `eyeLookInLeft` / `eyeLookInRight` | Look in |
| `eLOL` / `eLOR` | `eyeLookOutLeft` / `eyeLookOutRight` | Look out |
| `eWL` / `eWR` | `eyeWideLeft` / `eyeWideRight` | Wide |
| `eSL` / `eSR` | `eyeSquintLeft` / `eyeSquintRight` | Squint |

#### Mouth

| Short | ARKit BlendShape | Meaning |
| --- | --- | --- |
| `mL` / `mR` | `mouthLeft` / `mouthRight` | Mouth shift |
| `mSL` / `mSR` | `mouthSmileLeft/Right` | Smile |
| `mFL` / `mFR` | `mouthFrownLeft/Right` | Frown |
| `mPL` / `mPR` | `mouthPressLeft/Right` | Press |
| `mUUL` / `mUUR` | `mouthUpperUpLeft/Right` | Upper-lip up |
| `mLDL` / `mLDR` | `mouthLowerDownLeft/Right` | Lower-lip down |
| `mSTL` / `mSTR` | `mouthStretchLeft/Right` | Stretch |
| `mDL` / `mDR` | `mouthDimpleLeft/Right` | Dimple |
| `mC` | `mouthClose` | Close |
| `mF` | `mouthFunnel` | Funnel |
| `mP` | `mouthPucker` | Pucker |
| `mSSU` / `mSSL` | `mouthShrugUpper/Lower` | Shrug upper / lower |
| `mRU` / `mRL` | `mouthRollUpper/Lower` | Roll upper / lower |

#### Jaw / nose / cheek / tongue / brow

| Short | ARKit BlendShape | Meaning |
| --- | --- | --- |
| `jO` / `jF` / `jL` / `jR` | `jawOpen` / `jawForward` / `jawLeft` / `jawRight` | Jaw |
| `nSL` / `nSR` | `noseSneerLeft/Right` | Nose sneer |
| `cP` | `cheekPuff` | Cheek puff |
| `cSL` / `cSR` | `cheekSquintLeft/Right` | Cheek squint |
| `tO` | `tongueOut` | Tongue out |
| `bDL` / `bDR` | `browDownLeft/Right` | Brow down |
| `bOUL` / `bOUR` | `browOuterUpLeft/Right` | Brow outer up |
| `bIU` | `browInnerUp` | Brow inner up |

#### Head / position / gaze / status

| Short | Meaning |
| --- | --- |
| `R` / `Y` / `P` | Roll / yaw / pitch |
| `PX` / `PY` / `PZ` | Head position |
| `d` | Device-to-face distance |
| `eX` / `eY` | Gaze direction, -1–1 |
| `iT` | Whether a face is detected |

> When `iT` is missing (the underlying value is `nil`) it falls back to
> `true`. Missing BlendShape / head values fall back to `0`, avoiding
> `nil`-insertion crashes during JSON serialization.

### 4.2 Full example

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

## 5. Raw mode 

### 5.1 Fields

| Key | Type | Meaning |
| --- | --- | --- |
| `isTracked` | Bool | Whether a face is detected |
| `blendShapes` | Object<String, Number> | **Full** ARKit BlendShape dictionary, keyed by the ARKit official name (e.g. `eyeBlinkLeft`, `jawOpen`); values are 0–1 floats |
| `headPosition` | Object `{x, y, z}` | Head position; if centering is enabled, `centerPositionX/Y/Z` is subtracted |
| `transform` | Number[16] | Head 4×4 transform (`ARFaceAnchor.transform`), flattened row-wise: row 1 = `.x` of each column, row 2 = `.y`, row 3 = `.z`, row 4 = `.w` |
| `leftEyeTransform` | Number[16] | Left-eye 4×4 transform, same layout as `transform` |
| `rightEyeTransform` | Number[16] | Right-eye 4×4 transform |
| `lookAtPoint` | Object `{x, y, z}` | ARKit look-at point (face coordinate system, meters) |

> Unlike NekoDice / CuteNova, Raw **does not send** `uuid`, `xs` or `iT`. To
> detect the mode, check for fields such as `transform` / `headPosition`.

#### `transform` layout

The source writes the array in this order (where `t.columns[c]` is column
`c`):

```
[ c0.x, c1.x, c2.x, c3.x,
  c0.y, c1.y, c2.y, c3.y,
  c0.z, c1.z, c2.z, c3.z,
  c0.w, c1.w, c2.w, c3.w ]
```

i.e. the **row-major flattening** of a 4×4 matrix (the first four numbers are
the first row). `leftEyeTransform` / `rightEyeTransform` use the same layout.

### 5.2 Full example

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
