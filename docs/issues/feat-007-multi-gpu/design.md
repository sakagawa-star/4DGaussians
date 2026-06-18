# feat-007 機能設計書: マルチGPU運用対応（任意GPU選択）

## 1.1 対応要求マッピング

| 要求 | 内容 | 設計箇所 |
|------|------|----------|
| FR-001 | 任意 GPU（N≠0）での学習完走 | §1.4.1 / §1.4.2 |
| FR-002 | 任意 GPU（N≠0）でのレンダリング完走 | §1.4.1 / §1.4.2 |
| FR-003 | 任意 GPU（N≠0）での評価完走 | §1.4.1 / §1.4.2 |
| FR-004 | 指定物理 GPU への負荷限定の確認 | §1.4.3 |
| FR-005 | マルチGPU運用ルールの文書化 | §1.4.5 |
| FR-006 | 4DGaussians 本体コードの非改変 | §1.3 / ADR-1 |
| （全 FR 横断） | エラーハンドリングと境界条件 | §1.4.4 |

本案件は**コードを変更しない**。設計の中心は「`cuda:0` / `cuda` のハードコードがあるにもかかわらず `CUDA_VISIBLE_DEVICES=N` で物理 GPU N に乗る」というデバイス選択メカニズムを明示し、3 経路を GPU≠0 で完走させる手順と、負荷が指定 GPU に限定されることの確認方法、そして運用ルールの文面を確定することにある。Codex レビュー #01（`reviews/codex-01.result.md`）の指摘を反映し、(高) GPU 番号一致のため `CUDA_DEVICE_ORDER=PCI_BUS_ID` を全実行に付与、(中) FR-004 の証跡として PID を `train.pid` に記録、(中) network_gui ポートの空き確認を手順化した。

## 1.2 システム構成

### GPU 選択メカニズム（本案件の核心）

```
環境変数 CUDA_DEVICE_ORDER=PCI_BUS_ID  CUDA_VISIBLE_DEVICES=N
   │  CUDA_DEVICE_ORDER=PCI_BUS_ID: CUDA の列挙順を nvidia-smi の index 順(PCI順)に揃える
   │  CUDA_VISIBLE_DEVICES=N      : 物理 GPU N だけをプロセスから可視にする
   ▼
物理 GPU N だけが可視 → 物理 GPU N が論理デバイス cuda:0（先頭）に再番号付け
   ▼
本体コードの "cuda:0" / "cuda"（論理先頭）= 物理 GPU N を指す
   ├─ train.py:424   safe_state() → general_utils.py:139 set_device(cuda:0)
   ├─ render.py:113  safe_state() → general_utils.py:139 set_device(cuda:0)
   │                 render.py:66,84 .cuda() / device="cuda"
   └─ metrics.py:116-117  device=cuda:0; set_device(device)
```

要点1: `cuda:0` は「**マスク後の論理デバイス先頭**」であり「物理 GPU0」ではない。`CUDA_VISIBLE_DEVICES=N` を与えた瞬間、論理 `cuda:0` の実体は物理 GPU N になる（コードを変えずに物理 GPU を選べる。実機検証 2026-06-18 で N=5 を確認済み）。

要点2: ただし「`nvidia-smi` で見た index=N」と「`CUDA_VISIBLE_DEVICES=N` が選ぶ GPU」が一致するには **`CUDA_DEVICE_ORDER=PCI_BUS_ID` が必要**。CUDA の既定順 `FASTEST_FIRST` は計算能力順で、`nvidia-smi`（PCI バス順）と一致する保証がない。全 GPU が同一機種（A100×7）でも順序保証はないため、明示的に `PCI_BUS_ID` を付ける（Codex #01 高指摘）。

### GPU 固定箇所（コード全文 grep 結果。これ以外に固定なし）

| ファイル:行 | コード | 経路 |
|------|------|------|
| `utils/general_utils.py:139` | `torch.cuda.set_device(torch.device("cuda:0"))`（`safe_state` 内） | train / render |
| `metrics.py:116` | `device = torch.device("cuda:0")` | metrics |
| `metrics.py:117` | `torch.cuda.set_device(device)` | metrics |
| `render.py:66,84` | `view['image'].cuda()` / `device="cuda"` | render |

分散学習機構（`DataParallel`/`DistributedDataParallel`/`torch.distributed`）はコード全文に 0 件。

### network_gui ポートのコード挙動（Codex #01 中指摘・確認済み）

