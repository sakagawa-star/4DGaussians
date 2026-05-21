# feat-001 機能設計書: uv環境構築・依存インストール

本書は `docs/DESIGN_STANDARD.md` に準拠する。本書を読んだ実装者が判断を要しないよう、
コマンド・順序・検証方法・失敗時の挙動をすべて明記する。

---

## 1.1 対応要求マッピング

| 要求ID | 設計セクション |
|--------|----------------|
| FR-001 仮想環境の作成 | §1.4.1 |
| FR-002 PyTorch（cu116）の導入 | §1.4.2 |
| FR-003 残り依存パッケージの導入 | §1.4.3 |
| FR-004 GPU認識の検証 | §1.4.4 |
| FR-005 lock生成とgit管理 | §1.4.5 |

---

## 1.2 システム構成

本案件はアプリコードの追加ではなく**環境構築**であり、成果物はコードではなく
(1) 仮想環境 `.venv`、(2) `requirements.lock.txt` の2つ。処理は対話的なシェルコマンド列で実施する。

```
/data/sakagawa/4DGaussians/
├── .venv/                  ← FR-001 で生成（git管理外）
├── requirements.txt        ← 入力（公式・既存、変更しない）
├── requirements.lock.txt   ← FR-005 で生成（git管理対象、新規）
└── docs/issues/feat-001-env-setup/  ← 本ドキュメント群
```

実行系の依存関係: FR-001 →（FR-002 → FR-003 は順序固定）→ FR-004 → FR-005。
FR-002 を FR-003 より先に行うのは、torch系を確定させてから残りを追加するため（§1.4.3 参照）。

---

## 1.3 技術スタック

| 項目 | 値 | 選定理由 |
|------|-----|----------|
| パッケージ管理 | uv 0.11.6 | conda不使用方針。`uv venv`+`uv pip` で公式conda手順を等価置換（TECH_STACK.md） |
| Python | 3.10（uv managed cpython-3.10.16 が導入済み） | uvは3.7非提供（下限3.8.20）。torch 1.13.1にcp310 wheelあり |
| torch / torchvision / torchaudio | 1.13.1+cu116 / 0.14.1+cu116 / 0.13.1+cu116 | 公式実績版 `pytorch=1.13.1+cu116` に一致。A100=sm_80 をCUDA 11.6が完全サポート |
| torch導入index | `https://download.pytorch.org/whl/cu116` | cu116 wheel の配布元（PyPIには `+cu116` 版が無い） |
| mmcv | 1.6.0（mmcv-full ではない純Python版） | `arguments/*.py` 設定を `mmcv.Config.fromfile` で読む。CUDA/C++ ops のビルドは不要だが、cp310 wheel が配布されず sdist インストール（`setup.py` 実行）が発生する（§1.4.6 E2 参照） |
| その他 | matplotlib / lpips / plyfile / pytorch_msssim / open3d / imageio[ffmpeg] | `requirements.txt` 準拠。版は自然解決し lock で固定 |

---

## 1.4 各機能の詳細設計

> **前提**: 全コマンドはカレントディレクトリ `/data/sakagawa/4DGaussians` で実行する。
> 本マシンはグローバルの `.bashrc` で別環境（CUDA 12.8 用）が常時アクティブであり、`VIRTUAL_ENV` 等の環境変数が
> 別環境を指していると uv がインストール先を取り違える恐れがある。これを構造的に防ぐため、**全ての `uv pip` コマンドに
> `--python .venv/bin/python` を付けて対象venvを明示**する。検証スクリプトの実行も `.venv/bin/python` を直接呼ぶ。

### 1.4.1 FR-001: 仮想環境の作成

**処理ロジック**
1. 事前確認: `.venv` が既に存在するか確認する。
   - 存在しない（現状の想定）→ 手順2へ。
   - 存在する → **無断で削除しない**。状態をユーザーに報告し判断を仰ぐ（§1.4.6 境界条件）。
