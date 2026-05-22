# feat-003 機能設計書: D-NeRFデータ準備

## 1.1 対応要求マッピング

| 要求 | 設計箇所 |
|------|----------|
| FR-001 ダウンロード | §1.4.1 |
| FR-002 展開 | §1.4.2 |
| FR-003 正規化配置 | §1.4.3 |
| FR-004 整合性検証 | §1.4.4 |
| FR-005 クリーンアップ | §1.4.5 |
| エラーハンドリング・境界条件 | §1.4.6 |

## 1.2 システム構成

```
Dropbox (data.zip)
   │  FR-001: wget/curl で取得（dl=1）
   ▼
$WORK/data.zip                       … リポジトリ外の一時パス
   │  FR-002: unzip -t（破損検査）→ unzip 展開
   ▼
$WORK/extract/ …(zip内構成)          … data/{scene} もしくは {scene} 直下
   │  FR-003: 実構成を確認し data/dnerf/ へ移動・正規化
   ▼
data/dnerf/{scene}/                  … 本体が読み込む最終配置（.gitignore 管理外）
   │  FR-004: json 構造・画像実在・Blender 判定構成を検証
   ▼
（FR-005: $WORK を削除）
```

- 本体の読み込み経路（参考・変更しない）:
  - シーン判別: `scene/__init__.py:48` — `source_path/transforms_train.json` が存在 → `sceneLoadTypeCallbacks["Blender"]`
  - 読み込み: `scene/dataset_readers.py:313 readNerfSyntheticInfo` → `readCamerasFromTransforms`（`transforms_train.json` / `transforms_test.json`、各 frame の `file_path + extension` 画像）、`read_timeline`（train/test json の `time`）

## 1.3 技術スタック

- ダウンロード: `wget`（主）/ `curl -L`（フォールバック）。いずれも環境に存在
- 展開: `unzip`（`-t` で整合検査、`-q` で展開）
- 検証: `.venv/bin/python`（json 構造・画像存在チェック）。本体 import は不要（ファイル検証のみ）
- 一時パス: `/data` 配下（空き約17T）。`/tmp` は容量・tmpfs の懸念があるため使わない

## 1.4 各機能の詳細設計

共通の作業変数:

```bash
WORK=/data/sakagawa/tmp/feat003-dnerf      # リポジトリ外の一時パス
REPO=/data/sakagawa/4DGaussians
URL="https://www.dropbox.com/s/0bf6fl0ye2vz3vr/data.zip?dl=1"   # dl=1 で実体を取得
mkdir -p "$WORK"
```

### 1.4.1 FR-001: data.zip のダウンロード

```bash
wget -O "$WORK/data.zip" "$URL"
# wget が失敗した場合のフォールバック:
# curl -L -o "$WORK/data.zip" "$URL"
ls -lh "$WORK/data.zip"
```

- **データフロー**: Dropbox → `$WORK/data.zip`
- **処理ロジック**: `dl=1` を付与し HTML プレビューではなく zip 実体を取得する。`-O`/`-o` で出力先を明示
- **検証**: ファイルサイズが妥当（数百 MB 〜 GB オーダー、HTML 数 KB ではない）であること

### 1.4.2 FR-002: アーカイブの展開

```bash
unzip -t "$WORK/data.zip" >/dev/null && echo "zip integrity ok"
mkdir -p "$WORK/extract"
unzip -q "$WORK/data.zip" -d "$WORK/extract"
# 展開後の最上位構成と transforms_train.json の位置を確認
find "$WORK/extract" -maxdepth 3 -name transforms_train.json | head
ls -la "$WORK/extract"
```

- **データフロー**: `$WORK/data.zip` → `$WORK/extract/`
- **処理ロジック**: `-t` で破損検査を先行。`find` で `transforms_train.json` を持つディレクトリ（=各シーンのルート）を特定し、その親が「シーン群を束ねるディレクトリ」かを判断する
- **境界条件**: zip 内が `data/{scene}` 構成か `{scene}` 直下構成かは §1.4.3 の配置分岐で吸収する

