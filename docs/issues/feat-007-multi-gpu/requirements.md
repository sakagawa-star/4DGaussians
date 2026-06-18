# feat-007 要求仕様書: マルチGPU運用対応（任意GPU選択）

## 1.1 プロジェクト概要

**何を作るのか**: 複数人で共用する GPU サーバー（A100-SXM4-40GB × 7）上で、4DGaussians の「学習 → レンダリング → 評価」の 3 経路を、**GPU0 以外の空いている任意の 1 枚の GPU を選んで**動かせる運用を確立する。具体的には (1) D-NeRF `bouncingballs` を GPU≠0 で 3 経路完走させて確認し、(2) 指定 GPU にのみ負荷が乗ることを `nvidia-smi` で確認し、(3) GPU 選択・並行実行時のポート衝突回避・分散学習非対応を運用ルールとして文書化する。

**なぜ作るのか**: 本マシンは複数人が共用しており、GPU0 が他者に使われている／空いていないことが常態である。Phase 0-5（D-NeRF 動作確認）は GPU0（`CUDA_VISIBLE_DEVICES=0`）固定で実施したが、共用環境では**空いている任意の GPU を選んで実行できる**必要がある。また Phase 7 以降の実シーン 3 系統（HyperNeRF/DyNeRF/multipleview）も同じ運用で動かすため、ここで運用方式を確定・文書化しておく。

**誰が使うのか**: 本リポジトリで 4DGaussians を学習・評価する開発者（共用サーバー利用者）。

**どこで使うのか**: 本マシン（A100-SXM4-40GB × 7、Driver 565.57.01、CUDA Toolkit 12.8 がグローバル。CUDA 拡張は CUDA 11.6 ビルド、torch 1.13.1+cu116、uv 管理の `.venv`）。

**前提（調査・実機検証で確定済み）**:
- GPU 固定箇所は `utils/general_utils.py:139`（`safe_state` 内、train/render が呼ぶ）と `metrics.py:116-117`（`__main__`）の 2 か所のみ。いずれも `torch.device("cuda:0")` を `set_device` する。render.py は加えて `device="cuda"` / `.cuda()` を使う。これら `cuda:0` / `cuda` は **`CUDA_VISIBLE_DEVICES` でマスクした後の論理デバイス先頭**を指すため、物理 GPU0 固定ではない。
- **GPU 番号の一致には `CUDA_DEVICE_ORDER=PCI_BUS_ID` が必要**。CUDA の既定デバイス順は `FASTEST_FIRST` であり、`nvidia-smi` の表示順（PCI バス順）と一致する保証がない。`nvidia-smi` で見た index をそのまま `CUDA_VISIBLE_DEVICES` に使うには、両者の順序を `CUDA_DEVICE_ORDER=PCI_BUS_ID` で揃える必要がある（Codex レビュー #01 高指摘への対応）。本案件は全コマンド・運用ルールでこれを必須とする。
- 本体に分散学習機構（`DataParallel` / `DistributedDataParallel` / `torch.distributed`）は存在しない（コード全文 grep で 0 件）。
- 公式スクリプト群（`scripts/*.sh`）は既に `export CUDA_VISIBLE_DEVICES=N&&python ...` で GPU 0〜3 を使い分けており、本案件の運用方式は公式の想定と一致する。
- 2026-06-18 の実機検証で `CUDA_VISIBLE_DEVICES=5` 指定下にプロセスが物理 GPU5 に乗ることを確認済み。
- `train.py` の network_gui ポート（既定 6009）は `network_gui.init`（`gaussian_renderer/network_gui.py:30`）で `listener.bind()` するが **try/except されていない**ため、使用中ポートを指定すると `OSError` で学習開始前にクラッシュする（コード確認済み）。共用サーバーでは単発実行でも他者と衝突しうるため、実行前のポート空き確認を必須とする（Codex レビュー #01 中指摘への対応）。

## 1.2 用語定義