2. 仮想環境を作成する:
   ```bash
   uv venv --python 3.10
   ```
   uv は管理下の cpython-3.10系（現状の導入版は 3.10.16）を用いて `/data/sakagawa/4DGaussians/.venv` を生成する。

**データフロー**
- 入力: なし（uv managed Python 3.10系を参照）
- 出力: `.venv/`（`.venv/bin/python` = Python 3.10系）

**受け入れ確認**
```bash
.venv/bin/python --version          # → Python 3.10.16（メジャー・マイナーが 3.10 であれば合格）
```

### 1.4.2 FR-002: PyTorch（cu116）の導入

**処理ロジック**
```bash
uv pip install --python .venv/bin/python \
  torch==1.13.1 torchvision==0.14.1 torchaudio==0.13.1 \
  --index-url https://download.pytorch.org/whl/cu116
```
- `--index-url` はデフォルトindexを cu116 index に**置き換える**。このステップで取得するのは torch系3点のみのため、
  cu116 index に存在する torch/torchvision/torchaudio の `+cu116` wheel が確実に選択される。

**データフロー**
- 入力: cu116 index
- 出力: venv に `torch 1.13.1+cu116`, `torchvision 0.14.1+cu116`, `torchaudio 0.13.1+cu116` と推移的依存（numpy 等）

**受け入れ確認**
```bash
.venv/bin/python -c "import torch, torchvision; print(torch.__version__); print(torchvision.__version__)"
# → 1.13.1+cu116
# → 0.14.1+cu116
```

### 1.4.3 FR-003: 残り依存パッケージの導入

**設計の核心（torch退行の防止）**:
`requirements.txt` には `torch==1.13.1`（CUDAサフィックス無し）が含まれる。これを
デフォルトindex（PyPI）から解決させると、PyPI に存在するのは CPU版 `1.13.1` であり、
FR-002 で入れた `1.13.1+cu116` を CPU版へ置換する恐れがある。これを構造的に防ぐため、
**torch系3行を除外した派生 requirements でインストールする**。

**処理ロジック**
1. torch系3行と argparse 行を除外した一時ファイルを生成する:
   ```bash
   grep -vE '^(torch|torchvision|torchaudio)==|^argparse$' requirements.txt > /tmp/req-no-torch-feat001.txt
   ```
   - `^(torch|torchvision|torchaudio)==` で torch系3行（いずれも `==` 付き）を、`^argparse$` で argparse 行（行全体が `argparse`）を除外する。
   - argparse を除外する理由は ADR-6（PyPI版が標準ライブラリをシャドーイングするのを防ぐ）。`requirements.txt` 自体は変更しない。一時ファイルは git管理外の `/tmp` に置く。
2. 残り依存を追加インストールする（追加的・非破壊）:
   ```bash
   uv pip install --python .venv/bin/python -r /tmp/req-no-torch-feat001.txt
   ```
3. **numpy整合の確認と是正**（§1.4.6 のエラーE3に対応する計画内処理）:
   ```bash
   .venv/bin/python -c "import numpy; print(numpy.__version__)"
   ```
   - 出力の先頭が `1.` → 何もしない。
   - 出力の先頭が `2.` → numpy を 1.x 系に固定して再導入する:
     ```bash
     uv pip install --python .venv/bin/python "numpy==1.23.5"
     ```
     その後 **§1.4.3 の受け入れ確認（全 import）と §1.4.4（GPU検証）の両方を再実行**する（numpy ダウングレードで open3d/imageio 等が壊れていないことを含めて確認するため）。
     もし numpy 1.23.5 への固定で open3d/imageio など他パッケージが import 不能になった場合（ABI非整合）は、計画外の依存衝突のため**中断・報告**する（§1.4.6 E3）。

**データフロー**
- 入力: `/tmp/req-no-torch-feat001.txt`（mmcv==1.6.0, matplotlib, lpips, plyfile, pytorch_msssim, open3d, imageio[ffmpeg]）。torch系3行と argparse は除外済み。
- 出力: 上記がvenvに導入される。torch系は `+cu116` のまま維持。

