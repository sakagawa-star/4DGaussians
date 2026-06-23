# feat-010 要求仕様書: DyNeRF動作確認（実シーン・多視点、cut_roasted_beef）

本書は `docs/REQUIREMENTS_STANDARD.md` に準拠する。

---

## 1.1 プロジェクト概要

- **何を作るのか**: DyNeRF（Neural 3D Video / Plenoptic Video）実シーン・多視点の1シーン（**cut_roasted_beef**）で、4DGaussians の「前処理（フレーム抽出 → COLMAP → ダウンサンプル）→ 学習（`train.py`）→ レンダリング（`render.py`）→ 評価（`metrics.py`）」が完走する状態を構築・確認する。
- **なぜ作るのか**: 最終目的「4DGS全体が動く環境」のうち、**実シーン3系統の2つ目（DyNeRF＝実・多視点）**の動作確認。D-NeRF（合成・単眼）・HyperNeRF（実・単眼）に続き、実写・多視点系統が動くことを実証する。**DyNeRF は事前生成点群が提供されないため、本案件で初めて `colmap.sh` を実走する**（feat-008 で導入した COLMAP の本格利用、および feat-009 で先送りした `colmap.sh` の GPU ハードコード対処を含む）。
- **誰が使うのか**: 本リポジトリで 4DGS を動かす開発者（複数人共用 GPU サーバー利用）。
- **どこで使うのか**: Ubuntu / A100-SXM4-40GB ×7（共用）/ uv 管理 `.venv`（Python 3.10, torch 1.13.1+cu116）/ ヘッドレス。

## 1.2 用語定義

ドキュメント内・機能設計書・コードで同じ用語を使う。

- **DyNeRF / Plenoptic Video / Neural 3D Video**: Meta（facebookresearch）の実写・多視点動的シーンデータセット（CVPR 2022）。複数の固定カメラ（リグ）で同一シーンを同時撮影した動画群。本案件は **cut_roasted_beef** を用いる。
- **`cam*.mp4`**: 各カメラの動画ファイル（`cam00.mp4`〜`camNN.mp4`）。DyNeRF ローダ（`Neural3D_NDC_Dataset.load_meta`）が `glob("cam*.mp4")` で `datadir` 直下から読む。`poses_bounds.npy` のカメラ数と**枚数一致が必須**（`assert len(videos)==poses_arr.shape[0]`、`scene/neural_3D_dataset_NDC.py:268`）。
- **`poses_bounds.npy`**: LLFF 形式のカメラポーズ＋near/far。形状 `(N_cams, 17)`（`[:, :-2]`=ポーズ3×5、`[:, -2:]`=near/far）。**DyNeRF 判定キー**（`scene/__init__.py:52`、存在で `dynerf` 経路へ分岐）。配布 zip に同梱される。
- **`points3D_downsample2.ply`**: DyNeRF 学習で**必須**の初期点群（`scene/dataset_readers.py:444`、`readdynerfInfo` が `fetchPly` で読む）。COLMAP の `fused.ply` を `downsample_point.py` で ≤40,000 点に削減したもの。
- **eval_index=0**: DyNeRF ローダが固定で用いる **test カメラ index**（`readdynerfInfo` が `eval_index=0` を渡す）。**cam00（ソート先頭）が test、残り全カメラが train**（`scene/neural_3D_dataset_NDC.py:292-294, 313-318`）。
- **countss=300**: 各カメラから抽出・使用するフレーム数の上限（`scene/neural_3D_dataset_NDC.py:310`）。時刻は `idx/300` で [0,1) に正規化。
- **preprocess_dynerf.py**: `Neural3D_NDC_Dataset` を train/test の両 split でインスタンス化し、`load_images_path` 経由で全カメラの mp4 を `camXX/images/0000.png〜0299.png` に cv2 で抽出するラッパー（`scripts/preprocess_dynerf.py`）。
- **colmap.sh llff**: `scripts/llff2colmap.py`（`poses_bounds.npy`→COLMAP テキスト変換）→ COLMAP（feature_extractor〜stereo_fusion）を一括実行するシェルスクリプト。**各カメラの先頭フレーム `0000.png` のみ**（≈N枚）で SfM+MVS を行い `fused.ply` を生成する。**スクリプト内部は bare `python`（`colmap.sh:8` の `python scripts/llff2colmap.py`、`colmap.sh:16` の `python database.py`）を呼ぶ。本環境に bare `python` は無く `python3` のみのため、実行時に `PATH` 先頭へ `.venv/bin` を前置して `python` を `.venv/bin/python` に解決する（FR-003）。**
- **bare python 不在 / PATH 前置**: 本環境は `which python` が非0（`python` コマンドが無い）で `/usr/bin/python3` のみ。`PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH"` を前置すると bare `python` が `.venv/bin/python`（uv 管理 cpython-3.10）に解決される。`.venv/bin/python` を明示できない既存スクリプト（`colmap.sh`）の bare `python` 呼び出しはこの方法で `.venv` に向ける。
- **coarse / fine 段階**: 4DGS 学習の2段階。cut_roasted_beef では coarse=3000・fine=14000 反復（`arguments/dynerf/default.py`）。
- **本体変更点（colmap.sh）**: 本案件で唯一許容する 4DGS 本体改変。`colmap.sh:5` の `export CUDA_VISIBLE_DEVICES=0` を引数化し、任意GPU選択を可能にする（FR-003・ADR-3）。

