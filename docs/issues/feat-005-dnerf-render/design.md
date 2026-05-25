# feat-005 機能設計書: D-NeRFレンダリング動作確認

## 1.1 対応要求マッピング

| 要求 | 内容 | 設計箇所 |
|------|------|----------|
| FR-001 | レンダリングコマンドの実行とモデルロード | §1.4.1 / §1.4.2 |
| FR-002 | test セットのレンダリング（renders + gt） | §1.4.3 |
| FR-003 | video セットのレンダリング | §1.4.4 |
| FR-004 | 動画ファイル（mp4）の生成 | §1.4.5 |
| FR-005 | 実行ログの保存 | §1.4.1 / §1.8 |
| （全 FR 横断） | エラーハンドリングと境界条件 | §1.4.6 |

本案件はコードを変更しない（既存 `render.py` の実行とその出力検証のみ）。設計の中心は「公式 `render.py` が D-NeRF 学習済みモデルに対しどのような経路で動き、何をどこへ出力するか」を解析し、判定基準に落とすことにある。

## 1.2 システム構成

### 実行フロー（`render.py`）

```
python render.py --model_path "output/dnerf/bouncingballs/" --skip_train --configs arguments/dnerf/bouncingballs.py
  │
  ├─ get_combined_args (arguments/__init__.py:152)
  │     └─ {model_path}/cfg_args を読み、コマンドライン引数とマージ（コマンドライン優先）
  │        → source_path=data/dnerf/bouncingballs, white_background, eval 等が cfg_args から復元される
  ├─ --configs arguments/dnerf/bouncingballs.py を mmcv で読み hyperparam にマージ（render.py:107-111）
  ├─ safe_state (RNG 初期化)
  └─ render_sets (render.py:78)
        ├─ GaussianModel(sh_degree, hyperparam)
        ├─ Scene(dataset, gaussians, load_iteration=-1, shuffle=False)   ← render.py:81、args.iteration 既定 -1
        │     ├─ load_iteration=-1 → searchForMaxIteration(point_cloud/) → 20000  (§1.4.2)
        │     ├─ Blender 判定（transforms_train.json 存在）→ readNerfSyntheticInfo
        │     │     ├─ readCamerasFromTransforms(train/test)   ← test 20 カメラ。dataset_readers.py:287 を通る（feat-004 修正済み）
        │     │     └─ generateCamerasFromTransforms(video)    ← 円軌道 160 フレーム。Image.fromarray を通らない（§1.4.4）
        │     └─ gaussians.load_ply + load_model(iteration_20000)   ← scene/__init__.py:84-92
        ├─ skip_train=True  → train はスキップ
        ├─ skip_test=False  → render_set("test",  ...)   ← test/ours_20000/{renders,gt} + video_rgb.mp4
        └─ skip_video=False → render_set("video", ...)   ← video/ours_20000/renders + video_rgb.mp4（gt なし）
```

`render_set`（render.py:46）の各カメラ処理:
- `render(view, gaussians, pipeline, background, cam_type)["render"]` で推論（CUDA 拡張のラスタライザを使用）
- `renders/{00000..}.png`（torchvision.save_image, マルチスレッド書き出し）
- `name in ["train","test"]` のときのみ `gt/{00000..}.png` を書き出す（render.py:62）→ **video は gt なし**
- 全フレーム連結を `imageio.mimwrite(..., 'video_rgb.mp4', fps=30)` で書き出し（render.py:77）

### ディレクトリ構成（実行で生成されるパス）

```
output/dnerf/bouncingballs/
├── cfg_args                         # feat-004 で生成済み（入力。get_combined_args が読む）
├── point_cloud/
│   ├── iteration_14000/             # feat-004 成果物（入力）
│   └── iteration_20000/             # feat-004 成果物（入力。searchForMaxIteration で選択）
├── test/ours_20000/                 # ← 本案件で生成（FR-002）
│   ├── renders/00000.png .. 00019.png   # 推論画像 20 枚（= test カメラ数, §1.4.3）
│   ├── gt/00000.png .. 00019.png        # 正解画像 20 枚
│   └── video_rgb.mp4                     # FR-004
└── video/ours_20000/                # ← 本案件で生成（FR-003）
    ├── renders/00000.png .. 00159.png   # 推論画像 160 枚（= 円軌道フレーム数, §1.4.4）
    └── video_rgb.mp4                     # FR-004（gt なし）
```

