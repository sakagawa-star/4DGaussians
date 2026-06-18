# feat-007 Codex レビュー #01（初回）

- 実行日: 2026-06-18
- 対象: `requirements.md` / `design.md`（`README.md` も間接参照）
- codex: v0.140.0 / model `gpt-5.5` / sandbox read-only
- 指示: 初回レビュー（`docs/REVIEW_CRITERIA.md` 準拠、致命的な点のみ）
- 全過程ログ: `codex-01.full.log`
- 補足: codex のサンドボックスでは NVML/CUDA に接続できず実 GPU 列挙は未確認（`nvidia-smi` 失敗・`torch.cuda.device_count()`=0）。文書上の設計リスクとして指摘されている。

---

## レビュー結果（原文転記）

### 高

- `requirements.md:21` が `nvidia-smi` の GPU index をそのまま `CUDA_VISIBLE_DEVICES=N` に使う前提になっているが、CUDA の列挙順と `nvidia-smi` の表示順が常に一致する保証が文書化されていない。ここがズレると、空き GPU を選んだつもりで別 GPU に乗り、FR-004 の核心が破綻する。
  - **修正提案**: 全コマンドと運用ルールに `CUDA_DEVICE_ORDER=PCI_BUS_ID` を追加する。あわせて `nvidia-smi --query-gpu=index,pci.bus_id,uuid` の記録と、学習中の PID が選択 GPU index に紐づく確認を受け入れ基準に含める。

### 中

- `design.md:83` は「バックグラウンド＋ログ保存」と書く一方、実行例には `&` や PID 保存がなく、`design.md:132` の「train.log 起動時の PID」も実際には記録されない。`pgrep` 代替は同一 expname の残プロセスがあると誤認し得るため、FR-004 の証跡が曖昧になる。
  - **修正提案**: 学習起動時に `$!` を `/data/sakagawa/tmp/feat007-multi-gpu/train.pid` に保存し、`nvidia-smi` 出力と `wait` の終了コードもログ化する手順に差し替える。

- `--port 6107` 固定のまま、ポート使用中の扱いが未確定。実コードでは `network_gui.py:30` の `bind()` 失敗が捕捉されず、`train.py:427` で学習開始前にクラッシュする。共有サーバーでは単発実行でも他者と衝突し得る。
  - **修正提案**: 実行前に `6107` の空きを確認する手順を追加するか、ポート `P` を実行時に選ぶ変数として定義し、空き確認後に `--port P` で起動する設計に変更する。

### 低

- 致命的な点のみという依頼のため、低重要度の指摘はなし。

---

## 対応方針（Claude Code 記入）

3 件とも妥当。重要度「高」「中」は修正対象（`docs/REVIEW_CRITERIA.md`）。

| 指摘 | 対応 |
|------|------|
| 高: GPU index 不一致 | 全コマンド・運用ルールに `CUDA_DEVICE_ORDER=PCI_BUS_ID` を追加。手順 0 に `nvidia-smi --query-gpu=index,pci.bus_id,uuid` の記録を追加。FR-004 受け入れ基準に「選択 GPU index と PID の紐付け」を明記 |
| 中: PID 証跡 | 学習起動を `& echo $! > train.pid` 方式に変更し、PID を確実に記録。`nvidia-smi` 出力と終了コードをログ化。`pgrep` 依存を排す |
| 中: ポート衝突 | 手順 0 にポート空き確認（`ss -ltn`）を追加。bind 失敗時クラッシュ（`network_gui.py:30` は try/except なし）を E4 に確定情報として反映。ポートを変数 `P` 化 |

→ requirements.md / design.md / README.md を上記で修正後、`codex exec resume --last -o codex-02.result.md` で再レビュー。
