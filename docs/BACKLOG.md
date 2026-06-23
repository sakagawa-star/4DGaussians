# Backlog

## ロードマップ

4DGaussians（CVPR 2024, arXiv:2310.08528）が**正常に動作する環境を構築する**ことが最終目標。

**Phase 0-5（D-NeRF合成シーンの「学習 → レンダリング → 評価」）は 2026-05-25 に完了**し、最小の動作確認が取れた。ここから「**4DGS全体が動く環境**」へ拡張する（2026-06-18 ロードマップ策定）。

全体ゴールは次の2点：
1. **実装済み4データセット系統すべて**で「学習 → レンダリング → 評価」が動く ― D-NeRF（合成・単眼〔済〕）＋ HyperNeRF（実・単眼）・DyNeRF（実・多視点）・multipleview（多視点）の**実シーン3系統**
2. **複数人共用GPUサーバーで任意の1GPUを選んで動かせる**（マルチGPU運用）

> **スコープ確定（2026-06-18）**: `dycheck`（本体未実装）と `full_eval.py`（静的シーン MipNeRF-360 等専用で動的シーン非対応）はゴールから**除外**。SIBR_viewers可視化・応用ツール（merge_many_4dgs / export_perframe_3DGS）も**当面除外**（必要時に追加案件化）。詳細は末尾「対象外」節。

各案件は `CLAUDE.md` の機能追加フロー（feat-XXX）に従い、要求仕様書・機能設計書を作成 → レビュー → ユーザー承認 → 実装 → 手動テストの順で進める。**実装前に必ずドキュメントを作成・保存すること。**

> 下表の feat-XXX は**計画段階の案件**。着手時に `docs/issues/feat-{number}-{slug}/` を作成し、`requirements.md` / `design.md` を作成する。

---

### Phase 0: Python環境・依存パッケージ構築

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-001 | uv環境構築・依存インストール | uv（Python 3.10）で仮想環境を作成し、`TECH_STACK.md` の確定方針（**torch 1.13.1+cu116** を `--index-url .../whl/cu116` で導入）に従って `requirements.txt` の依存（torch系 + mmcv 1.6.0 等）をインストールする | - | **Closed**（2026-05-21完了。torch 1.13.1+cu116 / cuda 11.6 / A100認識を確認。mmcvはsetuptools<81＋`--no-build-isolation`、numpyは1.23.5固定で対処） |

**判定基準（案）**: `import torch` が成功し `torch.cuda.is_available()` が `True`、かつ `torch.version.cuda` が文字列 `'11.6'` を返す。GPU（A100）が認識される。

---

### Phase 1: CUDA拡張ビルド

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-002 | サブモジュール初期化・CUDA拡張ビルド | `git submodule update --init --recursive` でサブモジュールを取得し、`depth-diff-gaussian-rasterization` と `simple-knn` をソースビルドする。**ビルド時は `CUDA_HOME=/usr/local/cuda-11.6` を上書き**する（グローバルの12.8のままだと、torch(cu116)とnvccのメジャー版不一致 12 vs 11 でビルドエラーまたは重大な互換性警告が生じうる） | feat-001 | **Closed**（2026-05-22完了。FR-001〜005達成。rasterizer/simple-knn を CUDA11.6・no-build-isolation で editable ビルドしimport確認。simple-knn は uv editable 向けに simple_knn/__init__.py へ import torch を追加。手動テスト合格） |

**判定基準（案）**: 以下を全て満たす。
- 事前確認: `/usr/local/cuda-11.6/bin/nvcc --version` が `release 11.6` を返す
- `import diff_gaussian_rasterization` と `import simple_knn._C` がエラーなく成功する（このimport名は4DGS本体コードの実利用と一致: `gaussian_renderer/__init__.py`・`scene/gaussian_model.py` で確認済み）

---

### Phase 2: データセット準備（D-NeRF）

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-003 | D-NeRFデータ準備 | D-NeRF合成シーン（bouncingballs等）をDLし `data/dnerf/` に配置。ディレクトリ構成をREADME通りに整える | feat-002 | **Closed**（2026-05-22完了。`data.zip`（246MB）をDropboxからDL→展開→`data/dnerf/` へ全8シーン（bouncingballs/hellwarrior/hook/jumpingjacks/lego/mutant/standup/trex）配置。bouncingballs整合性検証合格（train=150/test=20、全画像実在、Blender判定構成OK）。`data/` は.gitignore管理外でコミットせず。手動テスト合格） |