## 1.3 技術スタック

- 実行: `.venv/bin/python`（Python 3.10、torch 1.13.1+cu116、mmcv 1.6.0）
- CUDA 拡張: `diff_gaussian_rasterization`（feat-002 ビルド）/ `simple_knn`
- 画像書き出し: `torchvision.utils.save_image`（PNG）
- 動画書き出し: `imageio` 2.37.3 + `imageio_ffmpeg` 0.6.0（導入済み。`mimwrite` で mp4）
- 新規ライブラリ追加なし（CLAUDE.md / requirements §1.5）

## 1.4 各機能の詳細設計

### 1.4.1 FR-001 / FR-005: レンダリングコマンドの実行（バックグラウンド＋ログ保存）

実行コマンド（公式 README:205・BACKLOG feat-005 と同一）:

```bash
mkdir -p /data/sakagawa/tmp/feat005-dnerf-render
CUDA_VISIBLE_DEVICES=0 .venv/bin/python render.py \
  --model_path "output/dnerf/bouncingballs/" \
  --skip_train \
  --configs arguments/dnerf/bouncingballs.py \
  > /data/sakagawa/tmp/feat005-dnerf-render/render.log 2>&1
```

- `CUDA_VISIBLE_DEVICES=0` で単一 GPU に固定（ADR-2）
- バックグラウンド実行（`run_in_background: true`）＋ログをファイルに保存。完了は harness の通知で受ける（ADR-1）
- 学習（feat-004 約10分）より短時間（180 フレーム）なので、完走をシグナル（`Rendering progress` 完了 / `Traceback` / `Error`）で監視する

### 1.4.2 FR-001: モデルロード（iteration の決定）

- `render.py:99` の `--iteration` 既定は `-1`。`Scene.__init__`（scene/__init__.py:35-39）で `load_iteration == -1` のとき `searchForMaxIteration(os.path.join(model_path, "point_cloud"))` を呼ぶ
- `searchForMaxIteration`（utils/system_utils.py:26-28）は `point_cloud/` 配下の各ディレクトリ名を `_` で split し最後の数値の最大値を返す
  - 本構成の `point_cloud/` は `iteration_14000`・`iteration_20000` のみ（feat-004 で coarse 保存なしを確定済み）→ `max(14000, 20000) = 20000`
  - → ロード対象は **iteration_20000**。出力パスの `ours_{20000}` もこの値に従う
- ロードは `gaussians.load_ply(iteration_20000/point_cloud.ply)` ＋ `gaussians.load_model(iteration_20000/)`（deformation*.pth）（scene/__init__.py:84-92）
- 標準出力に `Loading trained model at iteration 20000` が出れば、ロード対象の決定が想定どおり

### 1.4.3 FR-002: test セットのレンダリング（renders + gt）

- `--skip_train` 指定 → `skip_train=True` で train をスキップ。`skip_test` は未指定 → `False` で test を実行（render.py:89-90）
- test カメラは `readCamerasFromTransforms(path, "transforms_test.json", ...)` 由来（dataset_readers.py:318）。`transforms_test.json` のフレーム数は 20（feat-004 で確認）→ **20 カメラ**
- このカメラ読み込み経路は `dataset_readers.py:287`（`Image.fromarray(np.array(arr*255.0, dtype=np.uint8), "RGB")`）を通る。feat-004 で `np.byte`→`np.uint8` に修正済みのため Pillow 12.2.0 でも動作する（修正前は `TypeError`）
- 出力: `test/ours_20000/renders/{00000..00019}.png`（推論）と `test/ours_20000/gt/{00000..00019}.png`（正解、render.py:62 の `name in ["train","test"]` 分岐で生成）
- 本成果物は feat-006 評価（`metrics.py`）の入力。renders と gt の枚数一致（各 20 枚）が判定の核

