# feat-012 機能設計書: DyNeRF動作確認（実シーン・多視点、cut_roasted_beef）・再開

本書は `docs/DESIGN_STANDARD.md` に準拠する。設計書中のコードスニペットは「意図の伝達」が目的であり、そのままコピーして使うものではない。中止した **feat-010 の機能設計を継承**し、**前処理を feat-011 成果物流用・本体改変ゼロ**へ更新した再開版である。

---

## 1.1 対応要求マッピング

| 要求ID | 設計セクション |
|--------|----------------|
| FR-001 前処理成果物の前提確認 | §1.4.1 |
| FR-002 学習完走 | §1.4.2 |
| FR-003 レンダリング完走 | §1.4.3 |
| FR-004 評価完走 | §1.4.4 |
| FR-005 文書化・依存記録 | §1.4.5 |

## 1.2 システム構成

本案件は **既存の前処理成果物（feat-011 生成）を前提に、既存コード（train/render/metrics）を実行する**だけであり、**4DGS 本体の改変はゼロ**である（feat-009 HyperNeRF と同方針）。関与する既存モジュールと役割は以下。

| ファイル/モジュール | 役割 | 本案件での関わり |
|---------------------|------|------------------|
| `scene/__init__.py:52-54` | データセット種別判定（`poses_bounds.npy` 存在 → `dynerf`、`dataset_type="dynerf"`） | cut_roasted_beef が DyNeRF 経路に入る判定 |
| `scene/dataset_readers.py:441-482` `readdynerfInfo` | DyNeRF SceneInfo 構築。`Neural3D_NDC_Dataset` を train/test 生成、`format_infos`/`format_render_poses` で CameraInfo を作り、`points3D_downsample2.ply` を読む（`:444`/`:469`、`maxtime=300`〔`:480`〕）。**`plot_camera_orientations` は呼ばない** | 必須 ply の配置先・split を規定 |
| `scene/dataset_readers.py:353-370` `format_infos` | `Neural3D_NDC_Dataset` から train split の `CameraInfo[]` を生成して返すのみ。**`plot_camera_orientations` は呼ばない** | DyNeRF の CameraInfo 構築。MPL 非依存 |
| `scene/dataset_readers.py:373-400` `readHyperDataInfos` | **HyperNeRF 専用** reader。`:390` で `plot_camera_orientations` を呼ぶ | 本案件（DyNeRF）では非経由。reader 内の plot 依存はこちら側 |
| `scene/dataset_readers.py:510` `plot_camera_orientations` | matplotlib で `output.png` を CWD に savefig | **呼び出し元は `readHyperDataInfos`（`:390`）のみ。DyNeRF 経路では呼ばれず `output.png` は非生成**（ただし MPL は下記 import 経路で別途必要） |
| `scene/gaussian_model.py:26`→`scene/regulation.py:5` | `import matplotlib.pyplot as plt`（**top-level**） | **train/render 共通**。`gaussian_model` を import する時点で matplotlib をロード |
| `train.py:30`→`utils/scene_utils.py:4` | `from matplotlib import pyplot as plt`（**top-level**） | **train のみ**。`render_training_image` の import で matplotlib をロード（render.py は scene_utils 非 import） |
| `scene/neural_3D_dataset_NDC.py:210-377` `Neural3D_NDC_Dataset` | `cv2` を **top-level import**（`:6`）。`poses_bounds.npy`・`cam*.mp4` を読み（`:268` で `assert len(videos)==poses_arr.shape[0]`）、`load_images_path`（`:323`）で `cv2.VideoCapture` を生成（**既抽出PNGがあっても実行**。抽出ループ `:324-343` のみ image_path 不在時に限定）、train/test 分割（`eval_index=0`、`:293`=train/`:313`=test）、時間正規化（`idx/countss`、`countss=300`〔`:310`〕）、spiral video poses（`val_poses`、`:287`） | データ構造の要件・split・時間軸を規定。**フレーム抽出は feat-011 で実施済み（既存PNGはスキップ）。`cv2` 依存により opencv-python は import/runtime 必須** |
| `arguments/dynerf/cut_roasted_beef.py` | cut_roasted_beef 学習 config（`_base_=./default.py`、`batch_size=2` 上書き） | そのまま使用（非改変） |
| `arguments/dynerf/default.py` | DyNeRF 既定（`iterations=14000`・`coarse_iterations=3000`・`batch_size=4`・`dataloader=True`・`densify_until_iter=10000`・`opacity_reset_interval=60000`） | cut_roasted_beef が継承 |
| `train.py` / `render.py` / `metrics.py` | 学習・レンダ・評価のエントリポイント | コマンド実行（非改変） |
| `scripts/preprocess_dynerf.py` / `colmap.sh` / `scripts/llff2colmap.py` / `database.py` / `scripts/downsample_point.py` | 前処理（フレーム抽出・COLMAP・ダウンサンプル） | **本案件では実行しない**（feat-011 で実走・成果物流用）。FR-001 で成果物の実在のみ確認 |

