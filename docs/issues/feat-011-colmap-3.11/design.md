# feat-011 機能設計書: COLMAP 3.11.1 併設（rig 非互換回避）

本書は `docs/DESIGN_STANDARD.md` に準拠する。本書のコードスニペットは「意図の伝達」が目的であり、そのままコピー実行する前にパス・commit・版を確認すること。

> **前提となる確定事項（2026-06-23、実機調査で確認）**
> - vcpkg は classic モード（`vcpkg.json` 無し）、HEAD `36393d1ca008d0086488a9041afac26ed3b8edb9`、3.12.6 を `installed` に導入済み。
> - `versions/c-/colmap.json` に COLMAP 3.11.1 port-version 0〜4 が存在。**最新 port-version 4** の git-tree=`c166234c960ad821bfddccbe87089e1c3d5fa583`。baseline.json の colmap は 3.12.6。
> - **manifest + override + builtin-baseline で 3.11.1 が解決されることを dry-run で実証済み**（`colmap[core,cuda]:x64-linux@3.11.1#4`＋依存 ceres 2.2.0/lapack-reference/suitesparse 群がビルド対象に列挙）。
> - 検証データ cut_roasted_beef は20カメラ・各300フレーム抽出済み（`camXX/images/0000.png` 存在）。`/data` 空き 15TB。

---

## 1.1 対応要求マッピング

| 要求ID | 設計セクション |
|--------|----------------|
| FR-001 3.11.1 vcpkg ソースビルド（別 prefix・3.12.6 温存） | §1.4.1 |
| FR-002 3.11.1 PATH ラッパー作成・併設 | §1.4.2 |
| FR-003 colmap.sh ... llff を 3.11.1 実走（rig 非互換解消の実証） | §1.4.3 |
| FR-004 ダウンサンプル | §1.4.4 |
| FR-005 文書化 | §1.4.5 |
| 全体構成・配置 | §1.2, §1.6 |
| 技術選定理由（ADR） | §2.4 |

## 1.2 システム構成

本案件は **4DGaussians 本体コードを1行も改変せず**、外部ツール（COLMAP 3.11.1 バイナリ）を vcpkg manifest モードで**別 install prefix** にソースビルドして併設し、PATH ラッパー経由で 4DGS の `colmap.sh` から 3.11.1 を使えるようにする。

```
[既存（温存・非変更）]
  /data/sakagawa/opt/vcpkg/                         # vcpkg 本体（classic, HEAD 36393d1）
    ├ vcpkg                                          # ★本案件でもこの vcpkg バイナリを使う
    ├ downloads/                                     # ソース tarball キャッシュ（★3.11.1 ビルドで再利用）
    ├ buildtrees/ packages/                          # ビルド作業（★3.11.1 分が追加される）
    └ installed/x64-linux/tools/colmap/colmap        # ★3.12.6 実体（触れない）
  ~/.local/bin/colmap                                # ★3.12.6 ラッパー（デフォルト、触れない）

[本案件で新規作成]
  /data/sakagawa/opt/colmap-3.11/                    # 3.11.1 専用ディレクトリ（リポジトリ外・新規）
    ├ vcpkg.json                                     # manifest（override 3.11.1#4 + builtin-baseline）
    ├ bin/colmap                                     # 3.11.1 ラッパー（colmap.sh の bare colmap 用）
    └ installed/x64-linux/                           # ★3.11.1 専用 install-root（3.12.6 と完全分離）
        ├ tools/colmap/colmap                        #   3.11.1 実体
        └ lib/*.so                                   #   3.11.1 同梱依存

[検証で実走（4DGS 本体は非改変）]
  colmap.sh data/dynerf/cut_roasted_beef llff        # bare colmap → PATH 前置で 3.11.1
    └ scripts/llff2colmap.py / database.py           # bare python → PATH 前置で .venv/bin/python
  scripts/downsample_point.py                        # .venv の open3d で fused.ply を ≤40k 点に
```

- **依存方向**: `colmap.sh`（PATH の `colmap`）→ 3.11.1 ラッパー → 3.11.1 実体。Python 前処理 → `.venv`。循環なし。既存 3.12.6 経路（`~/.local/bin/colmap`）とは PATH 前置の有無だけで切り替わり、相互に干渉しない。
- **分離の原則**: 3.11.1 と 3.12.6 は (1) install-root が別、(2) ラッパーが別ファイル、(3) 既存ラッパーは PATH 前置時のみシャドウされる（恒久変更なし）—の3点で完全分離する。

### ディレクトリ構成（実パス）

```
/data/sakagawa/opt/vcpkg/                    # 既存 vcpkg（再利用・installed は触れない）
/data/sakagawa/opt/colmap-3.11/              # 本案件で新規作成
  ├ vcpkg.json                               #   manifest
  ├ bin/colmap                               #   3.11.1 ラッパー（chmod +x）
  └ installed/x64-linux/{tools/colmap,lib}/  #   3.11.1 install-root
~/.local/bin/colmap                          # 既存 3.12.6 ラッパー（非変更）
/data/sakagawa/tmp/feat011-colmap-3.11/      # scratch（build.log・build-info.txt・各種ログ。非コミット）
docs/issues/feat-011-colmap-3.11/            # 本案件ドキュメント（コミット対象）
data/dynerf/cut_roasted_beef/                # 検証データ（git 未追跡・流用。FR-003 で colmap/ 再生成）
```

