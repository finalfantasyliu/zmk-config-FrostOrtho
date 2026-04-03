# FrostOrtho moNa2 風格配列改動說明

---

## 第一部分：ZMK 韌體基礎知識

### 什麼是韌體（Firmware）？

韌體是燒錄在鍵盤微控制器（MCU）裡面的程式。它負責：
- 掃描你按了哪個按鍵
- 決定按鍵對應的輸出（例如按 A 鍵輸出字母 a）
- 處理層切換、組合鍵、滑鼠控制等高級功能
- 管理藍牙連線、電池、LED 等硬體

沒有韌體，鍵盤就是一塊沒有靈魂的電路板。

### 什麼是 ZMK？

ZMK（Zephyr Mechanical Keyboard）是一個開源的鍵盤韌體框架，基於 Zephyr RTOS（即時作業系統）。相比其他韌體（如 QMK），ZMK 的特點是：

- **原生支援藍牙**：專為無線鍵盤設計
- **原生支援分離式鍵盤**：左右半透過 BLE 通訊
- **低功耗**：支援深度睡眠，電池可用數週到數月
- **不需本地編譯環境**：透過 GitHub Actions 在雲端編譯

### FrostOrtho 的硬體架構

```
┌─────────────────┐          BLE 無線          ┌─────────────────┐
│    左半（L）      │  ◄──────────────────────►  │    右半（R）      │
│  Peripheral      │                            │  Central         │
│                  │                            │                  │
│  • 按鍵矩陣      │                            │  • 按鍵矩陣      │
│  • 旋鈕 Encoder  │                            │  • 軌跡球 PMW3610│
│  • RGB LED       │                            │  • RGB LED       │
│  • 電池          │                            │  • 電池          │
│                  │                            │  • 連接電腦/手機  │
│  MCU: XIAO BLE   │                            │  MCU: XIAO BLE   │
│  (nRF52840)      │                            │  (nRF52840)      │
└─────────────────┘                            └─────────────────┘
```

**重點：**
- **右半是 Central（中央）**：負責連接電腦/手機，處理軌跡球
- **左半是 Peripheral（周邊）**：把按鍵事件透過 BLE 傳給右半
- 兩邊都有 **Seeeduino XIAO BLE** 微控制器，內建 RGB LED
- 左半有**旋鈕（EC11 Encoder）**，右半有**軌跡球（PMW3610 光學感測器）**

### ZMK 的檔案結構與作用

```
zmk-config-FrostOrtho/
│
├── config/                              ← 【所有設定都在這裡】
│   ├── FrostOrtho.keymap                ← 鍵位配置：定義每一層每個按鍵的功能
│   ├── FrostOrtho.json                  ← 鍵盤物理配置（給 keymap 編輯器用的）
│   ├── west.yml                         ← 依賴管理：定義要用哪些外部模組及版本
│   │
│   └── boards/shields/FrostOrtho/       ← 硬體定義（Shield）
│       ├── FrostOrtho.dtsi              ← 共用硬體定義：按鍵矩陣、GPIO 腳位
│       ├── FrostOrtho_R.overlay         ← 右半專用：軌跡球 SPI 設定、輸入處理器
│       ├── FrostOrtho_L.overlay         ← 左半專用：旋鈕設定、GPIO 腳位
│       ├── FrostOrtho_R.conf            ← 右半設定：藍牙、軌跡球參數、LED、Studio
│       ├── FrostOrtho_L.conf            ← 左半設定：旋鈕、電池、LED
│       ├── FrostOrtho.zmk.yml           ← Shield 後設資料
│       ├── Kconfig.shield               ← Shield Kconfig 選項
│       └── Kconfig.defconfig            ← 預設 Kconfig 值（定義哪邊是 Central）
│
├── build.yaml                           ← Build 矩陣：告訴 CI 要編譯哪些韌體
├── .github/workflows/build.yml          ← CI/CD：GitHub Actions 工作流定義
│
├── firmware/                            ← 預先編譯好的韌體（可能過時）
│   ├── FrostOrtho_R-seeeduino_xiao_ble-zmk.uf2
│   ├── FrostOrtho_L-seeeduino_xiao_ble-zmk.uf2
│   └── settings_reset-seeeduino_xiao_ble-zmk.uf2
│
└── doc/                                 ← 文件
```

### 核心概念詳解

