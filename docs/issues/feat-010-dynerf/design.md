# feat-010 機能設計書: DyNeRF動作確認（実シーン・多視点、cut_roasted_beef）

本書は `docs/DESIGN_STANDARD.md` に準拠する。設計書中のコードスニペットは「意図の伝達」が目的であり、そのままコピーして使うものではない。

---

## 1.1 対応要求マッピング

| 要求ID | 設計セクション |
|--------|----------------|
| FR-001 データ取得・配置 | §1.4.1 |
| FR-002 フレーム抽出 | §1.4.2 |
| FR-003 COLMAP 点群生成＋colmap.sh 改変 | §1.4.3 |
| FR-004 ダウンサンプル | §1.4.4 |
| FR-005 学習完走 | §1.4.5 |
| FR-006 レンダリング完走 | §1.4.6 |
| FR-007 評価完走 | §1.4.7 |
| FR-008 文書化・本体変更点記録 | §1.4.8 |

## 1.2 システム構成

本案件は**環境・データ整備＋前処理スクリプト実走＋既存コードの実行**であり、4DGS 本体の改変は `colmap.sh:5` の1行（GPU 引数化）に限定する。関与する既存モジュールと役割は以下。

| ファイル/モジュール | 役割 | 本案件での関わり |
|---------------------|------|------------------|
| `scene/__init__.py:52-54` | データセット種別判定（`poses_bounds.npy` 存在 → `dynerf`） | cut_roasted_beef が DyNeRF 経路に入る判定 |
| `scene/dataset_readers.py:441-482` `readdynerfInfo` | DyNeRF SceneInfo 構築。`Neural3D_NDC_Dataset` を train/test 生成、`format_infos`/`format_render_poses` で CameraInfo を作り、`points3D_downsample2.ply` を読む。**`plot_camera_orientations` は呼ばない** | 必須 ply の配置先を規定 |
| `scene/dataset_readers.py:353-370` `format_infos` | `Neural3D_NDC_Dataset` から train split の `CameraInfo[]` を生成して返すのみ（`return cameras`、`:370`）。**`plot_camera_orientations` は呼ばない** | DyNeRF の CameraInfo 構築。MPL 非依存 |
| `scene/dataset_readers.py:373-400` `readHyperDataInfos` | **HyperNeRF 専用** reader。`:390` で `plot_camera_orientations` を呼ぶ | 本案件（DyNeRF）では非経由。MPL 依存はこちら側 |
| `scene/dataset_readers.py:510-534` `plot_camera_orientations` | matplotlib で `output.png` を CWD に savefig | **呼び出し元は `readHyperDataInfos`（`:390`）のみ。DyNeRF 経路では呼ばれず `output.png` は非生成**（ただし MPL は下記 import 経路で別途必要） |
| `scene/gaussian_model.py:26`→`scene/regulation.py:5`・`utils/scene_utils.py:4` | `import matplotlib.pyplot`（**top-level**） | train/render は `gaussian_model` を import する時点で matplotlib をロード。**MPL 環境変数（`MPLCONFIGDIR`/`MPLBACKEND=Agg`）が必要な真の理由**（reader の plot とは独立。フォントキャッシュ書込にディレクトリが要る）。metrics.py・preprocess_dynerf.py は gaussian_model 非 import のため不要 |
| `scene/neural_3D_dataset_NDC.py:210-377` `Neural3D_NDC_Dataset` | `poses_bounds.npy`・`cam*.mp4` を読み、cv2 でフレーム抽出、train/test 分割（eval_index=0）、時間正規化（idx/300） | データ構造の要件（cam*.mp4 + poses_bounds.npy）・抽出ロジックを規定 |
| `scripts/preprocess_dynerf.py` | `Neural3D_NDC_Dataset` を train/test でインスタンス化しフレーム抽出をトリガ | FR-002 で実行（非改変） |
| `colmap.sh` | COLMAP 一括前処理（llff 分岐で `llff2colmap.py` 経由） | **本案件で実走。`:5` を引数化改変（FR-003・ADR-3）** |
| `scripts/llff2colmap.py` | `poses_bounds.npy`→COLMAP テキスト（cameras/images/points3D.txt）変換、`image_colmap/` に各カメラ先頭フレームをコピー | colmap.sh が内部呼び出し（非改変） |
| `database.py` | COLMAP database に cameras.txt のカメラパラメータを反映 | colmap.sh が内部呼び出し（非改変） |
| `scripts/downsample_point.py` | `fused.ply` を ≤40,000 点に voxel ダウンサンプル | FR-004 で実行（非改変） |
| `arguments/dynerf/cut_roasted_beef.py` | cut_roasted_beef 学習 config（`_base_=default.py`、batch_size=2 上書き） | そのまま使用（非改変） |
| `arguments/dynerf/default.py` | DyNeRF 既定（fine 14000・coarse 3000・batch 4・dataloader=True） | cut_roasted_beef が継承 |
| `train.py` / `render.py` / `metrics.py` | 学習・レンダ・評価のエントリポイント | コマンド実行（非改変） |

### モジュール依存（DyNeRF 読み込みフロー）

