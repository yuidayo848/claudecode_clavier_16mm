# Clavier ZMK Firmware

左右分割・トラックボール付きカスタムキーボードのZMKファームウェア設定です。

## ハードウェア構成

| 項目 | 内容 |
|------|------|
| マイコン (左右共通) | Seeed Studio XIAO nRF52840 |
| キー数 | 左: 4行×5列 (20キー) / 右: 4行×6列 (24キー) / 合計44キー |
| トラックボール | PMW3610 (左手側のみ) |
| 接続方式 | BLE 無線スプリット + USB (左側) |

## ピンアサイン

### 左手側 (Central) - 20キー

| 信号 | XIAO端子 | nRF52840 GPIO |
|------|----------|---------------|
| Row0 | A4 | P0.03 |
| Row1 | A10 | P0.28 |
| Row2 | A11 | P0.29 |
| Row3 | B8_TX | P1.11 |
| Col0 | NFC2 | P0.10 |
| Col1 | B9_RX | P1.12 |
| Col2 | A7_SCK | P1.13 |
| Col3 | A5_MISO | P1.14 |
| Col4 | A6_MOSI | P1.15 |

### 右手側 (Peripheral) - 24キー

| 信号 | XIAO端子 | nRF52840 GPIO |
|------|----------|---------------|
| Row0 | A4 | P0.03 |
| Row1 | A11 | P0.29 |
| Row2 | A9_SCL | P0.05 |
| Row3 | B8_TX | P1.11 |
| Col0 | A6_MOSI | P1.15 |
| Col1 | A5_MISO | P1.14 |
| Col2 | B9_RX | P1.12 |
| Col3 | A7_SCK | P1.13 |
| Col4 | A8_SDA | P0.04 |
| Col5 | A2 | P0.02 |

> 左右でピンアサインが異なります。右手側は左手側のPMW3610用SPI/GPIOピンをキーマトリクスに転用しています。

### PMW3610 ピンアサイン (左手側のみ)

| 信号 | XIAO端子 | nRF52840 GPIO | 備考 |
|------|----------|---------------|------|
| SCLK | A9_SCL | P0.05 | |
| SDIO | A8_SDA | P0.04 | MOSI/MISO共有（3線SPI） |
| CS | NFC1 | P0.09 | NFCピンをGPIOとして解放 |
| MOTION (割り込み) | A2 | P0.02 | |

## キーマップ

44キー (左5列 + 右6列) × 4行

### Layer 0: QWERTY (デフォルト)

```
Q    W    E    R    T  |  Y    U    I    O    P    [
A    S    D    F    G  |  H    J    K    L    ;    '
Z    X    C    V    B  |  N    M    ,    .    /    ]
CTRL GUI  ALT  LWR SPC|  SPC  RSE  ENT BSPC  DEL  \
```

### Layer 1: LOWER - `LWR` 長押し

```
1    2    3    4    5  |  6    7    8    9    0    -
!    @    #    $    %  |  ^    &    *    (    )    =
`    ~                 |                           +
```

### Layer 2: RAISE - `RSE` 長押し

```
ESC  F1   F2   F3   F4 |  F5   F6   F7   F8   F9   F10
TAB  F11  F12 INS      | ←    ↓    ↑    →   END  HOME
CAPS BT0  BT1  BT2 CLR | PGDN PGUP           PRTSC
```

### Layer 3: MOUSE (トラックボール操作時に自動有効化)

```
右手ホームポジション: J=右クリック  K=左クリック  L=中クリック
```

## ビルド・フラッシュ方法

### GitHub Actions (推奨)

1. このリポジトリをGitHubにpush
2. `.github/workflows/build.yml` が自動的にビルドを実行
3. Actionsタブ → 完了したワークフロー → Artifactsからダウンロード

### フラッシュ手順

1. XIAO nRF52840 のリセットボタンを**素早く2回押す**
2. `XIAO-SENSE` ドライブがPCに表示される
3. `.uf2` ファイルをドライブにコピー
4. コピー後にエラーダイアログ（0x800701B1）が出ることがあるが**無視してよい**（正常に書き込まれている）

### 初回ペアリング（または再ペアリング）手順

1. 左手側・右手側どちらかのみのファームウェアを書き換えた場合、通常は電源ON後に自動再接続される
2. **両側を同時に書き換えた場合**や接続できない場合は以下のリセット手順を実施：
   - `settings_reset` ファームウェアを左右両側に書き込む（`build.yml`に追加可能）
   - 続けて通常ファームウェア（`clavier_left` / `clavier_right`）を書き込む
   - 左右の電源をONにして30秒〜1分待つ

---

## 開発中に遭遇したトラブルと解決策

このセクションは、次回の自作キーボード製作時の参考のために、開発中に発生した問題と解決策を詳細に記録したものです。

---

### 問題1: トラックボール用コードを追加したらキーが一切反応しなくなった

#### 症状
- キースイッチのみで動いていたファームウェアにトラックボール関連のコードを追加したところ、左右両方のキーが全く反応しなくなった
- PCには「Clavier」と表示されるが、デバイスマネージャーで「キーボード」や「ヒューマンインターフェイスデバイス」ではなく「ほかのデバイス」として認識された

#### 原因
`Kconfig.defconfig` が空になっており、以下の重要な設定が欠落していた：

```
config ZMK_SPLIT_ROLE_CENTRAL
    default y
