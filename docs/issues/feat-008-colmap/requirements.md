# feat-008 要求仕様書: COLMAP環境構築（vcpkgソースビルド版）

本書は `docs/REQUIREMENTS_STANDARD.md` に準拠する。

> **改訂履歴**
> - **2026-06-19 初版**: 公式 CUDA版 AppImage 前提で作成（→ 後述の理由で破棄）。
> - **2026-06-19 全面改訂（本版）**: 実装着手時の実機検証で、**COLMAP は公式に Linux バイナリ（AppImage 含む）を一切配布していない**ことが判明（GitHub Releases 3.3〜4.0.4 を全確認。配布は Windows/.zip と一部旧 Mac のみ）。初版が前提とした `colmap-3.12.6-linux-x86_64-cuda.AppImage` は実在しない。ユーザー判断により導入方式を **vcpkg によるソースビルド** に変更（2026-06-19 承認）。本書はその新方式に基づく。

---

## 1.1 プロジェクト概要

- **何を作るのか**: 本マシンに COLMAP（Structure-from-Motion / Multi-View Stereo ツール）を **vcpkg でソースビルドして導入**し、4DGaussians の実シーン前処理が依存する `colmap` CLI と Python 前処理依存（open3d / scikit-image）を整備する。さらに小規模実データで COLMAP の疎再構成パイプライン（feature_extractor → exhaustive_matcher → mapper）が完走することを確認する。
- **なぜ作るのか**: 実シーン3系統（HyperNeRF / DyNeRF / multipleview）の前処理（`convert.py` / `colmap.sh` / `multipleviewprogress.sh`）はすべて `colmap` バイナリと前処理スクリプトに依存する。本マシンは colmap 未導入・原則 sudo 不可・conda未導入であり、かつ **COLMAP 公式は Linux バイナリを配布していない**ため、ソースビルドによる導入が後続フェーズ（feat-009〜011）の前提となる。
- **誰が使うのか**: 本リポジトリで実シーンの学習・評価環境を構築する開発者（ユーザーおよび Claude Code）。
- **どこで使うのか**: A100×7 を複数人共用する Linux サーバー（Ubuntu 22.04.5 / glibc 2.35、原則 sudo 不可、conda無し、uv管理の `.venv`）。**ヘッドレス**（DISPLAY 未設定）。

## 1.2 用語定義

機能設計書・コード・コミットでも以下の用語を同一の意味で用いる。

| 用語 | 定義 |
|------|------|
| COLMAP | Structure-from-Motion（SfM）および Multi-View Stereo（MVS）を行うオープンソースの再構成ツール。CLI バイナリ名は `colmap`。 |
| vcpkg | Microsoft 製の C/C++ 向けソースベース・パッケージマネージャ。`git clone` + `bootstrap` でユーザー権限導入でき、ポートの依存を自動でソースビルドする。conda エコシステムとは無関係。 |
| ポート / port | vcpkg が管理するパッケージ定義（`ports/<name>/`）。`colmap` ポートは依存（Boost/Ceres/CGAL 等）込みで COLMAP をビルドする。 |
| フィーチャ / feature | ポートのビルドオプション。`colmap[cuda]` のように `[...]` で指定。`core` は「default features を含めない」を意味する特別指定。 |
| default features | フィーチャ未指定時に既定で有効になる機能。colmap ポートでは `gui`（Qt5 GUI）。`[core,...]` で除外できる。 |
| 疎再構成（sparse） | feature_extractor → matcher → mapper により得るカメラ姿勢と疎な3D点群（`cameras/images/points3D`）。 |
| 密再構成（dense） | image_undistorter → patch_match_stereo → stereo_fusion により得る稠密点群。**CUDA 必須**。 |
| GPU SIFT | `--SiftExtraction.use_gpu 1` / `--SiftMatching.use_gpu 1` による GPU 上の SIFT 特徴抽出・照合。CUDA 有効ビルドでは CUDA バックエンドで動作し得る。 |
| ヘッドレス | 物理/仮想ディスプレイ（X サーバー）が無い実行環境。本マシンは DISPLAY 未設定・Xvfb 無し。 |
| compute capability | NVIDIA GPU のアーキ世代。A100 は 8.0（sm_80）。CUDA バイナリが当該アーキ向けコードを含む必要がある。 |
| scratch | リポジトリ外の一時作業領域。本案件では `/data/sakagawa/tmp/feat008-colmap/`。 |

## 1.3 機能要求一覧