**判定基準（案）**: `data/dnerf/{scene}/` 配下に学習に必要なファイル（transforms, frames等）が揃っている。

---

### Phase 3: 学習動作確認

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-004 | D-NeRF学習動作確認 | `python train.py -s data/dnerf/bouncingballs --port 6017 --expname "dnerf/bouncingballs" --configs arguments/dnerf/bouncingballs.py` で学習が完走する。出力先は `--expname` から `./output/{expname}` と決まるため **`--expname` 指定が必須**（未指定だと `./output/` 直下に出力される） | feat-003 | **Closed**（2026-05-22完了。coarse3000+fine20000=計23000反復を約10分（A100×1）で完走。`output/dnerf/bouncingballs/point_cloud/iteration_20000/` に成果物（point_cloud.ply 6.9MB＋deformation*.pth）生成。ITER14000 test PSNR=39.84、最終点数27,769。実装中に発覚した Pillow 12.2.0 非互換を bug修正（`scene/dataset_readers.py:287` `np.byte`→`np.uint8`、investigation.md記録）。手動テスト合格） |

**判定基準（案）**: 学習がクラッシュせず完走し、fine 段階の `output/dnerf/bouncingballs/point_cloud/iteration_14000/` と `iteration_20000/` 配下に `point_cloud.ply` が生成される。
- `--configs arguments/dnerf/bouncingballs.py`（→ `_base_ = dnerf_default.py`）が `arguments/__init__.py` のクラスデフォルト `iterations=30000` を **20000** に、`coarse_iterations` を **3000** に上書きする。`--save_iterations` 既定は `[14000, 20000, 30000, 45000, 60000]`（実行時に `args.iterations`=20000 をappend）。fine 段階(20000)で到達するのは 14000・20000 のみ。coarse 段階は最大3000イテレーションで `save_iterations`（最小14000）に到達しないため `coarse_iteration_*` は**生成されない**
- チェックポイントもデフォルトでは生成されない。必要なら `--checkpoint_iterations <iter>` を付与する。iterationカウンタは**ステージごとに0から振り直される**（coarse: 0〜3000、fine: 0〜20000）ため、`<iter>` が `coarse_iterations`(=3000)以下なら coarse・fine 両ステージで到達し `output/dnerf/bouncingballs/` 直下に `chkpnt_coarse_<iter>.pth`・`chkpnt_fine_<iter>.pth` の両方が、3000を超える値なら fine のみ到達し `chkpnt_fine_<iter>.pth` のみが生成される（例: 公式READMEの `--checkpoint_iterations 200` は両方生成）

---

### Phase 4: レンダリング動作確認

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-005 | レンダリング動作確認 | `python render.py --model_path "output/dnerf/bouncingballs/" --skip_train --configs arguments/dnerf/bouncingballs.py` でレンダリング画像・動画が生成される | feat-004 | **Closed**（2026-05-25完了。約13秒（A100×1）で test 20枚＋video 160枚を完走、exit code 0。`test/ours_20000/{renders,gt}`各20枚・`video/ours_20000/renders`160枚・両セットの`video_rgb.mp4`（非空）を生成。iteration_20000を自動選択、`point nums: 27769`（feat-004最終と一致）。コード変更ゼロ・不具合なし（Pillow問題は再発せず）。目視確認①〜④合格。手動テスト合格） |

**判定基準（案）**: `--iteration` 未指定時は `point_cloud/` の最大iteration=20000が選択され、以下が生成・視認できる（`--skip_train` のため `train/` は生成されない）。
- `output/dnerf/bouncingballs/test/ours_20000/`: `renders/` 画像・`gt/` 画像・`video_rgb.mp4`
- `output/dnerf/bouncingballs/video/ours_20000/`: `renders/` 画像・`video_rgb.mp4`（`video` セットは `gt/` ディレクトリは作られるが画像は書き込まれない＝空。これは仕様）

---

### Phase 5: 評価動作確認

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-006 | 評価動作確認 | `python metrics.py --model_paths output/dnerf/bouncingballs/`（正式な引数名は `--model_paths`/`-m`、必須・複数指定可。公式READMEは単数 `--model_path` と記載するが、argparseの前置一致で単数形も受理される）でPSNR/SSIM/LPIPS/MS-SSIM/D-SSIMが算出される | feat-005 | **Closed**（2026-05-25完了。約62秒（A100×1）で test 20枚を評価完走、exit code 0。6指標算出: PSNR=40.68・SSIM=0.9943・LPIPS-vgg=0.0155・LPIPS-alex=0.0059・MS-SSIM=0.9954・D-SSIM=0.0023（論文 bouncingballs 値 PSNR 40.62/SSIM 0.9942/LPIPS 0.0155 とほぼ一致、特に LPIPS-vgg は論文と同値）。`output/dnerf/bouncingballs/{results,per_view}.json` 生成。LPIPS重み4ファイルは調査段階でDL済み・本番はキャッシュから読込（再DLなし）。コード変更ゼロ・不具合なし。手動テスト合格。**これをもってD-NeRF動作確認（学習→レンダリング→評価）の環境構築が完了**） |

