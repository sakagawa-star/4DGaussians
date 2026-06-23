致命的な新規指摘はありません。高/中/低いずれも修正必須の指摘なしです。

前回指摘の確認:

- **高1: MPL 全削除は不可**  
  解消済みです。FR-005 train は [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/requirements.md:80) と [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:232) で `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp` が復活しています。FR-006 render も [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/requirements.md:93) と [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:271) で復活済みです。  
  根拠も [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:420) で `scene.gaussian_model -> scene.regulation.py:5` / `utils.scene_utils.py:4` の top-level matplotlib import として整理されています。

- **中2: CLAUDE.md 追記案③ の colmap.sh コマンドに PATH 前置**  
  解消済みです。[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:322) の CLAUDE.md 追記案と [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:369) のインターフェース定義が、どちらも `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh ...` になっています。

補足確認として、MPL 未設定でも `metrics.py` と `preprocess_dynerf.py` の依存 import は通り、`PATH` 前置で `python` が `.venv/bin/python` に解決されることも確認しました。

---

## Claude Code の対応方針（2026-06-22、再々レビュー codex-03）

**Codex 再帰レビュー収束**（codex-01 → 02 → 03）。重要度「高」「中」の指摘がゼロに収束した。

- codex-01: 高1（colmap.sh bare python）→ PATH 前置で対応 ／ 低2（MPL 帰属の事実誤認）→ MPL 全削除（過剰）
- codex-02: 高1再（MPL 全削除は不可＝train/render の matplotlib top-level import）→ MPL 復活 ／ 中2（CLAUDE案 PATH 抜け）→ 追加
- codex-03: **新規致命指摘なし・高/中ゼロ → 収束**

CLAUDE.md フローに従い、ここから**人（ユーザー）レビュー**へ進む。