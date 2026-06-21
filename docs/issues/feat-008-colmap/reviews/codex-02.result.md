前回指摘のうち、matcher 独立フォールバックと `.venv/bin/python` 統一は解消済みです。疎再構成の偽陽性判定も大筋は直っていますが、TXT fallback の件数カウントにまだ致命的な穴があります。

**高**

1. `images.txt` の行数カウントで登録画像数を誤判定する  
[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:210) で、`model_converter` fallback 時に `images.txt` / `points3D.txt` のコメント行を除く行数で件数を数える、としています。COLMAP の `images.txt` は「1画像あたり2行」なので、非コメント行数をそのまま数えると登録画像数が2倍になります。登録画像1枚でも非コメント行が2行になり、`登録画像数 >= 2` を誤って満たします。これは前回の「疎再構成の偽陽性判定」が一部残っています。COLMAP 公式の出力形式でも `images.txt` は two lines per image とされています: https://colmap.github.io/format.html#images-txt

修正提案: `images.txt` は非コメント行数ではなく、ヘッダの `# Number of images: N` を読むか、2行1組として画像メタデータ行だけを数える手順に変更する。`points3D.txt` は1点1行なので非コメント行数でもよいが、同様に `# Number of points: N` を読む形に統一すると安全です。

**中**

なし。

**低**

なし。