**判定基準（案）**:
- 前提: feat-005で `test` セットを生成済みで、`output/dnerf/bouncingballs/test/ours_20000/{renders,gt}` に画像が揃っていること。`metrics.py` は `test/` 配下のみを評価し、`test/` 直下の各 `ours_*` を全列挙して評価する（複数iterationを実行して `ours_14000`・`ours_20000` が併存する場合は両方が results.json にmethod別で出力される）
- 異常系: `test/`・`renders/`・対応する `gt/` 画像のいずれかが欠落すると `FileNotFoundError`。`evaluate()` は例外を捕捉して `Unable to compute metrics for model <path>` を出力後に再送出し、非0終了する
- 各指標（PSNR/SSIM/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM）が数値出力され、`output/dnerf/bouncingballs/results.json`（と `per_view.json`）が生成される
- 合格目安は論文（CVPR 2024, arXiv:2310.08528 Table 6）のD-NeRF `bouncingballs` 報告値（PSNR 40.62/SSIM 0.9942/LPIPS 0.0155）と大きく乖離しないこと。**確定した合格閾値（feat-006で論文値・feat-004実測から確定）**: PSNR≥38dB / SSIM≥0.98 / LPIPS-vgg≤0.05 / LPIPS-alex≤0.03 / MS-SSIM≥0.98 / D-SSIM≤0.01。**実測は全てクリア**（PSNR 40.68 等、上のStatus欄参照）
- **この時点で「D-NeRFが正常動作する環境」の構築完了（2026-05-25 達成）。Phase 0-5 はこれで全て Closed。**

---

> **ここから「4DGS全体が動く環境」への拡張フェーズ（2026-06-18 ロードマップ策定）。** ゴール・スコープは本ファイル冒頭を参照。優先順は「Phase 6（マルチGPU運用）→ Phase 7（COLMAP）→ Phase 8〜10（実シーン3系統）」。各 feat-XXX は着手時に `docs/issues/feat-{number}-{slug}/` を作成する。
>
> **2026-06-23 番号繰り下げ**: 旧 feat-010（DyNeRF）の実装で COLMAP **3.12.6 の rig/frame 非互換**（`colmap.sh` の `point_triangulator`＋単一カメラ共有が `Check failed: existing_frame.RigId()==frame.RigId()` でクラッシュ）が判明したため、Phase 7 に **COLMAP 3.11.1 併設（feat-011）** を挿入し、**DyNeRF を feat-012・multipleview を feat-013 に繰り下げ**た（旧 feat-010 は**中止**、調査資産は feat-012 が継承）。

### Phase 6: マルチGPU運用対応（任意GPU選択）

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-007 | マルチGPU運用対応 | 複数人共用GPUサーバーで、GPU0以外の空いている任意の1GPUを選んで学習・レンダリング・評価を動かせるようにする。**実機検証で `CUDA_VISIBLE_DEVICES=N`（N≠0）だけで物理GPU N に乗ることを確認済み（2026-06-18、コード修正不要）**。本案件は train/render/metrics の全経路がD-NeRFで GPU≠0 完走することの確認と、運用ルールの文書化が主眼 | feat-006 | **Closed**（2026-06-18 完了。D-NeRF bouncingballs を GPU index=1 / port 6107 で実機検証し、train→render→metrics の3経路が exit 0 完走〔PSNR 40.71・feat-006 と整合〕。FR-004 でプロセスが物理GPU1のみに乗り他GPUに漏れないことを uuid 照合で確認、FR-006 本体コード差分ゼロ。運用ルールを CLAUDE.md「マルチGPU運用ルール」節に明文化。`docs/issues/feat-007-multi-gpu/`） |

