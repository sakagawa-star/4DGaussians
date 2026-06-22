# feat-009 機能設計書: HyperNeRF動作確認（実シーン・単眼、broom2）

本書は `docs/DESIGN_STANDARD.md` に準拠する。設計書中のコードスニペットは「意図の伝達」が目的であり、そのままコピーして使うものではない。

---

## 1.1 対応要求マッピング

| 要求ID | 設計セクション |
|--------|----------------|
| FR-001 broom2 データ取得・配置 | §1.4.1 |
| FR-002 事前生成点群の取得・配置 | §1.4.2 |
| FR-003 学習完走 | §1.4.3 |
| FR-004 レンダリング完走 | §1.4.4 |
| FR-005 評価完走 | §1.4.5 |
| FR-006 文書化・依存記録更新 | §1.4.6 |

## 1.2 システム構成

本案件は**環境・データ整備＋既存コードの実行**であり、4DGS 本体の改変は行わない。関与する既存モジュールと役割は以下。

| ファイル/モジュール | 役割 | 本案件での関わり |
|---------------------|------|------------------|
| `scene/__init__.py:45-63` | データセット種別判定（`dataset.json` 存在 → `nerfies`） | broom2 が HyperNeRF 経路に入る判定（`dataset.json` を置く） |
| `scene/dataset_readers.py:373-400` `readHyperDataInfos` | HyperNeRF SceneInfo 構築。`Load_hyper_data` を ratio=0.5 で生成、`points3D_downsample2.ply` を読む | 必須 ply の配置先を規定 |
| `scene/dataset_readers.py:510-534` `plot_camera_orientations` | 読み込み時に呼ばれ matplotlib で `output.png` を CWD に savefig | ヘッドレス（backend=agg）で動作。CWD に `output.png` を生成（副作用） |
| `scene/hyper_loader.py:37-182` `Load_hyper_data` | scene/metadata/dataset.json・camera/・rgb/2x を読み、train/test 分割・時間正規化 | データ構造の要件（rgb/2x 必須）を規定 |
| `arguments/hypernerf/broom2.py` | broom2 学習 config（`_base_=default.py`、kplanes 時間解像度100） | そのまま使用（非改変） |
| `arguments/hypernerf/default.py` | HyperNeRF 既定（fine 14000・batch 2・coarse 3000） | broom2 が継承 |
| `train.py` / `render.py` / `metrics.py` | 学習・レンダ・評価のエントリポイント | コマンド実行（非改変） |
| `scripts/downsample_point.py` | COLMAP点群のダウンサンプル | **本案件では使わない**（事前生成点群DLのため） |
| `colmap.sh` | COLMAP 前処理 | **本案件では使わない**（ADR-1・ADR-5） |

### モジュール依存（HyperNeRF 読み込みフロー）

```
Scene.__init__ (scene/__init__.py:55, dataset.json 判定)
  └─ sceneLoadTypeCallbacks["nerfies"] = readHyperDataInfos (dataset_readers.py:373)
       ├─ Load_hyper_data(datadir, ratio=0.5, split="train"/"test")  (hyper_loader.py:37)
       │     reads: scene.json / metadata.json / dataset.json / camera/{id}.json / rgb/2x/{id}.png
       ├─ format_hyper_data → CameraInfo[]  (hyper_loader.py:184)
       ├─ fetchPly("points3D_downsample2.ply")  (dataset_readers.py:384-385)  ← 必須
       └─ plot_camera_orientations(...) → ./output.png  (dataset_readers.py:390,510)
```

## 1.3 技術スタック