### 1.4.3 FR-003: data/dnerf/ への正規化配置

`transforms_train.json` を持つディレクトリの**親ディレクトリ**（シーン群の入れ物）を `SRC` とし、その直下の各シーンを `data/dnerf/` へ移動する。

```bash
cd "$REPO"
mkdir -p data/dnerf

# シーンルート（transforms_train.json の所在）から親を特定
SCENE_DIR=$(dirname "$(find "$WORK/extract" -maxdepth 3 -name transforms_train.json | head -1)")
SRC=$(dirname "$SCENE_DIR")     # 例: $WORK/extract/data  または  $WORK/extract
echo "SRC=$SRC"
ls -la "$SRC"

# SRC 直下の各シーンを data/dnerf/ へ移動（既存があれば上書きしない）
for d in "$SRC"/*/; do
  name=$(basename "$d")
  if [ -e "data/dnerf/$name" ]; then
    echo "SKIP（既存）: data/dnerf/$name"
  else
    mv "$d" "data/dnerf/$name"
    echo "MOVED: data/dnerf/$name"
  fi
done
ls -la data/dnerf
```

- **データフロー**: `$WORK/extract/.../{scene}` → `data/dnerf/{scene}`
- **処理ロジック**: zip 内構成差（`data/` の有無）を `find`→`dirname` で吸収し、シーン単位で移動。`bouncingballs` を含む全シーンを保持する（zip は一括、容量潤沢）
- **エラーハンドリング**: 既存ディレクトリは上書きせず SKIP 表示（§1.4.6 E5）
- **境界条件**: `SRC` 直下にシーン以外のファイル（README 等）が混在する場合は移動対象から外す（ディレクトリのみ対象）

### 1.4.4 FR-004: データ整合性検証

```bash
"$REPO/.venv/bin/python" - <<'PY'
import json, os
base = "data/dnerf/bouncingballs"
ok = True
for f in ["transforms_train.json", "transforms_test.json"]:
    p = os.path.join(base, f)
    assert os.path.exists(p), f"missing: {p}"
    d = json.load(open(p))
    assert "camera_angle_x" in d, f"no camera_angle_x in {f}"
    assert d.get("frames"), f"no frames in {f}"
    missing = []
    for fr in d["frames"]:
        for k in ("file_path", "time", "transform_matrix"):
            assert k in fr, f"frame missing {k} in {f}"
        img = os.path.join(base, fr["file_path"] + ".png")
        if not os.path.exists(img):
            missing.append(img)
    assert not missing, f"{f}: {len(missing)} missing images, e.g. {missing[:3]}"
    print(f"{f}: frames={len(d['frames'])} all images present")
# Blender 判定構成（transforms_train.json が source_path 直下）
assert os.path.exists(os.path.join(base, "transforms_train.json"))
print("OK: bouncingballs is a valid Blender(D-NeRF) scene")
PY
```

- **データフロー**: `data/dnerf/bouncingballs/` の json・画像を読み、構造と実在を確認
- **処理ロジック**: README「判定基準」1〜4 を検査。train/test 両 json の**全 frame** について `file_path`（例 `./train/r_000`）+ `.png` の実在を確認し、途中フレームの欠落・展開漏れを検出する（本体 `readCamerasFromTransforms`（`dataset_readers.py:269` 〜）は全 frame をループして `Image.open` するため、先頭1件検証では欠落を見逃す）
- **境界条件**:
  - `file_path` の表記（先頭 `./` の有無）は `os.path.join` が吸収。拡張子は本体既定（`extension=".png"`）に一致
  - 本体が必須とするのは `transforms_train.json` / `transforms_test.json` のみ。`readNerfSyntheticInfo`（`dataset_readers.py:313`）と `read_timeline`（同 `:298`）は val を読まない。`transforms_val.json` は存在すれば残すが検証・読み込み対象外
  - 本体は `source_path` を `arguments/__init__.py:65` で `os.path.abspath` 絶対化したうえで画像パスを構築する（`dataset_readers.py:270,277` で path を二重 join するが、絶対パスが優先され正しく解決される）。本検証スクリプトは相対 base でファイル実在のみを確認するものであり、本体経由の実読み込みは feat-004 で確認する（ADR-6）

