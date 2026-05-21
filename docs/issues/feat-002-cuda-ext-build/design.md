# feat-002 機能設計書: サブモジュール初期化・CUDA拡張ビルド

本設計書は `docs/DESIGN_STANDARD.md` に準拠する。**実装判断ゼロの原則**に従い、実装者が迷わないようコマンド列・環境変数・検証・失敗時対処を具体値で記述する。/clear 後でも本書のみで実装できることを目標とする。

---

## 1.1 対応要求マッピング

| 要求ID | 設計セクション |
|--------|---------------|
| FR-001（サブモジュール初期化） | §1.4.1 |
| FR-002（rasterizer ビルド） | §1.4.2 |
| FR-003（simple-knn ビルド） | §1.4.3 |
| FR-004（import 統合検証） | §1.4.4 |
| FR-005（lock 再生成） | §1.4.5 |
| 共通（エラー・境界条件） | §1.4.6 |

---

## 1.2 システム構成

本案件はビルド作業であり、新規にアプリコードを書かない。関与する要素は以下。

```
4DGaussians/
├── .venv/                                  # feat-001 で構築（Python 3.10 / torch 1.13.1+cu116 / setuptools 80.10.2）
├── submodules/
│   ├── depth-diff-gaussian-rasterization/  # FR-001で取得 → FR-002でビルド
│   │   ├── setup.py                        #   torch.utils.cpp_extension で CUDAExtension を定義
│   │   ├── diff_gaussian_rasterization/    #   Pythonパッケージ（__init__.py + ビルド後 _C*.so）
│   │   └── third_party/glm/                #   ネストしたサブモジュール（GLMヘッダ）。--recursive で取得
│   └── simple-knn/                         # FR-001で取得 → FR-003でビルド
│       ├── setup.py                        #   torch.utils.cpp_extension で CUDAExtension を定義
│       └── simple_knn/                     #   Pythonパッケージ（__init__.py + ビルド後 _C*.so）
├── requirements.lock.txt                   # FR-005で更新（版の正本）
├── gaussian_renderer/__init__.py:14        # 利用側: from diff_gaussian_rasterization import ...
├── scene/gaussian_model.py:22              # 利用側: from simple_knn._C import distCUDA2
├── scene/dataset_readers.py:485            # 利用側: from diff_gaussian_rasterization import ... as Camera
└── merge_many_4dgs.py:33                    # 利用側: from diff_gaussian_rasterization import GaussianRasterizationSettings, GaussianRasterizer
```

**依存関係**: FR-001（取得）→ FR-002/FR-003（ビルド、相互に独立）→ FR-004（検証）→ FR-005（記録）。FR-002 と FR-003 は順不同。

> **注記**: サブモジュールの内部ディレクトリ構成（`diff_gaussian_rasterization/`・`simple_knn/`・`third_party/glm/` 等）は標準的な 3DGS 系ラスタライザの構成に基づく想定。実体は FR-001 取得後に確認し、差異があれば §1.4.6 の E1 に従う。**import 名（`diff_gaussian_rasterization` / `simple_knn._C`）は4DGS本体コードの実利用で確定済み**（上記利用側）であり、これが判定の正。

---

## 1.3 技術スタック

| 項目 | 値 | 選定理由 |
|------|-----|---------|
| Python | 3.10（`.venv/bin/python`） | feat-001 で構築済み |
| torch | 1.13.1+cu116 | ビルド時に `torch.utils.cpp_extension` を提供。CUDA拡張の ABI を決める |
| ビルド用CUDA | `/usr/local/cuda-11.6`（nvcc 11.6.124） | torch cu116 とメジャー・マイナー一致。新規インストール不要 |
| ホストコンパイラ | gcc/g++ 11.4.0 | CUDA 11.6 は gcc ≤ 11 対応 → 適合 |
| setuptools | 80.10.2（< 81、pkg_resources 同梱） | feat-001 で導入。`--no-build-isolation` 時の build backend |
| GPU アーキ | A100 = sm_80 | CUDA 11.6 で完全サポート |
| ビルド方式 | `uv pip install -e`（editable） | 公式README手順に一致。`.so` をソースツリーに生成 |