### モジュール依存（DyNeRF 読み込みフロー）

```
Scene.__init__ (scene/__init__.py:52, poses_bounds.npy 判定 → dataset_type="dynerf")
  └─ sceneLoadTypeCallbacks["dynerf"] = readdynerfInfo (dataset_readers.py:441)
       ├─ Neural3D_NDC_Dataset(datadir, "train", eval_index=0)  (neural_3D_dataset_NDC.py:210)
       │     reads: poses_bounds.npy + glob(cam*.mp4)  → 既抽出PNGを使用（feat-011で抽出済み）
       │     split: train = cam01..cam20（cam00以外の19台、:293） / test = cam00 のみ（:313, eval_index=0）
       ├─ Neural3D_NDC_Dataset(datadir, "test", eval_index=0)
       ├─ format_infos(train_dataset, "train") → CameraInfo[]  (dataset_readers.py:353-370, plot 非呼び出し＝MPL 非依存)
       ├─ format_render_poses(test_dataset.val_poses, test_dataset) → video用 CameraInfo[]  (:465)
       ├─ fetchPly("points3D_downsample2.ply")  (dataset_readers.py:469)  ← 必須（feat-011生成物 37,361点）
       └─ SceneInfo(maxtime=300, ...)  (:474-481)
```

### matplotlib ロード経路（MPL 環境変数が必要な理由）

```
train.py → import scene.gaussian_model → gaussian_model.py:26 → scene/regulation.py:5  (import matplotlib.pyplot)   ★train/render 共通
train.py:30 → utils/scene_utils.py:4  (from matplotlib import pyplot)                                              ★train のみ
render.py → import scene.gaussian_model → ... → scene/regulation.py:5                                              ★render も該当
metrics.py → gaussian_model も matplotlib も import しない                                                          → MPL 不要
```

## 1.3 技術スタック

- **言語**: Python 3.10（uv 管理 `.venv`）。
- **既存ライブラリ（新規導入なし）**: torch 1.13.1+cu116 / mmcv 1.6.0 / numpy 1.23.5 / matplotlib（backend=agg）/ **imageio 2.37.3 + imageio-ffmpeg 0.6.0** ＋ システム ffmpeg 4.4.2（render の `video_rgb.mp4` 書き出し）/ plyfile（`fetchPly` 用）。CUDA 拡張（`diff_gaussian_rasterization`・`simple_knn`）は feat-002 ビルド済み。
- **opencv-python 4.13.0（ローダ依存で必須）**: DyNeRF ローダ `scene/neural_3D_dataset_NDC.py:6` が `cv2` を top-level import し、`:323` で既抽出PNG流用時も `cv2.VideoCapture` を生成する。前処理（フレーム抽出）は実走しないが、train/render のローダ import/runtime 依存として導入済み必須。
- **本案件で不使用（前処理流用のため）**: open3d（downsample）・COLMAP。feat-008/011 で導入済みだがクリティカルパス外。
- **パッケージ管理**: uv（`uv pip install` の追加運用のみ。`uv sync`/`uv pip sync` 禁止）。**本案件は新規ライブラリ導入が無いため `requirements.lock.txt` 再生成は不要**。
- **選定理由**: すべて feat-002〜011 までに導入済み。DyNeRF 学習〜評価に固有の新規依存は存在しない。

## 1.4 各機能の詳細設計