- **言語**: Python 3.10（uv 管理 `.venv`）。
- **既存ライブラリ**: torch 1.13.1+cu116 / mmcv 1.6.0 / numpy 1.23.5 / matplotlib 3.10.9（backend=agg）/ tqdm 4.67.3 / open3d 0.19.0 / scikit-image 0.22.0 / imageio 2.37.3 / plyfile（fetchPly 用、導入済み）。CUDA 拡張（`diff_gaussian_rasterization`・`simple_knn`）は feat-002 ビルド済み。
- **新規ライブラリ**: **gdown**（Google Drive DL）。`uv pip install gdown`（追加的）。選定理由: Google Drive の大容量ファイルは確認トークン（virus scan 警告ページ）を挟むため単純な curl では取得しづらく、gdown はこれを自動処理する。NeRF 系リポジトリで広く使われる標準ツール。
- **パッケージ管理**: uv（`uv pip install` の追加運用のみ。`uv sync`/`uv pip sync` 禁止）。導入後 `requirements.lock.txt` を `uv pip freeze` で再生成。
- **外部データ取得**: GitHub Release（curl/wget）、Google Drive（gdown）。

## 1.4 各機能の詳細設計

> 全コマンドは `.venv/bin/python` を明示使用し bare `python` に依存しない。GPU を使うコマンド（train/render/metrics）は `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N` を付け、train/render/metrics で同一 N を使う。作業用の一時DL/展開は scratch `/data/sakagawa/tmp/feat009-hypernerf/`（リポジトリ外）で行い、確定物のみ `data/hypernerf/virg/broom2/` に置く。

### 1.4.1 FR-001: broom2 データセット取得・配置

#### データフロー

- 入力: `vrig_broom.zip`（1.5GB、GitHub Release v0.1）。
- 中間: scratch に展開した HyperNeRF/Nerfies 構造のディレクトリ（トップ階層名は要確認。HyperNeRF v0.1 の zip は scene ごとのフォルダを内包する）。
- 出力: `data/hypernerf/virg/broom2/`（`dataset.json`/`metadata.json`/`scene.json`/`camera/`/`rgb/2x/` を含む）。

#### 処理ロジック（手順）

1. scratch 準備: `mkdir -p /data/sakagawa/tmp/feat009-hypernerf`。
2. DL: `curl -L -o /data/sakagawa/tmp/feat009-hypernerf/vrig_broom.zip https://github.com/google/hypernerf/releases/download/v0.1/vrig_broom.zip`（大容量のためバックグラウンド実行・ログ監視）。
3. 展開: `unzip -q vrig_broom.zip -d /data/sakagawa/tmp/feat009-hypernerf/extract`。
4. **構造確認**: 展開トップを `ls -R` で確認し、`dataset.json`・`metadata.json`・`scene.json`・`camera/`・`rgb/` を含むディレクトリ（以下「シーンルート」）を特定する。`rgb/` 配下に `2x/` が存在することを必ず確認する（無ければ FR-001 を満たせない＝境界条件参照）。
5. 配置: シーンルートを `data/hypernerf/virg/broom2/` として配置する（`mkdir -p data/hypernerf/virg` の上で `cp -r <シーンルート> data/hypernerf/virg/broom2`、またはシーンルート名が `broom` 等であれば `broom2` にリネームして配置）。
6. 検証:
   - `ls data/hypernerf/virg/broom2/{dataset.json,metadata.json,scene.json}` が3つとも存在。
   - `ls data/hypernerf/virg/broom2/rgb/2x/*.png | head` が1枚以上。
   - `camera/` の JSON 数と `dataset.json.ids` 数が整合（`.venv/bin/python` で `len(json["ids"])` と `ls camera/*.json | wc -l` を比較）。

#### エラーハンドリング

- DL 失敗（ネットワーク/サイズ不足）: ファイルサイズが 1.5GB 前後でなければ再取得。`curl -L`（リダイレクト追従）必須。
- `rgb/2x/` が無い: HyperNeRF の一部配布は解像度違いを含む。2x が無い場合は本案件をブロックし investigation に記録（ratio=0.5 はコード固定のため 2x が必須）。
- 配置後に `dataset.json` 等が見つからない: 展開構造のネスト（scene フォルダの入れ子）を `ls -R` で再確認し、正しいシーンルートを配置し直す。

