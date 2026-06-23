レビュー結果です。致命度があるものだけに絞ります。

**高**

1. `colmap.sh` 内部の bare `python` 依存が設計から漏れています。  
   [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:81) は「全コマンドは `.venv/bin/python` を明示」としていますが、実際の [colmap.sh](/data/sakagawa/4DGaussians/colmap.sh:8) と [colmap.sh](/data/sakagawa/4DGaussians/colmap.sh:16) は `python scripts/llff2colmap.py` / `python database.py` を呼びます。現在のこの環境では `python` は `command not found` でした。つまり、文書通り `CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh ...` で実行すると FR-003 が即死します。

   修正案: `colmap.sh:5` だけを改変対象に保つなら、FR-003 の実行コマンドを次に変えるのが最小です。
   ```bash
   PATH="$PWD/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID \
   bash colmap.sh data/dynerf/cut_roasted_beef llff N
   ```
   受け入れ基準にも「`PATH="$PWD/.venv/bin:$PATH" command -v python` が `.venv/bin/python` を指す」を追加してください。より堅牢にするなら `colmap.sh` 側で `.venv/bin/python` を使うべきですが、その場合「本体改変は5行目のみ」という制約を緩和する必要があります。

**低（指定論点なので報告）**

2. DyNeRF の MPL 依存説明は実コードと不整合です。  
   文書は [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:27) などで `readdynerfInfo -> format_infos -> plot_camera_orientations` としていますが、実際に `plot_camera_orientations` を呼ぶのは HyperNeRF 側の [readHyperDataInfos](/data/sakagawa/4DGaussians/scene/dataset_readers.py:390) です。DyNeRF 側の [readdynerfInfo](/data/sakagawa/4DGaussians/scene/dataset_readers.py:441) は `format_infos` と `fetchPly` を使うだけで、`output.png` 生成も matplotlib import も通りません。

   修正案: train/render に `MPLBACKEND=Agg` を付ける運用自体は無害なので残してもよいですが、「DyNeRF 読み込み時に必須」「output.png が生成される」という記述と受け入れ基準は削除または「保守的に付与」に弱めるべきです。

**問題なし**

- `colmap.sh:5` を `export CUDA_VISIBLE_DEVICES=${3:-0}` にする案は妥当です。`CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh ...` の環境変数は bash に渡り、COLMAP 子プロセスにも継承されます。
- `eval_index=0` の理解は実コードと一致しています。`cam00` 相当のソート先頭だけが test、残りが train です。
- 保存 iteration の理解も一致しています。`save_iterations.append(args.iterations)` は config merge 前なので 30000 も残りますが、DyNeRF config 後の fine は 14000 までしか回らず、実際に保存されるのは `iteration_14000` です。coarse 3000 は保存対象に届きません。
- 前処理 I/O は、上記 bare `python` 問題を除けば、`preprocess -> camXX/images/0000.png -> llff2colmap -> fused.ply -> points3D_downsample2.ply` の依存は押さえられています。

---

## Claude Code の対応方針（2026-06-22、初回レビュー codex-01 への対応）

両指摘とも実コード・実環境で検証し、いずれも妥当と確認。requirements.md / design.md を修正した。

### 高1: colmap.sh 内部の bare `python` 依存 → **対応済み（修正）**
- 検証: `which python` は非0（bare `python` 不在、`/usr/bin/python3` のみ）。`PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" command -v python` → `.venv/bin/python` に解決されることを確認。
- 修正: FR-003 の実行コマンドに `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH"` を前置（colmap.sh の改変は5行目=GPU引数化のみを維持）。用語定義・受け入れ基準（実行前の `command -v python` 確認）・エラーハンドリング・§1.7 表・ADR-3 に反映。`bash colmap.sh` の cwd はリポジトリルートとする旨も明記（`scripts/llff2colmap.py`・`database.py` の相対参照のため）。

### 低2: DyNeRF の MPL 依存は実コードと不整合 → **対応済み（事実誤認を訂正）**
- 検証: `format_infos`（`dataset_readers.py:353-370`）は `return cameras` で終了し `plot_camera_orientations` を呼ばない。`plot_camera_orientations`（`:390` 呼び出し）は **HyperNeRF 専用 `readHyperDataInfos`（`:373-400`）の内部**。`readdynerfInfo`（`:441`）は plot を呼ばない＝DyNeRF 経路は matplotlib 非経由。
- 修正: MPL 環境変数（`MPLBACKEND`/`MPLCONFIGDIR`/`TMPDIR`）と scratch の mplconfig/tmp・output.png 副作用を全 FR から削除。モジュール表・依存フロー図・非機能要求・ADR-6 を「DyNeRF は plot 非呼び出し＝MPL 不要」に訂正。エラーハンドリングに保険記述（万一の matplotlib エラー時のみ付与）を残置。

### 指摘なし項目（Codex が「問題なし」と確認）
- `colmap.sh:5` の `${3:-0}` 引数化と `CUDA_DEVICE_ORDER=PCI_BUS_ID` の子プロセス継承は妥当。
- `eval_index=0`（cam00=test、残り=train）・保存 iteration（`iteration_14000` のみ）・前処理 I/O 依存の理解は実コードと一致。

→ codex-02 で再レビュー（resume で同一セッション継続）。