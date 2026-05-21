# CLAUDE.md

このファイルはClaude Codeがプロジェクトを理解するためのガイドです。

## セッション引き継ぎ

- セッション開始時にプロジェクトルートの `.claude/handovers/` ディレクトリを確認し、ファイルが存在すれば最新のものを読み込む
- セッション終了時や作業の区切りでは `/handover` の実行を促す

## プロジェクト概要

本プロジェクトは、HUST/4DGaussians（4D Gaussian Splatting for Real-Time Dynamic Scene Rendering, CVPR 2024, arXiv:2310.08528）を動作させるための環境構築を行う。

### 現在の最終目的

- **4DGaussiansが正常に動作する環境を構築すること**
- まずD-NeRF（合成シーン）の学習・レンダリング・評価が一通り動くことをゴールとする
- 段階的に、CUDA拡張のビルド → 学習 → レンダリング → 評価、と積み上げて確認する

### 背景

- 公式README（`README.md`）はconda + Python 3.7 + PyTorch 1.13.1+cu116 を前提とした手順を案内している
- 本マシンには **conda が無い** ため、ViTPose環境で実績のある **uv** を用いて環境を構築する（[docs/TECH_STACK.md](docs/TECH_STACK.md) 参照）
- CUDA拡張（`depth-diff-gaussian-rasterization`, `simple-knn`）のソースビルドが必須であり、これがセットアップの主要難所となる
- 開発プロセスのルール（要求仕様書・機能設計書・レビュー・不具合修正の基準）は ViTPose プロジェクト（`~/git/ViTPose/docs/`）で確立したものを移植している

## 実行環境（本マシン）

| 項目 | 値 |
|------|-----|
| GPU | NVIDIA A100-SXM4-40GB × 7（Driver 565.57.01）。学習・レンダリングは単一GPUを使用 |
| CUDA Toolkit | 12.8（`~/cuda/cuda-12.8/`、`~/cuda/current` → `~/cuda/cuda-12.8` のシンボリックリンク） |
| nvcc | release 12.8, V12.8.93（`nvcc --version` で確認可能） |
| CUDA環境変数 | `CUDA_HOME=/home/sakagawa/cuda/current`、`PATH` と `LD_LIBRARY_PATH` にcurrent配下を設定済み |
| システムPython | 3.10.12 |
| パッケージ管理 | uv 0.11.6（インストール済み） |
| conda | **未インストール**（公式手順のconda部分はuvで代替する） |
| colmap | **未インストール**（実シーン用。D-NeRF合成シーンでは不要） |
| リポジトリ | `/data/sakagawa/4DGaussians`（git clone済み） |

## 技術スタック

- **詳細**: [docs/TECH_STACK.md](docs/TECH_STACK.md) を参照
- **本プロジェクトの採用版**: Python **3.10**（uv managed）/ torch **1.13.1+cu116** / mmcv 1.6.0 ＋各種ライブラリ（`requirements.txt`）。公式のPython 3.7はuvが提供しないため3.10を採用（根拠はTECH_STACK.md）
- **公式要件（参考）**: Python 3.7 / PyTorch 1.13.1+cu116 / mmcv 1.6.0（conda前提のオリジナル手順）
- **CUDA拡張**: `submodules/depth-diff-gaussian-rasterization`（深度対応版、ingra14m fork）、`submodules/simple-knn`
- **CUDA整合（決定済み）**: CUDA拡張のビルドは `/usr/local/cuda-11.6` を使う（torch cu116とメジャー・マイナー一致）。グローバルでアクティブな CUDA 12.8 のままビルドするとメジャー版不一致（12 vs 11）でビルドエラーまたは重大な互換性警告が生じうるため、**ビルド時のみ `CUDA_HOME=/usr/local/cuda-11.6` を上書き**する。詳細は TECH_STACK.md

## サブモジュールの状態

- `submodules/depth-diff-gaussian-rasterization` と `submodules/simple-knn` は **未初期化（空）**。`git submodule update --init --recursive` が必要
- `SIBR_viewers`（`.gitmodules` に定義）は可視化ビューア。環境構築の必須要件ではない

## データセット

学習・評価には外部データセットが必要（`data/` 配下、`.gitignore` 管理外を想定）。

- **D-NeRF（合成シーン）**: 最初の動作確認に使用。Dropbox から `data.zip` をDL（README参照）
- **HyperNeRF（実シーン）**: colmap 前処理が必要
- **Plenoptic / DyNeRF（Neural 3D Video）**: フレーム抽出 + colmap 前処理が必要

## ディレクトリ構成（主要部分）

