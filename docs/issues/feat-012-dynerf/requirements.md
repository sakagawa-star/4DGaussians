# feat-012 要求仕様書: DyNeRF動作確認（実シーン・多視点、cut_roasted_beef）・再開

本書は `docs/REQUIREMENTS_STANDARD.md` に準拠する。中止した **feat-010 の要求仕様を継承**し、**前処理を feat-011 成果物流用・本体改変ゼロ**へ更新した再開版である。

---

## 1.1 プロジェクト概要

- **何を作るのか**: DyNeRF（Neural 3D Video / Plenoptic Video）実シーン・多視点の1シーン（**cut_roasted_beef**）で、4DGaussians の「**学習（`train.py`）→ レンダリング（`render.py`）→ 評価（`metrics.py`）**」が完走する状態を確認する。前処理（フレーム抽出 → COLMAP → ダウンサンプル）は **feat-011 で実走済み・成果物が `data/dynerf/cut_roasted_beef/` に存在する**ため、本案件はそれを**流用**する。
- **なぜ作るのか**: 最終目的「4DGS全体が動く環境」のうち、**実シーン3系統の2つ目（DyNeRF＝実・多視点）**の動作確認。D-NeRF（合成・単眼〔済〕）・HyperNeRF（実・単眼〔済〕）に続き、実写・多視点系統が train→render→metrics まで動くことを実証する。
- **誰が使うのか**: 本リポジトリで 4DGS を動かす開発者（複数人共用 GPU サーバー利用）。
- **どこで使うのか**: Ubuntu / A100-SXM4-40GB ×7（共用）/ uv 管理 `.venv`（Python 3.10, torch 1.13.1+cu116）/ ヘッドレス。

### feat-010 からの変更点（本案件の前提・スコープ確定）

1. **前処理は feat-011 成果物を流用**（2026-06-24 ユーザー決定）。feat-010 の FR-001〜004（データ取得・フレーム抽出・COLMAP・ダウンサンプル）は feat-011 で実走・実証済み（`fused.ply` 387,496点・Mean reprojection error 0.852px・20/20登録 → `points3D_downsample2.ply` 37,361点）。本案件は再実走せず、**成果物の実在・整合の確認（FR-001）**に留め、学習以降（FR-002〜004）に集中する。
2. **本体改変はゼロ**（2026-06-24 ユーザー決定）。feat-010 で予定した `colmap.sh:5` の GPU 引数化（feat-010 FR-003 / ADR-3）は**行わない**。前処理を再実走しないため引数化しても動作確認できないからである。feat-009（HyperNeRF）と同じく **4DGS 本体コードを 1 行も改変しない**。
   - CLAUDE.md「COLMAP の使い分け」節に残る「任意 GPU 選択（colmap.sh:5 引数化）は feat-012 スコープ」の記述は、本決定により**将来 COLMAP 再実走が必要な案件（feat-013 等）で検討**へ更新する（FR-005）。
3. **COLMAP は再実走しない**が、万一点群の再生成が必要になった場合は feat-011 で併設した **3.11.1** を使う（3.12.6 は `colmap.sh ... llff` の `point_triangulator` で rig 非互換クラッシュ。CLAUDE.md「COLMAP の使い分け」節）。

## 1.2 用語定義

ドキュメント内・機能設計書・コードで同じ用語を使う。

