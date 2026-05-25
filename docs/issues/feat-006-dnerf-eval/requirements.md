# feat-006 要求仕様書: D-NeRF評価動作確認

## 1.1 プロジェクト概要

4DGaussians の動作確認の最終段階（環境構築 **Phase 5**）として、feat-005 でレンダリングした D-NeRF 合成シーン `bouncingballs` の test セット（`output/dnerf/bouncingballs/test/ours_20000/{renders,gt}/` 各20枚）を `metrics.py` で評価し、画質指標が算出されることを確認する。本案件は feat-002（CUDA 拡張ビルド）・feat-003（データ配置）・feat-004（学習）・feat-005（レンダリング）の積み上げの最上段に立つ。

解決したい課題は「学習・レンダリングした結果が、定量指標（PSNR/SSIM/LPIPS 等）で測定でき、論文報告値と大きく乖離しない品質に達しているか」を確認すること。指標の算出と妥当性確認をもって、**「D-NeRF が正常動作する環境」の構築完了**を判定する。

コードの変更は原則行わない（CLAUDE.md 開発方針）。実行環境は本マシン（A100-SXM4-40GB、CUDA 拡張は CUDA 11.6 ビルド、torch 1.13.1+cu116、torchvision 0.14.1+cu116、uv 管理の `.venv`）。

## 1.2 用語定義

- **評価（evaluation）**: レンダリング結果（renders）と正解画像（gt）を画素・特徴量レベルで比較し、画質指標を算出する処理。`metrics.py` の `evaluate()` が担う
- **model_path（scene_dir）**: 評価対象のモデル出力ルート。本案件は `output/dnerf/bouncingballs/`。`metrics.py` は引数 `--model_paths`/`-m`（必須・複数指定可、`nargs="+"`）で受け取り、各 path について評価する
- **method（ours_*）**: `test/` 直下のサブディレクトリ名。`render.py` が `ours_{iteration}` 形式で作る（本構成では `ours_20000`）。`metrics.py` は `os.listdir(test_dir)` で test 直下を全列挙し、各 method を独立に評価する
- **renders / gt**: 評価対象ディレクトリ。`test/ours_20000/renders/`（推論画像）と `test/ours_20000/gt/`（正解画像）。`readImages` が renders 側のファイル名で両方を `Image.open` する（同名前提）
- **6指標**: `metrics.py` が算出する画質指標。**SSIM**（`utils/loss_utils.ssim`）、**PSNR**（`utils/image_utils.psnr`）、**LPIPS-vgg / LPIPS-alex**（`lpipsPyTorch.lpips`, net_type=vgg/alex）、**MS-SSIM**（`pytorch_msssim.ms_ssim`）、**D-SSIM**（`(1 - MS-SSIM)/2` で算出）
- **LPIPS 事前学習重み**: LPIPS 計算に必要な4ファイル。バックボーン重み（torchvision 経由: `vgg16` 528MB / `alexnet` 233MB）と LPIPS 線形層重み（GitHub raw 経由: `vgg.pth` / `alex.pth` 各数KB）。**本マシンでは調査時に DL 済み・`~/.cache/torch/hub/checkpoints/` にキャッシュ済み**（本番実行では再 DL されない。design §1.3）
- **results.json / per_view.json**: `evaluate()` が `scene_dir` 直下に出力する評価結果。前者は method 別の6指標平均、後者は method 別・画像別の6指標値

## 1.3 機能要求一覧

### FR-001: 評価コマンドの実行と6指標の算出

- D-NeRF `bouncingballs` の評価を、README（公式 Evaluation 手順）と BACKLOG（feat-006 行）に記載のコマンドで実行する
  - コマンド: `python metrics.py --model_paths output/dnerf/bouncingballs/`
  - 実行は uv 管理の `.venv/bin/python` 経由とする
  - 単一 GPU を使用する（`CUDA_VISIBLE_DEVICES` で1枚に固定。`metrics.py` は `cuda:0` を固定使用するため `CUDA_VISIBLE_DEVICES` で物理 GPU を選ぶ）
- 入力: 上記コマンド、feat-005 成果物 `output/dnerf/bouncingballs/test/ours_20000/{renders,gt}/`（各20枚）、LPIPS 事前学習重み（キャッシュ済み）
- 出力: 標準出力に Scene/Method ログと6指標の数値
- 優先度: Must
- 受け入れ基準: コマンドが終了コード0で終了し、`Scene: output/dnerf/bouncingballs/`・`Method: ours_20000`・`Metric evaluation progress: 100%`（20枚分）が出力され、6指標（SSIM/PSNR/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM）がすべて数値で表示されること。LPIPS 重みのロード/DL 失敗、Pillow/numpy 由来の型エラー（design §1.4.5 E2/E3）が発生しないこと

### FR-002: 評価結果ファイル（results.json / per_view.json）の生成

- 評価結果が JSON ファイルとして scene_dir 直下に生成されること
  - `output/dnerf/bouncingballs/results.json`: `{"ours_20000": {6指標の平均値}}` の構造
  - `output/dnerf/bouncingballs/per_view.json`: `{"ours_20000": {指標名: {画像名: 値, ...×20}}}` の構造