## 1.3 技術スタック

- **vcpkg**（既存、HEAD `36393d1`）: manifest モード（`--x-manifest-root`）+ override で COLMAP 3.11.1 を版固定ビルド。vcpkg バイナリ・downloads・buildtrees は既存ツリーを共有し、install-root のみ別 prefix にする。
- **COLMAP 3.11.1（port-version 4、git-tree `c166234c…`）**。フィーチャ: `core,cuda`（`default-features:false`＝`gui`〔Qt5〕除外、`cuda`＝CUDA 有効。feat-008 の 3.12.6 と同フィーチャ）。CGAL は省略（4DG は mesher 不使用、feat-008 ADR-3 と同じ）。
- **ビルド CUDA**: **11.6**（`/usr/local/cuda-11.6`）第一候補。driver 565.57.01 完全対応・feat-002/008 実証済み。`CUDACXX`/`CUDA_HOME`/`PATH` をビルドシェル限定で 11.6 に設定（グローバルの 12.8 は変えない）。代替 12.8（`/home/sakagawa/cuda/cuda-12.8`）。
- **C++/Fortran コンパイラ**: gcc/g++ 11.4・gfortran 11.4（feat-008 で導入済み。再導入不要）。
- **Python 前処理依存**（既存・新規導入なし）: `.venv` の open3d 0.19.0（`downsample_point.py`）・plyfile（点数検証）。
- **選定理由**: vcpkg manifest+override は vcpkg 公式のバージョン固定機構で、既存ツリーを壊さず特定版を別 prefix に入れられる（dry-run 実証済み）。別 clone+旧 commit より省ディスク・低保守（§2.4 ADR-1）。

## 1.4 各機能の詳細設計

> 全 Python コマンドは `.venv/bin/python` を明示。ビルド・検証の長時間処理はバックグラウンド実行しログ監視する。scratch=`/data/sakagawa/tmp/feat011-colmap-3.11`。

### 1.4.1 FR-001: COLMAP 3.11.1 の vcpkg ソースビルド（別 install prefix）

#### データフロー
- 入力: manifest `vcpkg.json`（override 3.11.1#4 + builtin-baseline **`37c4e62c`**〔2025-11-04、colmap 3.11.1 が baseline・eigen 3.4.1〕）、CUDA 11.6。**当初は現 HEAD `36393d1` で計画したが、その eigen 5.0.1 が COLMAP 3.11.1 とビルド非互換のため旧 commit に変更した（investigation イテレーション1）**。
- 中間: 既存 vcpkg ツリーの downloads（ソース）/buildtrees（コンパイル）。
- 出力: `/data/sakagawa/opt/colmap-3.11/installed/x64-linux/tools/colmap/colmap`（3.11.1）。

#### 処理ロジック（手順）
0. **前提確認**: `gfortran --version` 成功（feat-008 導入済み）。`/usr/local/cuda-11.6/bin/nvcc --version` が 11.6 を返す。`/data/sakagawa/opt/vcpkg/vcpkg version` 成功。`/data` 空き十分（`df -h /data`）。
1. scratch 作成: `mkdir -p /data/sakagawa/tmp/feat011-colmap-3.11`。
2. 3.11.1 ディレクトリ作成: `mkdir -p /data/sakagawa/opt/colmap-3.11`。
3. manifest 作成: `/data/sakagawa/opt/colmap-3.11/vcpkg.json` を以下の内容で作成（意図伝達。実装時に baseline commit を `git -C /data/sakagawa/opt/vcpkg rev-parse HEAD` で再確認）。
   ```json
   {
     "name": "colmap-311-build",
     "version-string": "0.0.1",
     "dependencies": [
       { "name": "colmap", "default-features": false, "features": ["cuda"] }
     ],
     "overrides": [
       { "name": "colmap", "version": "3.11.1", "port-version": 4 }
     ],
     "builtin-baseline": "37c4e62c5ed20ac4cb09884917bde2cbbccf7aa3"
   }
   ```
4. **dry-run で版・依存を最終確認**（ビルド前の安全確認。実証済みだが本番ツリーで再確認）:
   ```bash
   /data/sakagawa/opt/vcpkg/vcpkg install \
     --x-manifest-root=/data/sakagawa/opt/colmap-3.11 \
     --x-install-root=/data/sakagawa/opt/colmap-3.11/installed \
     --triplet x64-linux --dry-run
   ```
   出力に `colmap[core,cuda]:x64-linux@3.11.1#4` が含まれることを確認（含まれなければ override/baseline を見直す）。
5. CUDA 11.6 をビルドシェル限定で設定（feat-008 と同方針。グローバルの 12.8 は変えない）:
   `export PATH=/usr/local/cuda-11.6/bin:$PATH CUDA_HOME=/usr/local/cuda-11.6 CUDACXX=/usr/local/cuda-11.6/bin/nvcc`。`nvcc --version` が 11.6 を返すことを確認。
