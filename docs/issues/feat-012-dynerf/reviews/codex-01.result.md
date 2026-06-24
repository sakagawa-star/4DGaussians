**高**
- [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/requirements.md:78)、[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/design.md:185)、[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/design.md:245) の `metrics.py --model_path ...` は誤り。実装は [metrics.py](/data/sakagawa/4DGaussians/metrics.py:121) で `--model_paths` / `-m` 必須なので、このままだと argparse で即失敗し FR-004 が完走不能。
  修正提案: 3箇所を `CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=N .venv/bin/python metrics.py --model_paths output/dynerf/cut_roasted_beef/` に修正する。

**中**
- opencv-python を「本案件で不使用」「クリティカルパス外」としている記述が実装と不一致。該当は [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/requirements.md:117)、[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/design.md:64)。DyNeRF ローダは [scene/neural_3D_dataset_NDC.py](/data/sakagawa/4DGaussians/scene/neural_3D_dataset_NDC.py:6) で `cv2` を top-level import し、既抽出PNG流用時でも [scene/neural_3D_dataset_NDC.py](/data/sakagawa/4DGaussians/scene/neural_3D_dataset_NDC.py:323) で `cv2.VideoCapture` を生成するため、未導入なら train/render 前に落ちる。
  修正提案: 「前処理は実走しないが、DyNeRF ローダの import/runtime 依存として opencv-python は導入済み必須」と明記する。現 `.venv` では `cv2 4.13.0` を確認済み。

- PLY の FR-001 確認が `fetchPly` 相当になっていない。[requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/requirements.md:45)、[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-012-dynerf/design.md:91) は点数確認だけだが、実際の [fetchPly](/data/sakagawa/4DGaussians/scene/dataset_readers.py:124) は `x/y/z` に加えて `red/green/blue` と `nx/ny/nz` を読む。点数だけ通ったPLYでも FR-002 で KeyError になり得る。
  修正提案: FR-001 の確認を `fetchPly(".../points3D_downsample2.ply")` 実行、または vertex dtype に `x,y,z,nx,ny,nz,red,green,blue` が全てあることの assert に変更する。現成果物はこのフィールド一式を持つことを確認済み。

**低**
- 致命的な指摘はなし。

---

## Claude Code の対応方針（2026-06-24 / session 019ef754-8c40-7903-bb6d-8818815b99be / 初回 codex-01）

実コードで3点を裏取りした上で全件対応した。

- **高（metrics `--model_path`）**: 検証の結果、**「argparse で即失敗」は不正確**。`metrics.py:121` は `--model_paths`/`-m` のみを定義し、argparse の前置一致で `--model_path`（単数）も受理される（`parse_args(['--model_path','foo/'])` → `['foo/']` で実証。feat-006 の知見と一致）。ただし**正式名 `--model_paths` への統一は明確化として妥当**なため採用。requirements FR-004 / design §1.4.4 / design 1.7 の計3箇所を `--model_paths` に修正し、正式名と前置一致の注記を追加。
- **中（opencv-python）**: 指摘どおり。`scene/neural_3D_dataset_NDC.py:6` の `cv2` top-level import ＋ `:323` の `VideoCapture`（`:324` の if 外で既抽出時も実行）により**ローダ依存で必須**。requirements 制約条件 / design 1.3 / design システム構成表を「opencv-python は必須」に修正。
- **中（fetchPly フィールド）**: 指摘どおり。`fetchPly`（`dataset_readers.py:124-130`）は x/y/z・red/green/blue・nx/ny/nz の9フィールドを要求。FR-001 の確認を点数のみから **`fetchPly` 実行/9フィールド確認**に変更（requirements FR-001 / design §1.4.1）。成果物は9フィールド一式・37,361点を確認済み。

→ 高・中すべて対応。codex-02 で再レビューする。