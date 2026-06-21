# feat-008 機能設計書: COLMAP環境構築（vcpkgソースビルド版）

本書は `docs/DESIGN_STANDARD.md` に準拠する。本書のコードスニペットは「意図の伝達」が目的であり、そのままコピー実行するものではない（実装時にパス・枚数・コミット等を確認すること）。

> **改訂履歴**
> - **2026-06-19 初版**: 公式 CUDA版 AppImage 前提で作成（→ 破棄）。
> - **2026-06-19 全面改訂（本版）**: 実装着手時の実機検証で **COLMAP は公式に Linux バイナリ（AppImage 含む）を一切配布していない**ことが判明（GitHub Releases 3.3〜4.0.4 全確認）。導入方式を **vcpkg ソースビルド** に変更（ユーザー承認）。本書はその新方式の正本。

---

## 1.1 対応要求マッピング

| 要求ID | 設計セクション |
|--------|----------------|
| FR-001 COLMAP の vcpkg ソースビルド・PATH解決 | §1.4.1 |
| FR-002 GPUサブコマンド動作（ヘッドレス対応） | §1.4.2 |
| FR-003 open3d/scikit-image 導入 | §1.4.3 |
| FR-004 疎再構成パイプライン完走 | §1.4.4 |
| FR-005 dense スモーク確認（Should） | §1.4.5 |
| FR-006 文書化 | §1.4.6 |
| 全体構成・配置 | §1.2, §1.6 |
| 技術選定理由（ADR） | §2.4 |

## 1.2 システム構成

本案件は 4DGaussians 本体コードを改変せず、**外部ツール（COLMAP バイナリ）を vcpkg でソースビルドして導入し、Python 前処理依存を整備**する。

```
[導入対象]
  vcpkg（Microsoft 製ソースパッケージマネージャ）
    └ /data/sakagawa/opt/vcpkg/                    # git clone（ビルドツリーが巨大なため /data に配置）
        ├ vcpkg                                     # bootstrap で生成する本体バイナリ
        ├ downloads/tools/{cmake,ninja}-*           # vcpkg が自動取得するビルドツール（本機に cmake/ninja 無し）
        ├ buildtrees/                               # 各依存のビルド作業（巨大・一時）
        ├ packages/                                 # 各依存のステージング
        └ installed/x64-linux/
            ├ tools/colmap/colmap                   # ビルドされた COLMAP 実体
            └ lib/*.so                              # 同梱依存の共有ライブラリ
  COLMAP（vcpkg ポート、版＝レジストリ準拠 3.12.6）を CUDA 有効・GUI 除外でビルド
  ~/.local/bin/colmap                               # PATH 上のラッパー（→ installed の colmap を exec）

[uv .venv に追加する Python 依存]
  open3d         # scripts/downsample_point.py が import
  scikit-image   # multipleviewprogress.sh の LLFF(imgs2poses) が利用

[本体側の利用箇所（本案件では実走行しない／参照のみ）]
  convert.py                : colmap {feature_extractor, exhaustive_matcher, mapper, image_undistorter}
  colmap.sh                 : colmap {feature_extractor, exhaustive_matcher, point_triangulator,
                                      image_undistorter, patch_match_stereo, stereo_fusion}
  multipleviewprogress.sh   : colmap {feature_extractor, exhaustive_matcher, mapper,
                                      image_undistorter, patch_match_stereo, stereo_fusion}
  scripts/downsample_point.py : open3d による点群ダウンサンプリング
```

- **依存方向**: 本体スクリプト → `colmap`（PATH）→ ラッパー → installed の colmap。Python 前処理 → `.venv` の open3d/skimage。循環なし。
- **本案件のゴール**: 上記「導入対象」「Python 依存」を整備し、§1.4.4 の最小パイプラインで疎再構成が通ること。各本体スクリプトの実走行は feat-009/010/011。

### ディレクトリ構成（実パス）

```
/data/sakagawa/opt/vcpkg/                   # vcpkg 一式（リポジトリ外・新規作成。十数 GB に達し得る）
~/.local/bin/colmap                          # PATH ラッパー（新規作成）
/data/sakagawa/tmp/feat008-colmap/           # scratch（検証データ・ログ・記録・非コミット）
  ├ dataset/                                 # 小規模サンプル画像（input/）
  ├ work/                                    # COLMAP 作業ディレクトリ（database.db, sparse/, dense/）
  ├ build.log                                # vcpkg ビルドログ
  └ *.log / build-info.txt                   # 各サブコマンドのログ・再現情報（vcpkg commit/CUDA版/feature）
docs/issues/feat-008-colmap/                 # 本案件ドキュメント（コミット対象）
```

## 1.3 技術スタック

