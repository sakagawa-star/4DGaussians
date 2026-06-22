# feat-009 要求仕様書: HyperNeRF動作確認（実シーン・単眼、broom2）

本書は `docs/REQUIREMENTS_STANDARD.md` に準拠する。

---

## 1.1 プロジェクト概要

- **何を作るのか**: HyperNeRF 実シーン・単眼の1シーン（**broom2**）で、4DGaussians の「学習（`train.py`）→ レンダリング（`render.py`）→ 評価（`metrics.py`）」が完走する状態を構築・確認する。
- **なぜ作るのか**: 最終目的「4DGS全体が動く環境」のうち、**実シーン3系統の1つ目（HyperNeRF＝実・単眼）**の動作確認。D-NeRF（合成・単眼）に続き、実写・単眼系統が動くことを実証する。
- **誰が使うのか**: 本リポジトリで 4DGS を動かす開発者（複数人共用 GPU サーバー利用）。
- **どこで使うのか**: Ubuntu 22.04.5 / A100-SXM4-40GB ×7（共用）/ uv 管理 `.venv`（Python 3.10, torch 1.13.1+cu116）/ ヘッドレス。

## 1.2 用語定義

ドキュメント内・機能設計書・コードで同じ用語を使う。

- **HyperNeRF**: 実写・単眼の動的シーンデータセット（Google, ECCV 2021）。本案件は検証リグ（vrig）系の **broom2** を用いる。
- **vrig（virg）**: validation rig。HyperNeRF の評価用に複数カメラを持つ撮影系統。リリース資産名は `vrig_*.zip`。4DGS のディレクトリ名は README に従い `virg`（綴りは README/config 準拠）。
- **事前生成点群**: 4DGS 作者が COLMAP で生成・ダウンサンプル済みの学習用点群 `points3D_downsample2.ply`。README 157行目の Google Drive リンクで配布。
- **`points3D_downsample2.ply`**: HyperNeRF 学習で**必須**の点群ファイル（`scene/dataset_readers.py:384`）。≤40,000 点を想定。
- **ratio=0.5**: HyperNeRF ローダが固定で用いる画像縮小率（`readHyperDataInfos` が `Load_hyper_data(datadir, 0.5, ...)` を呼ぶ）。`int(1/0.5)=2` のため画像は `rgb/2x/` から読む。
- **coarse / fine 段階**: 4DGS 学習の2段階。coarse は静的初期化、fine は変形フィールド学習。broom2 では coarse=3000・fine=14000 反復。
- **gdown**: Google Drive からファイルをDLする Python ユーティリティ。大容量ファイルの確認トークンを自動処理する。

## 1.3 機能要求一覧

### FR-001: broom2 データセット（画像・カメラ・メタ）の取得と配置

- **概要**: HyperNeRF v0.1 リリースの `vrig_broom.zip`（1.5GB）を取得・展開し、4DGS が読める構造で `data/hypernerf/virg/broom2/` へ配置する。
- **入力**: `https://github.com/google/hypernerf/releases/download/v0.1/vrig_broom.zip`（curl/wget で取得）。
- **出力**: `data/hypernerf/virg/broom2/` 配下に、HyperNeRF ローダが要求する以下が揃った状態。
  - `dataset.json`, `metadata.json`, `scene.json`（各1ファイル）
  - `camera/{id}.json`（各フレームのカメラパラメータ）
  - `rgb/2x/{id}.png`（ratio=0.5 で読む画像。**2x が必須**）
- **受け入れ基準**:
  - 上記5種（`dataset.json`/`metadata.json`/`scene.json`/`camera/` ディレクトリ/`rgb/2x/` ディレクトリ）がすべて存在する。
  - `rgb/2x/` に PNG が1枚以上あり、`camera/` の JSON 数と `dataset.json` の `ids` 数が整合する。
  - `data/hypernerf/` はリポジトリにコミットしない（`data/` は .gitignore 管理外＝git 未追跡で運用、feat-003 と同方針）。

### FR-002: 事前生成点群（points3D_downsample2.ply）の取得と配置

- **概要**: 事前生成COLMAP点群を Google Drive（README 157行目、file id `1fUHiSgimVjVQZ2OOzTFtz02E9EqCoWr5`）からDLし、broom2 用の `points3D_downsample2.ply` を `data/hypernerf/virg/broom2/` へ配置する。DL には gdown を新規導入する。
- **入力**: gdown（`uv pip install gdown`、追加的導入）。Google Drive file id `1fUHiSgimVjVQZ2OOzTFtz02E9EqCoWr5`。
- **出力**: `data/hypernerf/virg/broom2/points3D_downsample2.ply`（COLMAP 由来・ダウンサンプル済み点群）。
- **受け入れ基準**:
  - gdown が `.venv` に追加導入され、`.venv/bin/python -c "import gdown"` が成功する（既存依存非破壊：導入後も `.venv/bin/python -c "import torch; print(torch.cuda.is_available())"` が `True`、numpy が 1.23.5 のまま）。
  - `data/hypernerf/virg/broom2/points3D_downsample2.ply` が存在し、**`scene.dataset_readers.fetchPly` で読めて点数 > 0**（`.venv/bin/python` で `PlyData.read` し頂点数を確認。空・破損ファイルは不合格）。
  - DLした版が broom2 用であること（4DGS 配布構造に従い broom/broom2 に対応する ply）を確認した上で配置する。

