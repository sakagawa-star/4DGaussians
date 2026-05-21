# feat-001 要求仕様書: uv環境構築・依存インストール

本書は `docs/REQUIREMENTS_STANDARD.md` に準拠する。

---

## 1.1 プロジェクト概要

- **何を作るのか**: 4DGaussians本体を実行するためのPython仮想環境を、uv（Python 3.10）で構築し、
  PyTorch 1.13.1+cu116 系および `requirements.txt` の依存パッケージをインストールする。
- **なぜ作るのか**: 本マシンには conda が無く、公式手順（conda + Python 3.7）をそのまま実行できない。
  4DGaussiansの学習・レンダリング・評価（feat-004〜006）および CUDA拡張ビルド（feat-002）の前提として、
  PyTorch がGPU（CUDA 11.6）を認識する再現可能なPython環境が必要である。
- **誰が使うのか**: 本プロジェクトの開発者（Claude Code を含む）。構築後の仮想環境を後続の全案件で共用する。
- **どこで使うのか**: `/data/sakagawa/4DGaussians`。GPU=NVIDIA A100-SXM4-40GB（Driver 565.57.01）、
  CUDA Toolkit 11.6（`/usr/local/cuda-11.6`、ビルド用）、uv 0.11.6、gcc 11.4.0。
  グローバルの `CUDA_HOME` は `/home/sakagawa/cuda/current`（=CUDA 12.8、別環境用）。本案件では `CUDA_HOME` を変更しない（11.6 への一時上書きは feat-002 のビルド時のみ行う）。

---

## 1.2 用語定義

本書・機能設計書・コード内で同じ用語を用いる。

| 用語 | 定義 |
|------|------|
| venv（仮想環境） | `uv venv` が生成するプロジェクト隔離Python環境。本案件では `/data/sakagawa/4DGaussians/.venv` |
| uv managed Python | uv が自前でダウンロード・管理するPythonインタプリタ。本マシンには cpython-3.10.16 が導入済み |
| cu116 | CUDA 11.6 向けにビルドされたPyTorch wheel系列。PyTorch専用index `https://download.pytorch.org/whl/cu116` で配布 |
| cu116 index | 上記URLのPyTorch wheel配布index。torch/torchvision/torchaudio のCUDA版を配布する。PyPIとは別 |
| ローカルバージョン識別子 | PEP 440 のローカル版（`+cu116` 等、`+` 以降）。`torch==1.13.1` の制約は `1.13.1+cu116` を満たす |
| lockファイル | `uv pip freeze` 出力を保存した `requirements.lock.txt`。導入版（推移的依存含む）の厳密な正本 |
| `requirements.txt` | 4DGaussians公式の緩い依存スペック（`torch==1.13.1` 等、CUDAサフィックス無し） |
| 実行時CUDA | PyTorch wheel に同梱されるCUDAランタイム。`CUDA_HOME` のnvccとは独立（実行時は `CUDA_HOME` 非依存） |

---

## 1.3 機能要求一覧

### FR-001: 仮想環境の作成

- **機能名**: uv によるPython 3.10仮想環境の作成
- **概要**: プロジェクトルートに Python 3.10 の仮想環境 `.venv` を作成する。
- **入力**: `uv venv --python 3.10`（カレント=プロジェクトルート）
- **出力**: `/data/sakagawa/4DGaussians/.venv` ディレクトリ。Python実行系は 3.10系（3.10.16 以上）。
- **受け入れ基準**: `.venv/bin/python --version` のメジャー・マイナーが `3.10` であること（`uv venv --python 3.10` が選ぶ uv managed cpython-3.10系。本マシンの現状は 3.10.16。パッチ版は uv の導入状況により 3.10.16 以降になりうるが、3.10系であれば合格）。

### FR-002: PyTorch（cu116）の導入

- **機能名**: torch/torchvision/torchaudio の CUDA 11.6 版インストール
- **概要**: cu116 index から torch 1.13.1+cu116, torchvision 0.14.1+cu116, torchaudio 0.13.1+cu116 を導入する。
- **入力**: `uv pip install torch==1.13.1 torchvision==0.14.1 torchaudio==0.13.1 --index-url https://download.pytorch.org/whl/cu116`
- **出力**: venv 内に上記3パッケージ（いずれも `+cu116` ローカル版）がインストールされる。
- **受け入れ基準**: venv の Python で `torch.__version__ == '1.13.1+cu116'` かつ `torchvision.__version__` が `0.14.1+cu116` を返す。

### FR-003: 残り依存パッケージの導入

- **機能名**: `requirements.txt` の torch系を除く依存のインストール
- **概要**: `requirements.txt` のうち torch/torchvision/torchaudio および argparse を除くパッケージ
  （mmcv==1.6.0, matplotlib, lpips, plyfile, pytorch_msssim, open3d, imageio[ffmpeg]）を導入する。
  argparse は Python 3.10 標準ライブラリであり、PyPI 配布の argparse（最終リリース2010年）を導入すると標準ライブラリを
  古い実装でシャドーイングする恐れがあるため除外する（除外しても標準ライブラリの argparse が使えるため機能影響はない）。
  FR-002 で導入済みの torch系が CPU版へ置換されないよう、torch系3行も本ステップのインストール対象から除外する。
  なお mmcv 1.6.0 は CUDA/C++ ops のビルドは不要（純Python版）だが、cp310 wheel が配布されないため
  sdist からのインストール処理（`setup.py` 実行）が発生する。