#### 境界条件

- 既に `data/hypernerf/virg/broom2/` が存在する場合: 上書き前に内容を確認し、不完全なら削除してから再配置。
- ディスク: 展開で数GB 消費。`/data`（15TB 空き）に置く。scratch は完了後に削除可。

### 1.4.2 FR-002: 事前生成点群の取得・配置

#### データフロー

- 入力: Google Drive file id `1fUHiSgimVjVQZ2OOzTFtz02E9EqCoWr5`（README 157行目の「Pregenerated point clouds」）。
- 中間: gdown でDLしたファイル（zip 想定。HyperNeRF 各シーンの `points3D_downsample2.ply` を内包すると想定）。
- 出力: `data/hypernerf/virg/broom2/points3D_downsample2.ply`。

#### 処理ロジック（手順）

1. gdown 導入: `uv pip install gdown`。確認: `.venv/bin/python -c "import gdown; print(gdown.__version__)"`。
2. DL: `.venv/bin/gdown 1fUHiSgimVjVQZ2OOzTFtz02E9EqCoWr5 -O /data/sakagawa/tmp/feat009-hypernerf/hypernerf_points.zip`（gdown CLI。`--fuzzy` や URL 直指定でも可）。
3. 構造確認: zip なら展開し `ls -R`、単一 ply ならそのまま。broom/broom2 に対応する `points3D_downsample2.ply` を特定する。
4. 配置: 特定した ply を `data/hypernerf/virg/broom2/points3D_downsample2.ply` にコピーする。
5. 検証（必須・空ファイル誤合格防止）:
   ```python
   # 意図の伝達用スニペット（実装ではワンライナーで可）
   from plyfile import PlyData
   n = len(PlyData.read("data/hypernerf/virg/broom2/points3D_downsample2.ply")["vertex"])
   assert n > 0
   ```
   さらに `scene.dataset_readers.fetchPly` 相当（x/y/z・red/green/blue・nx/ny/nz を要求）で読めることを確認する。

#### エラーハンドリング

- gdown が確認トークンで失敗（大容量警告）: gdown は自動処理するが、失敗時は `gdown --fuzzy "https://drive.google.com/uc?id=1fUHiSgimVjVQZ2OOzTFtz02E9EqCoWr5"` を試す。
- DL物に broom2 用 ply が無い/破損: **フォールバック（ADR-1）として `colmap.sh data/hypernerf/virg/broom2 hypernerf` → `downsample_point.py` で自前生成**に切り替える（feat-008 で COLMAP は検証済み。ただし colmap.sh の `CUDA_VISIBLE_DEVICES=0` ハードコード対応が必要＝ADR-5。発生時は investigation に記録しユーザー承認の上で実施）。
- `fetchPly` が要求するプロパティ（normals 等）が無い ply: 4DGS の `points3D_downsample2.ply` は nx/ny/nz を含む。含まない場合は破損とみなし再取得 or フォールバック。

#### 境界条件

- 点数が 40,000 を大きく超える/極端に少ない: 4DGS は ≤40,000 を想定。極端な場合も学習自体は走るが、investigation に記録。

### 1.4.3 FR-003: HyperNeRF（broom2）学習の完走

#### データフロー

- 入力: `data/hypernerf/virg/broom2/`（FR-001）＋ `points3D_downsample2.ply`（FR-002）＋ `arguments/hypernerf/broom2.py`。
- 出力: `output/hypernerf/broom2/point_cloud/iteration_14000/`（`point_cloud.ply`・`deformation.pth`・`deformation_table.pth`・`deformation_accum.pth` 等）、`output/hypernerf/broom2/cfg_args`、`point_cloud/iteration_coarse_*`（生成されない想定）。

#### 処理ロジック

