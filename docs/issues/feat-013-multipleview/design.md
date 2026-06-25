# feat-013 multipleview動作確認 機能設計書

作成日: 2026-06-25
準拠: `docs/DESIGN_STANDARD.md`
ステータス: ドラフト（Codexレビュー前）
対象要求: `requirements.md`（FR-001〜FR-006・NFR-001〜006）

## 1. 概要

既存 DyNeRF データ `cut_roasted_beef`（20カメラ×300フレーム）を multipleview 形式へ再編成し、`multipleview` コードパス（`scene/__init__.py:61`→`readMultipleViewinfos`→`multipleview_dataset`）で前処理〜学習〜レンダリング〜評価まで完走させる。`multipleviewprogress.sh` は**非改変**とし、その処理を**個別コマンドの手動/ラッパー実行**で再現する（pip install を走らせず、依存は事前導入、COLMAP/GPU は環境変数・PATH で制御）。

## 2. 入力データ再編成（FR-001）

### 2.1 既存データ構造（実測）

`data/dynerf/cut_roasted_beef/`:
- カメラ20台: `cam00 cam01 cam02 cam03 cam05 cam06 … cam20`（**cam04 欠番**）
- 各カメラ: `cam??/images/0000.png … 0299.png`（300枚、1352×1014 RGB PNG）

### 2.2 multipleview が要求する構造（README + `scene/multipleview_dataset.py`）

```
data/multipleview/cut_roasted_beef/
  cam01/ frame_00001.jpg … frame_00300.jpg
  cam02/ frame_00001.jpg … frame_00300.jpg
  …
  cam20/ frame_00001.jpg … frame_00300.jpg
  （前処理後に sparse_/・points3D_multipleview.ply・poses_bounds_multipleview.npy が追加される）
```

**命名制約の根拠**（`scene/multipleview_dataset.py`）:
- `load_images_path`: `image_length = len(os.listdir(cam_folder/"cam01"))` → **`cam01` が必須**、全カメラ同数フレーム前提。
- `extr.name`（COLMAP が付ける画像名 `imageN.jpg`）から `number=name[5:-4]` → `cam"+number.zfill(2)`。`scripts/extractimages.py` は `sorted(os.listdir(scene))` の順に `image1.jpg, image2.jpg, …` を割り当てる。
  → **camフォルダは `cam01,cam02,…` の連番（欠番なし）**でなければ image⇄cam の対応が崩れる。DyNeRF の cam04 欠番をそのまま持ち込めない。
- フレームは `frame_+str(i+1).zfill(5)+".jpg"`（1始まり5桁、`.jpg` 固定）。

### 2.3 再編成マッピング（ADR-1）

DyNeRF の20カメラを sorted 順に **cam01〜cam20 の連番へ詰める**（欠番を解消）:

| DyNeRF（sorted） | → multipleview |
|------|------|
| cam00 | cam01 |
| cam01 | cam02 |
| cam02 | cam03 |
| cam03 | cam04 |
| cam05 | cam05 |
| cam06 | cam06 |
| … | …（以降は同番号） |
| cam20 | cam20 |

フレーム: `images/0000.png` → `frame_00001.jpg`、…、`images/0299.png` → `frame_00300.jpg`（index+1）。

- **D1 解決（PNG→JPG）**: PIL で**真の JPEG へ変換**（quality=95）。理由: コードが `.jpg` 名を要求し、形式を合わせるのが honest。JPEG は lossy だが本案件は動作確認（受入は破綻なし＋6指標有限）であり画質劣化は許容。元 PNG は保持（再編成は別ディレクトリへコピー）。
- 再編成は**使い捨ての準備スクリプト**（scratch 配下、本体外）で行う。`scripts/` には追加しない（NFR-001）。

### 2.4 再編成スクリプト（scratch、本体外）