### 1.4.5 FR-005: 一時ファイルのクリーンアップ

```bash
# FR-004（§1.4.4）が合格したことを確認してから実行する。
# 検証失敗時はこのステップをスキップし $WORK を残す（§1.4.6 E7）。
rm -rf "$WORK"
echo "cleaned: $WORK"
```

- **処理ロジック**: 検証合格後に一時パス（zip・展開物）を削除。`data/dnerf/` の成果物は残す
- **境界条件**: FR-004 が失敗した場合はクリーンアップを保留し、`$WORK` を残して原因調査できるようにする

### 1.4.6 エラーハンドリングと境界条件

| ID | 事象 | 対応 |
|----|------|------|
| E1 | Dropbox が HTML（プレビュー）を返す | URL に `dl=1` を付与済み。サイズ・`unzip -t` で検知し再取得 |
| E2 | wget 失敗（権限・プロキシ・ネットワーク設定が原因） | `curl -L` にフォールバック |
| E3 | zip 破損（`unzip -t` 失敗） | `$WORK/data.zip` を削除し再ダウンロード |
| E4 | 展開構成が想定と異なる | `find ... transforms_train.json`→`dirname` で実構成を特定して配置（§1.4.3） |
| E5 | `data/dnerf/{scene}` が既存 | 上書きせず SKIP 表示。要否をユーザーに確認 |
| E6 | 容量不足 | `df -h /data` で事前確認（空き約17T、想定問題なし） |
| E7 | 検証失敗（json/画像欠落） | クリーンアップを保留し `$WORK` を残して調査 |

## 1.5 状態遷移

```
[未取得] --FR-001--> [zip取得済] --FR-002--> [展開済] --FR-003--> [配置済]
   --FR-004--> [検証合格] --FR-005--> [完了]
                  |失敗(E7)
                  └--> [調査: $WORK 保持]
```

## 1.6 ファイル・ディレクトリ設計

- **生成（成果物・git管理外）**: `data/dnerf/{scene}/`（`transforms_train.json` / `transforms_test.json`（および存在すれば `transforms_val.json`）と、`train/`・`test/`（および `val/`）配下の画像）
- **一時（削除対象）**: `$WORK = /data/sakagawa/tmp/feat003-dnerf/`（`data.zip`・`extract/`）
- **コミット対象**: 本案件ドキュメント（`docs/issues/feat-003-dnerf-data/*.md`）と必要に応じた `CLAUDE.md`・`BACKLOG.md` の更新のみ。`data/` はコミットしない

## 1.7 インターフェース定義

- 入力: Dropbox URL（`data.zip?dl=1`）
- 出力: `data/dnerf/{scene}/`（本体の `train.py -s data/dnerf/{scene}` が読む `source_path`）
- 後続接続: feat-004 が `python train.py -s data/dnerf/bouncingballs --expname "dnerf/bouncingballs" --configs arguments/dnerf/bouncingballs.py` で使用

## 1.8 ログ・デバッグ設計

- 各ステップで `ls -lh` / `find` / 検証スクリプトの print により進捗と結果を標準出力へ表示
- FR-004 の検証スクリプトは frame 数・サンプル画像パスを出力し、配置の妥当性を可視化
- 失敗時は `$WORK` を保持（E7）し、`unzip -l "$WORK/data.zip"` 等で再調査可能にする

## 設計判断の記録（ADR簡易版）

### ADR-1: Dropbox URL は `dl=1` で直接ダウンロード
`dl=0`（既定）は HTML プレビューを返す。`dl=1` で zip 実体を取得し、wget/curl による自動化を可能にする。

