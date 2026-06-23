# feat-010 調査記録: DyNeRF前処理 COLMAP実走の不具合

本書は `docs/BUGFIX_STANDARD.md` に準拠する。実装（機能追加フロー ステップ6）中の動作確認で発覚した不具合を記録する。

---

## イテレーション1 (2026-06-23): COLMAP 3.12 rig/frame 非互換による point_triangulator クラッシュ

### 1.1 不具合の特定

- **現在の動作**: FR-003 で `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh data/dynerf/cut_roasted_beef llff 0` を実行すると、`colmap point_triangulator`（`colmap.sh:20`）が SIGABRT（exit 134）でクラッシュし、`colmap/sparse/0/` が空になる。後続の image_undistorter・patch_match_stereo・stereo_fusion も入力欠如で連鎖クラッシュし、`fused.ply` が生成されない。
  - 再現手順: FR-001（データ配置）→ FR-002（フレーム抽出）完了後、上記 colmap.sh を実行。`llff2colmap.py` 自体は正常完了（`sparse_/`・`image_colmap/r_000〜r_019.png` 生成）し、その後の colmap バイナリで落ちる。
- **期待する動作**: point_triangulator が既知ポーズから三角化し `colmap/sparse/0/` に cameras/images/points3D を生成、dense MVS を経て `colmap/dense/workspace/fused.ply`（点数>0）を生成する。
- **エラーメッセージ**（トレースバック要点）:
  ```
  E  reconstruction.cc:166] Check failed: existing_frame.RigId() == frame.RigId()
  terminate called after throwing an instance of 'std::invalid_argument'
    what():  [reconstruction.cc:166] Check failed: existing_frame.RigId() == frame.RigId()
  *** SIGABRT ... colmap::Reconstruction::Read(...) <- colmap::RunPointTriangulator
  colmap.sh: line 20: ... Aborted (core dumped) colmap point_triangulator ...
  # 以降、sparse/0 が空のため連鎖:
  reconstruction.cc:736] rigs, cameras, frames, images, points3D files do not exist at .../colmap/sparse/0
  ... image_undistorter / patch_match_stereo / stereo_fusion も Aborted
  ```

### 1.2 原因分析

- **原因箇所**: `scripts/llff2colmap.py` の COLMAP モデル生成部。
  - `images.txt`（138行）: 全画像の CAMERA_ID を固定値 `1` で出力（`print(idx+1," ".join(qevc)," ".join(T),1,image_name_list[idx],"\n",...)`）。
  - `cameras.txt`（141-142行）: カメラを1台（id=1）のみ出力（`print(1,"SIMPLE_PINHOLE",1352,1014,focal[0],676,507,...)`）。
- **原因の説明**: COLMAP **3.12**（feat-008 導入版、3.12.6）は rig/frame モデルがネイティブ化され、reconstruction 読込時に「**各カメラ＝独立 rig、各 frame＝単一画像**」をデフォルトで構築する（COLMAP 3.12 公式 rig ドキュメントの記述: "By default ... each camera is modeled with a separate rig and thus each frame contains only a single image."）。`llff2colmap.py` は20枚すべてを単一カメラ（CAMERA_ID=1）で共有させるため、COLMAP 3.12 は「1カメラ=1 rig」に20画像を詰め込もうとし、`existing_frame.RigId() == frame.RigId()` の整合チェックで矛盾を検出して abort する。旧 COLMAP（3.11 以前、rig 非ネイティブ）では単一カメラ共有でも三角化できたため、4DGS の `colmap.sh`/`llff2colmap.py` はそれを前提にしている。
- **根本原因 or 表面的原因**: **根本原因**。COLMAP 3.12 の rig モデルと「単一カメラ共有フォーマット」の非互換。`colmap.sh:5`（GPU 引数化）とは無関係で、design.md が「colmap.sh をそのまま実走できる」と想定していた見落とし（COLMAP 版差の検証漏れ）。
- **調査根拠**:
  - COLMAP 公式 rig ドキュメント（3.12）の上記デフォルト記述。
  - COLMAP GitHub Issue #3433: 既知ポーズ＋単一カメラ共有で同種エラー。回避策として「各画像に別カメラ ID を割り当てる（separate camera entries for each image）と rig 検証を回避できる（sidesteps the rig validation logic entirely）」。
  - `database.py`（`camTodatabase`）は cameras.txt の各カメラ行をループ読みし `update_camera` で database に反映する役割（`database.py:84-100`）。cameras.txt をN台にすればN台に対応する。
  - `colmap.sh:15` の feature_extractor は `--ImageReader.single_camera` 未指定のため、**デフォルトで各画像に別カメラを割り当てる**（database に20カメラができる）。現状はそこへ database.py が1台だけ反映し、images.txt も全部 camera_id=1 を参照する不整合構成になっている。