- 優先度: Must
- 受け入れ基準: 上記2ファイルが生成され、`json.load` でパース可能。`results.json` に `ours_20000` キーがあり、その下に6指標（SSIM/PSNR/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM）が数値で揃う。`per_view.json` に20枚分（test カメラ数）の per-view 値が揃う

### FR-003: 指標値の妥当性（論文報告値との整合）

- 算出された指標値が、論文（CVPR 2024, arXiv:2310.08528, Table 6）の D-NeRF `bouncingballs` 報告値（**PSNR 40.62 / SSIM 0.9942 / LPIPS 0.0155**）および feat-004 実測（ITER14000 test PSNR=39.84）と整合し、大きな乖離がないこと
- 補足: 論文の LPIPS はネット種別（vgg/alex）非明記の単一値。本案件は `metrics.py` が出力する LPIPS-vgg / LPIPS-alex の双方を design §1.4.4 の閾値で照合する
- 合格目安（品質の健全性確認。具体値・根拠は design §1.4.4）:
  - **PSNR ≥ 38 dB**（論文 40.62 / feat-004 39.84。-2dB 程度の許容）
  - **SSIM ≥ 0.98**（論文 0.9942）
  - **LPIPS-vgg ≤ 0.05 / LPIPS-alex ≤ 0.03**（論文 LPIPS 0.0155）
  - MS-SSIM ≥ 0.98 / D-SSIM ≤ 0.01（MS-SSIM から導出）
- 優先度: Should（動作確認が主目的。指標が「算出される」FR-001/002 が Must。値の妥当性は健全性の目安）
- 受け入れ基準: 6指標が上記目安を満たす。桁違いの乖離（例: PSNR < 30dB）が出た場合は学習・レンダリング・評価のいずれかに異常がある可能性として `investigation.md` に記録し原因を切り分ける

### FR-004: 実行ログの保存

- 評価はバックグラウンド実行とし、標準出力・標準エラーをログファイルに保存する
- ログ保存先はリポジトリ外の `/data/sakagawa/tmp/feat006-dnerf-eval/metrics.log` とする（design §1.4.1・§1.8 と同一）
- 優先度: Should
- 受け入れ基準: 上記ログファイルに評価ログ全体（Scene/Method・進捗・6指標）が記録され、完走後に内容を後から確認できること

## 1.4 非機能要求

- **対応環境**: NVIDIA A100-SXM4-40GB を単一使用。`.venv`（Python 3.10、torch 1.13.1+cu116、torchvision 0.14.1+cu116）。LPIPS バックボーンは torchvision、線形層は GitHub raw 由来（いずれもキャッシュ済み）
- **信頼性**: 評価対象画像の欠損（test/renders/gt のいずれか不在）、LPIPS 重みのロード失敗で完走しない場合は**異常終了とみなし、自動リトライはしない**。原因を `investigation.md` に記録する。失敗系の詳細は design §1.4.5（E1〜E6）を参照
- **VRAM**: 40GB に収まること。評価は1枚ずつ GPU に載せて指標計算するのみ（renders/gt を全20枚 list 保持するが 800×800×3 と LPIPS バックボーンで余裕）
- **処理時間**: 完走が受け入れ基準であり、時間そのものは基準にしない。参考として 20 枚の評価（LPIPS-vgg/alex 推論を含む）は数十秒〜数分を想定
- **再現性**: 実行コマンド・GPU 指定・ログ保存先を design に明記し、`/clear` 後でも同一手順で再実行できること。LPIPS 重みはキャッシュ済みのため再現実行で再 DL されない
- **隔離性**: ログ等の一時生成物はリポジトリ外の作業パスに置く。成果物（results.json/per_view.json）はリポジトリ内 `output/` 配下に生成される（`.gitignore` 管理外・非コミット）

## 1.5 制約条件

- 4DGaussians 本体のコードは原則変更しない（CLAUDE.md 開発方針）。`metrics.py` / `lpipsPyTorch/` / `utils/*.py` を編集しない。互換問題で変更が必要になった場合は中断し、investigation.md に記録のうえユーザー承認を得る（feat-004 の前例）
- feat-005 成果物 `output/dnerf/bouncingballs/test/ours_20000/{renders,gt}/`（各20枚）が存在することを前提とする。欠損時は評価不可（`FileNotFoundError`）
- 実行は `.venv/bin/python` 経由とする（システム Python を使わない）
- `output/` は `.gitignore` 管理対象であることを前提に、成果物（results.json/per_view.json）はコミットしない
- 新規ライブラリは追加しない（`lpipsPyTorch` はリポジトリ内、`pytorch_msssim` は導入済み）。LPIPS 重みは調査時に DL 済み

## 1.6 優先順位

1. FR-001（評価コマンド実行と6指標算出）= Must
2. FR-002（results.json / per_view.json 生成）= Must
3. FR-003（指標値の妥当性）= Should（品質の健全性目安）
4. FR-004（実行ログの保存）= Should

MVP の範囲: FR-001・FR-002（評価が完走し6指標が算出され JSON が生成される）。FR-003 は「正常動作する環境」と言えるかの品質確認。FR-004 は再現性のための補強。