#### Device Tree（裝置樹）：`.dtsi`、`.overlay`

ZMK 基於 Zephyr RTOS，用 **Device Tree** 來描述硬體。你可以把它想成硬體的「說明書」：

- **`.dtsi`（Device Tree Source Include）**：共用的硬體定義，例如按鍵矩陣有幾行幾列、旋鈕接在哪個腳位
- **`.overlay`**：覆蓋/補充 `.dtsi` 的設定。左半和右半有不同的 overlay，因為它們的硬體不同（左半有旋鈕，右半有軌跡球）

例如 `FrostOrtho_R.overlay` 裡定義了：
```
trackball: trackball@0 {
    compatible = "pixart,pmw3610";     // 用 PMW3610 驅動
    spi-max-frequency = <2000000>;     // SPI 通訊頻率 2MHz
    irq-gpios = <&gpio0 2 ...>;       // 中斷腳位在 GPIO 0.2
};
```
這告訴韌體：「右半板子上有一個 PMW3610 軌跡球感測器，透過 SPI 連接，中斷訊號在 GPIO 0.2」。

#### Kconfig（`.conf` 檔案）

Kconfig 是 Zephyr 的功能開關系統。`.conf` 檔案用 `CONFIG_XXX=y` 的格式啟用或設定功能：

```
CONFIG_PMW3610=y                    # 啟用 PMW3610 軌跡球驅動
CONFIG_PMW3610_CPI=1500             # 軌跡球靈敏度（CPI = Counts Per Inch）
CONFIG_ZMK_BLE=y                    # 啟用藍牙
CONFIG_RGBLED_WIDGET_SHOW_LAYER_COLORS=y  # 啟用 LED 層顏色顯示
```

每個 `CONFIG_` 都是一個功能開關或參數。編譯時，系統會根據這些設定決定要編譯哪些程式碼。

#### Keymap（鍵位配置）

`.keymap` 檔案是你最常改的東西。它定義：
1. **行為（Behaviors）**：按鍵可以做什麼（輸出字元、切層、滑鼠點擊等）
2. **層（Layers）**：每一層每個按鍵的綁定
3. **組合鍵（Combos）**：多鍵同按觸發的功能
4. **巨集（Macros）**：一鍵執行多個動作
5. **感測器綁定（Sensor Bindings）**：旋鈕旋轉時的動作

#### Layer（層）的運作方式

想像你的鍵盤有一疊透明膠片，每張膠片上寫著不同的按鍵功能：

```
Layer 8 (DESKTOP)   ← 最上面，只在按住 W 時啟用
Layer 7 (VIM)       ← 按住 Space 時啟用
Layer 6 (SCROLL)    ← 按住 DOT 時啟用
Layer 5 (MOUSE)     ← 軌跡球移動時自動啟用
Layer 4 (BT)        ← Combo 觸發
Layer 3 (MEDIA)     ← 按住 Alt 時啟用
Layer 2 (NUM)       ← 按住 Enter 時啟用
Layer 1 (SYM)       ← 按住 Backspace 時啟用
Layer 0 (Default)   ← 最下面，永遠啟用
```

按鍵時，系統**從上往下**找第一個有定義的綁定：
- 如果某層的某個鍵是 `&trans`（透明），就往下一層看
- 如果是 `&none`，就阻斷，什麼都不做
- 如果有具體綁定（如 `&kp A`），就執行它

#### Hold-Tap 行為

Hold-Tap 是 ZMK 最重要的功能之一。一個實體按鍵有兩種用途：

```
&lt 1 BACKSPACE
│   │     │
│   │     └── 點按（tap）= 輸出 Backspace
│   └──────── 按住（hold）= 啟用 Layer 1（SYM 符號層）
└──────────── 行為類型：Layer-Tap
```

```
&mt LEFT_SHIFT Z
│       │       │
│       │       └── 點按 = 輸出 Z
│       └────────── 按住 = 左 Shift
└────────────────── 行為類型：Mod-Tap
```

**參數說明：**
- `flavor = "balanced"`：判定方式。balanced 表示按下時不立刻判定，等你鬆開或按其他鍵時才決定是 tap 還是 hold
- `tapping-term-ms = <200>`：按住超過 200ms 就判定為 hold
- `quick-tap-ms = <300>`：如果 300ms 內連續點按同一個鍵，直接判定為 tap（快速連打）
- `require-prior-idle-ms = <150>`：如果 150ms 內有按過其他鍵（正在打字），直接判定為 tap。這避免打字時意外觸發 hold

