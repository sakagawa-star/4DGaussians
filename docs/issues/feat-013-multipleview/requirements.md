# feat-013 multipleview動作確認 要求仕様書

作成日: 2026-06-25
準拠: `docs/REQUIREMENTS_STANDARD.md`
ステータス: ドラフト（Codexレビュー前）

## 1. 背景・目的

### 1.1 背景

4DGaussians 環境構築の最終目的は「実装済み4データセット系統すべて（D-NeRF・HyperNeRF・DyNeRF・multipleview）で学習〜評価が動くこと」。このうち:

- D-NeRF（合成）: Phase 0-5 で完了（2026-05-25）
- HyperNeRF（実シーン・単眼）: feat-009 で完了（2026-06-21）
- DyNeRF（実シーン・多視点）: feat-012 で完了（2026-06-24、test PSNR 32.96）
- **multipleview（多視点・カスタム）: 本案件 feat-013 で実施 ＝ 実シーン系統の最後の1つ**

multipleview 系統は `scene/__init__.py:61` が `points3D_multipleview.ply` の存在で判定する独自経路（`readMultipleViewinfos`→`multipleview_dataset`）であり、DyNeRF（`readdynerfInfo`）とは別コードパスである。この経路が本環境で前処理〜学習〜レンダリング〜評価まで通ることを確認する。

### 1.2 目的

multipleview コードパス（`multipleview_dataset` / `readMultipleViewinfos`）が、本マシンの uv 環境で**前処理〜学習〜レンダリング〜評価まで完走する**ことを、1シーンで実証する。

### 1.3 スコープ

**対象（IN）**:
- 入力データの調達: **既存 DyNeRF データ `cut_roasted_beef` を multipleview 形式へ再編成**（ユーザー決定 2026-06-25）。新規ダウンロードはしない。
- 前処理: `multipleviewprogress.sh` 相当の処理（フレーム抽出→COLMAP SfM→密点群→downsample→LLFF poses）。ただし**本体スクリプトは非改変**で、ラッパー/手動手順として実行（ユーザー決定 2026-06-25）。
- config 作成: `arguments/multipleview/<scene>.py`
- 学習: `train.py`（`iteration` 生成）
- レンダリング: `render.py`
- 評価: `metrics.py`（6指標）
- 文書化・クローズ

**対象外（OUT）**:
- `multipleviewprogress.sh` 本体の改変（pip→uv 化・GPU引数化・COLMAP版対応）。→ ラッパー/手動で回避（ADR で詳述）。GPU引数化は将来案件に残す。
- 新規多視点データの取得（ユーザー判断で既存流用）
- 複数シーンでの実証（1シーンで足りる）
- SIBR 可視化・応用ツール

### 1.4 機能要求（FR）

| ID | 要求 | 受入基準 |
|----|------|----------|
| **FR-001** | 入力データ前提の検証 | 再編成後 `data/multipleview/cut_roasted_beef/camNN/frame_XXXXX.jpg` が、cam01〜（連番）・各カメラ同数フレーム・1始まり5桁ゼロ詰めで揃う。`multipleview_dataset` の前提（`cam01` 存在・`extr.name` 由来の `camNN` 対応）を満たす |
| **FR-002** | 前処理（COLMAP＋LLFF）完走 | `sparse_/{cameras,images}.bin`・`points3D_multipleview.ply`・`poses_bounds_multipleview.npy` が `data/multipleview/cut_roasted_beef/` に生成され、各ファイルが妥当。**COLMAP 幾何の健全性を `colmap model_analyzer` で検証**: ①`registered images = 20/20`（全カメラ登録）②`Mean reprojection error < 1.5 px` ③点数 > 0。加えて poses_bounds shape=(20,17)・images.bin のカメラ名が imageN 形式 |
| **FR-003** | 学習完走 | `train.py` が exit 0 で完走し、`output/multipleview/cut_roasted_beef/point_cloud/iteration_XXXXX/` が生成される（nan/例外なし） |
| **FR-004** | レンダリング完走 | `render.py` が exit 0 で完走し、`test/ours_XXXXX/{renders,gt}` が生成される（renders と gt のファイル名集合が一致） |
| **FR-005** | 評価完走＋健全性 | `metrics.py` が exit 0 で完走し、6指標がすべて有限値かつ次の数値基準を満たす: **PSNR ≥ 20.0 dB**／**SSIM・MS-SSIM ∈ [0,1]**／**D-SSIM ∈ [0,0.5]**／**LPIPS-vgg・LPIPS-alex は finite かつ ≥ 0**。`results.json`・`per_view.json` 生成。（multipleview は held-out カメラ無し＝学習視点再構成のため、通常は高PSNR。基準は破綻検知のための下限） |
| **FR-006** | 文書化・クローズ | CLAUDE.md データセット節に multipleview 行追記・`docs/BACKLOG.md` feat-013 を Closed・進捗台帳完了。本体改変が発生した場合は「オリジナルコードの変更点」に記録（本案件は原則ゼロ） |

