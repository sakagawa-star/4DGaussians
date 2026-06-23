# 技術スタック定義書

最終更新: 2026-05-21

> 本書は4DGaussians環境構築の技術スタックを定義する。確定していない項目は「未検証」と明記し、環境構築フェーズで決定・更新する。

---

## プロジェクト基盤

| 項目 | 値 | 根拠 |
|------|-----|------|
| 言語 | Python 3.10（uv managed Python） | 公式は3.7だが**uvは3.7を提供しない（managed Pythonの下限は3.8.20）**。torch 1.13.1のwheelはcp38〜cp311に存在し、3.10ならビルド済みwheelを利用可 |
| パッケージ管理 | uv 0.11.6（condaは使わない） | 公式のcondaは**Pythonインタープリタ生成のみ**に使われ、依存は全てpip。`uv venv` + `uv pip` で等価に置換できることを確認済み（2026-05-20調査）。本マシンにcondaは無い |
| 対象OS | Ubuntu Linux | 開発環境 |
| GPU | NVIDIA A100-SXM4-40GB × 7（Driver 565.57.01） | 4DGSの学習・レンダリングに使用（単一GPUで動作。マルチGPUは非要件） |
| CUDA Toolkit（ビルド用） | **11.6**（`/usr/local/cuda-11.6`） | torch 1.13.1+cu116 とメジャー・マイナーまで一致するCUDAでCUDA拡張をビルドする。新規インストール不要（既存） |
| ホストコンパイラ | gcc/g++ 11.4.0 | CUDA 11.6/11.7/11.8 は gcc ≤ 11 対応 → 適合（ダウングレード不要） |

### サーバー上のCUDA一覧（2026-05-20調査）

| パス | nvcc版 | 種別・別名 |
|------|--------|-----------|
| `/usr/local/cuda-11.6` | 11.6.124 | システム。`/usr/local/cuda` が update-alternatives 経由で現在ここに解決。**4DGSのビルドに使用** |
| `/usr/local/cuda-11.7` | 11.7.99 | システム。cu117採用時の代替 |
| `/usr/local/cuda-11.8` | 11.8.89 | システム（`/usr/local/cuda-11` → **11.8**） |
| `/usr/local/cuda-12.3` | 12.3.107 | システム（`/usr/local/cuda-12` → ここ） |
| `~/cuda/cuda-12.8` | 12.8.93 | ユーザー導入。`~/.bashrc` でグローバルにアクティブ（このマシンの別環境が使用。4DGSでは使わない） |

補助情報: ドライバ 565.57.01（CUDA 12.7まで対応、cu116ランタイムに後方互換）／ A100 = sm_80（CUDA 11.6で完全サポート）。

> **注意（リンク解決）**: `/usr/local/cuda` は二段階リンク（`/usr/local/cuda` → `/etc/alternatives/cuda` → `/usr/local/cuda-11.6`）。管理者が `update-alternatives` を切り替えると指す先が変わりうるため、**ビルドでは曖昧さ回避のため絶対パス `/usr/local/cuda-11.6` を明示**する。なお `/usr/local/cuda-11` は 11.6 ではなく **11.8** に解決される点に注意。

### CUDA環境変数の運用方針

`~/.bashrc` でグローバルに設定されている `CUDA_HOME=~/cuda/current`(=**12.8**) は **このマシンの別環境が依存しているため変更しない**。

```
# グローバル（変更しない。このマシンの別環境が使用）
CUDA_HOME=/home/sakagawa/cuda/current        # = 12.8
PATH=/home/sakagawa/cuda/current/bin:$PATH
LD_LIBRARY_PATH=/home/sakagawa/cuda/current/lib64:$LD_LIBRARY_PATH
```

4DGSの**CUDA拡張ビルド時のみ** `CUDA_HOME` を 11.6 に上書きする（下記「uvでの等価手順」参照）。実行時（学習・レンダリング）はtorch wheelが自前のCUDAランタイムを同梱するため `CUDA_HOME` の影響を受けない。

