# TK-FCP026BK 自作キーボード化プロジェクト 経過まとめ

---

## 1. プロジェクト概要と戦略

### 目的
市販キーボード TK-FCP026BK を分解し、キーボード部分のプリント基板を取り出して、XIAO nRF52840 + MCP23017 を使った自作キーボード（BLE対応）として再利用する。

### TK-FCP026BK のキー数
日本語86キー

### FFCの構成
キーボード基板から出ている FFC は **16本 + 8本 = 合計24本**。
- 16本 → Row（行）ライン
- 8本 → Column（列）ライン
- 16 × 8 = 最大128キー（86キーをカバー）

### ピン数の問題と解決策
XIAO nRF52840 の GPIO は **11本**（D4/D5はI2C専用で使用不可のため実質9本）であり、24本には足りない。
→ **MCP23017（I2Cポートエクスパンダー）1個**で解決。

### 最終的な構成
```
XIAO nRF52840
├── D0(P0.02), D1(P0.03), D2(P0.28), D3(P0.29)  → Col 0〜3
├── D6(P1.11)                                     → CapsLock LED
├── D7(P1.12), D8(P1.13), D9(P1.14), D10(P1.15) → Col 4〜7
├── NFC2(P0.10)                                   → NumLock LED（NFCピン転用）
├── D4(P0.04) = SDA → MCP23017 I2C
└── D5(P0.05) = SCL → MCP23017 I2C

MCP23017（I2Cアドレス 0x20）
├── GPA0〜GPA7（pin21〜28） → Row 0〜7
└── GPB0〜GPB7（pin1〜8）  → Row 8〜15
```

**D4・D5はI2C専用のためFFC接続不可。D6はCapsLock LED用。NFC2(P0.10)はNumLock LED用（GPIOに転用）。**
D7〜D10の4本をCol 4〜7として使用。（D7はP1.12、D8はP1.13、D9はP1.14、D10はP1.15）

### ファームウェアの選択
- QMK → **使用不可**（nRF52840を正式サポートしていない）
- **ZMK → 採用**（nRF52840・BLE・USB HID すべて対応）

---

## 2. ブレッドボード接続（キーマトリクス解析用）

### 必要部品

- XIAO nRF52840
- MCP23017（28ピンDIPパッケージ）
- FFC/FPC変換基板 18ピン※（1.0mmピッチ。上接触 or 下接触は現物確認） ** ※「FFCケーブルの物理構成」を参照 **
- FFC/FPC変換基板 8ピン（1.0mmピッチ。上接触 or 下接触は現物確認）
- ブレッドボード
- ジャンパーワイヤー
- 4.7kΩ抵抗 × 2本（I2Cプルアップ用）
- 100nF積層セラミックコンデンサ × 1個（ノイズ対策・推奨）
- TK-FCP026BK のプリント基板

#### FFCケーブルの物理構成
TK-FCP026BK のキーボード基板から出ているFFCケーブルは **2本** に分かれている。
Col側（8ピン）はそのまま8ピン変換基板に接続してFFC01〜08（Col0〜Col7）として扱う。
Row側（18ピン）は9番と18番がダミーで実効16ピン。18ピン変換基板に接続して、1〜8番をFFC09〜16（Row0〜Row7）、10〜17番をFFC17〜24（Row8〜Row15）として扱う。9番、16番は結線しない。

### MCP23017 正しいピン配置（データシートより）

```
         MCP23017 (DIP-28)
         ┌──────────────┐
GPB0  1 ─┤              ├─ 28  GPA7
GPB1  2 ─┤              ├─ 27  GPA6
GPB2  3 ─┤              ├─ 26  GPA5
GPB3  4 ─┤              ├─ 25  GPA4
GPB4  5 ─┤              ├─ 24  GPA3
GPB5  6 ─┤              ├─ 23  GPA2
GPB6  7 ─┤              ├─ 22  GPA1
GPB7  8 ─┤              ├─ 21  GPA0
VDD   9 ─┤              ├─ 20  INTA
VSS  10 ─┤              ├─ 19  INTB
 NC  11 ─┤              ├─ 18  RESET
SCL  12 ─┤              ├─ 17  A2
SDA  13 ─┤              ├─ 16  A1
 NC  14 ─┤              ├─ 15  A0
         └──────────────┘
```

### 接続リスト（完全版・テキスト形式）

#### 電源系
| From | To | 備考 |
|---|---|---|
| XIAO `3V3` | ブレッドボード `+`レール | 3.3V供給 |
| XIAO `GND` | ブレッドボード `−`レール | GND供給 |

#### MCP23017 電源・制御ピン
| From | To | 備考 |
|---|---|---|
| ブレッドボード `+`レール | MCP23017 `pin9`（VDD） | IC電源（1か所のみ） |
| ブレッドボード `−`レール | MCP23017 `pin10`（VSS） | GND |
| ブレッドボード `+`レール | MCP23017 `pin18`（RESET） | 常時HIGH必須 |
| ブレッドボード `−`レール | MCP23017 `pin15`（A0） | アドレスbit0 = 0 |
| ブレッドボード `−`レール | MCP23017 `pin16`（A1） | アドレスbit1 = 0 |
| ブレッドボード `−`レール | MCP23017 `pin17`（A2） | アドレスbit2 = 0 → I2Cアドレス **0x20** |

#### I2C通信線
| From | To | 備考 |
|---|---|---|
| XIAO `D4`（P0.04 / SDA） | MCP23017 `pin13`（SDA） | |
| XIAO `D5`（P0.05 / SCL） | MCP23017 `pin12`（SCL） | |

#### プルアップ抵抗（4.7kΩ × 2本）
| From | 経由 | To | 備考 |
|---|---|---|---|
| ブレッドボード `+`レール | 4.7kΩ | MCP23017 `pin13`（SDA） | SDAプルアップ |
| ブレッドボード `+`レール | 4.7kΩ | MCP23017 `pin12`（SCL） | SCLプルアップ |

#### バイパスコンデンサ（100nF・推奨）
| From | To | 備考 |
|---|---|---|
| MCP23017 `pin9`（VDD）直近 | MCP23017 `pin10`（VSS）直近 | ICのすぐ脇に配置 |

