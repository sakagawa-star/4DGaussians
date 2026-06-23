# feat-011 要求仕様書: COLMAP 3.11.1 併設（rig 非互換回避）

本書は `docs/REQUIREMENTS_STANDARD.md` に準拠する。

---

## 1.1 プロジェクト概要

- **何を作るのか**: rig 必須化前の最終 minor 版である **COLMAP 3.11.1** を、vcpkg で**別 install prefix** にソースビルドして本マシンに併設する。既存の COLMAP 3.12.6（feat-008 導入）は温存し、4DGS の `colmap.sh`（`scripts/llff2colmap.py` 経由で「既知ポーズ＋単一カメラ共有 sparse モデル」を `point_triangulator` に渡す方式）を **4DGS 本体非改変**のまま 3.11.1 で実走できる状態を構築・実証する。
- **なぜ作るのか**: feat-008 で導入した COLMAP **3.12.6** は、3.12 で rig/frame モデルがネイティブ化（「各カメラ＝独立 rig」がデフォルト）されたため、`llff2colmap.py` が20画像を単一カメラ（CAMERA_ID=1）で共有させる前提と非互換になり、`point_triangulator` が `Check failed: existing_frame.RigId()==frame.RigId()` で SIGABRT する（旧 feat-010 の実装中に判明）。この結果 DyNeRF（feat-012）・multipleview（feat-013）の COLMAP 前処理が動かない。**4DGS 本体を改変せずに**この前処理を通すため、rig 非ネイティブの 3.11.1 を併設する。
- **誰が使うのか**: 本リポジトリで 4DGS を動かす開発者（複数人共用 GPU サーバー利用）。直接の利用者は後続案件 feat-012（DyNeRF）・feat-013（multipleview）。
- **どこで使うのか**: Ubuntu / A100-SXM4-40GB ×7（共用）/ uv 管理 `.venv`（Python 3.10）/ ヘッドレス（X ディスプレイ非前提）。CUDA Toolkit 11.6（`/usr/local/cuda-11.6`、ビルド用）。

## 1.2 用語定義

ドキュメント内・機能設計書・コードで同じ用語を使う。