### 1.4.4 FR-003: video セットのレンダリング

- `skip_video` 未指定 → `False` で video を実行（render.py:91-92）
- video カメラは `generateCamerasFromTransforms(path, "transforms_train.json", extension, max_time)` 由来（dataset_readers.py:320）
  - `render_poses = pose_spherical(angle, -30.0, 4.0)` を `np.linspace(-180,180,160+1)[:-1]` で生成 → **160 フレーム**の円軌道カメラ（dataset_readers.py:226）
  - `render_times = torch.linspace(0, maxtime, render_poses.shape[0])`（=160 フレーム、dataset_readers.py:227）、`time = time/maxtime`（dataset_readers.py:247）。bouncingballs の `max_time = 1.0`（train/test の time 最大値、feat-004 で確認）→ **ZeroDivision なし**
  - この経路はテンプレート画像を1枚 `Image.open` → `PILtoTorch` するのみで、`Image.fromarray(...,"RGB")` の型変換を**通らない** → feat-004 の Pillow 問題とは無関係
- video は正解画像を持たない（`name="video"` は render.py:62 の分岐に該当しない）→ `gt/` は作られない（`makedirs` で空ディレクトリは作られるが画像は 0 枚）
- 出力: `video/ours_20000/renders/{00000..00159}.png`（160 枚）

### 1.4.5 FR-004: 動画ファイル（mp4）の生成

- `render_set` 末尾（render.py:77）で `imageio.mimwrite(os.path.join(model_path, name, "ours_20000", 'video_rgb.mp4'), render_images, fps=30)` を実行
  - `render_images` は `to8b(rendering).transpose(1,2,0)` で uint8 化した HxWx3 配列のリスト（render.py:45,60）。`to8b` は元から `np.uint8`（Pillow 問題なし）
  - mp4 エンコードに ffmpeg が必要。`imageio_ffmpeg` 0.6.0 が同梱 ffmpeg を提供（導入済み・確認済み）
- test・video それぞれで `video_rgb.mp4` が生成される（train はスキップのため生成されない）

### 1.4.6 エラーハンドリングと境界条件

| ID | 事象 | 箇所 | 設計上の扱い |
|----|------|------|--------------|
| E1 | feat-004 成果物欠損（`point_cloud/` 空・`cfg_args` 不在） | scene/__init__.py:37 / arguments:158 | `searchForMaxIteration` が空リストで `max()` 例外、または source_path 未復元でロード不可。事前に成果物の存在を確認してから実行（§1.6 前提条件）。発生時は異常終了として investigation.md に記録 |
| E2 | CUDA 拡張の import/実行エラー | gaussian_renderer | feat-002 ビルドが壊れていれば import 失敗。feat-004 で実機動作確認済みのため発生しない想定。発生時は停止・記録 |
| E3 | Pillow/numpy 由来の型エラー再発 | dataset_readers.py:287 | test 経路は feat-004 修正済みで回避。video 経路は型変換を通らず無関係（§1.4.4）。万一新規箇所で再発した場合はコード変更せず中断し investigation.md に記録（CLAUDE.md ステップ7） |
| E4 | mp4 書き出し失敗（ffmpeg 不在） | render.py:77 | `imageio_ffmpeg` 0.6.0 導入済みで回避。失敗時は PNG（FR-002/003）は残るため、FR-004 のみ未達として切り分け |
| E5 | ポート競合 | render.py | `render.py` は network_gui を起動しない（train.py 専用）ため `--port` 不要・競合なし |
| E6 | VRAM 不足（OOM） | 推論 | 学習が完走している以上、より軽い推論で OOM は想定しない。発生時は停止・記録 |

自動リトライはしない（requirements §1.4 信頼性）。E1〜E6 のいずれかで完走しない場合、原因を investigation.md に記録し、ユーザー承認のうえ対処する。

## 1.5 状態遷移

