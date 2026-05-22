# feat-004 調査・修正計画: 学習起動時の PIL TypeError

> 本ファイルは feat-004（D-NeRF学習動作確認）の実装中（ステップ6 動作確認）に発覚した不具合の修正計画。`docs/BUGFIX_STANDARD.md` に準拠。**上書きせずイテレーション番号を付けて追記する**。

---

## イテレーション1 (2026-05-22)

### 1.1 不具合の特定

- **現在の動作**: feat-004 の学習コマンドを起動すると、データ読み込み（`readCamerasFromTransforms`）の段階で `TypeError` が発生し、学習ループに入る前にプロセスが終了コード1でクラッシュする。`output/dnerf/bouncingballs/` には `cfg_args` のみ生成され、`point_cloud/` は作られない。
- **再現手順**:
  ```bash
  CUDA_VISIBLE_DEVICES=0 /data/sakagawa/4DGaussians/.venv/bin/python train.py \
    -s data/dnerf/bouncingballs --port 6017 \
    --expname "dnerf/bouncingballs" --configs arguments/dnerf/bouncingballs.py
  ```
  起動直後（`Found transforms_train.json file, assuming Blender data set!` / `Reading Training Transforms` の直後）に発生。
- **期待する動作**: 各 frame の PNG を読み込み、白背景合成した RGB 画像（0〜255）を生成して学習データに変換する。データ読み込みが完了し、coarse → fine の学習ループに入る。
- **エラーメッセージ（完全なトレースバック）**:
  ```
  Traceback (most recent call last):
    File ".../PIL/Image.py", line 3427, in fromarray
      typemode, rawmode, color_modes = _fromarray_typemap[typekey]
  KeyError: ((1, 1, 3), '|i1')

  The above exception was the direct cause of the following exception:

  Traceback (most recent call last):
    File "/data/sakagawa/4DGaussians/train.py", line 429, in <module>
      training(lp.extract(args), ...)
    File "/data/sakagawa/4DGaussians/train.py", line 303, in training
      scene = Scene(dataset, gaussians, load_coarse=None)
    File "/data/sakagawa/4DGaussians/scene/__init__.py", line 50, in __init__
      scene_info = sceneLoadTypeCallbacks["Blender"](args.source_path, args.white_background, args.eval, args.extension)
    File "/data/sakagawa/4DGaussians/scene/dataset_readers.py", line 316, in readNerfSyntheticInfo
      train_cam_infos = readCamerasFromTransforms(path, "transforms_train.json", white_background, extension, timestamp_mapper)
    File "/data/sakagawa/4DGaussians/scene/dataset_readers.py", line 287, in readCamerasFromTransforms
      image = Image.fromarray(np.array(arr*255.0, dtype=np.byte), "RGB")
    File ".../PIL/Image.py", line 3431, in fromarray
      raise TypeError(msg) from e
  TypeError: Cannot handle this data type: (1, 1, 3), |i1
  ```

### 1.2 原因分析

- **原因箇所**: `scene/dataset_readers.py:287`、関数 `readCamerasFromTransforms`
  ```python
  image = Image.fromarray(np.array(arr*255.0, dtype=np.byte), "RGB")
  ```
- **原因の説明**:
  - `np.byte` は `numpy.int8`（符号付き8bit、dtype 文字列 `'|i1'`）である（実機確認: `np.dtype(np.byte) == int8`）。
  - PIL の `Image.fromarray(obj, mode="RGB")` は内部で `_fromarray_typemap` に `((1, 1, 3), typestr)` をキーとして引く。**Pillow 12.2.0**（実機確認）の typemap には `'|u1'`（uint8）のエントリはあるが `'|i1'`（int8）のエントリが無いため `KeyError` → `TypeError: Cannot handle this data type: (1, 1, 3), |i1` となる。
  - 旧 Pillow（公式が前提とした Python 3.7 系の古い Pillow）では int8 配列を "RGB"（uint8 相当）として受理していたため、オリジナルコードでも動作していた。新しい Pillow で型チェックが厳格化され顕在化した、**ライブラリバージョン差に起因する非互換**。
- **根本原因 or 表面的原因**: **根本原因**。RGB 画素値は 0〜255 の範囲であり、本来 `uint8`（`np.uint8`）で表現すべきところを `np.byte`（int8、-128〜127）で確保しているオリジナルコードの型指定が誤り。int8 では 128〜255 の値が負にオーバーフローするため、仮に旧 Pillow で例外が出なくても値としても不正。`np.uint8` が意味的に正しい型。
- **環境情報（実機確認済み）**:
  - numpy 1.23.5 / Pillow 12.2.0（`requirements.lock.txt` と一致）
  - 同一の問題パターン（`np.byte` を `Image.fromarray(...,"RGB")` に渡す）は **`scene/dataset_readers.py:287` の1箇所のみ**（`grep -rn "np.byte"` で確認）

