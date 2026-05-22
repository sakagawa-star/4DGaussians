# feat-005 要求仕様書: D-NeRFレンダリング動作確認

## 1.1 プロジェクト概要

4DGaussians の動作確認の第3段階（環境構築 Phase 4）として、feat-004 で学習した D-NeRF 合成シーン `bouncingballs` の学習済みモデルを `render.py` で読み込み、レンダリング画像・動画が生成されることを確認する。本案件は feat-002（CUDA 拡張ビルド）・feat-003（データ配置）・feat-004（学習・成果物生成）の積み上げの上に立ち、学習済み 4D Gaussian と変形フィールドが推論経路で実際に動作することを実証する。

解決したい課題は「学習済みモデル（`output/dnerf/bouncingballs/`）から、任意視点・任意時刻のレンダリングが破綻なく出力されるか」を確認すること。レンダリング品質の定量評価（PSNR/SSIM/LPIPS 等）は本案件の対象外（feat-006 評価で扱う）。本案件の成果物 `output/dnerf/bouncingballs/test/ours_20000/{renders,gt}/` は **feat-006 評価の入力**になる。コードの変更は原則行わない（CLAUDE.md 開発方針）。実行環境は本マシン（A100-SXM4-40GB、CUDA 拡張は CUDA 11.6 ビルド、torch 1.13.1+cu116、uv 管理の `.venv`）。

## 1.2 用語定義

- **レンダリング（rendering）**: 学習済みの 4D Gaussian と変形フィールドを用い、指定カメラ視点・時刻から画像を合成する推論処理。`render.py` の `render_set` が `gaussian_renderer.render` を各カメラに対して呼ぶ
- **学習済みモデル（trained model）**: feat-004 が出力した `output/dnerf/bouncingballs/point_cloud/iteration_20000/` 配下の `point_cloud.ply`（Gaussian）と `deformation.pth` / `deformation_table.pth` / `deformation_accum.pth`（変形フィールド）。`Scene` が `load_iteration` 指定でこれをロードする
- **iteration（ロード対象）**: ロードする学習済みモデルの反復番号。`render.py` の既定 `--iteration -1` のとき `searchForMaxIteration` が `point_cloud/` 配下の最大値（本構成では **20000**）を選ぶ（導出は design.md §1.4.2）
- **test セット**: `transforms_test.json` 由来の 20 カメラ（学習で未使用の評価用視点・時刻）。各カメラに対応する正解画像（gt）を持つ
- **video セット**: `generateCamerasFromTransforms` がプログラム生成する 160 フレームの円軌道（spherical）カメラ列。正解画像を持たない（新規視点の動画用）
- **renders / gt**: `render_set` が書き出すサブディレクトリ。`renders/` は推論結果、`gt/` は正解画像（test/train のみ。video には gt を出力しない）
- **video_rgb.mp4**: `render_set` が `imageio.mimwrite` で書き出す、推論結果を 30fps で連結した動画
- **CUDA 拡張**: feat-002 でビルドした `diff_gaussian_rasterization`（ラスタライザ）と `simple_knn`。レンダリング推論で実際に呼ばれる

## 1.3 機能要求一覧

### FR-001: レンダリングコマンドの実行とモデルロード

- D-NeRF `bouncingballs` のレンダリングを、README（公式 Rendering 手順）と BACKLOG（feat-005 行）に記載のコマンドで実行する
  - コマンド: `python render.py --model_path "output/dnerf/bouncingballs/" --skip_train --configs arguments/dnerf/bouncingballs.py`
  - 実行は uv 管理の `.venv/bin/python` 経由とする
  - 単一 GPU を使用する（`CUDA_VISIBLE_DEVICES` で1枚に固定）
- 入力: 上記コマンド、feat-004 成果物 `output/dnerf/bouncingballs/`（`cfg_args` ＋ `point_cloud/iteration_{14000,20000}/`）、`arguments/dnerf/bouncingballs.py`、`data/dnerf/bouncingballs/`（カメラ情報の読み込み元）
- 出力: 標準出力にロードログ、`output/dnerf/bouncingballs/` 配下のレンダリング成果物
- 優先度: Must
- 受け入れ基準: コマンドが起動し、`Looking for config file in .../cfg_args` と `Loading trained model at iteration 20000`、`Found transforms_train.json file, assuming Blender data set!` が出力され、CUDA 拡張の import / 実行エラー（design.md §1.4.6 E2）なくレンダリングループに入ること。あわせて、カメラ読み込み時に Pillow/numpy 由来の型エラー（feat-004 で修正した `dataset_readers.py:287` を通る train/test の読み込み経路を含む）が発生しないこと（design.md §1.4.6 E3）

### FR-002: test セットのレンダリング（renders + gt）

