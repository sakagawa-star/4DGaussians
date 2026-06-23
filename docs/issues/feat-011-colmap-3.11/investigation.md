# feat-011 調査記録: COLMAP 3.11.1 ビルドの不具合

本書は `docs/BUGFIX_STANDARD.md` に準拠する。実装（機能追加フロー ステップ6）中の動作確認（FR-001 ビルド）で発覚した不具合を記録する。

---

## イテレーション1 (2026-06-23): 現 baseline の Eigen 5.0.1 非互換による COLMAP 3.11.1 ビルド失敗

### 1.1 不具合の特定

- **現在の動作**: `vcpkg install`（manifest: `overrides` colmap 3.11.1#4 + `builtin-baseline` `36393d1`〔現 HEAD〕）が **exit 1** で失敗。x64-linux-**dbg**（Debug）ビルドの `src/colmap/estimators/CMakeFiles/colmap_estimators.dir/covariance.cc.o` で `FAILED`。約2.4分で停止。
  - 再現手順: design §1.4.1 手順6 のビルドコマンド（CUDA 11.6）を実行。依存（ceres/eigen/suitesparse 等）は通り、COLMAP 本体のコンパイルで停止。
- **期待する動作**: COLMAP 3.11.1 バイナリが `/data/sakagawa/opt/colmap-3.11/installed/x64-linux/tools/colmap/colmap` に生成される。
- **エラーメッセージ**（`install-x64-linux-dbg-out.log`）:
  ```
  covariance.cc:185:28: error: 'const VectorXd' {aka 'const class Eigen::Matrix<double, -1, 1>'} has no member named 'nonZeros'
  FAILED: src/colmap/estimators/CMakeFiles/colmap_estimators.dir/covariance.cc.o
  ```
- 既存 3.12.6 は**無傷**（`command -v colmap`=`~/.local/bin/colmap`、版表示 3.12.6、ラッパー不変）。install-root 分離が機能している。

### 1.2 原因分析

- **原因箇所**: 現 baseline（vcpkg HEAD `36393d1`）の `eigen3` が **5.0.1**。COLMAP 3.11.1 のソース `covariance.cc:185`（`const int rank = D_dense.nonZeros();`）が `Eigen::VectorXd`（Dense）の `nonZeros()` を呼ぶ。
- **原因の説明**: Eigen 5.0 で `DenseBase::nonZeros()` が削除された（旧 Eigen 3.4 には存在）。COLMAP **3.11.1 は Eigen 3.4 前提**のソースで、現 baseline の Eigen 5.0.1 と**コンパイル非互換**。3.12.6 は Eigen 5.0.1 対応済み（`covariance.cc` が修正済み）のため feat-008 で通った。
- **根本原因 or 表面的原因**: **根本原因**。「3.11.1 を現 baseline（新依存）でビルドする」設計が、依存（Eigen）のメジャー版差を見落としていた。design §1.4.1 の **B1（3.11.1 が現 baseline 依存と非互換）が顕在化**した形。
- **調査根拠**:
  - `versions/baseline.json`（HEAD `36393d1`）の `eigen3`=**5.0.1**。
  - 3.12.6 化 commit `36da49e14a` の親 `37c4e62c`（2025-11-04）の `baseline.json`: **colmap=3.11.1#4 / eigen3=3.4.1#1 / ceres=2.2.0#5**（＝3.11.1 と整合する依存ツリー）。
  - `install-x64-linux-dbg-out.log` の FAILED ターゲット・`error:` 行（`buildtrees/colmap/`）。
  - 先に確認した `install-x64-linux-rel-out.log` は feat-008 の**古い 3.12.6 ログ**（`src/3.12.6-...`）で本失敗とは無関係（共有 buildtrees に残存）。本失敗は dbg-out.log（`src/3.11.1-...`）。

### 1.3 修正内容（対処B1: builtin-baseline を旧 commit に変更）

- **変更対象**: `/data/sakagawa/opt/colmap-3.11/vcpkg.json`（**リポジトリ外**の manifest。4DGS 本体は非改変のまま）。
  - `builtin-baseline` を `36393d1ca008...`（現 HEAD）→ **`37c4e62c5ed20ac4cb09884917bde2cbbccf7aa3`**（2025-11-04、colmap 3.11.1 が baseline・eigen 3.4.1）に変更。
  - これにより依存ツリー全体（eigen 3.4.1#1・ceres 2.2.0#5・suitesparse 群等）が**当時の整合版**で解決される。`overrides`（colmap 3.11.1#4）は維持。