### FR-003: HyperNeRF（broom2）学習の完走

- **概要**: `train.py` を broom2 の config で実行し、coarse 3000 + fine 14000 反復が完走して学習済み点群が生成されることを確認する。マルチGPU運用ルールに従い空きGPU・空きポートを選ぶ。
- **入力**: コマンド（matplotlib は読み込み時に必ず import されるため `MPLBACKEND=Agg` と書込可能な `MPLCONFIGDIR` を固定する）
  `MPLBACKEND=Agg MPLCONFIGDIR=/data/sakagawa/tmp/feat009-hypernerf/mplconfig TMPDIR=/data/sakagawa/tmp/feat009-hypernerf/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<空きN> .venv/bin/python train.py -s data/hypernerf/virg/broom2 --port <空きP> --expname "hypernerf/broom2" --configs arguments/hypernerf/broom2.py`
- **出力**: `output/hypernerf/broom2/point_cloud/iteration_14000/`（`point_cloud.ply` と deformation の重み `.pth` 群）。
- **受け入れ基準**:
  - プロセスが終了コード 0 で完走する（クラッシュしない）。
  - データロード時に matplotlib 由来の例外（backend 不正・キャッシュ書込失敗）でクラッシュしない。事前確認は**本番と同一の環境変数**で行う（別 backend/別 cache を確認してしまわないため）: `mkdir -p /data/sakagawa/tmp/feat009-hypernerf/{mplconfig,tmp}` の上で `MPLBACKEND=Agg MPLCONFIGDIR=/data/sakagawa/tmp/feat009-hypernerf/mplconfig TMPDIR=/data/sakagawa/tmp/feat009-hypernerf/tmp .venv/bin/python -c "import matplotlib; print(matplotlib.get_backend().lower())"` が `agg` を返すことを確認する。
  - `output/hypernerf/broom2/point_cloud/iteration_14000/point_cloud.ply` が生成される（非空）。
  - 指定した物理GPU N のみに負荷が乗る（`nvidia-smi` で確認）。
  - coarse 段階の点群（`coarse` イテレーション保存物）は生成されない想定（save_iterations 最小14000に coarse 3000 が未到達のため。生成有無は設計の根拠どおりであることを確認）。

### FR-004: レンダリングの完走（test + video セット）

- **概要**: `render.py` を `--skip_train` で実行し、test セットと video セットの画像・動画が生成されることを確認する（評価のため test を残す）。
- **入力**: コマンド（train と同様に MPL 環境変数を固定）
  `MPLBACKEND=Agg MPLCONFIGDIR=/data/sakagawa/tmp/feat009-hypernerf/mplconfig TMPDIR=/data/sakagawa/tmp/feat009-hypernerf/tmp CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<FR-003と同一N> .venv/bin/python render.py --model_path output/hypernerf/broom2 --skip_train --configs arguments/hypernerf/broom2.py`
- **出力**:
  - `output/hypernerf/broom2/test/ours_14000/{renders,gt}/`（各 PNG）
  - `output/hypernerf/broom2/video/ours_14000/renders/` と `video_rgb.mp4`
- **受け入れ基準**:
  - 終了コード 0 で完走する。
  - `test/ours_14000/renders/` と `test/ours_14000/gt/` の PNG 数が**同数かつ 1 枚以上**で、**両ディレクトリのファイル名集合が完全一致**する（空評価・対応欠落を排除。`ls` 比較または `.venv/bin/python` で `set(os.listdir(...))` 一致を確認）。
  - iteration は未指定（`--iteration -1`）で最大の 14000 が自動選択される。

### FR-005: 評価の完走（test セット6指標）

- **概要**: `metrics.py` を実行し、test セットに対して PSNR/SSIM/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM が算出されることを確認する。
- **入力**: コマンド
  `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=<FR-003と同一N> .venv/bin/python metrics.py --model_path output/hypernerf/broom2/`
- **出力**: `output/hypernerf/broom2/{results,per_view}.json`（method=`ours_14000` の6指標）。
- **受け入れ基準**:
  - 終了コード 0 で完走し、6指標すべてが数値出力される。
  - `results.json` に `ours_14000` の6指標が記録され、**6指標すべてが有限値（`math.isfinite` が True。NaN/Inf でない）**である（空評価による NaN を排除。`metrics.py` は画像0枚だと `torch.tensor([]).mean()` で NaN を生むため）。
  - `per_view.json` の `ours_14000` の各指標の件数が、`test/ours_14000/renders/` の PNG 数と一致する。
  - **健全性チェック（参考値）**: 4DGS 論文（CVPR 2024, broom）の PSNR 22.0 / MS-SSIM 0.70 を基準に、**PSNR ≥ 18 かつ MS-SSIM ≥ 0.60** を目安とする。これは健全性の目安であり、**合否は終了コードと6指標の生成（クラッシュせず有限値が出ること）で判定**する（4DGS の `metrics.py` は HyperNeRF 公式の covisible マスク付き評価とは手法が異なり、フル画像評価のため論文値と一致しない場合がある）。