```
[起動] → get_combined_args（cfg_args 読込）→ [hyperparam マージ]
   → Scene ロード（iteration=20000: ply + deformation）→ [カメラ生成 train/test/video]
   → render_set(test)  → renders/gt 20 枚 + mp4
   → render_set(video) → renders 160 枚 + mp4
   → [正常終了（プロセス終了）]

異常系: ロード失敗 / CUDA エラー / 型エラー / ffmpeg 不在 → [異常終了] → investigation.md 記録
```

`render.py` には学習のような完了メッセージ（`Training complete.`）はない。完走判定は「プロセスが終了コード0で終了し、想定の成果物が揃っていること」で行う（ログ末尾の `Rendering progress: 100%` と `FPS:` 出力も補助シグナル）。

## 1.6 ファイル・ディレクトリ設計

**前提条件（実行前に確認）**:
- `output/dnerf/bouncingballs/cfg_args` が存在する（feat-004 で生成済み）
- `output/dnerf/bouncingballs/point_cloud/iteration_20000/{point_cloud.ply,deformation.pth,deformation_table.pth,deformation_accum.pth}` が存在する（feat-004 で生成済み）
- `data/dnerf/bouncingballs/transforms_{train,test}.json` が存在する（feat-003 配置済み。カメラ生成に必要）

**生成物**: §1.2「ディレクトリ構成」参照。`output/` は `.gitignore` 管理外・非コミット。

## 1.7 インターフェース定義

- 入力引数:
  - `--model_path "output/dnerf/bouncingballs/"`（必須。成果物と cfg_args のルート）
  - `--skip_train`（train をスキップ）
  - `--configs arguments/dnerf/bouncingballs.py`（hyperparam）
  - `--iteration`（既定 -1 = 最大 iteration を自動選択。本案件は既定のまま）
  - `--skip_test` / `--skip_video`（本案件では指定しない＝両方出力）
- 出力: §1.2 のディレクトリ構成（PNG ＋ mp4）
- ログ: `/data/sakagawa/tmp/feat005-dnerf-render/render.log`（標準出力＋標準エラー）

## 1.8 ログ・デバッグ設計

- ログ保存先: `/data/sakagawa/tmp/feat005-dnerf-render/render.log`（リポジトリ外、隔離）
- 着目するログ行:
  - `Looking for config file in .../cfg_args` / `Config file found`（cfg_args 読込）
  - `Loading trained model at iteration 20000`（iteration 決定）
  - `Found transforms_train.json file, assuming Blender data set!`（Blender 経路）
  - `Reading Training Transforms` / `Reading Test Transforms` / `Generating Video Transforms`（カメラ生成）。`--skip_train` でも `Scene` 構築時に train カメラ（150 枚）が `readCamerasFromTransforms` 経由でロードされ、test と同様に `dataset_readers.py:287`（feat-004 修正済み）を通る（レンダリング自体は train スキップ）
  - `point nums: N`（ロードした Gaussian 数。feat-004 最終 27,769 前後と整合するか）
  - `Rendering progress: 100%` ＋ `FPS: ...`（各セット完走）
- 完走後の検証コマンド（手動テストで実行）:
  - `ls output/dnerf/bouncingballs/test/ours_20000/{renders,gt} | wc -l`（各 20）
  - `ls output/dnerf/bouncingballs/video/ours_20000/renders | wc -l`（160）
  - `ls -l output/dnerf/bouncingballs/{test,video}/ours_20000/video_rgb.mp4`（非空）

## 設計判断の記録（ADR簡易版）

### ADR-1: バックグラウンド実行＋ログファイルで起動する
feat-004 と同じ運用。180 フレームのレンダリングは学習より短いが、ターミナルブロックを避け完走シグナルで監視するため、`run_in_background` ＋ログ保存とする。

### ADR-2: `CUDA_VISIBLE_DEVICES=0` で単一 GPU に固定する
本マシンは A100×7。レンダリングは単一 GPU で十分（CLAUDE.md）。再現性のため 0 番に固定。

### ADR-3: コマンドは README・BACKLOG のものを変更せず使う
公式 README:205 と BACKLOG feat-005 のコマンドは同一。`--skip_train` のみ指定で test・video を出力する公式既定の挙動をそのまま使う。

