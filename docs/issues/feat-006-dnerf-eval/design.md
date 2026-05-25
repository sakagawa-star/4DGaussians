# feat-006 機能設計書: D-NeRF評価動作確認

## 1.1 対応要求マッピング

| 要求 | 内容 | 設計箇所 |
|------|------|----------|
| FR-001 | 評価コマンドの実行と6指標の算出 | §1.4.1 / §1.4.2 / §1.4.3 |
| FR-002 | results.json / per_view.json の生成 | §1.4.3 |
| FR-003 | 指標値の妥当性（論文報告値との整合） | §1.4.4 |
| FR-004 | 実行ログの保存 | §1.4.1 / §1.8 |
| （全 FR 横断） | エラーハンドリングと境界条件 | §1.4.5 |

本案件はコードを変更しない（既存 `metrics.py` の実行とその出力検証のみ）。設計の中心は「公式 `metrics.py` が feat-005 のレンダリング成果物に対しどのような経路で動き、何をどこへ出力するか」を解析し、判定基準（特に合格閾値）に落とすことにある。

## 1.2 システム構成

### 実行フロー（`metrics.py`）

```
python metrics.py --model_paths output/dnerf/bouncingballs/
  │
  ├─ __main__（metrics.py:115）
  │     ├─ torch.cuda.set_device(cuda:0)         ← cuda:0 を固定（CUDA_VISIBLE_DEVICES で物理GPUを選ぶ）
  │     ├─ ArgumentParser: --model_paths/-m（required, nargs="+"）
  │     └─ evaluate(args.model_paths)
  │
  └─ evaluate（metrics.py:36）
        for scene_dir in model_paths:        ← 本案件は ["output/dnerf/bouncingballs/"] の1件
          test_dir = scene_dir/"test"
          for method in os.listdir(test_dir): ← test直下を全列挙。現状 ["ours_20000"] のみ
            renders_dir = test_dir/method/"renders"
            gt_dir      = test_dir/method/"gt"
            readImages(renders_dir, gt_dir)  ← renders側のファイル名で両方を Image.open（§1.4.2）
            for idx in range(len(renders)):  ← 20枚。tqdm "Metric evaluation progress"
              ssim / psnr / lpips(vgg) / ms_ssim / lpips(alex) / D-SSIM を算出（§1.4.2）
            print 6指標の平均（metrics.py:81-86）
            full_dict[scene_dir][method]   = {6指標の平均}
            per_view_dict[scene_dir][method] = {指標: {画像名: 値}}
          results.json   ← full_dict[scene_dir] を dump（metrics.py:106）
          per_view.json  ← per_view_dict[scene_dir] を dump（metrics.py:108）
        例外時: "Unable to compute metrics for model <path>" を出力後 re-raise → 非0終了（§1.4.5 E1）
```

### ディレクトリ構成（入力と生成パス）

```
output/dnerf/bouncingballs/
├── test/ours_20000/                 # feat-005 成果物（入力）
│   ├── renders/00000.png .. 00019.png   # 推論画像 20 枚
│   └── gt/00000.png .. 00019.png        # 正解画像 20 枚
├── results.json                     # ← 本案件で生成（FR-002）。{"ours_20000": {6指標の平均}}
└── per_view.json                    # ← 本案件で生成（FR-002）。{"ours_20000": {指標: {画像名: 値×20}}}
```

注: `video/ours_20000/` は gt を持たないため `metrics.py` の評価対象外（`test/` のみ走査）。

## 1.3 技術スタック

- 実行: `.venv/bin/python`（Python 3.10、torch 1.13.1+cu116、torchvision 0.14.1+cu116）
- 画像読み込み: `PIL.Image.open` + `torchvision.transforms.functional.to_tensor`（0-1 正規化、metrics.py:29-32）
- 指標:
  - SSIM: `utils/loss_utils.py:36` `ssim`（window_size=11、標準 SSIM。0-1 値域テンソルを入力）
  - PSNR: `utils/image_utils.py:17` `psnr`（mask=None で標準 PSNR）
  - LPIPS: `lpipsPyTorch/`（`net_type='vgg'` と `'alex'` の2種。リポジトリ内実装）
  - MS-SSIM: `pytorch_msssim.ms_ssim`（導入済み、`data_range=1, size_average=True`）
  - D-SSIM: `(1 - MS-SSIM)/2`（metrics.py:79 で算出）