6. ビルド実行（**長時間。バックグラウンド＋ログ監視**）:
   ```bash
   /data/sakagawa/opt/vcpkg/vcpkg install \
     --x-manifest-root=/data/sakagawa/opt/colmap-3.11 \
     --x-install-root=/data/sakagawa/opt/colmap-3.11/installed \
     --triplet x64-linux \
     > /data/sakagawa/tmp/feat011-colmap-3.11/build.log 2>&1
   ```
7. インストール検証:
   `LD_LIBRARY_PATH=/data/sakagawa/opt/colmap-3.11/installed/x64-linux/lib /data/sakagawa/opt/colmap-3.11/installed/x64-linux/tools/colmap/colmap -h` が終了コード 0、1行目に **`COLMAP 3.11.1`** を含む。
8. 既存 3.12.6 の無傷確認（§FR-001 受け入れ基準）: `command -v colmap` が `~/.local/bin/colmap`、`colmap -h` の版が **3.12.6** のまま。`~/.local/bin/colmap` の中身が feat-008 のまま不変。
9. 再現情報を `build-info.txt`（scratch）に記録: vcpkg commit・採用 CUDA 版（11.6 or 代替）・フィーチャ（`core,cuda`）・3.11.1 の port-version（4）・ビルド所要時間・3.11.1 installed のディスク使用量。

#### 条件分岐（ソースビルドの不確実性に対応。feat-008 の B1〜B5 に準拠）
- **B1: 3.11.1 が現 baseline 依存（ceres 2.2.0 等）でビルド失敗**: `build.log` と `buildtrees/<port>/*.log` で失敗ポートを特定。(a) その依存を manifest の `overrides` に追加して 3.11.1 と整合する版に固定して再試行、(b) 解消困難なら **代替方式（別 vcpkg clone を 3.11.1 が baseline だった commit にチェックアウトして classic install。§2.4 ADR-1 却下案）** にフォールバック。採用法を記録。
- **B2: CUDA `native` arch 検出失敗**（ビルド時に GPU 不可視で arch 空）: カスタム triplet で `set(VCPKG_CMAKE_CONFIGURE_OPTIONS "-DCMAKE_CUDA_ARCHITECTURES=80")`（A100=sm_80）を与える、または代替 CUDA 12.8 を試す。feat-008 と同じ B2 対処。
- **B3: 共有ライブラリ不足で `colmap -h` が `error while loading shared libraries`**: 検証コマンドの `LD_LIBRARY_PATH` に 3.11.1 の `installed/x64-linux/lib` が含まれることを確認（ラッパー〔FR-002〕も同様に前置するため本番は問題にならない）。
- **B4: CGAL を要求（想定外）**: `features:["cuda","cgal"]` に切替えて再ビルド（feat-008 ADR-3 と同じ。4DG の利用範囲では不要の想定）。

#### エラーハンドリング
- `vcpkg install` の非0終了を検出して中断・報告。`build.log`・`buildtrees/<port>/*.log` を確認。
- ビルドは数時間。途中失敗は失敗ポートのログで原因特定し B1〜B4 へ。
- ログ: 全体を `build.log`、再現情報を `build-info.txt` に保存。

#### 境界条件
- 既存 3.12.6 の installed・vcpkg バイナリには触れない（install-root が別、manifest-root が別）。
- 再実行: `/data/sakagawa/opt/colmap-3.11/installed` を残したまま再 `vcpkg install` すれば差分のみビルド（冪等）。完全クリーンには installed を削除してやり直す。
- ディスク: buildtrees が十数 GB に達し得る。`/data`（15TB）に配置。

### 1.4.2 FR-002: 3.11.1 PATH ラッパーの作成と併設

#### データフロー
- 入力: 3.11.1 実体パス（FR-001 出力）。
- 出力: `/data/sakagawa/opt/colmap-3.11/bin/colmap`（実行可能ラッパー）。

#### 処理ロジック（手順）
1. bin ディレクトリ作成: `mkdir -p /data/sakagawa/opt/colmap-3.11/bin`。
2. ラッパー作成（FR-002 の内容）+ `chmod +x`。ファイル名は **`colmap`**（colmap.sh 内の bare `colmap` が PATH 前置で解決できるようにするため。別名 `colmap-3.11` ではない）。
3. 検証:
   - `/data/sakagawa/opt/colmap-3.11/bin/colmap -h` が終了コード 0・版 **3.11.1**。
   - `PATH="/data/sakagawa/opt/colmap-3.11/bin:$PATH" bash -c 'command -v colmap && colmap -h | head -1'` が当該ラッパー＋版 3.11.1。
   - PATH 前置を**外す**と `command -v colmap` が `~/.local/bin/colmap`（3.12.6）を返す（デフォルト非破壊）。

#### エラーハンドリング
- ラッパー実行で共有ライブラリ不足 → `ROOT/lib` が `LD_LIBRARY_PATH` 前置されているか確認（ラッパー内で設定済み）。
- `command -v colmap` がラッパーを指さない → PATH 前置順序を確認（先頭に置く）。

#### 境界条件
- 既存 `~/.local/bin/colmap` は変更しない。3.11.1 を使うのは PATH 前置した実行コンテキストのみ（feat-012/013 と本案件 FR-003）。

### 1.4.3 FR-003: cut_roasted_beef で colmap.sh ... llff を 3.11.1 実走（4DGS 本体非改変）