### ADR-4: `--iteration` は既定（-1）のままにする
最大 iteration（20000 = 最終学習結果）を自動選択させる。feat-006 評価も最終結果に対して行うため整合する。iteration_14000 を明示ロードする必要はない。

### ADR-5: FR-002 を Must、feat-006 の前提として重視する
`metrics.py` は `test/ours_*/{renders,gt}` を比較する。test の renders/gt が揃わないと feat-006 が成立しないため、video（FR-003）と同じ Must だが評価連鎖上の重要度は FR-002 が最上位。

### ADR-6: コードは変更しない／新規ライブラリ追加なし
CLAUDE.md 開発方針。`imageio_ffmpeg` は導入済みで mp4 出力に追加導入は不要。互換問題で変更が不可避な場合のみ、中断して investigation.md ＋ユーザー承認（feat-004 前例、E3）。

## 未検証事項（実装時に実地検証）

- U1: `searchForMaxIteration` が実際に 20000 を選ぶか（ログ `Loading trained model at iteration 20000` で確認）
- U2: test の renders/gt が各 20 枚、video の renders が 160 枚で揃うか
- U3: `video_rgb.mp4`（test・video）が非空で生成されるか（ffmpeg 経路の実動）
- U4: `point nums:` がロード後に妥当な値（feat-004 最終 27,769 前後）を示すか
- U5: レンダリング所要時間（参考値）と FPS の実測

## 実装結果（2026-05-25）

設計どおりのコマンド（§1.4.1）でレンダリングを実行し、**全 FR 合格・コード変更ゼロ**で完走した。所要時間は約13秒（16:42:44 起動 → 16:42:57 完了）、exit code 0。

### 計画外対処（不具合）

なし。feat-004 で危惧した Pillow 12.2.0 の型互換問題（`dataset_readers.py:287`）は再発しなかった。video 生成（`generateCamerasFromTransforms`）は `Image.fromarray(...,"RGB")` を通らず、test/train 読み込みは feat-004 で `np.uint8` に修正済みのため。設計どおりコードは一切変更していない。

### 判定結果（FR-001〜005 全合格）

| FR | 結果 | 根拠 |
|----|------|------|
| FR-001 | ✓ | ログに `Looking for config file in output/dnerf/bouncingballs/cfg_args`・`Loading trained model at iteration 20000`・`Found transforms_train.json file, assuming Blender data set!`。CUDA 拡張・Pillow/numpy 型エラーなし。`point nums: 27769`（feat-004 最終と一致） |
| FR-002 | ✓ | `test/ours_20000/renders/` 20 枚・`gt/` 20 枚（同数・非空）。`renders/00000.png`=84,531B 等 |
| FR-003 | ✓ | `video/ours_20000/renders/` 160 枚（非空）。`gt/` は空ディレクトリのみ（0 枚＝設計どおり） |
| FR-004 | ✓ | `test/ours_20000/video_rgb.mp4`=78,975B・`video/ours_20000/video_rgb.mp4`=259,638B（いずれも非空） |
| FR-005 | ✓ | `/data/sakagawa/tmp/feat005-dnerf-render/render.log` にログ全体を保存 |

目視確認（手動テスト）も合格: ① renders 単体がシーンとして妥当、② test renders と gt が概ね一致、③ video（円軌道 mp4）が破綻なく回り込む、④ test の各コマ（frame 5/12/19 等）が視点・時刻ともに変化し破綻なし。

### 未検証事項の確定（U1〜U5）

- **U1**: 確定。`searchForMaxIteration` は **20000** を選択（ログ `Loading trained model at iteration 20000`）
- **U2**: 確定。test renders=20・gt=20、video renders=160 で揃った
- **U3**: 確定。test・video の `video_rgb.mp4` が非空生成（imageio + imageio_ffmpeg 0.6.0 の ffmpeg 経路が実動）
- **U4**: 確定。`point nums: 27769`（feat-004 最終点数と一致＝正しくロード）
- **U5**: 確定。所要時間 約13秒。FPS = test **14.97** / video **82.54**（video は gt 比較なしで高速）