- **LPIPS 事前学習重み（DL 済み・キャッシュ済み）**: 調査（feat-006 着手時）に以下4ファイルを `~/.cache/torch/hub/checkpoints/` へ取得し、`lpips(vgg)`/`lpips(alex)` の推論成功を実機確認済み。**本番実行では再 DL されない**（オフライン懸念の解消）
  - `vgg16-397923af.pth`（528MB、download.pytorch.org、`models.vgg16` バックボーン）
  - `alexnet-owt-7be5be79.pth`（233MB、download.pytorch.org、`models.alexnet` バックボーン）
  - `vgg.pth`（7.12KB、raw.githubusercontent.com、LPIPS 線形層）
  - `alex.pth`（5.87KB、raw.githubusercontent.com、LPIPS 線形層）
  - 補足: `lpipsPyTorch/modules/networks.py` の `models.alexnet(True)` は torchvision 0.14.1 で **deprecation 警告**（`weights` を positional で渡す旧 API）を出すが、`AlexNet_Weights.IMAGENET1K_V1` 相当として正常動作する（実機確認済み・エラーではない）
- 新規ライブラリ追加なし（CLAUDE.md / requirements §1.5）

## 1.4 各機能の詳細設計

### 1.4.1 FR-001 / FR-004: 評価コマンドの実行（バックグラウンド＋ログ保存）

実行コマンド（公式 README Evaluation・BACKLOG feat-006 と同一の意図。引数は正式名 `--model_paths`）:

```bash
mkdir -p /data/sakagawa/tmp/feat006-dnerf-eval
CUDA_VISIBLE_DEVICES=0 .venv/bin/python metrics.py \
  --model_paths output/dnerf/bouncingballs/ \
  > /data/sakagawa/tmp/feat006-dnerf-eval/metrics.log 2>&1
```

- `CUDA_VISIBLE_DEVICES=0` で単一 GPU に固定（ADR-2）。`metrics.py` はコード内で `cuda:0` を固定使用するため、物理 GPU 選択は環境変数で行う
- バックグラウンド実行（`run_in_background: true`）＋ログをファイルに保存。完了は harness の通知で受ける（ADR-1）
- 引数名は **`--model_paths`** を使う（正式名。公式 README の単数 `--model_path` も argparse 前置一致で受理されるが、設計では曖昧さを避け正式名を採用。ADR-3）
- 末尾スラッシュ付き `output/dnerf/bouncingballs/` を渡す。出力 JSON は `scene_dir + "/results.json"` = `output/dnerf/bouncingballs//results.json`（二重スラッシュは OS が正規化、実体は `output/dnerf/bouncingballs/results.json`）

### 1.4.2 FR-001: 画像読み込みと6指標の算出

- `readImages`（metrics.py:24）は **`os.listdir(renders_dir)` のファイル名**で renders と gt の両方を `Image.open` する。feat-005 で renders/gt は同名（`00000.png`..`00019.png`）・各20枚を確認済み → 同名対応が成立
- 各画像は `tf.to_tensor(img).unsqueeze(0)[:, :3, :, :].cuda()` で **0-1 値域・3ch・(1,3,H,W)** テンソル化。`[:, :3, :, :]` で RGBA の場合もアルファを除外（D-NeRF の renders/gt は RGB だが安全側）
- **Pillow/numpy 型互換（feat-004 の懸念）は無関係**: `readImages` は `Image.open` + `tf.to_tensor` のみで、feat-004 で問題化した `Image.fromarray(..., "RGB")`（`dataset_readers.py:287`）を**通らない**（§1.4.5 E3）
- 各 idx で算出（metrics.py:74-79）:
  - `ssim(renders[idx], gts[idx])` — 標準 SSIM
  - `psnr(renders[idx], gts[idx])` — 標準 PSNR（mask 無し）
  - `lpips(renders[idx], gts[idx], net_type='vgg')` — LPIPS-vgg
  - `ms_ssim(renders[idx], gts[idx], data_range=1, size_average=True)` — MS-SSIM
  - `lpips(renders[idx], gts[idx], net_type='alex')` — LPIPS-alex
  - `(1 - ms_ssims[-1]) / 2` — D-SSIM
