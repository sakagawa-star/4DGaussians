# CLAUDE.md

このファイルはClaude Codeがプロジェクトを理解するためのガイドです。

## セッション引き継ぎ

- セッション開始時にプロジェクトルートの `.claude/handovers/` ディレクトリを確認し、ファイルが存在すれば最新のものを読み込む
- セッション終了時や作業の区切りでは `/handover` の実行を促す

## プロジェクト概要

本プロジェクトは、HUST/4DGaussians（4D Gaussian Splatting for Real-Time Dynamic Scene Rendering, CVPR 2024, arXiv:2310.08528）を動作させるための環境構築を行う。

### 現在の最終目的

- **4DGaussiansが正常に動作する環境を構築すること**
- Phase 0-5（D-NeRF合成シーンの学習・レンダリング・評価）は **2026-05-25 完了**
- 現在は「**4DGS全体が動く環境**」へ拡張中（2026-06-18 ロードマップ策定）。実装済み4データセット系統すべて（D-NeRF〔済〕＋ HyperNeRF・DyNeRF・multipleview の**実シーン3系統**）で学習〜評価が動くこと、および**複数人共用サーバーでの任意GPU選択（マルチGPU運用）**をゴールとする
- スコープ外: dycheck（本体未実装）・full_eval（静的シーン専用）・SIBR可視化・応用ツール（詳細は `docs/BACKLOG.md`）
- 段階的に積み上げて確認する（各Phaseの判定基準を満たしてから次へ）。ロードマップの正本は `docs/BACKLOG.md`

### 背景

- 公式README（`README.md`）はconda + Python 3.7 + PyTorch 1.13.1+cu116 を前提とした手順を案内している
- 本マシンには **conda が無い** ため、**uv** を用いて環境を構築する（[docs/TECH_STACK.md](docs/TECH_STACK.md) 参照）
- CUDA拡張（`depth-diff-gaussian-rasterization`, `simple-knn`）のソースビルドが必須であり、これがセットアップの主要難所となる
- 開発プロセスのルール（要求仕様書・機能設計書・レビュー・不具合修正の基準）は `docs/` 配下に独立して保持する汎用基準を用いる（本リポジトリ内で自己完結し、他プロジェクトに依存しない）

## 実行環境（本マシン）

| 項目 | 値 |
|------|-----|
| GPU | NVIDIA A100-SXM4-40GB × 7（Driver 565.57.01）。学習・レンダリングは単一GPUを使用 |
| CUDA Toolkit | 12.8（`~/cuda/cuda-12.8/`、`~/cuda/current` → `~/cuda/cuda-12.8` のシンボリックリンク） |
| nvcc | release 12.8, V12.8.93（`nvcc --version` で確認可能） |
| CUDA環境変数 | `CUDA_HOME=/home/sakagawa/cuda/current`、`PATH` と `LD_LIBRARY_PATH` にcurrent配下を設定済み |
| システムPython | 3.10.12 |
| パッケージ管理 | uv 0.11.6（インストール済み） |
| conda | **未インストール**（公式手順のconda部分はuvで代替する） |
| colmap | **vcpkg ソースビルド導入済み（feat-008, 2026-06-21）**。`colmap[core,cuda]:x64-linux@3.12.6`（CUDA有効・GUI除外）。PATH=`~/.local/bin/colmap`（→`/data/sakagawa/opt/vcpkg/installed/x64-linux/tools/colmap/colmap`）。ビルドに gfortran 必須（`sudo apt`、管理者承認）。ヘッドレスで GPU SIFT 動作実証。詳細は `docs/TECH_STACK.md`「COLMAP」節・`docs/issues/feat-008-colmap/`。**＋3.11.1 を別 prefix 併設（feat-011, 2026-06-23、rig 非互換回避）**: `/data/sakagawa/opt/colmap-3.11/`（ラッパー `bin/colmap`）。DyNeRF/multipleview の `colmap.sh`/`multipleviewprogress.sh` 実走時のみ使用（後述「COLMAP の使い分け」節・`docs/issues/feat-011-colmap-3.11/`） |
| gfortran | **導入済み（feat-008, 2026-06-21、`sudo apt`、管理者承認）**。GNU Fortran 11.4.0。COLMAP 依存 `lapack-reference`（Reference LAPACK）のビルドに必須 |
| リポジトリ | `/data/sakagawa/4DGaussians`（git clone済み） |