**判定基準（案）**: `CUDA_VISIBLE_DEVICES=<0以外の空きGPU>` を付けて train.py / render.py / metrics.py をD-NeRFで実行し、(1) 指定した物理GPUにのみ負荷が乗る（nvidia-smiで確認）、(2) 3経路ともクラッシュせず完走する、(3) 運用手順がCLAUDE.md等に明文化される。
- **背景（調査済み）**: `utils/general_utils.py:139` と `metrics.py:116-117` に `set_device(torch.device("cuda:0"))` のハードコードがあるが、`cuda:0` はCUDA_VISIBLE_DEVICESでマスクされた後の論理デバイス先頭を指すため物理GPU0固定にはならない。**2026-06-18 実機検証済み**: `CUDA_VISIBLE_DEVICES=5` 指定下で当該コードを再現し、プロセスが物理GPU5に乗ることを確認。よってコード変更は原則不要（分散学習機構=DataParallel/DDP等は本体に存在しない）

### Phase 7: COLMAP環境構築（実シーンの前提）

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-008 | COLMAP環境構築 | 実シーン全系統（HyperNeRF/DyNeRF/multipleview）の前提となるCOLMAPを本マシンに導入する。`convert.py`/`colmap.sh`/`multipleviewprogress.sh` が依存する `colmap` バイナリと、前処理の依存ライブラリ（open3d〔downsample_point.py〕等）を整備し、小規模データで動作確認する | feat-007 | **Closed**（2026-06-21、手動テスト合格）。導入方式=**vcpkg ソースビルド**〔`colmap[core,cuda]:x64-linux@3.12.6`、GUI除外・CUDA11.6有効〕。ビルドに gfortran 必須が判明し `sudo apt`〔管理者承認〕で導入。FR-001〜006 自己検証＋ユーザー手動テストとも合格: `colmap -h` 動作、`~/.local/bin/colmap` ラッパー、scikit-image 0.22.0 導入（open3d 0.19.0 維持）、south-building 18枚で疎再構成（登録18/18・3D点≈3755）・dense（fused.ply 約122万点）完走、GPU SIFT ヘッドレス動作実証。経緯・ADR・investigation は `docs/issues/feat-008-colmap/`）。**※2026-06-23 追記: 旧 feat-010（DyNeRF）の実装で、3.12.6 は 4DGS の `colmap.sh`（`point_triangulator`＋単一カメラ共有 sparse モデル）と rig/frame 非互換（`Check failed: existing_frame.RigId()==frame.RigId()`）と判明。feat-011 で rig 非ネイティブの 3.11.1 を併設する** |
| feat-011 | COLMAP 3.11.1 併設（rig非互換回避） | COLMAP **3.11.1**（rig 必須化前の最終版）を vcpkg で別 prefix にビルドし、4DGS の `colmap.sh`/`multipleviewprogress.sh` 系（既知ポーズ `point_triangulator`＋単一カメラ共有）を**本体非改変**で動かせるようにする。既存 3.12.6 は温存し、DyNeRF/multipleview 前処理時のみ 3.11.1 を使う | feat-008 | **Open**（2026-06-23 起票。旧 feat-010 の調査で rig 非互換が判明したため挿入。`docs/issues/feat-011-colmap-3.11/`） |

**判定基準（案・feat-008）**: `colmap --help`（または相当）が通る。`scripts/downsample_point.py`（open3d）が import エラーなく動く。小規模データでCOLMAP（feature_extractor〜mapper）が完走する。**本マシンはcolmap未インストール（CLAUDE.md実行環境表）のため、導入方法〔ソースビルド or 配布バイナリ〕の調査・選定が案件の主眼**

**判定基準（案・feat-011）**: COLMAP 3.11.1 が別 prefix にビルドされ、既存 3.12.6 を壊さず併存する。3.11.1 の colmap で 4DGS の `colmap.sh ... llff`（旧 feat-010 の cut_roasted_beef データ）が `point_triangulator`〜`stereo_fusion` まで完走し `fused.ply`（点数>0）を生成する（rig エラーが出ない）。`downsample_point.py` で `points3D_downsample2.ply`（≤40k点）まで通る。

### Phase 8: HyperNeRF動作確認（実シーン・単眼）

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-009 | HyperNeRF動作確認 | 実シーン・単眼の1シーン（例: broom2）で、前処理（`colmap.sh ... hypernerf`＋`downsample_point.py`、または事前生成COLMAP点群のDL）→学習→レンダリング→評価を一通り動かす | feat-008 | **Closed**（2026-06-22、手動テスト合格）。方式=事前生成点群DL（gdown 新規導入）＋broom2。FR-001〜006 全達成: vrig_broom DL→学習17分→render test197枚→metrics **PSNR 22.08/MS-SSIM 0.691**〔論文 22.0/0.70 と一致〕。視覚的裏付けに chicken も実施（**PSNR 28.65/MS-SSIM 0.930**〔論文 28.7/0.93〕、目視で鮮明と確認）。本体コード変更ゼロ。詳細は `docs/issues/feat-009-hypernerf/` |