#### データフロー
- 入力: `data/dynerf/cut_roasted_beef/{poses_bounds.npy, camXX/images/0000.png}`。
- 中間: `sparse_/`・`image_colmap/`・`colmap/database.db`・`colmap/sparse/0/`・`colmap/dense/workspace/`。
- 出力: `data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply`（点数>0）。

#### 処理ロジック（手順）
1. 前提確認: `data/dynerf/cut_roasted_beef/cam00/images/0000.png` 存在（実機確認済み）。欠落時のみ `.venv/bin/python scripts/preprocess_dynerf.py --datadir data/dynerf/cut_roasted_beef`。
2. GPU: colmap.sh:5 は `export CUDA_VISIBLE_DEVICES=0`（非改変）。`nvidia-smi` で GPU0 の空きを確認。**GPU0 が使用中の場合は空くのを待つ**（検証は20枚 SfM+MVS で数分〜十数分と軽量）。colmap.sh は一時的にも改変しない（`git checkout` での復元はユーザーの未コミット変更を破棄する恐れがあるため採らない）。任意 GPU 選択は feat-012 の引数化後に可能。
3. 実行（cwd=リポジトリルート。`PATH` 先頭へ 3.11.1 bin と `.venv/bin` を前置。**colmap.sh は非改変**）:
   ```bash
   cd /data/sakagawa/4DGaussians
   # 実行前確認（bare コマンド解決）
   PATH="/data/sakagawa/opt/colmap-3.11/bin:/data/sakagawa/4DGaussians/.venv/bin:$PATH" \
     bash -c 'command -v colmap; command -v python'   # → 3.11.1 ラッパー / .venv/bin/python
   # 本実行（bash -e で最初の COLMAP コマンド失敗時に即停止。バックグラウンド＋ログ監視）
   PATH="/data/sakagawa/opt/colmap-3.11/bin:/data/sakagawa/4DGaussians/.venv/bin:$PATH" \
     bash -e colmap.sh data/dynerf/cut_roasted_beef llff \
     > /data/sakagawa/tmp/feat011-colmap-3.11/colmap_llff.log 2>&1
   ```
   - 内部手順（`colmap.sh:6-25`、非改変）: `sparse_`/`image_colmap`/`colmap` 削除 → `llff2colmap.py`（`poses_bounds.npy`→COLMAP テキスト、各カメラ `0000.png` を `image_colmap/r_XXX.png` にコピー）→ `colmap` 構築 → `feature_extractor`（**CPU SIFT**: `colmap.sh:15` の `--SiftExtraction.estimate_affine_shape 1 --domain_size_pooling 1` は COLMAP の GPU SIFT が非対応のため CPU に自動フォールバックする〔3.11.1/3.12.6 共通仕様・正常〕）→ `database.py`（cameras 反映）→ `exhaustive_matcher` → **`point_triangulator --clear_points 1`**（3.12.6 で SIGABRT した箇所。3.11.1 は rig 非ネイティブ＋ filename transcribe で完走）→ `image_undistorter` → `patch_match_stereo`（**GPU MVS**: CUDA 必須・GPU0）→ `stereo_fusion` → `fused.ply`。
   - **COLMAP が処理する画像は各カメラ先頭フレームのみ（20枚）**。SfM+MVS は軽量（数分〜十数分）。
4. 検証:
   - `colmap_llff.log` に `Check failed: existing_frame.RigId()==frame.RigId()` や `SIGABRT`/`Aborted` が**無い**こと（rig 非互換解消の直接確認）。`point_triangulator` のログに registered images・三角化点数>0 が出ること。
   - `data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply` が存在し、`.venv/bin/python -c "from plyfile import PlyData; print(len(PlyData.read('data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply')['vertex']))"` が **> 0**。
   - **image↔database 対応の確認（偽陽性排除、FR-003 受け入れ基準）**: 3.11.1 の `point_triangulator --clear_points 1`（`colmap.sh:20`）は `TranscribeImageIdsToDatabase`（`src/colmap/scene/reconstruction.cc:471`、各 image を **filename で database から引き**〔`ReadImageWithName(Name)`〕、reconstruction の image_id を database の image_id に書き換える。name が DB に無ければ `LOG(FATAL_THROW)` で停止）を呼ぶ。よって `sparse_custom/images.txt` の image_id（`llff2colmap.py` が name 順に付与）と `database.db` の image_id（`feature_extractor` が並列登録順に付与）が**数値として食い違うのは正常**で、対応は filename で保たれる。合否は以下で判定する:
     - ① 20画像すべて registered（ログ `Reconstruction with 20 images and N points`、N>0）＝ transcribe が FATAL_THROW なく完了。
     - ② `colmap model_analyzer --path colmap/sparse/0`（3.11.1 ラッパー）の **Mean reprojection error が健全**（目安 < 1.5px。ポーズ↔特徴点が食い違えば数px〜数十pxに跳ねる。実測 **0.852px** = 合格）。
     - ③ 三角化点数 > 0（sparse、実測 4,566点）＋ `fused.ply` 点数 > 0（実測 387,496点）。
     - ※ 旧基準「`database.db` と `sparse_custom` の image_id↔name が数値完全一致（`db==sc`）」は **誤り**だった（`--clear_points 1` の filename transcribe を見落としていた）。詳細は `investigation.md` イテレーション2。
   - `git status --porcelain` で 4DGS 本体（追跡ファイル）に差分が無い（colmap.sh を一時的にも改変しないため。`submodules/{depth-diff-gaussian-rasterization,simple-knn}` の既存ビルド成果物差分〔実行前から存在・本案件と無関係〕を除き差分ゼロ）。

