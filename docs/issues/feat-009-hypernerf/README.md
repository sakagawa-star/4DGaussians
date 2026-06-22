# feat-009: HyperNeRF動作確認（実シーン・単眼）

## 概要

実シーン・単眼の HyperNeRF データセット1シーン（**broom2**、vrig_broom）で、4DGaussians の「**学習 → レンダリング → 評価**」が一通り完走することを確認する。点群（`points3D_downsample2.ply`）は **README 記載の事前生成COLMAP点群を Google Drive からDL**して用い、COLMAP 実走（`colmap.sh`）はスキップする（COLMAP 自体は feat-008 で検証済み）。

- **Phase**: 8（`docs/BACKLOG.md`）
- **依存**: feat-008（COLMAP環境構築・Closed）, feat-007（マルチGPU運用・Closed）
- **Status**: **Closed（2026-06-22、手動テスト合格）**。broom2 で学習〜評価を完走（PSNR 22.08/MS-SSIM 0.691、論文 22.0/0.70 と一致）。視覚的裏付けに chicken も追加実施（PSNR 28.65/MS-SSIM 0.930、論文 28.7/0.93、目視で鮮明と確認）。本体コード変更ゼロ

## 背景・課題

- D-NeRF（合成・単眼）は Phase 0-5 で「学習→レンダ→評価」完了済み。本案件は **実シーンの単眼系統（HyperNeRF）**へ初めて踏み込む。
- HyperNeRF 経路は D-NeRF（Blender）と**データ読み込み・前処理・config が大きく異なる**:
  - データ判定は `dataset.json` の存在（`scene/__init__.py:55` → `readHyperDataInfos`、内部 `Load_hyper_data`）。
  - 学習用点群 `points3D_downsample2.ply` が**必須**（`scene/dataset_readers.py:384-385`、欠落時フォールバック無しで例外）。これは COLMAP dense → `downsample_point.py` で生成するか、事前生成版をDLして用意する。
  - config は `arguments/hypernerf/broom2.py`（`_base_=default.py`、fine 14000 反復・batch_size 2 等）。
- 本案件では、ユーザー選択により**事前生成点群DL**ルートを採る（COLMAP 実走は feat-008 で検証済みのため省略し、学習〜評価の動作確認に集中する）。

## 導入方式（確定）

| 項目 | 採用 |
|------|------|
| 対象シーン | **broom2**（HyperNeRF v0.1 リリース `vrig_broom.zip`、1.5GB）。専用 config `arguments/hypernerf/broom2.py` 実在 |
| 画像・カメラ一式 | GitHub Release `https://github.com/google/hypernerf/releases/download/v0.1/vrig_broom.zip` を取得・展開し `data/hypernerf/virg/broom2/` へ配置 |
| 点群 | **事前生成COLMAP点群を Google Drive からDL**（README 157行目、file id `1fUHiSgimVjVQZ2OOzTFtz02E9EqCoWr5`）。`points3D_downsample2.ply` を `data/hypernerf/virg/broom2/` へ配置 |
| DL ツール | **gdown**（新規導入、`uv pip install`）。Google Drive 大容量ファイルの confirm token を自動処理 |
| GPU/ポート | マルチGPU運用ルール（`CLAUDE.md`）に従い空きGPU 1枚＋空きポートを選択。train/render/metrics は同一GPUを使う |

## スコープ

- **対象**: broom2 のデータ取得・配置、事前生成点群の取得・配置、`train.py`→`render.py`→`metrics.py` の完走確認、文書化。
- **対象外**: COLMAP 実走（`colmap.sh hypernerf`）= feat-008 で COLMAP 検証済みのため本案件では実行しない（**フォールバックとしてのみ design に記載**）。他の HyperNeRF シーン（chicken/3dprinter/banana）横展開、HyperNeRF 公式の covisible マスク付き評価（4DGS の `metrics.py` は非対応）。

## 判定基準（BACKLOG より）

