# モジュール構成別 `build.yaml` ガイド

このリポジトリでは、センサーごとのハードウェア設定を snippet に分離します。`snippet` は空白区切りで複数指定でき、`shield` は左から順に役割・キーボード・モジュールを指定します。

## snippet 名

| 用途 | snippet |
| --- | --- |
| PMW3610-alt トラックボール | `trackball_pmw3610_alt` |
| PAT9125 トラックボール | `trackball_pat9125` |
| PAW3222 トラックボール | `trackball_paw3222` |
| 左側 EC11 エンコーダ | `encoder_ec11` |

`encoder_ec11` は既存の `left_encoder`（A: P1.15、B: P1.14、60 steps）を有効化します。回転時の動作は `config/MKB.keymap` の各レイヤーにある `sensor-bindings` が決めます。

トラックボール snippet は、central では `MKB_TB_CENTRAL`、peripheral では `MKB_TB_PERIPHERAL` を同じ `shield` 行に必ず指定します。

## 最初に試す構成

以下は central と peripheral を別成果物としてビルドする例です。`Central`、`Peripheral`、`ENC_L` は `zmk-module-xiaord` 側で提供されるモジュール shield を前提にしています。`MKB_TB_CENTRAL` と `MKB_TB_PERIPHERAL` はこのリポジトリで追加したトラックボール接続用 shield です。

### 左 central: エンコーダ / 右 peripheral: PAW3222

```yaml
include:
  - board: seeeduino_xiao_ble
    shield: Central MKB_L ENC_L rgbled_adapter
    snippet: encoder_ec11
    artifact-name: MKB_L_ENC_Central

  - board: seeeduino_xiao_ble
    shield: Peripheral MKB_R MKB_TB_PERIPHERAL rgbled_adapter
    snippet: trackball_paw3222
    artifact-name: MKB_R_PAW3222_Peripheral
```

### 左 central: PAT9125 / 右 peripheral: PAW3222

```yaml
include:
  - board: seeeduino_xiao_ble
    shield: Central MKB_L MKB_TB_CENTRAL rgbled_adapter
    snippet: trackball_pat9125
    artifact-name: MKB_L_PAT9125_Central

  - board: seeeduino_xiao_ble
    shield: Peripheral MKB_R MKB_TB_PERIPHERAL rgbled_adapter
    snippet: trackball_paw3222
    artifact-name: MKB_R_PAW3222_Peripheral
```

この 2 構成に必要な 3 成果物（左 EC11 central、左 PAT9125 central、右 PAW3222 peripheral）は `build.yaml` に登録済みです。右 PAW3222 peripheral の成果物は 2 構成で共通です。

## central / peripheral 差分を分離する提案

PAW3222 を含む各トラックボール snippet は、センサー固有の設定だけを持ちます。central / peripheral の接続は shield で選択し、次の 3 層に分けます。

1. `listners.dtsi` は左右の `zmk,input-split` と、central 側で受ける listener だけを定義する。
2. 各トラックボール snippet は SPI/I2C、`trackball` センサーノード、センサー固有の input processor 設定だけを定義する。
3. `MKB_TB_CENTRAL` は central listener を、`MKB_TB_PERIPHERAL` は peripheral 用 split を、それぞれ `device = <&trackball>` へ接続する。`Central` / `Peripheral` shield は ZMK の split role を選択する。

`/aliases` には `trackball-split`、`trackball-central-listener` のような意味のある名前を置き、C 側の `DT_ALIAS()` やデバッグ時の参照を安定させます。ただし Devicetree の alias は `device = <&node_label>` の phandle を別ノードへ置換する仕組みではありません。そのため、DTS 内の接続自体は上記の shield overlay で node label を明示して行います。alias だけで central / peripheral の差分を吸収しようとすると、DTC の phandle 解決では期待どおりに機能しません。

この分割により、PAW3222 の central / peripheral 用 snippet を分けず、`trackball_paw3222` を両方の役割で使えます。左右は `MKB_L`／`MKB_R` shield で選び、トラックボール接続用 shield は左右を区別しません。`listners.dtsi` の `listner` は既存互換のため維持しています。