**受け入れ確認**
```bash
.venv/bin/python -c "import mmcv; from mmcv import Config; print('mmcv', mmcv.__version__)"
.venv/bin/python -c "import lpips, plyfile, pytorch_msssim, open3d, imageio, matplotlib, torchaudio; print('imports ok')"
.venv/bin/python -c "import torch; assert torch.__version__=='1.13.1+cu116', torch.__version__; print('torch ok', torch.__version__)"
# mmcv 1.6.0 / imports ok / torch ok 1.13.1+cu116
```

### 1.4.4 FR-004: GPU認識の検証

**処理ロジック**: venv の Python で検証スクリプトを実行する。
```bash
.venv/bin/python - <<'PY'
import torch
assert torch.version.cuda == '11.6', f"cuda={torch.version.cuda}"
assert torch.cuda.is_available() is True, "cuda not available"
assert torch.cuda.device_count() >= 1, f"devices={torch.cuda.device_count()}"
name = torch.cuda.get_device_name(0)
assert 'A100' in name, f"device0={name}"
print("OK:", torch.__version__, "| cuda", torch.version.cuda,
      "| devices", torch.cuda.device_count(), "| dev0", name)
PY
```
- 全 assert を通過し `OK: 1.13.1+cu116 | cuda 11.6 | devices N | dev0 ...A100...` が出力されれば合格。

**データフロー**
- 入力: なし（venv内 torch とドライバ）
- 出力: 検証結果の標準出力（合格/AssertionError）

### 1.4.5 FR-005: lock生成とgit管理

**処理ロジック**
1. lockを生成する:
   ```bash
   uv pip freeze --python .venv/bin/python > requirements.lock.txt
   ```
2. `.venv` がgit管理外であることを確認する（`.gitignore` に未登録なら追記。重複追記を防ぐ冪等な形にする）:
   ```bash
   grep -qxF '.venv/' .gitignore 2>/dev/null || echo '.venv/' >> .gitignore
   ```
3. lock を git に追加する（コミットはユーザー承認後／本案件の手動テスト合格後）:
   ```bash
   git add requirements.lock.txt .gitignore
   ```

**データフロー**
- 入力: venv の `uv pip freeze` 出力
- 出力: `requirements.lock.txt`（推移的依存含む全量＝版の正本）

**受け入れ確認**
```bash
test -s requirements.lock.txt && echo "lock non-empty"   # 空でない（必須）
grep -E '^mmcv==1\.6\.0' requirements.lock.txt            # mmcv 1.6.0 が存在
grep -i '^torch' requirements.lock.txt                    # torch行を表示し 1.13.1+cu116 を目視確認
```
- torch 行に `1.13.1+cu116` が含まれることを確認する。`uv pip freeze` の表記形式差（`torch==1.13.1+cu116` /
  `torch @ ...` 等）で機械的な grep が空振りすることがあるため、ここでは torch 行を**表示して目視確認**する。
  cu116保持の最終判定は FR-003/FR-004 の `torch.__version__`・`torch.version.cuda` の assert（§1.4.3/§1.4.4）を正とする
  （本 lock 検証は補助）。

### 1.4.6 エラーハンドリングと境界条件

各エラーは「検出方法 → 動作」の形で定義する。**計画外の対処が必要になった場合は中断してユーザーに報告**する
（CLAUDE.md フロー: 計画にない変更は中断・報告）。