- **COLMAP 3.12.6（既存）**: feat-008 で vcpkg ソースビルド導入した版。`/data/sakagawa/opt/vcpkg/installed/x64-linux/tools/colmap/colmap`、PATH ラッパー `~/.local/bin/colmap`。本案件で**変更・削除しない**（温存）。
- **COLMAP 3.11.1（本案件で導入）**: rig 必須化前の最終 minor 版。vcpkg versions DB に port-version 0〜4 が存在し、**最新の port-version 4**（git-tree `c166234c960ad821bfddccbe87089e1c3d5fa583`）を採用する。
- **rig/frame ネイティブ化**: COLMAP 3.12 で導入された再構成モデルの仕様変更。reconstruction 読込時に「各カメラ＝独立 rig、各 frame＝単一画像」をデフォルト構築し、単一カメラを複数画像で共有する旧フォーマットを受理しない。3.11.1 はこの仕様を持たない（rig 非ネイティブ）。
- **rig 非互換 SIGABRT**: 3.12.6 で `colmap point_triangulator` が `reconstruction.cc:166] Check failed: existing_frame.RigId()==frame.RigId()` を投げて abort する事象。root cause の全記録は `docs/issues/feat-010-dynerf/investigation.md`（イテレーション1〜2）。
- **vcpkg classic モード**: `vcpkg.json`（manifest）を使わず、`vcpkg install '<port>[<features>]'` でツリー HEAD の port をそのまま入れる方式。既存 3.12.6 はこの方式で導入済み。
- **vcpkg manifest モード + override**: `vcpkg.json` に `dependencies`・`overrides`（版固定）・`builtin-baseline`（versions DB の基準 commit）を記述し、`vcpkg install --x-manifest-root=<dir> --x-install-root=<dir>/installed` で**特定版を別 prefix にビルド**する方式。本案件で 3.11.1 を入れる手段（dry-run で 3.11.1 解決を実証済み、§FR-001）。
- **別 install prefix（別 install-root）**: 既存 3.12.6 の `installed`（`/data/sakagawa/opt/vcpkg/installed`）とは別の、3.11.1 専用の installed ディレクトリ（`/data/sakagawa/opt/colmap-3.11/installed`）。既存 3.12.6 に一切触れずに 3.11.1 を併存させるための分離。
- **3.11.1 ラッパー**: colmap.sh 内の bare `colmap` 呼び出しが 3.11.1 を解決するための実行可能スクリプト。専用 bin ディレクトリに **`colmap`** という名前で置き、実行時に PATH 先頭へ前置する（`LD_LIBRARY_PATH` に 3.11.1 の `installed/x64-linux/lib` を前置して実体を exec）。既存 `~/.local/bin/colmap`（3.12.6）はデフォルトのまま変更しない。
- **colmap.sh llff**: `scripts/llff2colmap.py`（`poses_bounds.npy`→COLMAP テキスト変換、各カメラ先頭フレームを `image_colmap/r_XXX.png` にコピー）→ COLMAP（feature_extractor〜stereo_fusion）を一括実行するシェルスクリプト。**各カメラの先頭フレーム `0000.png` のみ**（=N枚、cut_roasted_beef は20枚）で SfM+MVS を行い `fused.ply` を生成する。スクリプト内部は bare `python`（`colmap.sh:8`・`:16`）と bare `colmap` を呼ぶ。
- **bare python / bare colmap の PATH 解決**: 本環境に bare `python` は無く（`python3` のみ）、`colmap` は既定で 3.12.6。colmap.sh を**非改変**で 3.11.1 実走するため、実行時に `PATH="<3.11.1 bin>:/data/sakagawa/4DGaussians/.venv/bin:$PATH"` を前置し、bare `python`→`.venv/bin/python`、bare `colmap`→3.11.1 に解決する。
- **検証データ cut_roasted_beef**: 旧 feat-010 で取得・抽出済みの DyNeRF シーン（`data/dynerf/cut_roasted_beef/`、20カメラ mp4・`poses_bounds.npy`・各 `camXX/images/0000.png〜0299.png`）。本案件で**再取得・再抽出は不要**（実機確認済み）。

## 1.3 機能要求一覧

### FR-001: COLMAP 3.11.1 の vcpkg ソースビルド（別 install prefix、既存 3.12.6 温存）

- **概要**: vcpkg の manifest モード + override で COLMAP **3.11.1（port-version 4）** を CUDA 有効・GUI 除外でソースビルドし、既存 3.12.6 とは別の install prefix（`/data/sakagawa/opt/colmap-3.11/installed`）に配置する。既存 vcpkg ツリー（`/data/sakagawa/opt/vcpkg`、HEAD `36393d1`）の vcpkg バイナリ・downloads・buildtrees を再利用し、既存 classic installed（3.12.6）には一切触れない。
- **入力**:
  - manifest `vcpkg.json`（`/data/sakagawa/opt/colmap-3.11/vcpkg.json`）: `dependencies=[{name:colmap, default-features:false, features:[cuda]}]`、`overrides=[{name:colmap, version:"3.11.1", port-version:4}]`、`builtin-baseline:"37c4e62c5ed20ac4cb09884917bde2cbbccf7aa3"`（2025-11-04、colmap 3.11.1 が baseline・eigen 3.4.1。当初は現 HEAD `36393d1` を指定したが eigen 5.0.1 が 3.11.1 とビルド非互換のため旧 commit に変更。investigation イテレーション1）。
  - ビルドコマンド: `/data/sakagawa/opt/vcpkg/vcpkg install --x-manifest-root=/data/sakagawa/opt/colmap-3.11 --x-install-root=/data/sakagawa/opt/colmap-3.11/installed --triplet x64-linux`（ビルドシェルで CUDA 11.6 を `PATH=/usr/local/cuda-11.6/bin:$PATH CUDA_HOME=/usr/local/cuda-11.6 CUDACXX=/usr/local/cuda-11.6/bin/nvcc` に設定。feat-008 と同方針）。
