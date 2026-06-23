**高**

1. MPL 環境変数の全削除により、文書通りの train/render がこの環境で起動時クラッシュします。  
DyNeRF reader が `plot_camera_orientations` を呼ばない、という修正は正しいです。ただし train/render は reader 到達前の top-level import で matplotlib を読みます。具体的には [scene/gaussian_model.py](/data/sakagawa/4DGaussians/scene/gaussian_model.py:26) が [scene/regulation.py](/data/sakagawa/4DGaussians/scene/regulation.py:5) を import し、そこで `matplotlib.pyplot` が import されます。さらに train は [utils/scene_utils.py](/data/sakagawa/4DGaussians/utils/scene_utils.py:4) も import します。

実際に `MPLBACKEND` / `MPLCONFIGDIR` / `TMPDIR` を外して `import train` / `import render` すると、`Matplotlib requires access to a writable cache directory` で失敗しました。したがって [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/requirements.md:80), [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/requirements.md:92), [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/requirements.md:126) と [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:231), [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:263), [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:408) の「MPL 不要」は危険です。

修正案: reader 経路の説明は残しつつ、FR-005/FR-006 の train/render コマンドには以下を戻してください。metrics は Scene 非構築なので不要のままでよいです。

```bash
MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp \
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N ...
```

**中**

2. FR-008 の CLAUDE.md 追記案だけ、bare python 対策の `PATH` 前置が抜けています。  
FR-003 本体の実行コマンドは [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/requirements.md:59) と [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:357) で修正済みですが、[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/design.md:313) の CLAUDE.md 更新内容は `CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh <dir> llff N` のままです。これをそのまま運用記録へ転記すると、前回の bare `python` 即死が再発します。

修正案: CLAUDE.md 追記案も `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh <dir> llff N` に統一してください。

**前回指摘の確認**

- 高1 bare `python` 依存: FR-003 の実行コマンドと受け入れ基準に `PATH` 前置が入り、解消済みです。実際に `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" command -v python` は `.venv/bin/python` を返しました。
- 低2 `plot_camera_orientations` の事実誤認: reader 経路の記述は解消済みです。ただし上記の通り、matplotlib import 自体は別経路で発生するため、MPL 環境変数の全削除は不可です。

---

## Claude Code の対応方針（2026-06-22、再レビュー codex-02 への対応）

両指摘とも実コードで検証し妥当と確認。修正した。

### 高1（再）: MPL 全削除は不可（train/render の matplotlib top-level import）→ **対応済み（MPL 復活）**
- 検証: `scene/gaussian_model.py:26` が `from scene.regulation import ...` → `scene/regulation.py:5` で `import matplotlib.pyplot`。`utils/scene_utils.py:4` も同様。train/render は `gaussian_model` を import するため、**reader 到達前の top-level import で matplotlib がロードされる**。書込可能 `MPLCONFIGDIR` が無いと「Matplotlib requires access to a writable cache directory」で import 段階クラッシュしうることを確認。一方 `metrics.py`・`preprocess_dynerf.py` は matplotlib/gaussian_model を import しない（import 一覧で確認）。
- 修正: **train（FR-005）・render（FR-006）に `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp` を復活**。metrics/preprocess は不要のまま。MPL の根拠を「plot_camera_orientations の savefig」から「matplotlib の top-level import（フォントキャッシュ書込）」へ正確化。モジュール表・非機能要求・§1.4.5/1.4.6・§1.6 scratch・§1.7 表・ADR-6・各受け入れ基準/エラーハンドリングに反映。`output.png` は DyNeRF では非生成（plot 非呼び出し）の記述は維持。
- 教訓: codex-01 の「reader が plot を呼ばない→MPL 不要」を真に受け全削除したのが誤り。import 経路を見落としていた。codex-02 が top-level import を実証して訂正。

### 中2: CLAUDE.md 追記案③ の colmap.sh コマンドに PATH 前置欠落 → **対応済み（修正）**
- 修正: design.md §1.4.8 の CLAUDE.md 追記案③ を `PATH="/data/sakagawa/4DGaussians/.venv/bin:$PATH" CUDA_DEVICE_ORDER=PCI_BUS_ID bash colmap.sh <dir> llff N` に統一（運用記録への転記時の bare python 即死を防止）。

→ codex-03 で再レビュー（resume 継続）。