> 全コマンドは `.venv/bin/python` を明示使用し bare `python` に依存しない。GPU を使うコマンド（train/render/metrics）は `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N` を付け、同一ジョブで同一 N を使う。train/render は MPL 環境変数（`MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp`）を付与する。`<scratch>` = `/data/sakagawa/tmp/feat012-dynerf`。

### 1.4.1 FR-001: 前処理成果物の前提確認（feat-011 流用）

#### データフロー

- 入力: feat-011 が生成した `data/dynerf/cut_roasted_beef/` 配下の成果物（`cam*.mp4`・`poses_bounds.npy`・`camXX/images/*.png`・`points3D_downsample2.ply`）。
- 出力: なし（確認のみ。学習の前提を満たすことの検証）。

#### 処理ロジック（確認手順）

1. カメラ・ポーズ整合（ローダ `:268` の assert 事前検証）:
   ```python
   # 意図の伝達用（実装ではワンライナーで可）
   import numpy as np, glob
   p = np.load("data/dynerf/cut_roasted_beef/poses_bounds.npy")
   cams = sorted(glob.glob("data/dynerf/cut_roasted_beef/cam*.mp4"))
   assert p.shape == (20, 17) and len(cams) == p.shape[0]   # 20本・(20,17)・一致
   ```
2. フレーム抽出物の確認: 全20カメラ（cam00〜cam20、cam04欠番）の `camXX/images/*.png` が各300枚。
3. 初期点群の確認（**点数だけでなく `fetchPly` が読めることを保証する**。`fetchPly` は x/y/z・red/green/blue・nx/ny/nz の9フィールドを要求するため、色/法線欠落の PLY は FR-002 で `KeyError`）:
   ```python
   from scene.dataset_readers import fetchPly   # dataset_readers.py:124-130
   pcd = fetchPly("data/dynerf/cut_roasted_beef/points3D_downsample2.ply")
   n = pcd.points.shape[0]
   assert 0 < n <= 40000   # 実測 37,361。vertex に x,y,z,nx,ny,nz,red,green,blue が一式あることを確認済み
   ```

#### エラーハンドリング

- 成果物が欠落/不整合（例: `cam*.mp4` 数 ≠ `poses_bounds.npy` 行数、images 不足、ply 欠落）→ 学習が `:268` の assert や `fetchPly` で失敗する。その場合は feat-011 の前処理（`colmap.sh ... llff` を **3.11.1** で）を再実走して再生成する（CLAUDE.md「COLMAP の使い分け」節）。本案件のデフォルト経路ではない（流用前提）。

#### 境界条件

- フレーム数が 300 未満のカメラがある場合: 全カメラで同数なら学習は走る（feat-010 知見）。本案件は feat-011 実証で各300枚を確認済み。

### 1.4.2 FR-002: DyNeRF（cut_roasted_beef）学習の完走

#### データフロー

- 入力: `data/dynerf/cut_roasted_beef/`（cam*.mp4・camXX/images・poses_bounds.npy）＋ `points3D_downsample2.ply` ＋ `arguments/dynerf/cut_roasted_beef.py`。
- 出力: `output/dynerf/cut_roasted_beef/point_cloud/iteration_14000/`（`point_cloud.ply`・deformation の `.pth` 群）、`output/dynerf/cut_roasted_beef/cfg_args`。

#### 処理ロジック

1. 空きGPU N（`nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv`）・空きポート P（`ss -ltn | grep :<P>` が空）を選ぶ。
2. MPL scratch 準備と事前確認（train は `gaussian_model`→`matplotlib.pyplot` を top-level import するため。**本番と同一の環境変数**で確認）:
   ```bash
   mkdir -p <scratch>/{mplconfig,tmp}
   MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp \
   .venv/bin/python -c "import matplotlib.pyplot; print(matplotlib.get_backend().lower())"   # → agg（import 成功）
   ```
3. 実行（`MPLBACKEND=Agg`・`MPLCONFIGDIR`・`TMPDIR` を固定。約20分目安のためバックグラウンド実行・ログ監視）:
   ```bash
   MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp \
   CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N \
   .venv/bin/python train.py -s data/dynerf/cut_roasted_beef --port P \
     --expname "dynerf/cut_roasted_beef" --configs arguments/dynerf/cut_roasted_beef.py
   ```