- **DyNeRF / Plenoptic Video / Neural 3D Video**: Meta（facebookresearch）の実写・多視点動的シーンデータセット（CVPR 2022）。複数の固定カメラで同一シーンを同時撮影した動画群。本案件は **cut_roasted_beef**（20カメラ、`cam04` 欠番）を用いる。
- **`cam*.mp4`**: 各カメラの動画ファイル。DyNeRF ローダ（`Neural3D_NDC_Dataset.load_meta`）が `glob("cam*.mp4")` で `datadir` 直下から読む。`poses_bounds.npy` のカメラ数と**枚数一致が必須**（`assert len(videos)==poses_arr.shape[0]`、`scene/neural_3D_dataset_NDC.py:268`）。**本案件では既配置済み**（20本、`cam04` 欠番）。
- **`poses_bounds.npy`**: LLFF 形式のカメラポーズ＋near/far。形状 `(N_cams, 17)`。**DyNeRF 判定キー**（`scene/__init__.py:52`、存在で `dynerf` 経路へ分岐）。**本案件では既配置済み**（形状 `(20,17)`）。
- **`points3D_downsample2.ply`**: DyNeRF 学習で**必須**の初期点群（`scene/dataset_readers.py:444`、`readdynerfInfo` が `fetchPly` で読む）。COLMAP の `fused.ply` を `downsample_point.py` で ≤40,000 点に削減したもの。**本案件では feat-011 生成物（37,361点）を流用**。
- **`camXX/images/0000.png〜0299.png`**: `preprocess_dynerf.py` が各 mp4 から抽出した先頭300フレーム（解像度 1352×1014）。**本案件では feat-011 で抽出済み**（全20カメラ各300枚）。
- **eval_index=0**: DyNeRF ローダが固定で用いる **test カメラ index**（`readdynerfInfo:453,462` が `eval_index=0` を渡す）。**cam00（ソート先頭）が test、残り全カメラ（19台）が train**（`scene/neural_3D_dataset_NDC.py`）。config・引数からは変えられない（コード仕様）。
- **countss=300 / maxtime=300**: 各カメラから使用するフレーム数（`neural_3D_dataset_NDC.py`、`readdynerfInfo:480` の `maxtime=300`）。時刻は `idx/300` で [0,1) に正規化。
- **coarse / fine 段階**: 4DGS 学習の2段階。cut_roasted_beef では coarse=3000・fine=14000 反復（`arguments/dynerf/default.py`、`cut_roasted_beef.py` は `batch_size` のみ 2 に上書き）。
- **MPL 環境変数**: `MPLBACKEND=Agg`・`MPLCONFIGDIR`・`TMPDIR`。`train`/`render` は `scene.gaussian_model`→`scene/regulation.py:5`（`import matplotlib.pyplot`）・`utils/scene_utils.py:4` で matplotlib を **top-level import** するため、書込可能な `MPLCONFIGDIR` と `MPLBACKEND=Agg` が必要（フォントキャッシュ書込／ヘッドレス）。**`metrics` は不要**（gaussian_model/matplotlib 非 import）。DyNeRF 経路は `plot_camera_orientations` を呼ばない（HyperNeRF 専用）ため **`output.png` は生成されない**（feat-010 ADR-6 で実証済みの知見を継承）。

## 1.3 機能要求一覧

### FR-001: 前処理成果物の前提確認（feat-011 流用）

- **概要**: 学習開始に必要な前処理成果物が `data/dynerf/cut_roasted_beef/` に揃っていることを確認する。**前処理3段（`preprocess_dynerf.py` / `colmap.sh ... llff`〔3.11.1〕/ `downsample_point.py`）は feat-011 で完走実証済み**であり、本案件では再実走せず成果物の実在・整合を検証する。
- **入力**: feat-011 が生成した `data/dynerf/cut_roasted_beef/` 配下の成果物。
- **出力**: なし（確認のみ）。
- **受け入れ基準**:
  - `cam*.mp4` が **20本**存在し、`poses_bounds.npy` が形状 `(20,17)` で、**`20 == len(glob("cam*.mp4"))`** が成立する（ローダ `neural_3D_dataset_NDC.py:268` の assert を事前検証）。
  - 全20カメラ（cam00〜cam20、cam04欠番）について `camXX/images/*.png` が **各300枚**存在する。
  - `points3D_downsample2.ply` が **`fetchPly`（`scene/dataset_readers.py:124-130`）で実際に読める**こと。`fetchPly` は `x/y/z`・`red/green/blue`・`nx/ny/nz` の **9フィールドを要求**するため、点数だけでなく**この9フィールドが揃っていることを確認**する（色/法線が欠けると点数が通っても FR-002 の学習読込で `KeyError`）。実測: 9フィールド一式あり・**37,361点**（0 < n ≤ 40,000）。
  - **判定基準（BACKLOG「前処理3段完走」）の扱い**: 前処理3段の完走は **feat-011 の手動テスト合格で実証済み**（`docs/issues/feat-011-colmap-3.11/`）。本案件はその成果物の妥当性確認をもって前提を満たす（再実走による二重検証は行わない）。