**事前確認済み（2026-05-21、本セッション）**:
- `/usr/local/cuda-11.6/bin/nvcc --version` → `release 11.6, V11.6.124` ✅
- `gcc --version` → `11.4.0` ✅
- `.venv/bin/python --version` → `Python 3.10.16` ✅
- `.venv` の setuptools=80.10.2（<81）/ wheel=0.47.0 が存在 ✅、**ninja は未インストール**（torch が distutils にフォールバックしビルド可能。ビルドが数分かかりうる）
- CUDA の所在（E4 の切り分け誤誘導を防ぐため実機事実を明記）:
  - グローバルにアクティブな 12.8 は `$CUDA_HOME=~/cuda/current`（→ `~/cuda/cuda-12.8`）であり、`which nvcc` もこれを指す。
  - `/usr/local` 配下には別系統が併存する（実機: `/usr/local/cuda` → **11.6**、`/usr/local/cuda-11` → 11.8、`/usr/local/cuda-11.7`、`/usr/local/cuda-11.8`、`/usr/local/cuda-12` → 12.3、`/usr/local/cuda-12.3`。`/usr/local/cuda-12.8` は存在しない）。
  - したがって E4 で「別CUDAが拾われた」際は、グローバルの 12.8（`~/cuda/current`）／`/usr/local/cuda`（=11.6）／`/usr/local/cuda-12`（=12.3）のいずれかを所在で切り分ける。本案件のビルドは曖昧さ排除のため常に `/usr/local/cuda-11.6` を**絶対パスで明示**する（TECH_STACK.md の二段リンク注意と整合）。

---

## 1.4 各機能の詳細設計

> **コマンド表記の前提**: すべて `/data/sakagawa/4DGaussians` をカレントディレクトリとして実行する。`uv pip` には**必ず `--python .venv/bin/python` を付与**する（本マシンは別環境の bashrc が `VIRTUAL_ENV` を常時アクティブにしているため）。

> **データフローの記法**: 本案件はアプリのデータ処理ではなく**ビルド作業**であるため、`DESIGN_STANDARD.md` §1.4 のデータフローは数値テンソルの型・shape・dtype ではなく、**入出力のソース／ヘッダ／コンパイル成果物（`.so`）／lock ファイルのパスと形式**で記述する。

### 1.4.1 FR-001: サブモジュールの初期化

**処理ロジック**
```bash
git submodule update --init --recursive
```
- `--init` で `.gitmodules` 記載の未初期化サブモジュールを登録・取得する。
- `--recursive` で**ネストしたサブモジュール**（depth-diff-gaussian-rasterization が参照する `third_party/glm`）も取得する。これが無いと nvcc が `glm/glm.hpp: No such file or directory` でビルド失敗するため必須。
- `.gitmodules` には `SIBR_viewers`（可視化ビューア）も定義されているが、本案件では不要。`git submodule update --init --recursive` は全サブモジュールを取得しようとする。SIBR_viewers の取得は環境構築の必須要件ではないが、取得自体は害がない（ビルドはしない）。**取得に失敗しても本案件の対象2つが取得できていれば続行する**（§1.4.6 E5）。

**データフロー**
- 入力: `.gitmodules`（URL: ingra14m/depth-diff-gaussian-rasterization, gitlab.inria.fr/bkerbl/simple-knn）、ネットワーク。
- 出力: `submodules/depth-diff-gaussian-rasterization/`、`submodules/simple-knn/`、`submodules/depth-diff-gaussian-rasterization/third_party/glm/`。

**受け入れ確認**
```bash
git submodule status                                                      # 対象2つの行頭の '-' が消える
test -f submodules/depth-diff-gaussian-rasterization/setup.py && echo "rasterizer setup.py ok"
test -f submodules/simple-knn/setup.py && echo "simple-knn setup.py ok"
ls submodules/depth-diff-gaussian-rasterization/third_party/glm/ 2>/dev/null \
  && echo "glm present" || echo "glm MISSING (要 §1.4.6 E2)"
```
- 取得後に**実体のディレクトリ構成・パッケージ名・glm の有無を目視確認**し、§1.2 の想定と差があれば §1.4.6 E1/E2 に従う。
- **glm 欠落時の標準フォロー**: 上記 `ls ... third_party/glm/` が空（"glm MISSING"）の場合、初回の `--init --recursive` が第1段取得前の `.gitmodules` を再帰解決できずネストを取りこぼした可能性がある。**そのまま下記（E2 二段目）を標準手順として実行**する:
  ```bash
  git -C submodules/depth-diff-gaussian-rasterization submodule update --init --recursive
  ```
  再度 glm を確認し、なお欠落するなら中断・報告（§1.4.6 E2）。