4. config 反映（設計根拠）: `cut_roasted_beef.py`（`_base_=./default.py`、`batch_size=2`）→ `default.py` の `iterations=14000`・`coarse_iterations=3000`・`dataloader=True`・`densify_until_iter=10000` が有効。`maxtime=300`（`readdynerfInfo:480`）。
5. 保存イテレーションの根拠（feat-004/009 と同一機構）: `train.py` の `save_iterations.append(args.iterations)` は merge 前に実行され、その時点の `args.iterations` は OptimizationParams クラス既定 30000。`save_iterations` の最小は 14000。fine 段階（train_iter=14000）で 14000 に到達し保存。coarse 段階（train_iter=3000）は最小14000に未到達のため**保存されない**。結果 `iteration_14000` のみ生成。

#### エラーハンドリング

- `points3D_downsample2.ply` 欠落 → `readdynerfInfo` の `fetchPly`（`:469`）で例外（フォールバック無し）。FR-001 で実在を確認済み。
- ポート使用中 → `network_gui` の `bind()` 例外が未処理で起動時クラッシュ（CLAUDE.md 既知事項）。手順1で必ず空きポート確認。
- GPU OOM/他者と競合 → 別の空きGPU を選び直す。`CUDA_DEVICE_ORDER=PCI_BUS_ID` 必須（index ズレ防止）。
- matplotlib 例外（`Matplotlib requires access to a writable cache directory`）→ train は `gaussian_model`→`regulation.py:5` および `scene_utils.py:4` で matplotlib を top-level import するため、`MPLCONFIGDIR`（書込可能）と `MPLBACKEND=Agg` を必ず付与する。手順2の事前確認で import 成功（agg）を担保。
- DataLoader（`dataloader=True`）のワーカ起因エラー → ログを確認。多視点×300フレームのパス参照のみ（`__getitem__` で都度 `Image.open`）のためメモリ逼迫は通常起きない。

#### 境界条件

- 学習が極端に遅い/速い: 目安20分。逸脱しても終了コード0・生成物ありで合格判定。
- `output/dynerf/cut_roasted_beef/` が既存: 再実行時は削除してから実行。

### 1.4.3 FR-003: レンダリングの完走

#### データフロー

- 入力: `output/dynerf/cut_roasted_beef/point_cloud/iteration_14000/`、`arguments/dynerf/cut_roasted_beef.py`。
- 出力: `test/ours_14000/{renders,gt}/*.png`、`video/ours_14000/renders/*.png`、`video/ours_14000/video_rgb.mp4`、`test/ours_14000/video_rgb.mp4`。

#### 処理ロジック

1. 実行（render も `gaussian_model`→`regulation.py:5` で matplotlib を top-level import するため MPL 設定を固定。`output.png` は生成されない〔plot 非呼び出し〕が import に MPL が要る）:
   ```bash
   MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp \
   CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N \
   .venv/bin/python render.py --model_path output/dynerf/cut_roasted_beef \
     --skip_train --configs arguments/dynerf/cut_roasted_beef.py
   ```
2. `render_sets` は `--skip_train` で train をスキップ、test（cam00 の300フレーム）と video（spiral 300フレーム）を生成。`--iteration -1` 既定で最大 14000 を自動選択。
3. `cam_type="dynerf"`（`scene.dataset_type`）で render に渡る。test の gt は `view.original_image` から保存。
4. 検証: `test/ours_14000/renders/` と `test/ours_14000/gt/` の PNG 数が**同数かつ 1 枚以上**で、ファイル名集合が完全一致（`.venv/bin/python` で `set(os.listdir(renders))==set(os.listdir(gt))` かつ `len>0`）。これにより FR-004 の空評価・NaN を未然に排除する。

#### エラーハンドリング

- test セットが空 → renders/gt が0枚になり metrics が `FileNotFoundError`。FR-003 検証で renders/gt 同数>0 を確認。
- 動画書き出し（imageio/ffmpeg）失敗 → `video_rgb.mp4` 非生成でも、metrics が使う `test/ours_14000/{renders,gt}` が揃えば FR-004 は可能。動画は視認用。

