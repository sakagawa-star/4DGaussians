# feat-011 COLMAP 3.11.1 併設（rig 非互換回避）

## 概要

COLMAP **3.12.6**（feat-008 導入版）は 4DGS の `colmap.sh`（既知ポーズ `point_triangulator` ＋ 単一カメラ共有 sparse モデル）と **rig/frame 非互換**で、DyNeRF / multipleview の前処理が動かない（`Check failed: existing_frame.RigId()==frame.RigId()` で SIGABRT）。本案件は rig 必須化前の最終版 **COLMAP 3.11.1** を vcpkg で**別 prefix** にビルドして併設し、4DGS 本体を非改変（`colmap.sh:5` の GPU 引数化を除く）のまま `colmap.sh` 系を動かせるようにする。

- ロードマップ: `docs/BACKLOG.md` Phase 7（feat-011）
- 依存: feat-008（vcpkg 環境・3.12.6 導入済み）
- 後続: feat-012（DyNeRF 再開）・feat-013（multipleview）がこの 3.11.1 を使う
- 経緯: 旧 **feat-010-dynerf** の実装中に rig 非互換が判明し中止 → 本案件を挿入（2026-06-23 ユーザー決定）

## ステータス

- **In Progress**（2026-06-23 起票。要求仕様・設計の作成へ）

## 背景（rig 非互換の root cause）

- COLMAP **3.12** で rig/frame モデルがネイティブ化（公式: 「各カメラ＝独立 rig、各 frame＝単一画像」がデフォルト）。
- 4DGS の `scripts/llff2colmap.py` は20画像を単一カメラ（CAMERA_ID=1）で共有させるため、3.12 系は1 rig に20画像を詰めて矛盾 → `point_triangulator` が SIGABRT。
- **3.11.1 は rig 非ネイティブ**のため、単一カメラ共有でも `point_triangulator` が通る見込み（本案件で実証する）。
- 詳細な調査・トレースバック・検証は `docs/issues/feat-010-dynerf/investigation.md`（イテレーション1-2）。

## 検証用データ（feat-010（中止） から流用）

feat-010（中止） で取得・前処理済みの **cut_roasted_beef**（`data/dynerf/cut_roasted_beef/`、20カメラ×300フレーム抽出済み）を 3.11.1 の動作確認に流用する（再取得不要）。

## ドキュメント

- `requirements.md` — 要求仕様書（作成予定）
- `design.md` — 機能設計書（作成予定）
- `reviews/` — Codexレビュー出力

## 進め方

feat フローを最初から踏む（要求仕様 → 設計 → Codexレビュー収束 → 人レビュー → 実装 → 手動テスト）。