> **監視点**: グローバルの `LD_LIBRARY_PATH` が12.8を指すため、実行時に稀に干渉の可能性。問題が出れば4DGS環境でのみ11.6へ上書き／除外で対処（通常はtorch同梱libが優先され問題なし）。

---

## コア依存関係（公式 `requirements.txt`）

> 本表は主要ライブラリの**用途・選定理由・バージョン方針**（なぜ）を示すキュレーション記録。実際に導入された版の正本は `requirements.lock.txt`（後述「uv 依存管理ルール」参照）であり、本表はその全量ではない。

| ライブラリ名 | バージョン要件 | 用途 | 備考 |
|-------------|--------------|------|------|
| torch | == 1.13.1（CUDA版サフィックスなし） | テンソル演算・GPU学習 | requirements.txt は `torch==1.13.1`（バージョンは固定だが `+cu116` 等のCUDAサフィックスが付かない）。**cu116版を `--index-url https://download.pytorch.org/whl/cu116` で明示導入**する。CUDA拡張は `/usr/local/cuda-11.6` でビルド（メジャー・マイナー一致）。cu117 + cuda-11.7 でも可 |
| torchvision | == 0.14.1 | 画像変換 | torch 1.13.1 に対応 |
| torchaudio | == 0.13.1 | （依存解決用） | torch 1.13.1 に対応 |
| mmcv | == 1.6.0 | 設定ファイル（`arguments/*.py`）の読み込み（`mmcv.Config.fromfile`） | `train.py`/`render.py`/`export_perframe_3DGS.py`/`merge_many_4dgs.py` が `--configs` 指定時に使用。純Pythonの `mmcv`（`mmcv-full`ではない）でCUDA opsビルド不要。ただし**cp310 wheelが無くsdistビルドが必要**で、`setup.py` が `pkg_resources` に依存する。**feat-001で `setuptools<81`（pkg_resources同梱版）導入＋`--no-build-isolation` でビルド成功を確認** |
| lpips | — | 知覚的画質評価（学習・評価） | |
| plyfile | — | 点群PLY入出力 | |
| pytorch_msssim | — | MS-SSIM損失・評価 | |
| open3d | == 0.19.0（feat-001 最新解決） | 点群処理・可視化（`scripts/downsample_point.py`） | feat-008（2026-06-21）で**既存 0.19.0 を維持**（追加導入せず）。numpy 1.23.5 と ABI 互換 |
| scikit-image | == 0.22.0（feat-008で追加） | 実シーン前処理 LLFF（`multipleviewprogress.sh` の `imgs2poses`）が依存 | **feat-008（2026-06-21）で新規導入**。`uv pip install "numpy==1.23.5" "scikit-image==0.22.0"`（追加的・numpy アンカー）。numpy 1.23.5 維持で torch/open3d 非破壊を確認（追加: lazy-loader/networkx/tifffile） |
| gdown | == 6.1.0（feat-009で追加） | Google Drive からのデータ取得（HyperNeRF 事前生成COLMAP点群 `points3D_downsample2.ply` のDL）。大容量ファイルの確認トークン（virus scan 警告）を自動処理 | **feat-009（2026-06-21）で新規導入**。`uv pip install gdown`（追加的）。numpy 1.23.5・torch 非破壊を確認（追加: beautifulsoup4/soupsieve/pysocks/filelock） |
| imageio[ffmpeg] | — | 動画入出力 | |
| matplotlib | — | 可視化 | |
| argparse | （導入しない） | 引数解析（標準ライブラリと重複） | PyPI版（最終リリース2010年）は標準ライブラリをシャドーイングする恐れがあるため**feat-001で導入対象から除外**。Python 3.10標準ライブラリを使用（design ADR-6） |
| numpy | == 1.23.5（feat-001で固定） | 数値計算（torch/open3d等の基盤） | requirements.txt 無指定だが、torch依存解決で numpy 2.x が入ると torch 1.13.1（numpy 1.x ABI）の `torch.from_numpy` 等が `RuntimeError: Numpy is not available` で不能。**1.23.5 に明示固定**。open3d 0.19 / opencv 4.13 / scipy 1.15 等は 1.23.5 でもABI互換（feat-001検証済み） |
| setuptools | < 81（feat-001で 80.10.2） | mmcv 1.6.0 の sdist ビルド用ツール | mmcv 1.6.0 が `pkg_resources` に依存。setuptools 81+ は pkg_resources を削除済みのため **<81 が必須**。`--no-build-isolation` でビルド時に使用 |