#### Combo（組合鍵）

同時按下特定位置的兩個鍵，觸發第三個動作：

```
combos {
    tab {
        bindings = <&kp TAB>;
        key-positions = <1 2>;    // 位置 1（W）和位置 2（E）同時按
    };
};
```

位置編號按照 keymap 裡的順序：
```
行 0:  [0]Q  [1]W  [2]E  [3]R  [4]T    [5]Y  [6]U  [7]I  [8]O  [9]P
行 1: [10]A [11]S [12]D [13]F [14]G   [15]H [16]J [17]K [18]L [19];
行 2: [20]Z [21]X [22]C [23]V [24]B   [25]N [26]M [27], [28]. [29]/
行 3: [30]  [31]  [32]  [33]  [34] [35]  [36]  [37]  [38]  [39]  [40]
       └────── 左手拇指 ──────┘ └─┘  └──────── 右手拇指 ────────┘
```

#### Sensor Binding（感測器綁定 / 旋鈕）

旋鈕（Encoder）是一種旋轉感測器。每一層可以定義旋轉時的動作：

```
sensor-bindings = <&scroll_up_down>;  // 這一層旋鈕 = 捲動
```

```
scroll_up_down: behavior_sensor_rotate_mouse_wheel_up_down {
    compatible = "zmk,behavior-sensor-rotate";
    bindings = <&msc SCRL_UP>, <&msc SCRL_DOWN>;
    //          順時針=上捲      逆時針=下捲
};
```

#### Input Processor（輸入處理器 / 軌跡球）

軌跡球的行為不是在 keymap 裡定義，而是在 overlay 裡用 **Input Processor** 處理：

```
input-processors = <
    &mouse_runtime_input_processor       // DYA Studio 動態處理
    &zip_temp_layer 5 100000             // 自動切到第 5 層（MOUSE），10 萬 ms 逾時
    &zip_xy_scaler 2 3                   // 游標移動量縮放為 2/3
    &zip_xy_transform INPUT_TRANSFORM_Y_INVERT  // Y 軸反轉
>;
```

**Auto Mouse Layer（AML / 自動滑鼠層）**：
- `zip_temp_layer 5 100000`：軌跡球一動，自動切到 Layer 5（MOUSE），讓你可以用 G/B 鍵點擊
- 停止移動後 100 秒自動退出（但按一下滑鼠鍵會重置計時器）
- `require-prior-idle-ms = <200>`：打字後 200ms 內的軌跡球移動會被忽略，防止打字時手掌誤觸

**excluded-positions（排除位置）**：
- 在 MOUSE 層時，按這些位置的鍵**不會退出** MOUSE 層
- 通常設為滑鼠按鍵和修飾鍵，這樣你可以 Shift+點擊、Ctrl+點擊等

### 藍牙（BLE）在 ZMK 中的運作

```
左半 (Peripheral) ──BLE──► 右半 (Central) ──BLE──► 電腦/手機
                   內部連線                   外部連線
```

- **內部連線**：左右半之間的 BLE 連線，自動配對，不需要手動操作
- **外部連線**：右半（Central）和電腦/手機的 BLE 連線，支援最多 5 個配對設備
- **BT Profile**：藍牙設備槽。BT_SEL 0~4 可以各配對一個設備，用藍牙層切換
- **settings_reset**：清除所有儲存的設定（包括內部配對和外部配對），需要重新配對

### Shield 和 Board 的關係

在 ZMK 中：
- **Board（板子）**：硬體平台，例如 `seeeduino_xiao_ble`（XIAO BLE 開發板）
- **Shield（護盾）**：鍵盤本身的定義，例如 `FrostOrtho_R`（右半的按鍵矩陣、腳位等）
- **額外 Shield**：可以疊加功能，例如 `rgbled_adapter`（定義 XIAO BLE 上 RGB LED 的腳位）

Build 時組合方式：`board + shield [+ 額外 shield] [+ snippet]`
```yaml
board: seeeduino_xiao_ble
shield: FrostOrtho_R rgbled_adapter    # 兩個 shield 用空格分隔
snippet: studio-rpc-usb-uart           # 啟用 DYA Studio USB 支援
```

---