### 1.4.2 FR-002: depth-diff-gaussian-rasterization のビルド

**設計の核心（build isolation 無効化と CUDA 上書き）**:
1. サブモジュールの `setup.py` は `from torch.utils.cpp_extension import CUDAExtension, BuildExtension` を import する。build isolation 有効（デフォルト）だと隔離環境に torch が無く `ModuleNotFoundError: torch` で失敗する。→ **`--no-build-isolation` で `.venv` の torch を使う**。
2. `torch.utils.cpp_extension` は `CUDA_HOME`（無ければ PATH 上の `nvcc`）から CUDA を探す。グローバルの `CUDA_HOME`（12.8）のままだと torch(11) と nvcc(12) のメジャー版不一致でビルドエラー／重大警告。→ **ビルドコマンドのインライン環境変数で 11.6 に上書き**する。
3. `--no-build-isolation` 時の build backend は `.venv` の setuptools 80.10.2（pkg_resources 同梱）を使う。

**処理ロジック**
```bash
CUDA_HOME=/usr/local/cuda-11.6 PATH=/usr/local/cuda-11.6/bin:$PATH \
  uv pip install --python .venv/bin/python --no-build-isolation \
  -e submodules/depth-diff-gaussian-rasterization
```
- `CUDA_HOME=...` と `PATH=...` は**このコマンド限りのインライン環境変数**（永続化しない＝グローバル設定を汚さない）。
- `PATH` 先頭に 11.6 の bin を置くことで、`which nvcc` フォールバック時も 11.6 が選ばれる。
- `-e`（editable）で `.so` をソースツリー（`submodules/depth-diff-gaussian-rasterization/diff_gaussian_rasterization/`）に生成する。公式手順 `pip install -e ...` と等価。

**データフロー**
- 入力: サブモジュールソース + `third_party/glm` ヘッダ + `.venv` の torch + cuda-11.6 の nvcc。
- 出力: `.venv` に editable リンク（`__editable__.diff_gaussian_rasterization*.pth` 等。具体的な形式は uv 版に依存するため、インストール成否の判定は §1.4.4 の import 成否を正とする）と、ソースツリー内の `_C.cpython-310-x86_64-linux-gnu.so`。

**受け入れ確認**
```bash
.venv/bin/python -c "import diff_gaussian_rasterization; print('rasterizer import ok')"
.venv/bin/python -c "from diff_gaussian_rasterization import GaussianRasterizationSettings, GaussianRasterizer; print('rasterizer symbols ok')"
```

### 1.4.3 FR-003: simple-knn のビルド

**処理ロジック**（FR-002 と同方式。`third_party/glm` は不要）
```bash
CUDA_HOME=/usr/local/cuda-11.6 PATH=/usr/local/cuda-11.6/bin:$PATH \
  uv pip install --python .venv/bin/python --no-build-isolation \
  -e submodules/simple-knn
```

**データフロー**
- 入力: サブモジュールソース + `.venv` の torch + cuda-11.6 の nvcc。
- 出力: `.venv` に editable リンクと、ソースツリー内の `simple_knn/_C.cpython-310-x86_64-linux-gnu.so`。

**受け入れ確認**
```bash
.venv/bin/python -c "import simple_knn._C; print('simple_knn import ok')"
.venv/bin/python -c "from simple_knn._C import distCUDA2; print('distCUDA2 ok')"
```

### 1.4.4 FR-004: import 統合検証

**処理ロジック**: 4DGS本体の実利用と同一の import 文をまとめて実行する。
```bash
.venv/bin/python - <<'PY'
import torch
import diff_gaussian_rasterization
import simple_knn._C
from diff_gaussian_rasterization import GaussianRasterizationSettings, GaussianRasterizer
from simple_knn._C import distCUDA2
print("OK: CUDA ext import |",
      "torch", torch.__version__, "| cuda", torch.version.cuda,
      "| device", torch.cuda.get_device_name(0) if torch.cuda.is_available() else "N/A")
PY
```
- 全 import が通り `OK: CUDA ext import | torch 1.13.1+cu116 | cuda 11.6 | device ...A100...` が出れば合格。
- import 時に `ImportError: undefined symbol` や CUDA ランタイム不整合が出る場合は §1.4.6 E4 に従う。