## 1.3 機能要求一覧

### FR-001: cut_roasted_beef データセット（動画・ポーズ）の取得と配置

- **概要**: Neural_3D_Video v1.0 リリースの `cut_roasted_beef.zip`（約1.06GB）を取得・展開し、4DGS が読める構造で `data/dynerf/cut_roasted_beef/` へ配置する。
- **入力**: `https://github.com/facebookresearch/Neural_3D_Video/releases/download/v1.0/cut_roasted_beef.zip`（curl/wget、`-L` でリダイレクト追従）。
- **出力**: `data/dynerf/cut_roasted_beef/` 直下に以下が揃った状態。
  - `cam00.mp4`〜`camNN.mp4`（各カメラの動画、N台）
  - `poses_bounds.npy`（カメラポーズ＋near/far、形状 `(N,17)`）
- **受け入れ基準**:
  - `data/dynerf/cut_roasted_beef/` 直下に `cam*.mp4` が **2枚以上**存在し、`poses_bounds.npy` が存在する。
  - `.venv/bin/python` で `np.load("poses_bounds.npy")` が形状 `(N,17)` を返し、**`N == len(glob("cam*.mp4"))`** が成立する（ローダの assert を事前検証）。
  - `data/dynerf/` はリポジトリにコミットしない（`data/` は .gitignore 管理外＝git 未追跡で運用、feat-003/009 と同方針）。

### FR-002: フレーム抽出（preprocess_dynerf.py）

- **概要**: `preprocess_dynerf.py` で全カメラの mp4 から先頭300フレームを PNG 抽出し、`camXX/images/0000.png〜0299.png` を生成する。
- **入力**: `.venv/bin/python scripts/preprocess_dynerf.py --datadir data/dynerf/cut_roasted_beef`（本スクリプトは `Neural3D_NDC_Dataset` を直接使い `readdynerfInfo`/`plot_camera_orientations` を経由しない。後述のとおり DyNeRF 経路は matplotlib を呼ばないため MPL 環境変数は不要）
- **出力**: `data/dynerf/cut_roasted_beef/camXX/images/0000.png〜0299.png`（全 N カメラ分）。
- **受け入れ基準**:
  - 終了コード 0 で完走する。
  - **全 N カメラ**について `camXX/images/` が生成され、各ディレクトリに PNG が **300枚**ある（`ls camXX/images/*.png | wc -l == 300`）。
  - 抽出画像の解像度が 1352×1014（ローダ既定 `img_wh`）であること（1枚を `PIL.Image.open` で確認）。

### FR-003: COLMAP 点群生成（colmap.sh llff）＋ colmap.sh の GPU 引数化改変

- **概要**: `colmap.sh ... llff` を実行し、各カメラ先頭フレームから COLMAP で `fused.ply` を生成する。実行に先立ち、共用サーバーで任意GPUを選べるよう **`colmap.sh:5` を引数化改変**する（本体変更点として記録）。
- **入力**:
  - 改変: `colmap.sh:5` の `export CUDA_VISIBLE_DEVICES=0` を `export CUDA_VISIBLE_DEVICES=${3:-0}` に変更（第3引数で GPU 指定。未指定時は従来どおり 0）。
  - 実行: `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh data/dynerf/cut_roasted_beef llff <空きN>`（`colmap.sh` 内部の bare `python`〔`:8`・`:16`〕を `.venv/bin/python` に解決するため `PATH` 先頭へ `.venv/bin` を前置。`bash colmap.sh` の cwd はリポジトリルート `/data/sakagawa/4DGaussians` とする〔`scripts/llff2colmap.py`・`database.py` が相対参照のため〕）
