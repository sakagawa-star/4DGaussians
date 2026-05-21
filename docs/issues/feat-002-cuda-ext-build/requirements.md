# feat-002 要求仕様書: サブモジュール初期化・CUDA拡張ビルド

## 1.1 プロジェクト概要

- **何を作るのか**: 4DGaussians が依存する2つのCUDA拡張（`depth-diff-gaussian-rasterization`, `simple-knn`）を、feat-001 で構築した uv 仮想環境（`.venv`、Python 3.10.16 / torch 1.13.1+cu116）上でソースビルドし、Python から import 可能にする。
- **なぜ作るのか**: 4DGS の学習・レンダリング・評価は微分可能ガウシアンラスタライザ（`diff_gaussian_rasterization`）と近傍探索（`simple_knn._C`）のCUDA拡張に依存する。これらはプリビルドwheelが提供されず、ソースビルドが必須。本案件は環境構築 Phase 1 にあたり、以降の feat-003〜006（学習〜評価）の前提となる。
- **誰が使うのか**: 本プロジェクトで 4DGS を動作させる開発者（=ユーザー）。
- **どこで使うのか**: 本マシン（Ubuntu Linux / A100-SXM4-40GB / CUDA Toolkit 11.6 が `/usr/local/cuda-11.6` に存在 / gcc 11.4.0）。`/data/sakagawa/4DGaussians` リポジトリの `.venv`。

## 1.2 用語定義

ドキュメント内・機能設計書・コード内で同じ用語を使う。

- **CUDA拡張**: PyTorch の `torch.utils.cpp_extension`（`CUDAExtension`/`BuildExtension`）でビルドする C++/CUDA 製のPython拡張モジュール。
- **サブモジュール**: 本リポジトリが git submodule として参照する外部リポジトリ。本案件の対象は `submodules/depth-diff-gaussian-rasterization` と `submodules/simple-knn` の2つ。
- **ネストしたサブモジュール（nested submodule）**: サブモジュール自身がさらに参照する git submodule。本案件では `depth-diff-gaussian-rasterization` が参照するヘッダライブラリ `third_party/glm` を指す。`git submodule update --init --recursive` の `--recursive` で取得する。
- **editable インストール**: `pip install -e`（uvでは `uv pip install -e`）。ソースディレクトリをそのまま参照する形でインストールし、コンパイル成果物（`.so`）をソースツリー内に生成する。公式READMEの手順に一致する。
- **build isolation**: ビルド時に隔離された一時環境を作り、そこにビルド依存を入れてビルドする仕組み。これを無効化するのが `--no-build-isolation`（隔離せず `.venv` のツール・torch を使う）。
- **ビルド用CUDA**: CUDA拡張のコンパイルに使う CUDA Toolkit。本プロジェクトは `/usr/local/cuda-11.6`（nvcc 11.6.124）を使う。グローバルにアクティブな `~/cuda/current`（→ `~/cuda/cuda-12.8`、12.8）とは区別する。
- **import 名**: Python の `import` 文で参照するモジュール／パッケージ名（本案件では `diff_gaussian_rasterization`、`simple_knn._C`）。ビルド成果物の `.so` パスやディレクトリ構成ではなく、4DGS本体コードが実際に使う import 名で合否を判定する（design ADR-6）。
- **ベンダリング（vendoring）**: 外部依存（本案件では GLM ヘッダ）を git submodule 参照ではなく、リポジトリ内に直接コミットして同梱すること。glm が submodule 登録されず `third_party/glm/` にヘッダが直接存在する状態を指す（glm 取得経路の判定で使用、design §1.4.6 E2）。

## 1.3 機能要求一覧

### FR-001: サブモジュールの初期化（ネスト含む）

- **機能名**: サブモジュール取得
- **概要**: 未初期化の2サブモジュールと、そのネストしたサブモジュール（glm）を取得する。
- **入力**: `git submodule update --init --recursive` の実行（ネットワークアクセスあり）。
- **出力**: `submodules/depth-diff-gaussian-rasterization/`、`submodules/simple-knn/`、および `submodules/depth-diff-gaussian-rasterization/third_party/glm/` にソースが展開される。
- **受け入れ基準**:
  - `submodules/depth-diff-gaussian-rasterization/setup.py` が存在する。
  - `submodules/simple-knn/setup.py` が存在する。
  - `git submodule status` で両サブモジュールの行頭の `-`（未初期化マーク）が消える。
  - `submodules/depth-diff-gaussian-rasterization/third_party/glm/` 配下にヘッダ（`glm/glm.hpp` 等）が存在する（CUDAコードが GLM ヘッダを include するため必須）。取得経路がネスト submodule かベンダリングかは問わず、**ヘッダの存在を必須基準**とする。欠落時の対処は design.md §1.4.6 E2。

### FR-002: depth-diff-gaussian-rasterization のビルド

- **機能名**: ラスタライザのビルド
- **概要**: `submodules/depth-diff-gaussian-rasterization` を editable インストールでビルドする。
- **入力**: `.venv` の Python（torch 1.13.1+cu116）と、ビルド時に上書きした `CUDA_HOME=/usr/local/cuda-11.6`。
- **出力**: `diff_gaussian_rasterization` パッケージが `.venv` に editable インストールされ、コンパイル済み `_C` 拡張（`.so`）がソースツリー内に生成される。
- **受け入れ基準**:
  - `uv pip install --python .venv/bin/python -e submodules/depth-diff-gaussian-rasterization` が**ビルドエラーなく完了**する。
  - `.venv/bin/python -c "import diff_gaussian_rasterization"` が成功する。
  - `.venv/bin/python -c "from diff_gaussian_rasterization import GaussianRasterizationSettings, GaussianRasterizer"` が成功する。

