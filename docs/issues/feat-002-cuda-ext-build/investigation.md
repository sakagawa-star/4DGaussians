# feat-002 調査・対処記録（investigation）

実装（ステップ6）中に発生した計画外事態を、design.md §1.4.6「計画外の対処が必要になった場合は中断して報告し investigation.md に記録する」に従って記録する。イテレーション番号を付けて履歴を残す（上書きしない）。

---

## Iteration 1（2026-05-21）: simple-knn が editable ビルド成功後に import 不能

### 現象

FR-003 で `uv pip install --python .venv/bin/python --no-build-isolation -e submodules/simple-knn` は**ビルド成功**（`simple-knn==0.0.0` installed、`_C.cpython-310-x86_64-linux-gnu.so` を `submodules/simple-knn/simple_knn/` に生成）したが、受け入れ確認の `import simple_knn._C` が以下で失敗した。

```
ModuleNotFoundError: No module named 'simple_knn'
```

FR-002（rasterizer）は同方式で import まで成功しており、simple-knn 固有の問題。

### 根本原因（確定）

**uv の editable インストール（PEP 660 の editable finder 方式）は、対象トップレベルパッケージに `__init__.py` が無いと import 解決できない。**

- uv が生成した `.venv/lib/python3.10/site-packages/__editable___simple_knn_0_0_0_finder.py` の `_find_spec` は、`MAPPING={'simple_knn': '.../submodules/simple-knn/simple_knn'}` のディレクトリに対し **`__init__.py`（およびモジュールファイル `simple_knn.<suffix>`）の存在を要求**する。いずれも無ければ `None` を返し解決失敗。`NAMESPACES` も空（`{}`）。
- simple-knn の構成: `simple_knn/` には `.gitkeep` と（正しくビルドされた）`_C.cpython-310-x86_64-linux-gnu.so` のみで、**`__init__.py` が無い**。`setup.py` も `packages` 未指定で `ext_modules=[CUDAExtension(name="simple_knn._C", ...)]` のみ。
- 一方 rasterizer は `diff_gaussian_rasterization/__init__.py` を持つため、同じ finder の `_find_spec` が init を拾い解決でき、成功した。これが両者の明暗を分けた。

### なぜ公式手順 `pip install -e` では顕在化しないか

従来の setuptools develop（`pip install -e`）は**ソースディレクトリを `sys.path` に直接追加**するため、`simple_knn/` が暗黙的名前空間パッケージ（PEP 420）として解決され、`simple_knn._C` が import 可能だった。uv の PEP 660 finder 方式はこの暗黙解決を行わず `__init__.py` を必須とする。**design ADR-1（editable採用）で見落とした uv 特有の挙動**であり、design §1.4.6 の E1〜E7 にも該当ケースが無かった（=計画外）。

### 検出方法

- ビルドログ: `Installed 1 package ... simple-knn==0.0.0`（ビルド自体は成功）。
- `.venv/bin/python -c "import simple_knn._C"` → `ModuleNotFoundError: No module named 'simple_knn'`。
- `find submodules/simple-knn -name "*.so"` → `simple_knn/_C.cpython-310-x86_64-linux-gnu.so`（成果物は存在）。
- editable finder のソース `__editable___simple_knn_0_0_0_finder.py` の `_find_spec` を読み、`__init__.py` 必須を確認。
- `submodules/simple-knn/simple_knn/__init__.py` の不在を確認（rasterizer 側は存在）。

### 対処（採用案 A、ユーザー承認 2026-05-21）

`submodules/simple-knn/simple_knn/__init__.py`（**空ファイル**）を追加する。

- editable finder の `_find_spec` が `__init__.py` を拾い、`simple_knn` をパッケージとして解決できるようになる。その `__path__` 配下の `_C.cpython-310-x86_64-linux-gnu.so` も到達可能になる。
- **再ビルド不要**（finder は import 時にファイル存在を動的チェックするため、ファイルを置くだけで次回 import から解決される見込み）。
- 機能影響ゼロ（空の `__init__.py`）。

#### 却下案

- **案 B（非editable で `uv pip install`）**: サブモジュール不変更だが、ADR-1（editable採用・公式手順一致）から外れ、`.so` が site-packages にコピーされソース変更が反映されなくなる。
- **案 C（`setup.py` に `packages=["simple_knn"]` 追加）**: editable で namespace 登録されるが、サブモジュール変更に加え再ビルド（数分）が必要で案 A より重い。

### 影響範囲

- サブモジュール `submodules/simple-knn`（外部リポジトリ、INRIA）への空ファイル1個の追加のみ。4DGS本体コードは不変更。
- サブモジュール内のため本リポジトリの git 追跡対象外（commit されない）。環境を作り直す場合は本記録に従い再追加する。

### design への反映（知見）

- ADR-1 補足: **uv の editable インストールは対象パッケージに `__init__.py` を要求する**。`__init__.py` を持たない C拡張専用パッケージ（simple-knn 等）は、空 `__init__.py` の追加が必要。

### 追加対処（案 X、ユーザー承認 2026-05-21）: libc10.so（import 順序）

案 A（空 `__init__.py` 追加）で `ModuleNotFoundError: No module named 'simple_knn'` は解消したが、続けて第2の問題が連鎖発覚した。

- **現象**: `import simple_knn._C` 単独で `ImportError: libc10.so: cannot open shared object file: No such file or directory`。
- **根本原因**: `simple_knn/__init__.py` が空のため、torch の共有ライブラリ（`libc10.so` 等）がプロセスにロードされる前に `_C.so`（libc10.so にリンク）を読もうとして失敗。`import torch` を先に実行すれば成功する。rasterizer が単独 import で通ったのは `diff_gaussian_rasterization/__init__.py` が `import torch`（L13-14）を含むため。
- **検証**:
  - `import simple_knn._C` 単独 → `ImportError: libc10.so`
  - `import torch; import simple_knn._C; from simple_knn._C import distCUDA2` → 成功（`distCUDA2` 取得）
- **採用案 X**: `submodules/simple-knn/simple_knn/__init__.py` に `import torch` を記載（コメント付き）。rasterizer と挙動を揃え、単独 import でも libc10.so をロード済みにする。
- **却下案 Y**: `__init__.py` を空のまま、判定を torch 先行スクリプト（FR-004）に限定。4DGS 本体は常に torch 先行のため実運用は動作するが、単独 import は通らない。堅牢性のため案 X を採用。
- **補足**: 4DGS 本体（`scene/gaussian_model.py` 等）は必ず torch を先に import するため、案 Y でも実運用は動作した。案 X はその保険。

### 検証結果（最終、2026-05-21）

案 A（空 `__init__.py`）＋案 X（`import torch` 記載）適用後、FR-003〜005 の受け入れ基準を全て満たした。

- FR-003 単独 import: `import simple_knn._C` → ok、`from simple_knn._C import distCUDA2` → ok
- FR-004 統合検証: `OK: CUDA ext import | torch 1.13.1+cu116 | cuda 11.6 | device NVIDIA A100-SXM4-40GB`
- FR-005 lock 再生成: CUDA 拡張2点が `-e file:///...` 形式で記録、torch/mmcv/numpy の退行なし、`git diff` の追加行は当該2点のみ。

→ feat-002 の判定基準（`import diff_gaussian_rasterization` / `import simple_knn._C` 成功）を達成。simple-knn の最終構成は `simple_knn/__init__.py`（`import torch` 1行＋コメント）。**サブモジュール内のため git 追跡対象外**であり、環境を作り直す場合は本記録に従い再作成する。
