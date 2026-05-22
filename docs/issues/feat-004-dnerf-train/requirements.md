# feat-004 要求仕様書: D-NeRF学習動作確認

## 1.1 プロジェクト概要

4DGaussians の動作確認の第2段階として、D-NeRF 合成シーン `bouncingballs` の学習を一通り完走させ、学習成果物が生成されることを確認する。本案件は feat-002 でビルドした CUDA 拡張（`diff_gaussian_rasterization` / `simple_knn`）と feat-003 で配置したデータ（`data/dnerf/bouncingballs/`）を、学習パイプライン上で実際に動作させる最初のステップである。

解決したい課題は「環境（CUDA 拡張・データ・依存）が学習を完走できる状態にあるか」を実証すること。レンダリング品質や学習時間の最適化は本案件の対象外（feat-005 レンダリング / feat-006 評価で扱う）。本案件の成果物 `output/dnerf/bouncingballs/` は feat-005 レンダリングの入力（`--model_path`）になる。コードの変更は行わない（CLAUDE.md 開発方針）。実行環境は本マシン（A100-SXM4-40GB、CUDA 拡張は CUDA 11.6 ビルド、torch 1.13.1+cu116、uv 管理の `.venv`）。

## 1.2 用語定義

- **学習（training）**: `train.py` を実行し、4D Gaussian と変形フィールドのパラメータを最適化する処理。機能設計書・コードでも同じ意味で用いる
- **coarse ステージ**: 静的な 3D Gaussian を先に最適化する第1段階（`coarse_iterations = 3000` 回）。`scene_reconstruction(..., stage="coarse", ...)` で実行
- **fine ステージ**: 変形フィールド（HexPlane + 変形 MLP）を含めて最適化する第2段階（`iterations = 20000` 回）。`scene_reconstruction(..., stage="fine", ...)` で実行
- **イテレーション（iteration）**: 最適化ループの1反復。coarse 3000 + fine 20000 = 計 23000 反復
- **成果物（artifact）**: 学習が出力するファイル群。本体の `Scene.save` が `output/{expname}/point_cloud/iteration_{N}/` 配下に書き出す `point_cloud.ply` と変形パラメータ（`deformation.pth` / `deformation_table.pth` / `deformation_accum.pth`）を指す
- **expname**: 出力先を決める実験名。`output/{expname}/` が成果物のルートになる（本案件では `dnerf/bouncingballs`）
- **CUDA 拡張**: feat-002 でビルドした `diff_gaussian_rasterization`（ラスタライザ）と `simple_knn`（最近傍）。学習のレンダリング・初期化で実際に呼ばれる
- **saving_iterations**: 成果物を保存するイテレーション集合。本構成では実際に保存が発生するのは fine ステージの 14000 と 20000 のみ（導出は design.md §1.4.3）。FR-003 の `iteration_14000` / `iteration_20000` はこれに対応する
- **test_iterations**: test 評価を実行するイテレーション集合。既定 `[3000, 7000, 14000]`（`train.py:406`）。各ステージ内のカウントで判定され、fine ステージの ITER 14000 が学習終盤の評価点となる（詳細は design.md §1.4.4）。FR-004 の `ITER 14000` はこれを指す

## 1.3 機能要求一覧

### FR-001: 学習コマンドの実行

- D-NeRF `bouncingballs` の学習を、README（140〜143行目、コマンドは143行目）と BACKLOG（feat-004 行）に記載のコマンドで実行する
  - コマンド: `python train.py -s data/dnerf/bouncingballs --port 6017 --expname "dnerf/bouncingballs" --configs arguments/dnerf/bouncingballs.py`
  - 実行は uv 管理の `.venv/bin/python` 経由とする
  - 単一 GPU を使用する（`CUDA_VISIBLE_DEVICES` で1枚に固定）
- 入力: 上記コマンド、`data/dnerf/bouncingballs/`（feat-003 配置済み）、`arguments/dnerf/bouncingballs.py`
- 出力: 標準出力に学習ログ（Blender データセット認識・point 数・進捗バー・評価値）、`output/dnerf/bouncingballs/` 配下の成果物
- 優先度: Must
- 受け入れ基準: コマンドが起動し、`Found transforms_train.json file, assuming Blender data set!` と `data loading done` が出力され、CUDA 拡張の import / 実行エラーなく学習ループに入ること

### FR-002: coarse + fine 2ステージの完走

- coarse（3000 反復）→ fine（20000 反復）の2ステージが、例外・異常終了なく最後まで進むこと
- 優先度: Must
- 受け入れ基準: プロセスが終了コード0で終了し、標準出力末尾に `Training complete.` が出力されること。途中で `loss is nan,end training` による再起動（`os.execv`、design.md §1.4.5 E4）が発生しないこと