### FR-002: DyNeRF（cut_roasted_beef）学習の完走

- **概要**: `train.py` を cut_roasted_beef の config で実行し、**coarse 3000 + fine 14000 反復**が完走して学習済み点群が生成されることを確認する。マルチGPU運用ルールに従い空きGPU・空きポートを選ぶ。
- **入力**: コマンド（train は `scene.gaussian_model`→`scene/regulation.py:5`・`utils/scene_utils.py:4` で `matplotlib.pyplot` を **top-level import** するため、書込可能な `MPLCONFIGDIR` と `MPLBACKEND=Agg` が必要。reader〔`readdynerfInfo`〕は `plot_camera_orientations` を呼ばず `output.png` は生成されないが、import 時のフォントキャッシュ書込に MPL 設定が要る）
  `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<空きN> .venv/bin/python train.py -s data/dynerf/cut_roasted_beef --port <空きP> --expname "dynerf/cut_roasted_beef" --configs arguments/dynerf/cut_roasted_beef.py`
- **出力**: `output/dynerf/cut_roasted_beef/point_cloud/iteration_14000/`（`point_cloud.ply` と deformation の重み `.pth` 群）、`output/dynerf/cut_roasted_beef/cfg_args`。
- **受け入れ基準**:
  - プロセスが終了コード 0 で完走する（クラッシュしない）。
  - matplotlib の top-level import（`scene.regulation`/`utils.scene_utils`）でクラッシュしない。事前確認: `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig .venv/bin/python -c "import matplotlib.pyplot; print(matplotlib.get_backend().lower())"` が `agg` を返す（MPL 未設定だと「Matplotlib requires access to a writable cache directory」でクラッシュしうる）。
  - `output/dynerf/cut_roasted_beef/point_cloud/iteration_14000/point_cloud.ply` が生成される（非空）。
  - 指定した物理GPU N のみに負荷が乗る（`nvidia-smi` で確認）。
  - coarse 段階の保存物（`iteration_3000` 等）は生成されない想定（`save_iterations` 最小14000に coarse 3000 が未到達のため。生成有無が設計の根拠どおりであることを確認）。

### FR-003: レンダリングの完走（test + video セット）

- **概要**: `render.py` を `--skip_train` で実行し、test セット（評価用）と video セット（視認用）の画像・動画が生成されることを確認する。
- **入力**: コマンド（render も train と同様 `scene.gaussian_model`→`matplotlib.pyplot` を top-level import するため MPL 設定が必要。`output.png` は生成されない〔plot 非呼び出し〕が import に MPL が要る）
  `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<FR-002と同一N> .venv/bin/python render.py --model_path output/dynerf/cut_roasted_beef --skip_train --configs arguments/dynerf/cut_roasted_beef.py`
- **出力**:
  - `output/dynerf/cut_roasted_beef/test/ours_14000/{renders,gt}/`（各 PNG）
  - `output/dynerf/cut_roasted_beef/video/ours_14000/renders/` と `video_rgb.mp4`
- **受け入れ基準**:
  - 終了コード 0 で完走する。
  - `test/ours_14000/renders/` と `test/ours_14000/gt/` の PNG 数が**同数かつ 1 枚以上**で、**両ディレクトリのファイル名集合が完全一致**する（空評価・対応欠落を排除）。test は cam00 の300フレーム想定（renders/gt 各300枚）。
  - iteration は未指定（`--iteration -1`）で最大の 14000 が自動選択される。