```
4DGaussians/
├── CLAUDE.md               # 本ファイル
├── README.md               # オリジナルのREADME（公式セットアップ手順）
├── requirements.txt        # pip依存定義（torch 1.13.1 等）
├── train.py                # 学習エントリポイント
├── render.py               # レンダリング
├── metrics.py              # 評価
├── convert.py / colmap.sh  # COLMAP前処理（実シーン用）
├── full_eval.py            # 一括評価
├── export_perframe_3DGS.py # タイムスタンプ別3DGS書き出し
├── merge_many_4dgs.py      # 学習済み4DGSのマージ
├── multipleviewprogress.sh # 多視点データのポーズ・点群生成
├── arguments/              # 学習設定（シーン別Pythonコンフィグ）
│   ├── dnerf/              # D-NeRF（bouncingballs, lego, mutant 等）
│   ├── dynerf/             # DyNeRF（Neural 3D Video）
│   ├── hypernerf/          # HyperNeRF
│   ├── dycheck/            # DyCheck
│   └── multipleview/       # 多視点
├── scene/                  # シーン・データセット・モデル定義
│   ├── gaussian_model.py   # ガウシアンモデル
│   ├── deformation.py      # 変形フィールド
│   ├── hexplane.py         # Multi-resolution HexPlane
│   ├── dataset_readers.py  # データセット読み込み
│   └── ...
├── gaussian_renderer/      # レンダラ（network_gui.py 含む）
├── utils/                  # 各種ユーティリティ（loss, camera, graphics 等）
├── lpipsPyTorch/           # LPIPS実装
├── scripts/                # 前処理・変換・学習補助スクリプト
│   ├── preprocess_dynerf.py
│   ├── downsample_point.py
│   ├── *2colmap.py         # 各データセット→COLMAP変換
│   └── train_*.sh          # シーン別学習スクリプト
├── submodules/             # CUDA拡張（要ビルド・初期状態は未初期化）
│   ├── depth-diff-gaussian-rasterization/
│   └── simple-knn/
├── docs/                   # ドキュメント（開発プロセス基準）
│   ├── BACKLOG.md
│   ├── BUGFIX_STANDARD.md
│   ├── DESIGN_STANDARD.md
│   ├── REQUIREMENTS_STANDARD.md
│   ├── REVIEW_CRITERIA.md
│   ├── TECH_STACK.md
│   ├── viewer_usage.md     # 公式ビューアの使い方（オリジナル）
│   └── issues/             # 案件ディレクトリ
└── assets/                 # README用画像等
```

## 開発方針

- **シンプルな機能を一つずつ作り、積み重ねて目的を達成する**
- 大きな機能を一度に作らない。小さく作って動作確認し、次へ進む
- 4DGaussiansのオリジナルコードは可能な限り変更しない。変更が必要な場合は理由を案件ドキュメントに記録する
- 環境構築は段階的に進める（CUDA拡張ビルド → 学習 → レンダリング → 評価）。各段階で判定基準を満たしてから次へ進む

### 機能追加フロー（feat-XXX 案件）

新機能や新しい環境構築ステップを進める場合、以下のフローを**厳守**する。**planモードは使わない**（通常モードで調査・計画を行う）。

1. **案件作成** → `docs/issues/feat-{number}-{slug}/` フォルダを作成し、`docs/BACKLOG.md` に追加する
2. **調査・計画** → 通常モードで既存コード・公式手順を調査し、要求仕様書（`docs/REQUIREMENTS_STANDARD.md` 準拠）と機能設計書（`docs/DESIGN_STANDARD.md` 準拠）を作成する
3. **ドキュメント保存** → 要求仕様書を `docs/issues/{案件フォルダ}/requirements.md`、機能設計書を `docs/issues/{案件フォルダ}/design.md` にファイル保存する。**保存が完了するまで実装に進んではならない**
4. **レビュー（Subagent + 人）** → 保存されたドキュメントをSubagent（Agentツール）でレビューする。ユーザーも同時にレビューする。レビュー実行時は `docs/REVIEW_CRITERIA.md` の基準に従うこと
5. **修正（必要な場合）** → レビューで問題があれば、再調査してドキュメントを更新する。**ステップ2〜4を問題がなくなるまで繰り返す**
6. **実装** → ドキュメント（要求仕様書・機能設計書・CLAUDE.md）を読んで実装する。実装完了後、動作確認を実行する
7. **手動テスト** → ユーザーがテストする。以下の問題があれば `docs/BUGFIX_STANDARD.md` に従って修正計画を `docs/issues/{案件フォルダ}/investigation.md` に追記する（上書きしない。イテレーション番号を付けて履歴を残す）。**ユーザーの承認を得た上で、ステップ2〜7を繰り返す**（コード修正はステップ6で行う。ステップ7で直接コードを編集してはならない）
   - 不具合の発見
   - 要求通りに実装されていない
   - 要求仕様作成時のヒアリング漏れ