```

ZMKのスプリットキーボードでは、左手側が**Central（親機）**として動作しPCに接続する役割を持つ。この設定がないと左手側がCentralとして機能せず、USB HIDデバイスとして認識されない。

#### 解決策
`Kconfig.defconfig` に以下を記載する：

```
if SHIELD_CLAVIER_LEFT || SHIELD_CLAVIER_RIGHT

config ZMK_KEYBOARD_NAME
    default "clavier16"

config ZMK_SPLIT
    default y

if SHIELD_CLAVIER_LEFT

config ZMK_SPLIT_ROLE_CENTRAL
    default y

endif # SHIELD_CLAVIER_LEFT

endif # SHIELD_CLAVIER_LEFT || SHIELD_CLAVIER_RIGHT
```

#### 教訓
- ZMK Splitでは `Kconfig.defconfig` で `ZMK_SPLIT_ROLE_CENTRAL` を明示的に設定しないと左手側がUSBホストとして動作しない
- これはキーボード名の設定と同じファイルに書く

---

### 問題2: Windowsのデバイスマネージャーに「ほかのデバイス」として残り続ける

#### 症状
- ファームウェアを正しく書き直した後も、デバイスマネージャーで「ほかのデバイス」扱いのまま消えない
- キー入力が正常でない状態でPCに接続した際のドライバ情報がキャッシュされてしまっていた

#### 解決策
1. デバイスマネージャーで該当デバイスを右クリック→「デバイスのアンインストール」
2. USBケーブルを抜き差しして再認識させる

---

### 問題3: XIAOがPCから全く認識されなくなった

#### 症状
- UF2書き込み後、デバイスマネージャーにもエクスプローラーにも何も表示されない
- キーボードが起動しているかどうかも不明

#### 原因
書き込んだファームウェアに問題があり、XIAOが正常に起動できない状態になっていた。

#### 解決策
**ブートローダーモードへの強制移行**：
1. リセットボタンを**素早く2回押す**（ダブルクリック）
2. すぐにエクスプローラーを確認する
3. `XIAO-SENSE` ドライブが表示されたら、正常なUF2を書き込む

> ⚠️ ダブルクリックのタイミングがシビア。1回押しだとリセット、2回押しだとブートローダー起動。

---

### 問題4: UART(シリアル)ピンがキーマトリクスと競合する

#### 症状
- P1.11（Row3）と P1.12（Col1/Col2）がキーマトリクスに割り当てられているのに、一部のキーが反応しない

#### 原因
nRF52840のUARTドライバが起動時にP1.11（TX）とP1.12（RX）を自動的に占有する。

#### 解決策
`clavier_left.conf` と `clavier_right.conf` に以下を追加：

```conf
CONFIG_SERIAL=n
CONFIG_UART_CONSOLE=n
```

さらにoverlay内でuart0を明示的に無効化：

```dts
&uart0 {
    status = "disabled";
};
```

---

### 問題5: NFC用ピンがキーマトリクス・SPIと競合する

#### 症状
- P0.09（NFC1）をPMW3610のCS、P0.10（NFC2）をRow2として使いたいが動作しない

#### 原因
nRF52840はP0.09とP0.10をNFCアンテナとして予約している。

#### 解決策
`.conf` ファイルに以下を追加：

```conf
CONFIG_NFCT_PINS_AS_GPIOS=y
```

---

### 問題6: ビルドエラー - 存在しないKconfigシンボル

#### 症状
GitHub ActionsのビルドでKconfigエラーが発生：

```
error: Aborting due to Kconfig warnings
```

#### 原因と対策

| 誤ったオプション | 理由 | 正しい対応 |
|-----------------|------|-----------|
| `CONFIG_ZMK_POINTING=y` | urob fork には `ZMK_POINTING` シンボルが存在しない | 削除 |
| `CONFIG_ZMK_USB_LOGGING=y` + `CONFIG_SERIAL=n` | USB CDCログはSERIALサブシステムを必要とするが、SERIALをnにすると競合 | デバッグ時のみSERIAL=yに変更、またはUSB CDCノードをDTに追加が必要 |

---

### 問題7: トラックボール（PMW3610）が全く反応しない【最重要】

#### 症状
- キーは正常動作するがトラックボールが完全無反応
- 導通チェック（デジタルマルチメーター）では全ピン正常
- ログにPMW3610関連のメッセージが一切出力されない

#### 調査過程

**Step 1: 配線を疑う**  
→ デジタルマルチメーターの導通チェックで以下を確認：
  1. XIAOのピン ↔ 本体基板のコネクタランド
  2. 本体基板コネクタランド ↔ トラックボール基板コネクタランド
  3. トラックボール基板コネクタランド ↔ PMW3610本体ピン  
→ 全て導通OK。配線は問題なし。

**Step 2: デバッグログで調査**  
ファームウェアにUSBログを追加して調査。

```conf
CONFIG_SERIAL=y
CONFIG_ZMK_USB_LOGGING=y
CONFIG_LOG=y
CONFIG_PMW3610_LOG_LEVEL_DBG=y  # ← 実はこのシンボルは誤り（後述）
```

overlayにCDC ACM UARTノードを追加：

```dts
/ {
    chosen {
        zephyr,console = &cdc_acm_uart0;
    };
};