| ID | 想定エラー | 検出方法 | 動作 |
|----|-----------|----------|------|
| E1 | `.venv` が既に存在 | FR-001手順1 | 無断削除しない。状態を報告しユーザー判断を仰ぐ |
| E2 | mmcv 1.6.0 の sdist インストール失敗（cp310 wheel が無いため sdist ビルドが走る。`setup.py`/setuptools 起因の失敗等） | FR-003 の `uv pip install` が非0終了（mmcv 行のエラー） | 一次対処（計画内、順に試す）: (a) mmcv 単体で再試行しビルドログを確認 `uv pip install --python .venv/bin/python mmcv==1.6.0`。(b) setuptools 起因（`pkgutil.ImpImporter` 等）なら venv にビルドツールを導入後 build isolation を無効化して再試行 `uv pip install --python .venv/bin/python -U setuptools wheel` → `uv pip install --python .venv/bin/python --no-build-isolation mmcv==1.6.0`。(a)(b) で解決しない場合は**中断・報告**し、mmcv 近接版への変更可否を investigation.md に記録して承認を得る（計画外変更） |
| E3 | numpy が 2.x に解決され torch と非整合 | FR-003手順3 の `numpy.__version__` 出力先頭が `2.`（`import torch` 時ではなく `.numpy()`/`from_numpy()` 実行時に顕在化するため、版数チェックで検出する） | 計画内処理: `numpy==1.23.5` を追加導入し §1.4.3受け入れ確認と§1.4.4 を再実行（§1.4.3手順3）。固定後に open3d/imageio 等が import 不能になった場合は中断・報告 |
| E4 | torch系が CPU版に退行（`+cu116` が消える） | FR-003 受け入れ確認の assert 失敗 | 原因（PyPI再解決）を確認し、cu116 から torch系を再導入: `uv pip install --python .venv/bin/python --reinstall torch==1.13.1 torchvision==0.14.1 torchaudio==0.13.1 --index-url https://download.pytorch.org/whl/cu116`。再発時は中断・報告 |
| E5 | `torch.cuda.is_available()` が False | FR-004 の assert 失敗 | 中断・報告。ドライバ／GPU可視性（`CUDA_VISIBLE_DEVICES`）／wheel種別を調査 |
| E6 | wheelダウンロード失敗（ネットワーク） | `uv pip install` が非0終了 | 初回実行を含め最大3回まで実行し、3回目も非0終了なら中断・報告 |
| E7 | GPUが0台に見える | `torch.cuda.device_count()==0` | 中断・報告（E5と同様の調査） |

**境界条件**
- GPUビジー時: 他プロセスがGPUを占有していても、本案件の検証（`is_available`/`get_device_name`）はメモリ確保を伴わないため影響しない。
- ディスク不足: wheelキャッシュ展開に失敗した場合 E6 と同様（uvがエラー終了）。

---

## 1.5 状態遷移

**該当なし。** 本案件はGUI・ステートフル処理を含まず、一方向のコマンド列（FR-001→…→FR-005）で完結する。
途中失敗時は §1.4.6 の各動作に従う。

---

## 1.6 ファイル・ディレクトリ設計

| パス | 区分 | git | 説明 |
|------|------|-----|------|
| `/data/sakagawa/4DGaussians/.venv/` | 生成 | 管理外（`.gitignore`） | uv 仮想環境。再生成可能なため追跡しない |
| `/data/sakagawa/4DGaussians/requirements.lock.txt` | 生成 | 管理対象（新規） | `uv pip freeze` 全量。版の正本 |
| `/data/sakagawa/4DGaussians/requirements.txt` | 既存 | 管理対象（変更しない） | 公式の緩いスペック |
| `/tmp/req-no-torch-feat001.txt` | 一時 | 管理外 | torch系3行・argparse 行を除外した派生。FR-003でのみ使用。FR-005完了後に `rm -f /tmp/req-no-torch-feat001.txt` で削除する |

---

## 1.7 インターフェース定義

公開関数・クラスの追加は無い（環境構築のため）。本案件のインターフェースは
「§1.4 のコマンド列」と「成果物 `.venv` / `requirements.lock.txt`」である。
全ての `uv pip` 操作は §1.4 前提のとおり `--python .venv/bin/python` で対象venvを明示し、検証は `.venv/bin/python` を直接呼ぶ。
後続案件（feat-002以降）も本 `.venv` を共用する。

---