- **採用理由**: Eigen だけを 5.0.1→3.4.1 に override で下げると、ceres/suitesparse 等が新 Eigen 前提で連鎖破綻するため、依存ツリー全体を「3.11.1 が整合した旧 baseline」で一括に揃える方が確実。
- **既存環境の非破壊**: 既存 vcpkg をチェックアウトせず、versions DB（git 履歴）から旧版を解決する。旧版は install-root（別 prefix `/data/sakagawa/opt/colmap-3.11/installed`）のみにビルドし、既存 3.12.6 installed・既存 classic installed・`.venv` には触れない。
- **検証（実装前）**: dry-run で `colmap[core,cuda]:x64-linux@3.11.1#4` + `eigen3:x64-linux@3.4.1#1` + ceres 2.2.0#5 の解決を**実証済み**。

### 1.4 影響範囲

- **他機能への影響**: なし。旧 baseline 版は install-root（別 prefix）のみに入る。既存 3.12.6（feat-008）・既存 vcpkg classic installed・`.venv` は非影響。HyperNeRF（feat-009）は COLMAP 非使用で無関係。
- **リグレッションリスク**: install-root に前回（新 baseline）の部分インストール（約1.8G、eigen 5.0.1 等）が残るため、**再ビルド前に install-root を `rm -rf` でクリーン**して旧版と混在させない。buildtrees は版ごとにハッシュ分離（`src/3.11.1-...` と `src/3.12.6-...` が別）されるため共有で安全。

### 1.5 確認方法

- 再ビルドで `vcpkg install` が **exit 0**、`LD_LIBRARY_PATH=.../lib .../tools/colmap/colmap -h` の1行目が **`COLMAP 3.11.1`**。
- `install-x64-linux-dbg-out.log` / `rel` に `covariance.cc` の `nonZeros` エラーが出ない（eigen 3.4.1 で通る）。
- 既存 3.12.6 が無傷（`command -v colmap`=3.12.6、`~/.local/bin/colmap` 不変）。
- 以降は design §1.4.2 以降（ラッパー作成・colmap.sh llff 検証）へ進む。

### 1.6 設計への反映（FR-005 文書化で実施）

- 再ビルド成功後、`design.md` §1.4.1（manifest の `builtin-baseline` を旧 commit に）・ADR-1（B1 対処の経緯）、`requirements.md` FR-001（builtin-baseline 値）を更新する。`docs/TECH_STACK.md` の 3.11.1 併設節にも「builtin-baseline=旧 commit（Eigen 非互換回避）」を明記する。

---

## イテレーション1 検証結果 (2026-06-23): 対処B1 で再ビルド成功・成果物実証

前セッション終盤に Claude のツール呼び出し崩れ＋出力 garbled が発生し、再ビルドの成否が未確定のまま中断していた。本セッションで **garbled 出力に依らず、実成果物で clean に再検証**した（§1.5「確認方法」に対する確認結果）。

- **②-1 バイナリ実在**: `/data/sakagawa/opt/colmap-3.11/installed/x64-linux/tools/colmap/colmap`（36,417,752 バイト、6/23 13:09）。**実在**。
- **②-2 版表示**: `LD_LIBRARY_PATH=.../lib .../colmap -h` の1行目が **`COLMAP 3.11.1 -- Structure-from-Motion and Multi-View Stereo`**（`Commit 682ea9ac... on 2026-06-23 with CUDA`）。**期待通り・CUDA 有効**。
- **②-3 build2.log 精査**: `/data/sakagawa/tmp/feat011-colmap-3.11/build2.log`（135,506 バイト）。`error:`/`FAILED`/`nonZeros` の出現 **0 件**。末尾に `Completed submission of colmap[core,cuda]:x64-linux@3.11.1#4 ...` ＋ `All requested installations completed successfully in: 17 min` ＋ **`=== vcpkg install (rebuild) exit: 0 ===`**。**exit 行が明確に存在し成功確定**（前セッションの「exit 0 通知は当てにならない」懸念を実ログで払拭）。§1.2 の `covariance.cc:185 nonZeros` エラーは再ビルドで消失（eigen 3.4.1 で通った）。
- **②-4 既存 3.12.6 無傷**: `command -v colmap` = `/home/sakagawa/.local/bin/colmap`、版表示 `COLMAP 3.12.6`。**install-root 分離が機能し既存環境は非破壊**。

→ §1.5 の確認方法を全て満たし、**イテレーション1（対処B1: builtin-baseline を旧 commit `37c4e62c` に変更）の修正は有効**と確定。FR-001（ビルド）達成。

---

## 前提検証 (2026-06-23): rig 非互換理論の裏取り（colmap.sh 実走の前提）