#### エラーハンドリング
- **bare `colmap`/`python` が解決されない**: PATH 前置を忘れると `colmap.sh:8`/`:16` の `python` や各 `colmap` が `command not found`。手順3の事前確認（`command -v`）で担保。
- **依然 rig SIGABRT（B-想定外）**: 3.11.1 は rig 非ネイティブのため発生しない見込み（**実証済み**: 実走で `RigId`/`existing_frame`/`SIGABRT` 出現ゼロ、point_triangulator 完走）。万一同エラーが出たら investigation に記録し、(a) PATH が 3.12.6 を誤って拾っていないか（`colmap -h` で 3.11.1 か）を確認、(b) さらに古い版（3.10 系）or 対処A'（feat-010 investigation の database 整合スクリプト）を再検討。
- **CPU SIFT フォールバックは正常**: `colmap.sh:15` の `--SiftExtraction.estimate_affine_shape 1 --domain_size_pooling 1` は COLMAP の GPU SIFT 非対応オプションのため、feature_extractor は自動的に CPU SIFT になる（3.11.1/3.12.6 共通仕様。llff 経路は元から CPU SIFT が正しい。20枚と小規模で軽量）。feat-008 で GPU SIFT が動いたのは south-building の `mapper` 経路〔これらオプション無し〕のため。dense（`patch_match_stereo`）のみ CUDA(GPU0) を使う。
- `fused.ply` が空/極小: 抽出画像（FR-002 相当）の妥当性・カメラ数を確認し investigation に記録。
- **`bash -e` 実行**: `colmap.sh` は `set -e` を持たないが、本案件は `bash -e colmap.sh ...` で実行するため最初の COLMAP コマンド失敗時に即停止する（`sparse/0` 空による後段の連鎖 Aborted ログに惑わされない）。なお `colmap.sh:12` の `mkdir`（`-p` なし）は直前の `rm -rf $workdir/colmap` 後のため通常成功するが、万一 `colmap` ディレクトリが残存して mkdir 失敗→`bash -e` 停止した場合は当該ディレクトリを手動削除して再実行する。
- **B5: image_id 数値不一致は正常（実証で確定）**: feature_extractor が database に振る image_id が `sparse_custom/images.txt`（name 順）と数値で食い違っても、`point_triangulator --clear_points 1` の `TranscribeImageIdsToDatabase` が **filename で image_id を database に揃え直す**ため幾何対応は保たれる（実測 reprojection error 0.852px）。よって旧 B5 が懸念した「点群はできても幾何が壊れる偽陽性」は本経路では起きない。**真に異常なケース**は (a) reprojection error が大きい（>数px）、(b) `Image with name ... does not exist in database` の FATAL_THROW（name 不一致）、(c) registered images < 20 — いずれかが出た場合のみ investigation に記録し、PATH が 3.12.6 を拾っていないか（`colmap -h`）等を確認する。

#### 境界条件
- 再実行: `colmap.sh` 冒頭が `sparse_`/`image_colmap`/`colmap` を `rm -rf` して作り直すため冪等。旧 feat-010 の手動生成物も上書きされる。
- GPU 引数: colmap.sh 非改変のため第3引数は受け付けない（GPU0 固定）。任意 GPU 選択は feat-012 で引数化後に可能。

### 1.4.4 FR-004: ダウンサンプル（downsample_point.py）

#### データフロー
- 入力: `data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply`。
- 出力: `data/dynerf/cut_roasted_beef/points3D_downsample2.ply`（0 < 点数 ≤ 40,000）。

#### 処理ロジック（手順）
1. 実行:
   ```bash
   .venv/bin/python scripts/downsample_point.py \
     data/dynerf/cut_roasted_beef/colmap/dense/workspace/fused.ply \
     data/dynerf/cut_roasted_beef/points3D_downsample2.ply
   ```
2. 内部挙動（`downsample_point.py`、非改変）: `voxel_size=0.02` から開始し、点数が 40,000 を超える間 `voxel_down_sample` を繰り返し `voxel_size += 0.01`。40,000 点以下で `write_point_cloud`。
3. 検証: `points3D_downsample2.ply` を `PlyData.read` し `0 < 点数 ≤ 40000`。

#### エラーハンドリング
- `fused.ply` が無い → FR-003 未完。前段の完走を前提とする。
- 点数が極端に少ない（< 数百点）→ COLMAP 品質を疑い investigation に記録（学習自体は走る）。
- open3d import 失敗 → feat-008/009 で導入済み（0.19.0）。`.venv/bin/python` を明示。

#### 境界条件
- `fused.ply` が既に 40,000 点以下 → voxel ループに入らずそのまま書き出し（許容）。

### 1.4.5 FR-005: 文書化（導入手順・運用・依存記録）

