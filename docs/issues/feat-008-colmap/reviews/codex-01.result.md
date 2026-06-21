**高**

1. 疎再構成の合否判定が偽陽性になる  
[requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:70) と [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:199) が、`model_analyzer` 不在時に bin の存在・非空だけで「登録画像数 ≥ 2、3D点数 > 0」を代替可能としている。これは実際の登録画像数・点数を検証できず、空に近い失敗モデルを成功扱いする危険がある。

修正提案: 非空チェックを代替基準から削除する。`colmap model_analyzer` を必須にするか、`colmap model_converter --output_type TXT` で `images.txt` / `points3D.txt` を生成して、登録画像数と3D点数を数える手順を明記する。

**中**

2. GPU fallback が `feature_extractor` 片側に寄っており、`exhaustive_matcher` 失敗時の挙動が未定義  
FR-002 は `feature_extractor / exhaustive_matcher` を対象にしているが、受け入れ基準は主に feature 側のみで、設計も feature 成功後は matcher を GPU で実行する前提になっている [requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:48), [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:134)。ヘッドレス等で matcher だけ失敗した場合、CPU fallback すれば完走可能でも Must 要件が詰まる。

修正提案: feature と matcher を別々に fallback 判定する。feature が GPU 成功して matcher が GPU 失敗した場合は、`database.db` は保持したまま `exhaustive_matcher --SiftMatching.use_gpu 0` を再実行し、採用モードを `feature=gpu, matcher=cpu` のように記録する。

3. Python 依存のバージョンと検証インタプリタが固定されていない  
設計は `uv pip install open3d scikit-image` を未固定で解決させるとしており [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:156)、確認コマンドも途中から bare `python` になっている [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:158)。環境構築ドキュメントとしては、将来の resolver 結果やシェル状態で `.venv` 以外を検証する余地が残る。

修正提案: 実装前に採用版を決め、`open3d==...` / `scikit-image==...` / `numpy==1.23.5` の互換セットを明記する。検証コマンドはすべて `.venv/bin/python` または `uv pip --python .venv/bin/python ...` に統一する。

**低**

致命的な点に絞ったため、低重要度の指摘はありません。

外部確認に使った一次情報: [COLMAP 3.12.6 release](https://github.com/colmap/colmap/releases/tag/3.12.6), [COLMAP Datasets](https://demuc.de/colmap/datasets/)。