#### FFC接続（確定版・マトリクス解析および最終ZMK構成）
| FFC番号 | 接続先 | ZMKでの役割 |
|---|---|---|
| FFC01 | XIAO D0（P0.02） | Col0 |
| FFC02 | XIAO D1（P0.03） | Col1 |
| FFC03 | XIAO D2（P0.28） | Col2 |
| FFC04 | XIAO D3（P0.29） | Col3 |
| FFC05 | XIAO D7（P1.12） | Col4 |
| FFC06 | XIAO D8（P1.13） | Col5 |
| FFC07 | XIAO D9（P1.14） | Col6 |
| FFC08 | XIAO D10（P1.15） | Col7 |
| FFC09 | MCP23017 GPA0（pin21） | Row0 |
| FFC10 | MCP23017 GPA1（pin22） | Row1 |
| FFC11 | MCP23017 GPA2（pin23） | Row2 |
| FFC12 | MCP23017 GPA3（pin24） | Row3 |
| FFC13 | MCP23017 GPA4（pin25） | Row4 |
| FFC14 | MCP23017 GPA5（pin26） | Row5 |
| FFC15 | MCP23017 GPA6（pin27） | Row6 |
| FFC16 | MCP23017 GPA7（pin28） | Row7 |
| FFC17 | MCP23017 GPB0（pin1） | Row8 |
| FFC18 | MCP23017 GPB1（pin2） | Row9 |
| FFC19 | MCP23017 GPB2（pin3） | Row10 |
| FFC20 | MCP23017 GPB3（pin4） | Row11 |
| FFC21 | MCP23017 GPB4（pin5） | Row12 |
| FFC22 | MCP23017 GPB5（pin6） | Row13 |
| FFC23 | MCP23017 GPB6（pin7） | Row14 |
| FFC24 | MCP23017 GPB7（pin8） | Row15 |

※当初のマトリクス解析時はFFC01〜08をRow・FFC17〜24をColとして接続していたが、GPA連続入力問題の解決のため配線を変更した。

#### LED接続（v10追加）
| From | 経由 | To | 備考 |
|---|---|---|---|
| XIAO `D6`（P1.11） | 330Ω抵抗 | CapsLock LED（アノード）→ カソード → GND | CapsLock状態表示 |
| XIAO `NFC2`（P0.10） | 330Ω抵抗 | NumLock LED（アノード）→ カソード → GND | NumLock状態表示 |

※GPIO_ACTIVE_HIGH（HIGH=点灯）。抵抗値は3.3V・赤色LEDなら330Ωで約4mA。省電力を重視する場合は1kΩ（約1mA）でも視認可。

#### 未接続ピン（MCP23017）
| ピン | 名称 | 処置 |
|---|---|---|
| pin11 | NC | 未接続でOK |
| pin14 | NC | 未接続でOK |
| pin19 | INTB | 未接続でOK |
| pin20 | INTA | 未接続でOK |

---

## 3. 接続確認プログラムと実行方法

### PC環境のセットアップ

**1. Arduino IDE 2.x をインストール**
https://www.arduino.cc/en/software

**2. XIAO nRF52840 のボードパッケージを追加**
`ファイル → 環境設定` の「追加のボードマネージャのURL」に追加：
```
https://files.seeedstudio.com/arduino/package_seeeduino_boards_index.json
```

**3. ボードマネージャからインストール**
`ツール → ボードマネージャ` で `Seeed nRF52` を検索してインストール。

**4. ボード選択**
`ツール → ボード → Seeed nRF52 Boards → Seeed XIAO nRF52840`

**5. ライブラリのインストール**
`ツール → ライブラリを管理` で `Adafruit MCP23X17` を検索してインストール。
（依存ライブラリを聞かれたら「Install all」を選択）

---

### ① I2Cスキャン（最初に実行）

MCP23017 が 0x20 として認識されるか確認する。

```cpp
#include <Wire.h>

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);

  Serial.println("=== I2C Scanner ===");
  Wire.begin();
  delay(500);

  byte found = 0;
  for (byte addr = 0x01; addr < 0x7F; addr++) {
    Wire.beginTransmission(addr);
    byte err = Wire.endTransmission();
    if (err == 0) {
      Serial.print("Device found at 0x");
      if (addr < 16) Serial.print("0");
      Serial.println(addr, HEX);
      found++;
    }
  }

  if (found == 0) {
    Serial.println("No I2C devices found.");
    Serial.println("→ 配線を確認してください");
  } else {
    Serial.print(found);
    Serial.println(" device(s) found.");
  }
}

void loop() {}
```

**期待する出力：**
```
=== I2C Scanner ===
Device found at 0x20
1 device(s) found.
```

---

### ② MCP23017 GPIO テスト

全16ピンの入出力が正常かを確認する。

```cpp
#include <Wire.h>
#include <Adafruit_MCP23X17.h>

Adafruit_MCP23X17 mcp;

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);

  Serial.println("=== MCP23017 GPIO Test ===");

  if (!mcp.begin_I2C(0x20)) {
    Serial.println("ERROR: MCP23017 not found at 0x20");
    while (1) delay(100);
  }
  Serial.println("MCP23017 OK");

  for (int i = 0; i < 8; i++)  mcp.pinMode(i, INPUT_PULLUP);  // GPA0-7
  for (int i = 8; i < 16; i++) mcp.pinMode(i, INPUT_PULLUP);  // GPB0-7

  Serial.println("All pins set to INPUT_PULLUP");
  Serial.println("--- GPA/GPBピンをGNDに触れると0に変わります ---");
}

void loop() {
  uint8_t gpa = mcp.readGPIOA();
  uint8_t gpb = mcp.readGPIOB();

  Serial.print("GPA(pin21-28): 0b");
  for (int i = 7; i >= 0; i--) Serial.print((gpa >> i) & 1);

  Serial.print("  GPB(pin1-8): 0b");
  for (int i = 7; i >= 0; i--) Serial.print((gpb >> i) & 1);

  Serial.println();
  delay(500);
}
```

**確認方法：** 全ビットが `1` で表示されたあと、MCP23017 の任意ピンをジャンパーワイヤーで GND レールに触れると該当ビットが `0` に変わることを確認する。

---

## 4. キーマップ確認用プログラム

### マトリクススキャンコード

FFC 24本すべてをスキャンし、押したキーに対応するピアをシリアルモニタに出力する。