- **出力**: `data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply`（および `colmap/sparse/0/`・`colmap/database.db`・`image_colmap/`・`sparse_/`）。
- **受け入れ基準**:
  - 実行前に `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" command -v python` が `/data/sakagawa/4DGaussians/.venv/bin/python` を指すことを確認する（bare `python` 不在による即死を未然に防ぐ）。
  - 終了コード 0 で完走し、`colmap/dense/workspace/fused.ply` が生成される。
  - `fused.ply` が `fetchPly` 相当（`PlyData.read`）で読めて**点数 > 0**（空・破損は不合格）。
  - 改変後の `colmap.sh` が、第3引数 N を `CUDA_VISIBLE_DEVICES` に反映し、**指定した物理GPU N のみに COLMAP の負荷が乗る**（`nvidia-smi` で確認）。引数省略時は GPU0 を使う（後方互換）。
  - `git diff colmap.sh` の差分が **5行目の1箇所のみ**であること（最小改変）。

### FR-004: 点群ダウンサンプル（downsample_point.py）

- **概要**: `downsample_point.py` で `fused.ply` を ≤40,000 点に voxel ダウンサンプルし、`points3D_downsample2.ply` を生成する。
- **入力**: `.venv/bin/python scripts/downsample_point.py data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply data/dynerf/cut_roasted_beef/points3D_downsample2.ply`
- **出力**: `data/dynerf/cut_roasted_beef/points3D_downsample2.ply`。
- **受け入れ基準**:
  - 終了コード 0 で完走し、`points3D_downsample2.ply` が生成される。
  - `fetchPly` 相当で読めて **0 < 点数 ≤ 40,000**（`downsample_point.py` は 40,000 点以下になるまで voxel を粗くするループ）。

### FR-005: DyNeRF（cut_roasted_beef）学習の完走

- **概要**: `train.py` を cut_roasted_beef の config で実行し、coarse 3000 + fine 14000 反復が完走して学習済み点群が生成されることを確認する。マルチGPU運用ルールに従い空きGPU・空きポートを選ぶ。
- **入力**: コマンド（train は `scene.gaussian_model`→`scene.regulation:5` および `utils.scene_utils:4` 経由で `matplotlib.pyplot` を **top-level import** するため、書込可能な `MPLCONFIGDIR` と `MPLBACKEND=Agg` が必要。reader〔`readdynerfInfo`〕は `plot_camera_orientations` を呼ばず `output.png` は生成されないが、import 時のフォントキャッシュ書込に MPL 設定が要る）
  `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<空きN> .venv/bin/python train.py -s data/dynerf/cut_roasted_beef --port <空きP> --expname "dynerf/cut_roasted_beef" --configs arguments/dynerf/cut_roasted_beef.py`
- **出力**: `output/dynerf/cut_roasted_beef/point_cloud/iteration_14000/`（`point_cloud.ply` と deformation の重み `.pth` 群）、`output/dynerf/cut_roasted_beef/cfg_args`。
- **受け入れ基準**:
  - プロセスが終了コード 0 で完走する（クラッシュしない）。
  - matplotlib の top-level import（`scene.regulation`/`utils.scene_utils`）でクラッシュしない。事前確認: `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig .venv/bin/python -c "import matplotlib.pyplot; print(matplotlib.get_backend().lower())"` が `agg` を返す（書込可能キャッシュで import 成功。MPL 未設定だと「Matplotlib requires access to a writable cache directory」でクラッシュしうる）。
  - `output/dynerf/cut_roasted_beef/point_cloud/iteration_14000/point_cloud.ply` が生成される（非空）。
  - 指定した物理GPU N のみに負荷が乗る（`nvidia-smi` で確認）。
  - coarse 段階の保存物（`iteration_3000` 等）は生成されない想定（`save_iterations` 最小14000に coarse 3000 が未到達のため。生成有無が設計の根拠どおりであることを確認）。

### FR-006: レンダリングの完走（test + video セット）

- **概要**: `render.py` を `--skip_train` で実行し、test セット（評価用）と video セット（视認用）の画像・動画が生成されることを確認する。
- **入力**: コマンド（render も train と同様 `scene.gaussian_model`→`matplotlib.pyplot` を top-level import するため MPL 設定が必要。`output.png` は生成されない〔plot 非呼び出し〕が import に MPL が要る）
  `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<FR-005と同一N> .venv/bin/python render.py --model_path output/dynerf/cut_roasted_beef --skip_train --configs arguments/dynerf/cut_roasted_beef.py`
