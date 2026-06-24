# feat-012 実行進捗台帳（クリーン再実行）

暴走復旧用の進捗ledger。各CP完了時に1行追記する。新セッションはこのファイルで「どこまで済んだか」を復元する。

## 確定パラメータ
- 対象シーン: `cut_roasted_beef`
- GPU: **N=0**（CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=0）
- ポート: **P=6009**
- scratch: `/data/sakagawa/tmp/feat012-dynerf`（`mplconfig/`・`tmp/`・各種 `*.log`）
- 本体改変: ゼロ（既存 train/render/metrics の実行のみ）

## チェックポイント計画と状態

| CP | 内容 | 壊れない成果物 | 状態 |
|----|------|----------------|------|
| CP0 | クリーン準備＋FR-001検証＋GPU/ポート確定 | 検証済み前提・確定パラメータ | ✅ 完了 2026-06-24 12:53 |
| CP1 | 学習 train.py 完走 | `output/dynerf/cut_roasted_beef/point_cloud/iteration_14000/` | ⏳ 未着手 |
| CP2 | レンダリング render.py 完走 | `test/ours_14000/{renders,gt}`・`video/ours_14000/` | ⏳ 未着手 |
| CP3 | 評価 metrics.py 完走＋健全性チェック | `results.json`・`per_view.json` | ⏳ 未着手 |
| CP4 | 文書化・クローズ（FR-005） | CLAUDE.md/BACKLOG更新・handover | ⏳ 未着手 |

## CP0 結果（2026-06-24 12:53）
- 残骸掃除: 旧 `output/dynerf/cut_roasted_beef/`（cfg_args のみ）削除、孤児 `pymp-*`（16個）削除。
- FR-001 合格: `poses_bounds.npy`=(20,17)、`cam*.mp4`=20本、全 `camXX/images`=各300枚（cam04欠番の20台）、`fetchPly`=37,361点・9フィールド（x/y/z,nx/ny/nz,red/green/blue）完備。
- GPU0 空き（1MiB）・port6009 空きを確認。matplotlib backend=agg を確認。

## 次アクション
- CP1（学習起動）: ユーザー確認後に着手。コマンドは `design.md` §1.7 FR-002（バックグラウンド＋`<scratch>/train.log` リダイレクト）。
