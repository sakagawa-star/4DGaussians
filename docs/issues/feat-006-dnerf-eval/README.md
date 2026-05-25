# feat-006: D-NeRF評価動作確認

## 概要

4DGaussians の動作確認 第4段階（環境構築 **Phase 5**、最終段階）。feat-005 でレンダリングした D-NeRF 合成シーン **`bouncingballs`** の test セット成果物（`output/dnerf/bouncingballs/test/ours_20000/{renders,gt}/` 各20枚）を `metrics.py` で評価し、画質指標 **PSNR / SSIM / LPIPS-vgg / LPIPS-alex / MS-SSIM / D-SSIM** が算出され `results.json` / `per_view.json` が生成されることを確認する。

- 公式 README「Evaluation」のコマンドを使用する（正式な引数名は `--model_paths`/`-m`。公式 README は単数 `--model_path` と記載するが argparse の前置一致で受理される）
- `metrics.py` は `model_path` 配下の **`test/` のみ**を評価し、`test/` 直下の各 `ours_*`（method）を全列挙して評価する。**現状 test 配下は `ours_20000` のみ**
- 算出値を論文（CVPR 2024, arXiv:2310.08528）の D-NeRF `bouncingballs` 報告値（**PSNR 40.62 / SSIM 0.9942 / LPIPS 0.0155**）と照合し、大きな乖離がないことを確認する
- **本案件の完了をもって「D-NeRF が正常動作する環境」の構築が完了**する

## 実行コマンド

```bash
mkdir -p /data/sakagawa/tmp/feat006-dnerf-eval
CUDA_VISIBLE_DEVICES=0 .venv/bin/python metrics.py \
  --model_paths output/dnerf/bouncingballs/ \
  > /data/sakagawa/tmp/feat006-dnerf-eval/metrics.log 2>&1
```

（バックグラウンド実行＋ログをファイルに保存。詳細は design.md §1.4.1）

## ステータス

- **Closed**（2026-05-25 完了。約62秒（A100×1）で test 20枚を評価完走、exit code 0。6指標を算出: **PSNR 40.68 / SSIM 0.9943 / LPIPS-vgg 0.0155 / LPIPS-alex 0.0059 / MS-SSIM 0.9954 / D-SSIM 0.0023**（論文 bouncingballs 値とほぼ一致、特に LPIPS-vgg は論文と同値）。`results.json`/`per_view.json` 生成。LPIPS 重みはキャッシュから読込（再 DL なし）。**コード変更ゼロ・不具合なし**。手動テスト合格。**これをもって「D-NeRF が正常動作する環境」の構築完了**）
- 依存: feat-005（D-NeRFレンダリング動作確認）= Closed（成果物 `test/ours_20000/{renders,gt}` を評価入力に使う）
- 後続: なし（Phase 5 完了 = D-NeRF 動作確認の最終段階）

## 判定基準

以下が**いずれも満たされる**こと:

1. `metrics.py` プロセスが終了コード0で終了し、`Scene: output/dnerf/bouncingballs/` と `Method: ours_20000` が出力され、6指標（SSIM / PSNR / LPIPS-vgg / LPIPS-alex / MS-SSIM / D-SSIM）がすべて数値で表示される（FR-001、design §1.4.5 E1〜E3）
2. `output/dnerf/bouncingballs/results.json` と `output/dnerf/bouncingballs/per_view.json` が生成され、JSON として妥当で `ours_20000` の6指標と20枚分の per-view 値を含む（FR-002）
3. 各指標値が論文報告値・feat-004 実測（ITER14000 PSNR=39.84）と整合する（FR-003、品質妥当性の目安）。目安: **PSNR ≥ 38 dB / SSIM ≥ 0.98 / LPIPS-vgg ≤ 0.05 / LPIPS-alex ≤ 0.03**（詳細・根拠は design §1.4.4）
4. （補助）`/data/sakagawa/tmp/feat006-dnerf-eval/metrics.log` に評価ログ全体が記録される（FR-004、Should）

## 関連ドキュメント

- `requirements.md` — 要求仕様書（FR-001〜004）
- `design.md` — 機能設計書（評価フロー・指標定義・出力構成・合格閾値・エラーハンドリング・ADR）
- `README.md`（プロジェクトルート）— 公式 Evaluation 手順
- `docs/BACKLOG.md` — ロードマップ（Phase 5、feat-006 行）
- `docs/issues/feat-005-dnerf-render/` — 評価入力（test セット）を生成した依存案件