`train.py:427` が `network_gui.init(args.ip, args.port)` を呼ぶ。`gaussian_renderer/network_gui.py:26-32`:
```python
def init(wish_host, wish_port):
    global host, port, listener
    host = wish_host; port = wish_port
    listener.bind((host, port))   # ← try/except なし。使用中ポートだと OSError
    listener.listen()
    listener.settimeout(0)
```
`bind()` は例外処理されていないため、指定ポートが使用中だと **学習開始前に `OSError` でクラッシュ**する。よって実行前にポート空き確認が必須。

### ディレクトリ構成（入力と生成パス）

```
data/dnerf/bouncingballs/                 # 入力（feat-003 取得済み）
output/dnerf/bouncingballs_feat007/       # 本案件の生成物（feat-004〜006 とは別パス）
├── point_cloud/iteration_{14000,20000}/point_cloud.ply   # FR-001 学習成果物
├── test/ours_20000/{renders,gt}/*.png                    # FR-002 レンダリング成果物
├── video/ours_20000/renders/ + video_rgb.mp4             # FR-002
├── results.json / per_view.json                          # FR-003 評価成果物
└── （cfg_args, deformation*.pth 等）
/data/sakagawa/tmp/feat007-multi-gpu/
├── {train,render,metrics}.log            # 各経路のログ（リポジトリ外）
├── train.pid                             # 学習プロセスの PID（FR-004 証跡）
└── nvidia-smi-during-train.txt           # 学習中の nvidia-smi 出力（FR-004 証跡）
```

## 1.3 技術スタック

- 実行: `.venv/bin/python`（Python 3.10、torch 1.13.1+cu116、torchvision 0.14.1+cu116）
- GPU 選択: 環境変数 `CUDA_DEVICE_ORDER=PCI_BUS_ID` ＋ `CUDA_VISIBLE_DEVICES=N`（コード変更なし。ADR-1 / ADR-7）
- 監視: `nvidia-smi`（Driver 565.57.01 同梱、CUDA Toolkit 不要）
- ポート確認: `ss -ltn`（iproute2、システム標準）
- 設定: `arguments/dnerf/bouncingballs.py`（feat-004〜006 と同一。`_base_ = dnerf_default.py`、iterations=20000 / coarse_iterations=3000）
- **新規ライブラリ追加なし／コード変更なし**（CLAUDE.md 開発方針、FR-006）

## 1.4 各機能の詳細設計

### 1.4.1 FR-001〜003: 空き GPU・ポートの選定と 3 経路の実行手順

**手順 0: 空き GPU（N）と空きポート（P）の選定**

```bash
# GPU 一覧（index は PCI 順。空き＝memory.used 小・utilization.gpu 低）
nvidia-smi --query-gpu=index,pci.bus_id,uuid,memory.used,utilization.gpu --format=csv

# 選んだ N のポート候補(P=6107)が空いているか確認（出力が無ければ空き）
ss -ltn | grep -E ':6107\b' || echo "port 6107 free"
```

- GPU: `memory.used` が十分小さく（目安 < 1000 MiB）`utilization.gpu` が 0% に近い GPU を「空き」とみなす。**N は 0 以外**から 1 枚選ぶ（複数空きなら最小の N、例 1）。選んだ GPU の `index`/`pci.bus_id`/`uuid` を控える（FR-004 の照合に使う）。
- ポート: `P` の既定候補は **6107**（ADR-4）。`ss -ltn` で使用中なら 6108, 6109… と空いている値に変える。確定した `P` を学習コマンドの `--port` に渡す。
- 選んだ N を 3 経路すべてで共通に使う（検証の一貫性のため同一 N）。

**手順 1〜3: 3 経路の実行**（バックグラウンド＋ログ保存＋PID 記録。`run_in_background: true` で起動し完走通知で監視。ADR-3）