「3.11.1 なら `Check failed: existing_frame.RigId()==frame.RigId()` クラッシュ（3.12.6 で発生）が起きない」は feat-011 全体の**前提仮説**だが未検証で、かつ「他の 4DGS ユーザーが 3.11 でも同じ RigId クラッシュを報告した」という**反例情報**があった。colmap.sh 実走前に、両版ソースの grep で裏取りした。

- **対象**: 3.11.1 = `/data/sakagawa/opt/vcpkg/buildtrees/colmap/src/3.11.1-ba4648498c.clean`、3.12.6 = 同 `3.12.6-22005fa86a.clean`。

| 観点 | 3.11.1 | 3.12.6 | 判定 |
|---|---|---|---|
| **クラッシュアサーション本体** `THROW_CHECK(existing_frame.RigId() == frame.RigId())` | **0 件** | `src/colmap/scene/reconstruction.cc:166` | ✅ 3.11.1 に無い（決定的） |
| `existing_frame` / `RigId()` 呼び出し件数 | 0 / 0 | 8 / 34 | ✅ 概念ごと不在 |
| 新 `class Frame` / `class Rig` 定義 | **無し** | `src/colmap/scene/frame.h:46` 他 | ✅ 新 rig アーキ未導入 |
| `rigs.bin` / `frames.bin` 永続化 | 0 件 | `src/colmap/scene/reconstruction.cc` | ✅ ネイティブ永続化しない |
| `rig_configurator` サブコマンド登録 | 0 | 2 | ✅ 3.11.1 に無い |

- **3.11.1 の rig 関連ファイル**は旧来の `scene/camera_rig.{cc,h}`（旧 CameraRig 概念）と剛体変換 `geometry/rigid3.{cc,h}`（別物）のみ。問題を起こす**新 rig アーキテクチャ（`scene/frame.h`・`scene/rig.h`・`sensor/rig.h`）は 3.12 系で初導入**されたことがソース差分から明確。
- **結論**: クラッシュアサーション `reconstruction.cc:166` は 3.12 系で導入されたもので、**3.11.1 には存在しない**。`colmap.sh llff`（単一カメラ共有＋`point_triangulator`）経路でも 3.11.1 では当該整合チェックが走らないため、**RigId クラッシュは原理的に発生し得ない**。理論成立。
- **反例情報の解釈**: 「3.11 でも RigId クラッシュ」報告は、新 rig アーキが入った後期 3.11.x パッチや別系統 RigId（旧 camera_rig 由来）の誤認の可能性が高い。**我々が導入する 3.11.1 そのものには当該アサーションが無い**ことを実ソースで確認済みのため、本経路には当たらない。

→ 引き継ぎノートの判定基準「無ければ理論成立」を満たし、**前提クリア**。FR-002（ラッパー作成）以降に進んでよい。

---

## イテレーション2 (2026-06-23): FR-003 実走で検証3（image_id↔name 一致）が False → 真相は design 受け入れ基準の誤り

FR-002 ラッパー作成後、FR-003（`PATH="/data/sakagawa/opt/colmap-3.11/bin:.venv/bin:$PATH" bash -e colmap.sh data/dynerf/cut_roasted_beef llff`）を 3.11.1 で実走。exit 0 で完走したが、design §1.4.3 手順4の検証で **検証3（image_id↔name 完全一致 db==sc）が False** になった。調査の結果、**これは実装の不具合ではなく design の受け入れ基準（検証3）が誤っていた**と判明した。

### 2.1 実走結果（design §1.4.3 手順4 の5検証）

- **検証1 rig クラッシュ無し**: ✅ `colmap_llff.log` に `RigId`/`existing_frame`/`Check failed` = 0、`SIGABRT`/`Aborted` = 0。`point_triangulator` が20画像すべて三角化して完走（`image.cc:347 => Reconstruction with 20 images and 4566 points`）。**rig 非互換は理論どおり解消**。
- **検証2 fused.ply**: ✅ `colmap/dense/workspace/fused.ply` 387,496点（10MB、`stereo_fusion` 完走）。
- **検証3 image_id↔name 一致**: ❌ db==sc が **False**。database.db は `r_000.png→id5, r_002.png→id1` …と name 順でなくシャッフル。sparse_custom/images.txt は `r_000→1, r_001→2` …（name 順）。**→ ただし下記2.2の通り、これは正常動作で基準の方が誤り**。
- **検証4 exit 0**: ✅ `=== colmap.sh llff (3.11.1) finished exit: 0 ===`。
- **検証5 git 本体差分**: ✅ 追跡ファイル差分は `submodules/{depth-diff-gaussian-rasterization,simple-knn}`（CUDA拡張ビルド成果物、実行前から存在・colmap.sh と無関係）のみ。colmap.sh 実走による本体改変ゼロ。

