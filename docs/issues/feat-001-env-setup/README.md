# feat-001: uv環境構築・依存インストール

## 概要

uv（Python 3.10）でプロジェクト用の仮想環境を作成し、`TECH_STACK.md` の確定方針に従って
4DGaussians本体の実行に必要なPython依存パッケージ（torch 1.13.1+cu116 系 + `requirements.txt` の残り）を
インストールする。構築直後に `uv pip freeze > requirements.lock.txt` で厳密スナップショットを取得しgit管理する。

これは環境構築ロードマップ（`docs/BACKLOG.md`）の **Phase 0** に相当し、後続の
feat-002（CUDA拡張ビルド）以降の全工程の土台となる。

## ステータス

In Progress（要求仕様書・機能設計書の作成 → レビュー段階）

## スコープ

| 含む | 含まない（後続案件） |
|------|----------------------|
| `uv venv --python 3.10` による仮想環境作成 | サブモジュール初期化・CUDA拡張ビルド（feat-002） |
| torch/torchvision/torchaudio の cu116 導入 | D-NeRFデータ取得（feat-003） |
| `requirements.txt` 残り依存の導入 | 学習・レンダリング・評価（feat-004〜006） |
| `requirements.lock.txt` 生成・git管理 | |
| 判定（`torch.cuda.is_available()` / `torch.version.cuda`） | |

> 注: `git submodule update --init --recursive`（サブモジュール取得）は `BACKLOG.md` 上 feat-002 の先頭工程に
> 位置づけられているため、本案件のスコープから除外する。feat-001 の依存インストールはPyPI/PyTorch index 由来の
> パッケージのみを対象とし、ソースビルド対象（CUDA拡張）には触れない。

## 関連ドキュメント

- `requirements.md` — 要求仕様書（本案件）
- `design.md` — 機能設計書（本案件）
- `docs/TECH_STACK.md` — 技術スタック確定方針・uv依存管理ルール
- `docs/BACKLOG.md` — Phase 0（feat-001）

## 判定基準（サマリ）

仮想環境内の Python で以下が全て成立すること:

1. `import torch` が成功する
2. `torch.__version__` が `1.13.1+cu116` を返す
3. `torch.version.cuda` が `'11.6'` を返す
4. `torch.cuda.is_available()` が `True` を返す
5. `torch.cuda.get_device_name(0)` が A100 を示す文字列を返す
6. FR-002〜FR-003 で導入した主要パッケージ（torchvision, torchaudio, mmcv, lpips, plyfile, pytorch_msssim, open3d, imageio, matplotlib）が import 可能（mmcv は `from mmcv import Config` まで）。argparse は標準ライブラリを使うため PyPI からは導入しない（design ADR-6）
7. `requirements.lock.txt` が生成され、git管理下に置かれている

詳細・検証コマンドは `design.md` の「判定基準と検証手順」を参照。