- **物理 GPU 番号**: `nvidia-smi` が表示する GPU の通し番号（本マシンは 0〜6）。`CUDA_VISIBLE_DEVICES=N` の N に与える値。**`CUDA_DEVICE_ORDER=PCI_BUS_ID` を設定した前提で**、この index は CUDA の論理デバイス順と一致する。
- **論理デバイス**: PyTorch が見る GPU。`CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N` を設定すると、物理 GPU N が論理デバイス `cuda:0`（先頭）として見える。本体コードの `cuda:0` / `cuda` はこの論理先頭を指す。
- **CUDA_DEVICE_ORDER**: CUDA のデバイス列挙順を決める環境変数。既定 `FASTEST_FIRST`（計算能力順）/ `PCI_BUS_ID`（PCI バス順＝`nvidia-smi` index 順）。本案件は `PCI_BUS_ID` を必須とし、`nvidia-smi` の index と CUDA index を一致させる。
- **3 経路**: 学習（`train.py`）・レンダリング（`render.py`）・評価（`metrics.py`）の 3 つの実行経路。本プロジェクトの動作確認の単位。
- **任意 GPU 選択（マルチGPU運用）**: 複数 GPU の中から空いている 1 枚を実行ごとに選んで使う運用。**1 プロセス = 1 GPU**。複数 GPU を 1 ジョブで同時使用する分散学習とは異なる（本案件のスコープ外）。
- **負荷限定**: あるプロセスが、指定した物理 GPU N にのみ GPU メモリ・利用率を消費し、他 GPU には消費しない状態。
- **PID 証跡**: FR-004 の確認で、対象学習プロセスを一意に特定するための記録。学習起動時に `$!`（直近バックグラウンドプロセスの PID）を `train.pid` に保存し、`nvidia-smi` の Processes 欄の PID と照合する。
- **expname**: `train.py --expname` で与える実験名。出力先 `output/{expname}/` を決める。本案件は既存成果物（`output/dnerf/bouncingballs/`、feat-004〜006）を上書きしないため `dnerf/bouncingballs_feat007` を用いる。
- **network_gui ポート（P）**: `train.py --port`（既定 6009）。学習プロセスが可視化ビューア接続用に `network_gui.init` で開くポート。**学習経路のみ**が使用する（render/metrics は使わない）。本案件は実行前に空きを確認した値を変数 `P`（既定候補 6107）として与える。

## 1.3 機能要求一覧

### FR-001: 任意 GPU（N≠0）での学習完走

- `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N`（N は 0 以外の空き物理 GPU）を付けて `train.py` を D-NeRF `bouncingballs` で実行し、クラッシュせず完走する。
- 入力: 下記コマンド（実行は `.venv/bin/python` 経由、バックグラウンド＋ログ保存＋PID 記録。`P` は手順 0 で空きを確認したポート）
  - `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python train.py -s data/dnerf/bouncingballs --port P --expname "dnerf/bouncingballs_feat007" --configs arguments/dnerf/bouncingballs.py`
- 出力: 標準出力に coarse/fine の学習進捗、`output/dnerf/bouncingballs_feat007/point_cloud/iteration_{14000,20000}/point_cloud.ply` ほか成果物、`train.pid` に学習プロセスの PID
- 優先度: Must
- 受け入れ基準: プロセスが終了コード 0 で終了し、`output/dnerf/bouncingballs_feat007/point_cloud/iteration_20000/point_cloud.ply` が生成される（feat-004 の判定基準に準ずる）。`CUDA error: invalid device ordinal`・ポート `bind` 失敗（`OSError`）等の起動時エラーが出ないこと。

### FR-002: 任意 GPU（N≠0）でのレンダリング完走

- **FR-001（学習）の完走を終了コードで確認した後**、同一の N で `render.py` を実行し、完走する（学習失敗時は実行しない＝古い成果物での誤合格を防ぐ）。
- 入力: `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python render.py --model_path "output/dnerf/bouncingballs_feat007/" --skip_train --configs arguments/dnerf/bouncingballs.py`
- 出力: `output/dnerf/bouncingballs_feat007/test/ours_20000/{renders,gt}/` 各 20 枚、`video/ours_20000/renders/` と各 `video_rgb.mp4`
- 優先度: Must
- 受け入れ基準: プロセスが終了コード 0 で終了し、`test/ours_20000/renders/` に 20 枚の PNG が生成される（feat-005 の判定基準に準ずる）。render は network_gui を使わないためポート確認は不要。