## 第二部分：所有改動的完整說明

### 改動的檔案清單

| 檔案 | 改動類型 | 說明 |
|------|---------|------|
| `config/FrostOrtho.keymap` | **重寫** | 從 JIS 8 層改為 moNa2 風格 9 層 |
| `config/boards/shields/FrostOrtho/FrostOrtho_R.overlay` | **修改** | 更新軌跡球的層號和排除位置 |
| `config/boards/shields/FrostOrtho/FrostOrtho_R.conf` | **修改** | 加入 LED 設定、BLE 增強 |
| `config/boards/shields/FrostOrtho/FrostOrtho_L.conf` | **修改** | 加入 LED 設定、BLE 增強 |
| `config/west.yml` | **修改** | 升級 zmk-rgbled-widget 模組版本 |
| `build.yaml` | **修改** | 啟用 rgbled_adapter shield |
| `.github/workflows/build.yml` | **修改** | 加入 build.yaml 的觸發監聽 |

---

### 改動一：鍵位配置（FrostOrtho.keymap）

#### 原本（JIS 配列，8 層）
- Layer 0：日式 QWERTY（有 LANG_HANJA、INTERNATIONAL_1 等日文鍵）
- Layer 1：F 鍵
- Layer 2：數字和符號（日式排列）
- Layer 3：方向鍵
- Layer 4：藍牙
- Layer 5：捲動（全透明，靠軌跡球）
- Layer 6：滑鼠（J=左鍵、K=右鍵、L=中鍵，在右手）
- Layer 7：工具（Undo/Redo/Copy/Paste、截圖巨集）

#### 改為（moNa2 風格，9 層）
- Layer 0：標準 QWERTY（macOS 配列，Cmd 鍵取代 Win 鍵）
- Layer 1：SYM 符號（完整的程式設計符號層）
- Layer 2：NUM 數字 + F 鍵
- Layer 3：MEDIA 媒體控制
- Layer 4：BLUETOOTH 藍牙設定
- Layer 5：MOUSE 滑鼠（G=左鍵、B=右鍵，改到左手）
- Layer 6：SCROLL 捲動
- Layer 7：VIM 方向鍵（H/J/K/L）
- Layer 8：DESKTOP 桌面切換

#### 為什麼這樣改？

**1. 從 JIS 改為 macOS QWERTY**
- 原版有 `LANG_HANJA`（漢字鍵）、`INTERNATIONAL_1`（ろ鍵）等日式 Windows 按鍵
- moNa2 用 `LEFT_COMMAND`（Cmd）配合 macOS，用 Ctrl+Space combo 切輸入法
- 注意：在 ZMK 中 `LEFT_COMMAND` = `LEFT_GUI` = `LEFT_WIN`，同一個鍵碼，只是名稱不同

**2. 滑鼠按鍵從右手改到左手**
- 原版：右手 J/K/L = 左鍵/右鍵/中鍵
- moNa2：左手 G = 左鍵，B = 右鍵
- 原因：軌跡球在右手，如果滑鼠按鍵也在右手，你無法同時移動和點擊。左手點擊、右手移動的分工更自然

**3. 加入 VIM 層和 DESKTOP 層**
- VIM 層（按住 Space）：H/J/K/L = 左/下/上/右，不用移手到方向鍵區
- DESKTOP 層（按住 W）：H = Ctrl+Left（切到左邊桌面），L = Ctrl+Right（切到右邊桌面）
- 這兩層都是 moNa2 的特色功能，原版 FrostOrtho 沒有

**4. 符號層完全重做**
- 原版的符號是日式 JIS 排列（用 `LS(SQT)`、`LS(SEMICOLON)` 等）
- moNa2 的符號層是為程式設計最佳化的排列：
  - 左手：常用的引號、比較運算子、邏輯運算子
  - 右手：括號對（`[]`、`()`、`{}`）和其他符號
  - `C` 和 `V` 位置設為 `&trans`（透明），讓你可以在符號層用 Ctrl+C/V 複製貼上

**5. Enter 和 Backspace 對調**
- 初版 moNa2 適配時 Enter 在內側（pos 36）、BS 在外側（pos 37）
- 你反映 Delete（Backspace）不好按，所以對調：BS 到內側（更常用，更好按），Enter 到外側
- Enter 仍然可以透過 J+K combo 按出來