## マルチGPU運用ルール（任意GPU選択）

本マシンは A100 × 7 を複数人で共用する。学習・レンダリング・評価は**1 プロセス = 1 GPU**で動かし、空いている任意の GPU を選んで使う（feat-007 で GPU≠0 の3経路完走と負荷限定を実機検証済み。2026-06-18、**コード変更ゼロ**）。

1. **実行前に空き GPU を確認**する: `nvidia-smi --query-gpu=index,pci.bus_id,uuid,memory.used,utilization.gpu --format=csv`。`memory.used` が小さく `utilization.gpu` が低い GPU 番号 N を 1 枚選ぶ。
2. **実行コマンドには `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N` を付ける**。`CUDA_DEVICE_ORDER=PCI_BUS_ID` が無いと `nvidia-smi` の index と CUDA の番号がズレうる。同一ジョブの 3 経路（train/render/metrics）は同一 N を使う。
   例: `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=3 .venv/bin/python train.py ...`
3. コード内の `cuda:0` / `cuda`（`utils/general_utils.py:139`、`metrics.py:116-117`、`render.py`）は **マスク後の論理デバイス先頭**を指すため物理 GPU N に乗る。**コード変更は不要**。
4. **分散学習（複数 GPU 同時使用）は本体非対応**。1 ジョブで複数 GPU は使えない。
5. **学習はポートを使う**（`--port`、既定 6009）。実行前に `ss -ltn | grep :<port>` で空きを確認する。使用中だと `network_gui` の `bind()` が例外処理されておらず**起動時にクラッシュ**する。並行実行時もポートを変える。render/metrics はポートを使わない。

## COLMAP の使い分け（3.12.6 / 3.11.1）

本マシンには COLMAP が2系統入っている（feat-008: 3.12.6、feat-011: 3.11.1）。用途で使い分ける。4DGS 本体（`colmap.sh` 含む）は非改変。

1. **既定は 3.12.6**（`~/.local/bin/colmap`、`command -v colmap` がこれを指す）。静的シーンや `convert.py` の mapper 経路で使う。
2. **DyNeRF/multipleview の `colmap.sh ... llff` / `multipleviewprogress.sh` は 3.11.1 を使う**。3.12.6 は単一カメラ共有 sparse + `point_triangulator` で rig 非互換（`Check failed: existing_frame.RigId()==frame.RigId()`）クラッシュするため。実行は **PATH 前置**:
   ```bash
   PATH="/data/sakagawa/opt/colmap-3.11/bin:/data/sakagawa/4DGaussians/.venv/bin:$PATH" \
     bash -e colmap.sh <workdir> llff
   ```
   - 3.11.1 は別 prefix（`/data/sakagawa/opt/colmap-3.11/`）に分離され、PATH 前置を外せば既定 3.12.6 のまま（既存ワークフロー非破壊）。
   - GPU は `colmap.sh:5` のハードコード（`CUDA_VISIBLE_DEVICES=0`）に従う。**GPU0 使用中なら空くまで待つ**（任意 GPU 選択＝`colmap.sh:5` 引数化は将来の COLMAP 再実走案件〔feat-013 等〕で検討。feat-012 は前処理流用で `colmap.sh` を実走せず動作確認できないため対象外）。
   - `feature_extractor` は CPU SIFT（`colmap.sh:15` の `estimate_affine_shape`/`domain_size_pooling` 指定のため自動フォールバック、正常）。dense（`patch_match_stereo`）は GPU0。
   - `point_triangulator --clear_points 1` は filename で image_id を database に揃える（`sparse_custom` と `database.db` の image_id 数値不一致は正常）。
   - 詳細は `docs/TECH_STACK.md`「COLMAP 3.11.1 併設」節・`docs/issues/feat-011-colmap-3.11/`。

## 技術スタック