- **入力**: torch系3行を除外した `requirements.txt` 派生内容に対する `uv pip install`
- **出力**: 上記パッケージが venv にインストールされ、torch系は `+cu116` のまま維持される。
- **受け入れ基準**:
  - `import mmcv` 成功、かつ `mmcv.__version__ == '1.6.0'`、かつ `from mmcv import Config` 成功
  - `import lpips, plyfile, pytorch_msssim, open3d, imageio, matplotlib, torchaudio` が全て成功
  - 本ステップ完了後も `torch.__version__ == '1.13.1+cu116'`（CPU版に退行していない）

### FR-004: GPU認識の検証

- **機能名**: PyTorch からのCUDA 11.6 / A100 認識確認
- **概要**: venv の PyTorch がCUDA 11.6 を報告し、A100 GPU を認識することを確認する。
- **入力**: venv の Python での検証スクリプト実行（`design.md` 参照）
- **出力**: 検証結果（コンソール出力）。
- **受け入れ基準**: 以下が全て成立する。
  - `torch.version.cuda == '11.6'`
  - `torch.cuda.is_available() == True`
  - `torch.cuda.device_count() >= 1`
  - `torch.cuda.get_device_name(0)` が `'A100'` を部分文字列として含む

### FR-005: 依存スナップショット（lock）の生成とgit管理

- **機能名**: `requirements.lock.txt` の生成
- **概要**: 構築完了直後に `uv pip freeze` の全出力を `requirements.lock.txt` に保存し、git にコミットできる状態にする。
- **入力**: `uv pip freeze`（venv 有効）
- **出力**: `/data/sakagawa/4DGaussians/requirements.lock.txt`（推移的依存を含む全パッケージ＝版の正本）
- **受け入れ基準**:
  - `requirements.lock.txt` が存在し、空でない
  - `mmcv==1.6.0` の行を含む
  - torch 行に `1.13.1+cu116` の文字列が含まれる（`grep -i '^torch' requirements.lock.txt` の出力で確認。`uv pip freeze` の
    表記形式差で機械的grepが空振りした場合は出力を目視確認する）。**cu116保持の最終判定は FR-003/FR-004 の
    `torch.__version__ == '1.13.1+cu116'` および `torch.version.cuda == '11.6'` の assert を正とする**（lockのgrepは補助）

---

## 1.4 非機能要求

| 区分 | 要求 |
|------|------|
| 対応環境 | Ubuntu Linux、NVIDIA A100（GPU必須）、CUDA 11.6（ビルド用、本案件では実行時のみ間接利用）、uv 0.11.6 |
| 再現性 | 構築後に `requirements.lock.txt` を生成し、同一環境を復元可能にする（uv 依存管理ルールの第4項） |
| 信頼性 | 既存のグローバル環境（`~/cuda/current`=12.8、システムPython）を一切変更しない。venv 内に閉じる |
| 処理時間 | wheel ダウンロードを含め、通常のネットワーク条件で完了する（具体的な上限秒数は設けない＝ネットワーク依存のため） |
| 非破壊性 | `uv sync` / `uv pip sync` を使用しない（環境破壊防止、TECH_STACK.md「uv依存管理ルール」厳守） |

---

## 1.5 制約条件

- **使用必須**: uv 0.11.6（パッケージ管理）、Python 3.10（uv managed）、torch 1.13.1+cu116、mmcv 1.6.0
- **使用禁止**: conda（未インストール）、`uv sync` / `uv pip sync`（剪定動作で環境破壊の恐れ）、`pyproject.toml` の新規作成
- **CUDA方針**: 本案件では `CUDA_HOME` を上書きしない。torch wheel は自前のCUDAランタイムを同梱するため
  インストール・実行ともにグローバルの `CUDA_HOME`（12.8）に依存しない。`CUDA_HOME` の 11.6 上書きは feat-002（CUDA拡張ビルド）でのみ行う。
- **torch導入方式の制約**: torch系は cu116 index から明示導入する。`requirements.txt` の `torch==1.13.1`（無印）を
  デフォルトindex（PyPI）からそのまま解決させると CPU版になり得るため、torch系は分離して導入する（FR-002/FR-003）。
- **ネットワーク**: cu116 index（download.pytorch.org）および PyPI への到達が必要。オフライン動作は対象外。
- **既存venvの扱い**: 現状 `.venv` は存在しない。万一存在する場合は FR-001 実行前に状態を確認し、ユーザー判断を仰ぐ（無断削除しない）。

---

## 1.6 優先順位（MoSCoW）

| 要求ID | MoSCoW | 備考 |
|--------|--------|------|
| FR-001 | Must | 仮想環境がなければ何も始まらない |
| FR-002 | Must | torch cu116 が判定基準（CUDA 11.6認識）の核 |
| FR-003 | Must | 4DGS本体は mmcv 等に依存。学習・評価に必須 |
| FR-004 | Must | 判定基準そのもの |
| FR-005 | Must | uv依存管理ルールで義務化。再現・復旧の正本 |

- **MVP範囲**: FR-001〜FR-005 の全てが MVP。いずれか一つでも欠けると後続案件（feat-002以降）に進めないため、
  本案件に Should/Could/Won't の段階は設けない（全て Must）。
- **未検証事項（実装時に検証し、失敗時は計画内対処または中断・報告）**: 詳細は `design.md`「未検証事項と失敗時の扱い」。
  - mmcv 1.6.0 の sdist インストールが Python 3.10 で成功するか（cp310 wheel が無く sdist ビルドが走るため）
  - open3d / imageio の最新版が要求する numpy 版が torch 1.13.1 と整合するか