> **バージョン未固定の注意（feat-001で実地確認済み・2026-05-21）**: lpips / plyfile / pytorch_msssim / open3d / imageio / matplotlib は requirements.txt でバージョン無指定（最新解決）。最新解決で open3d 0.19.0 / imageio 2.37.3 / scipy 1.15.3 / opencv-python 4.13.0.92 等が入るが、**numpy のみ 1.23.5 に固定**すれば torch 連携・各依存とも整合する（懸念した依存群のABI破損・ダウングレードは不要だった）。解決版の正本は `requirements.lock.txt`。

## CUDA拡張サブモジュール（要ソースビルド）

| サブモジュール | URL | 用途 | 状態 |
|---------------|-----|------|------|
| depth-diff-gaussian-rasterization | https://github.com/ingra14m/depth-diff-gaussian-rasterization | 深度対応の微分可能ガウシアンラスタライザ | **ビルド済み（feat-002, 2026-05-21）**。glm（ネストsubmodule）含め取得し editable ビルド成功・import確認 |
| simple-knn | https://gitlab.inria.fr/bkerbl/simple-knn.git | 近傍探索（点群初期化） | **ビルド済み（feat-002）**。uv editable では `simple_knn/__init__.py`（`import torch` 記載）の追加が必要（investigation.md Iteration 1） |

> ビルド時は **`CUDA_HOME=/usr/local/cuda-11.6 PATH=/usr/local/cuda-11.6/bin:$PATH` をインライン上書き**し、**`--no-build-isolation`** を付けて `uv pip install --python .venv/bin/python -e` する（グローバルの12.8のままだと、PyTorch `cpp_extension` のCUDAメジャー版チェックで 11(torch) vs 12(nvcc) 不一致となりエラー／重大警告になる）。インライン上書きはコマンド限りでグローバル設定（12.8）を汚さない。feat-002（2026-05-21）で実地検証済み。手順・ハマりどころは `docs/issues/feat-002-cuda-ext-build/`（design.md・investigation.md）。

## ビューア（任意）

| 対象 | URL | 用途 | 備考 |
|------|-----|------|------|
| SIBR_viewers | https://gitlab.inria.fr/sibr/sibr_core | 学習結果のインタラクティブ可視化 | 環境構築の必須要件ではない。詳細は `docs/viewer_usage.md` |

## 外部ツール

| ツール | 状態 | 用途 |
|--------|------|------|
| COLMAP | **vcpkg ソースビルド導入済み（feat-008, 2026-06-21、3.12.6）＋ 3.11.1 別 prefix 併設（feat-011, 2026-06-23、rig 非互換回避）** | 実シーン（HyperNeRF/DyNeRF/multipleview）のSfM/MVS前処理。D-NeRF合成シーンでは不要。DyNeRF/multipleview の `colmap.sh`/`multipleviewprogress.sh` 実走時のみ 3.11.1 を使う |

### COLMAP（feat-008, 2026-06-21）