**データフロー**
- 入力: なし（`.venv` 内の torch・拡張・ドライバ）。
- 出力: 検証結果の標準出力（合格 / ImportError）。

### 1.4.5 FR-005: 依存スナップショットの更新

**処理ロジック**
```bash
uv pip freeze --python .venv/bin/python > requirements.lock.txt
git add requirements.lock.txt
```
- editable 2点が `-e file:///data/sakagawa/4DGaussians/submodules/...` または `... @ file://...` 形式で追記される（uv の出力形式に従う。どちらでも可）。
- コミットはユーザー承認後／手動テスト合格後（§1.5）。

**データフロー**
- 入力: `.venv` の `uv pip freeze` 出力。
- 出力: `requirements.lock.txt`（CUDA拡張2点を含む全量＝版の正本）。

**受け入れ確認**
```bash
test -s requirements.lock.txt && echo "lock non-empty"
grep -iE 'diff[-_]gaussian[-_]rasterization' requirements.lock.txt && echo "rasterizer recorded"
grep -iE 'simple[-_]knn' requirements.lock.txt && echo "simple-knn recorded"
grep -E '^torch==1\.13\.1\+cu116' requirements.lock.txt && echo "torch unchanged"
grep -E '^mmcv==1\.6\.0' requirements.lock.txt && echo "mmcv unchanged"
grep -E '^numpy==1\.23\.5' requirements.lock.txt && echo "numpy unchanged"
```
- editable 行は `uv pip freeze` の表記形式差で機械 grep が空振りすることがあるため、空振り時は lock を**目視確認**する（import 成功＝§1.4.4 を合否の正とする）。
- torch/mmcv/numpy の3行が退行（消失・別版化）していないことを確認する（これらは `uv pip freeze` が正規化版で出力するため grep は通常通るが、表記差で空振りした場合は editable 行と同様に目視確認する）。退行があれば §1.4.6 E6。
- **退行の網羅確認**: 上記3行に限らず想定外の版変化が無いことを、更新前の `requirements.lock.txt`（feat-001 版、git 管理）との差分で確認する。`git diff requirements.lock.txt` の追加・変更行が CUDA 拡張2点（と、E7 で追加した場合の build backend ツール）のみであることを目視する。

### 1.4.6 エラーハンドリングと境界条件

各エラーは「検出方法 → 動作」で定義する。**計画外の対処が必要になった場合は中断してユーザーに報告**し、`docs/issues/feat-002-cuda-ext-build/investigation.md` に記録する。