- **詳細**: [docs/TECH_STACK.md](docs/TECH_STACK.md) を参照
- **本プロジェクトの採用版**: Python **3.10**（uv managed）/ torch **1.13.1+cu116** / mmcv 1.6.0 ＋各種ライブラリ（`requirements.txt`）。公式のPython 3.7はuvが提供しないため3.10を採用（根拠はTECH_STACK.md）
- **公式要件（参考）**: Python 3.7 / PyTorch 1.13.1+cu116 / mmcv 1.6.0（conda前提のオリジナル手順）
- **CUDA拡張**: `submodules/depth-diff-gaussian-rasterization`（深度対応版、ingra14m fork）、`submodules/simple-knn`
- **CUDA整合（決定済み）**: CUDA拡張のビルドは `/usr/local/cuda-11.6` を使う（torch cu116とメジャー・マイナー一致）。グローバルでアクティブな CUDA 12.8 のままビルドするとメジャー版不一致（12 vs 11）でビルドエラーまたは重大な互換性警告が生じうるため、**ビルド時のみ `CUDA_HOME=/usr/local/cuda-11.6` を上書き**する。詳細は TECH_STACK.md

## サブモジュールの状態

- `submodules/depth-diff-gaussian-rasterization` と `submodules/simple-knn` は **ビルド済み（feat-002, 2026-05-22）**。`git submodule update --init --recursive`（glm含む）取得後、CUDA 11.6・`--no-build-isolation` で editable ビルド。simple-knn は uv editable 向けに `simple_knn/__init__.py`（`import torch`）を追加（git管理外。`docs/issues/feat-002-cuda-ext-build/investigation.md` 参照）
- `SIBR_viewers`（`.gitmodules` に定義）は可視化ビューア。環境構築の必須要件ではない

## データセット

学習・評価には外部データセットが必要（`data/` 配下、`.gitignore` 管理外を想定）。

- **D-NeRF（合成シーン）**: 最初の動作確認に使用。**取得済み（feat-003, 2026-05-22）**。Dropbox の `data.zip`（246MB）を展開し `data/dnerf/{scene}/` へ全8シーン（bouncingballs/hellwarrior/hook/jumpingjacks/lego/mutant/standup/trex）を配置。各シーンに `transforms_{train,val,test}.json` と `train/`・`val/`・`test/`（png）。本体は train/test のみ読込。`data/` は `.gitignore` 管理外（git未追跡。再取得手順は `docs/issues/feat-003-dnerf-data/design.md` 参照）
- **HyperNeRF（実シーン・単眼）**: **broom2 で学習〜評価 動作確認済み（feat-009, 2026-06-21）**。HyperNeRF v0.1 リリース `vrig_broom.zip`（1.5GB）を `data/hypernerf/virg/` へ展開（zipトップが `broom2/` のため `data/hypernerf/virg/broom2/` ができる。画像394枚・`rgb/2x` 必須）。点群は Google Drive の事前生成COLMAP点群（file id `1fUHiSgimVjVQZ2OOzTFtz02E9EqCoWr5`）を **gdown** でDLし `points3D_downsample2.ply`（38,569点）を配置（**COLMAP 実走不要**）。学習は `--configs arguments/hypernerf/broom2.py`、`render.py --skip_train`→`metrics.py` で PSNR 22.08/MS-SSIM 0.691（論文 22.0/0.70）。視覚的裏付けに chicken（`vrig_chicken.zip`→`data/hypernerf/virg/vrig-chicken/`、点群は事前生成zip内 `virg-chickchicken/`、config `chicken.py`）も実施し PSNR 28.65/MS-SSIM 0.930（論文 28.7/0.93、目視で鮮明と確認）。**train/render は `MPLBACKEND=Agg MPLCONFIGDIR=... TMPDIR=...` を付与**（読込時 `plot_camera_orientations` が matplotlib で `output.png` を CWD に savefig するため）。詳細は `docs/issues/feat-009-hypernerf/`。`data/` は `.gitignore` 管理外（git未追跡）
- **Plenoptic / DyNeRF（Neural 3D Video）**: **cut_roasted_beef で学習〜評価 動作確認済み（feat-012, 2026-06-24、test PSNR 32.96 / 論文 33.85）**。フレーム抽出 + colmap 前処理が必要（**COLMAP は feat-011 で併設した 3.11.1 を使う**。3.12.6 は `colmap.sh ... llff` の `point_triangulator` で rig 非互換クラッシュ。使い分けは「COLMAP の使い分け」節参照）。前処理（フレーム抽出・`colmap.sh llff`・downsample）は feat-011 で実証済み。入力は `data/dynerf/cut_roasted_beef/`（`cam*.mp4`×20・`camXX/images`各300枚〔cam04欠番〕・`poses_bounds.npy`(20,17)・`points3D_downsample2.ply` 37,361点）。学習 `--configs arguments/dynerf/cut_roasted_beef.py`（`iteration_14000` 生成）→`render.py --skip_train`（test/video）→`metrics.py --model_paths` で 6指標健全（PSNR 32.96 / SSIM 0.9471 / MS-SSIM 0.9750 / LPIPS-vgg 0.1526 / LPIPS-alex 0.0528 / D-SSIM 0.0125）。**train/render は `MPLBACKEND=Agg MPLCONFIGDIR=... TMPDIR=...` を付与**（`regulation.py:5`・`scene_utils.py:4` の matplotlib top-level import 対策。metrics は不要）。GPU 系コマンドは `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N`。詳細は `docs/issues/feat-012-dynerf/`。`data/` は `.gitignore` 管理外（git未追跡）