### ADR-2: 一時パスはリポジトリ外（`/data` 配下）に置く
`/tmp` は容量・tmpfs（メモリ消費）の懸念がある。空き約17T の `/data` 配下に `$WORK` を置き、展開物でリポジトリ作業ツリーを汚さない。

### ADR-3: 全シーンを一括展開・保持する
`data.zip` は D-NeRF 全シーンを一括で含む。`bouncingballs` のみ抽出するより、全シーンを `data/dnerf/` に保持する方が単純で、後続シーン（lego 等）の検証にも流用できる。容量も潤沢。

### ADR-4: 配置は zip 実構成を確認して正規化する
zip 内が `data/{scene}` か `{scene}` 直下かを決め打ちせず、`find transforms_train.json`→`dirname` で実構成を特定して `data/dnerf/` へ移動する。構成差に頑健。

### ADR-5: `data/` はコミットしない
`.gitignore`（9〜10行目）で `data/` は管理外。feat-001/002 と同じく成果物のうちドキュメントのみコミットし、データはコミットしない。

### ADR-6: 検証はファイル存在＋json 構造＋画像実在まで（学習は feat-004）
本案件のスコープは「データが揃い Blender データセットとして認識される構成になっている」こと。実際の学習完走・本体 import 経由の読み込みは feat-004 で確認し、責務を分離する。

## 未検証事項（実装時に実地検証）

- **U1**: `data.zip` 展開後の最上位構成（`data/` を含むか、シーン直下か）。→ §1.4.3 の `find`→`dirname` で吸収するが、実構成を確認してから移動する
- **U2**: `data.zip` の実サイズと含まれるシーン一覧（`bouncingballs` 以外の網羅範囲）
- **U3**: frame の `file_path` 表記（先頭 `./` の有無）と画像拡張子。→ §1.4.4 で実ファイル存在により確認
- **U4**: Dropbox の挙動（リダイレクト・レート制限）。失敗時は §1.4.6 E1/E2 で対処

## 実装結果（2026-05-22）

設計どおり FR-001〜005 を実行し、手動テスト合格・クローズ。

### 判定結果

- **FR-001**: `data.zip` を取得（246MB = 257,793,177 bytes、`application/binary`、`unzip -t` 整合OK）。Dropbox は 302 リダイレクト後 200 OK（U4 決着、`dl=1` で実体取得）
- **FR-002**: 展開成功。最上位構成は `extract/data/{scene}`（**U1 決着**: zip 内が `data/` を含む）
- **FR-003**: `find`→`dirname` で `SRC=extract/data` を特定し、8シーンを `data/dnerf/` へ移動（既存衝突なし）
- **FR-004**: `bouncingballs` 検証合格 — `transforms_train.json` frames=150 / `transforms_test.json` frames=20、**全 frame の画像が実在**、Blender 判定構成OK。`file_path` は `./train/r_000` 形式（先頭 `./` あり、`.png` 拡張子。**U3 決着**）。`transforms_val.json`・`val/` も同梱（本体は読まないため検証対象外）
- **FR-005**: 一時パス `$WORK` 削除完了

### 取得シーン一覧（U2 決着）

`data.zip` は D-NeRF 全8シーンを含み、`arguments/dnerf/` の設定群と完全一致:

| シーン | train | test | シーン | train | test |
|--------|-------|------|--------|-------|------|
| bouncingballs | 150 | 20 | mutant | 150 | 20 |
| hellwarrior | 100 | 20 | standup | 150 | 20 |
| hook | 100 | 20 | trex | 200 | 20 |
| jumpingjacks | 200 | 20 | lego | 50 | 20 |

配置容量: `data/dnerf/` 計 260M（`/data` 空き17T）。`data/` は `.gitignore` で管理外（`git check-ignore` 確認済み、コミット対象外）。

### 計画外対処

なし（設計どおり完走。U1〜U4 はすべて設計の分岐・検証で吸収）。