### 2.2 検証3 False の真相: image_id 数値不一致は COLMAP の正常動作（基準が誤り）

- **当初の懸念**: feat-010 investigation 1.3/2.1（line 105）に「`point_triangulator` は input reconstruction の image を **image_id で** database と突き合わせる」とあり、image_id 不整合だとポーズ↔特徴点が食い違い幾何が壊れる（3.12.6 では RigId クラッシュ）。3.11.1 はクラッシュしないぶん「黙って壊れた点群」を出す恐れを疑った。
- **決定的反証**: `colmap model_analyzer --path colmap/sparse/0`（3.11.1）の **Mean reprojection error = 0.852581px**（正常域 0.5〜1.5px）。20画像登録・Mean track length 5.35・Observations 24,443。**ポーズと特徴点が正しく対応している直接証拠**（食い違っていれば数px〜数十pxになる）。
- **メカニズム（3.11.1 ソースで確定）**: `colmap.sh:20` は `point_triangulator --clear_points 1`。3.11.1 の `RunPointTriangulatorImpl`（`src/colmap/exe/sfm.cc:599-602`）は `clear_points` 時に **`reconstruction->TranscribeImageIdsToDatabase(database)`** を呼ぶ。その実装（`src/colmap/scene/reconstruction.cc:471`）は各 image を **`database.ReadImageWithName(image.Name())`**（=**filename で database を引く**）して reconstruction の image_id を database の image_id に**書き換える**（name が DB に無ければ `LOG(FATAL_THROW)` で停止）。`point_triangulator` の `--clear_points` 説明文も「recompute the image_ids based on **matching filenames** between the model and the database」と明記。
- **結論**: llff2colmap.py が振る image_id（name 順）と feature_extractor が並列登録で振る image_id（順不同）が食い違うのは**設計上当然**で、`--clear_points 1` の `TranscribeImageIdsToDatabase` が **filename ベースで自動整合**する。よって **sparse_custom と database の image_id 数値不一致は正常**で、triangulation は filename で正しく行われる（reprojection error 0.852px が実証）。**design 検証3（db==sc 数値一致）は過剰かつ誤った基準**だった。

### 2.3 feat-010 との対比（対処A' は 3.11.1 では不要）

- feat-010 は **3.12.6** で「image_id 不整合 → `Check failed: existing_frame.RigId()==frame.RigId()` クラッシュ」と結論し、対処A'（feature_extractor 後に database 整合 sparse_custom を生成する新スクリプト + colmap.sh 構造変更）を必要とした。
- 真相: 3.12.6 のクラッシュ原因は image_id 不整合そのものではなく **rig/frame 整合チェック**（transcribe 後でも camera_id↔rig↔frame 構築で矛盾）。3.11.1 は **rig アーキが無く**（前提検証で実証）、`TranscribeImageIdsToDatabase`（filename 整合）だけで完結するため、**本体完全非改変のまま正しい点群が得られる**。
- **したがって対処A'（colmap.sh 構造変更・新スクリプト追加）は 3.11.1 では不要**。これは feat-011（3.11.1 併設）が狙った効果そのもので、完全に実証された。

### 2.4 design/requirements への反映（FR-005 文書化で実施）

- **検証3 の受け入れ基準を訂正する**。誤: 「`database.db` images と `sparse_custom/images.txt` の image_id↔name が全20枚一致（db==sc）」。正: 「① 全 image の name が database に存在し `TranscribeImageIdsToDatabase` が FATAL_THROW なく完了（=20画像登録）② `model_analyzer` の Mean reprojection error が健全（目安 < 1.5px、実測 0.852px）③ 三角化点数 > 0」。
- design §1.4.3 手順4・エラーハンドリング B5（image_id 不整合を異常とみなす前提）、requirements FR-003 受け入れ基準を上記に合わせて修正する。`image_id 数値不一致は --clear_points 1 の filename transcribe により正常` を明記する。
- **GPU/CPU SIFT の補足**: design は feature_extractor を「GPU SIFT」と記したが、`colmap.sh:15` は `--SiftExtraction.estimate_affine_shape 1 --SiftExtraction.domain_size_pooling 1` を指定しており、この2オプションは COLMAP の GPU SIFT 非対応のため **CPU SIFT に自動フォールバック**する（3.11.1/3.12.6 共通仕様、正常）。llff 経路は元から CPU SIFT が正しい。後段 `patch_match_stereo`（MVS）のみ CUDA(GPU0) を使う。FR-005 で TECH_STACK/design に注記する。

→ FR-003 は正しい判定軸では**全項目合格**（rig 解消・reprojection 0.852px・20/20登録・fused 387,496点・本体非改変）。受け入れ基準訂正の承認後、FR-004（downsample）へ進む。