```cpp
#include <Wire.h>
#include <Adafruit_MCP23X17.h>

Adafruit_MCP23X17 mcp;

// XIAO に直結する8本のピン（D4/D5はI2C専用のため除外、D6は未使用、D7〜D10使用）
const int XIAO_PINS[] = {D0, D1, D2, D3, D7, D8, D9, D10};
const int XIAO_COUNT = 8;
const int MCP_COUNT  = 16;
const int TOTAL      = XIAO_COUNT + MCP_COUNT; // 24本

const char* PIN_NAMES[] = {
  "FFC01(D0)",   "FFC02(D1)",   "FFC03(D2)",   "FFC04(D3)",
  "FFC05(D7)",   "FFC06(D8)",   "FFC07(D9)",   "FFC08(D10)",
  "FFC09(GPA0)", "FFC10(GPA1)", "FFC11(GPA2)", "FFC12(GPA3)",
  "FFC13(GPA4)", "FFC14(GPA5)", "FFC15(GPA6)", "FFC16(GPA7)",
  "FFC17(GPB0)", "FFC18(GPB1)", "FFC19(GPB2)", "FFC20(GPB3)",
  "FFC21(GPB4)", "FFC22(GPB5)", "FFC23(GPB6)", "FFC24(GPB7)"
};

void allPinsInput() {
  for (int i = 0; i < XIAO_COUNT; i++) pinMode(XIAO_PINS[i], INPUT_PULLUP);
  for (int i = 0; i < MCP_COUNT; i++)  mcp.pinMode(i, INPUT_PULLUP);
}

void drivePin(int index, bool active) {
  if (index < XIAO_COUNT) {
    pinMode(XIAO_PINS[index], active ? OUTPUT : INPUT_PULLUP);
    if (active) digitalWrite(XIAO_PINS[index], LOW);
  } else {
    int mp = index - XIAO_COUNT;
    mcp.pinMode(mp, active ? OUTPUT : INPUT_PULLUP);
    if (active) mcp.digitalWrite(mp, LOW);
  }
}

bool readPin(int index) {
  if (index < XIAO_COUNT) return digitalRead(XIAO_PINS[index]) == LOW;
  return mcp.digitalRead(index - XIAO_COUNT) == LOW;
}

bool detected[24][24] = {};

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);
  if (!mcp.begin_I2C(0x20)) {
    Serial.println("ERROR: MCP23017 not found");
    while(1);
  }
  allPinsInput();
  Serial.println("=== Matrix Scanner ===");
  Serial.println("キーを1つ押し続けると検出されます。'r'でリセット。");
  Serial.println("=====================");
}

void loop() {
  if (Serial.available()) {
    char c = Serial.read();
    if (c == 'r' || c == 'R') {
      memset(detected, 0, sizeof(detected));
      Serial.println("--- リセット ---");
    }
  }

  for (int drive = 0; drive < TOTAL; drive++) {
    drivePin(drive, true);
    delayMicroseconds(50);
    for (int sense = 0; sense < TOTAL; sense++) {
      if (sense == drive) continue;
      if (readPin(sense)) {
        int a = min(drive, sense);
        int b = max(drive, sense);
        if (!detected[a][b]) {
          detected[a][b] = true;
          Serial.print("KEY: ");
          Serial.print(PIN_NAMES[drive]);
          Serial.print(" <-> ");
          Serial.println(PIN_NAMES[sense]);
        }
      }
    }
    drivePin(drive, false);
    delay(1);
  }
}
```

**実行方法：**
1. スケッチを XIAO nRF52840 に書き込む
2. シリアルモニタを開く（115200bps）
3. キーを1つ押し続けると `KEY: FFC03(D2) <-> FFC19(GPB2)` のように表示される
4. シリアルモニタ上で `r` を送信するとリセット
5. 全キーを押して記録する

---

### スキャンで確認された86キーのマトリクス

（Col = FFC01〜FFC08（XIAO D0-D3,D7-D10）、Row = FFC09〜FFC24（MCP23017 GPA/GPB）。✅=キーあり、ー=キーなし）

| Row | D0(FFC01) | D1(FFC02) | D2(FFC03) | D3(FFC04) | D7(FFC05) | D8(FFC06) | D9(FFC07) | D10(FFC08) |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Row0** GPA0(FFC09) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Row1** GPA1(FFC10) | ✅ | ✅ | ✅ | ー | ✅ | ✅ | ✅ | ✅ |
| **Row2** GPA2(FFC11) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Row3** GPA3(FFC12) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Row4** GPA4(FFC13) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Row5** GPA5(FFC14) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Row6** GPA6(FFC15) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Row7** GPA7(FFC16) | ー | ー | ー | ✅ | ー | ✅ | ー | ー |
| **Row8** GPB0(FFC17) | ー | ー | ー | ー | ー | ✅ | ー | ー |
| **Row9** GPB1(FFC18) | ー | ー | ー | ✅ | ー | ✅ | ✅ | ー |
| **Row10** GPB2(FFC19) | ー | ✅ | ✅ | ー | ー | ー | ー | ー |
| **Row11** GPB3(FFC20) | ✅ | ✅ | ー | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Row12** GPB4(FFC21) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Row13** GPB5(FFC22) | ✅ | ー | ー | ✅ | ー | ✅ | ー | ✅ |
| **Row14** GPB6(FFC23) | ✅ | ー | ー | ー | ー | ー | ✅ | ✅ |
| **Row15** GPB7(FFC24) | ー | ✅ | ー | ー | ー | ー | ー | ー |

合計 **86キー** 確認済み（異常ペアなし）。

---

## 5. ZMK GitHub ビルド用ソース

### フォルダ構成

```
zmk-tk-fcp026bk/          ← GitHubリポジトリのルート
├── .github/
│   └── workflows/
│       └── build.yml
├── config/
│   ├── west.yml
│   └── boards/
│       └── shields/
│           └── tk_fcp026bk/
│               ├── Kconfig.shield
│               ├── Kconfig.defconfig
│               ├── tk_fcp026bk.overlay
│               └── keymaps/
│                   └── tk_fcp026bk.keymap
└── build.yaml
```

---

### `.github/workflows/build.yml`

```yaml
name: Build ZMK Firmware

on:
  push:
  pull_request:
  workflow_dispatch:

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

---

### `build.yaml`（リポジトリルートに配置）

```yaml
---
include:
  - board: xiao_ble//zmk
    shield: tk_fcp026bk