- **出力**: `/data/sakagawa/opt/colmap-3.11/installed/x64-linux/tools/colmap/colmap`（3.11.1 実体バイナリ）と `installed/x64-linux/lib/*.so`（同梱依存）。
- **受け入れ基準**:
  - `vcpkg install` が終了コード 0 で完走し、`/data/sakagawa/opt/colmap-3.11/installed/x64-linux/tools/colmap/colmap` が生成される。
  - `LD_LIBRARY_PATH=/data/sakagawa/opt/colmap-3.11/installed/x64-linux/lib /data/sakagawa/opt/colmap-3.11/installed/x64-linux/tools/colmap/colmap -h` が終了コード 0 で、出力1行目が **`COLMAP 3.11.1`** を含む（版が 3.11.1 であることを実数確認）。
  - 既存 3.12.6 が**無傷**であること: `~/.local/bin/colmap` の中身が feat-008 のまま不変、`command -v colmap` が `~/.local/bin/colmap` を指し、`colmap -h` の版表示が **`COLMAP 3.12.6`** のまま。`/data/sakagawa/opt/vcpkg/installed/x64-linux/tools/colmap/colmap` が存在し続ける。
  - ビルド再現情報（vcpkg commit・採用 CUDA 版・採用フィーチャ `core,cuda`・所要時間・3.11.1 installed のディスク使用量）を scratch の `build-info.txt` に記録する。

### FR-002: 3.11.1 PATH ラッパーの作成と併設（デフォルト 3.12.6 を壊さない）

- **概要**: colmap.sh 内の bare `colmap` が 3.11.1 を解決できるよう、専用 bin ディレクトリに **`colmap`** という名前のラッパーを作成する。既存 `~/.local/bin/colmap`（3.12.6）はデフォルトのまま変更しない。
- **入力**: ラッパー `/data/sakagawa/opt/colmap-3.11/bin/colmap` を以下の内容で作成し `chmod +x`。
  ```bash
  #!/usr/bin/env bash
  # COLMAP 3.11.1 (vcpkg manifest build) launcher -- feat-011
  ROOT=/data/sakagawa/opt/colmap-3.11/installed/x64-linux
  # LD_LIBRARY_PATH が空のとき末尾 ":" でカレントディレクトリが探索対象に入るのを防ぐ（空値分岐）
  if [ -n "${LD_LIBRARY_PATH:-}" ]; then
    export LD_LIBRARY_PATH="$ROOT/lib:$LD_LIBRARY_PATH"
  else
    export LD_LIBRARY_PATH="$ROOT/lib"
  fi
  exec "$ROOT/tools/colmap/colmap" "$@"
  ```
- **出力**: 実行可能な 3.11.1 ラッパー（PATH 前置で `colmap` 名として解決可能）。
- **受け入れ基準**:
  - `/data/sakagawa/opt/colmap-3.11/bin/colmap -h` が終了コード 0、版表示が **`COLMAP 3.11.1`**。
  - PATH 前置で `colmap` 名が 3.11.1 を指すこと: `PATH="/data/sakagawa/opt/colmap-3.11/bin:$PATH" bash -c 'command -v colmap'` が当該ラッパーを返し、`PATH="/data/sakagawa/opt/colmap-3.11/bin:$PATH" colmap -h` の版が **3.11.1**。
  - PATH 前置を**外した**状態では `command -v colmap` が `~/.local/bin/colmap`（3.12.6）を返す（デフォルトが 3.12.6 のまま＝既存ワークフロー非破壊）。

### FR-003: cut_roasted_beef で colmap.sh ... llff を 3.11.1 実走（4DGS 本体完全非改変、rig 非互換解消の実証）