```bash
mkdir -p /data/sakagawa/tmp/feat007-multi-gpu
TMP=/data/sakagawa/tmp/feat007-multi-gpu

# 再実行時は古い成果物での誤合格を防ぐため出力先を削除（初回は存在しない。E7 / ADR-8）
rm -rf output/dnerf/bouncingballs_feat007

# 1) 学習（N=選択GPU, P=空きポート。$! を train.pid に保存して FR-004 の証跡にする）
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python train.py \
  -s data/dnerf/bouncingballs --port P \
  --expname "dnerf/bouncingballs_feat007" \
  --configs arguments/dnerf/bouncingballs.py \
  > $TMP/train.log 2>&1 &
TRAIN_PID=$!; echo "$TRAIN_PID" > $TMP/train.pid

# （学習稼働中に別シェルで FR-004 の nvidia-smi 記録を取る: §1.4.3）

# 学習の終了コードをゲート: 失敗したら render/metrics に進まず中断（古い成果物での誤合格を防ぐ）
wait "$TRAIN_PID" || { echo "train failed (see $TMP/train.log)"; exit 1; }

# 2) レンダリング（学習成功後のみ。network_gui 不使用＝ポート確認不要）
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python render.py \
  --model_path "output/dnerf/bouncingballs_feat007/" --skip_train \
  --configs arguments/dnerf/bouncingballs.py \
  > $TMP/render.log 2>&1 || { echo "render failed (see $TMP/render.log)"; exit 1; }

# 3) 評価（レンダリング成功後のみ）
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python metrics.py \
  --model_paths output/dnerf/bouncingballs_feat007/ \
  > $TMP/metrics.log 2>&1 || { echo "metrics failed (see $TMP/metrics.log)"; exit 1; }
```

- `--expname "dnerf/bouncingballs_feat007"` で出力先を分離（既存 feat-004〜006 成果物を保護。ADR-2）。
- 行頭の **`CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N`** が GPU 選択。`N`/`P` は手順 0 で確定した実値に置換する。
- 学習のみ `& TRAIN_PID=$!`（→ `train.pid`）で PID を記録（FR-004）。harness の `run_in_background` で起動する場合も、このシェル全体を 1 ジョブとして渡し `train.pid` に実 PID を残す。
- **各経路は終了コードでゲートする**: 学習が非 0 終了したら render/metrics に進まない（`wait "$TRAIN_PID" || exit 1`）。固定出力先に古い成果物が残ると学習失敗でも後続が既存成果物で通り誤合格しうるため、再実行時は出力先を `rm -rf` してから開始する（E7 / ADR-8、Codex #02 高指摘）。
- render/metrics は network_gui を使わないためポート確認は不要。

### 1.4.2 FR-001〜003: 完走判定

各経路の完走シグナル（feat-004〜006 の判定基準を踏襲）:

| 経路 | 完走判定 |
|------|----------|
| 学習 | 終了コード 0。`output/dnerf/bouncingballs_feat007/point_cloud/iteration_20000/point_cloud.ply` 生成。ログに coarse(〜3000)→fine(〜20000) の進捗と最終 point 数。起動時に `OSError`（ポート）/`invalid device ordinal`（GPU）が出ない |
| レンダリング | 終了コード 0。`test/ours_20000/renders/` に 20 枚の PNG、`video_rgb.mp4` 生成。ログに `point nums:` と iteration_20000 自動選択 |
| 評価 | 終了コード 0。6 指標が数値 print、`results.json`/`per_view.json` 生成。指標が feat-006 実測と整合（PSNR≈40、桁違いの乖離なし） |

### 1.4.3 FR-004: 指定物理 GPU への負荷限定の確認

学習（手順 1）が稼働中に別シェルで以下を実行し、出力を `nvidia-smi-during-train.txt` に保存する:

```bash
TMP=/data/sakagawa/tmp/feat007-multi-gpu
PID=$(cat $TMP/train.pid)
nvidia-smi > $TMP/nvidia-smi-during-train.txt           # Processes 欄で PID と GPU index の対応
nvidia-smi --query-compute-apps=pid,gpu_uuid,used_memory --format=csv >> $TMP/nvidia-smi-during-train.txt
echo "target PID=$PID" >> $TMP/nvidia-smi-during-train.txt
```

**確認内容**:
1. 対象 PID は `train.pid` の値（`pgrep` は使わない＝同名残プロセス誤認を避ける。Codex #01 中指摘）。
2. `CUDA_DEVICE_ORDER=PCI_BUS_ID` 下で、`nvidia-smi` の Processes 欄の当該 PID が **GPU index=N の行にのみ**現れる。
3. GPU0 を含む他 GPU の行に当該 PID が**現れない**。
4. GPU N の `Memory-Usage` と `GPU-Util` が学習負荷で上昇している。
5. 手順 0 で控えた GPU N の `pci.bus_id`/`uuid` と、`--query-compute-apps` の `gpu_uuid` が一致する（index と物理 GPU の対応を二重確認）。

**判定**: 上記 2〜3（＋5 の uuid 照合）を満たせば「`cuda:0` ハードコードでも物理 GPU0 に漏れず、指定した物理 GPU N にのみ負荷が乗る」ことが実証される（FR-004 合格）。