- **出力**:
  - `output/dynerf/cut_roasted_beef/test/ours_14000/{renders,gt}/`（各 PNG）
  - `output/dynerf/cut_roasted_beef/video/ours_14000/renders/` と `video_rgb.mp4`
- **受け入れ基準**:
  - 終了コード 0 で完走する。
  - `test/ours_14000/renders/` と `test/ours_14000/gt/` の PNG 数が**同数かつ 1 枚以上**で、**両ディレクトリのファイル名集合が完全一致**する（空評価・対応欠落を排除）。test は cam00 の300フレーム想定（renders/gt 各300枚）。
  - iteration は未指定（`--iteration -1`）で最大の 14000 が自動選択される。

### FR-007: 評価の完走（test セット6指標）

- **概要**: `metrics.py` を実行し、test セットに対して PSNR/SSIM/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM が算出されることを確認する。
- **入力**: コマンド
  `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<FR-005と同一N> .venv/bin/python metrics.py --model_path output/dynerf/cut_roasted_beef/`
- **出力**: `output/dynerf/cut_roasted_beef/{results,per_view}.json`（method=`ours_14000` の6指標）。
- **受け入れ基準**:
  - 終了コード 0 で完走し、6指標すべてが数値出力される。
  - `results.json` に `ours_14000` の6指標が記録され、**6指標すべてが有限値（`math.isfinite` が True。NaN/Inf でない）**である。
  - `per_view.json` の `ours_14000` の各指標の件数が、`test/ours_14000/renders/` の PNG 数と一致する。
  - **健全性チェック（参考値）**: 4DGS 論文（CVPR 2024, arXiv:2310.08528 Table 4）の cut_roasted_beef PSNR **33.85** を基準に、**PSNR ≥ 30** を目安とする。これは健全性の目安であり、**合否は終了コードと6指標の生成（クラッシュせず有限値が出ること）で判定**する（バグがあれば PSNR は大きく低下する。D-NeRF/HyperNeRF では論文値に近接した実績）。

### FR-008: 文書化・依存記録・本体変更点の記録

- **概要**: DyNeRF 動作確認手順（データ取得〜評価）・`colmap.sh` 改変を再現可能な形で文書化し、依存記録・運用記録を更新する。
- **入力**: FR-001〜007 の実施結果。
- **出力**: `design.md` 手順、`CLAUDE.md`（①データセット節の DyNeRF を「動作確認済み・手順」に更新、②「オリジナルコードの変更点」に `colmap.sh` 改変を追記、③マルチGPU運用ルールに colmap.sh の使い方を追記）、`docs/TECH_STACK.md`（DyNeRF データ取得手順）、`docs/BACKLOG.md`（feat-010 Closed）。
- **受け入れ基準**:
  - `/clear` 後でも本書＋設計書のみで cut_roasted_beef のデータ取得〜評価を再現できる（DL URL・展開/配置パス・前処理3コマンド・train/render/metrics コマンド・GPU/ポート選択手順・colmap.sh 改変内容を明記）。
  - `CLAUDE.md` の「オリジナルコードの変更点」に `colmap.sh:5` の改変（理由・差分）が記録される。
  - 新規ライブラリ導入が無いことを確認する（cv2/imageio-ffmpeg/open3d/ffmpeg/colmap は導入済み）。**新規導入が無ければ `requirements.lock.txt` の再生成は不要**（依存変化なし。導入が発生した場合のみ `uv pip freeze` で再生成）。

## 1.4 非機能要求

