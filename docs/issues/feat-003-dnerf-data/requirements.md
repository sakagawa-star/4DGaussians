# feat-003 要求仕様書: D-NeRFデータ準備

## 1.1 プロジェクト概要

4DGaussians の動作確認（学習→レンダリング→評価）の第一歩として、D-NeRF 合成シーンのデータセットを取得し、本体コードが読み込める構成に配置する。本体は `source_path` 直下の `transforms_train.json` の有無で Blender（D-NeRF）形式と判定する（`scene/__init__.py:48`）ため、各シーンを `data/dnerf/{scene}/` に正規化配置することが要求の核となる。

本案件の成果物はデータファイル（`data/` 配下）のみであり、`data/` は `.gitignore` 管理外のためコミットしない。コードの変更は行わない。

## 1.2 用語定義

- **D-NeRF データセット**: 単眼動的シーンの合成データ。Blender 形式（NeRF synthetic 形式の時間拡張）で、`transforms_{train,val,test}.json` と対応画像からなる
- **シーン**: `bouncingballs`, `hellwarrior`, `hook`, `jumpingjacks`, `lego`, `mutant`, `standup`, `trex`（`arguments/dnerf/` に設定がある8シーン）の個別データ単位。`data.zip` が実際に含むシーン一覧は U2（design.md）で実構成から確定する
- **Blender 形式**: `source_path` 直下に `transforms_train.json` を持つデータ構成。本体が `sceneLoadTypeCallbacks["Blender"]`（`readNerfSyntheticInfo`）で読み込む
- **正規化配置**: zip 展開で得られた構成を、本体が要求する `data/dnerf/{scene}/` 構成へ移動・整形すること
- **transforms json**: `camera_angle_x` とフレーム配列 `frames`（各要素に `file_path` / `time` / `transform_matrix`）を持つカメラ・時刻定義ファイル

## 1.3 機能要求一覧

### FR-001: data.zip のダウンロード

- D-NeRF の `data.zip` を Dropbox（`https://www.dropbox.com/s/0bf6fl0ye2vz3vr/data.zip?dl=0`）からダウンロードする
- ダウンロード先は作業用一時パス（リポジトリ外、例: `/tmp` 配下）とし、リポジトリを汚さない
- 優先度: Must
- 検証: zip ファイルが取得でき、`unzip -t` で破損がないこと

### FR-002: アーカイブの展開

- 取得した `data.zip` を一時ディレクトリへ展開する
- 展開後の最上位構成（`data/` を含むか、シーンが直下か）を確認する
- 優先度: Must
- 検証: 展開ディレクトリに D-NeRF シーン（少なくとも `bouncingballs`）の `transforms_train.json` が見つかること

### FR-003: data/dnerf/ への正規化配置

- 展開で得た各シーンを、本体が要求する `data/dnerf/{scene}/` 構成へ配置する
- zip 内の構成（`data/{scene}` か `{scene}` 直か等）に応じて移動・整形する
- 既存の `data/dnerf/{scene}` がある場合は上書きせず、状態を報告したうえで対応を判断する
- 優先度: Must
- 検証: `data/dnerf/bouncingballs/transforms_train.json` が存在すること

### FR-004: データ整合性検証

- 配置後、`bouncingballs` について本体が読み込み可能な構成であることを検証する
  - `transforms_train.json` / `transforms_test.json` が存在する
  - 両 json に `camera_angle_x` と `frames` が存在し、各 frame に `file_path` / `time` / `transform_matrix` が含まれる
  - `frames` の**全要素**について `file_path` に対応する画像（`{file_path}.png`）が実在する（途中フレームの欠落・展開漏れを検出するため、先頭1件ではなく全 frame を確認する。本体 `readCamerasFromTransforms` は全 frame をループして画像を開くため）
  - `transforms_val.json` は本体（`readNerfSyntheticInfo` / `read_timeline`）が読まないため検証対象外（存在すれば残すが必須ではない）
  - `data/dnerf/bouncingballs/` が `scene/__init__.py:48` の Blender 判定分岐に入る構成である（`transforms_train.json` が直下に存在）
- 優先度: Must
- 検証: 上記すべてが満たされること（README「判定基準」と一致）

### FR-005: 一時ファイルのクリーンアップ

- ダウンロードした zip および一時展開ディレクトリのうち、`data/dnerf/` への配置後に不要となるものを削除する
- 優先度: Should
- 検証: リポジトリ外の一時パスに大容量の残骸が残らないこと

## 1.4 非機能要求

- **再現性**: 手順（URL・コマンド列）を design.md に明記し、`/clear` 後でも同一手順で再実行できること
- **隔離性**: ダウンロード・展開はリポジトリ外の一時パスで行い、最終成果物のみ `data/dnerf/` に置く
- **容量**: 取得・展開・配置を通じて `/data`（空き約17T）の容量を逼迫しないこと
- **ネットワーク**: 外部（Dropbox）への HTTPS アクセスが可能であること

## 1.5 制約条件

- `data/` は `.gitignore` で管理外（9〜10行目）。**データはコミットしない**
- 4DGaussians 本体のコードは変更しない（CLAUDE.md 開発方針）
- DL ツールは環境に存在するもの（wget / curl / unzip、いずれも確認済み）を用いる
- 本案件では学習・レンダリング・評価は実行しない（feat-004 以降）

## 1.6 優先順位

1. FR-001（ダウンロード）= Must
2. FR-002（展開）= Must
3. FR-003（正規化配置）= Must
4. FR-004（整合性検証）= Must
5. FR-005（クリーンアップ）= Should