- 各指標は20枚分を `torch.tensor(...).mean()` で平均し、標準出力に `SSIM/PSNR/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM` を表示（metrics.py:81-86）
- 補足: LPIPS は画像ごとに `LPIPS(net_type).to(device)` を生成する（`lpipsPyTorch/__init__.py`）。20枚 × 2ネットでバックボーンを都度構築するが、重みはキャッシュ済みで DL は走らない（初回の1回のみバックボーン重みファイルを読む）

### 1.4.3 FR-002: 評価結果ファイルの生成

- `full_dict[scene_dir][method]`（method 別6指標平均）を `results.json` に、`per_view_dict[scene_dir][method]`（method 別・画像別6指標値）を `per_view.json` に `json.dump(..., indent=True)` で出力（metrics.py:106-109）
- 出力構造（現状 method は `ours_20000` の1つ）:
  - `results.json`: `{"ours_20000": {"SSIM": …, "PSNR": …, "LPIPS-vgg": …, "LPIPS-alex": …, "MS-SSIM": …, "D-SSIM": …}}`
  - `per_view.json`: `{"ours_20000": {"SSIM": {"00000.png": …, …×20}, "PSNR": {…}, …}}`
- 注: `full_dict`/`per_view_dict` のキーは `scene_dir`（= 渡した文字列 `output/dnerf/bouncingballs/`）だが、`json.dump` するのは `full_dict[scene_dir]`（method 階層）であり、JSON のトップレベルキーは method（`ours_20000`）になる

### 1.4.4 FR-003: 指標値の妥当性（合格閾値の確定）

**参照値**:
- 論文（CVPR 2024, arXiv:2310.08528, Table 6）の D-NeRF `bouncingballs`: **PSNR 40.62 / SSIM 0.9942 / LPIPS 0.0155**
- 論文の D-NeRF 合成シーン平均: PSNR 34.05 / SSIM 0.98 / LPIPS 0.02
- feat-004 学習時の実測: ITER14000 test PSNR=**39.84**（学習ログ）

**合格閾値（品質の健全性確認の目安。動作確認が主目的のため Should）**:

| 指標 | 閾値 | 根拠 |
|------|------|------|
| PSNR | ≥ 38 dB | 論文 40.62・feat-004 39.84。再現の seed/iteration 差を見て -2dB 程度を許容 |
| SSIM | ≥ 0.98 | 論文 0.9942。構造類似度は高安定 |
| LPIPS-vgg | ≤ 0.05 | 論文 LPIPS 0.0155（ネット種別は論文非明記）。vgg は alex より値が大きめに出る傾向のため緩めの上限 |
| LPIPS-alex | ≤ 0.03 | 論文 0.0155 に近い側。alex は LPIPS 既定ネット |
| MS-SSIM | ≥ 0.98 | SSIM と同水準を期待 |
| D-SSIM | ≤ 0.01 | `(1 - MS-SSIM)/2`。MS-SSIM ≥ 0.98 と整合 |

- これらは「健全性の目安」であり、**主判定は FR-001/002（6指標が算出され JSON が生成され非0終了）**。値が桁違いに乖離（特に PSNR < 30dB）した場合のみ、学習・レンダリング・評価のいずれかの異常を疑い `investigation.md` に記録して切り分ける
- 論文値に対し多少下振れしても（例: PSNR 38〜40dB）、動作確認としては合格とする（再現の細部差は想定内）

### 1.4.5 エラーハンドリングと境界条件