- **vcpkg**（git クローン、commit はビルド時に固定・記録）: COLMAP と全依存をソースビルドする。conda 非依存・ユーザー権限・sudo 不要。
- **COLMAP**（vcpkg ポート、版＝レジストリ準拠 3.12.6）。フィーチャ: `core,cuda`（GUI〔Qt5〕除外、CUDA 有効）。CGAL は本案件の利用範囲外のため既定で省略（§2.4 ADR-3）。
- **CUDA Toolkit 11.6 を第一候補**（`/usr/local/cuda-11.6`）でビルド、**12.8（`/home/sakagawa/cuda/cuda-12.8`）は代替**。COLMAP は torch とは独立した別バイナリだが、**driver 565.57.01 が完全対応し torch cu116 で実証済みの 11.6** を第一に採る（gcc 11.4 と互換。CUDA 11.6 + gcc 11.4 のビルドは feat-002 の CUDA 拡張ビルドで実証済み）。12.8 は driver 565 で minor version compatibility に依存するため代替に温存（§2.4 ADR-4）。A100 = sm_80。
- **C++ コンパイラ**: gcc/g++ 11.4（システム）。
- **Fortran コンパイラ**: **gfortran**（system、`sudo apt-get install` で導入）。COLMAP は SuiteSparse(CHOLMOD)/Ceres 経由で BLAS/LAPACK に依存し、vcpkg の Linux LAPACK 提供ポート `lapack-reference` が Reference LAPACK を Fortran ソースからビルドするため必須。本機未導入のため導入する（非機能要求の権限例外・管理者承認済み。§2.4 ADR-6・investigation.md）。
- **Python 前処理依存**（uv 管理 `.venv`、Python 3.10）:
  - `open3d`（**既存 0.19.0 を維持**、Python 3.10・numpy<2 対応）— `downsample_point.py` の点群I/O・voxel down sample。新規導入はしない（§1.4.3）。
  - `scikit-image`（候補 **0.22.0**、py>=3.9・numpy>=1.22 対応）— LLFF `imgs2poses.py` が依存。
  - 導入は **`uv pip install`（追加的）のみ**。`uv sync`/`uv pip sync` 禁止。**`numpy==1.23.5` をアンカーして固定**し、既存（torch 1.13.1+cu116 / numpy 1.23.5 / mmcv 1.6.0）を壊さないことを §1.4.3 で確認。
- **検証データ**: COLMAP 公式 GitHub Release（tag 3.11.1）のサンプル `south-building.zip`（実在確認済み）。`demuc.de` には依存しない。

## 1.4 各機能の詳細設計

### 1.4.1 FR-001: COLMAP の vcpkg ソースビルドと PATH 解決

#### データフロー
- 入力: vcpkg リポジトリ（`https://github.com/microsoft/vcpkg`）、CUDA Toolkit（第一候補 11.6、代替 12.8）。
- 中間: bootstrap した vcpkg 本体 → `vcpkg install` による依存・COLMAP のソースビルド → `installed/x64-linux/tools/colmap/colmap`。
- 出力: `~/.local/bin/colmap`（PATH 上で解決される実行可能ラッパー）。

#### 処理ロジック（手順）
**前提（手順0）: Fortran コンパイラ導入**。LAPACK 依存（`lapack-reference`）のビルドに必須。`sudo apt-get install -y gfortran`（GPUサーバー管理者承認済み、2026-06-19。非機能要求の権限例外）。**この sudo コマンドはユーザーが実行する**（Claude は sudo パスワードを入力できない）。導入後 `gfortran --version` が成功することを確認してからビルドへ進む。未導入のままビルドすると `lapack-reference` が `Unable to find a Fortran compiler` で失敗する（investigation.md イテレーション1）。
1. 配置ディレクトリ作成: `mkdir -p /data/sakagawa/opt`。
2. vcpkg 取得: `git clone https://github.com/microsoft/vcpkg /data/sakagawa/opt/vcpkg`。再現性のため `git -C /data/sakagawa/opt/vcpkg rev-parse HEAD` を `build-info.txt` に記録。
3. bootstrap: `/data/sakagawa/opt/vcpkg/bootstrap-vcpkg.sh -disableMetrics`。これで `vcpkg` 本体バイナリが生成される。**cmake は不要**（vcpkg 本体はプリビルド取得 or 同梱ソースでビルド）。生成された `vcpkg` の存在を確認。
4. CUDA 環境を 11.6 に設定（このビルドシェル限定。driver 565 完全対応・torch cu116 実証済みの第一候補）: `PATH=/usr/local/cuda-11.6/bin:$PATH`、`CUDA_HOME=/usr/local/cuda-11.6`、`CUDACXX=/usr/local/cuda-11.6/bin/nvcc`。`nvcc --version` が 11.6 を返すことを確認。代替の 12.8 を使う場合は `/home/sakagawa/cuda/cuda-12.8` に読み替える。
5. ビルド実行（**長時間。バックグラウンド実行しログ監視**）:
   `/data/sakagawa/opt/vcpkg/vcpkg install 'colmap[core,cuda]:x64-linux' > /data/sakagawa/tmp/feat008-colmap/build.log 2>&1`
   - `core` = default feature（`gui`=Qt5）を除外（ヘッドレスで GUI 不要、Qt5 ソースビルド回避）。
   - `cuda` = `CUDA_ENABLED ON` + `CUDA_ARCHITECTURES "native"`（ビルドマシンの A100 を検出し sm_80 を選択）。ビルド時に GPU が不可視で `native` 検出に失敗する、または driver 要求エラーが出る場合は、カスタム triplet で `CMAKE_CUDA_ARCHITECTURES=80` を固定する（A100=sm_80 を明示。下記 B2 参照）。
   - vcpkg は `downloads/tools/` に cmake/ninja を自動取得して各依存をビルドする。
