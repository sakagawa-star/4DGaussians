# feat-012 DyNeRF動作確認（実シーン・多視点）・再開

## 概要

実シーン・多視点（Neural 3D Video / plenoptic video）の1シーン（**cut_roasted_beef**）で、
前処理（`preprocess_dynerf.py` フレーム抽出 → `colmap.sh ... llff`〔**COLMAP 3.11.1**〕→ `downsample_point.py`）
→ 学習（`train.py`）→ レンダリング（`render.py`）→ 評価（`metrics.py`）を一通り動かし、
DyNeRF 系統が本環境で動作することを確認する。

中止した **feat-010 の調査資産（データ・root cause・確定事項）を継承する再開案件**。

- ロードマップ: `docs/BACKLOG.md` Phase 9（feat-012）
- 依存: **feat-011（COLMAP 3.11.1 併設・Closed）**
- 前身: feat-010（DyNeRF・中止。COLMAP 3.12.6 rig 非互換のため）
- 関連: feat-009（HyperNeRF・実シーン単眼・Closed）

## ステータス

- **In Progress**（2026-06-24 着手・案件フォルダ作成）。
- 要求仕様書・機能設計書はこれから作成する（feat-010 資産を継承し、3.11.1 対応・前処理済みの現状へ更新）。

## feat-010 からの継承と変更点

### 継承する資産（feat-010 から）

- **データ**: `cut_roasted_beef.zip` DL・展開済み。20カメラ（cam00〜cam20、**cam04欠番**）。`data/dynerf/cut_roasted_beef/`。`poses_bounds.npy`=(20,17)。
- **root cause 分析**: COLMAP 3.12.6 の rig/frame 非互換（`feat-010-dynerf/investigation.md` イテレーション1-2）。
- **requirements.md / design.md**: MPL 環境変数・PATH 前置・GPU 運用などの確定事項。

### feat-011 で解決済み（feat-010 からの変更点）

- **COLMAP は 3.11.1 を使う**。feat-010 で検討した対処 A'（`colmap.sh` 構造変更＋新スクリプト `dynerf_sparse_from_db.py`）は **不要**になった。3.11.1 は rig 非ネイティブのため、**4DGS 本体非改変**のまま `colmap.sh ... llff` が完走する（feat-011 で実証）。
- **前処理3段は feat-011 で実証済み**（成果物が `data/dynerf/cut_roasted_beef/` に存在）:
  - フレーム抽出（`preprocess_dynerf.py`）済み（各 `camNN/` ディレクトリ）
  - `colmap.sh ... llff`（3.11.1）完走 → `colmap/dense/workspace/fused.ply` 387,496点・Mean reprojection error 0.852px・20/20 登録
  - `downsample_point.py` → `points3D_downsample2.ply` 37,361点（≤40k）

→ **feat-012 の残作業は「学習 → レンダリング → 評価」（`train.py` → `render.py` → `metrics.py`）**。
  （要求仕様の段階で、前処理を feat-011 の成果物のまま使うか／feat-012 で再実走して一貫性を確認するかを決める。）

## 判定基準（BACKLOG より）

前処理3段（`preprocess_dynerf.py` / `colmap.sh llff` / `downsample_point.py`）が完走し、
`train.py`→`render.py`→`metrics.py` が完走する。ffmpeg（imageio）依存に注意
（`scene/dataset_readers.py:readdynerfInfo`、判定は `poses_bounds.npy` の存在）。
**COLMAP は feat-011 で導入した 3.11.1 を使う**。

## ドキュメント

- `requirements.md` — 要求仕様書（作成予定、`docs/REQUIREMENTS_STANDARD.md` 準拠）
- `design.md` — 機能設計書（作成予定、`docs/DESIGN_STANDARD.md` 準拠）
- `investigation.md` — 不具合発生時に追記（`docs/BUGFIX_STANDARD.md` 準拠）
- `reviews/` — Codexレビュー出力（`codex-NN.result.md` は git 管理、`codex-NN.full.log` は `.gitignore`）

## 再現手順

（実装後に追記）