## オリジナルコードの変更点

4DGaussians 本体は原則変更しない（開発方針）。やむを得ず変更した箇所を以下に記録する（理由の詳細は各案件ドキュメント参照）。

- **`scene/dataset_readers.py:287`（feat-004, 2026-05-22）**: `Image.fromarray(np.array(arr*255.0, dtype=np.byte), "RGB")` の `np.byte`（int8）を `np.uint8` に変更。新しい **Pillow 12.2.0** が `Image.fromarray(..., "RGB")` に int8 配列（`|i1`）を受理せず `TypeError` でクラッシュするため（旧 Pillow は許容）。RGB 値 0〜255 は uint8 が意味的に正しい。Blender(D-NeRF) の train/test 読み込み（学習・レンダリング・評価の全経路）で使用。詳細・ADR は `docs/issues/feat-004-dnerf-train/investigation.md`

## ディレクトリ構成（主要部分）

```
4DGaussians/
├── CLAUDE.md               # 本ファイル
├── README.md               # オリジナルのREADME（公式セットアップ手順）
├── requirements.txt        # pip依存定義（torch 1.13.1 等、公式・緩いスペック）
├── requirements.lock.txt   # 依存の厳密スナップショット（uv pip freeze、版の正本。feat-001で生成）
├── train.py                # 学習エントリポイント
├── render.py               # レンダリング
├── metrics.py              # 評価
├── convert.py / colmap.sh  # COLMAP前処理（実シーン用）
├── full_eval.py            # 一括評価
├── export_perframe_3DGS.py # タイムスタンプ別3DGS書き出し
├── merge_many_4dgs.py      # 学習済み4DGSのマージ
├── multipleviewprogress.sh # 多視点データのポーズ・点群生成
├── arguments/              # 学習設定（シーン別Pythonコンフィグ）
│   ├── dnerf/              # D-NeRF（bouncingballs, lego, mutant 等）
│   ├── dynerf/             # DyNeRF（Neural 3D Video）
│   ├── hypernerf/          # HyperNeRF
│   ├── dycheck/            # DyCheck
│   └── multipleview/       # 多視点
├── scene/                  # シーン・データセット・モデル定義
│   ├── gaussian_model.py   # ガウシアンモデル
│   ├── deformation.py      # 変形フィールド
│   ├── hexplane.py         # Multi-resolution HexPlane
│   ├── dataset_readers.py  # データセット読み込み
│   └── ...
├── gaussian_renderer/      # レンダラ（network_gui.py 含む）
├── utils/                  # 各種ユーティリティ（loss, camera, graphics 等）
├── lpipsPyTorch/           # LPIPS実装
├── scripts/                # 前処理・変換・学習補助スクリプト
│   ├── preprocess_dynerf.py
│   ├── downsample_point.py
│   ├── *2colmap.py         # 各データセット→COLMAP変換
│   └── train_*.sh          # シーン別学習スクリプト
├── submodules/             # CUDA拡張（要ビルド・初期状態は未初期化）
│   ├── depth-diff-gaussian-rasterization/
│   └── simple-knn/
├── docs/                   # ドキュメント（開発プロセス基準）
│   ├── BACKLOG.md
│   ├── BUGFIX_STANDARD.md
│   ├── DESIGN_STANDARD.md
│   ├── REQUIREMENTS_STANDARD.md
│   ├── REVIEW_CRITERIA.md
│   ├── TECH_STACK.md
│   ├── viewer_usage.md     # 公式ビューアの使い方（オリジナル）
│   └── issues/             # 案件ディレクトリ
└── assets/                 # README用画像等
```