6. インストール検証: `/data/sakagawa/opt/vcpkg/installed/x64-linux/tools/colmap/colmap -h` が終了コード 0 でサブコマンド一覧を表示すること。
7. PATH ラッパー作成: `~/.local/bin/colmap` を以下の内容で作成し `chmod +x`。
   ```bash
   #!/usr/bin/env bash
   # COLMAP (vcpkg build) launcher
   ROOT=/data/sakagawa/opt/vcpkg/installed/x64-linux
   export LD_LIBRARY_PATH="$ROOT/lib:$LD_LIBRARY_PATH"
   exec "$ROOT/tools/colmap/colmap" "$@"
   ```
8. PATH 検証: `hash -r` 後 `command -v colmap` が当該ラッパーを指すこと、`colmap -h` が終了コード 0、バージョンを記録。
9. 再現情報の記録: `build-info.txt` に vcpkg commit・COLMAP ポート版・採用 CUDA 版（第一候補 11.6。代替 12.8 を使った場合はその版）・採用フィーチャ（`core,cuda`）・ビルド所要時間・`installed` ディスク使用量を残す。

#### 条件分岐（ソースビルドの本質的不確実性に対応）
- **B1: bootstrap が cmake/make を要求して失敗する**: vcpkg は通常自前ツールを取得するが、失敗時は cmake をユーザー権限導入（公式 cmake-linux-x86_64 tarball を `/data/sakagawa/opt/cmake` に展開し PATH 前置、または `uv pip install cmake ninja` で `.venv` 経由に PATH を通す）。どちらで解決したか記録。
- **B2: CUDA `native` arch の検出に失敗**（ビルド時に GPU が見えず arch 空）: ビルドシェルで `CUDA_VISIBLE_DEVICES` を空にしない。なお解決しない場合は **カスタム triplet** を作成し `set(VCPKG_CMAKE_CONFIGURE_OPTIONS "-DCMAKE_CUDA_ARCHITECTURES=80")` を与える（A100=80 を明示）、または `cuda` の代わりに `cuda-redist`（`all-major`）フィーチャを使う。採用法を記録。
- **B3: 特定依存のビルド失敗**（gcc 11.4 / Ubuntu 22.04 / CUDA 11.6 起因のコンパイルエラー）: `build.log` で失敗ポートを特定。(a) vcpkg を既知良好の commit/release ブランチ（例 `release` タグ）に固定して再試行、(b) 当該ポートのみ代替 CUDA（12.8）やコンパイラ設定を試す、(c) 解消困難なら中断・報告。いずれも記録。
- **B4: `colmap` 実行時に共有ライブラリ不足**（`error while loading shared libraries`）: ラッパーの `LD_LIBRARY_PATH` に `installed/x64-linux/lib` が含まれることを確認。vcpkg は通常 RPATH を埋めるため不要だが、保険として設定済み。
- **B5: 本案件の必須サブコマンドが CGAL を要求**（想定外）: `colmap[core,cuda,cgal]` に切替えて再ビルド（CGAL は GMP/MPFR/Boost 依存でビルドが重い。4DG の利用範囲〔feature〜stereo_fusion〕では不要の想定だが、要求された場合のみ追加）。

#### エラーハンドリング
- `git clone`/`bootstrap`/`vcpkg install` の非0終了を検出して中断・報告。`build.log` を確認。
- ビルドは長時間（数時間）。バックグラウンド実行し、`tail`/監視で進捗を追う。途中失敗は失敗ポートのログ（`buildtrees/<port>/*.log`）で原因特定。
- ログ: ビルド全体を `build.log`、再現情報を `build-info.txt` に保存。

#### 境界条件
- `~/.local/bin` 不在時は `mkdir -p`（PATH には既存登録あり＝`printenv PATH` で先頭に確認済み）。
- 既存 `colmap` がある場合（再実行）: ラッパーを上書きし `hash -r`。
- ディスク: vcpkg ツリーは十数 GB。`/data`（15TB 空き）に配置。