&usbd {
    cdc_acm_uart0: cdc_acm_uart0 {
        compatible = "zephyr,cdc-acm-uart";
    };
};
```

Tera Term（または PowerShell の SerialPort クラス）でCOMポートを開いてログを確認。

**Step 3: ドライバーソースを調査**  
inorichiのPMW3610ドライバーソース（`pmw3610.c`）を確認したところ：

```c
LOG_MODULE_REGISTER(pmw3610, CONFIG_INPUT_LOG_LEVEL);
```

PMW3610ドライバーは `CONFIG_PMW3610_LOG_LEVEL` ではなく **`CONFIG_INPUT_LOG_LEVEL`** を使っていた。そのため `CONFIG_PMW3610_LOG_LEVEL_DBG=y` は無効だった。

**Step 4: ドライバーREADMEを確認**  
inorichiドライバーのREADMEに以下の記載を発見：

```conf
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_ZMK_MOUSE=y
CONFIG_PMW3610=y   ← これが抜けていた！
```

#### 根本原因
**`CONFIG_PMW3610=y` が `.conf` ファイルに記載されていなかった。**

このオプションがないとPMW3610ドライバー自体がファームウェアにコンパイルされない。デバイスツリーにノードを書いても、ドライバーがなければセンサーは完全に無視される。

#### 解決策

`clavier_left.conf` に追加：

```conf
CONFIG_PMW3610=y
```

overlayに `zmk,input-listener` ノードを追加：

```dts
/ {
    trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;
    };
};
```

→ **これでトラックボールが動作するようになった。**

---

### 問題8: デバッグログ追加後に右手側が反応しなくなった

#### 症状
- トラックボール動作確認のためデバッグログ（`CONFIG_LOG_DEFAULT_LEVEL=4`）を追加したファームウェアを書き込んだ後、右手側のキーが全く反応しなくなった

#### 原因
`CONFIG_LOG_DEFAULT_LEVEL=4`（全モジュールをデバッグレベルに設定）を有効にしたことで、膨大な量のログメッセージが生成された。このログ処理がBLEスタックのCPU時間を奪い、左右のBLE接続（split接続）が維持できなくなった。

ログファイルを確認すると大量の `--- N messages dropped ---` が出ており、ログバッファが溢れていることが確認できた。

#### 解決策
デバッグログを全て削除し、プロダクション用の設定に戻す。

```conf
# 削除するもの
CONFIG_ZMK_USB_LOGGING=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4
CONFIG_PMW3610_LOG_LEVEL_DBG=y
CONFIG_SERIAL=y  # → n に戻す
```

#### 教訓
- `CONFIG_LOG_DEFAULT_LEVEL=4` はBLEスタックを含む全モジュールのデバッグログを有効化するため、スプリットキーボードのBLE接続に影響を与える可能性がある
- デバッグが終わったら必ずログ設定を元に戻す

---

### 問題9: 両側書き換え後に右手側が反応しなくなった（BLE再ペアリング）

#### 症状
- 左右両方のファームウェアを同時に書き換えた後、右手側のキーが反応しなくなった

#### 原因
ZMKはBLEペアリング情報をフラッシュに保存している。ファームウェアを書き換えるとフラッシュが消去されるため、ペアリング情報が失われる。特に両側同時に書き換えた場合は必ず再ペアリングが必要になる。

#### 解決策

**通常はしばらく待てば自動再接続される（30秒〜1分）。**

それでも繋がらない場合は `settings_reset` ファームウェアを使う：

1. `build.yml` に `settings_reset` ビルドを追加：

```yaml
- shield: settings_reset
  board: seeeduino_xiao_ble