`/data/sakagawa/tmp/feat013-multipleview/reorganize.py`（概略・確定版は実装時に置く）:
- 入力 `data/dynerf/cut_roasted_beef`、出力 `data/multipleview/cut_roasted_beef`
- `sorted(glob cam??)` を列挙し index i（0始まり）→ 出力 `cam{i+1:02d}`
- 各カメラの `images/NNNN.png` を `frame_{NNNN+1:05d}.jpg` として `Image.open(...).convert("RGB").save(..., quality=95)`
- 完了後、cam01 が存在し全カメラ300枚であることを assert（FR-001 検証）

## 3. 前処理（FR-002）— multipleviewprogress.sh 非改変での手動実行

`multipleviewprogress.sh` を実走せず、同じ処理を個別コマンドで実行する（NFR-001/002/003/004）。作業は**リポジトリルートで** `./colmap_tmp` を使う（スクリプトの相対パス前提に合わせる）。COLMAP は **3.11.1 を PATH 前置**、GPU は `CUDA_VISIBLE_DEVICES=N` で明示（GPU0 ハードコードに縛られない）。

### 3.1 事前準備（依存・LLFF）

- 依存（skimage 0.22.0 / scipy 1.15.3 / imageio 2.37.3）は **venv に導入済み**（実測確認）。→ スクリプトの `pip install scikit-image` は**実行不要**（NFR-002 を満たす。bare pip を走らせない）。
- LLFF は事前 clone（本体実行に `git clone` させない）:
  ```bash
  git clone https://github.com/Fyusion/LLFF.git /data/sakagawa/tmp/feat013-multipleview/LLFF
  ```

### 3.2 COLMAP（3.11.1・任意GPU）

`<N>` = 空きGPU。`PATH` に 3.11.1 と venv を前置。`multipleviewprogress.sh` の各 colmap 行をそのまま個別実行する:

```bash
export PATH="/data/sakagawa/opt/colmap-3.11/bin:/data/sakagawa/4DGaussians/.venv/bin:$PATH"
cd /data/sakagawa/4DGaussians
rm -rf ./colmap_tmp
# ① フレーム抽出（本体スクリプト reuse・各camの frame_00001 を image{i}.jpg へ）
.venv/bin/python scripts/extractimages.py multipleview/cut_roasted_beef
# ② SfM
colmap feature_extractor --database_path ./colmap_tmp/database.db --image_path ./colmap_tmp/images \
  --SiftExtraction.max_image_size 4096 --SiftExtraction.max_num_features 16384 \
  --SiftExtraction.estimate_affine_shape 1 --SiftExtraction.domain_size_pooling 1 \
  --SiftExtraction.use_gpu 0
colmap exhaustive_matcher --database_path ./colmap_tmp/database.db --SiftMatching.use_gpu 0
mkdir ./colmap_tmp/sparse
colmap mapper --database_path ./colmap_tmp/database.db --image_path ./colmap_tmp/images --output_path ./colmap_tmp/sparse
mkdir -p ./data/multipleview/cut_roasted_beef/sparse_
cp -r ./colmap_tmp/sparse/0/* ./data/multipleview/cut_roasted_beef/sparse_
# ③ 密点群（dense は GPU 使用 → CUDA_VISIBLE_DEVICES で任意GPU）
mkdir ./colmap_tmp/dense
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<N> \
  colmap image_undistorter --image_path ./colmap_tmp/images --input_path ./colmap_tmp/sparse/0 \
  --output_path ./colmap_tmp/dense --output_type COLMAP
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<N> \
  colmap patch_match_stereo --workspace_path ./colmap_tmp/dense --workspace_format COLMAP \
  --PatchMatchStereo.geom_consistency true
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<N> \
  colmap stereo_fusion --workspace_path ./colmap_tmp/dense --workspace_format COLMAP \
  --input_type geometric --output_path ./colmap_tmp/dense/fused.ply
# ④ downsample（本体スクリプト reuse）
.venv/bin/python scripts/downsample_point.py ./colmap_tmp/dense/fused.ply \
  ./data/multipleview/cut_roasted_beef/points3D_multipleview.ply
```