### 1.3 修正内容（対処A: 各画像を別カメラ＝別 rig にする）

- **変更対象ファイル**: `scripts/llff2colmap.py`（本体改変。`colmap.sh:5` に次ぐ2箇所目の本体改変）。
  - 修正1（images.txt の camera_id を画像ごとに変える、138行）:
    ```python
    # 修正前
    print(idx+1," ".join(qevc)," ".join(T),1,image_name_list[idx],"\n",file=object_images_file)
    # 修正後（camera_id を idx+1 に）
    print(idx+1," ".join(qevc)," ".join(T),idx+1,image_name_list[idx],"\n",file=object_images_file)
    ```
  - 修正2（cameras.txt をN台出力、141-142行）:
    ```python
    # 修正前
    object_cameras_file = open(os.path.join(colmap_dir,"cameras.txt"),"w")
    print(1,"SIMPLE_PINHOLE",1352,1014,focal[0],1352/2,1014/2,file=object_cameras_file)
    # 修正後（各画像に同一 intrinsics の別カメラ。camera_id は images.txt と対応）
    object_cameras_file = open(os.path.join(colmap_dir,"cameras.txt"),"w")
    for cam_idx in range(len(poses)):
        print(cam_idx+1,"SIMPLE_PINHOLE",1352,1014,focal[0],1352/2,1014/2,file=object_cameras_file)
    ```
- **変更しないファイル**:
  - `colmap.sh`: 5行目（GPU 引数化）以外は変更しない。feature_extractor が既にデフォルトで「各画像別カメラ」を作るため、cameras.txt をN台にすれば database と整合する。
  - `database.py`: cameras.txt をループ読みするため、N台でもそのまま動作（変更不要）。
  - 4DGS 学習側（`scene/neural_3D_dataset_NDC.py` 等）: `poses_bounds.npy` を直接読むため、COLMAP のカメラ分割は最終点群（fused.ply→points3D_downsample2.ply）の生成にのみ影響し、学習入力フォーマットは不変。

### 1.4 影響範囲

- **他の機能への影響**: `llff2colmap.py` は DyNeRF（`colmap.sh ... llff`）専用。`blender2colmap.py`（D-NeRF）・`hypernerf2colmap.py`（HyperNeRF）は別ファイルで影響なし。feat-011（multipleview）は別スクリプト（`multipleviewprogress.sh`）だが同種の rig 問題が出る可能性があり、その対処は feat-011 で別途行う（本案件のスコープ外）。
- **リグレッションリスク**:
  - 各画像別カメラは同一 intrinsics のため、三角化される点群の品質は単一カメラ共有時と実質同等（DyNeRF は実際に20物理カメラなので意味的にもむしろ正しい）。
  - `database.py` の assert（cameras.txt と database の一致）が、feature_extractor の camera_id 割り当て順（image_colmap の r_000→camera1, r_001→camera2, … 名前順）と llff2colmap の camera_id（idx+1）が一致することに依存する。実行で確認する（不一致なら assert で停止し検知できる）。
  - 旧 COLMAP に戻すわけではないため、他データセットへのリグレッションなし。