### 1.4.2 FR-002: GPU を用いるサブコマンドの動作確認（ヘッドレス対応）

#### 背景（ヘッドレス制約）
本機は DISPLAY 未設定・Xvfb 無し。COLMAP の **CUDA 有効ビルドでは GPU SIFT が CUDA バックエンドで動作**し、OpenGL ディスプレイ無しでも feature_extractor/matcher の `use_gpu 1` が通る見込み（公式 FAQ は「ディスプレイ無し＋CUDA 無しなら全ステップ CPU」と述べ、CUDA があれば GPU 実行可能を示唆）。実測で失敗する場合は CPU SIFT にフォールバックする。**dense の `patch_match_stereo` は CUDA を直接使い OpenGL 不要のためヘッドレス影響なし。**

#### データフロー
- 入力: §1.4.4 の小規模 `input/` 画像と作業用 `database.db`。
- 出力: 特徴抽出済み `database.db`。実行モード（GPU/CPU）をログに残す。

#### 処理ロジック（feature と matcher を独立に GPU→CPU フォールバック）
**feature_extractor と exhaustive_matcher は別々に GPU 可否を判定する**（片方が GPU 成功・他方が GPU 失敗でも、失敗側のみ CPU に切替える）。採用モードは `feature=<gpu|cpu>, matcher=<gpu|cpu>` の形式で記録する。GPU 選択は CLAUDE.md マルチGPUルールに従い `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N`（空き GPU N、本検証時点では 0 か 1）を付与する。

1. feature_extractor を GPU で試行:
   `colmap feature_extractor --database_path work/database.db --image_path dataset/input --ImageReader.single_camera 1 --ImageReader.camera_model OPENCV --SiftExtraction.use_gpu 1`
   （並行して別シェルで `nvidia-smi` を取得し A100 に負荷が乗るか確認。）
   - **成功（終了コード0）** → `feature=gpu`。
   - **失敗（OpenGL/EGL/display 起因、終了コード≠0）** → `work/database.db` を削除し `--SiftExtraction.use_gpu 0` で再実行 → `feature=cpu`。
2. exhaustive_matcher を、feature の採否と独立に GPU で試行:
   `colmap exhaustive_matcher --database_path work/database.db --SiftMatching.use_gpu 1`
   - **成功（終了コード0）** → `matcher=gpu`。
   - **失敗（終了コード≠0）** → **`database.db` は保持したまま**（feature の抽出結果を捨てない）`--SiftMatching.use_gpu 0` で再実行 → `matcher=cpu`。
3. いずれの採用モードでも両コマンドが終了コード0で完了することを確認。採用モードと GPU 失敗があった事実・回避を `investigation.md`／§1.4.6 に記録。

#### エラーハンドリング
- GPU 失敗の判定: 終了コード≠0 かつ stderr に `OpenGL`/`EGL`/`display`/`Qt`/`CUDA` 等のキーワード。→ 該当サブコマンドのみ CPU フォールバック。
- 注意: feature_extractor の CPU 再実行時は重複特徴を避けるため `database.db` を削除して作り直す。matcher の CPU 再実行時は抽出済み特徴を再利用するため `database.db` を保持する（削除しない）。
- CPU でも失敗する場合: データ/コマンド不備を疑い中断・報告。
- ログ: `feature.gpu.log`/`feature.cpu.log`、`match.gpu.log`/`match.cpu.log` を分けて保存し、採用モードを明記。

#### 境界条件
- 画像が極端に少ない/特徴が出ない場合は §1.4.4 の枚数増加フォールバックと連動。
- 複数人共用GPU: 実行前に空きGPUを確認し `CUDA_VISIBLE_DEVICES=N` を付与（CLAUDE.md マルチGPU運用ルール）。COLMAP は学習用 `--port` を使わないためポート競合なし。

### 1.4.3 FR-003: 前処理 Python 依存の整備（open3d 維持確認 + scikit-image 導入）

#### 採用バージョン方針（事前確定）
- **open3d は既に `.venv` に 0.19.0 で導入済み**（`requirements.lock.txt` 記載・実機確認済み）。**新規導入せず既存 0.19.0 を維持**し、非破壊（import 成功・numpy 1.23.5 維持）を確認するに留める。
- **未導入の scikit-image のみ新規導入**する（候補 **0.22.0**、Python 3.10・numpy<2 対応）。
- **numpy は 1.23.5 を固定したまま**解決する。インストールコマンドの先頭に `numpy==1.23.5` を明示してアンカーする。
- **検証は全て `.venv/bin/python` を明示**して行う。

