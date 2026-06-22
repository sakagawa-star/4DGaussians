**前回指摘の確認**
- 高: `matplotlib backend/cache 未固定` は、train/render 本体コマンドでは解消されています。`MPLBACKEND` / `MPLCONFIGDIR` / `TMPDIR` が [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/requirements.md:55) と [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/design.md:136) に入っています。
- 中: `空評価・NaN 排除不足` は解消されています。FR-004 で `>0` とファイル名集合一致、FR-005 で有限値と `per_view` 件数確認が追加されています: [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/requirements.md:74), [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/requirements.md:85)。

**高**
- なし。

**中**
- Matplotlib の事前確認コマンドだけが環境変数なしの bare 実行になっています。  
  [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/requirements.md:59) と [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/design.md:133) の `.venv/bin/python -c "import matplotlib; ..."` は、`MPLBACKEND` / `MPLCONFIGDIR` / `TMPDIR` を付けていないため、前回問題と同じ cache 書込失敗で落ちるか、実行本番とは別の backend を確認してしまいます。  
  修正提案: 事前確認も本番と同じ環境で実行する形に統一してください。
  ```bash
  mkdir -p /data/sakagawa/tmp/feat009-hypernerf/{mplconfig,tmp}
  MPLBACKEND=Agg \
  MPLCONFIGDIR=/data/sakagawa/tmp/feat009-hypernerf/mplconfig \
  TMPDIR=/data/sakagawa/tmp/feat009-hypernerf/tmp \
  .venv/bin/python -c "import matplotlib; print(matplotlib.get_backend().lower())"
  ```
  合格条件は `agg` とする。

**低**
- 致命的な点のみ確認したため、低重要度の指摘はありません。

---

## Claude Code 対応方針（codex-02）

- 日付: 2026-06-21 / 対象: requirements.md, design.md / session id: 019ee928-df73-7da3-8841-b65a20d882de / 再レビュー（前回指摘の高・中は解消確認）
- **中（事前確認コマンドが bare 実行）**: 対応。FR-003 受け入れ基準（requirements.md）と design §1.4.3 step3 の matplotlib 事前確認を、本番と同一の `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp` 付き実行に統一し、`mkdir -p` を前段に明記。合格条件 `get_backend().lower()==agg`。
- 高・低: 指摘なし。