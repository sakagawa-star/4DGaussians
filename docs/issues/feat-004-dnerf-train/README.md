# feat-004: D-NeRF学習動作確認

## 概要

4DGaussians の動作確認 第2段階。D-NeRF 合成シーン **`bouncingballs`** の学習を一通り完走させ、学習成果物が生成されることを確認する。feat-002（CUDA 拡張ビルド）と feat-003（データ配置）の成果を、学習パイプライン上で実際に動作させる最初のステップ。

- 公式 README「Training / For training synthetic scenes」（140〜143行目、コマンドは143行目）のコマンドをそのまま使用する
- coarse（3000 反復）→ fine（20000 反復）の2ステージを完走させる
- 学習成果物（`output/dnerf/bouncingballs/point_cloud/iteration_20000/`）の生成を確認することがゴール。**レンダリング・評価は feat-005 / feat-006 に委譲**

## 実行コマンド

```bash
CUDA_VISIBLE_DEVICES=0 .venv/bin/python train.py \
  -s data/dnerf/bouncingballs \
  --port 6017 \
  --expname "dnerf/bouncingballs" \
  --configs arguments/dnerf/bouncingballs.py
```

（バックグラウンド実行＋ログをファイルに保存。詳細は design.md §1.4.1）

## ステータス

- **Open**（未着手 → 調査・ドキュメント作成中）
- 依存: feat-002（CUDA拡張ビルド）= Closed、feat-003（D-NeRFデータ準備）= Closed
- 後続: feat-005（レンダリング動作確認）

## 判定基準

以下が**いずれも満たされる**こと:

1. 学習プロセスが終了コード0で終了し、ログ末尾に `Training complete.` が出力される（FR-002）
2. `output/dnerf/bouncingballs/point_cloud/iteration_20000/point_cloud.ply`（非空）と `deformation.pth` / `deformation_table.pth` / `deformation_accum.pth`、および `output/dnerf/bouncingballs/cfg_args` が存在する（FR-003）
3. 学習中の test 評価ログ `[ITER N] Evaluating test: L1 ... PSNR ...` が出力され、**fine ステージ**の ITER 14000（fine でのみ到達。評価はステージ内カウントで判定）の test PSNR が 20 以上（学習が破綻していない）（FR-004）
4. 途中で `loss is nan,end training`（NaN 再起動）が発生しない（FR-002）

## 関連ドキュメント

- `requirements.md` — 要求仕様書（FR-001〜005）
- `design.md` — 機能設計書（実行手順・出力構成・saving_iterations 解析・エラーハンドリング・ADR）
- `README.md`（プロジェクトルート）— 公式 Training 手順（140〜143行目、コマンドは143行目）
- `docs/BACKLOG.md` — ロードマップ（Phase 2、feat-004 行）
- `docs/issues/feat-003-dnerf-data/` — 入力データの配置（依存）