#### 境界条件

- test は cam00 の300フレーム想定（renders/gt 各300枚）。

### 1.4.4 FR-004: 評価の完走

#### データフロー

- 入力: `output/dynerf/cut_roasted_beef/test/ours_14000/{renders,gt}/`。
- 出力: `output/dynerf/cut_roasted_beef/results.json`・`per_view.json`（method=`ours_14000`、6指標）。

#### 処理ロジック

1. 実行（metrics は Scene 非構築・matplotlib 非経由のため MPL 不要。引数の正式名は `--model_paths`/`-m`〔`metrics.py:121`〕。前置一致で `--model_path` も可だが正式名を使う）:
   ```bash
   CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N \
   .venv/bin/python metrics.py --model_paths output/dynerf/cut_roasted_beef/
   ```
2. `metrics.py:evaluate` は `test/` 配下の各 `ours_*` を列挙し、`renders/`・`gt/` から PSNR/SSIM/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM を算出。LPIPS 重みは feat-006 でDL済み・キャッシュ読込。
3. 検証: `results.json` の `ours_14000` の6指標すべてが**有限値**（`.venv/bin/python` で `all(math.isfinite(v))`）。`per_view.json` の各指標件数が `renders/` の PNG 数と一致。

#### エラーハンドリング

- `renders/`・`gt/` 欠落 → `FileNotFoundError`（`evaluate` が捕捉し `Unable to compute metrics for model <path>` 出力後に再送出・非0終了）。FR-003 完了が前提。
- 健全性目安（PSNR≥30、論文33.85基準）を外れる → クラッシュしなければ FR-004 自体は合格扱いとし、値の乖離は investigation に記録（バグ時は PSNR が大きく低下する。D-NeRF/HyperNeRF では論文値に近接した実績）。

#### 境界条件

- `test/` 配下に複数 `ours_*` が併存（再レンダ等）→ 両方が results.json に method 別で出る。本案件は `ours_14000` のみの想定。

### 1.4.5 FR-005: 文書化・依存記録

- `CLAUDE.md`:
  - ①「データセット」節の「Plenoptic / DyNeRF」行を、本案件の確定手順（cut_roasted_beef、前処理成果物の前提・train/render/metrics・MPL注意）に更新し「学習〜評価まで動作確認済み」とする。
  - ②「COLMAP の使い分け」節の「任意 GPU 選択（colmap.sh:5 引数化）は feat-012 スコープ」記述を「**将来 COLMAP 再実走が必要な案件（feat-013 等）で検討**」に更新する（本案件は前処理流用・本体改変ゼロのため）。
  - **「オリジナルコードの変更点」への追記は無い**（本体改変ゼロ）。
- `docs/TECH_STACK.md`: 必要に応じ DyNeRF 学習〜評価の補足を追記（前処理は feat-011 を参照）。
- `requirements.lock.txt`: 新規ライブラリ導入が無いため再生成不要。
- `docs/BACKLOG.md`: feat-012 を Closed に更新（手動テスト合格後）。

## 1.6 ファイル・ディレクトリ設計

```
data/dynerf/cut_roasted_beef/          # git 未追跡（.gitignore 管理外運用）。FR-001 で実在確認（feat-011 生成物を流用）
├── cam00.mp4 .. cam20.mp4             # 20本（cam04 欠番）
├── poses_bounds.npy                   # (20,17)、DyNeRF 判定キー
├── cam00/images/0000.png..0299.png    # 全20カメラ・各300枚
├── cam01/images/...                   #   〃
├── sparse_/ image_colmap/ colmap/     # feat-011 の COLMAP 中間生成物（本案件では未使用）
└── points3D_downsample2.ply           # 学習用点群（必須、37,361点）

output/dynerf/cut_roasted_beef/        # 学習・レンダ・評価の出力（git 未追跡）
├── point_cloud/iteration_14000/       # FR-002: point_cloud.ply + deformation*.pth
├── cfg_args
├── test/ours_14000/{renders,gt}/      # FR-003（評価対象、cam00 の300フレーム）
├── video/ours_14000/renders/          # FR-003（視認用、spiral 300フレーム）
├── results.json / per_view.json       # FR-004
└── ...

/data/sakagawa/tmp/feat012-dynerf/     # scratch（リポジトリ外・非コミット）
├── mplconfig/                         # MPLCONFIGDIR（train/render の matplotlib top-level import 用）
├── tmp/                               # TMPDIR
└── *.log                              # 長時間処理（学習）のログ保存先

# 注: DyNeRF 経路は plot_camera_orientations を呼ばないため output.png は生成されない
#     （HyperNeRF/feat-009 との差異）。ただし train/render は gaussian_model→matplotlib.pyplot を
#     top-level import するため MPLCONFIGDIR/MPLBACKEND=Agg は必要（metrics は不要）。
```

