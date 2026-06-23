# feat-010 DyNeRF動作確認（実シーン・多視点）

## 概要

実シーン・多視点（Neural 3D Video / plenoptic video）の1シーンで、
前処理（`preprocess_dynerf.py` フレーム抽出 → `colmap.sh ... llff` → ダウンサンプリング）
→ 学習（`train.py`）→ レンダリング（`render.py`）→ 評価（`metrics.py`）を一通り動かし、
DyNeRF 系統が本環境で動作することを確認する。

- ロードマップ: `docs/BACKLOG.md` Phase 9（feat-010）
- 依存: feat-008（COLMAP 導入済み）
- 関連: feat-009（HyperNeRF。事前生成点群DL方式で動作確認済み）

## ステータス

- **中止（Cancelled）**（2026-06-23）。COLMAP **3.12.6**（feat-008 導入版）が 4DGS の DyNeRF 前処理（`colmap.sh` の `point_triangulator` ＋ 単一カメラ共有 sparse モデル）と **rig/frame 非互換**で動かないことが判明（`investigation.md` イテレーション1-2 参照: `Check failed: existing_frame.RigId() == frame.RigId()`）。
- 対処として **COLMAP を rig 非ネイティブの 3.11.1 に入れ直す新案件 `feat-011-colmap-3.11` を先に実施**し、その後 DyNeRF を **`feat-012-dynerf`** として再開する（番号繰り下げ。2026-06-23 ユーザー決定）。
- 本案件の調査資産は feat-012 で引き継ぐ: **FR-001/FR-002 は完了済み**（`cut_roasted_beef.zip` DL・展開・配置、20カメラ×300フレーム抽出済み。`data/dynerf/cut_roasted_beef/`）、`investigation.md` の root cause 分析、`requirements.md`/`design.md`（MPL・PATH前置・GPU運用などの確定事項）。
- 検証で手動生成した中間生成物（`colmap/` 配下の sparse/dense・`fused.ply`）は COLMAP 3.11.1 導入後に `colmap.sh` フル実走で正規に作り直す。

## 判定基準（BACKLOG より）

前処理3段（preprocess_dynerf.py / colmap.sh llff / downsample_point.py）が完走し、
`train.py`→`render.py`→`metrics.py` が完走する。ffmpeg（imageio）依存に注意
（`scene/dataset_readers.py:readdynerfInfo`、判定は poses_bounds.npy の存在）。

## ドキュメント

- `requirements.md` — 要求仕様書（作成予定）
- `design.md` — 機能設計書（作成予定）
- `reviews/` — Codexレビュー出力

## 再現手順

（実装後に追記）
