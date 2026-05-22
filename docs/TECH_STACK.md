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
| open3d | — | 点群処理・可視化 | |
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
| COLMAP | **未インストール** | 実シーン（HyperNeRF/DyNeRF/カスタム）のSfM前処理。D-NeRF合成シーンでは不要 |

---

## データセット

| データセット | 種別 | 取得元 | 前処理 |
|-------------|------|--------|--------|
| D-NeRF | 合成シーン | Dropbox（README参照） | 不要 |
| HyperNeRF | 実シーン | HyperNeRF releases | COLMAP（事前生成点群あり） |
| Plenoptic / DyNeRF | 実シーン（多視点動画） | Neural 3D Video公式 | フレーム抽出 + COLMAP |

`data/` 配下に配置する（`.gitignore` 管理外を想定）。

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