- **導入方式**: vcpkg（Microsoft 製ソースパッケージマネージャ）で依存込みソースビルド。**COLMAP 公式は Linux バイナリ（AppImage 含む）を一切配布していない**ため（GitHub Releases 3.3〜4.0.4 全確認）、conda 非依存・sudo 原則不要のソースビルドを採用。
- **版・フィーチャ**: `colmap[core,cuda]:x64-linux@3.12.6#1`。`core`=default feature（`gui`=Qt5 GUI）を除外（ヘッドレス・Qt5 回避）、`cuda`=CUDA 有効（GPU SIFT・dense）。**CGAL は省略**（4DG は mesher 不使用。必要時のみ `[core,cuda,cgal]` 再ビルド）。
- **ビルド CUDA**: **11.6**（`/usr/local/cuda-11.6`、第一候補）。driver 565.57.01 完全対応・torch cu116 実証済み。12.8 は代替（driver 565 では minor version compatibility 依存）。A100=sm_80。
- **Fortran 依存（重要）**: COLMAP は SuiteSparse(CHOLMOD)/Ceres 経由で BLAS/LAPACK 必須。vcpkg の Linux LAPACK 提供 `lapack-reference` が Reference LAPACK を Fortran ソースからビルドするため **gfortran が必須**。本機未導入だったため **`sudo apt-get install gfortran`（GNU Fortran 11.4.0、GPUサーバー管理者承認 2026-06-21）** で導入。これが無いとビルドが `Unable to find a Fortran compiler` で失敗する。
- **配置**: vcpkg=`/data/sakagawa/opt/vcpkg`（installed 約 3.7GB、十数 GB に達し得るため /data 配置）。バイナリ=`installed/x64-linux/tools/colmap/colmap`。**PATH ラッパー=`~/.local/bin/colmap`**（`LD_LIBRARY_PATH` に installed/lib を前置して実体を exec）。
- **ヘッドレス運用**: 本機は DISPLAY 未設定。**GPU SIFT（feature_extractor/exhaustive_matcher の `--use_gpu 1`）は CUDA 有効ビルドによりヘッドレスでも動作**（feat-008 で南棟データ18枚を全登録・実証）。失敗時は `--use_gpu 0`（CPU）にフォールバック（feature/matcher は独立判定）。dense（patch_match_stereo）は CUDA 直で OpenGL 不要。GPU 選択は `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N`（CLAUDE.md マルチGPU運用）。
- **再現情報**: ビルドスクリプト・ログ・`build-info.txt` は scratch（`/data/sakagawa/tmp/feat008-colmap/`、非コミット）。vcpkg commit `36393d1`。手順の正本は `docs/issues/feat-008-colmap/design.md`。

### COLMAP 3.11.1 併設（feat-011, 2026-06-23、rig 非互換回避）

