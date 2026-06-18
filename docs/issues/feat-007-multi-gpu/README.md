# feat-007: マルチGPU運用対応（任意GPU選択）

## 概要

4DGaussians 全体動作環境への拡張 **Phase 6**。複数人共用 GPU サーバー（A100-SXM4-40GB × 7）で、**GPU0 以外の空いている任意の 1 枚**を選んで「学習 → レンダリング → 評価」の 3 経路を動かせるようにする。

- 実機検証（2026-06-18）で `CUDA_VISIBLE_DEVICES=N`（N≠0）だけで物理 GPU N に処理が乗ることを確認済み（**コード修正不要の見込み**）。本案件はその確認の本実行と、運用ルールの文書化が主眼
- コードの `cuda:0` / `cuda` は **`CUDA_VISIBLE_DEVICES` でマスクした後の論理デバイス先頭**を指すため、環境変数で物理 GPU を選べる（物理 GPU0 固定ではない）
- ただし `nvidia-smi` の index と CUDA の番号を一致させるため **`CUDA_DEVICE_ORDER=PCI_BUS_ID` を必須**とする（CUDA 既定の `FASTEST_FIRST` は PCI 順と一致保証がない。Codex レビュー #01 高指摘）
- 本体に分散学習機構（DataParallel/DDP）は存在しない。本案件は「**任意の 1 GPU を選ぶ**」運用であり、複数 GPU 同時使用（分散学習）はスコープ外

## 本案件で行うこと

1. D-NeRF `bouncingballs` を **GPU≠0** で train → render → metrics の 3 経路すべて完走させる
2. 学習実行中、`nvidia-smi` で **指定した物理 GPU N にのみ**当該プロセス（`train.pid` の PID）が乗り、他 GPU（特に GPU0）には乗らないことを確認する
3. 共用サーバーでの **マルチGPU運用ルール**（GPU 選択・`CUDA_DEVICE_ORDER`・ポート衝突回避・分散非対応）を `CLAUDE.md` と本案件 `design.md` に明文化する

## 実行コマンド（例。N は空き GPU、P は空きポートを実行時に選ぶ）

```bash
TMP=/data/sakagawa/tmp/feat007-multi-gpu; mkdir -p $TMP

# 0) 空き GPU と空きポートを確認して N, P を決める（N≠0）
nvidia-smi --query-gpu=index,pci.bus_id,uuid,memory.used,utilization.gpu --format=csv
ss -ltn | grep -E ':6107\b' || echo "port 6107 free"   # 使用中なら別ポートへ

# 再実行時は古い成果物での誤合格を防ぐため出力先を削除（初回は存在しない）
rm -rf output/dnerf/bouncingballs_feat007

# 1) 学習（feat-004 と別 expname で既存成果物を汚さない。$! を train.pid に保存）
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python train.py \
  -s data/dnerf/bouncingballs --port P \
  --expname "dnerf/bouncingballs_feat007" \
  --configs arguments/dnerf/bouncingballs.py \
  > $TMP/train.log 2>&1 &
TRAIN_PID=$!; echo "$TRAIN_PID" > $TMP/train.pid

# 学習の終了コードをゲート（失敗したら後続に進まない）
wait "$TRAIN_PID" || { echo "train failed"; exit 1; }

# 2) レンダリング（学習成功後のみ。network_gui 不使用＝ポート確認不要）
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python render.py \
  --model_path "output/dnerf/bouncingballs_feat007/" --skip_train \
  --configs arguments/dnerf/bouncingballs.py \
  > $TMP/render.log 2>&1 || { echo "render failed"; exit 1; }

# 3) 評価（レンダリング成功後のみ）
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python metrics.py \
  --model_paths output/dnerf/bouncingballs_feat007/ \
  > $TMP/metrics.log 2>&1 || { echo "metrics failed"; exit 1; }
```

（詳細・GPU/ポート選定・負荷確認手順は `design.md` を参照）

## ステータス

- **In Progress**（要求仕様・設計 作成済み。Codex レビュー #01 反映済み、再レビューへ）
- 依存: feat-006（D-NeRF 評価動作確認）= Closed
- 後続: feat-008（COLMAP 環境構築）。Phase 7 以降の実シーン系統は本案件の運用ルールを前提に任意 GPU で動かす

## 判定基準

以下が**いずれも満たされる**こと（詳細・受け入れ基準は requirements.md / design.md）:

1. `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N`（N≠0）で train.py / render.py / metrics.py の 3 経路がいずれも終了コード 0 で完走し、`output/dnerf/bouncingballs_feat007/` 配下に各段階の成果物（point_cloud.ply / test 画像 / results.json）が生成される（FR-001〜003）
2. 学習実行中の `nvidia-smi` で、`train.pid` の PID が**物理 GPU index=N にのみ**現れ、GPU0 を含む他 GPU には現れない。選択 GPU の `uuid` も照合する（FR-004）
3. `CLAUDE.md` に「マルチGPU運用ルール」節が追記され、GPU 選択・`CUDA_DEVICE_ORDER`・ポート衝突回避・分散非対応が明文化される（FR-005）
4. 上記を達成するために 4DGaussians 本体のコードを変更していない（FR-006・コード変更ゼロ）

## レビュー記録

- `reviews/codex-01.result.md` — Codex 初回レビュー結果（高×1・中×2）。`reviews/codex-01.full.log` に全過程
- 反映内容: (高)`CUDA_DEVICE_ORDER=PCI_BUS_ID` 全付与、(中)`train.pid` で PID 証跡、(中)ポート空き確認の手順化

## 関連ドキュメント

- `requirements.md` — 要求仕様書（FR-001〜006）
- `design.md` — 機能設計書（検証フロー・GPU 選択・負荷確認・運用ルール・ADR）
- `reviews/` — Codex レビュー結果（`codex-NN.result.md`）と過程ログ（`codex-NN.full.log`）
- `docs/BACKLOG.md` — ロードマップ（Phase 6、feat-007 行）
- `docs/issues/feat-004-dnerf-train/` `feat-005-dnerf-render/` `feat-006-dnerf-eval/` — 3 経路の単一 GPU 動作確認（本案件の前提）
- `CLAUDE.md` — 実行環境表・開発方針・（本案件で追記する）マルチGPU運用ルール
