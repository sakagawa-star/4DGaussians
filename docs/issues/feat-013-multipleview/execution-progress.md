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
| CP2 | COLMAP+LLFF 前処理（FR-002） | sparse_・points3D_multipleview.ply・poses_bounds_multipleview.npy | ✅ 完了 2026-06-25 |
| CP3 | config作成＋学習（FR-003） | `output/multipleview/cut_roasted_beef/point_cloud/iteration_XXXXX/` | ✅ 完了 2026-06-25 |
| CP4 | レンダリング（FR-004、--skip_video） | `test/ours_XXXXX/{renders,gt}` | ✅ 完了 2026-06-25 |
| CP5 | 評価（FR-005） | results.json・per_view.json | ✅ 完了 2026-06-25 |
| CP6 | 文書化・クローズ（FR-006） | CLAUDE.md/BACKLOG/台帳 | ✅ 完了 2026-06-25 |

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

## CP2 結果（2026-06-25）

- COLMAP 3.11.1（PATH前置）で前処理を個別コマンド実行（`multipleviewprogress.sh` 非改変）。
- ① `extractimages.py`: 各camの frame_00001 → image1〜image20.jpg（20枚）。
- ② SfM（feature/matcher は **CPU固定** `use_gpu 0`、mapper）: exit0、約0.19分。**model_analyzer 健全性合格**: Registered images **20/20**、Points 4718、**Mean reprojection error 0.769px（<1.5px）**。
- ③ dense（GPU0、バックグラウンド＋ログ）: image_undistorter→patch_match_stereo（約10分）→stereo_fusion。`fused.ply`(10MB) 生成。
- ④ `downsample_point.py`: fused.ply → `points3D_multipleview.ply`（**36,357点**）。
- ⑤ LLFF `imgs2poses.py`（事前clone・py3互換OK・GPU不要）: `poses_bounds.npy` 生成 → `poses_bounds_multipleview.npy`（**shape=(20,17)**）。Cameras5/Images20/Points(4718,3)。
- 成果物検証通過: sparse_/{cameras,images}.bin（20画像・imageN.jpg形式）・points3D_multipleview.ply 36,357点・poses_bounds (20,17)。`colmap_tmp` 掃除済み。

## CP3 結果（2026-06-25）

- config `arguments/multipleview/cut_roasted_beef.py` を `default.py` 複製で作成（iterations=15000・coarse3000・kplanes [64,64,64,150]）。
- 学習（GPU0・port6017・MPL環境変数・バックグラウンド＋ログ）: exit0、`Training complete.`（15:33→15:56、約23分）。例外/nan なし。
- 学習中eval（健全）: [ITER3000] train PSNR 30.00 / [ITER7000] test 30.06・train 31.01 / **[ITER14000] test 31.24・train 32.68**。
- 成果物: `point_cloud/iteration_14000/` に point_cloud.ply(45M)・deformation.pth(9.9M)・deformation_table.pth・deformation_accum.pth。保存 iteration は feat-012 と同機構（`save_iterations` 最小14000、fine段階で到達。15000は未保存）。

## CP4 結果（2026-06-25）

- レンダリング（GPU0・MPL環境変数・`--skip_train --skip_video`）: exit0、約7秒（point nums 182050、FPS 12.85）。
- 成果物: `test/ours_14000/renders`=60枚・`gt`=60枚（20カメラ×3時刻、**ファイル名集合 完全一致** 00000〜00059.png）・`video_rgb.mp4`（testセットのモンタージュ）。`--skip_video` で spiral video（`video/`）は非生成を確認。
- ffmpeg macro_block_size 警告（1352×1014→1360×1024）は動画化のみ・正常。

## CP5 結果（2026-06-25）

- 評価（GPU0、バックグラウンド＋ログ）: exit0、`results.json`・`per_view.json` 生成。
- 6指標すべて FR-005 数値基準を満たす: **PSNR 32.38**（≥20）/ SSIM 0.9326（[0,1]）/ MS-SSIM 0.9658（[0,1]）/ D-SSIM 0.0171（[0,0.5]）/ LPIPS-vgg 0.1648（finite≥0）/ LPIPS-alex 0.0687（finite≥0）。
- multipleview は held-out カメラ無し＝学習視点再構成。学習中eval test 31.24 と整合（評価は20カメラ×3時刻＝全時刻平均で test より高め、健全）。

## CP6 結果（2026-06-25）

- ユーザー承認後（レンダ画像の目視確認＝floater/ソフトさは4DGS既知特性で正常と合意）に文書化クローズ実施。FR-006 完了:
  - `CLAUDE.md`「データセット」節に **multipleview 行**を追記（再編成手順・本体非改変の前処理コマンド・COLMAP使い分け・config・MPL注意・PSNR 32.38）。
  - `docs/BACKLOG.md` feat-013 を **Closed（2026-06-25）** に更新＋判定基準に実測合格を追記。
  - 本体「オリジナルコードの変更点」への追記なし（本体改変ゼロ）。追加は config `arguments/multipleview/cut_roasted_beef.py` のみ（通常運用・CP3でコミット済み）。`requirements.lock.txt` 再生成なし（新規ライブラリなし）。

## 完了サマリ

- **multipleview（多視点・カスタム）の前処理〜学習〜評価が 4DGaussians 環境で正常動作することを確認**（cut_roasted_beef 再編成、PSNR 32.38・6指標健全）。
- これでロードマップの**実シーン3系統（HyperNeRF〔feat-009〕・DyNeRF〔feat-012〕・multipleview〔feat-013〕）がすべて完了**。D-NeRF（合成）と合わせ実装済み4データセット系統すべてで学習〜評価が動くことを実証。
- 本体改変ゼロ達成。`multipleviewprogress.sh` 非改変（uv非互換の pip install/git clone と GPU0 ハードコードはラッパー/事前導入/環境変数で回避）。GPU引数化は将来 COLMAP 再実走案件へ残置。