- **背景**: COLMAP **3.12 系**は SfM に rig/frame アーキテクチャ（`scene/frame.h`・`scene/rig.h`）を導入し、`reconstruction.cc:166` に `THROW_CHECK(existing_frame.RigId() == frame.RigId())` を持つ。4DGS の `colmap.sh ... llff`（単一カメラ共有 sparse + `point_triangulator`）は DyNeRF/multipleview で `Check failed: existing_frame.RigId()==frame.RigId()` クラッシュする（feat-010 で 3.12.6 が中止に至った原因）。**3.11.1 はこの rig アーキ導入前**で当該アサーションを持たないため、4DGS 本体非改変のまま完走する。
- **導入方式**: vcpkg manifest + override で **`colmap[core,cuda]:x64-linux@3.11.1#4`** を CUDA 有効・GUI 除外でソースビルド。**別 install-root** `/data/sakagawa/opt/colmap-3.11/installed`（既存 3.12.6 の `/data/sakagawa/opt/vcpkg/installed` と完全分離）。vcpkg バイナリ・downloads・buildtrees は既存ツリーを共有。
- **builtin-baseline = `37c4e62c5ed20ac4cb09884917bde2cbbccf7aa3`**（2025-11-04、colmap 3.11.1 が baseline）。現 HEAD `36393d1` の eigen 5.0.1 は COLMAP 3.11.1 とビルド非互換（`covariance.cc` の `DenseBase::nonZeros()` 削除）のため、3.11.1 が整合する旧 commit（eigen 3.4.1#1・ceres 2.2.0#5）に固定。
- **採用 CUDA**: 11.6（feat-008 と同方針、torch cu116 整合）。
- **ラッパー方式**: `/data/sakagawa/opt/colmap-3.11/bin/colmap`（名前は `colmap`、`LD_LIBRARY_PATH` 空値分岐で `installed/x64-linux/lib` を前置して実体を exec）。**3.11.1 を使うときだけ PATH 先頭に前置**する。既定 `~/.local/bin/colmap`（3.12.6）は不変。
- **3.12.6 との使い分け**: 既定（`convert.py` の mapper 経路、静的シーン等）は **3.12.6**（`~/.local/bin/colmap`）。DyNeRF/multipleview の `colmap.sh ... llff` / `multipleviewprogress.sh`（既知ポーズ `point_triangulator` + 単一カメラ共有）実走時のみ **3.11.1** を PATH 前置: `PATH="/data/sakagawa/opt/colmap-3.11/bin:/data/sakagawa/4DGaussians/.venv/bin:$PATH" bash -e colmap.sh <dir> llff`。
- **CPU SIFT**: `colmap.sh:15` の feature_extractor は `--SiftExtraction.estimate_affine_shape 1 --domain_size_pooling 1` 指定により CPU SIFT に自動フォールバック（GPU SIFT 非対応オプション。3.11.1/3.12.6 共通仕様・正常。south-building の mapper 経路と異なり llff 経路は元から CPU SIFT）。dense（`patch_match_stereo`）は GPU を使う。
- **image_id**: `point_triangulator --clear_points 1` は `TranscribeImageIdsToDatabase`（`reconstruction.cc:471`）で filename ベースに image_id を database へ揃えるため、`sparse_custom` と `database.db` の image_id 数値不一致は正常（Mean reprojection error 0.852px で実証）。
- **動作実証**: cut_roasted_beef（20カメラ）で `colmap.sh llff` が rig クラッシュなく完走、`fused.ply` 387,496点、Mean reprojection error 0.852px、`points3D_downsample2.ply` 37,361点。git 本体差分ゼロ（非改変）。
- **再現情報**: manifest `/data/sakagawa/opt/colmap-3.11/vcpkg.json`、ログは scratch（`/data/sakagawa/tmp/feat011-colmap-3.11/`、非コミット）。手順の正本は `docs/issues/feat-011-colmap-3.11/design.md`。

---

## データセット

| データセット | 種別 | 取得元 | 前処理 |
|-------------|------|--------|--------|
| D-NeRF | 合成シーン | Dropbox（README参照） | 不要 |
| HyperNeRF | 実シーン（単眼） | HyperNeRF releases（`vrig_*.zip`） | COLMAP。**事前生成点群あり**（feat-009 で broom2 を採用） |
| Plenoptic / DyNeRF | 実シーン（多視点動画） | Neural 3D Video公式 | フレーム抽出 + COLMAP |

`data/` 配下に配置する（`.gitignore` 管理外を想定）。

### HyperNeRF（feat-009, 2026-06-21、broom2 で学習〜評価 動作確認済み）

- **データ取得**: HyperNeRF v0.1 リリース `https://github.com/google/hypernerf/releases/download/v0.1/vrig_broom.zip`（1.5GB）を取得・展開。zip のトップ階層が `broom2/` のため `data/hypernerf/virg/` へ展開すると `data/hypernerf/virg/broom2/`（`dataset.json`/`metadata.json`/`scene.json`/`camera/`/`rgb/2x/` 等、画像394枚）が直接できる。
- **点群**: 4DGS 作者の事前生成COLMAP点群を Google Drive（file id `1fUHiSgimVjVQZ2OOzTFtz02E9EqCoWr5`、README 記載）から **gdown** でDL（zip 37.9MB）。内部 `hypernerf/virg/broom2/points3D_downsample2.ply`（38,569点）を `data/hypernerf/virg/broom2/` へ配置。**COLMAP 実走は不要**（feat-008 で COLMAP は検証済み）。
- **学習〜評価**: `train.py`（config `arguments/hypernerf/broom2.py`、coarse3000+fine14000、約17分@A100×1）→ `render.py --skip_train` → `metrics.py`。**PSNR 22.08 / MS-SSIM 0.691**（論文 broom 22.0/0.70 とほぼ一致）。
- **ヘッドレス注意**: HyperNeRF 読み込み時に `scene/dataset_readers.py:plot_camera_orientations` が matplotlib で `output.png` を CWD に savefig するため、train/render は `MPLBACKEND=Agg` と書込可能な `MPLCONFIGDIR`/`TMPDIR` を固定する（metrics は不要）。詳細は `docs/issues/feat-009-hypernerf/`。