### FR-004: 評価の完走（test セット6指標）

- **概要**: `metrics.py` を実行し、test セットに対して PSNR/SSIM/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM が算出されることを確認する。引数の**正式名は `--model_paths`（`-m`、必須・複数指定可、`metrics.py:121`）**。argparse の前置一致で `--model_path`（単数）も受理される（feat-006 で確認済み・本案件でも実証済み）が、本書は**正式名 `--model_paths` を用いる**（前置一致への依存を避け明確化）。
- **入力**: コマンド（metrics は Scene 非構築・matplotlib 非経由のため MPL 不要）
  `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<FR-002と同一N> .venv/bin/python metrics.py --model_paths output/dynerf/cut_roasted_beef/`
- **出力**: `output/dynerf/cut_roasted_beef/{results,per_view}.json`（method=`ours_14000` の6指標）。
- **受け入れ基準**:
  - 終了コード 0 で完走し、6指標すべてが数値出力される。
  - `results.json` に `ours_14000` の6指標が記録され、**6指標すべてが有限値（`math.isfinite` が True。NaN/Inf でない）**である。
  - `per_view.json` の `ours_14000` の各指標の件数が、`test/ours_14000/renders/` の PNG 数と一致する。
  - **健全性チェック（参考値）**: 4DGS 論文（CVPR 2024, arXiv:2310.08528 Table 4）の cut_roasted_beef PSNR **33.85** を基準に、**PSNR ≥ 30** を目安とする。これは健全性の目安であり、**合否は終了コードと6指標の生成（クラッシュせず有限値が出ること）で判定**する（バグがあれば PSNR は大きく低下する。D-NeRF/HyperNeRF では論文値に近接した実績）。

### FR-005: 文書化・依存記録

- **概要**: DyNeRF 動作確認手順（前処理成果物の前提〜学習〜評価）を再現可能な形で文書化し、運用記録を更新する。**本案件は本体改変ゼロ**のため「オリジナルコードの変更点」への追記は無い。
- **入力**: FR-001〜004 の実施結果。
- **出力**:
  - `design.md` 手順（再現コマンド一式）。
  - `CLAUDE.md`: ①「データセット」節の DyNeRF 行を「学習〜評価まで動作確認済み（手順）」に更新、②「COLMAP の使い分け」節の「任意 GPU 選択（colmap.sh:5 引数化）は feat-012 スコープ」記述を「将来 COLMAP 再実走が必要な案件（feat-013 等）で検討」に更新。
  - `docs/BACKLOG.md`: feat-012 を Closed に更新（手動テスト合格後）。
  - 必要に応じ `docs/TECH_STACK.md`（DyNeRF 学習〜評価の補足）。
- **受け入れ基準**:
  - `/clear` 後でも本書＋設計書のみで、前処理成果物の前提確認〜学習〜レンダ〜評価を再現できる（成果物パス・train/render/metrics コマンド・GPU/ポート選択手順・MPL 環境変数を明記）。
  - **新規ライブラリ導入が無いことを確認する**（前処理を実走しないため cv2/imageio-ffmpeg/open3d/COLMAP 等の追加も不要）。新規導入が無ければ `requirements.lock.txt` の再生成は不要。
  - 4DGS 本体コードの差分がゼロであること（`git diff` で本体ファイルに変更が無い）。

## 1.4 非機能要求

- **対応環境**: Ubuntu / A100（sm_80）/ ヘッドレス（X ディスプレイ非前提）。**train/render は `scene.gaussian_model`→`scene/regulation.py:5`（`import matplotlib.pyplot`）および `utils/scene_utils.py:4` 経由で matplotlib を top-level import する**ため、書込可能な `MPLCONFIGDIR`（scratch 配下）と `MPLBACKEND=Agg` を固定する。これは reader 内の `plot_camera_orientations`（**HyperNeRF 経路専用**で DyNeRF では非呼び出し＝`output.png` は生成されない）とは独立した、**import 段階での要件**。**metrics.py は matplotlib も scene/gaussian_model も import しないため MPL 不要**（feat-010 ADR-6 の知見を継承）。
- **権限**: sudo 不可。ユーザー権限（`~/`・`/data/sakagawa/` 配下）で完結。本案件は apt 等のシステム導入を要しない。
- **処理時間**:
  - 学習（FR-002）: README 目安で約20分以下（A100×1）。レンダ・評価は数分以内を想定。
  - いずれも目安であり、合否は終了コードと生成物で判定する。