## 開発方針

- **シンプルな機能を一つずつ作り、積み重ねて目的を達成する**
- 大きな機能を一度に作らない。小さく作って動作確認し、次へ進む
- 4DGaussiansのオリジナルコードは可能な限り変更しない。変更が必要な場合は理由を案件ドキュメントに記録する
- 環境構築は段階的に進める（CUDA拡張ビルド → 学習 → レンダリング → 評価）。各段階で判定基準を満たしてから次へ進む

### 機能追加フロー（feat-XXX 案件）

新機能や新しい環境構築ステップを進める場合、以下のフローを**厳守**する。**planモードは使わない**（通常モードで調査・計画を行う）。

1. **案件作成** → `docs/issues/feat-{number}-{slug}/` フォルダを作成し、`docs/BACKLOG.md` に追加する
2. **調査・計画** → 通常モードで既存コード・公式手順を調査し、要求仕様書（`docs/REQUIREMENTS_STANDARD.md` 準拠）と機能設計書（`docs/DESIGN_STANDARD.md` 準拠）を作成する
3. **ドキュメント保存** → 要求仕様書を `docs/issues/{案件フォルダ}/requirements.md`、機能設計書を `docs/issues/{案件フォルダ}/design.md` にファイル保存する。**保存が完了するまで実装に進んではならない**
4. **レビュー（Codex → 人）** → 保存されたドキュメントを **Codex** でレビューする。実行方法は後述の「Codexによるレビューの実行方法」を参照。**まず Codex の再帰レビュー（修正→再レビュー）を重要度「高・中」がゼロに収束するまで回し、その後に人（ユーザー）がレビューする**（収束前に人レビューはしない）。レビュー実行時は `docs/REVIEW_CRITERIA.md` の基準に従うこと
5. **修正（必要な場合）** → レビューで問題があれば、再調査してドキュメントを更新する。**ステップ2〜4を問題がなくなるまで繰り返す**
6. **実装** → ドキュメント（要求仕様書・機能設計書・CLAUDE.md）を読んで実装する。実装完了後、動作確認を実行する
7. **手動テスト** → ユーザーがテストする。以下の問題があれば `docs/BUGFIX_STANDARD.md` に従って修正計画を `docs/issues/{案件フォルダ}/investigation.md` に追記する（上書きしない。イテレーション番号を付けて履歴を残す）。**ユーザーの承認を得た上で、ステップ2〜7を繰り返す**（コード修正はステップ6で行う。ステップ7で直接コードを編集してはならない）
   - 不具合の発見
   - 要求通りに実装されていない
   - 要求仕様作成時のヒアリング漏れ
8. **完了** → `docs/BACKLOG.md` のステータスを Closed に更新する。ファイルの追加・削除があった場合は `CLAUDE.md` のディレクトリ構成を最新に更新する

### 不具合修正フロー（bug-XXX 案件）

既存機能・既存環境の不具合を修正する場合、以下のフローを**厳守**する。