- **概要**: 旧 feat-010 の検証データ cut_roasted_beef を用い、`colmap.sh ... llff` を **3.11.1** で実走し、3.12.6 で SIGABRT していた `point_triangulator` が rig エラーなく完走して `fused.ply`（点数>0）が生成されることを実証する。**4DGS 本体（colmap.sh を含む）は1行も改変しない**（GPU 選択は colmap.sh:5 のハードコード=GPU0 に従う。`colmap.sh:5` の引数化は feat-012 のスコープ）。
- **入力**:
  - 前提: cut_roasted_beef のフレーム抽出済み（実機確認済み: 20カメラ・各 `camXX/images/0000.png` 存在）。万一欠落時のみ `.venv/bin/python scripts/preprocess_dynerf.py --datadir data/dynerf/cut_roasted_beef` を再実行。
  - 実行（cwd=リポジトリルート `/data/sakagawa/4DGaussians`。`PATH` 先頭へ 3.11.1 bin と `.venv/bin` を前置。**`bash -e`** で最初の COLMAP コマンド失敗時に即停止させ、`sparse/0` 空による後段の連鎖失敗を排除する＝colmap.sh 自体は非改変で実行方法のみ変更）:
    ```bash
    PATH="/data/sakagawa/opt/colmap-3.11/bin:/data/sakagawa/4DGaussians/.venv/bin:$PATH" \
      bash -e colmap.sh data/dynerf/cut_roasted_beef llff
    ```
- **出力**: `data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply`（および `colmap/sparse/0/`・`colmap/database.db`・`image_colmap/`・`sparse_/`）。
- **受け入れ基準**:
  - 実行前確認: `PATH="/data/sakagawa/opt/colmap-3.11/bin:/data/sakagawa/4DGaussians/.venv/bin:$PATH" command -v colmap` が 3.11.1 ラッパーを、同 PATH で `command -v python` が `.venv/bin/python` を返す（bare コマンドの即死を未然に防ぐ）。
  - `colmap point_triangulator`（`colmap.sh:20`）が **`Check failed: existing_frame.RigId()==frame.RigId()` の SIGABRT を出さず**完走する（rig 非互換が解消されたことの直接の合格条件）。
  - 全 COLMAP コマンド（feature_extractor〜stereo_fusion）が完走し、`colmap/dense/workspace/fused.ply` が生成される。
  - `fused.ply` が `PlyData.read` で読めて**点数 > 0**（空・破損は不合格）。
  - **image↔database 対応の確認（偽陽性の排除）**: `point_triangulator --clear_points 1`（`colmap.sh:20`）は `TranscribeImageIdsToDatabase`（`reconstruction.cc:471`）で各 image を **filename で database から引き**、reconstruction の image_id を database の id に書き換える。よって `sparse_custom/images.txt` の image_id（`llff2colmap.py` が name 順に付与）と `database.db` の image_id（`feature_extractor` が並列登録順に付与）が**数値として食い違うのは正常**で、対応は filename で保たれる。合否は次で判定する: ① 20画像すべて registered（transcribe が `FATAL_THROW` なく完了）② `colmap model_analyzer --path colmap/sparse/0`（3.11.1）の **Mean reprojection error < 1.5px**（実測 0.852px。食い違えば数px〜数十pxに跳ねる）③ 三角化点数 > 0（実測 sparse 4,566点 / fused 387,496点）。**異常（reprojection error 過大、`Image with name ... does not exist in database` の FATAL_THROW、registered < 20）の場合のみ不合格**とし investigation に記録・協議する。〔旧基準「image_id↔name が数値完全一致（db==sc）」は filename transcribe を見落とした**誤り**だった。feat-010 の 3.12.6 クラッシュは image_id 不整合そのものでなく rig/frame 整合チェックが原因。検証手順は design §1.4.3、経緯は investigation イテレーション2 参照〕
  - `git status --porcelain` で 4DGS 本体（`colmap.sh`・`scripts/`・`scene/` 等の追跡ファイル）に差分が無い（`data/` 配下の生成物は .gitignore 管理外運用のため対象外）。本案件で 4DGS 本体改変ゼロを確認する。

### FR-004: 点群ダウンサンプル（downsample_point.py）