### 1.4.4 エラーハンドリングと境界条件

| ID | 事象 | 検出 | 設計上の扱い |
|----|------|------|--------------|
| E1 | 指定 GPU N が他者使用中で VRAM 不足（OOM） | 学習/レンダリングが `CUDA out of memory` で異常終了 | 実行前 `nvidia-smi`（§1.4.1 手順 0）で空きを確認して回避。発生時は別の空き N で再実行（自動リトライはしない）。`investigation.md` に記録 |
| E2 | 存在しない GPU 番号 N を指定（例 N=7） | 起動時 `CUDA error: invalid device ordinal` | N は 0〜6 の範囲で指定。誤指定時はコマンド修正のうえ再実行 |
| E3 | `CUDA_VISIBLE_DEVICES`/`CUDA_DEVICE_ORDER` を付け忘れる | 前者欠落→論理 `cuda:0`＝（PCI順先頭の）GPU を使い混雑なら OOM。後者欠落→`nvidia-smi` index と CUDA 順がズレ別 GPU に乗る | 両環境変数を運用ルール（§1.4.5）で必須化。3 経路すべてに付ける |
| E4 | ポート使用中で `bind` 失敗 | `train.py:427`→`network_gui.py:30` の `listener.bind()` が `OSError`（try/except なし）→ 学習開始前にクラッシュ | 手順 0 で `ss -ltn` により `P` の空きを確認して回避。並行実行・他者使用時は別ポートに変える。render/metrics は無関係 |
| E5 | 学習成果物の出力先が既存と衝突 | `output/dnerf/bouncingballs/` を上書き | `--expname dnerf/bouncingballs_feat007` で別パスに分離（ADR-2）。誤って同一 expname を使わない |
| E6 | Pillow/numpy 型互換（feat-004 の前例）の再発 | 学習/評価で `TypeError` | feat-004 で `dataset_readers.py:287` を `np.uint8` 修正済み（既適用）。GPU 変更と無関係。再発時はコード変更せず中断・記録 |
| E7 | 学習失敗時に後続（render/metrics）へ進み、古い成果物で誤合格 | 学習が非 0 終了したのに後続が既存 `output/.../bouncingballs_feat007/` を使って通る | 各経路を終了コードでゲートし、学習失敗時は中断（§1.4.1）。再実行時は出力先を `rm -rf` してから開始（Codex #02 高指摘） |

自動リトライはしない（requirements §1.4 信頼性）。E1〜E7 で完走しない場合は原因を `investigation.md` に記録し、ユーザー承認のうえ対処する。

### 1.4.5 FR-005: マルチGPU運用ルールの文書化（CLAUDE.md 追記文面案）

`CLAUDE.md` に新節「## マルチGPU運用ルール」を追加する。文面案（実装ステップで CLAUDE.md に反映）:

> ### マルチGPU運用ルール（任意GPU選択）
>
> 本マシンは A100 × 7 を複数人で共用する。学習・レンダリング・評価は**1 プロセス = 1 GPU**で動かし、空いている任意の GPU を選んで使う。
>
> 1. **実行前に空き GPU を確認**する: `nvidia-smi --query-gpu=index,pci.bus_id,uuid,memory.used,utilization.gpu --format=csv`。`memory.used` が小さく `utilization.gpu` が低い GPU 番号 N を 1 枚選ぶ。
> 2. **実行コマンドには `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N` を付ける**。`CUDA_DEVICE_ORDER=PCI_BUS_ID` が無いと `nvidia-smi` の index と CUDA の番号がズレうる。同一ジョブの 3 経路（train/render/metrics）は同一 N を使う。
>    例: `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=3 .venv/bin/python train.py ...`
> 3. コード内の `cuda:0` / `cuda`（`utils/general_utils.py:139`、`metrics.py:116-117`、`render.py`）は **マスク後の論理デバイス先頭**を指すため物理 GPU N に乗る。**コード変更は不要**。
> 4. **分散学習（複数 GPU 同時使用）は本体非対応**。1 ジョブで複数 GPU は使えない。
> 5. **学習はポートを使う**（`--port`、既定 6009）。実行前に `ss -ltn | grep :<port>` で空きを確認する。使用中だと `network_gui` の `bind()` が例外処理されておらず**起動時にクラッシュ**する。並行実行時もポートを変える。render/metrics はポートを使わない。

この文面案は requirements FR-005 の 5 項目と一対一対応する。

## 1.5 状態遷移

ステートフル GUI は無い。本案件のフローは直線的:

```
[手順0: nvidia-smi で N 決定（N≠0）/ ss -ltn で P 決定]
   → 学習 train.py (CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N --port P, $!→train.pid)
        ─（稼働中に別シェルで nvidia-smi → train.pid の PID で FR-004 負荷限定確認）
   → [point_cloud.ply 生成・exit 0]
   → レンダリング render.py (同 N) → [test/renders 20枚・exit 0]
   → 評価 metrics.py (同 N) → [6指標・results.json・exit 0]
   → CLAUDE.md にマルチGPU運用ルール節を追記
   → [全 FR 充足 → クローズ]

異常系: OOM / invalid device ordinal / ポート bind 失敗 / GPU順序ズレ / 型エラー → investigation.md 記録 → ユーザー承認のうえ対処
```

## 1.6 ファイル・ディレクトリ設計

**前提条件（実行前に確認）**:
- `data/dnerf/bouncingballs/`（feat-003 取得済み）が存在する
- `.venv` と CUDA 拡張（feat-001/002）が利用可能
- `dataset_readers.py:287` の `np.uint8` 修正（feat-004）が適用済み

**生成物**: `output/dnerf/bouncingballs_feat007/`（学習・レンダリング・評価成果物。`.gitignore` 管理外・非コミット）。ログ・PID・nvidia-smi 記録は `/data/sakagawa/tmp/feat007-multi-gpu/`。

**文書更新**: `CLAUDE.md`（マルチGPU運用ルール節を追記）、`docs/BACKLOG.md`（feat-007 を Closed に）。

## 1.7 インターフェース定義

コードは変更しないため新規 API は無い。本案件の「インターフェース」は実行コマンドと環境変数:

- 環境変数: `CUDA_DEVICE_ORDER=PCI_BUS_ID`（必須）＋ `CUDA_VISIBLE_DEVICES=N`（N=物理 GPU 番号、本案件は N≠0）
- 学習: `train.py -s <data> --port <P> --expname <name> --configs <cfg>`（`P` は空きポート）
- レンダリング: `render.py --model_path <out> --skip_train --configs <cfg>`
- 評価: `metrics.py --model_paths <out>`
- 監視: `nvidia-smi` / `nvidia-smi --query-gpu=index,pci.bus_id,uuid,memory.used,utilization.gpu --format=csv` / `nvidia-smi --query-compute-apps=pid,gpu_uuid,used_memory --format=csv`
- ポート確認: `ss -ltn | grep :<P>`

## 1.8 ログ・デバッグ設計

- ログ・証跡の保存先: `/data/sakagawa/tmp/feat007-multi-gpu/`（リポジトリ外、隔離）
  - `{train,render,metrics}.log`（各経路の標準出力＋標準エラー）
  - `train.pid`（学習プロセスの PID。FR-004 で照合）
  - `nvidia-smi-during-train.txt`（学習中の nvidia-smi 出力。FR-004 証跡）
- 着目するログ行:
  - 学習: coarse/fine の iteration 進捗、`point nums:`、最終 `iteration_20000` 保存、exit 0（`OSError`/`invalid device ordinal` が出ないこと）
  - レンダリング: `point nums:`、iteration_20000 自動選択、test 20 枚／video 書き出し、exit 0
  - 評価: `Scene:`/`Method: ours_20000`/6 指標、`results.json` 生成、exit 0
  - GPU 確認: `nvidia-smi` 出力で `train.pid` の PID が GPU index=N にのみ紐づく（FR-004）
- 完走後の検証コマンド（手動テスト）:
  - `ls output/dnerf/bouncingballs_feat007/point_cloud/iteration_20000/point_cloud.ply`
  - `ls output/dnerf/bouncingballs_feat007/test/ours_20000/renders/ | wc -l`（20）
  - `.venv/bin/python -c "import json; print(json.load(open('output/dnerf/bouncingballs_feat007/results.json'))['ours_20000'])"`
  - `git status --short`（本体 `*.py` に差分が無いこと＝FR-006）

## 設計判断の記録（ADR簡易版）

### ADR-1: コードは変更せず `CUDA_VISIBLE_DEVICES` で物理 GPU を選ぶ
**採用**: `cuda:0` / `cuda` は `CUDA_VISIBLE_DEVICES` でマスクした後の論理先頭を指すため、環境変数だけで物理 GPU を選べる（実機検証 2026-06-18・N=5 で確認）。
**却下案**: `set_device` の引数を CLI/環境変数で受けるようコード改修する案 → 本体改変（CLAUDE.md 方針違反）かつ不要。公式スクリプトも環境変数方式。