#### データフロー
- 入力: `uv pip install "numpy==1.23.5" "scikit-image==0.22.0"`（`.venv` のインタプリタに対して。**追加的**・numpy アンカー付き。**open3d は既存 0.19.0 を維持するためコマンドに含めない**）。
- 出力: `.venv` に scikit-image が新規導入され、open3d 0.19.0 が維持された状態。`requirements.lock.txt` の更新スナップショット。

#### 処理ロジック（手順）
1. 導入前スナップショット: `.venv/bin/python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"`、`.venv/bin/python -c "import open3d; print(open3d.__version__)"`（既存 0.19.0 を確認）、`uv pip freeze > requirements.lock.before.txt`（scratch）で現状を記録。
2. 追加導入: `uv pip install "numpy==1.23.5" "scikit-image==0.22.0"`（**追加的**・numpy アンカー。**open3d は含めない**＝既存 0.19.0 を維持）。解決された具体的バージョンを記録する。
3. 既存環境非破壊の確認（全て `.venv/bin/python`）:
   - `import numpy` が **1.23.5 のまま**であること（巻き上げが起きたら B1 へ）。
   - `import torch` が `1.13.1+cu116 True` を返すこと。
   - `import open3d` が **0.19.0 のまま**であること（scikit-image 導入で巻き戻り・巻き上げが起きていないこと）。
   - `import mmcv`（1.6.0）が import できること。
4. import 検証（全て `.venv/bin/python`）:
   - `import open3d; print(open3d.__version__)` 成功（既存 0.19.0）。
   - `import skimage; print(skimage.__version__)` 成功（新規導入版）。
   - `scripts/downsample_point.py`（引数なし）→ **`IndexError`（sys.argv 不足）に到達**すること（= open3d の import は成功。`ModuleNotFoundError` が出たら失敗）。
5. 記録: `docs/TECH_STACK.md` に scikit-image（用途・選定理由・**解決された具体的バージョン**）を追記し、open3d は既存 0.19.0 を維持した旨を記す。`uv pip freeze > requirements.lock.txt` で再生成。

#### 条件分岐
- **B1: scikit-image 0.22.0 が numpy 1.23.5 を巻き上げる／既存（torch・open3d 0.19.0）と非互換**: numpy==1.23.5 を保ったまま scikit-image の版を1段ずつ下げて再試行（候補: 0.21系・0.20系）。**open3d は 0.19.0 を維持し、その巻き戻り・巻き上げが起きない scikit-image 版を選ぶ**。**numpy を 1.23.5 から動かさない**。採用版を TECH_STACK.md に記録。
- **B2: open3d import が共有ライブラリ不足で失敗**（`libGL`/`libgomp` 等）: 本機は `libGL.so.1` 存在。不足時のみ `ldd` で欠落特定→報告（sudo不可のためシステム導入はしない。多くは wheel 同梱で解決）。

#### 境界条件
- `downsample_point.py` は引数2個（入力ply/出力ply）必須。検証では「引数不足で IndexError」をもって import 成功と判定する。

### 1.4.4 FR-004: 小規模実データでの疎再構成パイプライン完走

#### 入力データの取得
- COLMAP 公式 GitHub Release（tag 3.11.1）の `south-building.zip` を scratch に取得・展開（実在確認済み: `https://github.com/colmap/colmap/releases/download/3.11.1/south-building.zip`）。取得不可時の代替: 同 release の `gerrard-hall.zip`。
- 展開後の画像（`images/`）から先頭 **15〜20 枚**を `dataset/input/` にコピー（連続撮影のため先頭群は重なりが確保される）。
- データ型: 実写画像、各 1枚 = 1視点。`single_camera 1`（全画像同一内部パラメータ）想定。

#### データフロー
```
dataset/input/*.jpg
  → colmap feature_extractor  → work/database.db（特徴）
  → colmap exhaustive_matcher → work/database.db（対応）
  → colmap mapper             → work/sparse/0/{cameras,images,points3D}.bin
```

#### 処理ロジック（手順）
1. 作業ディレクトリ準備: `rm -rf work && mkdir -p work/sparse`（再実行の冪等性）。
2. feature_extractor: §1.4.2 で採用したモード（GPU or CPU）で実行。`--ImageReader.single_camera 1 --ImageReader.camera_model OPENCV`。
3. exhaustive_matcher: 採用モードで実行。
4. mapper: `colmap mapper --database_path work/database.db --image_path dataset/input --output_path work/sparse`（出力は `work/sparse/0/` に作られる）。
5. 検証（**実数カウントで合否判定**。bin の存在・サイズのみでの合格は禁止）:
   - 3コマンドすべて終了コード0。
   - `work/sparse/0/cameras.bin`・`images.bin`・`points3D.bin` が存在する。
   - **登録画像数・3D点数を実数で確認する**: `colmap model_analyzer --path work/sparse/0` の出力（`Registered images` / `Points` 等）を読み、**登録画像数 ≥ 2 かつ 3D点数 > 0** を確認。
     - 代替: `colmap model_converter --input_path work/sparse/0 --output_path work/sparse/0_txt --output_type TXT` で `images.txt`/`points3D.txt` を生成し、**各ファイル冒頭のヘッダコメント行を読む**:
       - 登録画像数 = `images.txt` の `# Number of images: N, ...` の N（`grep -m1 'Number of images' images.txt`）。
       - 3D点数 = `points3D.txt` の `# Number of points: N, ...` の N（`grep -m1 'Number of points' points3D.txt`）。
     - **注意**: COLMAP の `images.txt` は**1画像あたり2行**であり、非コメント行数をそのまま数えると登録画像数が2倍になる。**行数カウントで代替してはならない**（ヘッダの N を読むこと。公式フォーマット: https://colmap.github.io/format.html ）。
   - いずれの手段でも「登録画像数 ≥ 2 かつ 3D点数 > 0」を満たさない場合は B1 へ。