| ID | 想定エラー | 検出方法 | 動作 |
|----|-----------|---------|------|
| E1 | サブモジュールの実体構成が §1.2 想定と異なる（パッケージ名・ディレクトリ） | FR-001 取得後の `ls` / `setup.py` 確認 | import 名は確定済み（`diff_gaussian_rasterization`/`simple_knn._C`）なので**判定は変えない**。事実を investigation.md に記録する。変更してよいのは `uv pip install -e` の**引数パス（ビルド対象ディレクトリ）が実体と異なる場合に実パスへ置換する**ことのみ。それ以外（import 名・判定・本体4DGSコード）は変えない |
| E2 | `third_party/glm` が無く nvcc が `glm/glm.hpp: No such file` で失敗 | FR-001 の glm 確認 / ビルドログ | 判定の正は**取得経路を問わず `submodules/depth-diff-gaussian-rasterization/third_party/glm/` 配下のヘッダ（`glm/glm.hpp` 等）が存在すること**。まず `git -C submodules/depth-diff-gaussian-rasterization submodule status` で glm がネスト submodule として登録されているか確認し、(a) 登録あり → `git -C submodules/depth-diff-gaussian-rasterization submodule update --init --recursive` で取得、(b) 登録なしだが既にヘッダが存在 → ベンダリング済みとみなし続行、(c) 登録もヘッダも無く欠落 → 中断・報告 |
| E3 | build isolation 有効のまま `ModuleNotFoundError: torch` でビルド失敗 | ビルドログ | `--no-build-isolation` の付与漏れ。付与して再実行（§1.4.2/1.4.3 のとおり） |
| E4 | CUDA メジャー版不一致（torch 11 vs nvcc 12）でビルドエラー／重大警告、または import 時 `undefined symbol`/CUDA 不整合 | ビルドログ / FR-004 の ImportError | `CUDA_HOME`/`PATH` のインライン上書き漏れ、または別CUDAが拾われた。`/usr/local/cuda-11.6` 明示で再ビルド。実行時 `LD_LIBRARY_PATH`（グローバルが12.8）が干渉する場合は当該検証コマンド限りで `LD_LIBRARY_PATH=/usr/local/cuda-11.6/lib64:$LD_LIBRARY_PATH` を付与（TECH_STACK 監視点）。通常は torch 同梱libが優先され不要 |
| E5 | SIBR_viewers など対象外サブモジュールの取得失敗 | FR-001 のログ | 対象2つ（rasterizer / simple-knn）が取得できていれば**続行**。SIBR_viewers は非要件 |
| E6 | lock 再生成で torch/mmcv/numpy が退行 | FR-005 受け入れ確認の grep | editable ビルドで依存が引き込まれた可能性。中断・報告。`requirements.lock.txt`（feat-001版、git管理）から差分を確認し是正方針を investigation.md に記録 |
| E7 | `--no-build-isolation` 下で build backend ツール（wheel 等）不足 | ビルドログ（`No module named 'wheel'` 等） | `.venv` に不足ツールを追加（`uv pip install --python .venv/bin/python wheel`）して再ビルド。追加した版は §1.4.5 の lock に反映 |

**境界条件**
- **再ビルド**: editable は冪等。失敗後の再ビルドは同コマンドの再実行で可。クリーンが必要なら `submodules/{...}/build/` と生成 `.so` を削除して再 `uv pip install -e`。
- **ビルド済み再実行**: 既にビルド済みで再度 `uv pip install -e` してもエラーにはならない（変更が無ければ再コンパイルされないか、短時間で完了）。
- **ninja 不在**: torch は ninja 不在時 distutils にフォールバックしビルド可能（遅いだけ）。本案件は ninja を必須としない（§非機能）。

---

## 1.5 状態遷移

GUI・ステートフル処理は無い。作業は「取得 → ビルド(rasterizer) → ビルド(simple-knn) → 検証 → lock → コミット」の一方向手順。各ステップの受け入れ確認を満たしてから次へ進む。コミットは**手動テスト合格・ユーザー承認後**に行う。

---

## 1.6 ファイル・ディレクトリ設計

- 入力: `.gitmodules`（変更しない）、`submodules/`（FR-001で展開）、`.venv`（feat-001）。
- 出力: `submodules/*/`（ソース＋ビルド成果物 `.so`）、`requirements.lock.txt`（更新）。
- 本案件で**新規に4DGS本体ファイルは作らない・変更しない**。
- 一時生成物（`submodules/*/build/`、`*.egg-info`、`.so`）は git 管理対象外（サブモジュール内のため本リポジトリの追跡外）。
- 案件ドキュメント: `docs/issues/feat-002-cuda-ext-build/{README,requirements,design}.md`。不具合発生時は `investigation.md` を追記。

---

## 1.7 インターフェース定義

本案件はビルド作業であり、公開関数・クラスを新設しない。ビルドにより**利用可能になる**外部インターフェース（4DGS本体が依存する側）は以下:

- `diff_gaussian_rasterization.GaussianRasterizationSettings`（クラス）
- `diff_gaussian_rasterization.GaussianRasterizer`（クラス）
- `simple_knn._C.distCUDA2`（関数）

これらは本案件で**新規定義せず**、サブモジュールのソースをビルドして提供する。シグネチャはサブモジュール側の定義に従う（本案件で変更しない）。

---

## 1.8 ログ・デバッグ設計

- ビルドログは `uv pip install` の標準出力／標準エラーをそのまま確認する。`nvcc` の警告・エラーは原文で investigation.md に転記する。
- 切り分け順序: (1) FR-001 の取得完全性（glm 含む）→ (2) `--no-build-isolation` の有無 → (3) `CUDA_HOME`/`PATH` の上書き有無 → (4) torch/nvcc のメジャー版一致 → (5) 実行時 `LD_LIBRARY_PATH` 干渉。
- デバッグ用補助コマンド:
  ```bash
  .venv/bin/python -c "import torch; from torch.utils.cpp_extension import CUDA_HOME; print('torch sees CUDA_HOME =', CUDA_HOME)"
  /usr/local/cuda-11.6/bin/nvcc --version
  ```