### FR-001: COLMAP の vcpkg ソースビルドと PATH 解決

- **概要**: vcpkg を導入し、`colmap[core,cuda]:x64-linux` をソースビルドして、`colmap` コマンドを PATH 上（`~/.local/bin/colmap`）で解決できるようにする。**前提**: ビルドに必須の Fortran コンパイラ（gfortran）を事前に導入しておく（非機能要求の「権限」例外。LAPACK 依存のビルドに必須）。
- **入力**: vcpkg リポジトリ（git clone）、COLMAP ポート（vcpkg 同梱、バージョンは vcpkg レジストリ準拠＝調査時点 3.12.6）。CUDA Toolkit（driver 565.57.01 が完全対応し torch cu116 で実証済みの **11.6 を第一候補**、12.8 は代替）。**Fortran コンパイラ（gfortran、`sudo apt-get install` で導入）**。
- **出力**: vcpkg の `installed/x64-linux/tools/colmap/colmap` にビルドされた COLMAP 実体と、`~/.local/bin/colmap`（PATH 上）から起動できる状態。
- **受け入れ基準**:
  - **`gfortran --version` が成功する**（ビルド前提の Fortran コンパイラが導入済み。未導入だと `lapack-reference` ビルドが失敗する）。
  - vcpkg の bootstrap が成功し、`vcpkg install 'colmap[core,cuda]:x64-linux'` が終了コード 0 で完了する（CUDA 有効・GUI〔Qt5〕除外）。
  - `colmap -h`（ヘルプ表示）が終了コード 0 で完了し、サブコマンド一覧が表示される。
  - `colmap` のバージョンが確認できる（`colmap -h` のヘッダ等。vcpkg レジストリの版＝3.12.x 系を想定）。
  - ビルドに用いた vcpkg の commit（`git rev-parse HEAD`）と CUDA バージョン、採用フィーチャを記録する（再現性のため）。

### FR-002: GPU を用いるサブコマンドの動作確認（ヘッドレス対応）

- **概要**: GPU を使う COLMAP サブコマンド（feature_extractor / exhaustive_matcher）が、本機の A100（sm_80）上で動作することを確認する。ヘッドレスで GPU SIFT が失敗する場合は、CPU SIFT（`use_gpu 0`）にフォールバックして完走させる。**feature_extractor と exhaustive_matcher は独立にフォールバック判定する**（片方が GPU 成功・他方が GPU 失敗でも、失敗側のみ CPU に切替えて完走させる）。
- **入力**: 小規模実データ（FR-004 と共用）。
- **出力**: 特徴抽出・照合結果（database.db への書き込み）。各サブコマンドを GPU/CPU いずれで実行したかをログに記録する（採用モードは `feature=gpu/cpu, matcher=gpu/cpu` の形式で明記）。
- **受け入れ基準**:
  - feature_extractor: `--SiftExtraction.use_gpu 1` がヘッドレスで成功すれば GPU 採用（A100 に負荷が乗ることを `nvidia-smi` で確認）。失敗した場合は `--SiftExtraction.use_gpu 0` で完了させ採用する。
  - exhaustive_matcher: feature の結果と独立に、`--SiftMatching.use_gpu 1` を試し、失敗した場合は `database.db` を保持したまま `--SiftMatching.use_gpu 0` で再実行して採用する。
  - GPU 失敗時はその制約と回避策を `investigation.md`／設計書の運用注記に記録する。
  - 2 サブコマンドとも（採用モードによらず）終了コード 0 で完了し、採用モードがログに残る。

### FR-003: 前処理 Python 依存の整備（open3d 維持確認 + scikit-image 導入）