### 1.5 非機能要求（NFR）

| ID | 要求 |
|----|------|
| NFR-001 | **本体改変ゼロ**（`multipleviewprogress.sh` 含む 4DGS 本体は非改変。ラッパー/手動で実行） |
| NFR-002 | **uv 依存管理ルール厳守**: スクリプト内の `pip install scikit-image`・`git clone LLFF` を本体実行に任せず、依存は事前に `uv pip install`、LLFF は事前 clone（bare pip を走らせない） |
| NFR-003 | **マルチGPU運用ルール準拠**: GPU を使う処理（COLMAP dense・train/render/metrics）は空きGPUを選び `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N` を付ける。COLMAP は `multipleviewprogress.sh` を実走しない（手動/ラッパー）ため GPU0 ハードコードに縛られず任意GPU選択可 |
| NFR-004 | **COLMAP は 3.11.1 を使用**（PATH 前置。CLAUDE.md「COLMAP の使い分け」節に準拠。経路は `mapper` で rig 非互換とは別だが、既定運用に合わせ保守的に 3.11.1。3.12.6 検証は design で扱う） |
| NFR-005 | 長時間処理（COLMAP dense・学習）はバックグラウンド＋ログをファイルにリダイレクトし、会話に進捗を流さない |
| NFR-006 | train/render は matplotlib top-level import 対策（`MPLBACKEND=Agg MPLCONFIGDIR=... TMPDIR=...`）を付与（feat-009/012 と同機構。`scene/gaussian_model.py`→`regulation.py` 経路）。metrics は不要 |

### 1.6 制約・前提

- 入力: 取得済み `data/dynerf/cut_roasted_beef/`（cam00〜cam20〔cam04欠番〕の20カメラ、各 `images/0000.png`〜`0299.png` 300枚、1352×1014 RGB PNG）。
- COLMAP 3.11.1（`/data/sakagawa/opt/colmap-3.11/bin`）・LLFF（`imgs2poses.py`）・`scene/multipleview_dataset.py` の前提（`cam01` 必須、frame_XXXXX.jpg）。
- multipleview の test は held-out カメラを持たない（train=全カメラ全フレーム、test=全カメラ×3時刻）。**評価値は未知視点の汎化ではなく学習視点の再構成品質**であり、PSNR は参考値として扱う（受入は「破綻なし＋6指標有限」）。

### 1.7 受入条件（DoD）

- FR-001〜FR-006 をすべて満たす
- Codexレビューで重要度「高・中」がゼロに収束
- 進捗台帳に各CP結果を記録、`docs/BACKLOG.md` feat-013 を Closed
- 本体改変があれば CLAUDE.md に記録（原則ゼロ）

## 2. 未決事項（design で確定する）

- D1: PNG→JPG の扱い（真のJPEG変換 か PNGバイトを.jpg名で流用か）
- D2: COLMAP 3.11.1 で前処理が通るか（mapper 経路の実走検証）。3.12.6 を使う必要が出るか
- D3: LLFF `imgs2poses.py` の依存（scikit-image 等）と py3 互換の実走確認
- D4: config は `default.py` 流用か `cut_roasted_beef.py` 新規作成か