```
Scene.__init__ (scene/__init__.py:52, poses_bounds.npy 判定)
  └─ sceneLoadTypeCallbacks["dynerf"] = readdynerfInfo (dataset_readers.py:441)
       ├─ Neural3D_NDC_Dataset(datadir, "train"/"test", eval_index=0)  (neural_3D_dataset_NDC.py:210)
       │     reads: poses_bounds.npy + glob(cam*.mp4)  → cv2 でフレーム抽出（既抽出ならスキップ）
       │     split: train=cam01..camNN / test=cam00 のみ（eval_index=0）
       ├─ format_infos(train_dataset, "train") → CameraInfo[]  (dataset_readers.py:353-370, plot 呼び出し無し＝MPL 非依存)
       ├─ fetchPly("points3D_downsample2.ply")  (dataset_readers.py:444)  ← 必須
       └─ SceneInfo(maxtime=300, ...)
```

### 前処理パイプライン依存（FR-001→004）

```
[FR-001] cut_roasted_beef.zip
   └─ 展開 → data/dynerf/cut_roasted_beef/{cam00.mp4..camNN.mp4, poses_bounds.npy}
[FR-002] preprocess_dynerf.py --datadir data/dynerf/cut_roasted_beef
   └─ camXX/images/0000.png..0299.png（全カメラ、各300枚）
[FR-003] colmap.sh data/dynerf/cut_roasted_beef llff N
   ├─ llff2colmap.py: poses_bounds.npy + camXX/images/0000.png → sparse_/(cameras,images,points3D).txt + image_colmap/r_XXX.png
   ├─ feature_extractor → exhaustive_matcher → point_triangulator → image_undistorter → patch_match_stereo → stereo_fusion
   └─ colmap/dense/workspace/fused.ply
[FR-004] downsample_point.py fused.ply points3D_downsample2.ply
   └─ data/dynerf/cut_roasted_beef/points3D_downsample2.ply（≤40k点）
[FR-005] train.py -s data/dynerf/cut_roasted_beef ...
```

## 1.3 技術スタック

- **言語**: Python 3.10（uv 管理 `.venv`）。
- **既存ライブラリ（新規導入なし）**: torch 1.13.1+cu116 / mmcv 1.6.0 / numpy 1.23.5 / matplotlib（backend=agg）/ **opencv-python 4.13.0**（cv2、mp4 フレーム抽出）/ **imageio 2.37.3 + imageio-ffmpeg 0.6.0** ＋ システム ffmpeg 4.4.2（動画書き出し）/ **open3d 0.19.0**（ダウンサンプル）/ plyfile（fetchPly 用）。CUDA 拡張（`diff_gaussian_rasterization`・`simple_knn`）は feat-002 ビルド済み。
- **外部ツール**: COLMAP（feat-008、vcpkg、`~/.local/bin/colmap`、CUDA有効・GUI除外）。
- **パッケージ管理**: uv（`uv pip install` の追加運用のみ。`uv sync`/`uv pip sync` 禁止）。**本案件は新規ライブラリ導入が無い見込みのため `requirements.lock.txt` 再生成は原則不要**（導入が発生した場合のみ `uv pip freeze` で再生成）。
- **外部データ取得**: GitHub Release（curl/wget）。
- **選定理由**: 上記はすべて公式手順（README:146-156）が前提とするツールで、feat-008/009 までに導入済み。DyNeRF 固有の新規依存は存在しない。

## 1.4 各機能の詳細設計

> 全コマンドは `.venv/bin/python` を明示使用し bare `python` に依存しない。GPU を使うコマンド（COLMAP/train/render/metrics）は `CUDA_DEVICE_ORDER=PCI_BUS_ID` ＋ GPU 指定（COLMAP は colmap.sh の第3引数 N、train/render/metrics は `CUDA_VISIBLE_DEVICES=N`）を付け、同一ジョブで同一 N を使う。作業用の一時DL/展開は scratch `/data/sakagawa/tmp/feat010-dynerf/`（リポジトリ外）で行い、確定物のみ `data/dynerf/cut_roasted_beef/` に置く。`<scratch>` = `/data/sakagawa/tmp/feat010-dynerf`。

### 1.4.1 FR-001: データセット取得・配置

#### データフロー

- 入力: `cut_roasted_beef.zip`（約1.06GB、GitHub Release v1.0）。
- 中間: scratch に展開した動画群＋ポーズ。
- 出力: `data/dynerf/cut_roasted_beef/`（`cam00.mp4`〜`camNN.mp4`・`poses_bounds.npy`）。

#### 処理ロジック（手順）