### FR-003: 成果物の生成

- 学習完了後、最終イテレーション（fine 20000）の成果物が生成されていること
  - `output/dnerf/bouncingballs/point_cloud/iteration_20000/point_cloud.ply` が存在する
  - 同ディレクトリに `deformation.pth` / `deformation_table.pth` / `deformation_accum.pth` が存在する
  - `output/dnerf/bouncingballs/cfg_args`（実行時引数のスナップショット）が存在する
- 優先度: Must
- 受け入れ基準: 上記4ファイル（+iteration_20000 の3 pth）がすべて存在し、`point_cloud.ply` のサイズが 0 バイトでないこと。中間保存 `iteration_14000/` も生成されるが必須は 20000 とする

### FR-004: 学習が機能していることの確認（発散していないこと）

- 学習中の test 評価で PSNR が出力され、学習が破綻（NaN・発散）していないことを確認する
- 優先度: Should
- 受け入れ基準: 標準出力に `[ITER N] Evaluating test: L1 ... PSNR ...` 形式の評価ログが出力され、**fine ステージ**の ITER 14000（14000 は fine でのみ到達するため一意。評価はステージ内カウントで判定される。詳細は design.md §1.4.4）の test PSNR が 20 以上であること（D-NeRF 合成シーンは論文・公式実装で 30 以上に達するが、本案件は「環境が正しく学習を回せる」確認が目的のため下限を 20 に設定。値が極端に低い/NaN の場合は環境異常を疑う）

### FR-005: 実行ログの保存

- 学習は時間を要するためバックグラウンド実行とし、標準出力・標準エラーをログファイルに保存する
- ログ保存先はリポジトリ外の `/data/sakagawa/tmp/feat004-dnerf-train/train.log` とする（design.md §1.4.1・§1.8 と同一）
- 優先度: Should
- 受け入れ基準: 上記ログファイルに学習ログ全体が記録され、完走後にステージ進捗・評価値・`Training complete.` を後から確認できること

## 1.4 非機能要求

- **対応環境**: NVIDIA A100-SXM4-40GB を単一使用。CUDA 拡張は CUDA 11.6 ビルド・実行時は torch 同梱 CUDA 11.6 ランタイム。`.venv`（Python 3.10、torch 1.13.1+cu116）
- **信頼性・リカバリ方針**: NaN 発散（`train.py:220` 検出、`os.execv` で再起動）や VRAM 不足（OOM）で完走しない場合は**異常終了とみなし、自動リトライはしない**（os.execv 再起動が起きたらプロセスを停止する）。原因を `investigation.md` に記録する。失敗系の詳細は design.md §1.4.5（E1〜E6）を参照。成果物 `output/` は完走後の検証で確認し、未完走時は中間保存（あれば `iteration_14000`）の有無で進捗を切り分ける
- **VRAM**: 40GB に収まること。D-NeRF 合成シーンは Gaussian 数上限 360,000（`train.py:270`）と小規模で、40GB に対し十分余裕がある想定
- **処理時間**: 完走が受け入れ基準であり、時間そのものは基準にしない。参考値として A100 単一で coarse+fine 計 23000 反復が数分〜20分程度を想定（公式更新履歴の「D-NeRF 約8分」相当）
- **再現性**: 実行コマンド・GPU 指定・ログ保存先を design.md に明記し、`/clear` 後でも同一手順で再実行できること
- **隔離性**: ログ等の一時生成物はリポジトリ外の作業パスに置く。成果物 `output/` のみリポジトリ内に生成される

## 1.5 制約条件

- 4DGaussians 本体のコードは変更しない（CLAUDE.md 開発方針）。`train.py` / `arguments/dnerf/*.py` を編集しない
- feat-002 でビルド済みの CUDA 拡張（`submodules/depth-diff-gaussian-rasterization`, `submodules/simple-knn`）を使用する。再ビルドはしない
- feat-003 で配置済みの `data/dnerf/bouncingballs/` を使用する。データの再取得・再配置はしない
- 実行は `.venv/bin/python` 経由とする（システム Python を使わない）
- `--port 6017`（`network_gui` のソケット）を使用する。占有時は別ポートに変更する（学習自体は GUI 接続なしで進む）
- `output/` は `.gitignore` 管理対象かを確認し、成果物はコミットしない（feat-003 同様、データ・成果物は非コミット）

## 1.6 優先順位

1. FR-001（学習コマンドの実行）= Must
2. FR-002（2ステージ完走）= Must
3. FR-003（成果物の生成）= Must
4. FR-004（学習が機能している確認）= Should
5. FR-005（実行ログの保存）= Should

MVP の範囲: FR-001〜003（学習が完走し成果物が生成される）。FR-004・005 は学習の妥当性確認と再現性のための補強。