1. 空きGPU 選択: `nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv` で `memory.used` 小・`utilization.gpu` 低の N を1枚選ぶ。
2. 空きポート選択: `ss -ltn | grep :<P>` が空のポート P を選ぶ（既定 6009 以外でも可。例 6019）。
3. matplotlib 用 scratch ディレクトリ準備と事前確認（**本番と同一の環境変数**で確認する。bare 実行だと別 backend/別 cache を見てしまうため）:
   ```bash
   mkdir -p /data/sakagawa/tmp/feat009-hypernerf/{mplconfig,tmp}
   MPLBACKEND=Agg MPLCONFIGDIR=/data/sakagawa/tmp/feat009-hypernerf/mplconfig \
   TMPDIR=/data/sakagawa/tmp/feat009-hypernerf/tmp \
   .venv/bin/python -c "import matplotlib; print(matplotlib.get_backend().lower())"
   ```
   出力が `agg` であること。
4. 実行（`MPLBACKEND=Agg`・`MPLCONFIGDIR`・`TMPDIR` を固定。`plot_camera_orientations` がデータロード時に matplotlib を import し CWD に `output.png` を savefig するため、backend 不正・キャッシュ書込失敗でのクラッシュを防ぐ）:
   ```bash
   MPLBACKEND=Agg MPLCONFIGDIR=/data/sakagawa/tmp/feat009-hypernerf/mplconfig \
   TMPDIR=/data/sakagawa/tmp/feat009-hypernerf/tmp \
   CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N \
   .venv/bin/python train.py -s data/hypernerf/virg/broom2 --port P \
     --expname "hypernerf/broom2" --configs arguments/hypernerf/broom2.py
   ```
   長時間（約30分目安）のためバックグラウンド実行・ログ監視。
5. config 反映の確認（設計根拠）: `merge_hparams`（`utils/params_utils.py`）は処理順 `[Optimization, ModelHidden, ModelParams, Pipeline]` でフラットな args に `setattr` する。broom2.py の `ModelParams=dict(kplanes_config=...res100)` は ModelHiddenParams（base の res150）より**後**に適用され、最終的に時間解像度100が有効（ADR-4）。`iterations=14000`・`coarse_iterations=3000`・`batch_size=2` は default.py 由来。
6. 保存イテレーションの根拠: `train.py:415` で `save_iterations.append(args.iterations)` は **merge 前**に実行され、その時点の `args.iterations` は OptimizationParams クラス既定 30000。よって `save_iterations=[14000,20000,30000,45000,60000,30000]`。fine 段階（train_iter=14000）は 14000 で保存。coarse 段階（train_iter=3000）は最小14000に未到達のため**保存されない**。結果、`iteration_14000` のみ生成。

#### エラーハンドリング

- `points3D_downsample2.ply` 欠落 → `readHyperDataInfos` の `fetchPly` で例外（フォールバック無し）。FR-002 完了を前提とする。
- ポート使用中 → `network_gui` の `bind()` 例外が未処理で**起動時クラッシュ**（CLAUDE.md 既知事項）。手順2で必ず空きポート確認。
- GPU OOM/他者と競合 → 別の空きGPU を選び直す。`CUDA_DEVICE_ORDER=PCI_BUS_ID` 必須（index ズレ防止）。
- Pillow 等の既知非互換: D-NeRF で対処済み（`dataset_readers.py:287` `np.uint8`）。HyperNeRF 経路でも同関数を通るため追加対処不要の見込み（実行で確認）。

#### 境界条件

- 学習が極端に遅い/速い: 目安30分。逸脱しても終了コード0・生成物ありで合格判定。
- `output/hypernerf/broom2/` が既存: 再実行時は削除してから実行（成果物の混在回避）。

### 1.4.4 FR-004: レンダリングの完走

#### データフロー