```

2. `settings_reset` UF2を左右両側に書き込む
3. 続けて通常ファームウェアを左右両側に書き込む
4. 左右の電源をONにして30秒〜1分待つ

---

### 問題10: 右手側が電池では反応しない（USB接続時だけ動く）【ソフトウェアではなくハードウェア故障だった】

#### 症状
- 左手側をUSBでPCに接続し、右手側は電池＋電源スイッチのみで動かすと、右手側のキーが一切反応しない
- ファームウェアの書き直し・`settings_reset`によるBLE再ペアリング・電源投入順序の変更など、ソフトウェア側の対策をひと通り試しても改善しない
- 一時的に右手側もUSBでPCに直接つなぐと、なぜかキー入力が通ってしまう

#### 調査の迷走
この症状にBLE split接続の不具合を疑い、以下をかなり深く調査した。

- Kconfig・デバイスツリー（`col-offset`、`Kconfig.defconfig`のCentral宣言など）の再確認
- trackball_testブランチ→masterへのマージ差分の確認（右手側ファイルは無変更と判明）
- 右手側・左手側それぞれにUSB CDC ACMデバッグログを追加し、実機ログを取得
  - 右手側: `kscan_matrix_read` で正しくキー入力を検出できていることを確認
  - 左手側: `split_central_notify_func: [NOTIFICATION]` を確認し、BLE split接続自体は正常に機能していることを確認
- 「右手をUSBに繋いだから動いたのでは（BLE経由ではなく右手が直接PCに送っているのでは）」という疑問に対しては、ZMKの `CMakeLists.txt` を確認し、Peripheralビルドには `keymap.c` / `endpoints.c`（HIDレポート生成・USB出力）が一切コンパイルされないことを確認して否定した

→ **ここまでの調査で、ファームウェア・BLE周りは実際に正常であることが判明した。**

#### 真の原因
テスターで電圧を実測したところ、電池そのものは3.97Vで正常だったが、**電池と基板の接続部分（配線）が断線しており0Vだった**。

つまり「電源スイッチをONにする」という操作自体が、右手側の基板に電力を供給できていなかった。USB接続時だけ反応していたのは、USBが電池を経由しない別経路で給電していたため。ファームウェアやBLEには最初から問題がなかった。

#### 解決策
断線箇所を修復し、テスターで電池⇔基板間に電圧が来ていることを確認する。

#### 教訓
- **「左右分離型キーボードの片側だけ反応しない」場合、ソフトウェア（ファームウェア・BLEペアリング）を疑う前に、まず電池⇔基板間の電圧をテスターで実測して、そもそも基板に電気が来ているかを確認すべき**
- USB接続時は正常に見えても、電池だけで動かした時の給電経路は別物なので、必ず両方のケースを分けて検証する
- ログ解析やソースコード調査は「ソフトウェアが正常であること」を証明はできるが、それは「原因がハードウェアにある」ことの裏付けにもなる。切り分けとしては無駄ではない
- テスターでの電圧測定は、電池のリード線根本（電池自体の生死）と、基板側のコネクタ受け（実際に届いている電圧）の両方を測って比較すると、断線箇所を素早く特定できる

---

## 使用するZMK Fork・外部モジュール

| 名前 | 用途 | URL |
|------|------|-----|
| urob's ZMK fork | ZMK本体（ポインティングデバイス対応が充実） | https://github.com/urob/zmk |
| inorichi's PMW3610 driver | PMW3610トラックボールセンサードライバー | https://github.com/inorichi/zmk-pmw3610-driver |

### west.yml の設定例

```yaml
manifest:
  remotes:
    - name: urob
      url-base: https://github.com/urob
    - name: inorichi
      url-base: https://github.com/inorichi
  projects:
    - name: zmk
      remote: urob
      revision: main
      import: app/west.yml
    - name: zmk-pmw3610-driver
      remote: inorichi
      revision: main
  self:
    path: config