1. **案件作成** → `docs/issues/bug-{number}-{slug}/` フォルダを作成し、`docs/BACKLOG.md` に追加する。`README.md` に不具合の概要と再現手順を記録する
2. **調査・修正計画** → `docs/BUGFIX_STANDARD.md` に従い、既存コードを調査する。修正計画を `docs/issues/{案件フォルダ}/investigation.md` に記録する。**この時点でコードを編集してはならない**
3. **ドキュメント保存** → investigation.md の保存を確認する。**保存が完了するまで実装に進んではならない**
4. **レビュー（Codex → 人）** → 保存されたドキュメントを **Codex** でレビューする。実行方法は後述の「Codexによるレビューの実行方法」を参照。**まず Codex の再帰レビュー（修正→再レビュー）を重要度「高・中」がゼロに収束するまで回し、その後に人（ユーザー）がレビューする**（収束前に人レビューはしない）。レビュー実行時は `docs/REVIEW_CRITERIA.md` の基準に従うこと
5. **修正（必要な場合）** → レビューで問題があれば、再調査してドキュメントを更新する。**ステップ2〜4を問題がなくなるまで繰り返す**
6. **実装** → 承認された修正計画に沿ってコードを修正する。計画にない変更が必要になった場合は中断して報告する
7. **手動テスト** → ユーザーがテストする。問題があれば `docs/BUGFIX_STANDARD.md` に従って investigation.md にイテレーション番号を付けて追記し、**ユーザーの承認を得た上で、ステップ2〜7を繰り返す**（コード修正はステップ6で行う。ステップ7で直接コードを編集してはならない）
8. **完了** → `docs/BACKLOG.md` のステータスを Closed に更新する。ファイルの追加・削除があった場合は `CLAUDE.md` のディレクトリ構成を最新に更新する

### ドキュメント作成ルール

- **実装前に必ずドキュメントを作成し、案件フォルダにファイル保存すること**
- ドキュメントが保存されていない場合は、**実装を中止**する
- 機能追加時: 要求仕様書（`docs/REQUIREMENTS_STANDARD.md` 準拠）と機能設計書（`docs/DESIGN_STANDARD.md` 準拠）を作成する
- 不具合修正時: `docs/BUGFIX_STANDARD.md` の基準に従い、修正計画を `investigation.md` に記録する
- レビュー実行時は `docs/REVIEW_CRITERIA.md` の基準に従うこと
- ドキュメントは `docs/issues/{案件フォルダ}/` に置く（`requirements.md`, `design.md`, `investigation.md`）
- **/clear 後でも実装がスムーズにできるよう、必要な情報を全て記述する**
- 暗黙知に頼らず、**自己完結したドキュメント**にする（前の会話コンテキストがなくても実装できること）
- ライブラリの追加・変更・削除を行った場合は `docs/TECH_STACK.md` を更新し、**`requirements.lock.txt` を `uv pip freeze` で再生成**すること
- 新規ライブラリ導入時は用途・選定理由・バージョンを `TECH_STACK.md` に追記すること
- **uv 依存管理ルール（厳守）**: パッケージ追加は常に `uv pip install`（追加的）で行い、**`uv sync` / `uv pip sync` は使わない**（未宣言パッケージを削除し環境を破壊するため）。`pyproject.toml` は作らない。背景・詳細・依存記録の役割分担は `docs/TECH_STACK.md`「uv 依存管理ルール」を参照

### 案件ディレクトリ構成

```
docs/issues/
└── {type}-{number}-{slug}/    # 例: feat-001-env-setup, bug-001-xxx
    ├── README.md              # 概要、ステータス、再現手順
    ├── requirements.md        # 要求仕様書（機能追加時、REQUIREMENTS_STANDARD.md 準拠）
    ├── design.md              # 機能設計書（機能追加時、DESIGN_STANDARD.md 準拠）
    ├── investigation.md       # 不具合の調査・修正計画（BUGFIX_STANDARD.md 準拠）
    └── reviews/               # Codexレビュー出力（codex-NN.result.md は git 管理、codex-NN.full.log は .gitignore）
```

### 命名規則

- フォルダ名は英語で統一（例: `feat-001-env-setup`, `bug-001-rasterizer-build-fail`）
- 案件フォルダは完了後も削除・移動しない

### Codexによるレビューの実行方法

機能追加・不具合修正フローのステップ4（レビュー）では、Claude Code 自身が `codex exec` コマンドを実行して Codex にレビューさせる。Subagent は使わない。**Codex は逐次（前回セッションを `resume` で継続）で回し、重要度「高・中」がゼロに収束してから人レビューに進む**。並列にはしない（再レビューの収束確認＝「前回指摘が直ったか」の判定に前回文脈の引き継ぎが必要なため。初回の発見網羅性を上げたい大規模案件でのみ「初回だけ多観点並列→以降逐次」を検討）。