## 1.8 ログ・デバッグ設計

- 各ステップは標準出力と終了コードで成否を判断する（専用ロガーは導入しない＝対話的コマンド実行のため）。
- 検証スクリプト（FR-003/FR-004）は合格時に `OK:` / `imports ok` 等の明示メッセージを出力し、
  失敗時は `AssertionError`（メッセージに実測値を含む）で停止する。
- 構築後、`uv pip list` と `requirements.lock.txt` を突き合わせ可能にしておく（デバッグ時の参照）。

---

## 設計判断の記録（ADR簡易版）

- **ADR-1: torch系を分離して2段階導入（FR-002→FR-003）**
  - 採用: torch系を cu116 index で先行導入し、残りは torch系除外の派生 requirements で追加導入。
  - 却下: `uv pip install -r requirements.txt --extra-index-url .../cu116` の一括導入。
  - 理由: 一括だと `torch==1.13.1`（無印）がPyPI側のCPU版に解決され得る。分離すれば cu116 版を確実に保持できる。

- **ADR-2: requirements.txt から grep 除外した派生ファイルを使う**
  - 採用: `grep -vE '^(torch|torchvision|torchaudio)==|^argparse$'` で torch系3行と argparse 行を除外。
  - 却下: 残り依存を手動列挙して `uv pip install pkg1 pkg2 ...`。
  - 理由: requirements.txt を正本に保ち、将来 requirements.txt が更新されても同期ミスが起きない。

- **ADR-6: argparse は PyPI から導入せず標準ライブラリを使う**
  - 採用: `requirements.txt` の `argparse` 行を grep で除外し、Python 3.10 標準ライブラリの argparse を使う。
  - 却下: `requirements.txt` 通り PyPI の argparse を導入。
  - 理由: PyPI の argparse は最終リリースが2010年（Python 2.x 時代のバックポート）。これを site-packages に入れると
    標準ライブラリの argparse をシャドーイングし、4DGS本体の引数解析に潜在不具合を生む恐れがある。除外しても
    標準ライブラリの argparse が使えるため機能影響はない。

- **ADR-3: feat-001 では `CUDA_HOME` を上書きしない**
  - 採用: グローバルの `CUDA_HOME`（12.8）のまま導入・検証する。
  - 理由: torch prebuilt wheel は自前のCUDAランタイムを同梱し、インストール・実行ともに `CUDA_HOME` に依存しない。
    `CUDA_HOME=/usr/local/cuda-11.6` の上書きはソースビルドを行う feat-002 でのみ必要。

- **ADR-4: numpy は自然解決させ、2.x のときだけ 1.23.5 に固定**
  - 採用: FR-003 後に numpy 版を確認し、`2.x` の場合のみ `numpy==1.23.5` を追加導入。
  - 却下: 最初から numpy を固定。
  - 理由: open3d/imageio 等の numpy 下限要求と衝突しないかが事前に不明。まず自然解決を観察し、torch 1.13.1 と
    非整合な 2.x に上がった場合のみ是正する方が、不要な制約を持ち込まない。1.23.5 は torch 1.13系の動作実績域。

- **ADR-5: サブモジュール初期化は feat-002 に委譲**
  - 採用: 本案件では `git submodule update --init --recursive` を行わない。
  - 理由: `BACKLOG.md` 上 feat-002（CUDA拡張ビルド）の先頭工程に位置づけられており、feat-001 の依存インストール
    （PyPI/PyTorch index 由来）には不要。スコープを Phase 0 に限定する。

---

## 未検証事項と失敗時の扱い（実装時に実地検証）

| 事項 | 検証ステップ | 失敗時 |
|------|-------------|--------|
| mmcv 1.6.0 の sdist インストールが Python 3.10 で成功するか（cp310 wheel 無し→sdist ビルド） | FR-003 のインストールと import | E2: 一次対処（ビルドツール更新＋build isolation無効化で再試行）→ 不可なら中断・報告 |
| open3d/imageio 最新版と torch 1.13.1 の numpy 整合 | FR-003手順3 の numpy 版確認 | E3: numpy==1.23.5 固定（計画内）。固定で他パッケージが壊れたら中断・報告 |
| torch系が cu116 のまま維持されるか | FR-003 受け入れ確認の assert | E4: cu116 から再導入（完全コマンドはE4参照）、再発時は中断 |