- **信頼性**: 学習出力は `output/dynerf/cut_roasted_beef/` に置く。再実行時は出力先を削除してから再実行できること。データ（`data/dynerf/`）はリポジトリにコミットしない。
- **マルチGPU**: 学習・レンダ・評価はいずれも 1 プロセス = 1 GPU。`CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N` を付け、同一ジョブ（train/render/metrics）で同一 N を使う。実行前に空きGPU（`nvidia-smi`）と空きポート（`ss -ltn | grep :<port>`、train のみ）を確認する（`CLAUDE.md` マルチGPU運用ルール準拠）。

## 1.5 制約条件

- **使用必須ライブラリ**（すべて導入済み・新規導入なし）:
  - 既存 `.venv`（torch 1.13.1+cu116 / mmcv 1.6.0 / numpy 1.23.5 等）と CUDA 拡張（feat-002 ビルド済み）。
  - **imageio 2.37.3 + imageio-ffmpeg 0.6.0** ＋ システム ffmpeg 4.4.2（render の `video_rgb.mp4` 書き出し）。
  - **plyfile**（`fetchPly` 用）。
  - matplotlib（導入済み。train/render の top-level import で必要。DyNeRF 経路では `plot_camera_orientations` は呼ばれず `output.png` は非生成）。
  - **opencv-python 4.13.0（必須）**: DyNeRF ローダ `scene/neural_3D_dataset_NDC.py:6` が `cv2` を top-level import し、`load_images_path`（`:323`）で既抽出PNG流用時も `cv2.VideoCapture(video_path)` を生成する（フレーム抽出ループ `:324-343` は `image_path` 不在時のみだが、`VideoCapture` 自体は常に実行）。前処理（フレーム抽出）は実走しないが、**train/render のローダ import/runtime 依存として opencv-python は導入済み必須**。
  - **注**: open3d（downsample）・COLMAP は前処理を実走しないため本案件のクリティカルパス外（feat-008/011 で導入済み）。
- **使用禁止 / 回避**:
  - **`uv sync` / `uv pip sync` は使わない**（追加的 `uv pip install` のみ）。`pyproject.toml` は作らない。
  - **4DGaussians 本体コードの改変は行わない（本体改変ゼロ）**。
  - numpy の版を 1.23.5 から動かさない。
- **ネットワーク**: 本案件はデータ取得（前処理流用のため）を要しない。LPIPS 重みは feat-006 でDL済み・キャッシュ読込。
- **ディスク**: 学習出力（`output/dynerf/cut_roasted_beef/`）＋ render の画像/動画で数 GB。`/data`（空き十分）に置く。`output/` はコミットしない。

## 1.6 優先順位（MoSCoW）

| 要求 | 優先度 | 備考 |
|------|--------|------|
| FR-001 前処理成果物の前提確認 | **Must** | 学習の前提（feat-011 流用物の実在・整合） |
| FR-002 学習完走 | **Must** | 判定基準（train.py 完走）に対応 |
| FR-003 レンダリング完走 | **Must** | 判定基準（render.py 完走）に対応 |
| FR-004 評価完走 | **Must** | 判定基準（metrics.py 完走）に対応 |
| FR-005 文書化・記録 | **Must** | 再現性・運用・スコープ更新の追跡のため |

- **MVP の範囲**: FR-001〜FR-005 すべて。これらの達成をもって BACKLOG の判定基準（前処理3段完走〔feat-011 実証を継承〕＋ train→render→metrics 完走）を満たす。