- **feature/matcher は CPU 固定**（`--SiftExtraction.use_gpu 0` / `--SiftMatching.use_gpu 0`）。`estimate_affine_shape`/`domain_size_pooling` 指定は元々 CPU SIFT へ自動フォールバックする（feat-011 と同じ）が、NFR-003（任意GPU運用・GPU0競合回避）と決定性のため明示的に CPU 固定する。入力20枚で CPU でも軽量。**GPU を使うのは dense（image_undistorter/patch_match_stereo/stereo_fusion）のみ**で、これは `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<N>` で任意GPUに乗せる。
- **D2 解決**: 経路は `mapper`（完全SfM、`point_triangulator` 非使用）で、feat-011 の rig 非互換（単一カメラ共有＋point_triangulator）とは別物。3.11.1 で問題なく通る想定。仮に失敗時は 3.12.6 も候補だが、まず 3.11.1（既定運用）で実走確認する。

### 3.3 LLFF poses（poses_bounds_multipleview.npy）

```bash
cd /data/sakagawa/4DGaussians
.venv/bin/python /data/sakagawa/tmp/feat013-multipleview/LLFF/imgs2poses.py ./colmap_tmp/
cp ./colmap_tmp/poses_bounds.npy ./data/multipleview/cut_roasted_beef/poses_bounds_multipleview.npy
rm -rf ./colmap_tmp
```

- **D3 解決**: `imgs2poses.py` は `./colmap_tmp/`（images/ + sparse/ あり）から poses_bounds.npy を生成。依存（numpy/scipy/skimage）は導入済み。LLFF は colmap 出力読取りのため GPU 不要。実走で py3 互換・出力 shape=(20,17) を検証する（**リスク R-1**）。

### 3.4 前処理の成果物検証（FR-002 受入）

- **COLMAP 幾何の健全性**（mapper 直後、sparse/0 に対して）:
  ```bash
  colmap model_analyzer --path ./colmap_tmp/sparse/0
  ```
  受入: ①`Registered images = 20`（20/20 全カメラ登録）②`Mean reprojection error < 1.5 px` ③`Points > 0`。いずれか未達なら CP2 失敗として investigation.md に記録（疎すぎ/登録不足/誤差過大の偽陽性を後段へ流さない）。
- `data/multipleview/cut_roasted_beef/sparse_/{cameras,images}.bin` 存在、images.bin の名が `imageN.jpg`
- `points3D_multipleview.ply` 点数 > 0
- `poses_bounds_multipleview.npy` shape=(20,17)

## 4. config 作成（FR-003 準備）

- **D4 解決**: `arguments/multipleview/cut_roasted_beef.py` を新規作成（既存 `default.py` の内容をそのまま複製）。理由: README が「dataset 名.py を作る」ワークフローを案内。config 追加は通常運用であり本体コード改変ではない（`arguments/dynerf/cut_roasted_beef.py` 等と同列）。CLAUDE.md ディレクトリ構成へ追記。
- 内容は `default.py`（iterations=15000・coarse 3000・kplanes resolution [64,64,64,150] 等）をそのまま採用。チューニング不要（動作確認）。

## 5. 学習・レンダリング・評価（FR-003/004/005）

`<N>` = 空きGPU、`<P>` = 空きポート（既定 6017）、`<scratch>` = `/data/sakagawa/tmp/feat013-multipleview`。train/render に MPL 環境変数を付与（NFR-006）。

**長時間処理の実行形（NFR-005・確定）**: 以下の長時間ステップは**バックグラウンド起動＋ログ/PID をファイルへ固定**する（会話に進捗を流さない）。`<scratch>/logs/`・`<scratch>/pids/` を CP0 で作成。対象と起動形:
- **CP2 dense**（`patch_match_stereo`。数十分規模）: `… > <scratch>/logs/cp2_dense.log 2>&1 & echo $! > <scratch>/pids/cp2_dense.pid`
- **CP3 学習**（`train.py`。15000iter、十数分〜規模）: `… > <scratch>/logs/cp3_train.log 2>&1 & echo $! > <scratch>/pids/cp3_train.pid`
- 監視は `grep -avE "it/s|it\]|findfont"` で進捗バー除外、終了は exit code / 成果物存在で判定。
- 短時間ステップ（再編成・feature/matcher/mapper/downsample/LLFF・render・metrics）は foreground 可だが、出力が多い場合は同様にログへリダイレクトする。