| ID | 事象 | 箇所 | 設計上の扱い |
|----|------|------|--------------|
| E1 | 評価対象画像の欠損（`test/`・`renders/`・`gt/` のいずれか不在、または renders にあって gt に無い同名画像） | metrics.py:52,65 / readImages | `os.listdir` または `Image.open` が `FileNotFoundError`。`evaluate` は `except` で捕捉し `Unable to compute metrics for model <path>` を出力後 `raise e` → 非0終了。feat-005 成果物（各20枚・同名）の存在を事前確認して回避（§1.6 前提条件） |
| E2 | LPIPS 重みのロード/DL 失敗 | lpipsPyTorch / torch.hub | 調査時に4重みを DL・キャッシュ済みのため本番では再 DL されず回避。万一キャッシュ破損時は再 DL（接続性は調査時に到達確認済み）。それも失敗する場合は停止・記録 |
| E3 | Pillow/numpy 由来の型エラー再発 | readImages | `Image.open` + `tf.to_tensor` のみで `Image.fromarray(...,"RGB")` を通らない（§1.4.2）。feat-004 の問題とは無関係。万一新規箇所で再発した場合はコード変更せず中断し investigation.md に記録（CLAUDE.md ステップ7） |
| E4 | 複数 method の併存（`ours_14000` 等が test 直下に追加された場合） | metrics.py:54 | `os.listdir(test_dir)` で全 method を評価し results.json に method 別で出力（仕様）。**現状 test 直下は `ours_20000` のみ**のため `ours_20000` 1件のみ評価される |
| E5 | VRAM 不足（OOM） | LPIPS 推論 | 1枚ずつ評価し、LPIPS バックボーン（vgg16/alexnet）も 40GB に対し小さい。学習・レンダリングが完走済みのため想定しない。発生時は停止・記録 |
| E6 | MS-SSIM の最小解像度制約（低解像度時） | metrics.py:77 / pytorch_msssim | `ms_ssim` は内部で5段ダウンサンプルするため最小辺が概ね 160px 以上必要（win_size=11）。**D-NeRF は 800×800 固定のため本案件は非該当**。他データセット（小解像度）へ展開する際の落とし穴として記録 |

自動リトライはしない（requirements §1.4 信頼性）。E1〜E6 のいずれかで完走しない場合、原因を investigation.md に記録し、ユーザー承認のうえ対処する。

## 1.5 状態遷移

```
[起動] → set_device(cuda:0) → argparse(--model_paths)
   → evaluate: scene_dir ループ（1件）
       → test_dir 走査 → method ループ（ours_20000 の1件）
           → readImages（renders/gt 各20枚 → テンソル化）
           → 20枚 × 6指標を算出 → 平均を print
       → results.json / per_view.json を dump
   → [正常終了（終了コード0）]

異常系: 画像欠損 / 重みロード失敗 / 型エラー → "Unable to compute metrics" → re-raise → [非0終了] → investigation.md 記録
```

`metrics.py` には明示的な完了メッセージはない。完走判定は「終了コード0で終了し、6指標が print され results.json/per_view.json が生成されていること」で行う（ログ末尾の `D-SSIM: ...` 出力と JSON 生成が完走シグナル）。

## 1.6 ファイル・ディレクトリ設計

**前提条件（実行前に確認）**:
- `output/dnerf/bouncingballs/test/ours_20000/renders/` に 20 枚の PNG（feat-005 で生成済み）
- `output/dnerf/bouncingballs/test/ours_20000/gt/` に renders と同名の 20 枚の PNG（feat-005 で生成済み）
- LPIPS 重み4ファイルが `~/.cache/torch/hub/checkpoints/` に存在（feat-006 調査時に DL 済み）

**生成物**: `output/dnerf/bouncingballs/results.json` と `output/dnerf/bouncingballs/per_view.json`。`output/` は `.gitignore` 管理外・非コミット。

## 1.7 インターフェース定義

- 入力引数:
  - `--model_paths`（`-m`、必須、`nargs="+"`）: 評価対象 model_path のリスト。本案件は `output/dnerf/bouncingballs/` の1件
- 入力ファイル: `{model_path}/test/{method}/{renders,gt}/*.png`
- 出力ファイル: `{model_path}/results.json`、`{model_path}/per_view.json`
- 標準出力: `Scene:` / `Method:` / `Metric evaluation progress`（tqdm）/ 6指標の平均値
- ログ: `/data/sakagawa/tmp/feat006-dnerf-eval/metrics.log`（標準出力＋標準エラー）

## 1.8 ログ・デバッグ設計