- `--skip_train` 指定のため train はスキップし、test セット（20 カメラ）のレンダリングが完走すること
  - `output/dnerf/bouncingballs/test/ours_20000/renders/` に推論画像（PNG）が生成される
  - `output/dnerf/bouncingballs/test/ours_20000/gt/` に正解画像（PNG）が生成される
  - renders と gt の枚数が一致し、いずれも 20 枚（test カメラ数）であること
- 本成果物は feat-006 評価の入力となる（`metrics.py` が `test/ours_20000/{renders,gt}` を比較する）
- 優先度: Must
- 受け入れ基準: 上記2ディレクトリに同数（20 枚）の非空 PNG が生成されること

### FR-003: video セットのレンダリング

- video セット（円軌道 160 フレーム）のレンダリングが完走すること
  - `output/dnerf/bouncingballs/video/ours_20000/renders/` に推論画像（PNG）160 枚が生成される
  - video セットは正解画像を持たないため `gt/` に PNG は生成されない（空ディレクトリ自体は `render_set` の `makedirs` で作られる場合がある）
- 優先度: Must
- 受け入れ基準: `video/ours_20000/renders/` に非空 PNG が 160 枚生成されること

### FR-004: 動画ファイル（mp4）の生成

- 各レンダリングセットについて、推論画像を連結した動画 `video_rgb.mp4` が生成されること
  - `output/dnerf/bouncingballs/test/ours_20000/video_rgb.mp4`
  - `output/dnerf/bouncingballs/video/ours_20000/video_rgb.mp4`
- 優先度: Should
- 受け入れ基準: 上記2つの mp4 が存在し、サイズが 0 バイトでないこと（mp4 書き出しは `imageio` + `imageio_ffmpeg` 0.6.0 を使用）

### FR-005: 実行ログの保存

- レンダリングはバックグラウンド実行とし、標準出力・標準エラーをログファイルに保存する
- ログ保存先はリポジトリ外の `/data/sakagawa/tmp/feat005-dnerf-render/render.log` とする（design.md §1.4.1・§1.8 と同一）
- 優先度: Should
- 受け入れ基準: 上記ログファイルにレンダリングログ全体（ロード・進捗・FPS）が記録され、完走後に内容を後から確認できること

## 1.4 非機能要求

- **対応環境**: NVIDIA A100-SXM4-40GB を単一使用。CUDA 拡張は CUDA 11.6 ビルド・実行時は torch 同梱 CUDA 11.6 ランタイム。`.venv`（Python 3.10、torch 1.13.1+cu116）
- **信頼性**: ロード失敗（成果物欠損・iteration 不一致）、CUDA 拡張の import/実行エラー、mp4 書き出し失敗（ffmpeg 不在）で完走しない場合は**異常終了とみなし、自動リトライはしない**。原因を `investigation.md` に記録する。失敗系の詳細は design.md §1.4.6（E1〜E6）を参照
- **VRAM**: 40GB に収まること。推論は学習よりメモリ要求が小さく、学習が完走している以上余裕がある想定
- **処理時間**: 完走が受け入れ基準であり、時間そのものは基準にしない。参考として test 20 + video 160 = 180 フレームのレンダリング（A100 単一で数十秒〜数分を想定）
- **再現性**: 実行コマンド・GPU 指定・ログ保存先を design.md に明記し、`/clear` 後でも同一手順で再実行できること
- **隔離性**: ログ等の一時生成物はリポジトリ外の作業パスに置く。成果物はリポジトリ内 `output/` に生成される（`.gitignore` 管理外・非コミット）

## 1.5 制約条件

- 4DGaussians 本体のコードは原則変更しない（CLAUDE.md 開発方針）。`render.py` / `arguments/dnerf/*.py` を編集しない。互換問題で変更が必要になった場合は中断し、investigation.md に記録のうえユーザー承認を得る（feat-004 の前例）
- feat-002 でビルド済みの CUDA 拡張を使用する。再ビルドはしない
- feat-004 成果物 `output/dnerf/bouncingballs/`（特に `cfg_args` と `point_cloud/iteration_20000/`）が存在することを前提とする。欠損時はレンダリング不可
- 実行は `.venv/bin/python` 経由とする（システム Python を使わない）
- `output/` は `.gitignore` 管理対象であることを前提に、成果物はコミットしない
- 新規ライブラリは追加しない（`imageio` / `imageio_ffmpeg` は既に導入済み）

## 1.6 優先順位

1. FR-001（コマンド実行とモデルロード）= Must
2. FR-002（test セットのレンダリング）= Must
3. FR-003（video セットのレンダリング）= Must
4. FR-004（動画 mp4 の生成）= Should
5. FR-005（実行ログの保存）= Should

MVP の範囲: FR-001〜003（モデルがロードされ test・video の画像が生成される）。特に FR-002 は feat-006 評価の前提となるため重要。FR-004・005 は動画出力の確認と再現性のための補強。