**判定基準（案）**: `data/hypernerf/.../` 配下を整備し、`train.py`→`render.py`→`metrics.py` が完走する。データ形式は scene.json/metadata.json/dataset.json/rgb/camera 構造（`scene/dataset_readers.py:readHyperDataInfos`、判定は dataset.json の存在）

### Phase 9: DyNeRF動作確認（実シーン・多視点）

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| ~~feat-010~~ | DyNeRF動作確認（**中止**） | feat-008 の COLMAP 3.12.6 が rig 非互換と判明し中止（2026-06-23）。調査資産（cut_roasted_beef データDL・フレーム抽出済、root cause 分析、requirements/design）は feat-012 が継承 | feat-008 | **中止（Cancelled, 2026-06-23）**。`docs/issues/feat-010-dynerf/` 参照 |
| feat-012 | DyNeRF動作確認（再開） | 実シーン・多視点の1シーン（cut_roasted_beef）で、フレーム抽出（`preprocess_dynerf.py`）→COLMAP（`colmap.sh ... llff`、**COLMAP 3.11.1 を使用**）→ダウンサンプリング→学習→レンダリング→評価を動かす。旧 feat-010 の調査資産（データ・root cause・確定事項）を継承 | feat-011 | **Open** |

**判定基準（案）**: 前処理3段（preprocess_dynerf.py / colmap.sh llff / downsample_point.py）が完走し、`train.py`→`render.py`→`metrics.py` が完走する。ffmpeg（imageio）依存に注意（`scene/dataset_readers.py:readdynerfInfo`、判定は poses_bounds.npy の存在）。**COLMAP は feat-011 で導入する 3.11.1 を使う**

### Phase 10: multipleview動作確認（多視点・カスタム）

| ID | Title | 概要 | 依存 | Status |
|----|-------|------|------|--------|
| feat-013 | multipleview動作確認 | 多視点カスタムデータで `multipleviewprogress.sh`（フレーム抽出→COLMAP→LLFFポーズ→ダウンサンプリング）→config作成→学習を動かす。COLMAP 実走は **3.11.1** を使う（同種の rig 非互換回避） | feat-011 | **Open** |

**判定基準（案）**: `multipleviewprogress.sh` が完走し `sparse_/`・`points3D_multipleview.ply`・`poses_bounds_multipleview.npy` が生成される。`arguments/multipleview/{name}.py` 作成の上 `train.py` が完走する。LLFF（scikit-image）依存に注意（`scene/dataset_readers.py:readMultipleViewinfos`、判定は points3D_multipleview.ply の存在）

---

## 対象外（スコープ外・2026-06-18 確定）

| 項目 | 理由 |
|------|------|
| dycheck データセット | **本体が未実装**（READMEにも `scene/__init__.py` のreaderにも対応経路なし、`arguments/dycheck/` はスケルトンのみ）。動かすには4DGS本体の改修が必要なため環境構築のスコープ外 |
| full_eval.py による一括評価 | **静的シーン専用**（MipNeRF-360 / Tanks&Temples / Deep Blending）で4DGSの動的シーンには非対応。本プロジェクトの目的（動的シーン）と合致しないため対象外 |
| SIBR_viewers による可視化 | 環境構築の必須要件ではない。CMakeビルド＋GUI/ポートフォワードが必要。必要になった時点で別途案件化 |
| 応用ツール（merge_many_4dgs / export_perframe_3DGS） | 学習済みモデルの合成・フレーム別点群書き出し。コア動作の確認後、必要なら案件化 |
| マルチGPU**分散**学習（複数GPU同時使用） | Phase 6（feat-007）の「任意の1GPU選択」とは別物。本体に分散機構（DataParallel/DDP等）の実装がなく、単一GPU運用で足りるため対象外 |
| D-NeRF他シーン横展開（lego/mutant等7シーン） | bouncingballsで論文値一致を確認済みのため必須ではない。データは feat-003 で取得済みで、必要なら低コストで追加可能（任意） |

---

## ステータス凡例

- **Open**: 未着手
- **In Progress**: 要求仕様・設計の作成中、またはレビュー・実装中
- **Closed**: 完了（手動テストで判定基準を満たし、ユーザー承認済み）