### 5.1 学習（FR-003）
```bash
MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp \
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<N> \
.venv/bin/python train.py -s data/multipleview/cut_roasted_beef --port <P> \
  --expname "multipleview/cut_roasted_beef" --configs arguments/multipleview/cut_roasted_beef.py
```
- 受入: exit 0、`output/multipleview/cut_roasted_beef/point_cloud/iteration_XXXXX/` 生成。保存 iteration は OptimizationParams/`--save_iterations` 既定に従う（実測で確認）。

### 5.2 レンダリング（FR-004）
```bash
MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp \
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<N> \
.venv/bin/python render.py --model_path output/multipleview/cut_roasted_beef \
  --skip_train --skip_video --configs arguments/multipleview/cut_roasted_beef.py
```
- **`--skip_video` を付与**（FR-004 は test レンダリングのみが対象）。multipleview の video は `get_video_cam_infos` の spiral 300 view 固定（`multipleview_dataset.py:63`）で、FR外かつ高負荷・失敗時に CP4 を巻き込むため除外する（`render.py:91` の `if not skip_video`）。
- 受入: exit 0、`test/ours_XXXXX/{renders,gt}` 生成・ファイル名集合一致。

### 5.3 評価（FR-005）
```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<N> \
.venv/bin/python metrics.py --model_paths output/multipleview/cut_roasted_beef/
```
- 受入（数値基準）: exit 0、6指標すべて有限かつ **PSNR ≥ 20.0 dB**／**SSIM・MS-SSIM ∈ [0,1]**／**D-SSIM ∈ [0,0.5]**／**LPIPS-vgg・LPIPS-alex は finite かつ ≥ 0**。`results.json`・`per_view.json` 生成。（held-out カメラ無し＝学習視点再構成のため通常は高PSNR。下限基準は破綻検知用）

## 6. チェックポイント計画（暴走防止・各CPで停止確認）

| CP | 内容 | 壊れない成果物 | 受入 |
|----|------|----------------|------|
| CP0 | 準備（空きGPU/ポート確定・LLFF clone・依存確認・scratch＋`logs/`/`pids/`/`mplconfig/`/`tmp/` 作成） | 確定パラメータ | GPU/port 空き、skimage import OK、LLFF clone 済、scratch 配下ディレクトリ作成済 |
| CP1 | データ再編成（FR-001） | `data/multipleview/cut_roasted_beef/camNN/frame_XXXXX.jpg` | cam01〜cam20 連番・各300枚・jpg |
| CP2 | COLMAP+LLFF 前処理（FR-002） | sparse_・points3D_multipleview.ply・poses_bounds_multipleview.npy | §3.4 検証通過（model_analyzer: 20/20登録・再投影誤差<1.5px・点数>0 含む） |
| CP3 | config 作成（FR-003準備）＋学習（FR-003） | `output/.../iteration_XXXXX/` | train exit 0 |
| CP4 | レンダリング（FR-004、`--skip_video`） | `test/ours_XXXXX/{renders,gt}` | render exit 0・名一致 |
| CP5 | 評価（FR-005） | results.json・per_view.json | metrics exit 0・§5.3 数値基準（PSNR≥20 等）達成 |
| CP6 | 文書化・クローズ（FR-006） | CLAUDE.md/BACKLOG/台帳 | 反映・commit |

各CP完了で進捗台帳（`execution-progress.md`）に1行追記＋commit。

## 7. ADR（設計判断）

