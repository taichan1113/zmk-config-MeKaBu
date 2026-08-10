# MeKaBu ユーザーガイド

MeKaBu は ZMK を使う左右分割キーボードです。トラックボールと EC11 エンコーダは、`build.yaml` の snippet を選ぶことで構成します。この README はファームウェアを選んで書き込む利用者向けの案内です。

## まず選ぶもの

左右それぞれについて、**役割**と**接続するモジュール**を選びます。左右（`MKB_L` / `MKB_R`）と central / peripheral の役割は独立しています。

| 分類 | snippet | 用途 |
| --- | --- | --- |
| 役割 | `central` | PC と Bluetooth/USB 接続する側。もう一方からの入力も受け取ります。 |
| 役割 | `peripheral` | central に接続する側。 |
| トラックボール | `trackball_paw3222` | PAW3222 モジュール。 |
| トラックボール | `trackball_pat9125` | PAT9125 モジュール。 |
| トラックボール | `trackball_pmw3610_alt` | PMW3610-alt モジュール。 |
| エンコーダ | `encoder_ec11` | 左側 EC11 エンコーダ。 |

同じ半分に取り付けられている物理モジュールに対応する snippet だけを指定してください。ピンを共有するモジュールを同時に指定すると、正常に動作しません。

## `build.yaml` の編集

`build.yaml` の `include` に、左右それぞれ1件ずつ build 定義を書きます。`snippet` は空白区切りで複数指定できます。

左を central + EC11、右を peripheral + PAW3222 とする例です。

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

PAT9125 と PAW3222 を使う例です。

```yaml
snippet: central trackball_pat9125
# 反対側
snippet: peripheral trackball_paw3222
```

このリポジトリの有効な build 定義と artifact 名は [build.yaml](build.yaml) が正です。詳しい組み合わせの書式は [build-guidelines.md](docs/build-guidelines.md) を参照してください。

## GitHub Actions で firmware を取得する

1. `build.yaml` を編集して commit・push します。GitHub の **Actions** タブから **Build ZMK firmware** を手動実行しても構いません。
2. workflow が成功したら、該当 run の下部にある **Artifacts** から `firmware` をダウンロードします。
3. zip を展開し、`artifact-name` に対応する `.uf2` ファイルを確認します。左・右・central・peripheral を取り違えないでください。

Actions が失敗した場合は、まず build 定義の snippet 名、shield 名、同時に指定したモジュールの競合を確認してください。Actions は新規環境で依存モジュールを取得するため、ローカルで以前に成功した build と結果が異なる場合があります。

## Seeed XIAO BLE への書き込み

1. 対象の半分を USB 接続します。
2. リセットボタンを素早く2回押し、USB ストレージとして認識させます。
3. 対象の `.uf2` ファイルを、そのドライブの直下へコピーします。
4. コピー後に自動で再起動します。認識されない場合は USB を抜き差しして確認してください。

central と peripheral は同じ build 世代の firmware を両方に書き込んでください。split 接続や Bluetooth 接続が不安定なときは、まず左右の役割、書き込んだ artifact、電源状態を確認します。設定を初期化する必要がある場合は、`settings_reset` artifact を**対象の半分だけ**に書き込む運用ができます。Bluetooth の保存情報などが消えるため、必要な場合だけ使用してください。

## 追加情報

- キーマップは [config/MKB.keymap](config/MKB.keymap) にあります。snippet は配線・センサーを選ぶためのもので、通常のキー配列は変えません。
- USB を PC に接続するのは central 側です。peripheral 側の USB 接続は書き込みや充電に使用します。
- firmware の書き込み中は、他方の半分や外部モジュールを不用意に抜き差ししないでください。