1. scratch 準備: `mkdir -p /data/sakagawa/tmp/feat010-dynerf`。
2. DL: `curl -L -o <scratch>/cut_roasted_beef.zip https://github.com/facebookresearch/Neural_3D_Video/releases/download/v1.0/cut_roasted_beef.zip`（約1.06GB。バックグラウンド実行・ログ監視）。
3. 展開: `unzip -q <scratch>/cut_roasted_beef.zip -d <scratch>/extract`。
4. **構造確認**: `ls -R <scratch>/extract` で `cam*.mp4` と `poses_bounds.npy` を含むディレクトリ（シーンルート）を特定する。zip トップが `cut_roasted_beef/` フォルダを内包する場合とフラットな場合の両方を想定し、`find <scratch>/extract -name "poses_bounds.npy"` で確実に位置を特定する。
5. 配置: `mkdir -p data/dynerf` の上で、シーンルートを `data/dynerf/cut_roasted_beef/` として配置する（`cp -r <シーンルート> data/dynerf/cut_roasted_beef`。シーンルート名が既に `cut_roasted_beef` ならそのまま、異なれば配置先名を `cut_roasted_beef` にする）。
6. 検証:
   ```python
   # 意図の伝達用（実装ではワンライナーで可）
   import numpy as np, glob
   p = np.load("data/dynerf/cut_roasted_beef/poses_bounds.npy")
   cams = sorted(glob.glob("data/dynerf/cut_roasted_beef/cam*.mp4"))
   assert p.shape[1] == 17 and len(cams) == p.shape[0] and len(cams) >= 2
   ```

#### エラーハンドリング

- DL 失敗（ネットワーク/サイズ不足）: ファイルサイズが約1.06GB でなければ再取得。`curl -L`（リダイレクト追従）必須。GitHub の release アセットは S3 へリダイレクトされるため `-L` 無しだと HTML が返る。
- `cam*.mp4` 数と `poses_bounds.npy` のカメラ数が不一致: ローダの `assert len(videos)==poses_arr.shape[0]`（`neural_3D_dataset_NDC.py:268`）で FR-002 がクラッシュする。配置物が欠損していないか `ls -R` で再確認（欠番カメラがある場合も実数で一致していれば可）。
- zip 内に `poses_bounds.npy` が無い: Neural_3D_Video のシーン zip には同梱される想定。万一欠落していれば investigation に記録しブロック（4DGS は LLFF ポーズの自前生成手段を DyNeRF 経路に持たない）。

#### 境界条件

- 既に `data/dynerf/cut_roasted_beef/` が存在する場合: 内容を確認し、不完全なら削除してから再配置。
- ディスク: zip＋展開で数GB。`/data`（15TB 空き）に置く。scratch は完了後に削除可。

### 1.4.2 FR-002: フレーム抽出（preprocess_dynerf.py）

#### データフロー

- 入力: `data/dynerf/cut_roasted_beef/{cam*.mp4, poses_bounds.npy}`。
- 出力: `data/dynerf/cut_roasted_beef/camXX/images/0000.png〜0299.png`（全 N カメラ、各300枚、解像度1352×1014）。

#### 処理ロジック

1. 実行（DyNeRF 経路は matplotlib 非経由のため MPL 環境変数は不要）:
   ```bash
   .venv/bin/python scripts/preprocess_dynerf.py --datadir data/dynerf/cut_roasted_beef
   ```
2. 内部挙動（`Neural3D_NDC_Dataset.load_images_path`、`neural_3D_dataset_NDC.py:304-366`）:
   - train split のインスタンス化で cam00 をスキップし cam01..camNN を処理、test split で cam00 を処理 → 結果として**全カメラ**の `camXX/images/` が生成される。
   - 各 mp4 を `cv2.VideoCapture` で開き、先頭から最大 `countss=300` フレームを `cv2.cvtColor(BGR→RGB)` → `PIL.Image` → `resize(img_wh, LANCZOS)` → `images/%04d.png` 保存。
   - `images/` が既存なら抽出をスキップして既存 PNG を使う（再実行時の冪等性）。
3. 検証: 全 N カメラについて `ls data/dynerf/cut_roasted_beef/camXX/images/*.png | wc -l == 300`、かつ 1枚を `PIL.Image.open` して `size==(1352,1014)`。

#### エラーハンドリング

- cv2 が mp4 を開けない（コーデック/破損）: `VideoCapture.isOpened()` が False で images/ が空。opencv-python 4.13.0 は ffmpeg バックエンド同梱のため通常は読める。空になった場合は mp4 を `ffprobe`/`ffmpeg -i` で確認し investigation に記録。
- 抽出が300枚未満（動画が300フレーム未満）: ループは `ret==False`（動画末尾）で break するため枚数が不足しうる。受け入れ基準は300枚だが、全カメラで同数（< 300 でも揃っている）なら学習自体は走る。逸脱時は investigation に記録。
- ディスク不足: N×300枚の PNG（各 ~数百KB）を生成。空き確認（`df -h /data`）。

#### 境界条件

- 再実行: `images/` 存在時は抽出スキップ。作り直す場合は `camXX/images/` を削除してから実行。
- カメラ数 N: cut_roasted_beef は約19〜21台想定（実数は FR-001 で確定）。`poses_bounds.npy` のカメラ数に一致。

### 1.4.3 FR-003: COLMAP 点群生成 ＋ colmap.sh の GPU 引数化改変

#### データフロー

