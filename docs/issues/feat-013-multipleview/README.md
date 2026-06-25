# feat-013 multipleview動作確認

## ステータス

**Open（2026-06-25 着手）** — 案件ディレクトリ作成。要求仕様・設計はこれから。

## 概要

多視点カスタムデータ系統（`multipleview`）で、4DGaussians の前処理〜学習が動くことを確認する。実シーン3系統（D-NeRF〔済〕・HyperNeRF〔feat-009 済〕・DyNeRF〔feat-012 済〕）の**最後の1系統**。これをもってロードマップの実シーン系統が一巡する。

想定パイプライン:

1. `multipleviewprogress.sh`（フレーム抽出 → COLMAP → LLFFポーズ生成 → ダウンサンプリング）
2. config 作成（`arguments/multipleview/`）
3. `train.py` 学習 →（必要に応じ `render.py` / `metrics.py`）

## 前提・既知事項

- **COLMAP 実走は 3.11.1 を使う**（`/data/sakagawa/opt/colmap-3.11/bin` を PATH 前置）。3.12.6 は rig 非互換でクラッシュ（feat-011 で判明）。CLAUDE.md「COLMAP の使い分け」節参照。
- `colmap.sh:5` / `multipleviewprogress.sh` の **GPU ハードコード（`CUDA_VISIBLE_DEVICES=0`）引数化** を検討する案件（feat-012 で将来送りにした項目）。本案件で実走するなら引数化＋動作検証をセットで判断する。
- マルチGPU運用ルール（空きGPU選択・`CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N`）・長時間処理のバックグラウンド＋ログ方針を継続。
- 本体改変は原則ゼロ（やむを得ない場合は理由を本ドキュメントに記録）。

## 依存

feat-011（COLMAP 3.11.1 併設）

## ファイル

- `requirements.md` — 要求仕様書（REQUIREMENTS_STANDARD.md 準拠、これから作成）
- `design.md` — 機能設計書（DESIGN_STANDARD.md 準拠、これから作成）
- `reviews/` — Codex レビュー出力

## 次のステップ

CLAUDE.md 機能追加フローのステップ2（調査・計画）へ。既存コード（`multipleviewprogress.sh`・`scripts/*2colmap.py`・`arguments/multipleview/`・`scene/dataset_readers.py` の multipleview 読込経路）と入力データの所在を調査し、requirements.md / design.md を作成する。**planモードは使わない**。