- **対応環境**: Ubuntu / A100（sm_80）/ ヘッドレス（X ディスプレイ非前提）。**train/render は `scene.gaussian_model`→`scene.regulation:5`（`import matplotlib.pyplot`）および `utils.scene_utils:4` 経由で matplotlib を top-level import する**ため、書込可能な `MPLCONFIGDIR`（scratch 配下）と `MPLBACKEND=Agg` を固定する（import 時のフォントキャッシュ書込にディレクトリが要り、ヘッドレスで GUI backend を避けるため。MPL 未設定だと「Matplotlib requires access to a writable cache directory」で import がクラッシュしうる）。これは reader 内の `plot_camera_orientations`（`scene/dataset_readers.py:510`、**HyperNeRF 経路 `readHyperDataInfos:390` 専用**で DyNeRF では非呼び出し＝`output.png` は生成されない）とは独立した、**import 段階での要件**。metrics.py は matplotlib も scene/gaussian_model も import しない（`import` 一覧で確認）ため MPL 不要。preprocess_dynerf.py も `Neural3D_NDC_Dataset` のみ import（matplotlib 非ロード）のため不要。
- **権限**: sudo 不可。ユーザー権限（`~/`・`/data/sakagawa/` 配下）で完結する。本案件は apt 等のシステム導入を要しない（COLMAP は feat-008 で導入済み）。
- **処理時間**:
  - フレーム抽出（FR-002）: 全カメラ×300フレームで数分〜十数分の目安。
  - COLMAP（FR-003）: **各カメラ先頭1フレームのみ（≈N枚）で SfM+MVS** を行うため、HyperNeRF broom（約200枚で数時間）より大幅に軽く、数分〜十数分の目安。
  - 学習（FR-005）: README 目安で約20分以下（A100×1）。レンダ・評価は数分以内を想定。
  - いずれも目安であり、合否は終了コードと生成物で判定する。
- **信頼性**: 学習出力は `output/dynerf/cut_roasted_beef/` に置く。再実行時は出力先を削除してから再実行できること。データ（`data/dynerf/`）はリポジトリにコミットしない。
- **マルチGPU**: 前処理（COLMAP）・学習・レンダ・評価はいずれも 1 プロセス = 1 GPU。COLMAP は改変後 `colmap.sh ... llff N`、train/render/metrics は `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N` を付け、同一ジョブで同一 N を使う。実行前に空きGPU（`nvidia-smi`）と空きポート（`ss -ltn | grep :<port>`、train のみ）を確認する（`CLAUDE.md` マルチGPU運用ルール準拠）。

## 1.5 制約条件

- **使用必須ライブラリ**（すべて導入済み・新規導入なし）:
  - 既存 `.venv`（torch 1.13.1+cu116 / mmcv 1.6.0 / numpy 1.23.5 等）と CUDA 拡張（feat-002 ビルド済み）。
  - **opencv-python 4.13.0**（`cv2.VideoCapture` で mp4 フレーム抽出）。
  - **imageio 2.37.3 + imageio-ffmpeg 0.6.0** ＋ システム ffmpeg 4.4.2（動画書き出し）。
  - **open3d 0.19.0**（`downsample_point.py` の voxel ダウンサンプル）。
  - **COLMAP**（feat-008、vcpkg ソースビルド、`~/.local/bin/colmap`、CUDA有効）。
  - matplotlib（導入済み。ただし DyNeRF 経路では `plot_camera_orientations` は呼ばれず、機能上は不使用）。
- **使用禁止 / 回避**:
  - **`uv sync` / `uv pip sync` は使わない**（追加的 `uv pip install` のみ）。`pyproject.toml` は作らない。
  - **4DGaussians 本体コードの改変は `colmap.sh:5` の1行（GPU 引数化）に限定する**。それ以外の本体改変は行わない。改変は「オリジナルコードの変更点」に記録する（FR-008）。
  - numpy の版を 1.23.5 から動かさない。
- **ネットワーク**: GitHub Release（`cut_roasted_beef.zip`）の取得にインターネットアクセスを用いる。
- **ディスク**: `cut_roasted_beef.zip` 約1.06GB ＋展開した mp4 群 ＋抽出画像（N×300枚）＋ COLMAP 中間生成物（dense workspace）で計数GB。`/data`（15TB 空き）に置く。`data/dynerf/` はコミットしない。

## 1.6 優先順位（MoSCoW）

| 要求 | 優先度 | 備考 |
|------|--------|------|
| FR-001 データ取得・配置 | **Must** | 前処理の前提 |
| FR-002 フレーム抽出 | **Must** | COLMAP・学習の前提 |
| FR-003 COLMAP 点群生成＋colmap.sh 改変 | **Must** | 学習用点群の生成。GPU 引数化が本案件の主要技術論点 |
| FR-004 ダウンサンプル | **Must** | 学習に必須の ply |
| FR-005 学習完走 | **Must** | 判定基準（train.py 完走）に対応 |
| FR-006 レンダリング完走 | **Must** | 判定基準（render.py 完走）に対応 |
| FR-007 評価完走 | **Must** | 判定基準（metrics.py 完走）に対応 |
| FR-008 文書化・本体変更点記録 | **Must** | 再現性・運用・本体改変の追跡のため |

- **MVP の範囲**: FR-001〜FR-008 すべて。これらの達成をもって BACKLOG の判定基準（前処理3段完走 + train→render→metrics 完走）を満たす。