- 入力: `data/dynerf/cut_roasted_beef/{poses_bounds.npy, camXX/images/0000.png}`。
- 中間: `sparse_/`・`image_colmap/`・`colmap/database.db`・`colmap/sparse/0/`・`colmap/dense/workspace/`。
- 出力: `data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply`。

#### 処理ロジック

1. **colmap.sh の改変（本体変更点。ADR-3）**:
   ```diff
   # colmap.sh:5
   - export CUDA_VISIBLE_DEVICES=0
   + export CUDA_VISIBLE_DEVICES=${3:-0}
   ```
   - これにより第3引数 N が `CUDA_VISIBLE_DEVICES` に入る。未指定時は `0`（後方互換）。
   - `CUDA_DEVICE_ORDER` は colmap.sh 内では触らず、呼び出し側でコマンドプレフィックスとして渡す。`VAR=val bash colmap.sh ...` 形式で渡した変数は bash の環境に入り、子プロセス（colmap コマンド）へ継承される。
   - **bare `python` 対策**: colmap.sh 内部の `python scripts/llff2colmap.py`（`:8`）・`python database.py`（`:16`）は bare `python` を呼ぶが、本環境に `python` は無い（`python3` のみ）。実行時に `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH"` を前置して bare `python` を `.venv/bin/python` に解決する（colmap.sh は5行目以外を改変しない）。`bash colmap.sh` の cwd はリポジトリルートとする（`scripts/llff2colmap.py`・`database.py` が相対参照のため）。
2. 空きGPU N 選択: `nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv` で N を1枚選ぶ。
3. 実行（`PATH` 先頭へ `.venv/bin` を前置して colmap.sh 内部の bare `python` を解決）:
   ```bash
   PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID \
   bash colmap.sh data/dynerf/cut_roasted_beef llff N
   ```
   - 内部手順（`colmap.sh:6-25`）: `sparse_`/`image_colmap` 削除 → `llff2colmap.py`（`poses_bounds.npy`→COLMAP テキスト、各カメラ `0000.png` を `image_colmap/r_XXX.png` にコピー）→ `colmap/` 構築 → `feature_extractor`（GPU SIFT）→ `database.py`（cameras 反映）→ `exhaustive_matcher` → `point_triangulator` → `image_undistorter` → `patch_match_stereo`（GPU MVS）→ `stereo_fusion` → `fused.ply`。
   - **COLMAP が処理する画像は各カメラ先頭フレームのみ（≈N枚）**。少数画像のため SfM+MVS は軽量。
   - バックグラウンド実行・ログ監視（GPU SIFT・MVS の進捗）。
4. 検証:
   - `ls -la data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply`（非空）。
   - `.venv/bin/python -c "from plyfile import PlyData; print(len(PlyData.read('.../fused.ply')['vertex']))"` が > 0。
   - 実行中に `nvidia-smi` で物理GPU N に COLMAP プロセスが乗ることを確認。
   - `git diff colmap.sh` が5行目1箇所のみ。

#### エラーハンドリング

- **bare `python` not found（最頻の即死要因）**: PATH 前置を忘れると `colmap.sh:8`/`:16` の `python` が `command not found` で失敗する。実行前に `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" command -v python` が `.venv/bin/python` を返すことを確認する。
- GPU SIFT が使えない（ヘッドレスで GL/CUDA 不整合）: feat-008 で GPU SIFT ヘッドレス動作は実証済み。万一失敗時は `--SiftExtraction.use_gpu 0` での CPU フォールバックを investigation に記録の上で検討（colmap.sh 改変が増えるため、まず GPU で実行）。
- `point_triangulator`/`patch_match_stereo` が点群を生成できない（特徴マッチ不足）: 各カメラ先頭フレームの視差が十分なら通常は成立。`fused.ply` が空/極小の場合は、抽出画像（FR-002）の妥当性とカメラ数を確認し investigation に記録。
- `colmap.sh` がエラーで途中終了: スクリプトは `set -e` を持たないため後続コマンドが走り続けうる。各 colmap コマンドの exit と最終 `fused.ply` の生成有無で判定する。

#### 境界条件

- 再実行: `colmap.sh` は冒頭で `sparse_`/`image_colmap`/`colmap` を削除してから再構築するため冪等。
- GPU 引数省略（`bash colmap.sh ... llff`）: 従来どおり GPU0 を使う（後方互換を維持）。

### 1.4.4 FR-004: ダウンサンプル（downsample_point.py）

#### データフロー

- 入力: `data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply`。
- 出力: `data/dynerf/cut_roasted_beef/points3D_downsample2.ply`（0 < 点数 ≤ 40,000）。

#### 処理ロジック

1. 実行:
   ```bash
   .venv/bin/python scripts/downsample_point.py \
     data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply \
     data/dynerf/cut_roasted_beef/points3D_downsample2.ply
   ```
2. 内部挙動（`downsample_point.py`）: `voxel_size=0.02` から開始し、点数が 40,000 を超える間 `voxel_down_sample` を繰り返し `voxel_size += 0.01`。40,000 点以下で `write_point_cloud`。
3. 検証: `points3D_downsample2.ply` を `PlyData.read` し `0 < 点数 ≤ 40000`。