#### 条件分岐
- **B1: mapper の登録画像が 0〜1 枚（再構成失敗）**: 入力枚数を 20→30→40 と増やして再試行。最大40枚で成立しない場合は別サンプル（`gerrard-hall`）へ切替。
- **B2: feature が極端に少なくマッチ0**: GPU/CPU モードを §1.4.2 の他方へ切替、それでも0なら画像セットを差し替え。

#### エラーハンドリング
- 各コマンドの非0終了を検出して中断、`*.log` を確認。`FileNotFoundError`／空 database はパス・前段未実行を疑う。
- ログ: `feature.log`・`match.log`・`mapper.log` を scratch に保存。

#### 境界条件
- 入力0枚: 前段でデータ存在を確認。
- ディスク: 疎再構成は軽量（数十MB）。dense（§1.4.5）実行時のみ増大。

### 1.4.5 FR-005: 密再構成（dense）スモーク確認（Should）

#### データフロー
```
work/sparse/0 + dataset/input
  → colmap image_undistorter   → work/dense/{images,sparse}
  → colmap patch_match_stereo  → work/dense/stereo/*（CUDA直・GL不要）
  → colmap stereo_fusion       → work/dense/fused.ply
```

#### 処理ロジック（手順）
1. `colmap image_undistorter --image_path dataset/input --input_path work/sparse/0 --output_path work/dense --output_type COLMAP`。
2. `colmap patch_match_stereo --workspace_path work/dense --workspace_format COLMAP --PatchMatchStereo.geom_consistency true`。
3. `colmap stereo_fusion --workspace_path work/dense --workspace_format COLMAP --input_type geometric --output_path work/dense/fused.ply`。
4. 検証: 3コマンド終了コード0、`work/dense/fused.ply` が存在・非空（サイズ > 0）。

#### エラーハンドリング・位置づけ
- 本要求は **Should**。失敗（例: VRAM不足・パッチマッチの数値エラー）時は制約を記録し、**FR-004（必須）の合否には影響させない**。dense は feat-009/010 で本走行するため、ここでは前倒し確認に留める。
- `patch_match_stereo` は CUDA を直接使うためヘッドレス無関係。VRAM は A100 40GB で小規模データには十分。

#### 境界条件
- dense は画像枚数・解像度で計算量が増える。スモークでは §1.4.4 の小規模セットをそのまま使う。

### 1.4.6 FR-006: 導入手順とヘッドレス運用注記の文書化

#### 出力（文書更新先と内容）
1. **本設計書 §1.4.1〜1.4.5**: vcpkg ソースビルド・検証手順の正本（vcpkg 取得元・配置パス・`vcpkg install` コマンド・CUDA設定・PATH ラッパー・検証コマンド）。`/clear` 後もこれだけで再現可能とする。
2. **`docs/TECH_STACK.md`**: 「COLMAP」節を追加（vcpkg ソースビルド・版・配置パス・採用 CUDA 版〔第一候補 11.6〕・採用フィーチャ・**ビルド前提の gfortran 導入〔sudo apt、管理者承認〕**・PATH ラッパー方式・選定理由＝§2.4 ADR）。scikit-image（新規）の用途・選定理由・解決版を追記し、open3d は既存 0.19.0 を維持した旨を記す。
3. **`requirements.lock.txt`**: `uv pip freeze` で再生成（scikit-image 反映。open3d 0.19.0 は既存）。
4. **`CLAUDE.md`**: 実行環境表の「colmap 未インストール」を実態に更新（vcpkg ビルド導入済み・パス・ヘッドレス注記）。必要なら「実シーン前処理時の COLMAP 運用」短節を追加。
5. **ヘッドレス運用注記（必須記載事項）**: 「GPU SIFT（feature_extractor/matcher の `use_gpu 1`）は CUDA 有効ビルドでヘッドレスでも動作する見込み。失敗時は `use_gpu 0`（CPU）で実行する。dense（patch_match_stereo）は CUDA 直で影響なし」を明記。§1.4.2 の実測結果を反映。