### 1.3 修正内容

- **変更対象ファイル**: `scene/dataset_readers.py`（1ファイル・1行）
- **修正コード**:
  - 修正前（`:287`）:
    ```python
    image = Image.fromarray(np.array(arr*255.0, dtype=np.byte), "RGB")
    ```
  - 修正後:
    ```python
    image = Image.fromarray(np.array(arr*255.0, dtype=np.uint8), "RGB")
    ```
  - 変更点: `dtype=np.byte`（int8）→ `dtype=np.uint8`（符号なし8bit）。`arr*255.0` は 0.0〜255.0 の float であり、uint8 へのキャストで 0〜255 の整数になる。PIL "RGB" モードが期待する `'|u1'` 型と一致する。
- **変更しないファイル**:
  - `utils/scene_utils.py:33` — 既に `.astype('uint8')` を使用しており正しい。変更不要。
  - `scene/neural_3D_dataset_NDC.py:128,333` — DyNeRF（Neural 3D Video）用データローダで、入力はデコード済み動画フレーム（既に uint8）であり今回のコードパスを通らない。本案件（D-NeRF）と無関係のため変更しない。
  - ライブラリ（Pillow）はダウングレードしない（§代替案で却下理由を記載）。

### 1.4 影響範囲

- **他の機能への影響**:
  - `readCamerasFromTransforms` は **Blender（D-NeRF）データセットの train/test 読み込み全般**で使われる（`readNerfSyntheticInfo` 経由）。本修正により学習（train.py）・レンダリング（render.py、feat-005）・評価（feat-006）すべての D-NeRF 経路が同じく恩恵を受ける。
  - 出力画像の画素値が int8 オーバーフロー（128〜255 が負値化）から正しい 0〜255 に修正されるため、**学習品質はむしろ正常化**する（劣化方向の影響はない）。
- **リグレッションリスク**:
  - 低。`np.uint8` は PIL "RGB" の標準型であり、他データセット（colmap/dynerf/nerfies 等）の読み込み経路は本行を通らないため影響しない。
  - オリジナルコードの挙動を変える点は「int8→uint8」のみで、これは公式が旧 Pillow で意図していた挙動（uint8 相当として解釈）に一致させる修正であり、意味的後退はない。

### 1.5 確認方法

- **テスト項目**:
  1. 修正後、feat-004 の学習コマンドが `TypeError` を出さずデータ読み込みを完了し、`data loading done` 以降の学習ループに入る
  2. coarse(3000)+fine(20000) が完走し `Training complete.` が出力される（feat-004 FR-002）
  3. `output/dnerf/bouncingballs/point_cloud/iteration_20000/` に成果物が生成される（FR-003）
  4. fine の ITER 14000 の test PSNR ≥ 20（FR-004）
- **テストコマンド**（修正後・バックグラウンド実行＋ログ保存）:
  ```bash
  CUDA_VISIBLE_DEVICES=0 /data/sakagawa/4DGaussians/.venv/bin/python train.py \
    -s data/dnerf/bouncingballs --port 6017 \
    --expname "dnerf/bouncingballs" --configs arguments/dnerf/bouncingballs.py \
    > /data/sakagawa/tmp/feat004-dnerf-train/train.log 2>&1
  # 期待: ログに TypeError が出ず、末尾に "Training complete."
  grep -c "TypeError" /data/sakagawa/tmp/feat004-dnerf-train/train.log   # → 0 を期待
  grep "Training complete." /data/sakagawa/tmp/feat004-dnerf-train/train.log
  ```

### 代替案と却下理由（ADR）

- **代替案A: Pillow をダウングレードする**（int8 を受理する旧バージョンへ）
  - 却下。(1) どのバージョンまで許容するかの追加調査が必要で、他依存（torch 同梱要件・mmcv 等）とのコンフリクトリスクがある。(2) `requirements.lock.txt` の正本を広範に変える。(3) 根本原因（int8 という型の誤り）を解消せず、128〜255 の負値オーバーフローという値の不正は残る。(4) feat-005/006 でも同じ非互換に再び当たる。
- **採用案: `np.byte`→`np.uint8` のコード1行修正**
  - 採用。最小・局所・意味的に正しい（RGB 0〜255 は uint8）。CLAUDE.md「オリジナルコードは可能な限り変更しない／変更時は理由を案件ドキュメントに記録」に従い本ファイルへ理由を記録。BUGFIX_STANDARD §2.2「報告された不具合以外のコード変更を行わない」に従い、当該1行以外は変更しない。