- 入力: `output/hypernerf/broom2/point_cloud/iteration_14000/`、`arguments/hypernerf/broom2.py`。
- 出力: `test/ours_14000/{renders,gt}/*.png`、`video/ours_14000/renders/*.png`、`video/ours_14000/video_rgb.mp4`、`test/ours_14000/video_rgb.mp4`。

#### 処理ロジック

1. 実行（render も Scene を構築し `plot_camera_orientations` を通るため MPL 環境変数を固定）:
   ```bash
   MPLBACKEND=Agg MPLCONFIGDIR=/data/sakagawa/tmp/feat009-hypernerf/mplconfig \
   TMPDIR=/data/sakagawa/tmp/feat009-hypernerf/tmp \
   CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N \
   .venv/bin/python render.py --model_path output/hypernerf/broom2 \
     --skip_train --configs arguments/hypernerf/broom2.py
   ```
2. `render_sets`（`render.py:78`）は `--skip_train` で train をスキップ、test（`getTestCameras`）と video（`getVideoCameras`）を生成。`--iteration -1` 既定で最大 14000 を自動選択（`scene.loaded_iter`）。
3. `cam_type="nerfies"`（`scene.dataset_type`）で render に渡る。`render_set`（`render.py:46`）は name∈{train,test} のとき gt を `view.original_image[0:3]` から保存（PanopticSports 以外）。
4. 検証: `test/ours_14000/renders/` と `test/ours_14000/gt/` の PNG 数が**同数かつ 1 枚以上**で、ファイル名集合が完全一致することを確認（`.venv/bin/python` で `set(os.listdir(renders))==set(os.listdir(gt))` かつ `len>0`）。これにより FR-005 の空評価・NaN を未然に排除する。

#### エラーハンドリング

- test セットが空（i_test 不整合）→ renders/gt が0枚になり metrics が `FileNotFoundError`。FR-004 検証で renders/gt 同数>0 を確認。
- 動画書き出し（imageio/ffmpeg）失敗 → `video_rgb.mp4` 非生成でも、metrics が使う `test/ours_14000/{renders,gt}` が揃えば FR-005 は可能。動画は视認用。

#### 境界条件

- **参考スクリプト `scripts/train_hyper_virg.sh` は broom2 を `--skip_train --skip_test`（video のみ）でレンダする**が、本案件は**評価のため test を残す**ため `--skip_train` のみとする（ADR-3）。

### 1.4.5 FR-005: 評価の完走

#### データフロー

- 入力: `output/hypernerf/broom2/test/ours_14000/{renders,gt}/`。
- 出力: `output/hypernerf/broom2/results.json`・`per_view.json`（method=`ours_14000`、6指標）。

#### 処理ロジック

1. 実行:
   ```bash
   CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N \
   .venv/bin/python metrics.py --model_path output/hypernerf/broom2/
   ```
2. `metrics.py:evaluate` は `test/` 配下の各 `ours_*` を列挙し、`renders/` と `gt/` から PSNR/SSIM/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM を算出。`metrics.py` は Scene を構築せず（`readImages` で PNG を読むのみ）matplotlib を経由しないため、MPL 環境変数は不要。
3. LPIPS 重み（vgg/alex）は feat-006 でDL済み・キャッシュ読込（再DLなし）。
4. 検証: `results.json` の `ours_14000` の6指標すべてが**有限値**であること（`.venv/bin/python` で `all(math.isfinite(v) for v in 6指標)`）。`metrics.py` は画像0枚だと `torch.tensor([]).mean()` で NaN を出すため、NaN/Inf は不合格とする。`per_view.json` の各指標の件数が `renders/` の PNG 数と一致することも確認。

#### エラーハンドリング

- `renders/`・`gt/` 欠落 → `FileNotFoundError`（`evaluate` が捕捉し `Unable to compute metrics for model <path>` 出力後に再送出・非0終了）。FR-004 完了が前提。
- 健全性目安（PSNR≥18 / MS-SSIM≥0.60）を外れる → クラッシュしなければ FR-005 自体は合格扱いとし、値の乖離は investigation に記録（4DGS metrics はフル画像評価で論文の covisible マスク評価とは異なる）。

