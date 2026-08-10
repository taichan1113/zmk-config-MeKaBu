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

`central` または `peripheral` を、各センサー snippet の前に指定します。左右は `MKB_L` / `MKB_R` shield で決め、split role は shield ではなく snippet で決めます。

このリポジトリの keymap は `config/MKB.keymap` で固定のため、各 build 定義には `cmake-args: -DKEYMAP_FILE=$GITHUB_WORKSPACE/config/MKB.keymap` を指定します。

## 最初に試す構成

以下は central と peripheral を別成果物としてビルドする例です。`central` と `peripheral` はこのリポジトリの role snippet です。

### 左 central: エンコーダ / 右 peripheral: PAW3222

```yaml
include:
  - board: seeeduino_xiao_ble
    shield: MKB_L rgbled_adapter
    snippet: central encoder_ec11
    cmake-args: -DKEYMAP_FILE=$GITHUB_WORKSPACE/config/MKB.keymap
    artifact-name: MKB_L_ENC_Central

  - board: seeeduino_xiao_ble
    shield: MKB_R rgbled_adapter
    snippet: peripheral trackball_paw3222
    cmake-args: -DKEYMAP_FILE=$GITHUB_WORKSPACE/config/MKB.keymap
    artifact-name: MKB_R_PAW3222_Peripheral
```

### 左 central: PAT9125 / 右 peripheral: PAW3222

```yaml
include:
  - board: seeeduino_xiao_ble
    shield: MKB_L rgbled_adapter
    snippet: central trackball_pat9125
    cmake-args: -DKEYMAP_FILE=$GITHUB_WORKSPACE/config/MKB.keymap
    artifact-name: MKB_L_PAT9125_Central

  - board: seeeduino_xiao_ble
    shield: MKB_R rgbled_adapter
    snippet: peripheral trackball_paw3222
    cmake-args: -DKEYMAP_FILE=$GITHUB_WORKSPACE/config/MKB.keymap
    artifact-name: MKB_R_PAW3222_Peripheral
```

上記は組み合わせの書き方を示す例です。必要な成果物だけを `build.yaml` の `include` に登録してください。右 PAW3222 peripheral の成果物は 2 構成で共通です。

## central / peripheral 差分を分離する提案

PAW3222 を含む各トラックボール snippet は、センサー固有の設定と role に応じた接続先を持ちます。central / peripheral の判定は snippet で行い、次の 3 層に分けます。

1. `listners.dtsi` は `zmk,input-split` と、local / forwarded input を受ける listener を定義する。
2. `central` / `peripheral` snippet は ZMK の split role を設定する。`central` は forwarded input の listener を有効化する。
3. `central` snippet が定義する前処理マクロに応じて、各トラックボール snippet は central では local listener、peripheral では `trackball_split` を実センサーへ接続する。

`/aliases` には `trackball-split`、`trackball-listener-central` のような意味のある名前を置き、C 側の `DT_ALIAS()` やデバッグ時の参照を安定させます。ただし Devicetree の alias は `device = <&node_label>` の phandle を別ノードへ置換する仕組みではありません。そのため、DTS 内の接続自体は各トラックボール snippet で node label を明示して行います。alias だけで central / peripheral の差分を吸収しようとすると、DTC の phandle 解決では期待どおりに機能しません。

この分割により、PAW3222 の central / peripheral 用 snippet を分けず、`trackball_paw3222` を両方の役割で使えます。左右は `MKB_L` / `MKB_R` shield、role は `central` / `peripheral` snippet として完全に独立します。`listners.dtsi` の `listner` は既存互換のため維持しています。