```

> ボード名は `xiao_ble//zmk` が正しい（ZMK 2025年12月以降の新仕様）。

---

### `config/west.yml`

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
  self:
    path: config
```

---

### `config/boards/shields/tk_fcp026bk/Kconfig.shield`

```kconfig
config SHIELD_TK_FCP026BK
    def_bool $(shields_list_contains,tk_fcp026bk)
```

---

### `config/boards/shields/tk_fcp026bk/Kconfig.defconfig`

> v10確定版。CapsLock / NumLock LEDインジケーター対応。

```kconfig
if SHIELD_TK_FCP026BK

config ZMK_KEYBOARD_NAME
    default "TK-FCP026BK"

config ZMK_USB
    default y

config ZMK_BLE
    default y

# I2C（MCP23017接続用）
config I2C
    default y

# GPIO expander（MCP23017）
config GPIO_MCP230XX
    default y

# バッテリー残量レポート（Windows側でパーセント表示）
config ZMK_BATTERY_REPORTING
    default y

# HIDインジケーター（CapsLock / NumLock LED状態をHIDで受け取るために必要）
# ZMK_INDICATOR_LEDS はoverlayに zmk,indicator-leds ノードがあれば自動で有効になる
# → Kconfig.defconfigへの記述は不要（ZMK_LED_INDICATORSというシンボルは存在しない）
config ZMK_HID_INDICATORS
    default y

endif
```

---

### `config/boards/shields/tk_fcp026bk/tk_fcp026bk.overlay`

> v10確定版。CapsLock LED（D6=P1.11）・NumLock LED（NFC2=P0.10）追加。

```dts
/*
 * TK-FCP026BK Shield Overlay v10
 * XIAO nRF52840 + MCP23017
 *
 * 配線:
 *   Col 0-7 : XIAO GPIO D0-D3, D7-D10  ※D4=I2C(SDA), D5=I2C(SCL), D6=CapsLock LED
 *             D0=Col0, D1=Col1, D2=Col2, D3=Col3,
 *             D7=Col4, D8=Col5, D9=Col6, D10=Col7
 *   Row 0-15: MCP23017 GPA0-7(Row0-7) + GPB0-7(Row8-15)
 *
 * LEDピン:
 *   CapsLock LED : D6   = P1.11 (XIAO GPIO)
 *   NumLock  LED : NFC2 = P0.10 (NFCピンをGPIOに転用)
 *   接続: GPIO → 330Ω → LED(アノード) → LED(カソード) → GND
 *
 * ダイオードなしマトリクス:
 *   diode-direction = "row2col"
 *   Row(MCP23017): OUTPUT (GPIO_ACTIVE_LOW)
 *   Col(XIAO)    : INPUT  (GPIO_ACTIVE_LOW | GPIO_PULL_UP)  ← XIAO内蔵プルアップ使用
 *
 * ColはXIAO直結GPIOのため、キー押下時にnRF52840が直接割り込み検知可能。
 * poll-period-msを省略することで割り込みモードが有効になる。
 * MCP23017のINTA/INTBは不要。
 *
 * v10変更点:
 *   - CapsLock / NumLock LED用に gpio-leds + zmk,indicator-leds ノードを追加
 *   - nfct-pins-as-gpios は既存のまま（NFC2=P0.10をNumLock LEDに使用）
 */
#include "keymaps/tk_fcp026bk.keymap"
#include <dt-bindings/zmk/hid_indicators.h>

&i2c0 {
    status = "okay";
    pinctrl-0 = <&i2c0_default>;
    pinctrl-names = "default";
    clock-frequency = <I2C_BITRATE_FAST>; /* 400kHz: MCP23017ポーリングに必要 */
    mcp23017: mcp23017@20 {
        compatible = "microchip,mcp23017";
        reg = <0x20>;
        gpio-controller;
        #gpio-cells = <2>;        /* gpiosプロパティの引数数：<ピン番号> <フラグ> の2つ */
        ngpios = <16>;
        status = "okay";
    };
};

/* P0.28/P0.29 (D2/D3) をNFCピンからGPIOとして使用するための設定 */
/* P0.09/P0.10 (NFC1/NFC2) も同様にGPIOとして使用可能になる         */
&uicr {
    nfct-pins-as-gpios;
};

&pinctrl {
    i2c0_default: i2c0_default {
        group1 {
            psels = <NRF_PSEL(TWIM_SDA, 0, 4)>,
                    <NRF_PSEL(TWIM_SCL, 0, 5)>;
        };
    };
};

