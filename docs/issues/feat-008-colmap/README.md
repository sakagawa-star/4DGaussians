# feat-008: COLMAP環境構築

## 概要

実シーン全系統（HyperNeRF / DyNeRF / multipleview）の前処理が依存する **COLMAP** を本マシン（conda無し・sudo無し・**公式 Linux バイナリ無し**）に **vcpkg でソースビルド**して導入し、前処理のPython依存（open3d / scikit-image）を整備して、小規模実データで COLMAP パイプライン（feature_extractor〜mapper）が完走することを確認する。

- **Phase**: 7（`docs/BACKLOG.md`）
- **依存**: feat-007（Closed）
- **Status**: In Progress（方式を vcpkg ソースビルドに変更し、ドキュメント再作成・レビュー中）

## 背景・課題

- 本マシンは **colmap 未インストール**（`CLAUDE.md` 実行環境表）。`convert.py` / `colmap.sh` / `multipleviewprogress.sh` はいずれも `colmap` CLI バイナリを前提とする。
- **sudo 不可**（パスワード必須）のため `apt install colmap` ができない。**conda / micromamba も未導入**。
- **COLMAP は公式に Linux バイナリ（AppImage 含む）を配布していない**（後述）。したがって「**どの方式で COLMAP を導入するか**」の選定が本案件の主眼であり、結論として **vcpkg ソースビルド**を採る。

## 導入方式（確定）

**vcpkg による COLMAP のソースビルド**を採用（2026-06-19 ユーザー承認）。`colmap[core,cuda]:x64-linux`（GUI〔Qt5〕除外・CUDA 有効）をビルドし、`~/.local/bin/colmap` ラッパーで PATH 解決する。

### 方式変更の経緯（重要）

- **初版（2026-06-19）は「公式 CUDA版 AppImage `colmap-3.12.6-linux-x86_64-cuda.AppImage`」を導入方式に確定していたが、これは誤りだった。**
- 実装着手時に GitHub Releases を実機確認した結果、**COLMAP は全リリース（3.3〜4.0.4）で Linux バイナリ（AppImage 含む）を一切配布していない**ことが判明（配布物は Windows/.zip と一部旧 Mac/.zip のみ）。初版が「公式 Linux AppImage の存在を確認」とした記録は事実に反していた（前セッションの調査誤り）。
- ユーザー判断により、導入方式を **vcpkg ソースビルド**に変更。`requirements.md` / `design.md` を全面改訂した。

### 選定根拠

- COLMAP 公式は Linux バイナリ非配布のため、prebuilt 利用は不可能。
- **vcpkg** は sudo 不要・ユーザー権限・**conda 非依存**で、依存（Boost/Ceres/glew/FLANN/OpenImageIO 等）を全自動ソースビルドする（CLAUDE.md の conda 回避方針に抵触しない）。
- `core` で default feature（gui=Qt5）を除外し、`cuda` で CUDA を有効化（A100 = sm_80 をビルド時自動検出）。ヘッドレス CLI 用途に最適。
- ビルド成果物・vcpkg ツリーは `/data`（15TB 空き）に置き、`~/.local/bin` で PATH 解決する。

### 却下した代替

- **公式 AppImage / prebuilt** ＝ 存在しない（上記）。
- **conda-forge（micromamba）** ＝ 確実だが回避中の conda エコシステムを持ち込む。
- **依存を手動で個別ソースビルド** ＝ 依存の依存（SuiteSparse/METIS/GMP/MPFR/Boost 等）まで sudo 無しで揃える工数・失敗リスクが過大。

## スコープ

- **対象**: vcpkg による `colmap` バイナリのソースビルド導入、前処理Python依存（open3d / scikit-image）の導入、小規模データでの feature_extractor〜mapper 完走確認（dense は Should の前倒し確認）。
- **対象外（後続案件）**: 各データセットの前処理スクリプト全体（`colmap.sh` / `multipleviewprogress.sh` / `*2colmap.py` / `database.py`）の実走行は feat-009/010/011 で扱う。本案件は COLMAP 単体の動作基盤を整える。

## 判定基準（BACKLOG より）

1. `colmap -h`（`--help` 相当）が通る。
2. `scripts/downsample_point.py`（open3d）が import エラーなく動く。
3. 小規模実データで COLMAP（feature_extractor〜mapper）が完走する。

## 関連環境事実（調査時点 2026-06-19、すべて実機検証済み）

| 項目 | 値 |
|------|-----|
| OS | Ubuntu 22.04.5 LTS / glibc 2.35 |
| colmap | 未導入。**公式 Linux バイナリ無し**（vcpkg でソースビルドする） |
| conda/micromamba | 未導入（持ち込まない） |
| sudo | 不可（パスワード必須） |
| ビルド系 | **cmake 無し・ninja 無し**（vcpkg が `downloads/tools/` に自動取得）。gcc/g++ **11.4.0** / make 4.3 / git 2.34.1 / curl あり |
| 既存ライブラリ | Eigen3 3.4.0 / Qt5 一式 / OpenGL・EGL（libGL/libEGL/libGLX_nvidia）あり。Boost・Ceres・CGAL・FreeImage/OpenImageIO・glog・gflags・FLANN・GLEW・SuiteSparse・METIS・GMP・MPFR は**無し**（vcpkg がビルド） |
| CUDA | `/usr/local/cuda-11.6`（V11.6.124）/ `/home/sakagawa/cuda/cuda-12.8`。**ビルドは 12.8 を採用**（gcc 11.4 互換。torch 拡張用の 11.6 とは独立） |
| GPU | A100-SXM4-40GB ×7、compute_cap 8.0（sm_80）。検証時の空き = index 0,1 |
| ディスプレイ | DISPLAY 未設定・Xvfb無し（**ヘッドレス**）。EGL/GL ライブラリは存在 |
| PATH書込先 | `~/.local/bin`（PATH上・書込可） |
| vcpkg 配置 | `/data/sakagawa/opt/vcpkg`（十数 GB に達し得るため /data） |
| scratch | `/data/sakagawa/tmp/feat008-colmap`（リポジトリ外） |

## ドキュメント

- `requirements.md` — 要求仕様書（REQUIREMENTS_STANDARD.md 準拠、vcpkg ソースビルド版）
- `design.md` — 機能設計書（DESIGN_STANDARD.md 準拠、vcpkg ソースビルド版・ADR-1〜5）
- `reviews/` — Codexレビュー出力