使用するモデルは `~/.codex/config.toml` のデフォルト設定に従う。本ファイルのコマンドにはモデル指定（`-m`）を書かない。モデルを切り替えたい場合は `~/.codex/config.toml` を編集する（全プロジェクト共通で反映される）。

#### 出力の保存（結果と過程を分離）

- レビュー結果と過程ログは **案件フォルダの `docs/issues/{案件フォルダ}/reviews/`** に保存する（事前に `mkdir -p` する）。
- **初回から `-o`（`--output-last-message`）を必ず付ける**。`-o` で最終レビュー結果だけを `codex-NN.result.md` に書き、stdout 全体（過程ログ）は `> codex-NN.full.log 2>&1` で別ファイルに保存する（混在させない）。
- ファイル名はレビュー回ごとに連番（`codex-01`, `codex-02`, …）。
- `result.md` のみ git 管理し、`full.log` は `.gitignore`（`docs/issues/*/reviews/*.full.log`）でローカルのみとする（リポジトリ肥大回避）。
- `result.md` には Codex の生出力に加え、Claude Code の対応方針を追記してよい（冒頭に日付・対象・session id・初回/再の定型メタを置くと追いやすい）。

#### 初回レビュー（機能追加の場合）

```bash
mkdir -p docs/issues/{案件フォルダ}/reviews
codex exec -o docs/issues/{案件フォルダ}/reviews/codex-01.result.md \
  "docs/REVIEW_CRITERIA.md の基準に従い、以下のドキュメントをレビューせよ: docs/issues/{案件フォルダ}/requirements.md docs/issues/{案件フォルダ}/design.md 。瑣末な点へのクソリプはしないで、致命的な点のみ指摘して。発見した問題を重要度(高/中/低)で分類し、修正提案とともに報告すること。" \
  > docs/issues/{案件フォルダ}/reviews/codex-01.full.log 2>&1
```

#### 初回レビュー（不具合修正の場合）

```bash
mkdir -p docs/issues/{案件フォルダ}/reviews
codex exec -o docs/issues/{案件フォルダ}/reviews/codex-01.result.md \
  "docs/REVIEW_CRITERIA.md および docs/BUGFIX_STANDARD.md の基準に従い、以下のドキュメントをレビューせよ: docs/issues/{案件フォルダ}/investigation.md 。瑣末な点へのクソリプはしないで、致命的な点のみ指摘して。発見した問題を重要度(高/中/低)で分類し、修正提案とともに報告すること。" \
  > docs/issues/{案件フォルダ}/reviews/codex-01.full.log 2>&1
```

#### 再レビュー（共通）

ドキュメントを更新して再レビューする場合、最初のレビューの文脈を保持するため**同一セッションを `resume` で継続**する。セッション ID は `codex-01.full.log` 冒頭の `session id:` 行に記録されるので、それを明示指定するのが確実（`--last` でも可だが、別の codex 実行が挟まると意図しないセッションを掴む恐れがある）。連番を1つ進める:

```bash
codex exec resume {SESSION_ID} -o docs/issues/{案件フォルダ}/reviews/codex-02.result.md \
  "ドキュメントを更新したので再レビューして。前回と同じ基準で。前回指摘が解消されたかを含めて確認して。瑣末な点へのクソリプはしないで、致命的な点のみ指摘して。重要度(高/中/低)で分類し、修正提案とともに報告すること。" \
  > docs/issues/{案件フォルダ}/reviews/codex-02.full.log 2>&1
```

**注意**: `resume`（セッション継続）を使わないと最初のレビューの文脈が失われる。`-o` と `> ...full.log 2>&1` は毎回付け、連番（`codex-03`, `codex-04`, …）を進める。

#### レビュー終了条件

重要度「高」「中」の指摘がゼロに収束するまで、修正 → 再レビュー（連番を進める）を繰り返す。**収束したら人（ユーザー）レビューに進む**（収束前に人レビューはしない）。

### コードレビュー

- レビューでは重要度(高/中/低)で分類し、修正提案とともに報告する
- 重要度:高と中は修正対象とする
- レビュー基準の詳細は `docs/REVIEW_CRITERIA.md` を参照

