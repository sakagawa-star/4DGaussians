# Backlog

## ロードマップ

4DGaussians（CVPR 2024, arXiv:2310.08528）が**正常に動作する環境を構築する**ことが現在の最終目標。
まずD-NeRF（合成シーン）で「学習 → レンダリング → 評価」が一通り動くことをゴールとし、段階的に積み上げて確認する。

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
| feat-006 | 評価動作確認 | `python metrics.py --model_paths output/dnerf/bouncingballs/`（正式な引数名は `--model_paths`/`-m`、必須・複数指定可。公式READMEは単数 `--model_path` と記載するが、argparseの前置一致で単数形も受理される）でPSNR/SSIM/LPIPS/MS-SSIM/D-SSIMが算出される | feat-005 | Open |

**判定基準（案）**:
- 前提: feat-005で `test` セットを生成済みで、`output/dnerf/bouncingballs/test/ours_20000/{renders,gt}` に画像が揃っていること。`metrics.py` は `test/` 配下のみを評価し、`test/` 直下の各 `ours_*` を全列挙して評価する（複数iterationを実行して `ours_14000`・`ours_20000` が併存する場合は両方が results.json にmethod別で出力される）
- 異常系: `test/`・`renders/`・対応する `gt/` 画像のいずれかが欠落すると `FileNotFoundError`。`evaluate()` は例外を捕捉して `Unable to compute metrics for model <path>` を出力後に再送出し、非0終了する
- 各指標（PSNR/SSIM/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM）が数値出力され、`output/dnerf/bouncingballs/results.json`（と `per_view.json`）が生成される
- 合格目安は論文（CVPR 2024）のD-NeRF報告値と大きく乖離しないこと。**具体的な合格閾値（例: PSNRの下限dB）はfeat-006着手時に論文値を参照して確定**する
- **この時点で「D-NeRFが正常動作する環境」の構築完了。**

---

## 対象外（当面）

| 項目 | 理由 |
|------|------|
| 実シーン（HyperNeRF / DyNeRF）の学習 | COLMAP導入が前提。まずD-NeRF合成シーンで動作確認する |
| SIBR_viewers による可視化 | 環境構築の必須要件ではない |
| マルチGPU分散学習 | 単一GPUで動作確認後に検討 |

---

## ステータス凡例

- **Open**: 未着手
- **In Progress**: 要求仕様・設計の作成中、またはレビュー・実装中
- **Closed**: 完了（手動テストで判定基準を満たし、ユーザー承認済み）