---

## セットアップ手順（公式README、conda前提）

> **注意**: 本マシンにcondaは無い。下記はオリジナル手順の記録であり、実際はuvで等価な環境を構築する（手順は環境構築フェーズの設計書で確定する）。

```bash
git clone https://github.com/hustvl/4DGaussians
cd 4DGaussians
git submodule update --init --recursive
conda create -n Gaussians4D python=3.7
conda activate Gaussians4D

pip install -r requirements.txt
pip install -e submodules/depth-diff-gaussian-rasterization
pip install -e submodules/simple-knn
```

公式の動作実績環境: `pytorch=1.13.1+cu116`

### uvでの等価手順（方針、2026-05-20調査）

condaの役割（Python 3.7環境の生成）をuvで置換する。**Python 3.7はuvで提供されないため3.10を使う**（torch 1.13.1のwheelはcp310にある）。**CUDA拡張のビルドは `/usr/local/cuda-11.6` を使う**（torch cu116とメジャー・マイナー一致）。

```bash
cd /data/sakagawa/4DGaussians
git submodule update --init --recursive          # サブモジュール取得（現状は未初期化）
uv venv --python 3.10                             # 仮想環境作成（managed Python 3.10）

# 依存導入（torchはcu116を明示）
uv pip install torch==1.13.1 torchvision==0.14.1 torchaudio==0.13.1 \
  --index-url https://download.pytorch.org/whl/cu116
uv pip install -r requirements.txt                # 残りの依存（torch指定が効くよう順序に注意）

# CUDA拡張ビルド：ビルド時のみCUDA_HOMEを11.6へ上書き（グローバルの12.8は変えない）
CUDA_HOME=/usr/local/cuda-11.6 PATH=/usr/local/cuda-11.6/bin:$PATH \
  uv pip install -e submodules/depth-diff-gaussian-rasterization
CUDA_HOME=/usr/local/cuda-11.6 PATH=/usr/local/cuda-11.6/bin:$PATH \
  uv pip install -e submodules/simple-knn
```

> 上記は方針スケッチ。確定コマンド（`requirements.txt` 内のtorch指定との競合回避、ビルド時の環境変数の永続化方法等）はfeat-001/feat-002の設計書で定める。

---

## uv 依存管理ルール（環境破壊の防止）

> **背景**: 別のuvプロジェクトで、`uv pip install` で命令的に構築した環境に対し、依存を完全宣言していない `pyproject.toml` に1パッケージだけ足して `uv sync` を実行した結果、未宣言の主要依存が「余分」として一括削除され、環境が破壊された事例があった。同型事故を防ぐため本プロジェクトでは以下を厳守する。

1. **`uv sync` / `uv pip sync` を使わない**。これらは「宣言（`pyproject.toml`+`uv.lock`、または requirements ファイル）に無いパッケージを削除」する剪定動作のため、ソースビルドのCUDA拡張・editable・特殊indexのtorchを巻き込んで壊す。パッケージ追加は常に **`uv pip install`**（追加的・非破壊）で行う。
2. **`pyproject.toml` を作らない**。無ければ `uv sync` は実行できず（エラー）、事故が構造的に起きない。torch の CUDA index は `pyproject.toml` ではなく `uv pip install ... --index-url https://download.pytorch.org/whl/cu116` で指定する。（注: 措置2は `uv sync` だけを塞ぐ。`uv pip sync` は `pyproject.toml` 無しでも動くため、措置1の禁止も併せて必要）
3. **中途半端な `dependencies` リストを作らない**。宣言管理を採るなら全依存（ソースビルド・editable 含む）を完全宣言する必要があり非現実的なため、本プロジェクトでは宣言管理を採用しない。
4. **構築直後に `uv pip freeze > requirements.lock.txt` でスナップショットを取得**し、git管理する。これが「実際に動いた厳密な全依存」の正本となり、万一壊れても復元できる（TECH_STACK 上の「バージョン未固定の注意」「解決版を記録」の具体的手段でもある）。

