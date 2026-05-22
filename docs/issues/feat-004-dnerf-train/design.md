# feat-004 機能設計書: D-NeRF学習動作確認

## 1.1 対応要求マッピング

| 要求 | 設計箇所 |
|------|----------|
| FR-001 学習コマンドの実行 | §1.4.1 |
| FR-002 coarse+fine 2ステージ完走 | §1.4.2 |
| FR-003 成果物の生成 | §1.4.3 |
| FR-004 学習が機能している確認 | §1.4.4 |
| FR-005 実行ログの保存 | §1.4.1（実行形態）・§1.8 |
| エラーハンドリング・境界条件 | §1.4.5 |

## 1.2 システム構成

本案件はコードを変更せず、既存の学習パイプラインを起動・観測する。関与するモジュールと役割（すべて変更しない）:

```
train.py                                  … エントリポイント（引数解析→training()）
  │  args 解析（ModelParams/OptimizationParams/PipelineParams/ModelHiddenParams）
  │  --configs を mmcv.Config で読み、merge_hparams で上書き
  ▼
training(dataset, hyper, opt, pipe, ...)  … 出力先準備＋Scene構築＋2ステージ実行
  │  prepare_output_and_logger(expname) → ./output/{expname} 作成・cfg_args 書出し
  │  scene = Scene(dataset, gaussians)   → データ読込・点群初期化
  ├─ scene_reconstruction(stage="coarse", train_iter=opt.coarse_iterations=3000)
  └─ scene_reconstruction(stage="fine",   train_iter=opt.iterations=20000)
       │  各反復: render() → loss(L1[+ssim][+TV]) → backward → optimizer.step()
       │  densify / prune / grow / reset_opacity（densify_until_iter=15000 まで）
       │  saving_iterations で Scene.save()
       ▼
   gaussian_renderer.render() → diff_gaussian_rasterization（CUDA拡張・feat-002）
   scene.gaussian_model.create_from_pcd() → simple_knn distCUDA2（CUDA拡張・feat-002）
```

- データ読込経路（feat-003 で確認済み）: `scene/__init__.py:48` が `data/dnerf/bouncingballs/transforms_train.json` を検出し Blender データセットとして読み込む。点群は colmap が無いため `readNerfSyntheticInfo`（`scene/dataset_readers.py:327-339`）が**毎回ランダム 2000 点をメモリ上で生成する**（`xyz = np.random.random((2000,3))*2.6-1.3`。`setup_seed(6666)`（`train.py:397`）により決定的）。`fused.ply` への書き出しはコード上コメントアウトされている（`dataset_readers.py:337` `# storePly(...)`）ため**生成されない**。ただし `data/dnerf/bouncingballs/fused.ply` が事前に存在すれば `else` 分岐（`fetchPly`、`:339`）でそれを読み込む。本案件の調査時点で当該 `fused.ply` は存在しない（feat-003 配布データに含まれない）ため、毎回メモリ生成される
- 出力先: `prepare_output_and_logger`（`train.py:312`）が `args.model_path` 未指定時に `./output/{expname}` を採用する（`train.py:318-320`）。本案件は `--expname "dnerf/bouncingballs"` を渡すため `./output/dnerf/bouncingballs/` となる

### ディレクトリ構成（実行で生成されるパス）

```
output/dnerf/bouncingballs/
├── cfg_args                              … 実行時引数スナップショット（Namespace 文字列）
└── point_cloud/                          … （注）TensorBoard ログ events.out.tfevents.* は本環境では生成されない（後述）
    ├── iteration_14000/                  … 中間保存（saving_iterations のうち到達分）
    │   ├── point_cloud.ply
    │   ├── deformation.pth
    │   ├── deformation_table.pth
    │   └── deformation_accum.pth
    └── iteration_20000/                  … 最終保存（FR-003 の必須対象）
        ├── point_cloud.ply
        ├── deformation.pth
        ├── deformation_table.pth
        └── deformation_accum.pth
```