/ {
    chosen {
        zmk,kscan = &kscan0;
    };

    /*
     * GPIO LED定義（Zephyrの gpio-leds ドライバ）
     *
     * CapsLock LED: XIAO D6 = P1.11
     * NumLock  LED: XIAO NFC2 = P0.10（nfct-pins-as-gpiosで転用）
     *
     * 接続例（ACTIVE_HIGH = HIGHで点灯）:
     *   GPIO pin → 330Ω → LED(アノード) → LED(カソード) → GND
     */
    leds {
        compatible = "gpio-leds";

        caps_lock_led: caps_lock_led {
            gpios = <&gpio1 11 GPIO_ACTIVE_HIGH>;  /* D6 = P1.11 */
        };

        num_lock_led: num_lock_led {
            gpios = <&gpio0 10 GPIO_ACTIVE_HIGH>;  /* NFC2 = P0.10 */
        };
    };

    /*
     * ZMK HIDインジケーター → LED対応定義
     * ZMK_INDICATOR_LEDS は zmk,indicator-leds ノードの存在で自動有効化される
     * （Kconfig.defconfigへの記述不要）
     */
    indicators {
        compatible = "zmk,indicator-leds";

        caps_lock_indicator: caps_lock {
            indicator = <HID_INDICATOR_CAPS_LOCK>;
            leds = <&caps_lock_led>;
        };

        num_lock_indicator: num_lock {
            indicator = <HID_INDICATOR_NUM_LOCK>;
            leds = <&num_lock_led>;
        };
    };

    kscan0: kscan {
        compatible = "zmk,kscan-gpio-matrix";
        diode-direction = "row2col";
        /* poll-period-ms = <5>; */          /* 省略 → 割り込みモード有効 */
        debounce-press-ms = <2>;
        debounce-release-ms = <1>;

        row-gpios
            = <&mcp23017 0  GPIO_ACTIVE_LOW>  /* Row0  GPA0 */
            , <&mcp23017 1  GPIO_ACTIVE_LOW>  /* Row1  GPA1 */
            , <&mcp23017 2  GPIO_ACTIVE_LOW>  /* Row2  GPA2 */
            , <&mcp23017 3  GPIO_ACTIVE_LOW>  /* Row3  GPA3 */
            , <&mcp23017 4  GPIO_ACTIVE_LOW>  /* Row4  GPA4 */
            , <&mcp23017 5  GPIO_ACTIVE_LOW>  /* Row5  GPA5 */
            , <&mcp23017 6  GPIO_ACTIVE_LOW>  /* Row6  GPA6 */
            , <&mcp23017 7  GPIO_ACTIVE_LOW>  /* Row7  GPA7 */
            , <&mcp23017 8  GPIO_ACTIVE_LOW>  /* Row8  GPB0 */
            , <&mcp23017 9  GPIO_ACTIVE_LOW>  /* Row9  GPB1 */
            , <&mcp23017 10 GPIO_ACTIVE_LOW>  /* Row10 GPB2 */
            , <&mcp23017 11 GPIO_ACTIVE_LOW>  /* Row11 GPB3 */
            , <&mcp23017 12 GPIO_ACTIVE_LOW>  /* Row12 GPB4 */
            , <&mcp23017 13 GPIO_ACTIVE_LOW>  /* Row13 GPB5 */
            , <&mcp23017 14 GPIO_ACTIVE_LOW>  /* Row14 GPB6 */
            , <&mcp23017 15 GPIO_ACTIVE_LOW>  /* Row15 GPB7 */
            ;

        col-gpios
            = <&gpio0 2  (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>  /* Col0 D0  P0.02 */
            , <&gpio0 3  (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>  /* Col1 D1  P0.03 */
            , <&gpio0 28 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>  /* Col2 D2  P0.28 */
            , <&gpio0 29 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>  /* Col3 D3  P0.29 */
            , <&gpio1 12 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>  /* Col4 D7  P1.12 */
            , <&gpio1 13 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>  /* Col5 D8  P1.13 */
            , <&gpio1 14 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>  /* Col6 D9  P1.14 */
            , <&gpio1 15 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>  /* Col7 D10 P1.15 */
            ;
    };
};
```

---

### `config/boards/shields/tk_fcp026bk/keymaps/tk_fcp026bk.keymap`

> v10確定版。NumLockレイヤー追加・BT_CLR/Ins変更・数字キーコード修正。

```dts
/*
 * TK-FCP026BK Keymap v10
 * 86キー 日本語配列
 * 新配線: Col=XIAO(D0-D3,D7-D10)、Row=MCP23017(GPA0-7,GPB0-7)
 *
 * bindingsの並び順: Row行優先 16行×8列=128エントリ
 *
 * レイヤー構成:
 *   Layer 0: default_layer  … 通常入力
 *   Layer 1: fn_layer       … FNキー押しっぱなし中
 *   Layer 2: numlock_layer  … NumLockキーでトグル（テンキー的数字入力）
 *
 * v10変更点:
 *   - NumLockキー(&tog 2)でnumlock_layerをトグル
 *   - numlock_layer追加（m→0、j→1、k→2、l→3、u→4、i→5、o→6、
 *                        7→7、8→8、9→9、/→KP_SLASH、;→KP_PLUS、
 *                        p→KP_MINUS、0→KP_MULTIPLY）
 *   - 数字キーはN0〜N9（OSのNumLock状態に依存しない通常数字コード）を使用
 *     KP_Nxはテンキー専用コードでOSのNumLock OFFでカーソルキーになるため不採用
 *   - fn_layer: BT_CLRをfn+5に移動、fn+DelにInsを割り当て
 */

#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>

/ {
    keymap {
        compatible = "zmk,keymap";

        default_layer {
            bindings = <
/* Col:           D0                D1                D2                D3                D7                D8                D9                D10           */
/* Row0  GPA0 */ &kp Q             &kp TAB           &kp A             &kp ESC           &kp Z             &kp INT5          &kp GRAVE         &kp N1
/* Row1  GPA1 */ &kp W             &kp CAPS          &kp S             &trans            &kp X             &kp INT4          &kp F1            &kp N2
/* Row2  GPA2 */ &kp E             &kp F3            &kp D             &kp F4            &kp C             &kp INT2          &kp F2            &kp N3
/* Row3  GPA3 */ &kp R             &kp T             &kp F             &kp G             &kp V             &kp B             &kp N5            &kp N4
/* Row4  GPA4 */ &kp U             &kp Y             &kp J             &kp H             &kp M             &kp N             &kp N6            &kp N7
/* Row5  GPA5 */ &kp I             &kp RBKT          &kp K             &kp F6            &kp COMMA         &kp INT1          &kp EQUAL         &kp N8
/* Row6  GPA6 */ &kp O             &kp F7            &kp L             &mo 1             &kp DOT           &kp K_APP         &kp F8            &kp N9
/* Row7  GPA7 */ &trans            &trans            &trans            &kp UP            &trans            &kp LEFT          &trans            &trans
/* Row8  GPB0 */ &trans            &trans            &trans            &trans            &trans            &kp RIGHT         &trans            &trans
/* Row9  GPB1 */ &trans            &trans            &trans            &kp SPACE         &trans            &kp DOWN          &kp DEL           &trans
/* Row10 GPB2 */ &trans            &kp LSHFT         &kp RSHFT         &trans            &trans            &trans            &trans            &trans
/* Row11 GPB3 */ &kp INT_YEN       &kp BSPC          &trans            &kp F11           &kp RET           &kp F12           &kp F9            &kp F10
/* Row12 GPB4 */ &kp P             &kp LBKT          &kp SEMI          &kp QUOT          &kp NUHS          &kp FSLH          &kp MINUS         &kp N0
/* Row13 GPB5 */ &tog 2            &trans            &trans            &kp LALT          &trans            &kp RALT          &trans            &kp PSCRN
/* Row14 GPB6 */ &kp PAUSE_BREAK   &trans            &trans            &trans            &trans            &trans            &kp LCTRL         &kp F5
/* Row15 GPB7 */ &trans            &kp LGUI          &trans            &trans            &trans            &trans            &trans            &trans
            >;
        };

        fn_layer {
            bindings = <
/* Col:           D0                D1                D2                D3                D7                D8                D9                D10           */
/* Row0  GPA0 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &bt BT_SEL 0
/* Row1  GPA1 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &bt BT_SEL 1
/* Row2  GPA2 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &bt BT_SEL 2
/* Row3  GPA3 */ &trans            &trans            &trans            &trans            &trans            &trans            &bt BT_CLR        &trans
/* Row4  GPA4 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row5  GPA5 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row6  GPA6 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row7  GPA7 */ &trans            &trans            &trans            &kp PG_UP         &trans            &kp HOME          &trans            &trans
/* Row8  GPB0 */ &trans            &trans            &trans            &trans            &trans            &kp END           &trans            &trans
/* Row9  GPB1 */ &trans            &trans            &trans            &trans            &trans            &kp PG_DN         &kp INS           &trans
/* Row10 GPB2 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row11 GPB3 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row12 GPB4 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row13 GPB5 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row14 GPB6 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row15 GPB7 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
            >;
        };

        numlock_layer {
            bindings = <
/* Col:           D0                D1                D2                D3                D7                D8                D9                D10           */
/* Row0  GPA0 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row1  GPA1 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row2  GPA2 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row3  GPA3 */ &trans            &trans            &trans            &trans            &trans            &trans            &kp N5            &kp N4
/* Row4  GPA4 */ &kp N4            &kp N7            &kp N1            &trans            &kp N0            &trans            &kp N6            &kp N7
/* Row5  GPA5 */ &kp N5            &trans            &kp N2            &trans            &trans            &trans            &trans            &kp N8
/* Row6  GPA6 */ &kp N6            &trans            &kp N3            &trans            &trans            &trans            &trans            &kp N9
/* Row7  GPA7 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row8  GPB0 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row9  GPB1 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row10 GPB2 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row11 GPB3 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row12 GPB4 */ &kp KP_MINUS      &trans            &kp KP_PLUS       &trans            &trans            &kp KP_SLASH      &trans            &kp KP_MULTIPLY
/* Row13 GPB5 */ &tog 2            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row14 GPB6 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
/* Row15 GPB7 */ &trans            &trans            &trans            &trans            &trans            &trans            &trans            &trans
            >;
        };
    };
};
```

---

### GitHubでのビルド手順

1. GitHub でリポジトリを新規作成（例：`zmk-tk-fcp026bk`）
2. 上記ファイル構成をアップロード（`.github` フォルダは「Create new file」で手動作成）
3. `Actions` タブでビルドが完了したら Artifacts から `firmware.zip` をダウンロード
4. ZIP を解凍して `.uf2` ファイルを取り出す

### XIAO nRF52840 への書き込み

1. XIAO をPCにUSBケーブルで接続
2. リセットボタンを**素早く2回押す**（ダブルクリック）
3. PC上に `XIAO-SENSE` または `XIAO BLE` のUSBドライブが表示される
4. コマンドプロンプトで書き込み（エクスプローラーのドラッグ＆ドロップはエラーが出る場合がある）：
   ```cmd
   copy /B "パス\to\firmware.uf2" F:\
   ```
   （`F:` の部分は実際のドライブ文字に合わせる）
5. ドライブが自動的に消えれば書き込み成功

### ZMK書き込み後にArduino IDEに戻す場合

ZMKが書き込まれた状態からArduino IDEで再書き込みするには、ブートローダーモードに入る必要がある：
1. リセットボタンを素早く2回押してブートローダーモードに入る
2. Arduino IDE でボードを `Seeed XIAO BLE nRF52840` に設定
3. 通常通り「書き込み」ボタンで書き込む

---

## 6. ZMK動作確認と試行錯誤の記録

### 6-1. 最初のビルドとBLE接続確認

ZMKファームウェアをGitHub Actionsでビルドし、XIAOに書き込んだ結果：

- BLEデバイスとして `TK-FCP026BK` がWindowsのBT設定に表示された
- ペアリング・接続は成功（緑丸で表示）
- **しかし全キーが無反応**

### 6-2. 失敗：キーマップの構造ミス（v1〜v2）

**問題：** 最初のkeymapはZMKのkscan-gpio-matrixの仕様を無視した構造になっていた。

ZMKのkscan-gpio-matrixでは `bindings` に **Row×Col の行優先で128エントリ（16行×8列）** を並べる必要があるが、初期版は6行分しかなく行の区切りも誤っていた。

**教訓：** ZMKのkeymap bindingsはRow/Colの物理マトリクスに対応した128エントリが必要。キーなしセルは `&trans` で埋める。

### 6-3. 失敗：Kconfig.defconfigの不足（v2）

**問題：** I2C・GPIO・MCP23017の有効化設定が不足していてビルドエラーが発生。

**修正：** 以下をKconfig.defconfigに追加した。

```kconfig
config I2C
    default y
config GPIO_MCP230XX   # ← MCP23017はこのシンボル名（MCP230XXが正しい）
    default y
```

### 6-4. 失敗：キーマップのスキャンデータ誤り（v3）

**問題：** キーマップ取得時に使ったArduinoスキャンプログラムが古い版（D6使用・D10未使用）だったため、FFC05〜FFC08のピンアサインがズレていた可能性があった。

正しいプログラム（D6未使用・D7〜D10使用）で再スキャンしたところ、Row/Col座標は変わらず同一だった。keymapデータ自体は正しかった。

**教訓：** スキャンプログラムのピンアサインとoverlay・keymapのピンアサインは必ず一致させる。

### 6-5. 失敗：diode-directionとGPIOフラグの不整合（v3）

**問題：** 全キー無反応の根本原因はkeymapではなくoverlayの設定誤りだった。

v3のoverlay：
```dts
diode-direction = "row2col";
row-gpios = GPIO_ACTIVE_HIGH      # ← 誤
col-gpios = GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN  # ← 誤
```

Arduinoのスキャンは「RowをOUTPUT LOWに落としてColをINPUT_PULLUPで読む」動作だったが、ZMKの設定がこれと整合していなかった。

**修正の試み（v4）：** `col2row` + `GPIO_ACTIVE_LOW` + `GPIO_PULL_UP` に変更した。これでXIAO側のキーは入力できるようになったが、GPA側の連続入力問題が発生。最終的にはさらに配線変更・row2col化（v5c）で解決した（詳細は第7章）。

### 6-6. 失敗：GPA側キーの連続入力が止まらない問題

**症状：** v4でRow=XIAO・Col=GPBのキーは正常入力できたが、Row=GPA（MCP23017側）のキーを一度押すと連続入力が止まらなくなった。

**原因：** MCP23017の内蔵プルアップ抵抗が約100kΩと弱いため、GPA側のRow読み取りが不安定になっていた。XIAOのRow（XIAO側）は内蔵プルアップが十分強いため問題が出なかった。

**試みた対策（効果なし）：**
- GPA各ピンへの外付けプルアップ抵抗追加
- `poll-period-ms` の調整
- debounce値の調整

**解決策：** 配線を根本から変更した。

---

## 7. 配線変更と動作確定（v5系）

### 7-1. 配線変更の方針

MCP23017のGPA/GPBを全てRow（INPUT側）にして、XIAOをCol（OUTPUT側）にすることでXIAO内蔵プルアップを活用できる構成に変更した。

**新配線：**
```
Col 0-7 : XIAO GPIO D0-D3, D7-D10
Row 0-15: MCP23017 GPA0-7(Row0-7) + GPB0-7(Row8-15)
```

この配線変更に合わせてキーマトリクスを再スキャンし、新しいRow/Col座標でkeymapを作り直した。

### 7-2. 失敗：col2rowでXIAOプルアップが効かなかった（v5）

**問題：** 新配線でcol2row（Col=XIAO OUTPUT、Row=MCP INPUT）にしたが動作しなかった。

**原因：** 以下の設定が不足していた。
- `clock-frequency = <I2C_BITRATE_FAST>` が未指定（デフォルト100kHz）
- `poll-period-ms` が未指定
- `debounce` が未指定

**修正（v5b）：** 上記3点を追加した。

### 7-3. 失敗：col2rowではXIAO内蔵プルアップが活きない

**問題：** col2row構成ではXIAO（Col）がOUTPUTになるためプルアップが不要。一方MCP23017（Row）がINPUTになるが内蔵プルアップが弱いため問題が再発。

**解決策（v5c）：** `row2col` に変更した。

```
row2col: Row(MCP23017)をOUTPUT LOW → Col(XIAO)をINPUT PULLUP で読む
```

XIAOがINPUT側（Col）になり、XIAO内蔵プルアップが有効に使われる。これで全キーが正常動作した。

### 7-4. 確定版overlay設定（v5c・動作確認済み）

```dts
diode-direction = "row2col";
clock-frequency = <I2C_BITRATE_FAST>;  /* 400kHz */
/* poll-period-ms省略 → 割り込みモード */
debounce-press-ms = <20>;
debounce-release-ms = <10>;

row-gpios = MCP23017 GPIO_ACTIVE_LOW          /* OUTPUT */
col-gpios = XIAO     GPIO_ACTIVE_LOW | GPIO_PULL_UP  /* INPUT・XIAO内蔵プルアップ */
```

### 7-5. BLE接続の確立

USB充電器経由でBLE動作確認時、ペアリングができない問題が発生した。

**原因：** USB接続時の古いペアリング情報がフラッシュに残っていた。

**解決手順：**
1. Windows側でデバイス一覧から `TK-FCP026BK` を削除
2. キーボード側で **FN + Del**（`&bt BT_CLR`）を押してペアリング情報をクリア
3. 再ペアリングで接続成功

### 7-6. 打鍵チューニング

同じキーの2回連続押しで反応しないことがあったため、以下の値にチューニングした。

```dts
debounce-press-ms = <4>;
debounce-release-ms = <2>;
/* poll-period-ms省略（割り込みモード） */
```

人間の高速打鍵（600打鍵/分）でも連打サイクルは約100msのため、5ms×2=10msのdebounceで十分追いつく。

---

## 8. 追加機能実装（v7）

### 8-1. バッテリー残量

`ZMK_BATTERY_REPORTING` を有効化することでBLE経由でWindows側にバッテリー残量をパーセント通知できる。XIAOのJSTコネクタに3.7Vリポバッテリーを接続し、USB充電器接続中は自動充電される。

### 8-2. 省電力（割り込みモード）

`poll-period-ms` を省略することでZMKの割り込みモードが有効になる。ColがXIAO直結GPIOのため、キー押下時にnRF52840が直接割り込みを検知できる。無操作時はCPUがスリープしたままになり消費電力が下がる。

MCP23017のINTA/INTBピンへの配線は不要。

---

## 9. ゴーストキー問題と対策

### 9-1. 発生している問題

| 症状 | 原因 |
|---|---|
| 左Shift + T/Y が入力できない | 左Shift(Col1)・T(Col1)・Y(Col1)が同じCol線を共有 |
| 右Shift + A/S/D/F/; が入力できない | 右Shift(Col2)・A/S/D/F/;(Col2)が同じCol線を共有 |

ダイオードなしマトリクスでは同じCol上のキーを複数同時押しすると、Row間で電流の逆流が発生してスキャンが誤動作する。

### 9-2. 対策の検討

**対策A：Row線（MCP23017側）にダイオードを16本追加**

MCP23017の各Row出力ピンとFFC変換基板の間に1N4148を直列に挿入。

向きは **FFC変換基板側がカソード（帯あり）、MCP23017側がアノード（帯なし）**。（注：最初に逆向きを指示してしまい、実際に試して逆向きが正しいと確認された。）

効果：キースイッチ個別にダイオードを入れるのと同等の効果が得られる。必要本数は16本。

**対策B：col2row構成に変更する（配線変更なし・ファームウェアのみ）**

現行row2col構成でのCol配置を分析すると：
- 左Shift・右Shift は両方とも **Row10** に存在する
- col2row視点では「同じRow上の複数キーの同時押し」がゴーストの原因になる
- 左Shift + 文字キー、右Shift + 文字キーは **異なるRowの組み合わせ** → **ゴーストなし**
- 左Shift + 右Shift の同時押しのみRow10同士で問題が残るが、通常使用では発生しない

**col2rowに変更するだけでShift絡みのゴーストが解消される。keymapの変更は不要。**

col2row版overlayの変更点：
```dts
diode-direction = "col2row";
col-gpios = XIAO GPIO_ACTIVE_LOW          /* OUTPUT */
row-gpios = MCP23017 GPIO_ACTIVE_LOW | GPIO_PULL_UP  /* INPUT・MCP内蔵プルアップ */
```

なおcol2row構成ではMCP23017（Row）がINPUTになるため内蔵プルアップ（100kΩ）を使用することになるが、row2col時に問題となったGPA連続入力問題が再発するかどうかは要確認。

### 9-3. 現状

対策A：Row線（MCP23017側）にダイオードを16本追加し、ゴーストは解消。USB給電で完全動作確認済み。
対策B：col2row構成に変更する、col2row版overlayは `tk_fcp026bk.overlay.col2row` として記録済み。構成変更は未実施。

---

## 10. 追加機能実装（v10）

### 10-1. CapsLock / NumLock LED

#### 概要

| LED | ピン | 用途 |
|---|---|---|
| CapsLock LED | XIAO D6（P1.11） | CapsLock ON時に点灯 |
| NumLock LED | XIAO NFC2（P0.10） | NumLockレイヤー有効時に点灯 |

NFC2ピンはもともとNFC通信用だが、overlayの `nfct-pins-as-gpios` 設定（D2/D3のためにすでに記載済み）によってGPIOとして転用できる。

#### overlayでのLED定義構造

ZMKのLEDインジケーターは **2段構成** で定義する。

```
gpio-leds ノード       … ZephyrのLEDドライバ（GPIOをLEDとして登録）
    ↓ phandle参照
zmk,indicator-leds ノード … ZMKがHID状態（CapsLock/NumLock）をLEDに反映
```

`ZMK_INDICATOR_LEDS` というKconfigシンボルは、`zmk,indicator-leds` ノードがoverlayに存在することで **自動的に有効化** される。Kconfig.defconfigへの記述は不要（存在しないシンボル `ZMK_LED_INDICATORS` を書くとビルドエラーになる）。

#### LED接続

```
XIAO D6 (P1.11)   ──→ 330Ω ──→ LED(+) ──→ LED(-) ──→ GND  ← CapsLock
XIAO NFC2 (P0.10) ──→ 330Ω ──→ LED(+) ──→ LED(-) ──→ GND  ← NumLock
```

GPIO_ACTIVE_HIGH 設定（HIGH=点灯）。3.3V・赤色LED・330Ωで約4mA。

### 10-2. NumLockレイヤー

#### 設計方針

NumLockキー単体押しで `&tog 2`（Layer 2トグル）。NumLockレイヤーでは以下のキーが数字・記号入力に変わる。

| キー | 通常 | NumLockレイヤー |
|---|---|---|
| m | m | 0 |
| j | j | 1 |
| k | k | 2 |
| l | l | 3 |
| u | u | 4 |
| i | i | 5 |
| o | o | 6 |
| 7 | 7 | 7 |
| 8 | 8 | 8 |
| 9 | 9 | 9 |
| / | / | KP_SLASH |
| ; | ; | KP_PLUS |
| p | p | KP_MINUS |
| 0 | 0 | KP_MULTIPLY |

#### キーコードの選択について

数字キーには `KP_N0`〜`KP_N9`（テンキー専用コード）ではなく **`N0`〜`N9`（通常数字コード）** を使用する。

`KP_Nx` はOSのNumLock状態に依存し、NumLock OFFのとき数字ではなくカーソルキー・Ins・Delとして解釈される。このキーボードではNumLockレイヤーで「常に数字を入力する」のが目的のため、OS状態に依存しない `Nx` が適切。

記号（`KP_SLASH`・`KP_PLUS`・`KP_MINUS`・`KP_MULTIPLY`）はNumLock非依存のためテンキーコードのまま使用。

### 10-3. BT_CLR / Ins の割り当て変更

| 操作 | 旧 | 新 |
|---|---|---|
| BT_CLR（ペアリング情報クリア） | FN + Del | **FN + 5** |
| Ins | （未割当） | **FN + Del** |

---

## 11. 失敗・誤りのまとめ

本プロジェクトで発生した主な誤りを記録する。

| # | 誤り | 正しい内容 |
|---|---|---|
| 1 | MCP23017のVDDはpin9とpin15の両方と説明した | VDDはpin9のみ。pin15はA0（アドレスビット0） |
| 2 | ダイオードの向きを「MCP23017側カソード（帯あり）、FFC側アノード」と指示した | 正しくはFFC変換基板側がカソード（帯あり）、MCP23017側がアノード（実際に試して確認） |
| 3 | QMKがnRF52840で使えると案内した | QMKはnRF52840を正式サポートしていない。ZMKを使う必要がある |
| 4 | スキャンプログラムにD6を含む古い版を使い続けた | D6は未使用。D7〜D10を使う正しい版でスキャンし直す必要があった |
| 5 | v3のoverlayでdiode-direction="row2col"とGPIO_ACTIVE_HIGHを組み合わせた | v4でcol2row + GPIO_ACTIVE_LOWに修正したがGPA連続入力問題が発生。最終的にはrow2col + XIAO内蔵プルアップ（GPIO_ACTIVE_LOW \| GPIO_PULL_UP）が正しい解決策 |
| 6 | GPA側キーの連続入力問題に対してファームウェア調整で対応しようとした | MCP23017内蔵プルアップの弱さが根本原因。配線変更（XIAOをCol側に移動）が正しい解決策 |
| 7 | col2rowでXIAOプルアップが活きると説明した | col2rowではXIAOはOUTPUTのためプルアップ不要。XIAOのプルアップを活かすにはrow2colが正しい |
| 8 | LEDノードを `zmk,led-indicators` として定義した | 正しいcompatible名は `zmk,indicator-leds`。またGPIOピンの定義は先に `gpio-leds` ノードで行い、`zmk,indicator-leds` からphandle参照する2段構成が必要 |
| 9 | `ZMK_LED_INDICATORS` をKconfig.defconfigに記述した | このシンボルは存在しない。正しくは `ZMK_INDICATOR_LEDS` だが、overlayに `zmk,indicator-leds` ノードがあれば自動有効化されるためKconfigへの記述自体が不要 |
| 10 | NumLockレイヤーの数字キーに `KP_N0`〜`KP_N9` を使用した | `KP_Nx` はOSのNumLock状態に依存し、NumLock OFFでカーソルキーになる。OS状態に依存しない `N0`〜`N9` を使うのが正しい |

---

*作成日：2026-06-01 / 更新日：2026-06-24*