### FR-006: 文書化と依存記録の更新

- **概要**: HyperNeRF 動作確認手順・gdown 導入・データ取得手順を再現可能な形で文書化し、依存記録を更新する。
- **入力**: FR-001〜005 の実施結果。
- **出力**: `design.md` 手順、`docs/TECH_STACK.md`（gdown 追記）、`CLAUDE.md`（データセット節の HyperNeRF を「取得済み・手順」に更新）、`requirements.lock.txt`（再生成）、`docs/BACKLOG.md`（Closed 化）。
- **受け入れ基準**:
  - `/clear` 後でも本書＋設計書のみで broom2 のデータ取得〜評価を再現できる（DL URL・gdown コマンド・配置パス・train/render/metrics コマンド・GPU/ポート選択手順を明記）。
  - `docs/TECH_STACK.md` に gdown（用途・選定理由・解決バージョン）を追記し、`requirements.lock.txt` を `uv pip freeze` で再生成する。
  - `CLAUDE.md` のデータセット節「HyperNeRF（実シーン）: colmap 前処理が必要」を、本案件の確定手順（事前生成点群DL）に更新する。

## 1.4 非機能要求

- **対応環境**: Ubuntu 22.04.5 / A100（sm_80）/ ヘッドレス（X ディスプレイ非前提）。`plot_camera_orientations`（`scene/dataset_readers.py:510`）が HyperNeRF 読み込み時に必ず matplotlib を import し `output.png` を savefig するため、**train/render 実行時は `MPLBACKEND=Agg` と書込可能な `MPLCONFIGDIR`（scratch 配下）を固定**してデータロード時のクラッシュを防ぐ（metrics.py は Scene を構築せず matplotlib を経由しないため不要）。
- **権限**: sudo 不可。ユーザー権限（`~/`・`/data/sakagawa/` 配下）で完結する。本案件は apt 等のシステム導入を要しない（gdown は `uv pip install` で `.venv` に入る）。
- **処理時間**: broom2 学習は README 目安で約30分（A100×1）。レンダ・評価は数分以内を想定。いずれも目安であり、合否は終了コードと生成物で判定する。
- **信頼性**: 学習出力は `output/hypernerf/broom2/` に置く。再実行時は出力先を削除してから再実行できること。データ（`data/hypernerf/`）はリポジトリにコミットしない。
- **マルチGPU**: 学習・レンダ・評価は 1 プロセス = 1 GPU。`CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N` を付け、実行前に空きGPU（`nvidia-smi`）と空きポート（`ss -ltn | grep :<port>`）を確認する（`CLAUDE.md` マルチGPU運用ルール準拠）。

## 1.5 制約条件

- **使用必須ライブラリ**:
  - 既存 `.venv`（torch 1.13.1+cu116 / mmcv 1.6.0 / numpy 1.23.5 等）と CUDA 拡張（feat-002 ビルド済み）。
  - matplotlib（導入済み。`plot_camera_orientations` が HyperNeRF 読み込み時に必ず呼ばれる）。
  - **gdown**（新規。`uv pip install gdown`）。
- **使用禁止 / 回避**:
  - **`uv sync` / `uv pip sync` は使わない**（追加的 `uv pip install` のみ）。`pyproject.toml` は作らない。
  - **4DGaussians 本体コードの改変は原則行わない**（本案件は環境・データ整備であり本体改変を要しない見込み。`colmap.sh` も本案件では実行しない）。
  - numpy の版を 1.23.5 から動かさない（gdown 導入で巻き上げが起きないこと）。
- **ネットワーク**: GitHub Release（vrig_broom.zip）と Google Drive（事前生成点群）の取得にインターネットアクセスを用いる。
- **ディスク**: `vrig_broom.zip` 1.5GB ＋展開数GB を `/data` 配下に置く。`data/hypernerf/` はコミットしない。

## 1.6 優先順位（MoSCoW）

| 要求 | 優先度 | 備考 |
|------|--------|------|
| FR-001 broom2 データ取得・配置 | **Must** | 学習の前提 |
| FR-002 事前生成点群の取得・配置 | **Must** | 学習に必須の ply。判定基準のデータ整備に対応 |
| FR-003 学習完走 | **Must** | 判定基準（train.py 完走）に対応 |
| FR-004 レンダリング完走 | **Must** | 判定基準（render.py 完走）に対応 |
| FR-005 評価完走 | **Must** | 判定基準（metrics.py 完走）に対応 |
| FR-006 文書化・依存記録更新 | **Must** | 再現性・運用のため |

- **MVP の範囲**: FR-001〜FR-006 すべて。これらの達成をもって BACKLOG の判定基準（`data/hypernerf/.../` 整備 + train→render→metrics 完走）を満たす。
