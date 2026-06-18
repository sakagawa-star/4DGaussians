**高**
- [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-007-multi-gpu/design.md:112) と [README.md](/data/sakagawa/4DGaussians/docs/issues/feat-007-multi-gpu/README.md:27) の実行手順が、学習失敗時に render/metrics へ進めてしまう。design は `wait` しているが失敗時に中断せず、README は `wait` 自体がないため、学習完了前に render/metrics が走る。固定出力先 `output/dnerf/bouncingballs_feat007/` に古い成果物が残っていると、FR-001 が失敗しても後続が既存成果物で通り、誤って合格判定する危険がある。  
  **修正提案**: 学習後に必ず終了コードをゲートする。例: `TRAIN_PID=$!; echo "$TRAIN_PID" > "$TMP/train.pid";`、FR-004 の `nvidia-smi` 記録後、`wait "$TRAIN_PID" || { echo "train failed"; exit 1; }` を入れ、その後に render/metrics を実行する。README にも同じ待機・中断手順を入れる。

**中**
- なし。

**低**
- なし。

**前回指摘3点の確認**
- `CUDA_DEVICE_ORDER=PCI_BUS_ID`: requirements/design/README 全体に反映済み。
- PID証跡: `train.pid` による照合へ更新済み。
- ポート衝突: `ss -ltn` による事前確認と `P` の可変化が反映済み。

残っている修正対象は上記の実行順序と終了コードゲートです。

---

## 対応方針（Claude Code 記入）

前回3点は解消確認。新規の高指摘1点（終了コードゲート）を反映済み（design / README / requirements）:
- 各経路を終了コードでゲート（`wait "$TRAIN_PID" || exit 1`、render/metrics も `|| exit 1`）
- 再実行時は出力先 `output/dnerf/bouncingballs_feat007/` を `rm -rf`
- requirements FR-002 / FR-003 に前段成功前提を明記、design に E7 / ADR-8 を追加

→ `codex-03` で再々レビュー（収束確認）。
