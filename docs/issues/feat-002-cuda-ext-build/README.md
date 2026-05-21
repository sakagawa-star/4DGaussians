# feat-002: サブモジュール初期化・CUDA拡張ビルド

## 概要

4DGaussians が依存する2つのCUDA拡張サブモジュールをソースビルドし、Python から import できる状態にする。

- `submodules/depth-diff-gaussian-rasterization`（深度対応の微分可能ガウシアンラスタライザ、ingra14m fork）
- `submodules/simple-knn`（点群初期化用の近傍探索、INRIA）

両サブモジュールは現在**未初期化（空）**であり、`git submodule update --init --recursive` で取得したうえで、editable インストール（`uv pip install -e`）でビルドする。**ビルド時のみ `CUDA_HOME=/usr/local/cuda-11.6` を上書き**する（グローバルにアクティブな CUDA 12.8 のままだと、torch 1.13.1+cu116 と nvcc のメジャー版不一致（11 vs 12）でビルドエラーまたは重大な互換性警告が生じうる）。

## ステータス

- **In Progress**（2026-05-21 着手）
- 依存: feat-001（uv環境構築・依存インストール）= Closed
- 後続: feat-003（D-NeRFデータ準備）

## 判定基準

`.venv/bin/python` で以下が**いずれもエラーなく成功**する:

```bash
.venv/bin/python -c "import diff_gaussian_rasterization; print('rasterizer ok')"
.venv/bin/python -c "import simple_knn._C; print('simple_knn ok')"
```

> 上記は概要レベルの判定。シンボル import（`GaussianRasterizationSettings` 等）まで含む詳細な統合検証は `requirements.md` FR-004 / `design.md` §1.4.4 を正とする。

- import名は4DGS本体コードの実利用と一致:
  - `gaussian_renderer/__init__.py:14`: `from diff_gaussian_rasterization import GaussianRasterizationSettings, GaussianRasterizer`
  - `scene/gaussian_model.py:22`: `from simple_knn._C import distCUDA2`

## 関連ドキュメント

- `requirements.md` — 要求仕様書（FR-001〜005）
- `design.md` — 機能設計書（コマンド列・エラーハンドリング・ADR）
- `docs/TECH_STACK.md` — 技術スタック（CUDA整合方針）
- `docs/BACKLOG.md` — ロードマップ（Phase 1）
