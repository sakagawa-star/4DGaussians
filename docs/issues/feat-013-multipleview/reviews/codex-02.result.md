再レビュー結果です。前回4点は本文レベルではほぼ解消されていますが、実行手順としてまだ中程度の不整合が残っています。

**高**
なし。

**中**
1. 前回中1（`--skip_video`）は §5.2 とCP4では解消済みですが、[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-013-multipleview/design.md:219) の「確定コマンド一覧」が `render.py --skip_train` のままで、`--skip_video` が落ちています。§10は「確定コマンド一覧」なので、ここを見て実行すると前回の高負荷 video 300 view 問題が再発します。  
修正提案: FR-004 行を `§5.2 render.py --skip_train --skip_video` に更新する。

2. NFR-005 は「長時間処理はバックグラウンド＋ログ」と定義していますが、設計のCOLMAP/学習コマンドは foreground のままで、リダイレクト先ログもPID保存も具体化されていません（[requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-013-multipleview/requirements.md:60), [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-013-multipleview/design.md:145)）。これは明示NFRと実行手順の不一致です。  
修正提案: CP2 dense と CP3 train について、ログパスと起動形を確定する。例: `... > <scratch>/logs/cp3_train.log 2>&1 & echo $! > <scratch>/pids/cp3_train.pid`。もしくはNFR-005を外す。

**前回指摘の解消状況**
- 中1 `skip_video追加`: 部分解消。§5.2/CP4はOK、§10が未反映。
- 中2 COLMAP feature/matcher GPU方針: 解消。`--SiftExtraction.use_gpu 0` / `--SiftMatching.use_gpu 0` でCPU固定済み。
- 中3 FR-002 `model_analyzer` 健全性: 解消。20/20登録・再投影誤差・点数基準が追加済み。
- 中4 FR-005 数値基準化: 解消。PSNR/SSIM/MS-SSIM/D-SSIM/LPIPS の基準が明記済み。

**低**
致命的でないため省略。