### 1.5 確認方法

- **テスト項目**: colmap.sh 再実行で全 colmap コマンド（feature_extractor〜stereo_fusion）が exit 0 で完走し、`fused.ply`（点数>0）が生成される。
- **テストコマンド**:
  ```bash
  # colmap.sh 冒頭で sparse_/image_colmap/colmap は自動削除・再生成される
  PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID \
    bash colmap.sh data/dynerf/cut_roasted_beef llff 0
  # 検証（点数 > 0）
  .venv/bin/python -c "from plyfile import PlyData; print(len(PlyData.read('data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply')['vertex']))"
  ```
  - 期待: point_triangulator のログに registered images=20・三角化点数>0、`fused.ply` の点数>0。

### 対処の選択肢（ユーザー承認用）

| 案 | 内容 | 改変範囲 | コスト | 評価 |
|----|------|---------|--------|------|
| **A（推奨）** | `llff2colmap.py` を各画像別カメラ化 | `colmap.sh:5` + `llff2colmap.py`（計2ファイル） | 小（数行） | COLMAP 公式回避策（Issue #3433）。意味的にも DyNeRF の20物理カメラに合致。最小・確実 |
| B | COLMAP を rig 非ネイティブ版（3.11 以前）に併設/切替 | 4DGS 非改変を維持 | 大（vcpkg 旧版ビルド、feat-008 相当の作業やり直し） | 4DGS 完全非改変だが導入コスト大・将来保守性低 |
| C | `colmap.sh` に `rig_configurator` ステップを追加 | `colmap.sh` に複数行追加 + rig_config.json | 中 | 手順複雑・rig 定義の設計が必要 |

**推奨は A**（最小改変・COLMAP 公式推奨の回避策）。承認後、design.md に本体変更点（`llff2colmap.py`）と ADR を追記し、実装→再テストする。

---

## イテレーション2 (2026-06-23): 検証実験で対処Aの不足が判明、正しい対処A'を実証

ユーザー指示（対処A の影響範囲をさらに詳細調査）を受け、**本体コードを変えずに中間生成物（sparse_custom）を編集して point_triangulator を単体実行する非破壊の検証実験**を実施した。

### 2.1 追加で判明した事実

- **feature_extractor は既に各画像を別カメラ・別 rig・別 frame で登録済み**（colmap.sh:15 は `--ImageReader.single_camera` 未指定のため）。database を読むと `cameras=20 / images=20`、point_triangulator のログでも `Loading rigs... 20 / Loading frames... 20`。
- ただし **database の image_id↔name 対応が乱れている**（並列登録の影響）。例: `image_id=1→r_001.png(camera_id=2)`、`image_id=5→r_000.png(camera_id=1)`。`llff2colmap.py` の前提（`image_id=idx+1 ↔ r_idx.png ↔ camera_id=1`）と**不一致**。
- **検証1（対処A 相当: camera_id=image_id の単純連番、image_id/name は llff2colmap のまま）→ 失敗**。`Reconstruction::Load(DatabaseCache)` で同じ `Check failed: existing_frame.RigId() == frame.RigId()`。`point_triangulator` は input reconstruction の image を **image_id で database と突き合わせる**ため、image_id↔name↔camera_id が database と食い違うと rig/frame が矛盾する。**対処A（llff2colmap の単純2行改変）は不十分**と確定。
- **検証2（対処A': image_id・camera_id・name を database に完全整合させ、各 image の name=r_idx に対応するポーズを poses から計算した sparse_custom を生成）→ 成功**。`point_triangulator` が20画像すべて三角化して完走し、`colmap/sparse/0/` に `cameras.bin`・`frames.bin`・`images.bin`・`points3D.bin`・`rigs.bin` を生成（各画像 300〜1500 点を視認）。

### 2.2 正しい対処（A'）と、対処A が成立しない構造的理由