- **概要**: COLMAP 後段の前処理が依存する Python ライブラリのうち、**open3d（`scripts/downsample_point.py` が利用）は既に `.venv` に 0.19.0 で導入済み**のため維持（非破壊）を確認し、**未導入の scikit-image（`multipleviewprogress.sh` の LLFF が利用）を新規に追加導入**する。
- **入力**: `uv pip install`（**追加的**。`uv sync`/`uv pip sync` は使わない）。
- **出力**: `.venv` で open3d（既存 0.19.0 維持）と scikit-image（新規導入）がともに import でき、既存依存（torch 1.13.1+cu116 / numpy 1.23.5 / mmcv 1.6.0 等）が破壊されていない状態。
- **受け入れ基準**（検証は全て `.venv/bin/python` を明示使用し、bare `python` に依存しない）:
  - `numpy` を **1.23.5 に固定したまま**導入が解決され、導入後も `.venv/bin/python -c "import numpy; print(numpy.__version__)"` が **1.23.5** を返す（巻き上げ禁止）。
  - `.venv/bin/python -c "import open3d"`（既存 0.19.0）と `.venv/bin/python -c "import skimage"`（新規導入版）が成功し、両版が記録される。
  - `.venv/bin/python scripts/downsample_point.py`（引数不足での起動）が **import エラー無し**で実行され、引数不足由来の `IndexError` に到達する（= モジュール解決は成功。`ModuleNotFoundError` は不合格）。
  - 導入後に `.venv/bin/python -c "import torch; print(torch.cuda.is_available())"` が引き続き `True` を返す（既存環境非破壊の確認）。
  - `docs/TECH_STACK.md` に追加ライブラリ（用途・選定理由・**解決された具体的バージョン**）を追記し、`requirements.lock.txt` を `uv pip freeze` で再生成する。

### FR-004: 小規模実データでの疎再構成パイプライン完走

- **概要**: 小規模な実写画像セットに対し、`colmap feature_extractor` → `colmap exhaustive_matcher` → `colmap mapper` を実行し、疎再構成モデルが生成されることを確認する。
- **入力**: 小規模実写画像セット（**COLMAP 公式 GitHub Release のサンプルデータ**＝例 `south-building.zip`〔release tag 3.11.1〕の先頭 15〜20 枚。scratch 配下に取得、非コミット）。
- **出力**: `sparse/0/` 配下に `cameras.bin`・`images.bin`・`points3D.bin`（COLMAP モデル一式）。
- **受け入れ基準**:
  - 3 サブコマンドがいずれも終了コード 0 で完了する。
  - `sparse/0/cameras.bin`・`images.bin`・`points3D.bin` が生成される。
  - **登録画像数・3D点数を実数で検証する**: `colmap model_analyzer --path sparse/0` の出力で、**登録画像数 ≥ 2 かつ 3D点数 > 0** を確認する。代替手段として `colmap model_converter --output_type TXT` で生成した `images.txt`/`points3D.txt` の**ヘッダコメント `# Number of images: N` / `# Number of points: N` を読む**（`images.txt` は1画像2行のため非コメント行数で数えると2倍になる。行数カウント禁止）。bin ファイルの存在・サイズのみでの合格判定は禁止（空に近い失敗モデルを誤合格させないため）。

### FR-005: 密再構成（dense）スモーク確認（Should）

- **概要**: CUDA を直接用いる `image_undistorter` → `patch_match_stereo` → `stereo_fusion` が本機で起動・完走することをスモーク確認する（ヘッドレス無関係に CUDA 直実行）。後続 feat-009/010 が dense を使うため前倒し検証。
- **入力**: FR-004 で得た疎再構成モデルと画像。
- **出力**: `dense/fused.ply`（非空）。
- **受け入れ基準**: 3 サブコマンドが終了コード 0 で完了し、`fused.ply` が生成・非空である。**本要求は Should（BACKLOG の必須判定は mapper まで）。失敗時は制約を記録し、必須判定（FR-004）の合否には影響させない。**

### FR-006: 導入手順とヘッドレス運用注記の文書化

- **概要**: COLMAP の vcpkg ソースビルド手順・PATH 解決方法・ヘッドレスでの GPU SIFT 制約と回避策を、再現可能な形で文書化する。
- **入力**: FR-001〜005 の実施結果。
- **出力**: 設計書（`design.md`）への手順記載、`CLAUDE.md` 実行環境表・必要なら運用ルールの更新、`docs/TECH_STACK.md` の更新、`requirements.lock.txt` の再生成。
- **受け入れ基準**:
  - `/clear` 後でも本書＋設計書のみで COLMAP 導入を再現できる記述になっている（vcpkg 取得元・配置パス・`vcpkg install` コマンド・CUDA設定・PATH 設置・検証コマンドを明記）。
  - ヘッドレスでの GPU SIFT 制約と回避策（`use_gpu 0` 等）が明記されている。
  - `CLAUDE.md` 実行環境表の「colmap 未インストール」を実態（vcpkg ビルド導入済み・パス）に更新する。

## 1.4 非機能要求