- coarse ステージでの保存は発生しない（後述 §1.4.3 の saving_iterations 解析より、coarse の上限 3000 以下に該当する保存タイミングが無いため）
- **TensorBoard ログ（`events.out.tfevents.*`）は本環境では生成されない**: `train.py:36-40` は `from torch.utils.tensorboard import SummaryWriter` を try/except し、失敗時 `TENSORBOARD_FOUND = False` とする。本案件調査で `.venv` に **tensorboard が未インストール**であることを確認済み（`import tensorboard` が `ModuleNotFoundError`）。よって `prepare_output_and_logger`（`train.py:329-332`）は `SummaryWriter` を作らず `Tensorboard not available: not logging progress` を出力して継続する（正常動作）。新規ライブラリは追加しない（ADR-6）ため、成果物検証で `events.out.tfevents.*` の存在を期待しないこと

## 1.3 技術スタック

- 実行: `.venv/bin/python`（uv 管理、Python 3.10、torch 1.13.1+cu116）
- CUDA 拡張: `diff_gaussian_rasterization` / `simple_knn`（feat-002 ビルド済み、import 動作は本案件調査で確認済み）
- 設定読込: `mmcv` 1.6.0（`mmcv.Config.fromfile` で `arguments/dnerf/bouncingballs.py` を読む）
- GPU: NVIDIA A100-SXM4-40GB を単一使用。本体は `device="cuda"`（= 可視 GPU の 0 番）を使うため、`CUDA_VISIBLE_DEVICES` で物理 GPU を1枚に固定する
- ログ保存: バックグラウンド実行＋リダイレクトでログファイルに保存（§1.8）
- 選定理由: いずれも公式手順・既存環境の踏襲であり、新規ライブラリの追加は無い（`requirements.lock.txt` / `TECH_STACK.md` の更新は不要）

## 1.4 各機能の詳細設計

共通の作業変数:

```bash
REPO=/data/sakagawa/4DGaussians
PY="$REPO/.venv/bin/python"
LOGDIR=/data/sakagawa/tmp/feat004-dnerf-train      # リポジトリ外のログ保存先
GPU=0                                              # 使用する物理GPU（全7枚空き、0を使用）
mkdir -p "$LOGDIR"
```

### 1.4.1 FR-001 / FR-005: 学習コマンドの実行（バックグラウンド＋ログ保存）

```bash
cd "$REPO"
CUDA_VISIBLE_DEVICES=$GPU "$PY" train.py \
  -s data/dnerf/bouncingballs \
  --port 6017 \
  --expname "dnerf/bouncingballs" \
  --configs arguments/dnerf/bouncingballs.py \
  > "$LOGDIR/train.log" 2>&1
```

- **データフロー**: コマンド引数 → `train.py` の argparse → `merge_hparams` で config 上書き → `training()`
- **処理ロジック**:
  - `CUDA_VISIBLE_DEVICES=0` で物理 GPU を1枚に固定（本体は単一 GPU 前提）
  - 標準出力・標準エラーを `$LOGDIR/train.log` に集約
  - **実行形態**: Bash ツールの `run_in_background: true` で起動し、完了通知を待つ（学習は数分〜20分。foreground はデフォルト 120s タイムアウトに当たるため使わない。引き継ぎノートの学び）
- **検証**: ログに以下が順に現れること
  1. `Found transforms_train.json file, assuming Blender data set!`（Blender 認識）
  2. `Loading Training Cameras` / `Loading Test Cameras`（データ読込）
  3. `data loading done`（`train.py:84`、ループ突入直前）
  4. `Training progress` の tqdm バー（CUDA 拡張が実行され反復が回っている証跡）
- **port について**: `network_gui.init(ip, 6017)`（`train.py:427`）がソケットを listen する。GUI クライアントが繋がらなくても `network_gui.conn == None` のまま学習は進む（`train.py:109-111`）。6017 が占有されている場合は §1.4.5 E1 に従い別ポートへ変更する

### 1.4.2 FR-002: coarse + fine 2ステージの完走

- **処理ロジック**: `training()`（`train.py:297-310`）が `scene_reconstruction` を coarse（`train_iter=opt.coarse_iterations=3000`）→ fine（`train_iter=opt.iterations=20000`）の順に2回呼ぶ。各ステージは `for iteration in range(first_iter, final_iter+1)` のループ（`train.py:108`）
- **設定値の出所**（`arguments/dnerf/dnerf_default.py` が `bouncingballs.py` の `_base_`）:
  - `coarse_iterations = 3000`
  - `iterations = 20000`
  - `pruning_interval = 8000`
  - `render_process = False`（→ 学習中の画像書き出し `render_training_image` は行われない。`train.py:247`）