### 依存記録の役割分担

| ファイル | 役割 | 内容・正本性 |
|---------|------|-------------|
| `requirements.txt`（公式） | 上流の緩いスペック | 4DGaussians公式が定義。`torch==1.13.1`（CUDA無印）等。**版の正本ではない** |
| `docs/TECH_STACK.md`（本書） | 人間向けキュレーション記録 | 主要ライブラリの**用途・選定理由・バージョン方針**（なぜ）。全量ではない |
| `requirements.lock.txt`（我々が生成） | 機械的な厳密スナップショット | `uv pip freeze` の全出力（推移的依存含む）。**正確な導入版の正本**。復旧・再現に使う |

> 運用: ライブラリを追加・変更・削除したら、(1) `uv pip install`/`uv pip uninstall` で操作し、(2) `docs/TECH_STACK.md` の用途・方針を更新し、(3) `uv pip freeze > requirements.lock.txt` を再生成する（CLAUDE.md「ドキュメント作成ルール」と一致）。

---

## 制約・未検証事項

### 技術上の必須条件

| 制約 | 内容 | 根拠 |
|------|------|------|
| NVIDIA GPU 必須 | 4DGSの学習・レンダリングに必要 | CUDA拡張がGPU依存 |
| CUDA拡張のビルド必須 | depth-diff-gaussian-rasterization, simple-knn のソースビルド | プリビルドwheelが提供されない |
| CUDA_HOME設定必須 | nvccがビルド時に必要 | CUDA拡張のコンパイル |

### 決定済み（2026-05-20）

- **環境管理はconda不使用・uvを使う**: 公式condaはPython生成のみで依存は全てpip。`uv venv` + `uv pip` で等価に置換可能
- **Pythonは3.10**: uvが3.7を提供しない（下限3.8.20）。torch 1.13.1はcp310 wheelあり、3.10で導入可能
- **torchは1.13.1+cu116、CUDA拡張のビルドは `/usr/local/cuda-11.6`**: 公式実績版に一致し、必要なCUDA(11.6)が既存。新規CUDAインストール不要、gcc 11.4も適合。「12.8 vs 11」のメジャー版不一致は**ビルド時のみ`CUDA_HOME`を11.6へ上書き**して回避（グローバルの12.8はこのマシンの別環境用に維持）

### 未検証事項（環境構築フェーズで実地検証・記録する）

- 上記方針でCUDA拡張（depth-diff-gaussian-rasterization, simple-knn）が実際にビルド・import成功するか → **feat-002で検証済み（2026-05-21）**: 両者とも editable ビルド成功・import確認。simple-knn は uv editable 向けに `simple_knn/__init__.py`（`import torch`）の追加が必要だった（investigation.md Iteration 1）
- mmcv 1.6.0 のインストール可否とバージョン整合（cu116 torch環境下） → **feat-001で検証済み（2026-05-21）**: `setuptools<81`（pkg_resources同梱）導入＋`--no-build-isolation` でビルド・import 成功
- `requirements.txt` のtorch指定（無印`torch==1.13.1`）と、cu116 index-url 明示インストールの競合回避方法 → **feat-001で検証済み**: torch系を cu116 index で先行導入し、`requirements.txt` から torch系3行と argparse を grep 除外して残りを導入する2段階方式で競合なし（`+cu116` 維持を確認）

### 非要件（当面の対象外）

- 実シーン（HyperNeRF/DyNeRF）の学習（COLMAP導入が前提。まずD-NeRF合成シーンで動作確認）
- SIBR_viewersによる可視化
- マルチGPU分散学習