- **ADR-1: cam を連番へ詰める再編成**。採用理由: §2.2 の image⇄cam 対応制約（欠番不可）。代替「cam04 を空で作る」は frame 不在で `image_length`/読込が破綻するため却下。
- **ADR-2: PNG→真JPEG変換（quality95）**。採用理由: `.jpg` 名要求と形式整合。代替「PNGバイトを.jpg名で流用」は PIL/COLMAP は読めるが形式偽装で不健全。lossy は動作確認上許容。
- **ADR-3: multipleviewprogress.sh 非改変・手動/ラッパー実行**（ユーザー決定）。採用理由: 本体改変ゼロ方針＋uv 方針（bare pip 禁止）。スクリプト内 `pip install`/`git clone`/GPU0 ハードコードを回避し、依存事前導入・LLFF事前clone・`CUDA_VISIBLE_DEVICES` 明示で代替。代替「スクリプト改変（pip→uv・GPU引数化）」は本体改変になり却下。**GPU引数化は将来案件へ残す**（feat-012 と同方針）。
- **ADR-4: 入力は既存 cut_roasted_beef 再編成**（ユーザー決定）。新規DL回避・最速。素材はDyNeRFと同一だが**コードパスが別**（multipleview_dataset）なので動作確認の目的を満たす。
- **ADR-5: COLMAP 3.11.1 を使用**。mapper 経路は rig 非互換と無関係だが、CLAUDE.md 既定運用に合わせ保守的に 3.11.1。3.12.6 切替は不要の見込み（実走で確認）。
- **ADR-6: config は cut_roasted_beef.py 新規（default.py 複製）**。README ワークフロー準拠。本体コード改変ではない（config 追加）。

## 8. リスクと対策

| ID | リスク | 対策 |
|----|--------|------|
| R-1 | LLFF `imgs2poses.py` の py3/依存非互換で poses 生成失敗 | 依存は導入済み。CP2 で実走・shape 検証。失敗時は investigation.md に記録し修正計画（依存版調整 or 最小パッチ＝本体外） |
| R-2 | COLMAP mapper が20視点で再構成失敗/疎 | DyNeRF 経路で同データの COLMAP 実績あり（feat-011）。失敗時は sparse の登録画像数を確認 |
| R-3 | 再編成の cam⇄image 対応ズレ | CP1 で cam01 存在・連番・枚数を assert。CP2 後 images.bin の名（imageN.jpg）と cam 対応を確認 |
| R-4 | ディスク増（jpg 約6000枚） | 元 PNG 保持のまま別ディレクトリ生成。容量を CP1 で確認（数GB想定、許容） |
| R-5 | multipleview の評価が低PSNR | held-out カメラ無し＝学習視点再構成のため通常は高PSNR。破綻（極端な低値/nan）のみ異常として扱う |

## 9. 本体改変の有無

- 原則ゼロ。`multipleviewprogress.sh` 非改変、`extractimages.py`/`downsample_point.py` は reuse（実行のみ）。
- **追加ファイル**: `arguments/multipleview/cut_roasted_beef.py`（config 追加＝通常運用、コード改変ではない）。CLAUDE.md ディレクトリ構成へ追記。
- やむを得ず LLFF 等に最小パッチが必要になった場合は investigation.md に記録（本体4DGSではなく外部 LLFF への対応として区別）。

## 10. 確定コマンド一覧（再掲・FR対応）

| FR | コマンド要旨 |
|----|------|
| FR-001 | scratch の reorganize.py（PNG→jpg・cam連番化） |
| FR-002 | §3.2 COLMAP（3.11.1・PATH前置・dense は CUDA_VISIBLE_DEVICES=N）＋ §3.3 LLFF imgs2poses |
| FR-003 | §5.1 train.py（MPL環境変数・--configs cut_roasted_beef.py） |
| FR-004 | §5.2 render.py --skip_train **--skip_video** |
| FR-005 | §5.3 metrics.py --model_paths |
| FR-006 | CLAUDE.md/BACKLOG/台帳更新・commit |
