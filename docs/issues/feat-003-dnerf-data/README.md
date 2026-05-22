# feat-003: D-NeRFデータ準備

## 概要

4DGaussians の最初の動作確認に用いる **D-NeRF 合成シーン**のデータセットを取得し、本体コードが読み込める構成（`data/dnerf/{scene}/`）に配置する。

- 公式 README「Data Preparation / For synthetic scenes」に従い、[D-NeRF](https://github.com/albertpumarola/D-NeRF) の `data.zip` を [Dropbox](https://www.dropbox.com/s/0bf6fl0ye2vz3vr/data.zip?dl=0) からダウンロードする
- zip 展開後の構成を確認し、各シーンを `data/dnerf/{scene}/` 配下へ正規化配置する
- 学習・レンダリング・評価（feat-004 以降）の入力データを揃えることがゴール。**本案件では学習は実行しない**（feat-004 に委譲）

## ステータス

- **Closed**（2026-05-22 完了。FR-001〜005 達成。`data.zip`（246MB）取得→展開→`data/dnerf/` へ全8シーン配置→`bouncingballs` 整合性検証（train=150/test=20、全画像実在、Blender判定構成OK）→クリーンアップ。手動テスト合格）
- 依存: feat-002（CUDA拡張ビルド）= Closed
- 後続: feat-004（D-NeRF学習動作確認）

## 判定基準

主に使用する `bouncingballs` シーンについて、以下が**いずれも満たされる**こと:

1. `data/dnerf/bouncingballs/transforms_train.json` と `data/dnerf/bouncingballs/transforms_test.json` が存在する
2. 両 json に `camera_angle_x` と `frames`（各 frame に `file_path` / `time` / `transform_matrix`）が含まれる
3. 両 json の `frames` の**全要素**について `file_path` に対応する画像（`{file_path}.png`）が実在する（本体は全 frame の画像を開くため、先頭1件ではなく全件を確認する）
4. `data/dnerf/bouncingballs/` を `source_path` としたとき、本体のシーン判別ロジック（`scene/__init__.py:48`）が「Found transforms_train.json file, assuming Blender data set!」の分岐に入る構成になっている

> 学習の完走は feat-004 で確認する。本案件は「ファイルが揃い、Blender データセットとして認識される構成になっている」ことまでを判定範囲とする。

## 関連ドキュメント

- `requirements.md` — 要求仕様書（FR-001〜005）
- `design.md` — 機能設計書（コマンド列・エラーハンドリング・ADR）
- `README.md`（プロジェクトルート）— 公式 Data Preparation 手順
- `docs/BACKLOG.md` — ロードマップ（Phase 2）
