# feat-005: D-NeRFレンダリング動作確認

## 概要

4DGaussians の動作確認 第3段階（環境構築 Phase 4）。feat-004 で学習した D-NeRF 合成シーン **`bouncingballs`** の学習済みモデルを `render.py` で読み込み、レンダリング画像・動画が生成されることを確認する。学習済み 4D Gaussian と変形フィールドが推論経路で動作することを実証するステップ。

- 公式 README「Rendering」（200〜206行目、コマンドは205行目）のコマンドをそのまま使用する
- `--skip_train` 指定で **test（20 カメラ）** と **video（円軌道 160 フレーム）** をレンダリングする
- 学習済みモデルは `--iteration` 既定（-1）で最大 iteration（**20000**）が自動選択される
- 成果物 `output/dnerf/bouncingballs/test/ours_20000/{renders,gt}/` は **feat-006 評価の入力**になる。**品質の定量評価は feat-006 に委譲**

## 実行コマンド

```bash
mkdir -p /data/sakagawa/tmp/feat005-dnerf-render
CUDA_VISIBLE_DEVICES=0 .venv/bin/python render.py \
  --model_path "output/dnerf/bouncingballs/" \
  --skip_train \
  --configs arguments/dnerf/bouncingballs.py \
  > /data/sakagawa/tmp/feat005-dnerf-render/render.log 2>&1
```

（バックグラウンド実行＋ログをファイルに保存。詳細は design.md §1.4.1）

## ステータス

- **Closed**（2026-05-25 完了。約13秒（A100×1）で test 20 枚＋video 160 枚をレンダリング完走、exit code 0。`test/ours_20000/{renders,gt}` 各20枚・`video/ours_20000/renders` 160枚・両セットの `video_rgb.mp4`（非空）を生成。`point nums: 27769`（feat-004 最終と一致）。**コード変更ゼロ・不具合なし**（Pillow 問題は再発せず）。目視確認①〜④合格。手動テスト合格）
- 依存: feat-004（D-NeRF学習動作確認）= Closed（成果物 `output/dnerf/bouncingballs/` を入力に使う）
- 後続: feat-006（評価動作確認）— 本案件が生成する `test/ours_20000/{renders,gt}` を評価入力にする

## 判定基準

以下が**いずれも満たされる**こと:

1. レンダリングプロセスが終了コード0で終了し、ログに `Loading trained model at iteration 20000` が出力され、CUDA 拡張の import/実行エラー・Pillow/numpy 由来の型エラーが発生しない（FR-001、design §1.4.6 E2/E3）
2. `output/dnerf/bouncingballs/test/ours_20000/renders/` と `gt/` に同数（各 20 枚）の非空 PNG が生成される（FR-002）
3. `output/dnerf/bouncingballs/video/ours_20000/renders/` に非空 PNG が 160 枚生成される（FR-003）
4. `output/dnerf/bouncingballs/{test,video}/ours_20000/video_rgb.mp4` が非空で生成される（FR-004、Should）
5. （補助）`/data/sakagawa/tmp/feat005-dnerf-render/render.log` にレンダリングログ全体が記録される（FR-005、Should。合否必須ではない再現性のための補強）

## 関連ドキュメント

- `requirements.md` — 要求仕様書（FR-001〜005）
- `design.md` — 機能設計書（実行フロー・出力構成・iteration 解析・エラーハンドリング・ADR）
- `README.md`（プロジェクトルート）— 公式 Rendering 手順（200〜206行目、コマンドは205行目）
- `docs/BACKLOG.md` — ロードマップ（Phase 4、feat-005 行）
- `docs/issues/feat-004-dnerf-train/` — 入力モデルの学習（依存）。`investigation.md` に Pillow 12.2.0 非互換修正の記録