#### 受け入れ確認
- 上記1〜4が更新され、ヘッドレス注記が記載されていること。`docs/BACKLOG.md` の feat-008 を Closed に更新（実装・手動テスト合格後）。

## 1.5 状態遷移

本案件は GUI／ステートフルな常駐処理を持たない（バッチ的なツール導入・検証）。明示的な状態遷移設計は不要。唯一の分岐は §1.4.2 の「GPU SIFT 成否による GPU/CPU モード採択」で、これは当該節に分岐として記載済み。

## 1.6 ファイル・ディレクトリ設計

- **vcpkg**: `/data/sakagawa/opt/vcpkg/`（git クローン。`installed/x64-linux/tools/colmap/colmap` にバイナリ、`installed/x64-linux/lib` に共有ライブラリ）。
- **PATH ラッパー**: `~/.local/bin/colmap`（実行可能、§1.4.1 の内容）。
- **scratch**: `/data/sakagawa/tmp/feat008-colmap/`（`dataset/`・`work/`・`build.log`・`build-info.txt`・各種 `*.log`）。**リポジトリにコミットしない**。
- **命名**: ログは `<step>.log`（`build/feature/match/mapper/dense`）。
- 設定ファイルは新規作成しない（COLMAP は CLI 引数駆動。vcpkg は既定の x64-linux triplet。B2 でカスタム triplet を作る場合のみ追加）。

## 1.7 インターフェース定義

- **`colmap` CLI**（PATH 解決）: `colmap <subcommand> [options]`。使用サブコマンド = `help(-h)`, `feature_extractor`, `exhaustive_matcher`, `mapper`, `image_undistorter`, `patch_match_stereo`, `stereo_fusion`, `model_analyzer`, `model_converter`。
- **PATH ラッパー `~/.local/bin/colmap`**: 引数を素通しで COLMAP 実体へ転送し、`LD_LIBRARY_PATH` に installed/lib を前置する（§1.4.1）。
- **Python 依存**: `import open3d` / `import skimage` が `.venv` で解決可能であること。

## 1.8 ログ・デバッグ設計

- 本案件のログは scratch（`/data/sakagawa/tmp/feat008-colmap/`）に集約。
- レベル運用:
  - INFO 相当: 各ステップ開始・終了コード・採用モード（GPU/CPU）・生成物パス・ビルド再現情報。
  - WARNING 相当: GPU SIFT 失敗→CPU フォールバック、mapper 枚数増加リトライ、CUDA arch/依存ビルドのリトライ。
  - ERROR 相当: clone/bootstrap/`vcpkg install` 失敗、`colmap -h` 不通、import 失敗、必須コマンドの非0終了。
- フォーマット: ビルドは `build.log`、各検証は `<step>.log` に生出力。要約（終了コード・採用モード・生成物・所要時間・vcpkg commit/CUDA版）を `build-info.txt`／手動テスト報告に記す。

## 2.4 設計判断の記録（ADR）

### ADR-1: COLMAP 導入方式 = vcpkg ソースビルド
- **採用**: vcpkg で `colmap[core,cuda]:x64-linux` を依存込みソースビルド。
- **却下0（公式 prebuilt バイナリ／AppImage）**: **存在しない**。COLMAP は GitHub Releases（3.3〜4.0.4）で Linux バイナリを一切配布していない（Windows/.zip と一部旧 Mac のみ）。初版設計が前提とした `colmap-3.12.6-linux-x86_64-cuda.AppImage` は実在しないことを実機で確認。
- **却下1（conda-forge / micromamba）**: 確実だが、プロジェクトが回避している conda エコシステムを持ち込む（CLAUDE.md 制約）。
- **却下2（依存を手動で個別ソースビルド）**: Boost/Ceres/CGAL/SuiteSparse/METIS/GMP/MPFR 等の依存の依存まで sudo 無しで揃えるのは工数・失敗リスクが過大。
- **根拠**: vcpkg は sudo 不要・ユーザー権限・conda 非依存で、依存を全自動ソースビルドする。`core` で GUI(Qt5) を外し、`cuda` で CUDA を有効化できる。ユーザー承認済み（2026-06-19）。

### ADR-2: GUI（Qt5）除外 = `core` フィーチャ指定
- **採用**: `colmap[core,cuda]`（`core` で default feature `gui` を除外）。
- **却下（GUI 込み `colmap[cuda]`）**: ヘッドレスで GUI は不要。Qt5 のソースビルドは重く（X11 依存・時間）、リスクと時間を増やす。
- **根拠**: 本案件は CLI のみ使用。GUI 除外でビルドを軽量化しヘッドレスに整合。`core` が default features を除外することは vcpkg 公式仕様で確認済み。