- **概要**: `downsample_point.py` で `fused.ply` を ≤40,000 点に voxel ダウンサンプルし、`points3D_downsample2.ply` を生成する（DyNeRF 学習で必須の点群形式まで一貫して通ることの確認）。
- **入力**:
  ```bash
  .venv/bin/python scripts/downsample_point.py \
    data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply \
    data/dynerf/cut_roasted_beef/points3D_downsample2.ply
  ```
- **出力**: `data/dynerf/cut_roasted_beef/points3D_downsample2.ply`。
- **受け入れ基準**:
  - 終了コード 0 で完走し、`points3D_downsample2.ply` が生成される。
  - `PlyData.read` で読めて **0 < 点数 ≤ 40,000**。

### FR-005: 文書化（導入手順・運用・依存記録）

- **概要**: COLMAP 3.11.1 併設の導入手順・運用方法（3.11.1 を使う実行コマンド形）を再現可能な形で文書化し、技術記録・ロードマップを更新する。
- **入力**: FR-001〜004 の実施結果。
- **出力**:
  - `design.md`: §1.4 にビルド・ラッパー・検証手順の正本（`/clear` 後もこれだけで再現可能）。
  - `docs/TECH_STACK.md`: 「COLMAP」節に **3.11.1 併設**のサブ節を追加（vcpkg manifest+override・別 prefix・採用 CUDA 11.6・ラッパー方式・3.12.6 との使い分け・選定理由）。
  - `CLAUDE.md`: 実行環境表の COLMAP 行と「マルチGPU運用ルール」/「データセット」周辺に、**DyNeRF/multipleview 前処理（feat-012/013）では 3.11.1 を PATH 前置で使う**旨と実行コマンド形を追記。
  - `docs/BACKLOG.md`: feat-011 を Closed に更新（手動テスト合格後）。
- **受け入れ基準**:
  - `/clear` 後でも本書＋design.md のみで 3.11.1 の導入（vcpkg.json 内容・ビルドコマンド・CUDA 設定・ラッパー作成）と検証（colmap.sh llff の実行コマンド形）を再現できる。
  - `docs/TECH_STACK.md` に 3.11.1 併設の用途・選定理由・採用版（3.11.1#4）・配置パスが記録される。
  - 本案件は **Python 依存の追加・変更が無い**（COLMAP は外部バイナリ）。よって `requirements.lock.txt` の再生成は不要（依存変化なし）。

## 1.4 非機能要求

- **対応環境**: Ubuntu / A100（sm_80）/ ヘッドレス。ビルド用 CUDA Toolkit **11.6**（`/usr/local/cuda-11.6`、第一候補。driver 565.57.01 完全対応・feat-002/008 実証済み）、代替 12.8。dense（`patch_match_stereo`）は CUDA 有効ビルドによりヘッドレスで GPU 動作（feat-008 で 3.12.6 実証済み。3.11.1 も同 CUDA バックエンド・実証済み）。なお `colmap.sh` の `feature_extractor` は `--SiftExtraction.estimate_affine_shape 1 --domain_size_pooling 1` 指定により **CPU SIFT に自動フォールバック**する（これらは COLMAP の GPU SIFT 非対応オプション。3.11.1/3.12.6 共通仕様・正常。llff 経路は元から CPU SIFT）。
- **権限**: sudo 不要（ユーザー権限で完結）。ビルド前提の gfortran は feat-008 で `sudo apt` 導入済み（再導入不要）。
- **信頼性（既存環境の非破壊が最重要）**:
  - 既存 COLMAP 3.12.6（feat-008）・既存 vcpkg classic installed・`~/.local/bin/colmap`・`.venv`（torch/numpy 等）を**一切破壊しない**。3.11.1 は別 install prefix・別ラッパーで完全分離する。
  - HyperNeRF（feat-009）は事前生成点群で COLMAP 非使用のため、本案件の影響を受けない。
- **処理時間（目安・合否非依存）**:
  - ビルド（FR-001）: 数時間目安（依存の多くは既存 vcpkg ツリーの downloads/buildtrees にキャッシュ済みのため短縮余地あり）。バックグラウンド実行・ログ監視。
  - colmap.sh llff（FR-003）: 各カメラ先頭1フレーム（20枚）の SfM+MVS のため数分〜十数分目安。