#### エラーハンドリング

- `fused.ply` が無い: FR-003 未完。前段の完走を前提とする。
- 点数が 40,000 を大きく下回る（fused が小さい）: 学習自体は走る。極端に少ない（例 < 数百点）場合は COLMAP 品質を疑い investigation に記録。

#### 境界条件

- `fused.ply` が既に 40,000 点以下: voxel ループに入らずそのまま書き出される（許容）。

### 1.4.5 FR-005: DyNeRF（cut_roasted_beef）学習の完走

#### データフロー

- 入力: `data/dynerf/cut_roasted_beef/`（cam*.mp4・camXX/images・poses_bounds.npy）＋ `points3D_downsample2.ply` ＋ `arguments/dynerf/cut_roasted_beef.py`。
- 出力: `output/dynerf/cut_roasted_beef/point_cloud/iteration_14000/`（`point_cloud.ply`・deformation の `.pth` 群）、`output/dynerf/cut_roasted_beef/cfg_args`。

#### 処理ロジック

1. 空きGPU N（FR-003 と同一が望ましいが、学習は別タイミングのため再確認）・空きポート P（`ss -ltn | grep :<P>` が空）を選ぶ。
2. MPL scratch 準備と事前確認（train は `gaussian_model`→`matplotlib.pyplot` を top-level import するため。**本番と同一の環境変数**で確認）:
   ```bash
   mkdir -p <scratch>/{mplconfig,tmp}
   MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp \
   .venv/bin/python -c "import matplotlib.pyplot; print(matplotlib.get_backend().lower())"   # → agg（import 成功）
   ```
3. 実行（`MPLBACKEND=Agg`・`MPLCONFIGDIR`・`TMPDIR` を固定。train は reader 到達前の `import scene.gaussian_model`→`scene.regulation:5`〔`import matplotlib.pyplot`〕でフォントキャッシュ書込が要るため。MPL 未設定だと import 段階でクラッシュしうる）:
   ```bash
   MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp \
   CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N \
   .venv/bin/python train.py -s data/dynerf/cut_roasted_beef --port P \
     --expname "dynerf/cut_roasted_beef" --configs arguments/dynerf/cut_roasted_beef.py
   ```
   約20分目安のためバックグラウンド実行・ログ監視。
4. config 反映（設計根拠）: `cut_roasted_beef.py`（`_base_=default.py`、`batch_size=2`）→ `default.py` の `iterations=14000`・`coarse_iterations=3000`・`dataloader=True`・`densify_until_iter=10000` が有効。`maxtime=300`（`readdynerfInfo`）。
5. 保存イテレーションの根拠（feat-009 と同一機構）: `train.py` の `save_iterations.append(args.iterations)` は merge 前に実行され、その時点の `args.iterations` は OptimizationParams クラス既定 30000。`save_iterations` の最小は 14000。fine 段階（train_iter=14000）で 14000 に到達し保存。coarse 段階（train_iter=3000）は最小14000に未到達のため**保存されない**。結果 `iteration_14000` のみ生成。

#### エラーハンドリング

- `points3D_downsample2.ply` 欠落 → `readdynerfInfo` の `fetchPly` で例外（フォールバック無し）。FR-004 完了を前提とする。
- ポート使用中 → `network_gui` の `bind()` 例外が未処理で起動時クラッシュ（CLAUDE.md 既知事項）。手順1で必ず空きポート確認。
- GPU OOM/他者と競合 → 別の空きGPU を選び直す。`CUDA_DEVICE_ORDER=PCI_BUS_ID` 必須（index ズレ防止）。
- matplotlib 例外（`Matplotlib requires access to a writable cache directory`）→ train は `gaussian_model`→`matplotlib.pyplot` を top-level import するため、`MPLCONFIGDIR`（書込可能）と `MPLBACKEND=Agg` を必ず付与する。手順2の事前確認で import 成功（agg）を担保。
- DataLoader（`dataloader=True`）のワーカ起因エラー → ログを確認。多視点×300フレームのパス参照のみ（`__getitem__` で都度 `Image.open`、全画像のメモリ常駐はしない）のためメモリ逼迫は通常起きない。

#### 境界条件

- 学習が極端に遅い/速い: 目安20分。逸脱しても終了コード0・生成物ありで合格判定。
- `output/dynerf/cut_roasted_beef/` が既存: 再実行時は削除してから実行。

### 1.4.6 FR-006: レンダリングの完走

#### データフロー

- 入力: `output/dynerf/cut_roasted_beef/point_cloud/iteration_14000/`、`arguments/dynerf/cut_roasted_beef.py`。
- 出力: `test/ours_14000/{renders,gt}/*.png`、`video/ours_14000/renders/*.png`、`video/ours_14000/video_rgb.mp4`、`test/ours_14000/video_rgb.mp4`。

#### 処理ロジック

1. 実行（render も train と同様 `gaussian_model`→`matplotlib.pyplot` を top-level import するため MPL 設定を固定。`output.png` は生成されない〔plot 非呼び出し〕が import に MPL が要る）:
   ```bash
   MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp \
   CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N \
   .venv/bin/python render.py --model_path output/dynerf/cut_roasted_beef \
     --skip_train --configs arguments/dynerf/cut_roasted_beef.py
   ```