実装完了後、本表の検証結果（実際の解決版・遭遇したエラーと対処）を本ファイル末尾または investigation.md に追記する。

---

## 実装結果（2026-05-21）

FR-001〜FR-005 を実装し、判定基準を全て満たした。遭遇したエラーと対処（E2/E3 の実地結果）を記録する。

### 判定結果
- FR-001: `.venv/bin/python` = **Python 3.10.16**（メジャー・マイナー 3.10 ✓）
- FR-002: `torch==1.13.1+cu116` / `torchvision==0.14.1+cu116` / `torchaudio==0.13.1+cu116` 導入 ✓
- FR-003: 全パッケージ import 成功、`from mmcv import Config` ✓、torch退行なし（`+cu116` 維持）
- FR-004: `cuda 11.6` / `is_available()=True` / `device_count()=7` / `get_device_name(0)='NVIDIA A100-SXM4-40GB'` ✓
- FR-005: `requirements.lock.txt`（88行）生成、`.venv` は ignore 対象 ✓

### E2（mmcv 1.6.0 sdist ビルド失敗）の実地対処
- 症状: `uv pip install mmcv==1.6.0` が `ModuleNotFoundError: No module named 'pkg_resources'` で失敗。
- 原因: mmcv 1.6.0 の `setup.py` は `pkg_resources` に依存するが、(1) uv の build isolation 環境に無く、(2) venv に入れた **setuptools 最新版（82.0.1）は pkg_resources を削除済み**（setuptools 81 で削除）。
- 対処: `pkg_resources` を含む **`setuptools<81`（実際は 80.10.2）** を venv に導入し、`uv pip install --python .venv/bin/python --no-build-isolation mmcv==1.6.0` でビルド成功。
- 設計との差分: E2(b) は「ビルドツール導入＋build isolation 無効化」と記載していたが、setuptools の**バージョン上限（<81）が必須**である点が新たに判明。これは pkg_resources 提供という同一目的の具体化であり、計画内対処の範囲とした。

### E3（numpy 2.x と torch の ABI 非整合）の実地対処
- 症状: torch 依存解決で numpy 2.2.6 が入り、`torch.from_numpy(np.zeros(3))` が `RuntimeError: Numpy is not available`（torch 1.13.1 は numpy 1.x ABI でビルド。torchvision import 時に `Failed to initialize NumPy: _ARRAY_API not found` 警告）。
- 対処: ADR-4 通り `numpy==1.23.5` に固定 → torch⇔numpy 連携が復活。
- **設計時の懸念（numpy 1.x で open3d 0.19 / opencv 4.13 / scipy 1.15 / sklearn 1.7 / pandas 2.3 が ABI 破損する）は発生しなかった**。これらは import・機能テスト（open3d Vector3dVector、cv2.cvtColor、scipy cKDTree）が全て numpy 1.23.5 で正常動作。よって numpy 単純固定で十分だった（依存群のダウングレードは不要）。

### 確定した主要解決版（正本は requirements.lock.txt）
`torch 1.13.1+cu116` / `torchvision 0.14.1+cu116` / `torchaudio 0.13.1+cu116` / `numpy 1.23.5` / `mmcv 1.6.0` / `open3d 0.19.0` / `opencv-python 4.13.0.92` / `scipy 1.15.3` / `setuptools 80.10.2`。

### 残存する警告（機能影響なし）
- `pkg_resources is deprecated`（setuptools<81 由来）。torch の `cpp_extension` が `pkg_resources` を使うため feat-002 のビルドでも出るが、80.10.2 では使用可能で問題なし。