#### 境界条件

- `test/` 配下に複数 `ours_*` が併存（再レンダ等）→ 両方が results.json に method 別で出る。本案件は `ours_14000` のみの想定。

### 1.4.6 FR-006: 文書化・依存記録更新

- `docs/TECH_STACK.md`: gdown を追記（用途=Google Drive DL、選定理由、解決バージョン）。HyperNeRF データ取得手順の要点（vrig_broom URL・事前生成点群 file id・配置パス）を記す。
- `CLAUDE.md`: データセット節「HyperNeRF（実シーン）: colmap 前処理が必要」を「**取得・動作確認済み（feat-009）**。事前生成点群DL方式・配置パス・コマンド」に更新。必要なら本体変更点（無い見込み）を記録。
- `requirements.lock.txt`: `uv pip freeze` で再生成。
- `docs/BACKLOG.md`: feat-009 を Closed に更新（手動テスト合格後）。

## 1.6 ファイル・ディレクトリ設計

```
data/hypernerf/virg/broom2/        # git 未追跡（.gitignore 管理外運用）
├── dataset.json                   # ids / train_ids / val_ids（HyperNeRF 経路判定キー）
├── metadata.json                  # camera_id / warp_id（時間）
├── scene.json                     # near/far/scale/center
├── camera/{id}.json               # カメラパラメータ
├── rgb/2x/{id}.png                # ratio=0.5 で読む画像（2x 必須）
└── points3D_downsample2.ply       # 事前生成点群（FR-002、必須）

output/hypernerf/broom2/           # 学習・レンダ・評価の出力（git 未追跡）
├── point_cloud/iteration_14000/   # point_cloud.ply + deformation*.pth
├── cfg_args
├── test/ours_14000/{renders,gt}/  # FR-004（評価対象）
├── video/ours_14000/renders/      # FR-004（视認用）
├── results.json / per_view.json   # FR-005
└── ...

/data/sakagawa/tmp/feat009-hypernerf/   # scratch（リポジトリ外・非コミット）
├── vrig_broom.zip / extract/           # FR-001 作業
├── hypernerf_points.zip / ...          # FR-002 作業
├── mplconfig/                          # MPLCONFIGDIR（matplotlib キャッシュ。FR-003/004）
└── tmp/                                # TMPDIR（FR-003/004）

output.png                          # plot_camera_orientations が CWD に生成する副作用
                                    # （.gitignore の `output` は output/ ディレクトリのみ一致し
                                    #  output.png は未追跡で残る。コミットしない）
```

## 1.7 インターフェース定義（実行コマンド）

| 機能 | コマンド（GPU N・ポート P は実行直前に選択） |
|------|----------------------------------------------|
| FR-001 DL | `curl -L -o <scratch>/vrig_broom.zip https://github.com/google/hypernerf/releases/download/v0.1/vrig_broom.zip` |
| FR-002 gdown 導入 | `uv pip install gdown` |
| FR-002 DL | `.venv/bin/gdown 1fUHiSgimVjVQZ2OOzTFtz02E9EqCoWr5 -O <scratch>/hypernerf_points.zip` |
| FR-003 学習 | `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python train.py -s data/hypernerf/virg/broom2 --port P --expname "hypernerf/broom2" --configs arguments/hypernerf/broom2.py` |
| FR-004 レンダ | `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python render.py --model_path output/hypernerf/broom2 --skip_train --configs arguments/hypernerf/broom2.py` |
| FR-005 評価 | `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python metrics.py --model_path output/hypernerf/broom2/`（MPL 不要） |

（`<scratch>` = `/data/sakagawa/tmp/feat009-hypernerf`）

- 4DGS 本体の関数シグネチャ変更は行わない（非改変）。