2. `render_sets` は `--skip_train` で train をスキップ、test（cam00 の300フレーム）と video（spiral 300フレーム）を生成。`--iteration -1` 既定で最大 14000 を自動選択。
3. `cam_type="dynerf"`（`scene.dataset_type`）で render に渡る。test の gt は `view.original_image` から保存。
4. 検証: `test/ours_14000/renders/` と `test/ours_14000/gt/` の PNG 数が**同数かつ 1 枚以上**で、ファイル名集合が完全一致（`.venv/bin/python` で `set(os.listdir(renders))==set(os.listdir(gt))` かつ `len>0`）。これにより FR-007 の空評価・NaN を未然に排除する。

#### エラーハンドリング

- test セットが空 → renders/gt が0枚になり metrics が `FileNotFoundError`。FR-006 検証で renders/gt 同数>0 を確認。
- 動画書き出し（imageio/ffmpeg）失敗 → `video_rgb.mp4` 非生成でも、metrics が使う `test/ours_14000/{renders,gt}` が揃えば FR-007 は可能。動画は视認用。

#### 境界条件

- test は cam00 の300フレーム想定（renders/gt 各300枚）。フレーム数が FR-002 で 300 未満だった場合はその数に一致する。

### 1.4.7 FR-007: 評価の完走

#### データフロー

- 入力: `output/dynerf/cut_roasted_beef/test/ours_14000/{renders,gt}/`。
- 出力: `output/dynerf/cut_roasted_beef/results.json`・`per_view.json`（method=`ours_14000`、6指標）。

#### 処理ロジック

1. 実行（metrics は Scene 非構築・matplotlib 非経由のため MPL 不要）:
   ```bash
   CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N \
   .venv/bin/python metrics.py --model_path output/dynerf/cut_roasted_beef/
   ```
2. `metrics.py:evaluate` は `test/` 配下の各 `ours_*` を列挙し、`renders/`・`gt/` から PSNR/SSIM/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM を算出。LPIPS 重みは feat-006 でDL済み・キャッシュ読込。
3. 検証: `results.json` の `ours_14000` の6指標すべてが**有限値**（`.venv/bin/python` で `all(math.isfinite(v))`）。`per_view.json` の各指標件数が `renders/` の PNG 数と一致。

#### エラーハンドリング

- `renders/`・`gt/` 欠落 → `FileNotFoundError`（`evaluate` が捕捉し `Unable to compute metrics for model <path>` 出力後に再送出・非0終了）。FR-006 完了が前提。
- 健全性目安（PSNR≥30、論文33.85基準）を外れる → クラッシュしなければ FR-007 自体は合格扱いとし、値の乖離は investigation に記録（バグ時は PSNR が大きく低下する。D-NeRF/HyperNeRF では論文値に近接した実績）。

#### 境界条件

- `test/` 配下に複数 `ours_*` が併存（再レンダ等）→ 両方が results.json に method 別で出る。本案件は `ours_14000` のみの想定。

### 1.4.8 FR-008: 文書化・依存記録・本体変更点記録

- `CLAUDE.md`:
  - ①「データセット」節の「Plenoptic / DyNeRF: フレーム抽出 + colmap 前処理が必要」を、本案件の確定手順（cut_roasted_beef、取得URL・前処理3段・train/render/metrics・MPL注意）に更新。
  - ②「オリジナルコードの変更点」節に `colmap.sh:5` の改変（`export CUDA_VISIBLE_DEVICES=0` → `${3:-0}`、理由＝マルチGPU運用の任意GPU選択、影響範囲＝前処理のみ）を追記。
  - ③「マルチGPU運用ルール」節に、COLMAP 前処理の GPU 指定方法（`PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh <dir> llff N`。**PATH 前置で colmap.sh 内部の bare `python` を `.venv` に解決**。これを欠くと `command not found` で即死）を追記。
- `docs/TECH_STACK.md`: DyNeRF（feat-010）節を追加（データ取得元・前処理パイプライン・cv2/imageio-ffmpeg の用途）。
- `requirements.lock.txt`: 新規ライブラリ導入が無ければ再生成不要（依存変化なし）。
- `docs/BACKLOG.md`: feat-010 を Closed に更新（手動テスト合格後）。

## 1.6 ファイル・ディレクトリ設計