- **構造的理由**: `colmap.sh` は `llff2colmap.py`（sparse_custom 生成）を **feature_extractor の前**に実行する。この時点で database は存在せず、llff2colmap は database の image_id/camera_id 割り当てを知り得ない。よって llff2colmap 内でいくら camera_id を変えても、後から feature_extractor が割り当てる（乱れた）image_id とは整合できない。
- **対処A'**: **feature_extractor 実行後に database を読み、`image_id`・`camera_id`・`name` を database に完全整合させた sparse_custom（各 image の name=r_idx に対応するポーズ）を生成する追加ステップ**を colmap.sh に挟む。検証2 で point_triangulator 完走を実証済み。
- **database.py の AssertionError**: 検証2 では cameras.txt を image_id 順（camera_id が非ソート）で書いたため、`database.py` の最終 assert（`SELECT * FROM cameras` は camera_id 昇順）で停止した（ただし `update_camera` 自体は全行成功し point_triangulator は動いた）。対処では **cameras.txt を camera_id 昇順でソートして書く**ことで解消する。

### 2.3 対処A' の実装方針（案）

`colmap.sh:5`（GPU 引数化、実施済み）に加え、以下を行う。本体改変は **colmap.sh の構造変更 + 新スクリプト1本**に増える。

1. **新スクリプト `scripts/dynerf_sparse_from_db.py`（仮）を追加**: 引数 `workdir` で `workdir/colmap/database.db` と `workdir/poses_bounds.npy` を読み、database の各 image（image_id, camera_id, name=r_idx）に対応するポーズ（poses[idx] を llff2colmap と同一変換）を計算して、database 整合の `workdir/colmap/sparse_custom/{images.txt, cameras.txt}` を生成する（cameras.txt は camera_id 昇順、各カメラ同一 SIMPLE_PINHOLE intrinsics）。検証2 の実証ロジックをそのままスクリプト化。
2. **colmap.sh のロジック変更**: `llff2colmap.py`（image_colmap コピー + 仮 sparse_）→ `cp sparse_ colmap/sparse_custom` → `feature_extractor` → **`dynerf_sparse_from_db.py`（colmap/sparse_custom を database 整合に上書き）** → `database.py` → `exhaustive_matcher` → `point_triangulator` → …（以降は不変）。
   - 注: `llff2colmap.py` 自体は image_colmap コピーのために残す（sparse_ は新スクリプトが上書きするので実質未使用になるが、image_colmap 生成は必要）。
3. **検証3（実装後）**: 改変版 colmap.sh をフル実走し、`fused.ply`（点数>0）→ `downsample_point.py` → `points3D_downsample2.ply`（≤40k点）まで完走することを確認。

### 2.4 対処の選択肢（更新版・ユーザー承認用）

| 案 | 内容 | 改変範囲 | 実証状況 | 評価 |
|----|------|---------|---------|------|
| **A'（推奨）** | feature_extractor 後に database 整合 sparse_custom を生成する新スクリプト + colmap.sh 構造変更 | `colmap.sh:5` + colmap.sh 構造 + 新スクリプト `scripts/dynerf_sparse_from_db.py` | **検証2 で point_triangulator〜dense 全段完走を実証（`fused.ply` 387,693 点生成）** | COLMAP 3.12 で確実に動く唯一実証済みの対処。本体改変は増えるが DyNeRF 前処理専用で影響限定 |
| B | COLMAP 旧版（3.11 以前）に切替/併設 | 4DGS 非改変 | 未検証 | 導入コスト大・保守性低 |
| D | feature_extractor の image_id を名前順に強制し対処A を成立させる | colmap.sh（feature_extractor オプション） + llff2colmap | 未検証（image_id 順制御は COLMAP 仕様上не保証） | 最小だが確実性が低く、検証2 ほどの確証なし |

**推奨は A'**（検証2 で実証済み）。承認後、design.md に本体変更点（colmap.sh 構造 + 新スクリプト）と ADR を追記し、実装→再テストする。