`data/hypernerf/.../` を整備し、`train.py`→`render.py`→`metrics.py` が完走する。データ形式は scene.json/metadata.json/dataset.json/rgb/camera 構造（`scene/dataset_readers.py:readHyperDataInfos`、判定は dataset.json の存在）。

## 関連環境事実（調査時点 2026-06-21、実機確認済み）

| 項目 | 値 |
|------|-----|
| colmap | feat-008 で導入済み（`~/.local/bin/colmap`）。本案件では未使用（事前生成点群DLのため） |
| Python 依存 | matplotlib 3.10.9（backend=agg、ヘッドレス安全）/ tqdm 4.67.3 / open3d 0.19.0 / scikit-image 0.22.0 / imageio 2.37.3 はいずれも導入済み |
| gdown | **未導入**（本案件で `uv pip install` 予定） |
| データ | `data/` に `dnerf` のみ。`data/hypernerf` は未取得 |
| ディスク | `/data` 15TB 空き。`vrig_broom.zip` 1.5GB + 展開数GB を `/data` 配下に置く |
| GPU | A100-SXM4-40GB ×7。学習は単一GPU。空きGPUは実行直前に `nvidia-smi` で確認 |

## ドキュメント

- `requirements.md` — 要求仕様書（REQUIREMENTS_STANDARD.md 準拠）
- `design.md` — 機能設計書（DESIGN_STANDARD.md 準拠・ADR 付き）
- `reviews/` — Codexレビュー出力（`codex-NN.result.md` は git 管理、`*.full.log` は .gitignore）

## 手動テスト結果（2026-06-22、合格）

### broom2（主シーン、FR-001〜006）

- データ: `vrig_broom.zip`（1.5GB）展開 → `data/hypernerf/virg/broom2/`（rgb/2x・camera 各394件）。事前生成点群 `points3D_downsample2.ply`（38,569点）配置。
- 学習: GPU0/port6019、約17分（coarse3000+fine14000）、`iteration_14000` のみ生成（coarse 保存なし＝設計どおり）。
- レンダ: `--skip_train`、test renders/gt 各197枚（名前集合一致）、video 生成。
- 評価: **PSNR 22.08 / MS-SSIM 0.691 / SSIM 0.368 / LPIPS-vgg 0.556 / LPIPS-alex 0.547 / D-SSIM 0.154**。6指標すべて有限、per_view 197件一致。
- 論文（CVPR2024 Table 5）broom = PSNR 22.0 / MS-SSIM 0.70 と一致。

### chicken（視覚的裏付けの追加検証、ユーザー要望 B）

- 経緯: broom2 は benchmark 最難シーンで、論文でも最低スコア（MS-SSIM 0.70）。箒（細く動きの速い物体）がぼやけるのは手法・シーンの既知の限界で、目視では良否判断が難しかった。そこで、より易しい chicken（論文 28.7/0.93）を回して視覚的に確認した。
- データ: `vrig_chicken.zip`（871MB）展開 → `data/hypernerf/virg/vrig-chicken/`（rgb/2x・camera 各328件）。事前生成点群は zip 内 `hypernerf/virg/virg-chickchicken/points3D_downsample2.ply`（80,418点）を配置（命名は不規則: 画像zipトップ=`vrig-chicken`、点群フォルダ=`virg-chickchicken`。ローダはパス名に非依存）。
- 学習: GPU0/port6029、約27分。レンダ test 164枚。評価: **PSNR 28.65 / MS-SSIM 0.930 / SSIM 0.814 / LPIPS-vgg 0.277 / LPIPS-alex 0.182 / D-SSIM 0.035**（論文 28.7/0.93 と一致）。
- **ユーザーが renders を目視し「綺麗」と確認**。これによりパイプラインの正しさが視覚的に確定し、broom2 の地味さはシーン難易度に起因すると結論。

### 結論

HyperNeRF（実シーン・単眼）が学習〜評価で正しく動作する環境を確認。2シーンとも論文値に一致。本体コード変更ゼロ。`data/`・`output/` は git 追跡外（コミットしない）。`output.png`（matplotlib 副作用）も未追跡。