```
data/dynerf/cut_roasted_beef/          # git 未追跡（.gitignore 管理外運用）
├── cam00.mp4 .. camNN.mp4             # FR-001（各カメラ動画）
├── poses_bounds.npy                   # FR-001（LLFF ポーズ、DyNeRF 判定キー）
├── cam00/images/0000.png..0299.png    # FR-002（フレーム抽出、全カメラ分）
├── cam01/images/...                   #   〃
├── sparse_/                           # FR-003（llff2colmap.py 出力: cameras/images/points3D.txt）
├── image_colmap/r_000.png..           # FR-003（各カメラ先頭フレーム）
├── colmap/                            # FR-003（COLMAP 作業領域）
│   ├── database.db
│   ├── sparse/0/                      #   point_triangulator 出力
│   └── dense/workspace/fused.ply      #   stereo_fusion 出力（ダウンサンプル元）
└── points3D_downsample2.ply           # FR-004（学習用点群、必須、≤40k点）

output/dynerf/cut_roasted_beef/        # 学習・レンダ・評価の出力（git 未追跡）
├── point_cloud/iteration_14000/       # point_cloud.ply + deformation*.pth
├── cfg_args
├── test/ours_14000/{renders,gt}/      # FR-006（評価対象、cam00 の300フレーム）
├── video/ours_14000/renders/          # FR-006（视認用、spiral 300フレーム）
├── results.json / per_view.json       # FR-007
└── ...

/data/sakagawa/tmp/feat010-dynerf/     # scratch（リポジトリ外・非コミット）
├── cut_roasted_beef.zip / extract/    # FR-001 作業
├── mplconfig/                         # MPLCONFIGDIR（train/render の matplotlib top-level import 用）
├── tmp/                               # TMPDIR
└── *.log                              # 長時間処理（DL/抽出/COLMAP/学習）のログ保存先

# 注: DyNeRF 経路は plot_camera_orientations を呼ばないため output.png は生成されない
#     （HyperNeRF/feat-009 との差異）。ただし train/render は gaussian_model→matplotlib.pyplot を
#     top-level import するため MPLCONFIGDIR/MPLBACKEND=Agg は必要（metrics/preprocess は不要）。
```

## 1.7 インターフェース定義（実行コマンド）

| 機能 | コマンド（GPU N・ポート P は実行直前に選択。`<scratch>`=`/data/sakagawa/tmp/feat010-dynerf`） |
|------|----------------------------------------------|
| FR-001 DL | `curl -L -o <scratch>/cut_roasted_beef.zip https://github.com/facebookresearch/Neural_3D_Video/releases/download/v1.0/cut_roasted_beef.zip` |
| FR-002 抽出 | `.venv/bin/python scripts/preprocess_dynerf.py --datadir data/dynerf/cut_roasted_beef` |
| FR-003 改変 | `colmap.sh:5` を `export CUDA_VISIBLE_DEVICES=${3:-0}` に変更（本体変更点） |
| FR-003 COLMAP | `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh data/dynerf/cut_roasted_beef llff N` |
| FR-004 ダウンサンプル | `.venv/bin/python scripts/downsample_point.py data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply data/dynerf/cut_roasted_beef/points3D_downsample2.ply` |
| FR-005 学習 | `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python train.py -s data/dynerf/cut_roasted_beef --port P --expname "dynerf/cut_roasted_beef" --configs arguments/dynerf/cut_roasted_beef.py` |
| FR-006 レンダ | `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python render.py --model_path output/dynerf/cut_roasted_beef --skip_train --configs arguments/dynerf/cut_roasted_beef.py` |
| FR-007 評価 | `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python metrics.py --model_path output/dynerf/cut_roasted_beef/`（MPL 不要） |

- 4DGS 本体の関数シグネチャ変更は行わない。`colmap.sh:5` の1行のみ改変（GPU 引数化、後方互換維持）。

## 1.8 ログ・デバッグ設計

- 長時間処理（FR-001 DL・FR-002 抽出・FR-003 COLMAP・FR-005 学習）はバックグラウンド実行し、`tee`/リダイレクトでログを scratch に保存して監視する。
- 学習/レンダのログは進捗バー（`\r`）で巨大化しうる。要点抽出に `grep -avE "it/s|it\]|findfont"` 等を用いる（feat-009 の知見）。
- COLMAP ログ: `feature_extractor`/`patch_match_stereo` の GPU 使用、`stereo_fusion` の `fused.ply` 出力点数を確認。
- 学習ログ: tqdm 進捗（PSNR・点数）と `[ITER N] Evaluating`、`save` メッセージで `iteration_14000` 生成を確認。
- レンダ: `point nums: <N>` が学習最終点数と一致することを確認。
- 評価: 6指標と `results.json` 生成を確認。値は health-check（PSNR≥30、論文33.85）と照合。
- DyNeRF 経路は `plot_camera_orientations` を呼ばないため `output.png` は生成されない（HyperNeRF/feat-009 との差異。生成された場合は経路の想定外として investigation に記録）。

## 2.4 設計判断の記録（ADR）

### ADR-1: 対象シーンは cut_roasted_beef

- **採用**: cut_roasted_beef（ユーザー選択、2026-06-22）。
- **却下**: sear_steak（README train_dynerf.sh の例）、その他4シーン。
- **理由**: BACKLOG・README の公式例であり、4DGS 論文 Table 4 に個別 PSNR **33.85dB** と定性記述（「最初のフレームの点群だけで大きな動きにも高忠実に追従」）があり、健全性チェックの基準値が明確。約1.06GB で他シーンと同程度。

### ADR-2: COLMAP は実走必須（事前生成点群なし）