#### 出力（文書更新先と内容）
1. **本設計書 §1.4.1〜1.4.4**: vcpkg manifest+override ビルド・ラッパー作成・検証手順の正本（`vcpkg.json` 内容・ビルドコマンド・CUDA 設定・ラッパー内容・colmap.sh llff 実行コマンド形）。`/clear` 後もこれだけで再現可能とする。
2. **`docs/TECH_STACK.md`**: 「COLMAP」節に **「3.11.1 併設（feat-011）」サブ節**を追加。記載事項: 導入方式（vcpkg manifest+override、別 install-root `/data/sakagawa/opt/colmap-3.11/installed`）・採用版（3.11.1#4）・採用 CUDA（11.6）・フィーチャ（`core,cuda`）・ラッパー方式（`/data/sakagawa/opt/colmap-3.11/bin/colmap` を PATH 前置）・**3.12.6 との使い分け**（既定 `~/.local/bin/colmap`=3.12.6 はそのまま。DyNeRF/multipleview の `colmap.sh`/`multipleviewprogress.sh` 実走時のみ 3.11.1 を PATH 前置）・選定理由（rig 非互換回避、§2.4 ADR）。
3. **`CLAUDE.md`**: ①実行環境表の COLMAP 行に「3.11.1 を別 prefix 併設（feat-011、rig 非互換回避）」を追記。②「マルチGPU運用ルール」または新規の短節に、3.11.1 を使う実行コマンド形（`PATH="/data/sakagawa/opt/colmap-3.11/bin:/data/sakagawa/4DGaussians/.venv/bin:$PATH" bash -e colmap.sh <dir> llff`）と、**3.12.6/3.11.1 の使い分け**を明記。③「データセット」節の DyNeRF/multipleview に「COLMAP は 3.11.1 を使う」を補足。
4. **`docs/BACKLOG.md`**: feat-011 を Closed に更新（手動テスト合格後）。`CLAUDE.md` のディレクトリ構成に `docs/issues/feat-011-colmap-3.11/` を反映。
5. **`requirements.lock.txt`**: 本案件は Python 依存の追加・変更が無いため**再生成不要**（依存変化なし）。

#### 受け入れ確認
- 上記1〜4が更新され、3.11.1 の導入・運用が `/clear` 後も再現可能であること。

## 1.5 状態遷移

本案件は GUI／常駐処理を持たない（バッチ的なツール導入・検証）。明示的な状態遷移設計は不要。唯一の分岐は §1.4.1 のビルド失敗時フォールバック（B1〜B4）で、各節に記載済み。

## 1.6 ファイル・ディレクトリ設計

```
/data/sakagawa/opt/colmap-3.11/             # 新規（リポジトリ外）
├── vcpkg.json                              # manifest（override 3.11.1#4 + builtin-baseline 37c4e62c）
├── bin/colmap                              # 3.11.1 ラッパー（chmod +x、colmap.sh の bare colmap 用）
└── installed/x64-linux/                    # 3.11.1 install-root（3.12.6 と完全分離）
    ├── tools/colmap/colmap                 #   3.11.1 実体
    └── lib/*.so                            #   3.11.1 同梱依存

/data/sakagawa/tmp/feat011-colmap-3.11/     # scratch（非コミット）
├── build.log                               # vcpkg ビルドログ
├── build-info.txt                          # 再現情報（vcpkg commit/CUDA版/フィーチャ/所要時間/ディスク）
└── colmap_llff.log                         # FR-003 colmap.sh llff の実行ログ

data/dynerf/cut_roasted_beef/               # 検証データ（git 未追跡・流用）
├── poses_bounds.npy / cam00.mp4..cam19.mp4 # 既存（20カメラ）
├── camXX/images/0000.png..0299.png         # 既存（抽出済み）
├── sparse_/ image_colmap/ colmap/          # FR-003 で colmap.sh が rm -rf → 再生成
│   └── dense/workspace/fused.ply           #   FR-003 出力（点数>0）
└── points3D_downsample2.ply                # FR-004 出力（≤40k点）

# 既存（非変更）: /data/sakagawa/opt/vcpkg/（3.12.6）, ~/.local/bin/colmap（3.12.6 ラッパー）
```

- **命名**: ログは `<step>.log`（build / colmap_llff）。設定ファイルは `vcpkg.json` のみ新規（COLMAP は CLI 引数駆動）。
- **設定ファイル形式**: `vcpkg.json`（JSON、vcpkg manifest スキーマ）。スキーマは §1.4.1 手順3 のとおり。

## 1.7 インターフェース定義

- **vcpkg manifest install**: `/data/sakagawa/opt/vcpkg/vcpkg install --x-manifest-root=<dir> --x-install-root=<dir>/installed --triplet x64-linux [--dry-run]`。
- **3.11.1 ラッパー `/data/sakagawa/opt/colmap-3.11/bin/colmap`**: 引数を素通しで 3.11.1 実体へ転送し、`LD_LIBRARY_PATH` に 3.11.1 `installed/x64-linux/lib` を前置する。
- **3.11.1 を使う実行コンテキスト**: `PATH="/data/sakagawa/opt/colmap-3.11/bin:/data/sakagawa/4DGaussians/.venv/bin:$PATH"` を前置して `bash -e colmap.sh <workdir> llff`（bare `colmap`→3.11.1、bare `python`→`.venv`。`-e` で最初の COLMAP 失敗時に即停止）。
- **使用 COLMAP サブコマンド**（colmap.sh 経由、3.11.1）: `feature_extractor`, `exhaustive_matcher`, `point_triangulator`, `image_undistorter`, `patch_match_stereo`, `stereo_fusion`。
- **Python**: `.venv/bin/python scripts/downsample_point.py <fused.ply> <out.ply>`。
- **4DGS 本体の関数・スクリプト・シグネチャは一切変更しない**（colmap.sh:5 の GPU 引数化は feat-012）。