**6. Encoder 從全層音量改為按層不同功能**
- 原版 FrostOrtho 用 DYA Studio 的 `rsr_vol`（Runtime Sensor Rotate）在所有層都是音量
- 改為 moNa2 的按層設計：
  - 預設層/大多數層：`scroll_up_down`（縱向捲動）← 最常用
  - SYM 層：`scroll_right_left`（水平捲動）← 看程式碼時好用
  - MEDIA 層：`vol_up_down`（音量）← 在媒體層才控制音量

**7. 加入 `&ind_bat` 電量顯示鍵**
- 在藍牙層的 B 鍵（pos 24）放了 `&ind_bat`
- 需要 `#include <behaviors/rgbled_widget.dtsi>` 引入此行為
- 按下後 LED 會閃對應電量的顏色

**8. Hold-Tap 參數最佳化**
- `&mt`：保留 `require-prior-idle-ms = <150>`（防打字時誤觸 Shift）
- `&lt`：**移除** `require-prior-idle-ms`（讓切層隨時響應，不受打字速度影響）
- `&lt` 的 `quick-tap-ms` 從 300 降到 200（減少連打判定窗口）
- [ZMK Hold-Tap 文件](https://zmk.dev/docs/keymaps/behaviors/hold-tap)

> **為什麼分開設定？** `&mt` 管修飾鍵（Shift on Z），打字時常碰到需要防誤觸。`&lt` 管切層（Backspace→SYM），是主動操作不需要門檻。

**9. Combo 防重複觸發**
- 所有 combo（Tab/Enter/Input Switch）加入 `slow-release`：兩鍵都放開後才結束，防止放開時機不同導致重複
- Input Switch combo 改用 `&input_tap` macro：不管按多久只送一次 Ctrl+Space，防止 macOS 循環切換輸入法
- [ZMK Combo 文件](https://zmk.dev/docs/keymaps/combos)

#### FrostOrtho vs moNa2 的物理差異適配

| | moNa2 | FrostOrtho | 處理方式 |
|---|-------|-----------|---------|
| 總鍵數 | 42 | 41 | FrostOrtho 少 1 鍵 |
| 行 0（字母頂排） | 10 鍵 | 10 鍵 | 完全相同 |
| 行 1（Home row） | 11 鍵（含內側 ESC） | 10 鍵 | ESC 搬到拇指 pos 38 |
| 行 2（字母底排） | 12 鍵（含內側 LALT/RALT） | 10 鍵 | RALT 搬到拇指 pos 39 |
| 行 3（拇指列） | 9 鍵 | 11 鍵 | 多出的 pos 38/39 放 ESC/RALT |

moNa2 在左右半之間有「內側列」按鍵（col 5 和 col 7），FrostOrtho 沒有。但 FrostOrtho 的拇指列多了 3 個鍵（col 9/10/11），所以把 moNa2 內側列的功能搬到拇指多出來的位置。

各層的 moNa2 內側列按鍵搬遷對照：

| 層 | moNa2 內側列按鍵 | FrostOrtho 搬到 |
|---|----------------|----------------|
| Default | ESC (行1), LALT/RALT (行2) | pos 38 (ESC), pos 39 (RALT) |
| NUM | F11 (行2左), F12 (行2右) | pos 35 (F11), pos 39 (F12) |
| MEDIA | BRI_UP (行1), BRI_DN (行2) | pos 38 (BRI_UP), pos 39 (BRI_DN) |
| BLUETOOTH | BT_CLR (行1), bootloader (行2左), BT_CLR_ALL (行2右) | pos 35 (bootloader), pos 38 (BT_CLR), pos 39 (BT_CLR_ALL) |
| 其他層 | 全 &trans | 維持 &trans |

---

### 改動二：右半硬體設定（FrostOrtho_R.overlay）

#### 改了什麼？

**只改了我需要改的部分**，原作者的硬體定義（SPI、GPIO、pinctrl）完全沒動。

#### 1. 自動滑鼠層（AML）層號

```c
// 改前
&zip_temp_layer 6 100000

// 改後
&zip_temp_layer 5 100000
```

**為什麼？** 因為 MOUSE 層從 Layer 6 變成 Layer 5。這個數字必須對應 keymap 裡 MOUSE 層的位置，否則軌跡球移動時會切到錯誤的層。

#### 2. 排除位置（excluded-positions）

```c
// 改前（右手點擊）
excluded-positions = <
    16  // J 左鍵
    17  // K 右鍵
    18  // L 中鍵
    20  // Z Shift
    30  // Control
    32  // Alt
>;

// 改後（左手點擊）
excluded-positions = <
    14  // G 左鍵 (MB1)
    24  // B 右鍵 (MB2)
    20  // Z 左Shift (mod-tap)
    30  // 左Shift (拇指)
    31  // 左Ctrl (拇指)
    32  // 左Alt / Layer 3 (拇指)
    33  // 左Command (拇指)
>;
```

**為什麼？**
- 排除位置的意思：「在 MOUSE 層按這些鍵時，不要退出 MOUSE 層」
- 原版的滑鼠鍵在右手（J/K/L），所以排除 J/K/L
- 改成左手點擊（G/B），所以排除 G/B 和左手的所有修飾鍵
- 這樣你可以：右手移動軌跡球 → 左手按 G 點擊 → 繼續移動，整個過程不會意外退出 MOUSE 層

#### 3. 捲動層層號

```c
// 改前
layers = <5>;  // 捲動層

// 改後
layers = <6>;  // 捲動層（SCROLL=6）
```

**為什麼？** SCROLL 層從 Layer 5 變成 Layer 6。這告訴軌跡球輸入處理器：「當 Layer 6 啟用時，把軌跡球的 X/Y 移動轉換為捲動」。

#### 4. 移除水平捲動

```c
// 改前：有 scroller_x 區塊，在 Layer 7 時軌跡球做水平捲動
// 改後：移除，改用註解說明

// 已移除水平捲動（moNa2 風格：僅縱向捲動）
```

**為什麼？** moNa2 不用軌跡球做水平捲動，只用旋鈕在 SYM 層做水平捲動。而且新的 Layer 7 是 VIM（方向鍵），不適合同時做水平捲動。

---

### 改動三：設定檔（.conf）

#### FrostOrtho_L.conf 和 FrostOrtho_R.conf 都加了：

**RGB LED Widget 電量和層顏色：**
```
CONFIG_RGBLED_WIDGET_BATTERY_LEVEL_HIGH=70     # 電量 >70% = 高
CONFIG_RGBLED_WIDGET_BATTERY_LEVEL_LOW=30      # 電量 30~70% = 中
CONFIG_RGBLED_WIDGET_BATTERY_LEVEL_CRITICAL=10 # 電量 <10% = 危險
CONFIG_RGBLED_WIDGET_BATTERY_COLOR_HIGH=2      # 高電量 = 綠色 (2)
CONFIG_RGBLED_WIDGET_BATTERY_COLOR_MEDIUM=3    # 中電量 = 黃色 (3)
CONFIG_RGBLED_WIDGET_BATTERY_COLOR_LOW=5       # 低電量 = 洋紅 (5)
CONFIG_RGBLED_WIDGET_BATTERY_COLOR_CRITICAL=1  # 危險 = 紅色 (1)
CONFIG_RGBLED_WIDGET_SHOW_LAYER_COLORS=y       # 切層時 LED 閃對應顏色
```

顏色代碼：0=黑(關), 1=紅, 2=綠, 3=黃, 4=藍, 5=洋紅, 6=青, 7=白

**為什麼加？**
- 原作者在 R.conf 裡註解掉了這些設定，備註「ケースの形状上LEDがほぼ見えない」（外殼形狀導致 LED 幾乎看不到）
- 你的是透明外殼，可以看到 LED
- 這些設定讓 LED 在三種情況顯示：
  1. **開機/插 USB**：自動閃電量顏色
  2. **切層**：自動閃層對應的顏色
  3. **按 ind_bat 鍵**：手動觸發電量顯示

**BLE 發射功率提升：**
```
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y
```

**為什麼加？**
- nRF52840 的 BLE 預設發射功率是 0dBm
- +8dBm 是最大值，大幅增加無線訊號強度
- 功耗差異極小（微安培等級），但連線距離和穩定性明顯提升
- 你之前遇到「左手沒反應」的問題，就是左右半之間 BLE 訊號不穩。這個設定能預防

---

### 改動四：依賴模組版本（west.yml）

```yaml
# 改前
- name: zmk-rgbled-widget
  remote: caksoylar
  revision: v0.3.0         # 舊穩定版

# 改後
- name: zmk-rgbled-widget
  remote: caksoylar
  revision: v0.3-branch    # 新功能版
```

**什麼是 west.yml？**
- ZMK 使用 **West**（Zephyr 的套件管理工具）來管理依賴
- `west.yml` 列出所有外部模組的來源和版本
- 編譯時，West 會從 GitHub 下載這些模組

**為什麼要升級？**
| 功能 | v0.3.0 | v0.3-branch |
|------|--------|-------------|
| 基本電量顯示 | 有（只有高/低兩級） | 有 |
| 電量顏色分級（4 級） | 沒有 | 有 |
| 層顏色顯示 | 沒有 | 有 |
| `&ind_bat` 手動查電量 | 沒有 | 有 |
| `&ind_con` 查連線狀態 | 沒有 | 有 |
| `behaviors/rgbled_widget.dtsi` | 沒有 | 有 |

moNa2 也用 `v0.3-branch`，所以升級是必要的。

---

### 改動五：Build 目標（build.yaml）

```yaml
# 改前（rgbled_adapter 被原作者註解掉）
- board: seeeduino_xiao_ble
  shield: FrostOrtho_R
  snippet: studio-rpc-usb-uart
- board: seeeduino_xiao_ble
  shield: FrostOrtho_L

# 改後（啟用 rgbled_adapter）
- board: seeeduino_xiao_ble
  shield: FrostOrtho_R rgbled_adapter
  snippet: studio-rpc-usb-uart
- board: seeeduino_xiao_ble
  shield: FrostOrtho_L rgbled_adapter
```

**什麼是 `rgbled_adapter`？**

`rgbled_adapter` 是 `zmk-rgbled-widget` 模組提供的一個小型 Shield。它的內容大概是：

```
// 告訴韌體 XIAO BLE 上的 RGB LED 接在哪些 GPIO
aliases {
    led-red = &led0;
    led-green = &led1;
    led-blue = &led2;
};
```

**為什麼它是必要的？**

`.conf` 裡的 `CONFIG_RGBLED_WIDGET_*` 設定只是告訴韌體「你要怎麼顯示」，但韌體還需要知道「LED 在哪裡」。`rgbled_adapter` 就是提供這個硬體資訊的。

沒有 `rgbled_adapter`：
- ❌ 韌體不知道 LED 的 GPIO 腳位
- ❌ 所有 LED 相關功能都不會生效
- ❌ `.conf` 裡的 LED 設定形同虛設

有了 `rgbled_adapter`：
- ✅ 韌體知道怎麼控制 XIAO BLE 上的 RGB LED
- ✅ 開機電量、切層顏色、`ind_bat` 全部正常運作

---

### 改動六：CI/CD 觸發條件（.github/workflows/build.yml）

```yaml
# 改前
on:
  push:
    paths:
      - 'config/**'       # 只監聽 config/ 目錄

# 改後
on:
  push:
    paths:
      - 'config/**'       # 監聽 config/ 目錄
      - 'build.yaml'      # 也監聽 build.yaml
```

**為什麼要改？**

`build.yaml` 放在專案根目錄（不在 `config/` 裡），但原本的 workflow 只監聽 `config/**` 的改動。這導致：

1. 我改了 `build.yaml`（啟用 rgbled_adapter）並 push
2. GitHub Actions **沒有觸發** build，因為改動的檔案不在 `config/` 裡
3. 你下載到的是**舊的韌體**（沒有 rgbled_adapter），LED 當然不亮

加上 `'build.yaml'` 後，以後改 build 目標也會自動觸發編譯。

---

## 第三部分：CI/CD 完整流程

### 什麼是 CI/CD？

- **CI（Continuous Integration）**：持續整合，每次 push 程式碼都自動編譯
- **CD（Continuous Delivery）**：持續交付，編譯結果自動可供下載

在這個專案中，CI/CD 讓你不需要安裝任何開發工具，就能把設定改動變成可刷入的韌體。

### 完整流程圖

```
[你改設定檔]
      │
      ▼
[git push 到 GitHub]
      │
      ▼
[GitHub 檢查：改動的檔案路徑有沒有匹配 workflow 的 paths？]
      │
      ├── config/** 有改動？ → 觸發 ✓
      ├── build.yaml 有改動？ → 觸發 ✓（新加的）
      └── 其他檔案？ → 不觸發 ✗
      │
      ▼
[GitHub Actions 啟動 build.yml workflow]
      │
      ▼
[讀取 build.yaml，知道要 build 3 個目標]
      │
      ├── 1. seeeduino_xiao_ble + FrostOrtho_R + rgbled_adapter + studio-rpc-usb-uart
      ├── 2. seeeduino_xiao_ble + FrostOrtho_L + rgbled_adapter
      └── 3. seeeduino_xiao_ble + settings_reset
      │
      ▼
[West 根據 config/west.yml 下載依賴模組]
      │
      ├── ZMK 核心（cormoran 的 v0.3-branch+dya 分支，支援 DYA Studio）
      ├── zmk-pmw3610-driver（inorichi 的軌跡球驅動）
      ├── zmk-rgbled-widget（caksoylar 的 v0.3-branch，支援 LED 功能）
      ├── zmk-module-ble-management（DYA Studio 藍牙管理）
      ├── zmk-module-battery-history（DYA Studio 電池歷史）
      ├── zmk-module-settings-rpc（DYA Studio 設定 RPC）
      ├── zmk-module-runtime-input-processor（DYA Studio 動態輸入處理）
      └── zmk-behavior-runtime-sensor-rotate（DYA Studio 動態旋鈕）
      │
      ▼
[CMake 設定 + Zephyr 編譯系統]
      │
      ├── 讀取 .dtsi + .overlay → 產生完整的 Device Tree
      ├── 讀取 .conf + Kconfig → 決定啟用哪些功能
      ├── 讀取 .keymap → 編譯鍵位配置
      └── 連結所有模組 → 生成 .uf2 韌體檔案
      │
      ▼
[3 個 .uf2 檔案打包成 firmware.zip]
      │
      ▼
[上傳到 GitHub Actions Artifacts]
      │
      ▼
[你從 Actions 頁面下載 firmware.zip]
      │
      ▼
[解壓取得 .uf2，拖入鍵盤刷入]
```

### 為什麼改完 push 就能用？

因為整條鏈是自動化的：

1. **build.yaml** 定義「要編譯什麼硬體組合」
2. **west.yml** 定義「要用哪些模組、哪個版本」
3. **config/ 裡的檔案** 定義「鍵盤的功能和行為」
4. **GitHub Actions** 在雲端把以上全部組合起來，用 Zephyr 工具鏈編譯成韌體
5. 你只需要下載結果，拖入鍵盤

完全不需要在自己電腦上安裝 Zephyr SDK、CMake、West 等開發工具。

---

## 第四部分：韌體刷入與疑難排解

### 刷入步驟

1. USB 連接**右半**（Central，有軌跡球的那邊）
2. 快速**雙擊 RST 按鈕**（兩下之間間隔要短）
3. 電腦上會出現一個叫 **XIAO-SENSE** 的磁碟
4. 把 `FrostOrtho_R-seeeduino_xiao_ble-zmk.uf2` 拖進去
5. 磁碟會自動消失，代表刷入完成
6. 拔掉 USB，換接**左半**，重複步驟 2-5，使用 `FrostOrtho_L` 的 UF2

### 什麼時候需要 settings_reset？

| 狀況 | 需要 reset？ |
|------|------------|
| 只改了 keymap / conf | 不需要，直接刷 |
| 左右半無法連線 | 需要 |
| 要清除所有藍牙配對 | 需要 |
| 換了 ZMK 版本（west.yml 大改） | 建議做 |

**Reset 步驟：**
1. 右半刷 `settings_reset` UF2 → 拔掉
2. 左半刷 `settings_reset` UF2 → 拔掉
3. 右半刷正常的 `FrostOrtho_R` UF2
4. 左半刷正常的 `FrostOrtho_L` UF2
5. 兩邊都開電源，等它們自動配對
6. 電腦上**刪除舊的 FrostOrtho 藍牙配對**，重新配對

### 軌跡球問題

你遇到的「要傾斜才有反應」是物理問題（感測器對焦距離），不是韌體問題。詳見前面的對話討論。

### DYA Studio

FrostOrtho 支援 DYA Studio（https://studio.dya.cormoran.works/），可以在瀏覽器中即時修改部分鍵位，不需要重新編譯韌體。但進階功能（層結構、軌跡球設定、LED 設定）仍需要改 config 檔案並重新 build。