### FR-003: 任意 GPU（N≠0）での評価完走

- **FR-002（レンダリング）の完走を確認した後**、FR-001/002 と同一の N で `metrics.py` を実行し、完走する。
- 入力: `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python metrics.py --model_paths output/dnerf/bouncingballs_feat007/`
- 出力: 標準出力に 6 指標（SSIM/PSNR/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM）、`output/dnerf/bouncingballs_feat007/{results,per_view}.json`
- 優先度: Must
- 受け入れ基準: プロセスが終了コード 0 で終了し、6 指標が数値出力され `results.json` が生成される（feat-006 の判定基準に準ずる）。指標値が feat-006 実測（PSNR≈40.68 等）と整合し、桁違いの乖離（PSNR<30dB 等）がないこと（健全性目安、Should 相当）。

### FR-004: 指定物理 GPU への負荷限定の確認

- FR-001 の学習が GPU N で稼働中に `nvidia-smi` を実行し、`train.pid` に記録した PID が**物理 GPU N にのみ**現れ、GPU0 を含む他 GPU には現れないことを確認する。
- 入力: 学習稼働中の `nvidia-smi`（Processes 欄）、`nvidia-smi --query-gpu=index,pci.bus_id,uuid,memory.used,utilization.gpu --format=csv`（GPU 一覧）、`train.pid`（対象 PID）
- 出力: 選択した GPU の `index` / `pci.bus_id` / `uuid` の記録、GPU N の行に当該 PID と GPU メモリ消費が表示され、他 GPU 行に当該 PID が無いことの記録（テキスト）
- 優先度: Must
- 受け入れ基準: `CUDA_DEVICE_ORDER=PCI_BUS_ID` 下で、`nvidia-smi` の Processes 欄の当該 PID（`train.pid` の値）が GPU index=N にのみ紐づく。他 GPU（特に GPU0）に当該 PID が無い。選択 GPU の `index`/`pci.bus_id`/`uuid` を記録し、`CUDA_VISIBLE_DEVICES=N` で選んだ物理 GPU と一致することを確認する。これにより「`cuda:0` ハードコードでも物理 GPU0 に漏れず、指定 GPU に乗る」ことを実証する。

### FR-005: マルチGPU運用ルールの文書化

- 共用サーバーでの任意 GPU 運用手順を `CLAUDE.md` の新節「マルチGPU運用ルール」と本案件 `design.md` に明文化する。
- 記載必須項目（design §1.4.5 と一致させる）:
  1. 実行前に `nvidia-smi --query-gpu=index,pci.bus_id,uuid,memory.used,utilization.gpu --format=csv` で空き GPU を確認し N（≠0、空き 1 枚）を選ぶ
  2. 実行コマンドには **`CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N`** を付ける（`nvidia-smi` index と CUDA index を一致させる）。同一ジョブの 3 経路（train/render/metrics）は同一 N を使う
  3. `cuda:0` / `cuda` は論理デバイス先頭＝マスク後の物理 GPU N を指すため**コード変更は不要**
  4. 分散学習（複数 GPU 同時使用）は本体非対応＝**1 プロセス 1 GPU**
  5. 学習実行前に network_gui ポート（既定 6009、本案件は `P`）の空きを `ss -ltn` で確認する。使用中なら別ポートを選ぶ（`bind` 失敗は `train.py` 起動時クラッシュになる）。並行実行時もポートを実行ごとに変える。render/metrics はポート不使用
- 優先度: Must
- 受け入れ基準: `CLAUDE.md` に上記 5 項目を含む節が存在し、`design.md` §1.4.5 と整合する。

### FR-006: 4DGaussians 本体コードの非改変