- ログ保存先: `/data/sakagawa/tmp/feat006-dnerf-eval/metrics.log`（リポジトリ外、隔離）
- 着目するログ行:
  - `Scene: output/dnerf/bouncingballs/`（評価対象 scene）
  - `Method: ours_20000`（評価対象 method）
  - `Metric evaluation progress: 100%|...| 20/20`（20枚の評価完走）
  - `Scene: ... SSIM/PSNR/LPIPS-vgg/LPIPS-alex/MS-SSIM/D-SSIM : <値>`（6指標の平均）
  - LPIPS 関連: 重みがキャッシュ済みなら `Downloading:` 行は出ない。`Using 'weights' as positional parameter(s) is deprecated`（torchvision 警告、無害）が出る
- 完走後の検証コマンド（手動テストで実行）:
  - `cat output/dnerf/bouncingballs/results.json`（6指標の平均値を確認）
  - `.venv/bin/python -c "import json; d=json.load(open('output/dnerf/bouncingballs/results.json')); print(d['ours_20000'])"`（JSON 妥当性・キー確認）
  - `.venv/bin/python -c "import json; d=json.load(open('output/dnerf/bouncingballs/per_view.json')); print(len(d['ours_20000']['PSNR']))"`（per-view 20 枚分を確認）

## 設計判断の記録（ADR簡易版）

### ADR-1: バックグラウンド実行＋ログファイルで起動する
feat-004/005 と同じ運用。LPIPS-vgg/alex の推論を含む20枚評価はレンダリングより時間がかかりうるため、ターミナルブロックを避け完走シグナルで監視する。

### ADR-2: `CUDA_VISIBLE_DEVICES=0` で単一 GPU に固定する
本マシンは A100×7。`metrics.py` はコード内で `cuda:0` を固定使用するため、物理 GPU は `CUDA_VISIBLE_DEVICES=0` で 0 番に固定する（再現性）。

### ADR-3: 引数は正式名 `--model_paths` を使う
公式 README は単数 `--model_path` と記載するが、`metrics.py:121` の定義は `--model_paths`/`-m`（`nargs="+"`）。argparse の前置一致で単数形も受理されるが、設計・コマンドでは曖昧さを避け正式名 `--model_paths` を採用する。

### ADR-4: LPIPS 重みは調査段階で DL・キャッシュ済みとする
オフライン懸念（torch hub キャッシュが空だった）を調査段階で解消。バックボーン（torchvision）と線形層（GitHub raw）の4重みを DL し、`lpips(vgg)`/`lpips(alex)` の推論成功を実機確認した。本番実行は再 DL なしで完走する見込み。これは「依存の健全性検証」でありコード変更ではない。

### ADR-5: 合格閾値は論文値・feat-004 実測から確定し、主判定は算出可否とする
BACKLOG の指示（合格閾値は着手時に論文値を参照して確定）に従い §1.4.4 で確定。ただし本案件の主目的は「評価が動作するか」であり、主判定は FR-001/002（6指標算出＋JSON 生成＋非0終了せず）。値の妥当性（FR-003）は健全性の目安とし、桁違いの乖離時のみ異常として切り分ける。

### ADR-6: コードは変更しない／新規ライブラリ追加なし
CLAUDE.md 開発方針。`metrics.py`/`lpipsPyTorch/`/`utils/*.py` は編集しない。`pytorch_msssim` は導入済み、LPIPS 重みは DL 済み。互換問題で変更が不可避な場合のみ、中断して investigation.md ＋ユーザー承認（feat-004 前例、E3）。

## 未検証事項（実装時に実地検証）

- U1: `metrics.py` が終了コード0で完走し、6指標がすべて数値で print されるか
- U2: `results.json` / `per_view.json` が生成され、JSON として妥当・`ours_20000` キーと20枚分の per-view を含むか
- U3: 各指標値が §1.4.4 の合格閾値を満たすか（特に PSNR ≥ 38dB。論文 40.62 / feat-004 39.84 との整合）
- U4: LPIPS 重みがキャッシュから読まれ再 DL されないか（ログに `Downloading:` 行が出ないこと）
- U5: 評価所要時間（参考値）と、LPIPS バックボーンの都度構築によるオーバーヘッドの実測