### ADR-2: 検証は専用 expname（`dnerf/bouncingballs_feat007`）で行う
**採用**: 既存 feat-004〜006 成果物（`output/dnerf/bouncingballs/`、論文値一致を実証済み）を保護するため出力先を分離。
**却下案**: 既存 `output/dnerf/bouncingballs/` を再利用（学習をスキップし render/metrics のみ GPU≠0 で確認）→ FR-001（学習の GPU≠0 完走）が検証できない。3 経路すべてを GPU≠0 で通すため新規学習する。

### ADR-3: バックグラウンド実行＋ログファイル＋PID 記録
feat-004〜006 と同じ運用に PID 記録を追加。学習は約 10 分かかるためターミナルブロックを避け、完走シグナルで監視する。学習プロセスの `$!` を `train.pid` に保存し、FR-004 の対象 PID を一意に特定する（`pgrep` だと同名残プロセスを誤認しうる。Codex #01 中指摘）。

### ADR-4: ポート `P` は 6107 を既定候補とし、実行前に空き確認する
**採用**: 既定 6009 と公式スクリプトの常用ポート帯（6066〜6572）を避けた 6107 を既定候補とし、`ss -ltn` で空きを確認してから使う。使用中なら別値に変える。
**却下案1**: 既定 6009 のまま → 他者使用中だと `bind` 失敗で起動時クラッシュ（network_gui は例外処理なし）。**却下案2**: 6107 固定で空き確認なし → 共用サーバーでは衝突リスクが残る（Codex #01 中指摘）。空き確認を必須化する。

### ADR-5: 検証シーンは bouncingballs（feat-004〜006 と同一）
論文値・feat-006 実測（PSNR≈40.68）が既知で、GPU 変更による異常を指標で検知しやすい。データも取得済みで追加コスト無し。

### ADR-6: 運用ルールは CLAUDE.md（恒久）＋ design.md（根拠）の二層で記載
CLAUDE.md は日常運用で参照する簡潔な手順、design.md は背景（論理/物理デバイスの仕組み、固定箇所、ポート理由、デバイス順序）を保持する。FR-005 の 5 項目で両者を整合させる。

### ADR-7: 全実行に `CUDA_DEVICE_ORDER=PCI_BUS_ID` を付与する（Codex #01 高指摘）
**採用**: CUDA の既定デバイス順 `FASTEST_FIRST` は `nvidia-smi` の表示順（PCI バス順）と一致する保証がない。`nvidia-smi` で見た空き GPU の index をそのまま `CUDA_VISIBLE_DEVICES` に使うと、別 GPU に乗って FR-004 が破綻しうる。`CUDA_DEVICE_ORDER=PCI_BUS_ID` で両者を一致させ、加えて `gpu_uuid` で物理 GPU を二重照合する。
**却下案**: 同一機種（A100×7）だから `FASTEST_FIRST` でも index は一致するはず → 順序保証はなく、共用サーバーで誤った GPU に乗るリスクを残せない。明示する。

### ADR-8: 各経路を終了コードでゲートし、再実行時は出力先を削除する（Codex #02 高指摘）
**採用**: 固定出力先を使うため、学習失敗時に後続 render/metrics が古い成果物で「通って」しまい誤合格する危険がある。学習→render→metrics を終了コードでゲート（失敗で `exit 1`）し、再実行時は `output/dnerf/bouncingballs_feat007/` を `rm -rf` してから開始する。
**却下案**: expname にタイムスタンプを付け毎回新規パスにする → 検証手順・後片付けが煩雑。固定パス＋事前削除＋終了コードゲートで十分。

## 未検証事項（実装時に実地検証）

- U1: `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N`（N≠0）で 3 経路がいずれも exit 0 で完走し、成果物が生成されるか
- U2: 学習稼働中の `nvidia-smi` で `train.pid` の PID が物理 GPU index=N にのみ現れ、GPU0 を含む他 GPU に現れないか。手順 0 の `uuid` と `--query-compute-apps` の `gpu_uuid` が一致するか（FR-004 の核心）
- U3: `--port P`（既定 6107）での network_gui 初期化が正常か。ポート使用中時に `OSError` で起動時クラッシュする挙動（E4）の確認
- U4: 評価指標が feat-006 実測（PSNR≈40.68 / SSIM≈0.994 等）と整合するか（GPU 変更で数値が変わらないこと）
- U5: 本体 `*.py` に差分が出ていないこと（FR-006、`git status`）