- **採用**: `colmap.sh ... llff` を実走して点群を自前生成する。
- **背景**: README:157 は事前生成点群の提供を **HyperNeRF のみ**に明記し、「最初の2ステップ（preprocess・colmap）をスキップ可」とする。DyNeRF にはその記載がなく、配布物も動画＋ポーズのみで点群を含まない。
- **理由**: feat-009（HyperNeRF）は事前生成点群DLで COLMAP 実走を回避したが、DyNeRF は手段が無いため実走する。ただし **COLMAP が処理するのは各カメラ先頭フレームのみ（≈N枚）** で、HyperNeRF broom（約200枚）より大幅に軽く現実的（数分〜十数分目安）。

### ADR-3: colmap.sh の GPU0 ハードコードを引数化の最小改変で解消

- **採用**: `colmap.sh:5` を `export CUDA_VISIBLE_DEVICES=0` → `export CUDA_VISIBLE_DEVICES=${3:-0}`（第3引数で GPU 指定、未指定時 0）に1行改変。「オリジナルコードの変更点」に記録（FR-008）。呼び出しは `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh <dir> llff N`（`PATH` 前置で colmap.sh 内部の bare `python`〔`:8`・`:16`〕を `.venv/bin/python` に解決。本環境に bare `python` は無いため必須。これは環境変数のみで対応し colmap.sh の改変を5行目に限定するための措置）。
- **却下案A（コピー改変版を新規作成）**: `scripts/colmap_gpu.sh` 等の派生スクリプト。本体非改変を厳守できるが、オリジナルとの二重管理・乖離リスク。feat-011 でも別ファイル（`multipleviewprogress.sh`）が同問題を持つため、原本に最小改変を入れて記録する方が一貫する。
- **却下案B（GPU0 空き時に素実行）**: スクリプト非改変。GPU0 が常時使用中だと前処理が実行できず、マルチGPU運用ルール（任意GPU選択）と矛盾する。
- **理由**: スクリプト内 `export` が呼び出し側の環境変数を上書きするため、環境変数だけでは GPU≠0 を選べない。最小改変＋後方互換（引数省略時 0）が、変更量・運用一貫性・マルチGPU両立の最良バランス。COLMAP の GPU 依存は `feature_extractor`（SIFT）と `patch_match_stereo`（MVS）で、`CUDA_VISIBLE_DEVICES` で論理デバイス先頭＝物理 N に乗る（feat-007 で検証済みの挙動と同型）。**この改変は feat-011 では `multipleviewprogress.sh` の同種ハードコードに横展開できる（本案件のスコープは colmap.sh のみ）**。

### ADR-4: レンダは `--skip_train` のみ（test を残す）

- **採用**: `render.py --skip_train`（test + video を生成）。
- **却下**: `--skip_train --skip_test`（video のみ）。
- **理由**: 本案件は評価（FR-007）まで行うため test セットの renders/gt が必須。

### ADR-5: train/test split は eval_index=0 固定（コード仕様、非改変）

- **採用**: `readdynerfInfo` がローダに渡す `eval_index=0` をそのまま使う（**cam00 が test、残り全カメラが train**）。
- **背景**: DyNeRF コミュニティ慣行（中央カメラを held-out）。`readdynerfInfo:451-461` は `eval_index=0` をハードコードしており config からは変えられない。
- **理由**: 本体・config 非改変の方針。論文・公式スクリプトと同じ評価設定（cam00 held-out）で apples-to-apples の比較になる。

### ADR-6: train/render に MPL 環境変数を付与（matplotlib top-level import のため）。metrics/preprocess は不要

- **採用**: `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp` を train（FR-005）・render（FR-006）に付与。metrics（FR-007）・preprocess（FR-002）には付与しない。
- **背景（実コード確認）**: MPL が必要な理由は2つあり、本質は前者。
  1. **matplotlib の top-level import（本質）**: train/render は `import scene.gaussian_model`→`scene/regulation.py:5`（`import matplotlib.pyplot`）および `utils/scene_utils.py:4` で matplotlib を**モジュール読込時にロード**する。書込可能な `MPLCONFIGDIR` が無いと「Matplotlib requires access to a writable cache directory」で **import 段階クラッシュ**しうる（read-only/権限制約環境で再現）。ヘッドレスのため `MPLBACKEND=Agg`。
  2. reader 内の savefig（DyNeRF では非該当）: `plot_camera_orientations`（`dataset_readers.py:510`）は **HyperNeRF 専用 `readHyperDataInfos`（`:390`）でのみ呼ばれ**、DyNeRF の `readdynerfInfo`→`format_infos`（`:353-370`、plot 非呼び出し）では呼ばれない。よって DyNeRF では `output.png` は生成されないが、上記1により MPL は必要。
- **理由**: metrics.py・preprocess_dynerf.py は `gaussian_model`/`matplotlib` を import しない（`import` 一覧で確認）ため MPL 不要。**却下案（codex-01 を受けた MPL 全削除）**: reader が plot を呼ばないことのみに着目し MPL を全削除したが、top-level import で matplotlib が読まれるため train/render が import 段階で落ちる（codex-02 で検出）。よって train/render には付与する。