```

---

## PMW3610（3線SPI）のデバイスツリー設定

PMW3610はSDIOラインを送受信で共用する3線SPIを使用する。nRF52840 SPIMでは MOSI と MISO に同じピンを割り当てることで対応する。

```dts
&pinctrl {
    spi2_default: spi2_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK,  0, 5)>,
                    <NRF_PSEL(SPIM_MOSI, 0, 4)>,   /* SDIO */
                    <NRF_PSEL(SPIM_MISO, 0, 4)>;   /* SDIO（同じピン） */
        };
    };
    spi2_sleep: spi2_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK,  0, 5)>,
                    <NRF_PSEL(SPIM_MOSI, 0, 4)>,
                    <NRF_PSEL(SPIM_MISO, 0, 4)>;
            low-power-enable;
        };
    };
};

&spi2 {
    compatible = "nordic,nrf-spim";
    status = "okay";
    pinctrl-0 = <&spi2_default>;
    pinctrl-1 = <&spi2_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 9 GPIO_ACTIVE_LOW>;

    trackball: pmw3610@0 {
        compatible = "pixart,pmw3610";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
        automouse-layer = <3>;
    };
};
```

---

## PMW3610を使う際の必須設定チェックリスト

次回PMW3610を搭載するキーボードを作る際に確認するリスト：

- [ ] `west.yml` に `inorichi/zmk-pmw3610-driver` を追加
- [ ] `clavier_left.conf` に `CONFIG_PMW3610=y` を追加（**これがないとドライバーがコンパイルされない**）
- [ ] `clavier_left.conf` に `CONFIG_ZMK_MOUSE=y`, `CONFIG_INPUT=y`, `CONFIG_SPI=y` を追加
- [ ] overlayに `spi` ノード・`pmw3610` ノードを追加（MOSI/MISOに同じピンを指定）
- [ ] overlayに `zmk,input-listener` ノードを追加
- [ ] overlayで `chosen` に `zmk,pointing = &trackball` を追加
- [ ] CS として使うピンがNFCピンの場合、`CONFIG_NFCT_PINS_AS_GPIOS=y` を追加
- [ ] UARTと競合するピンがある場合、`CONFIG_SERIAL=n`, `CONFIG_UART_CONSOLE=n` を追加し、overlayで `&uart0 { status = "disabled"; };` を設定
- [ ] `Kconfig.defconfig` に `ZMK_SPLIT_ROLE_CENTRAL` を設定（左手側Central宣言）