## 1.7 インターフェース定義（実行コマンド）

| 機能 | コマンド（GPU N・ポート P は実行直前に選択。`<scratch>`=`/data/sakagawa/tmp/feat012-dynerf`） |
|------|----------------------------------------------|
| FR-001 前提確認 | `.venv/bin/python` で poses_bounds 形状 `(20,17)`・`cam*.mp4`=20・各 `camXX/images`=300枚・`points3D_downsample2.ply` を `fetchPly` で読込確認（点数 0<n≤40k ＋ `x/y/z/nx/ny/nz/red/green/blue` の9フィールド）を検証 |
| FR-002 学習 | `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python train.py -s data/dynerf/cut_roasted_beef --port P --expname "dynerf/cut_roasted_beef" --configs arguments/dynerf/cut_roasted_beef.py` |
| FR-003 レンダ | `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python render.py --model_path output/dynerf/cut_roasted_beef --skip_train --configs arguments/dynerf/cut_roasted_beef.py` |
| FR-004 評価 | `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python metrics.py --model_paths output/dynerf/cut_roasted_beef/`（MPL 不要。正式名 `--model_paths`/`-m`） |

- **4DGS 本体の改変は行わない（本体改変ゼロ）**。関数シグネチャ変更・スクリプト改変なし。

## 1.8 ログ・デバッグ設計

- 長時間処理（FR-002 学習）はバックグラウンド実行し、`tee`/リダイレクトでログを scratch に保存して監視する。
- 学習/レンダのログは進捗バー（`\r`）で巨大化しうる。要点抽出に `grep -avE "it/s|it\]|findfont"` 等を用いる（feat-009 の知見）。
- 学習ログ: tqdm 進捗（PSNR・点数）と `[ITER N] Evaluating`、`save` メッセージで `iteration_14000` 生成を確認。
- レンダ: `point nums: <N>` が学習最終点数と一致することを確認。
- 評価: 6指標と `results.json` 生成を確認。値は health-check（PSNR≥30、論文33.85）と照合。
- DyNeRF 経路は `plot_camera_orientations` を呼ばないため `output.png` は生成されない（HyperNeRF/feat-009 との差異。生成された場合は経路の想定外として investigation に記録）。

## 2.4 設計判断の記録（ADR）

### ADR-1: 対象シーンは cut_roasted_beef（feat-010 から継承）

- **採用**: cut_roasted_beef（ユーザー選択、2026-06-22）。
- **却下**: sear_steak（README train_dynerf.sh の例）、その他4シーン。
- **理由**: BACKLOG・README の公式例であり、4DGS 論文 Table 4 に個別 PSNR **33.85dB** の報告があり健全性チェックの基準値が明確。feat-011 でも本シーンで前処理を実証済みで一貫する。

### ADR-2: 前処理は feat-011 成果物を流用（feat-010 ADR-2「COLMAP 実走必須」を更新）

- **採用**: feat-011 が `colmap.sh ... llff`（**3.11.1**）で生成した成果物（`points3D_downsample2.ply` 37,361点ほか）を流用し、本案件では前処理を再実走しない（2026-06-24 ユーザー決定）。
- **背景**: DyNeRF は事前生成点群が配布されないため、feat-011 で初めて `colmap.sh` を実走し点群生成済み（`fused.ply` 387,496点・Mean reprojection error 0.852px・20/20登録 → downsample 37,361点）。この実証は feat-011 の手動テストで合格済み。
- **却下（再実走）**: feat-012 内で前処理3段を再実走して案件内完結させる案。前処理は feat-011 で実証済みで二度手間であり、`colmap.sh` は GPU0 固定・dense MVS で時間がかかる。本プロジェクトの「小さく積む」方針に照らし流用が妥当。
- **再生成が必要な場合**: 成果物が壊れた等の際は **3.11.1** で `colmap.sh ... llff` を再実走する（3.12.6 は rig 非互換クラッシュ。CLAUDE.md「COLMAP の使い分け」節）。