## 1.8 ログ・デバッグ設計

- 本案件のログは scratch（`/data/sakagawa/tmp/feat011-colmap-3.11/`）に集約。
- レベル運用:
  - INFO 相当: 各ステップ開始・終了コード・3.11.1 版確認・生成物パス・ビルド再現情報・所要時間。
  - WARNING 相当: ビルド依存のリトライ（B1）、CUDA arch 固定（B2）、GPU SIFT 失敗時の検討。
  - ERROR 相当: `vcpkg install` 失敗、`colmap -h` 不通、`point_triangulator` の rig SIGABRT 再発、`fused.ply` 非生成、3.12.6 への巻き込み破壊。
- 監視点:
  - ビルド: `build.log` の失敗ポート、最終 `colmap 3.11.1` の生成。
  - colmap.sh llff: `colmap_llff.log` で `point_triangulator` が rig エラーなく通るか、`stereo_fusion` の `fused.ply` 出力点数、各 colmap コマンドの版（3.11.1）。
  - 進捗バー（`\r`）でログが肥大する場合は要点抽出（`grep -aiE 'rig|abort|triangulat|fused|error|registered'`）。

## 2.4 設計判断の記録（ADR）

### ADR-1: 導入方式 = vcpkg manifest + override（既存ツリー再利用・別 install-root）
- **採用**: 既存 vcpkg ツリー（HEAD `36393d1`）で manifest モード（`vcpkg.json` に `overrides:[{colmap,3.11.1#4}]` + `builtin-baseline:37c4e62c`〔2025-11-04 commit〕）を使い、`--x-install-root` を別 prefix（`/data/sakagawa/opt/colmap-3.11/installed`）に指定してビルドする。**ビルド実証済み**（`COLMAP 3.11.1 ... with CUDA`）。builtin-baseline は当初 vcpkg HEAD `36393d1` を指定したが、その eigen 5.0.1 が 3.11.1 とビルド非互換（`covariance.cc` の `DenseBase::nonZeros()` 削除）だったため、**3.11.1 が baseline だった旧 commit `37c4e62c`**（eigen 3.4.1#1・ceres 2.2.0#5 が整合）に変更した（investigation イテレーション1）。
- **却下A（別 vcpkg を新規 clone し、3.11.1 が baseline だった頃の commit にチェックアウトして classic install）**: 確実だが (1) vcpkg ツリーをもう一式（十数 GB）持つ、(2) ports 全体が旧 commit に巻き戻り依存ライブラリ群も当時版になる（現環境との整合確認が増える）、(3) downloads/buildtrees キャッシュを共有できず再DL・再ビルド、(4) 二重保守。manifest+override なら現ツリー1つで colmap だけ版固定でき省ディスク・低保守。**ただしビルド失敗時（ADR-1 採用案で 3.11.1 が現 baseline 依存と非互換）はこの却下Aをフォールバックに用いる**（§1.4.1 B1）。
- **却下B（既存 3.12.6 を 3.11.1 へ完全置換）**: feat-008 で実証済みの 3.12.6 を失う。将来 3.12 系が必要になった際に後戻り不能で、HyperNeRF(feat-009)は点群DLで COLMAP 非使用だが他の用途で 3.12.6 を使う可能性を潰す。併設の方が安全。
- **却下C（conda-forge / システムパッケージ）**: CLAUDE.md の conda 回避方針に反する（feat-008 ADR-1 と同じ）。

### ADR-2: 版 = COLMAP 3.11.1 port-version 4（最新 port-version）
- **採用**: versions DB に存在する 3.11.1 の port-version 0〜4 のうち**最新の 4**（git-tree `c166234c…`）。
- **却下（port-version 0〜3）**: port-version はパッケージング修正の積み上げで、最新 4 が最も修正が入った状態。古い port-version を選ぶ理由が無い。
- **根拠**: 3.11.1 は rig 必須化（3.12）前の最終 minor 版。これより新しい 3.12.x は rig ネイティブで本問題が再発、これより古い版（3.10 等）を選ぶ必要は無い（3.11.1 で rig 非ネイティブ要件を満たす）。

### ADR-3: ビルド CUDA = 11.6 第一候補（feat-008 と同方針）
- **採用**: CUDA Toolkit **11.6**（`/usr/local/cuda-11.6`）。ビルドシェル限定で `CUDACXX`/`CUDA_HOME`/`PATH` を 11.6 に設定（グローバルの 12.8 は変えない）。
- **根拠**: driver 565.57.01 が 11.6 を完全対応し、feat-002（CUDA 拡張）・feat-008（COLMAP 3.12.6）で 11.6 + gcc 11.4 ビルドを実証済み。COLMAP は torch と独立だが「driver 完全対応版」で実行時リスク最小。
- **却下（12.8 を第一）**: 12.8 は driver 565 で minor version compatibility 依存（feat-008 ADR-4 と同じ理由）。代替として B フォールバックに温存。