### ADR-3: CGAL = 既定で省略
- **採用**: `cgal` フィーチャを付けない。
- **却下（`cgal` 付与）**: CGAL は COLMAP の Delaunay/Poisson **meshing** に用いられるが、4DGaussians の利用範囲（feature_extractor〜mapper〜stereo_fusion で `fused.ply` まで）では mesher を使わない。CGAL は GMP/MPFR/Boost 依存でビルドが重い。
- **分岐**: 必須サブコマンドが CGAL を要求した場合のみ `colmap[core,cuda,cgal]` に切替（§1.4.1 B5）。

### ADR-4: ビルド CUDA = 11.6 を第一候補（12.8 は代替）
- **採用**: CUDA Toolkit **11.6**（`/usr/local/cuda-11.6`）でビルドを第一候補とする。12.8（`/home/sakagawa/cuda/cuda-12.8`）は代替（B3 フォールバック）。
- **根拠（11.6 第一）**: (1) **driver 565.57.01 が CUDA 11.6 を完全対応**し、本機で torch cu116 が実証済み（実行時 driver エラーのリスクが最も低い）。(2) **CUDA 11.6 + gcc 11.4 のビルドは feat-002 の CUDA 拡張ビルドで実証済み**（CUDA 11.6 は GCC 11 を公式サポートする）。(3) COLMAP は torch と無関係の独立バイナリのため torch cu116 との一致は本来不要だが、「driver が完全対応する版」という観点で 11.6 が最も確実。
- **却下（12.8 を第一にしない）**: 12.8 は **driver 565.57.01 では CUDA minor version compatibility に依存**する（CUDA 12.8 GA の対応 driver は 570 系。565 では PTX・新 driver 機能に触れると `cudaErrorCallRequiresNewerDriver` 系で実行時に落ち得る）。ビルドが通っても実行時 driver エラーでは無意味なため、driver 完全対応の 11.6 を優先。12.8 は 11.6 で依存ビルドが失敗した場合の代替として温存（§1.4.1 B3）。
- **注記**: 旧版 ADR-4 の「CUDA 11.6 は gcc 11 を公式サポートしないことがある」は誤り（CUDA 11.6 は GCC 11 対応。feat-002 で gcc 11.4 + CUDA 11.6 ビルド実証済み）のため訂正した。

### ADR-5: 検証データ = COLMAP 公式 GitHub Release のサンプル
- **採用**: release tag 3.11.1 の `south-building.zip`（実在確認済み）の先頭 15〜20 枚。
- **却下（`demuc.de` のサンプル）**: 初版が参照した配布元は到達・実在の確証が薄い（前提捏造の経緯）。GitHub Release は入手再現性が高い。
- **却下（D-NeRF 合成画像の流用）**: 合成・単一物体で SfM が安定しにくく、実シーン前処理の妥当性確認にならない。

### ADR-6: Fortran コンパイラ = `sudo apt-get install gfortran`（管理者承認の例外）
- **背景**: 実装着手時、`vcpkg install colmap[core,cuda]` が依存 `lapack-reference` のビルドで失敗。原因は**本機に Fortran コンパイラ（gfortran）が一切無い**こと（gcc も `f951` 非配備）。COLMAP は SuiteSparse(CHOLMOD)/Ceres 経由で BLAS/LAPACK 必須で、vcpkg の Linux LAPACK 提供 `lapack-reference` は Reference LAPACK を Fortran ソースからビルドするため Fortran 必須（investigation.md イテレーション1）。
- **採用**: **`sudo apt-get install -y gfortran` でシステム導入**（GPUサーバー管理者の承認を取得、2026-06-19）。要求の「sudo 原則不可」を、ビルド必須のシステムツールである gfortran に限り例外的に緩和する。
- **却下1（userspace 展開: `apt-get download` + `dpkg-deb -x`）**: sudo 不要だが、gfortran ドライバが `f951` を `GCC_EXEC_PREFIX` 経由で探す設定が脆く、手間も大きい。管理者承認が得られたため、よりクリーンで再現性の高い sudo 導入を採る。
- **却下2（system LAPACK 利用で vcpkg ビルド回避）**: 本機に `liblapack.so.3`/`libblas.so.3` は存在するが実行時 `.so.3` のみ。vcpkg は依存を自前ビルドする方針で system 切替が困難、COLMAP の ceres/suitesparse が `lapack`/`blas` ポートを要求するため回避は煩雑・不確実。
- **却下3（ceres を suitesparse 無し・LAPACK off でビルド）**: colmap vcpkg port が suitesparse 前提のため port 改変が必要で、上流非改変方針に反し機能（CHOLMOD）も落ちる。
- **注記**: OpenBLAS をビルドして LAPACK を得る案も、OpenBLAS の LAPACK 部分が Fortran を要するため本ブロッカーの回避にならない。