### ADR-3: 本体改変ゼロ（feat-010 ADR-3「colmap.sh GPU 引数化」を撤回）

- **採用**: 4DGS 本体コードを 1 行も改変しない（feat-009 HyperNeRF と同方針）。feat-010 が予定した `colmap.sh:5` の GPU 引数化（`export CUDA_VISIBLE_DEVICES=${3:-0}`）は**行わない**（2026-06-24 ユーザー決定）。
- **理由**: 前処理を流用し `colmap.sh` を実走しないため、引数化を入れても**動作確認できない**。「動作確認してから次へ」という本プロジェクト方針に照らし、未検証の改変を残すのは不適切。train/render/metrics は `CUDA_VISIBLE_DEVICES` 環境変数で任意GPUに乗る（feat-007 検証済み）ため、学習以降のマルチGPU運用に本体改変は不要。
- **CLAUDE.md スコープ記述の更新**: 「任意 GPU 選択（colmap.sh:5 引数化）は feat-012 スコープ」を「**将来 COLMAP 再実走が必要な案件（feat-013 等）で検討**」に更新する（FR-005）。将来 `colmap.sh`/`multipleviewprogress.sh` を任意GPUで実走したい場合は、その案件で引数化＋実走検証をセットで行う。

### ADR-4: レンダは `--skip_train` のみ（test を残す。feat-010 から継承）

- **採用**: `render.py --skip_train`（test + video を生成）。
- **却下**: `--skip_train --skip_test`（video のみ）。
- **理由**: 本案件は評価（FR-004）まで行うため test セットの renders/gt が必須。

### ADR-5: train/test split は eval_index=0 固定（コード仕様、非改変。feat-010 から継承）

- **採用**: `readdynerfInfo` がローダに渡す `eval_index=0`（`:453`/`:462`）をそのまま使う（**cam00 が test、残り19台が train**、`neural_3D_dataset_NDC.py:293`/`:313`）。
- **背景**: DyNeRF コミュニティ慣行（中央カメラを held-out）。`eval_index=0` はハードコードで config からは変えられない。
- **理由**: 本体・config 非改変の方針。論文・公式スクリプトと同じ評価設定（cam00 held-out）で apples-to-apples の比較になる。

### ADR-6: train/render に MPL 環境変数を付与（matplotlib top-level import のため）。metrics は不要（feat-010 から継承・本案件で再確認）

- **採用**: `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp` を train（FR-002）・render（FR-003）に付与。metrics（FR-004）には付与しない。
- **背景（実コード確認、2026-06-24 再確認）**: MPL が必要な理由は matplotlib の top-level import。
  1. **train/render 共通**: `import scene.gaussian_model`→`scene/gaussian_model.py:26`→`scene/regulation.py:5`（`import matplotlib.pyplot as plt`）。
  2. **train のみ**: `train.py:30`→`utils/scene_utils.py:4`（`from matplotlib import pyplot as plt`）。
  - いずれもモジュール読込時にロードされ、書込可能な `MPLCONFIGDIR` が無いと「Matplotlib requires access to a writable cache directory」で **import 段階クラッシュ**しうる。ヘッドレスのため `MPLBACKEND=Agg`。
  - reader 内の savefig（`plot_camera_orientations`、`:510`）は **HyperNeRF 専用 `readHyperDataInfos`（`:390`）でのみ呼ばれ**、DyNeRF の `readdynerfInfo`→`format_infos`（plot 非呼び出し）では呼ばれない。よって DyNeRF では `output.png` は生成されないが、上記 top-level import により MPL は必要。
- **理由**: metrics.py は `gaussian_model`/`matplotlib` を import しないため MPL 不要。feat-010 で codex-02 が検出した「MPL 全削除では train/render が import 段階で落ちる」という知見をそのまま継承する。