### ADR-4: 3.11.1 運用 = 専用 bin の `colmap` ラッパー + PATH 前置（デフォルト 3.12.6 温存）
- **採用**: `/data/sakagawa/opt/colmap-3.11/bin/colmap`（名前は `colmap`）を作り、3.11.1 を使うときだけ PATH 先頭に前置する。既存 `~/.local/bin/colmap`（3.12.6）はデフォルトのまま変更しない。ラッパーの `LD_LIBRARY_PATH` は**空値分岐**で設定する（既存値が空のとき末尾 `:` が残りカレントディレクトリが共有ライブラリ探索対象に入る事故を防ぐ。FR-002 のラッパー本体参照）。既存 feat-008 の 3.12.6 ラッパーは末尾コロン型だが、それは feat-008 のスコープのため本案件では触れず（温存）、本案件の新規ラッパーのみ改善版を採る。
- **却下A（デフォルト `~/.local/bin/colmap` を 3.11.1 に差し替え）**: feat-008/009 の検証手順や将来の 3.12 利用に影響。3.12.6 を既定に保ち、必要時のみ 3.11.1 を前置する方が副作用が局所化する。
- **却下B（colmap.sh 内の `colmap` を 3.11.1 の絶対パスに書き換え）**: 4DGS 本体改変が増える（本案件は本体完全非改変が制約）。PATH 前置なら非改変で実現できる。
- **却下C（別名 `colmap-3.11` だけ作る）**: colmap.sh は bare `colmap` を呼ぶため、別名では拾えない。`colmap` 名のラッパーを専用 bin に置き PATH で切り替えるのが、非改変と分離を両立する唯一の方法。

### ADR-5: 検証データ = cut_roasted_beef（colmap.sh llff 経路）。south-building ではない
- **採用**: 旧 feat-010 の cut_roasted_beef（LLFF・単一カメラ共有 → `point_triangulator`）で検証する。
- **却下（feat-008 の south-building）**: south-building は `colmap mapper`（増分 SfM）経路で、単一カメラ共有 sparse モデルを `point_triangulator` に渡さないため **rig 非互換が発生しない**＝本問題の検証にならない。rig SIGABRT は `llff2colmap.py`＋`point_triangulator` 経路に固有なので、その経路（`colmap.sh ... llff`）で検証する必要がある。
- **根拠**: 本案件のゴールは「3.11.1 で rig 非互換が解消されること」の実証であり、再現経路そのもの（cut_roasted_beef + colmap.sh llff）で確認するのが妥当。データは取得・抽出済みで追加コストゼロ。

### ADR-6: feat-011 は 4DGS 本体完全非改変（colmap.sh:5 GPU 引数化は feat-012）
- **採用**: 本案件では `colmap.sh` を含む 4DGS 本体を1行も改変しない（一時改変も行わない）。検証（FR-003）は colmap.sh:5 のハードコード（GPU0）に従い、GPU0 使用中の場合は**空くのを待つ**（検証は20枚 SfM+MVS で軽量なため待てる）。
- **却下（GPU0 使用中に colmap.sh:5 を一時改変→`git checkout` で復元）**: 「1行も改変しない」と矛盾し、かつ `git checkout colmap.sh` がユーザーの未コミット変更を破棄する恐れがある。検証は軽量なので待機で足り、一時改変は不要。
- **背景**: ユーザー決定（2026-06-23）。`colmap.sh:5` の `export CUDA_VISIBLE_DEVICES=0` → `${3:-0}` 引数化は、実際に任意 GPU 選択が必要になる DyNeRF 前処理案件（feat-012）で、その文脈とともに正式導入・レビュー・記録する。
- **却下（feat-011 に引数化を含める）**: feat-011 のスコープ（3.11.1 導入）に本体改変が混じり、変更の文脈が曖昧になる。feat-011 の検証は軽量（20枚 SfM+MVS、数分）で GPU0 が空くタイミングを取れるため、引数化を前倒しする必然性が無い。
- **根拠**: 変更は「それが必要になる案件」で行うのが、改変追跡（CLAUDE.md「オリジナルコードの変更点」）と案件スコープの明確化に資する。

### ADR-7: 配置 = `/data/sakagawa/opt/colmap-3.11/`（vcpkg.json + bin + installed）。buildtrees/downloads は既存ツリー共有
- **採用**: 3.11.1 専用ディレクトリ `/data/sakagawa/opt/colmap-3.11/` に manifest・ラッパー・install-root を集約。vcpkg バイナリ・downloads・buildtrees は既存 `/data/sakagawa/opt/vcpkg` を共有（install-root のみ分離）。
- **却下（全て別 vcpkg ツリーに隔離）**: downloads（ソース tarball）を再取得し buildtrees を二重に持つことになり非効率。install-root を分けるだけで 3.12.6 の installed には触れず、共有部分（ソースキャッシュ）の再利用で省時間・省ディスク。
- **根拠**: vcpkg は `--x-install-root` で成果物のみ別 prefix にでき、buildtrees/downloads はツリー共通で安全に共有できる（既存 installed を上書きしない）。
