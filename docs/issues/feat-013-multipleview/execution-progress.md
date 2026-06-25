# feat-013 実行進捗台帳

暴走復旧用の進捗ledger。各CP完了時に1行追記する。新セッションはこのファイルで「どこまで済んだか」を復元する。

## 確定パラメータ

- 対象シーン: `cut_roasted_beef`（既存 DyNeRF データを multipleview 形式へ再編成）
- GPU: **N=0**（2026-06-25 時点で GPU0 空き 1MiB/0%。GPU2〜6 は 100% 占有、GPU1 は低負荷）。GPU を使うのは COLMAP dense と train/render/metrics
- ポート: **P=6017**（学習時のみ使用、空き確認済み）
- COLMAP: **3.11.1**（`/data/sakagawa/opt/colmap-3.11/bin`、PATH 前置）。feature/matcher は CPU 固定（`use_gpu 0`）、dense のみ GPU
- scratch: `/data/sakagawa/tmp/feat013-multipleview`（`logs/`・`pids/`・`mplconfig/`・`tmp/`・`LLFF/`・`reorganize.py`）
- 本体改変: ゼロ（`multipleviewprogress.sh` 非改変・個別コマンド実行。`extractimages.py`/`downsample_point.py` は reuse）。追加 config `arguments/multipleview/cut_roasted_beef.py`（通常運用）

## チェックポイント計画と状態

| CP | 内容 | 壊れない成果物 | 状態 |
|----|------|----------------|------|
| CP0 | 準備（GPU/ポート確定・LLFF clone・依存確認・scratch作成） | 確定パラメータ | ✅ 完了 2026-06-25 |
| CP1 | データ再編成（FR-001） | `data/multipleview/cut_roasted_beef/camNN/frame_XXXXX.jpg` | ✅ 完了 2026-06-25 |
| CP2 | COLMAP+LLFF 前処理（FR-002） | sparse_・points3D_multipleview.ply・poses_bounds_multipleview.npy | ⏳ 未着手 |
| CP3 | config作成＋学習（FR-003） | `output/multipleview/cut_roasted_beef/point_cloud/iteration_XXXXX/` | ⏳ 未着手 |
| CP4 | レンダリング（FR-004、--skip_video） | `test/ours_XXXXX/{renders,gt}` | ⏳ 未着手 |
| CP5 | 評価（FR-005） | results.json・per_view.json | ⏳ 未着手 |
| CP6 | 文書化・クローズ（FR-006） | CLAUDE.md/BACKLOG/台帳 | ⏳ 未着手 |

## CP0 結果（2026-06-25）

- 空きGPU確認: GPU0=1MiB/0%（採用 N=0）、GPU1=3392MiB/0%、GPU2〜6=100%占有。port6017 空き。
- scratch 配下に `logs/`・`pids/`・`mplconfig/`・`tmp/` 作成。
- 依存確認（venv）: skimage 0.22.0 / scipy 1.15.3 / imageio 2.37.3 / numpy 1.23.5 → LLFF `imgs2poses.py` の依存充足（`pip install` 不要、NFR-002 充足）。
- COLMAP 3.11.1（CUDA有効）確認。
- LLFF を `$SCRATCH/LLFF` に `--depth 1` clone。`imgs2poses.py`→`gen_poses` は `sparse/0` が在れば COLMAP 再実行せず読込のみ（前処理で生成するため問題なし）。

## CP1 結果（2026-06-25）

- 再編成スクリプト `$SCRATCH/reorganize.py`（本体外）で DyNeRF→multipleview 形式へ変換（バックグラウンド実行、約数分）。
- 結果: **cam01〜cam20 の連番20カメラ**（DyNeRF の cam04 欠番を詰めて解消）、**各300枚**（`frame_00001.jpg`〜`frame_00300.jpg`）、**真JPEG quality95**・1352×1014 RGB。総容量 1.7GB。
- FR-001 検証通過: cam01 存在・全カメラ同数（300）・命名連番（cam01..cam20）をスクリプト内 assert で確認。

## 次アクション

- **CP2（COLMAP+LLFF 前処理）**: design §3.2/§3.3 のコマンドを実行（COLMAP 3.11.1 PATH前置、feature/matcher CPU固定、dense は GPU0、LLFF imgs2poses）。dense はバックグラウンド＋ログ。完了後 §3.4 検証（model_analyzer 20/20登録・再投影誤差<1.5px・点数>0）。