8. **完了** → `docs/BACKLOG.md` のステータスを Closed に更新する。ファイルの追加・削除があった場合は `CLAUDE.md` のディレクトリ構成を最新に更新する

### 不具合修正フロー（bug-XXX 案件）

既存機能・既存環境の不具合を修正する場合、以下のフローを**厳守**する。

1. **案件作成** → `docs/issues/bug-{number}-{slug}/` フォルダを作成し、`docs/BACKLOG.md` に追加する。`README.md` に不具合の概要と再現手順を記録する
2. **調査・修正計画** → `docs/BUGFIX_STANDARD.md` に従い、既存コードを調査する。修正計画を `docs/issues/{案件フォルダ}/investigation.md` に記録する。**この時点でコードを編集してはならない**
3. **ドキュメント保存** → investigation.md の保存を確認する。**保存が完了するまで実装に進んではならない**
4. **レビュー（Subagent + 人）** → 保存されたドキュメントをSubagent（Agentツール）でレビューする。ユーザーも同時にレビューする。レビュー実行時は `docs/REVIEW_CRITERIA.md` の基準に従うこと
5. **修正（必要な場合）** → レビューで問題があれば、再調査してドキュメントを更新する。**ステップ2〜4を問題がなくなるまで繰り返す**
6. **実装** → 承認された修正計画に沿ってコードを修正する。計画にない変更が必要になった場合は中断して報告する
7. **手動テスト** → ユーザーがテストする。問題があれば `docs/BUGFIX_STANDARD.md` に従って investigation.md にイテレーション番号を付けて追記し、**ユーザーの承認を得た上で、ステップ2〜7を繰り返す**（コード修正はステップ6で行う。ステップ7で直接コードを編集してはならない）
8. **完了** → `docs/BACKLOG.md` のステータスを Closed に更新する。ファイルの追加・削除があった場合は `CLAUDE.md` のディレクトリ構成を最新に更新する

### ドキュメント作成ルール

- **実装前に必ずドキュメントを作成し、案件フォルダにファイル保存すること**
- ドキュメントが保存されていない場合は、**実装を中止**する
- 機能追加時: 要求仕様書（`docs/REQUIREMENTS_STANDARD.md` 準拠）と機能設計書（`docs/DESIGN_STANDARD.md` 準拠）を作成する
- 不具合修正時: `docs/BUGFIX_STANDARD.md` の基準に従い、修正計画を `investigation.md` に記録する
- レビュー実行時は `docs/REVIEW_CRITERIA.md` の基準に従うこと
- ドキュメントは `docs/issues/{案件フォルダ}/` に置く（`requirements.md`, `design.md`, `investigation.md`）
- **/clear 後でも実装がスムーズにできるよう、必要な情報を全て記述する**
- 暗黙知に頼らず、**自己完結したドキュメント**にする（前の会話コンテキストがなくても実装できること）
- ライブラリの追加・変更・削除を行った場合は `docs/TECH_STACK.md` を更新し、**`requirements.lock.txt` を `uv pip freeze` で再生成**すること
- 新規ライブラリ導入時は用途・選定理由・バージョンを `TECH_STACK.md` に追記すること
- **uv 依存管理ルール（厳守）**: パッケージ追加は常に `uv pip install`（追加的）で行い、**`uv sync` / `uv pip sync` は使わない**（未宣言パッケージを削除し環境を破壊するため）。`pyproject.toml` は作らない。背景・詳細・依存記録の役割分担は `docs/TECH_STACK.md`「uv 依存管理ルール」を参照

### 案件ディレクトリ構成

```
docs/issues/
└── {type}-{number}-{slug}/    # 例: feat-001-env-setup, bug-001-xxx
    ├── README.md              # 概要、ステータス、再現手順
    ├── requirements.md        # 要求仕様書（機能追加時、REQUIREMENTS_STANDARD.md 準拠）
    ├── design.md              # 機能設計書（機能追加時、DESIGN_STANDARD.md 準拠）
    └── investigation.md       # 不具合の調査・修正計画（BUGFIX_STANDARD.md 準拠）
```

### 命名規則

- フォルダ名は英語で統一（例: `feat-001-env-setup`, `bug-001-rasterizer-build-fail`）
- 案件フォルダは完了後も削除・移動しない

### コードレビュー

- レビューでは重要度(高/中/低)で分類し、修正提案とともに報告する