---

## 設計判断の記録（ADR簡易版）

### ADR-1: editable インストール（`-e`）でビルドする
- **採用**: `uv pip install -e submodules/...`。
- **却下**: 非 editable（`uv pip install submodules/...`）でのインストール。
- **理由**: 公式READMEが `pip install -e` を採用しており手順を一致させる。editable は `.so` をソースツリーに置くため、再ビルド・デバッグ時の所在が明確。学習・レンダリングでの利用に差は無い。

### ADR-2: `--no-build-isolation` を必須付与する
- **採用**: `--no-build-isolation` を付け、`.venv` の torch・setuptools<81 を使う。
- **却下**: デフォルト（build isolation 有効）。
- **理由**: `setup.py` が import 時に `torch.utils.cpp_extension` を必要とするため、隔離環境（torch 不在）ではビルド不能。feat-001 の mmcv sdist ビルドと同じ理由（pkg_resources を持つ setuptools<81 も `.venv` 側を使える）。

### ADR-3: CUDA は `/usr/local/cuda-11.6` を**インライン環境変数**で上書き
- **採用**: `CUDA_HOME=/usr/local/cuda-11.6 PATH=/usr/local/cuda-11.6/bin:$PATH` をビルドコマンド前置（このコマンド限り）。
- **却下案A**: グローバルの 12.8 のままビルド → torch(11) vs nvcc(12) のメジャー版不一致でエラー／重大警告。
- **却下案B**: `~/.bashrc` やセッションで `CUDA_HOME` を永続変更 → このマシンの別環境（12.8依存）を壊すリスク。
- **理由**: torch cu116 とメジャー・マイナー一致する 11.6 を、グローバル設定を汚さずビルド時のみ適用するのが最小副作用（TECH_STACK 決定事項）。

### ADR-4: 全 `uv pip` に `--python .venv/bin/python` を明示
- **採用**: 明示。
- **却下**: `VIRTUAL_ENV` 自動検出に任せる。
- **理由**: 本マシンは別環境の bashrc が `VIRTUAL_ENV` を常時アクティブにしており、明示しないと取り違える（feat-001 で確立した運用）。

### ADR-5: `--recursive` でネストしたサブモジュール（glm）を取得
- **採用**: `git submodule update --init --recursive`。
- **却下**: `--init` のみ（非 recursive）。
- **理由**: depth-diff-gaussian-rasterization は GLM ヘッダ（`third_party/glm`）に依存し、CUDA コードが `glm/...` を include する。非 recursive だと glm 未取得で nvcc がヘッダ解決に失敗する。

### ADR-6: import 名を判定の正とする（実体構成差に頑健）
- **採用**: 合否は `import diff_gaussian_rasterization` / `import simple_knn._C` の成功で判定。
- **却下**: ビルド成果物の `.so` パスやディレクトリ構成を判定基準にする。
- **理由**: import 名は4DGS本体コードの実利用（`gaussian_renderer/__init__.py:14`, `scene/gaussian_model.py:22`）で確定しており、後続の学習・レンダリングが必要とするものと完全一致する。サブモジュールの内部構成は upstream 更新で変わりうるため、判定は import の成否に置く。

---

## 未検証事項（実装時に実地検証）

- depth-diff-gaussian-rasterization の実体ディレクトリ構成・`third_party/glm` の有無（FR-001 取得後に確認。標準3DGS構成を想定）。
- `--no-build-isolation` 下の build backend ツール: 本セッションで `.venv` に setuptools 80.10.2 / wheel 0.47.0 が存在することを確認済み。E7 は不足時の保険として残す。
- ビルド済み `.so` の import 時に、グローバル `LD_LIBRARY_PATH`（12.8）が干渉しないか（通常 torch 同梱lib優先で問題なしの想定。干渉時 E4）。
- editable 2点が `uv pip freeze` でどの形式（`-e file://` / `@ file://`）で出力されるか（FR-005、どちらでも可）。
