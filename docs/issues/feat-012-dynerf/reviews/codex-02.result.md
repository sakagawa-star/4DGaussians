再レビュー結果です。

**高**
なし。前回の `metrics.py --model_path` 指摘は解消済みです。正式名 `--model_paths` に [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/requirements.md:78)、[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/design.md:187)、[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/design.md:247) で統一されています。`argparse` の前置一致で `--model_path` も受理される点も確認しました。

**中**
[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/design.md:244) の FR-001 インターフェース表だけ、まだ `points3D_downsample2.ply` を「点数（0<n≤40k）を検証」としており、前回指摘した「点数だけでは `fetchPly` 相当にならない」問題が要約表に残っています。詳細手順 [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/design.md:91) と要求 [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/requirements.md:45) は修正済みなので、表だけの残存です。

修正提案: line 244 を「`points3D_downsample2.ply` は `fetchPly` で読込確認し、点数 0<n≤40k と `x/y/z/nx/ny/nz/red/green/blue` の9フィールドを検証」に更新。

**低**
なし。

opencv-python の前回指摘は解消済みです。[requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/requirements.md:117)、[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/design.md:64) で DyNeRF ローダ依存として明記されています。

---

## Claude Code の対応方針（2026-06-24 / 再 codex-02）

- **中（1.7 表の PLY 確認残存）**: 指摘どおり、design 1.7 インターフェース表の FR-001 行だけ「点数のみ」が残っていた（§1.4.1 詳細手順・requirements FR-001 は codex-01 対応で修正済み）。表の該当行を **`fetchPly` 読込確認＋点数 0<n≤40k＋9フィールド検証**に更新した。
- **高・低**: なし。metrics（前置一致／正式名統一）・opencv-python（ローダ依存）の前回対応が確認された。
- → codex-03 で再確認する。