### FR-003: simple-knn のビルド

- **機能名**: simple-knn のビルド
- **概要**: `submodules/simple-knn` を editable インストールでビルドする。
- **入力**: `.venv` の Python（torch 1.13.1+cu116）と、ビルド時に上書きした `CUDA_HOME=/usr/local/cuda-11.6`。
- **出力**: `simple_knn` パッケージが `.venv` に editable インストールされ、コンパイル済み `_C` 拡張（`.so`）がソースツリー内に生成される。
- **受け入れ基準**:
  - `uv pip install --python .venv/bin/python -e submodules/simple-knn` が**ビルドエラーなく完了**する。
  - `.venv/bin/python -c "import simple_knn._C"` が成功する。
  - `.venv/bin/python -c "from simple_knn._C import distCUDA2"` が成功する。

### FR-004: import 統合検証

- **機能名**: import 検証
- **概要**: 4DGS本体が実際に使うimport文が成立することを確認する。
- **入力**: `.venv/bin/python` での検証スクリプト実行（具体は design.md §1.4.4 の heredoc スクリプト）。
- **出力**: 検証結果の標準出力（成功/失敗）。
- **受け入れ基準**:
  - design.md §1.4.4 の検証スクリプト（`diff_gaussian_rasterization` / `simple_knn._C` を import し、`GaussianRasterizationSettings` / `GaussianRasterizer` / `distCUDA2` のシンボルを import する）が**いずれもエラーなく**完了し、最後に `OK: CUDA ext import | torch 1.13.1+cu116 | cuda 11.6 | device <A100名>` が出力される。**出力文字列は design.md §1.4.4 を正とする**。

### FR-005: 依存スナップショットの更新

- **機能名**: lock 再生成
- **概要**: CUDA拡張2点が editable インストールされたことを `requirements.lock.txt` に反映する。
- **入力**: `uv pip freeze --python .venv/bin/python`。
- **出力**: `requirements.lock.txt` を上書き更新する。
- **受け入れ基準**:
  - `requirements.lock.txt` が空でない。
  - `diff_gaussian_rasterization` と `simple_knn` の2行が含まれる（editable のため `-e file://...` 形式または `@ file://...` 形式で出力されることを許容する。grep が表記形式差で空振りした場合は目視で確認する）。
  - feat-001 で固定した主要版（`torch==1.13.1+cu116`, `mmcv==1.6.0`, `numpy==1.23.5`）が**退行していない**（消えていない・別版に置換されていない）。grep または目視で確認する。

## 1.4 非機能要求

- **パフォーマンス**: ビルド時間は本案件の合否に含めない（数分程度を許容）。`ninja` が `.venv` にあればビルドが並列化され高速になるが、無くても torch のフォールバック（distutils）でビルド可能であり**必須としない**。
- **対応環境**: NVIDIA GPU（A100, sm_80）必須。CUDA Toolkit 11.6（`/usr/local/cuda-11.6`、nvcc 11.6.124）必須。gcc/g++ ≤ 11（本マシンは 11.4.0、適合）。
- **信頼性**: ビルド失敗時は `.venv` 全体を作り直さず、原因を切り分けて再ビルドできること（editable のため再ビルドはソースツリーのクリーン＋再 `uv pip install -e` で可能）。`requirements.lock.txt` は feat-001 時点のものが git 管理されており、退行時は復元できる。
- **再現性**: 全 `uv pip` 操作で `--python .venv/bin/python` を明示する（本マシンは別環境の bashrc が `VIRTUAL_ENV` を常時アクティブにしており、明示しないと取り違える）。

## 1.5 制約条件

- **使用必須**: feat-001 で構築した `.venv`（Python 3.10 / torch 1.13.1+cu116 / setuptools 80.10.2）。CUDA拡張のビルドには `/usr/local/cuda-11.6` を使う。
- **ビルド時のみ CUDA_HOME を上書き**: `CUDA_HOME=/usr/local/cuda-11.6 PATH=/usr/local/cuda-11.6/bin:$PATH` をビルドコマンドの**インライン環境変数**として与える。グローバルの `CUDA_HOME=~/cuda/current`（12.8）は**変更しない**（このマシンの別環境が依存）。
- **build isolation 無効化必須**: `--no-build-isolation` を付与する。サブモジュールの `setup.py` が `torch.utils.cpp_extension` を import するため、隔離環境（torch 不在）ではビルドできない。`.venv` の torch・setuptools<81（pkg_resources 同梱）を使う。
- **uv 依存管理ルール（厳守）**: パッケージ追加は `uv pip install`（追加的）のみ。`uv sync` / `uv pip sync` は使わない。`pyproject.toml` は作らない（`docs/TECH_STACK.md`「uv 依存管理ルール」）。
- **オリジナルコード不変更**: サブモジュールおよび4DGS本体のコードは原則変更しない。変更が必要になった場合は中断し、理由を本案件ドキュメントに記録してユーザーの承認を得る。
- **ネットワーク**: FR-001 で GitHub（ingra14m）と GitLab（INRIA）への git アクセスが必要。

## 1.6 優先順位

| FR | 内容 | MoSCoW |
|----|------|--------|
| FR-001 | サブモジュール初期化（ネスト含む） | **Must** |
| FR-002 | depth-diff-gaussian-rasterization のビルド | **Must** |
| FR-003 | simple-knn のビルド | **Must** |
| FR-004 | import 統合検証 | **Must** |
| FR-005 | 依存スナップショット更新 | **Must** |

- **MVP の範囲**: FR-001〜005 全て。いずれか1つでも欠けると後続の学習・レンダリングが動かないため、全てを Must とする。