- **検証**: プロセスが終了コード0で終了し、ログ末尾に `Training complete.`（`train.py:432`）が出力されること。`loss is nan,end training`（`train.py:221`）が出ていないこと
- **境界条件**: NaN 検出時、本体は `os.execv` で同一コマンドを**再実行**する（§1.4.5 E4）。これが起きるとログに学習が複数回現れ完走しないため、異常として扱う

### 1.4.3 FR-003: 成果物の生成

- **saving_iterations の確定**（`train.py:407,415` と config 上書きの相互作用を明示する）:
  - `train.py:407` の既定: `save_iterations = [14000, 20000, 30000, 45000, 60000]`
  - `train.py:415` `args.save_iterations.append(args.iterations)`: この時点の `args.iterations` は **argparse 既定の 30000**（config 上書きは `train.py:416-420` の後段で行われる）。よって 30000 が追加され `[14000, 20000, 30000, 45000, 60000, 30000]`
  - `train.py:419-420` の `merge_hparams` で `args.iterations` が config 値 **20000** に更新される（save_iterations のリスト要素自体は変わらない）
  - fine ステージは 20000 反復で終わるため、保存が実際に発生するのは **14000 と 20000**（30000 以降は到達しない）。coarse は 3000 反復で終わり、リスト中の最小値 14000 にも届かないため coarse 保存は無し
- **保存処理**: `train.py:244-246` が `if (iteration in saving_iterations): scene.save(iteration, stage)` を実行。`Scene.save`（`scene/__init__.py:96-103`）が fine では `point_cloud/iteration_{N}/` を作り、`save_ply`（`point_cloud.ply`）と `save_deformation`（`deformation.pth` / `deformation_table.pth` / `deformation_accum.pth`、`scene/gaussian_model.py:246-249`）を書き出す
- **検証コマンド**:

```bash
OUT="$REPO/output/dnerf/bouncingballs"
ls -la "$OUT/cfg_args"
ls -la "$OUT/point_cloud/iteration_20000/"
"$PY" - <<PY
import os
out = "$OUT"
need = [
  "cfg_args",
  "point_cloud/iteration_20000/point_cloud.ply",
  "point_cloud/iteration_20000/deformation.pth",
  "point_cloud/iteration_20000/deformation_table.pth",
  "point_cloud/iteration_20000/deformation_accum.pth",
]
bad = []
for f in need:
    p = os.path.join(out, f)
    if not os.path.exists(p):
        bad.append(("missing", f)); continue
    if f.endswith("point_cloud.ply") and os.path.getsize(p) == 0:
        bad.append(("empty", f))
assert not bad, bad
print("FR-003 OK:", {f: os.path.getsize(os.path.join(out, f)) for f in need})
PY
```

- **受け入れ基準**: 上記スクリプトが `FR-003 OK` を出力（必須5項目が存在し ply が非空）すること

### 1.4.4 FR-004: 学習が機能していることの確認

- **処理ロジック**: `train.py:406` の既定 `test_iterations = [3000, 7000, 14000]` の各反復で `training_report`（`train.py:335`）が test/train 各サンプルの L1・PSNR を計算し `[ITER N] Evaluating test: L1 {x} PSNR {y}`（`train.py:372`）を出力する
- **イテレーションはステージごとにリセットされる**: `scene_reconstruction` は冒頭で `first_iter = 0`（`train.py:44`）とし、ループは `range(first_iter, final_iter+1)`（`train.py:108`）でステージ内カウントを回す。`test_iterations` はこのステージ内カウントで判定されるため、評価ログは次のとおり計4回出力される（`training_report` の print にステージ名が含まれない（`train.py:372`）点に注意）:
  - coarse（上限3000）: ITER 3000 で1回
  - fine（上限20000）: ITER 3000 / 7000 / 14000 で3回
  - 結果、`[ITER 3000] Evaluating test:` はログに2回現れる（1回目=coarse、2回目=fine）
- **検証**: ログから test 評価行を抽出し、**fine ステージの ITER 14000**（14000 は fine でのみ到達するため一意）の test PSNR を確認する

```bash
grep "Evaluating test" "$LOGDIR/train.log"
```

- **受け入れ基準**: 評価行が出力され、ITER 14000 の test PSNR が **20 以上**であること（FR-004 受入基準）。NaN や 1桁台の場合は環境異常（CUDA 演算・データ）を疑い、investigation.md に記録する
- **境界条件**: tqdm の進捗バー postfix にも `psnr` が表示される（`train.py:235`）が、これは直近バッチの train PSNR でありゲート判定には test 評価行を使う