- **対応環境**: Ubuntu 22.04.5 / glibc 2.35 / A100（sm_80）。**ヘッドレス**で動作すること（X ディスプレイを前提にしない）。
- **権限**: **原則 sudo 不可**。システムへのインストール（apt 等）を行わず、ユーザー権限（`~/`・`/data/sakagawa/` 配下）のみで完結することを基本とする。vcpkg・ビルド成果物はユーザー権限ディレクトリに置く。**例外**: ビルドに必須の Fortran コンパイラ（gfortran）は本機に存在せず userspace 展開も煩雑・脆いため、**GPUサーバー管理者の承認（2026-06-19）のもと `sudo apt-get install gfortran` でシステム導入することを、当該パッケージに限り例外的に許可**する（経緯・ADR は design.md ADR-6 / investigation.md イテレーション1〜2）。これ以外の依存は引き続きユーザー権限の vcpkg ビルドで完結する。
- **ビルドツール**: 本機は **cmake / ninja が未導入**。vcpkg が `downloads/tools/` に自前の cmake/ninja を取得して使う想定（実装時に検証）。C++ コンパイラは gcc/g++ 11.4 を使用。**Fortran コンパイラ（gfortran）が必須**: COLMAP は SuiteSparse(CHOLMOD)/Ceres 経由で BLAS/LAPACK に依存し、vcpkg の Linux 向け LAPACK 提供ポート `lapack-reference` は Reference LAPACK を Fortran ソースからビルドするため。本機未導入のため上記「権限」例外に基づき gfortran を導入する。
- **ビルド時間・ディスク**: 依存（Boost/Ceres/CGAL/glew/FLANN/OpenImageIO 等）を全てソースビルドするため、**初回ビルドは数時間規模・ディスク十数 GB** を要し得る。vcpkg ツリーは `/data`（15TB 空き）に配置する。長時間ビルドはバックグラウンド実行し、ログで監視する。
- **処理時間（検証側）**: 小規模データ（画像 〜数十枚）の疎再構成は実用的時間（おおむね 10 分以内、A100×1）で完了すること。これは目安であり合否判定は終了コードと生成物で行う。
- **信頼性**: 検証データ・中間生成物は scratch（`/data/sakagawa/tmp/feat008-colmap/`）に置き、リポジトリにコミットしない。再実行時は出力先を削除してから再実行できること。
- **再現性**: ビルドに用いた vcpkg の commit ハッシュ、COLMAP ポート版、CUDA バージョン、採用フィーチャを記録する。

## 1.5 制約条件

- **使用必須**:
  - COLMAP（**vcpkg ポートの版＝調査時点 3.12.6**、CUDA 有効ビルド）。
  - Python 前処理依存: **scikit-image**（未導入のため新規に `uv pip install`）。**open3d は既存 0.19.0 を維持**（追加導入しない）。
- **使用禁止 / 回避**:
  - **conda / micromamba の導入は行わない**（プロジェクト方針＝uvでconda回避）。vcpkg は conda エコシステムとは無関係であり、本制約に抵触しない。
  - **`uv sync` / `uv pip sync` は使わない**（未宣言パッケージを削除し環境破壊するため。`uv pip install` の追加的運用のみ）。
  - **4DGaussians 本体コードの改変は原則行わない**（本案件は環境構築であり本体改変を要しない見込み）。
- **ネットワーク**: vcpkg リポジトリ（GitHub）・各依存ソース・COLMAP 公式サンプルデータの取得にインターネットアクセスを用いる（調査時に GitHub 到達確認済み）。
- **ディスク**: vcpkg ツリー（downloads/buildtrees/packages/installed）は十数 GB に達し得る。`/data` に配置しリポジトリにはコミットしない。

## 1.6 優先順位（MoSCoW）

| 要求 | 優先度 | 備考 |
|------|--------|------|
| FR-001 COLMAP の vcpkg ソースビルド・PATH解決 | **Must** | 判定基準1に対応 |
| FR-002 GPUサブコマンド動作（CPUフォールバック可） | **Must** | GPU不可時もCPUで完走すれば可 |
| FR-003 open3d/scikit-image 導入 | **Must** | 判定基準2に対応 |
| FR-004 疎再構成パイプライン完走 | **Must** | 判定基準3に対応 |
| FR-005 dense スモーク確認 | **Should** | 後続前提の前倒し検証。必須判定には含めない |
| FR-006 文書化 | **Must** | 再現性・運用のため |

- **MVP の範囲**: FR-001 + FR-002（GPUまたはCPU）+ FR-003 + FR-004 + FR-006。これらの達成をもって BACKLOG の判定基準（colmap -h 通過 / downsample_point.py import / 小規模 feature_extractor〜mapper 完走）を満たす。FR-005 は前倒し価値はあるが MVP 外。