- **マルチGPU**: 本案件は 4DGS 本体非改変のため、検証（colmap.sh llff）は **colmap.sh:5 のハードコード（GPU0）に従う**。GPU0 が使用中の場合は**空くのを待つ**（検証は20枚の SfM+MVS で数分〜十数分と軽量なため、GPU0 が空くタイミングを取れる）。**colmap.sh は一時的にも改変しない**（一時改変＋`git checkout` での復元はユーザーの未コミット変更を破棄する恐れがあり採らない）。任意 GPU 選択を可能にする `colmap.sh:5` の引数化は feat-012 のスコープ（ユーザー決定 2026-06-23）。
- **ディスク**: 3.11.1 の installed（数 GB）＋ buildtrees（既存ツリーに追加、十数 GB に達し得る）。`/data`（15TB 空き）に配置。

## 1.5 制約条件

- **使用必須ツール・ライブラリ**（すべて導入済み・新規 Python 依存なし）:
  - vcpkg（`/data/sakagawa/opt/vcpkg`、feat-008 導入済み。HEAD `36393d1`）。
  - CUDA Toolkit 11.6（`/usr/local/cuda-11.6`、ビルド用）。gcc/g++ 11.4・gfortran 11.4（feat-008 導入済み）。
  - 既存 `.venv`（`downsample_point.py` が使う open3d 0.19.0、`plyfile`。feat-008/009 導入済み）。
- **使用禁止 / 回避**:
  - **4DGaussians 本体コード（`colmap.sh`・`scripts/`・`scene/` 等の追跡ファイル）を1行も改変しない**。`colmap.sh:5` の GPU 引数化は **feat-012** で行う（本案件のスコープ外）。
  - 既存 COLMAP **3.12.6 を削除・置換・上書きしない**（温存）。デフォルトの `~/.local/bin/colmap` は 3.12.6 のまま変更しない。
  - **`uv sync` / `uv pip sync` は使わない**（本案件は Python 依存追加が無いため、そもそも `uv pip install` も発生しない見込み）。`pyproject.toml` は作らない。
- **ネットワーク**: vcpkg が 3.11.1 とその依存のソース tarball を取得する（GitHub 等）。既存 downloads キャッシュにある分は再DLしない。
- **既存 colmap.sh 中間生成物**: cut_roasted_beef の `colmap/`・`sparse_`・`image_colmap`（旧 feat-010 検証の手動生成物・`fused.ply` 含む）は残存するが、`colmap.sh` 冒頭が `rm -rf` して作り直すため FR-003 実行で上書きされる（事前手動削除は不要）。

## 1.6 優先順位（MoSCoW）

| 要求 | 優先度 | 備考 |
|------|--------|------|
| FR-001 3.11.1 vcpkg ソースビルド（別 prefix・3.12.6 温存） | **Must** | 本案件の中核。これが無いと後続が成立しない |
| FR-002 3.11.1 PATH ラッパー作成・併設 | **Must** | colmap.sh を非改変で 3.11.1 実走するために必須 |
| FR-003 colmap.sh ... llff を 3.11.1 実走（rig 非互換解消の実証） | **Must** | 判定基準の核心。rig エラー解消＋fused.ply 生成 |
| FR-004 ダウンサンプル | **Must** | DyNeRF 学習点群形式まで一貫して通る確認 |
| FR-005 文書化 | **Must** | 再現性・運用・feat-012/013 への引き継ぎ |

- **MVP の範囲**: FR-001〜FR-005 すべて。これらの達成をもって BACKLOG の判定基準（3.11.1 別 prefix ビルド＋3.12.6 温存／colmap.sh llff 完走で fused.ply 点数>0／downsample で points3D_downsample2.ply ≤40k点）を満たす。
- **スコープ外**: 学習（train.py）・レンダ・評価は本案件で行わない（feat-012 DyNeRF のスコープ）。`colmap.sh:5` の GPU 引数化も feat-012。multipleview の `multipleviewprogress.sh` 実走は feat-013。