## 1.8 ログ・デバッグ設計

- 長時間処理（FR-001 DL・FR-003 学習）はバックグラウンド実行し、`tee`/リダイレクトでログを scratch に保存して監視する。
- 学習ログ: tqdm 進捗（PSNR・点数）と `[ITER N] Evaluating test/train` が出る。`scene.save(iteration, stage)` の保存メッセージで `iteration_14000` 生成を確認。
- レンダ: `point nums: <N>` が学習最終点数と一致することを確認（feat-005 と同様の整合チェック）。
- 評価: 6指標と `results.json` 生成を確認。値は health-check（PSNR≥18 / MS-SSIM≥0.60）と照合。
- 副作用ファイル `output.png` がリポジトリルートに生成される点を運用注記（コミットしない）。

## 2.4 設計判断の記録（ADR）

### ADR-1: 点群は事前生成版DL（COLMAP 実走スキップ）

- **採用**: README 配布の事前生成 `points3D_downsample2.ply` を Google Drive からDLして使う。
- **却下**: `colmap.sh hypernerf` をフル実走して自前生成。
- **理由**: COLMAP 自体は feat-008 で疎再構成〜dense（south-building）を検証済み。broom の dense MVS は `hypernerf2colmap.py` が約200フレームに間引くとはいえ patch_match_stereo が数時間規模になり得る。また `colmap.sh:5` が `CUDA_VISIBLE_DEVICES=0` をハードコードしており共用サーバーの空きGPU運用と衝突する（ADR-5）。本案件の主眼は「HyperNeRF が学習〜評価で動くこと」であり、点群入手の最短経路（DL）を採る。**フォールバック**: DL物が使えない場合に限り colmap.sh 実走へ切替（FR-002 エラーハンドリング）。

### ADR-2: Google Drive DL に gdown を採用

- **採用**: gdown（`uv pip install`）。
- **却下**: curl/wget で確認トークンを手動処理。
- **理由**: Google Drive の大容量ファイルは virus-scan 確認ページを挟み、単純な curl では HTML が返る。gdown はトークンを自動処理し、NeRF 系で標準的。依存は `.venv` 内に閉じ sudo 不要。

### ADR-3: レンダは `--skip_train` のみ（test を残す）

- **採用**: `render.py --skip_train`（test + video を生成）。
- **却下**: 参考スクリプト `scripts/train_hyper_virg.sh` の `--skip_train --skip_test`（video のみ）。
- **理由**: 本案件は評価（FR-005）まで行うため test セットの renders/gt が必須。参考スクリプトは视認用 video 生成が目的で test を省くが、本案件の判定基準（metrics 完走）には test が要る。

### ADR-4: broom2.py の kplanes_config 配置は非改変で使う

- **採用**: `arguments/hypernerf/broom2.py` をそのまま使用。
- **背景**: broom2.py は `kplanes_config`（本来 ModelHiddenParams のキー）を `ModelParams` 配下に置く。一見すると効かないように見えるが、`merge_hparams` は param グループのラベルに依存せずフラットな args に `setattr` し、処理順が `ModelHiddenParams` →（後に）`ModelParams` のため、最終的に broom2 の時間解像度100が有効になる。
- **理由**: 本体・config の改変は方針外。実挙動として意図どおり（res100）動くため非改変で使う。実行時に `cfg_args` 等で反映を確認する。

### ADR-5: colmap.sh は本案件で実行しない

- **採用**: 本案件では `colmap.sh` を使わない。
- **理由**: ADR-1 のとおり点群はDLで入手。加えて `colmap.sh:5` の `export CUDA_VISIBLE_DEVICES=0` ハードコードは共用サーバーの任意GPU運用（CLAUDE.md ルール）と衝突する。この衝突への対処（colmap.sh 改変 or 手動GPU指定）は feat-010/011 で COLMAP 実走が必要になった時点で別途設計する（本案件のスコープ外）。