### 1.4.5 エラーハンドリングと境界条件

| ID | 事象 | 検出方法 | 対応 |
|----|------|----------|------|
| E1 | ポート 6017 が使用中（`network_gui.init` でバインド失敗・例外） | 起動直後にログへ traceback | `--port` を別の空きポート（例 6018）に変更して再実行。学習結果に影響しない |
| E2 | CUDA 拡張の import / 実行失敗（`undefined symbol`・ABI 不一致等） | ログに ImportError / RuntimeError | feat-002 のビルドを `docs/issues/feat-002-cuda-ext-build/` 手順で確認。本案件調査では import 成功を確認済み |
| E3 | VRAM 不足（CUDA out of memory） | ログに `CUDA out of memory` | 他プロセスの GPU 占有を `nvidia-smi` で確認。空き GPU 番号を `CUDA_VISIBLE_DEVICES` に指定 |
| E4 | 学習発散・NaN（`train.py:220` で検出し `os.execv` 再起動） | ログに `loss is nan,end training, reexecv program now.` | 異常終了として扱い再実行を止める（プロセス kill）。investigation.md に記録し原因（データ・乱数・設定）を調査 |
| E5 | データ未配置（`assert False, "Could not recognize scene type!"`） | ログに AssertionError | feat-003 の `data/dnerf/bouncingballs/` 配置を確認（本案件では配置済み） |
| E6 | バックグラウンドプロセスが途中で停止（OOM kill 等で `Training complete.` が出ない） | 完了通知後にログ末尾を確認 | ログ末尾・終了コードを確認し原因を切り分け。investigation.md に記録 |

### 1.4.6 GPU 選択の境界条件

- 調査時点で全7枚の A100 が空き（`nvidia-smi` 確認）。`GPU=0` を使用する
- 実行直前に `nvidia-smi` で 0 番の空きを再確認し、占有されていれば空き番号に変更する（学習は単一 GPU で完結するため番号は結果に影響しない）

## 1.5 状態遷移

```
[未実行] --FR-001 起動--> [coarse 学習中(0→3000)] --自動--> [fine 学習中(0→20000)]
   --iteration∈{14000,20000} で保存--> [成果物書出し]
   --Training complete.--> [完走] --FR-003/004 検証--> [合格]
        |NaN(E4)                |OOM/停止(E6)
        └--> [異常: 再起動ループ]  └--> [異常: 途中停止]  → investigation.md に記録
```

## 1.6 ファイル・ディレクトリ設計

- **生成（成果物・git管理外）**: `output/dnerf/bouncingballs/`（`.gitignore` 3行目 `output` で管理対象、`git check-ignore` で確認済み。コミットしない）
  - `cfg_args`、`point_cloud/iteration_{14000,20000}/{point_cloud.ply, deformation.pth, deformation_table.pth, deformation_accum.pth}`
  - `events.out.tfevents.*`（TensorBoard ログ）は本環境（tensorboard 未導入）では**生成されない**（§1.2 注記参照）
- **データ側の副作用**: なし。点群はメモリ上でランダム生成され、`fused.ply` の書き出しはコメントアウト済み（`dataset_readers.py:337`）のため `data/dnerf/bouncingballs/` 配下に新規ファイルは作られない（§1.2 データ読込経路参照）
- **一時（リポジトリ外・任意削除）**: `$LOGDIR/train.log`（= `/data/sakagawa/tmp/feat004-dnerf-train/train.log`。`$LOGDIR` はディレクトリ、ログ本体はその直下の `train.log`）。実行ログは完走確認後も当面残し、クローズ時に削除可
- **コミット対象**: 本案件ドキュメント（`docs/issues/feat-004-dnerf-train/*.md`）と必要に応じた `BACKLOG.md`・`CLAUDE.md` の更新のみ。`output/`・`data/` はコミットしない

## 1.7 インターフェース定義

本案件はコードを書かない（既存 CLI の実行のみ）。インターフェースは `train.py` の CLI:

- 入力 CLI: `python train.py -s {source_path} --port {int} --expname {str} --configs {config.py}`
  - `-s data/dnerf/bouncingballs`（`ModelParams._source_path`、`arguments/__init__.py:65` で絶対パス化）
  - `--port 6017`（`network_gui` ソケット）
  - `--expname "dnerf/bouncingballs"`（出力先 `./output/{expname}`）
  - `--configs arguments/dnerf/bouncingballs.py`（mmcv で読み hparams を上書き）
- 出力: §1.6 の成果物
- 後続接続: feat-005 が `python render.py --model_path "output/dnerf/bouncingballs/" --skip_train --configs arguments/dnerf/bouncingballs.py` で本成果物を読む

## 1.8 ログ・デバッグ設計

- 学習ログは `$LOGDIR/train.log` に集約（標準出力＋標準エラー）
- 進捗観測ポイント（ログ内文字列）:
  - 起動: `Found transforms_train.json file, assuming Blender data set!` / `data loading done`
  - 評価: `[ITER N] Evaluating test: L1 ... PSNR ...`（`train.py:372`）
  - 保存: `[ITER N] Saving Gaussians`（`train.py:245`）
  - 完了: `Training complete.`（`train.py:432`）
  - 異常: `loss is nan,end training`（`train.py:221`）/ `CUDA out of memory` / traceback
- バックグラウンド実行中は `tail -n 30 "$LOGDIR/train.log"` で進捗を随時確認できる。完了は Bash ツールの背景完了通知で受け取る

## 設計判断の記録（ADR簡易版）

### ADR-1: バックグラウンド実行＋ログファイルで起動する
学習は数分〜20分かかり、Bash の foreground デフォルト 120s タイムアウト・`sleep` ブロック（引き継ぎノートのハマりどころ）に当たる。`run_in_background: true` で起動し完了通知を待ち、ログはファイルに集約して後から検証する。

### ADR-2: `CUDA_VISIBLE_DEVICES=0` で単一 GPU に固定する
本体は `device="cuda"`（可視 GPU の 0 番）を使う単一 GPU 前提。全7枚空きのうち物理 0 番を割り当て、他作業との干渉を避ける。番号は学習結果に影響しない。

### ADR-3: コマンドは README・BACKLOG のものを変更せず使う
公式 README（140〜143行目、コマンドは143行目）と BACKLOG（feat-004）に記載のコマンドをそのまま使う。`--expname` 必須（未指定だと `./output/` 直下に出力。引き継ぎノートの注意点）。引数を独自に増やさない（本体挙動を素直に確認するため）。ただしポート 6017 が占有されている場合の `--port` 値変更は E1 に従い許容する（ポート値は学習結果に影響しない。引数の追加ではなく既存引数の値変更）。

### ADR-4: FR-003 の必須成果物は iteration_20000 とする
fine 最終の 20000 が学習完走の証跡。中間 14000 も生成されるが、完走判定には最終保存の存在を用いる（§1.4.3 の saving_iterations 解析に基づく）。

### ADR-5: FR-004 の PSNR 下限は 20 とする
本案件の目的は品質最適化ではなく「環境が正しく学習を回せる」確認。D-NeRF 合成は公式で 30+ に達するが、環境健全性のゲートとしては 20 を下限とし、NaN・1桁台を環境異常として弾く。厳密な品質評価は feat-006 で行う。

### ADR-6: コードは変更しない／新規ライブラリ追加なし
CLAUDE.md 開発方針に従い本体コードを変更しない。使用ライブラリは既存（torch / mmcv / CUDA 拡張 / lpips）のみで、`requirements.lock.txt`・`TECH_STACK.md` の更新は不要。

## 未検証事項（実装時に実地検証）

- **U1**: 実際の学習完走時間（A100 単一での coarse+fine 計 23000 反復の所要）。→ ログのタイムスタンプで実測し「実装結果」に記録
- **U2**: ITER 14000 の test PSNR 実値（FR-004 ゲート 20 に対する余裕）。→ ログから実測値を記録
- **U3**: 学習後の最終 Gaussian 点数（初期ランダム 2000 点から densify/prune を経て、上限 360,000（`train.py:270`）に対する到達点）。→ tqdm postfix の `point` 値・最終保存 ply から確認
- **U4**: ポート 6017 の実バインド可否（占有時は E1 で別ポート）。→ 起動時に確認
- **U5**: 6017 を使う `network_gui` がバックグラウンド常駐ソケットとして残らないか（プロセス終了で解放されるはず）。→ 完走後に確認