- 上記 FR-001〜005 をコード変更なしで達成する。`utils/general_utils.py` / `metrics.py` / `train.py` / `render.py` 等の本体は編集しない。
- 優先度: Must
- 受け入れ基準: 本案件のクローズ時点で 4DGaussians 本体（`*.py`）に差分が無い（`git status` / `git diff` で確認）。万一コード変更が不可避と判明した場合は中断し `investigation.md` に記録のうえユーザー承認を得る（CLAUDE.md 開発方針・feat-004 の前例）。

## 1.4 非機能要求

- **対応環境**: A100-SXM4-40GB × 7 のうち**任意の 1 枚**（N=0〜6、ただし検証は N≠0）。`.venv`（Python 3.10、torch 1.13.1+cu116）。CUDA 拡張は CUDA 11.6 ビルド済み。`CUDA_DEVICE_ORDER=PCI_BUS_ID` を全実行に付与する。
- **処理時間**: 動作確認が目的で時間は基準にしない。参考: 学習 約 10 分（feat-004 実績）、レンダリング 約 13 秒（feat-005）、評価 約 62 秒（feat-006）。GPU が変わっても同等を見込む。
- **VRAM**: 単一ジョブが 40GB に収まること（feat-004〜006 で実証済み。GPU 変更で消費量は変わらない）。
- **信頼性**: 指定 GPU が他者使用中で OOM／占有、またはポート使用中の場合は**異常終了とみなし自動リトライしない**。実行前 `nvidia-smi`（GPU 空き）と `ss -ltn`（ポート空き）で確認して回避する。原因は `investigation.md` に記録。
- **再現性**: GPU 番号 N・`CUDA_DEVICE_ORDER`・ポート P・出力先・ログ保存先・PID 保存先を design に明記し、`/clear` 後でも同一手順で再実行できること。N は実行時の空き状況で選ぶため可変だが、選び方を手順化する。
- **隔離性**: ログ・PID 等の一時生成物はリポジトリ外（`/data/sakagawa/tmp/feat007-multi-gpu/`）に置く。成果物は `output/dnerf/bouncingballs_feat007/`（`.gitignore` 管理外・非コミット）。

## 1.5 制約条件

- 4DGaussians 本体のコードは変更しない（FR-006、CLAUDE.md 開発方針）。
- 実行は `.venv/bin/python` 経由とする（システム Python を使わない）。
- すべての実行に `CUDA_DEVICE_ORDER=PCI_BUS_ID` を付ける（GPU 番号の一致のため）。
- 学習実行前に network_gui ポートの空きを確認する（`bind` 失敗は起動時クラッシュ）。
- 既存成果物 `output/dnerf/bouncingballs/`（feat-004〜006）を上書きしない。本案件は `output/dnerf/bouncingballs_feat007/` を使う。
- 各経路は終了コードでゲートし、前段が失敗したら後段に進まない。再実行時は古い成果物での誤合格を防ぐため、実行前に `output/dnerf/bouncingballs_feat007/` を削除する。
- データ `data/dnerf/bouncingballs/`（feat-003 取得済み）が存在することを前提とする。
- 新規ライブラリは追加しない。
- **分散学習（複数 GPU 同時使用）は対象外**（本体非対応、BACKLOG スコープ外）。本案件は「任意の 1 GPU 選択」に限定する。
- GPU 選択は `CUDA_VISIBLE_DEVICES`（＋`CUDA_DEVICE_ORDER`）で行う（コード内 `set_device` の引数化はしない＝本体改変回避）。

## 1.6 優先順位

1. FR-001・FR-002・FR-003（3 経路の GPU≠0 完走）= Must（MVP の中核）
2. FR-004（負荷限定の確認）= Must（「物理 GPU0 に漏れない」ことの実証が本案件の肝）
3. FR-005（運用ルールの文書化）= Must（後続 Phase の前提）
4. FR-006（コード非改変）= Must（開発方針）

MVP の範囲: FR-001〜004（GPU≠0 で 3 経路が完走し、指定 GPU にのみ負荷が乗ることを確認）。FR-005 はそれを運用知識として定着させる文書化、FR-006 は全 FR に課す制約。Should 相当の品質目安は FR-003 の指標健全性（桁違いの乖離がないこと）のみ。
