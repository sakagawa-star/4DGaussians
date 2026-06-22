**高**
- [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/requirements.md:99) と [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/design.md:49) で `matplotlib backend=agg` 前提になっていますが、実行コマンド側で `MPLBACKEND` / `MPLCONFIGDIR` を固定していません。HyperNeRF 読み込み時に [plot_camera_orientations](/data/sakagawa/4DGaussians/scene/dataset_readers.py:510) が必ず `matplotlib.pyplot` を import し、[output.png](/data/sakagawa/4DGaussians/scene/dataset_readers.py:534) を保存するため、ヘッドレス環境や Matplotlib cache dir が書けない環境では train/render がデータロード時にクラッシュします。  
  修正提案: scratch 配下に `mplconfig` と `tmp` を作り、FR-003/FR-004 のコマンドを `MPLBACKEND=Agg MPLCONFIGDIR=/data/sakagawa/tmp/feat009-hypernerf/mplconfig TMPDIR=/data/sakagawa/tmp/feat009-hypernerf/tmp CUDA_DEVICE_ORDER=...` の形に更新する。事前確認として `.venv/bin/python -c "import matplotlib; print(matplotlib.get_backend())"` も追加する。

**中**
- FR-004/FR-005 の合格条件が空評価・NaN を十分に排除していません。[requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/requirements.md:73) は renders/gt の「同数」だけで `>0` と同一ファイル名を要求しておらず、[requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-009-hypernerf/requirements.md:83) も 6 指標が有限値かを要求していません。`metrics.py` は画像数 0 の場合でも空テンソル平均で NaN を作り得る実装です。  
  修正提案: FR-004 に `renders` と `gt` の PNG 数が同数かつ `>0`、ファイル名集合が完全一致することを追加する。FR-005 に `results.json` の 6 指標すべてが `math.isfinite(value)` を満たし、`per_view.json` の件数が PNG 数と一致することを合格条件として追加する。

**低**
- 致命的な点のみ確認したため、低重要度の指摘は省略します。

---

## Claude Code 対応方針（codex-01）

- 日付: 2026-06-21 / 対象: requirements.md, design.md / session id: 019ee928-df73-7da3-8841-b65a20d882de / 初回レビュー
- **高（matplotlib backend/cache 未固定）**: 対応。`metrics.py` は Scene を構築せず matplotlib を経由しないことを確認（train/render のみ該当）。FR-003/FR-004 のコマンドに `MPLBACKEND=Agg MPLCONFIGDIR=<scratch>/mplconfig TMPDIR=<scratch>/tmp` を付与し、事前確認 `import matplotlib; get_backend()==agg` を追加。NFR・design §1.4.3/§1.4.4・§1.7・§1.6 に反映。
- **中（空評価・NaN 排除不足）**: 対応。FR-004 受け入れに「renders/gt が同数かつ>0かつファイル名集合一致」、FR-005 受け入れに「6指標すべて `math.isfinite`」「per_view 件数が renders PNG 数と一致」を追加。design §1.4.4 step4・§1.4.5 step4 に検証手順を追記。
- 低: 指摘なし。