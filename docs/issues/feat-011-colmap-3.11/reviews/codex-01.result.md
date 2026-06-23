> **メタ**: 対象=feat-011 `requirements.md` / `design.md` ／ session id=`019ef23b-baeb-7363-b435-ff54d39596c1` ／ 初回レビュー（2026-06-23）。
> **Claude Code の対応方針（codex-02 で高中低ゼロ収束を確認）**:
> - 高1（ラッパー末尾コロン）→ FR-002 ラッパーを `LD_LIBRARY_PATH` 空値分岐に修正（design ADR-4 に注記。既存 feat-008 ラッパーは温存）。
> - 高2（非改変と一時改変の矛盾）→ 一時改変手順を全削除し「GPU0 が空くまで待つ」に統一（requirements 非機能/FR-003・design §1.4.3/ADR-6）。
> - 高3（image_id↔name 偽陽性）→ FR-003 受け入れ基準と design §1.4.3 検証に整合チェック（検証コード）＋不整合時 B5 分岐を追加。
> - 中1（`set -e` なし）→ 実行を `bash -e colmap.sh` に統一（本体非改変＝実行方法のみ変更）。
>
> ---

**高**

1. [requirements.md:53](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/requirements.md:53) のラッパーが `LD_LIBRARY_PATH="$ROOT/lib:$LD_LIBRARY_PATH"` になっており、既存値が空だと末尾 `:` によりカレントディレクトリも共有ライブラリ探索対象になります。共用サーバー上では意図しない `.so` 読み込みやセキュリティ事故になり得ます。  
修正提案: 空値を分岐して末尾 `:` を出さない。

```bash
if [ -n "${LD_LIBRARY_PATH:-}" ]; then
  export LD_LIBRARY_PATH="$ROOT/lib:$LD_LIBRARY_PATH"
else
  export LD_LIBRARY_PATH="$ROOT/lib"
fi
```

2. 「4DGS 本体を1行も改変しない」としつつ、GPU0 使用中の対処で `colmap.sh:5` を一時改変し `git checkout colmap.sh` で戻す手順が入っています。これは [requirements.md:64](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/requirements.md:64) と [requirements.md:118](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/requirements.md:118)、[design.md:176](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/design.md:176) と [design.md:325](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/design.md:325) で矛盾しています。さらに `git checkout colmap.sh` は未コミットのユーザー変更を破棄し得ます。  
修正提案: feat-011 では「GPU0 が空くまで待つ」に統一し、一時改変手順を削除する。任意GPU対応は文書どおり feat-012 に送る。

3. feat-010 で database の `image_id` と `name` が名前順保証でないことが判明していますが、FR-003 の合格条件は rig エラーなし + `fused.ply` 点数 > 0 に留まっています。既知の `image_id/name` 不整合は [investigation.md:103](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/investigation.md:103) から [investigation.md:111](/data/sakagawa/4DGaussians/docs/issues/feat-010-dynerf/investigation.md:111) に記録済みです。この状態で 3.11 が rig 検証だけ回避すると、非空点群でも入力ポーズと画像対応が壊れた偽陽性を通す余地があります。対象文書の基準は [requirements.md:75](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/requirements.md:75) から [requirements.md:77](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/requirements.md:77)、[design.md:191](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/design.md:191) から [design.md:192](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/design.md:192)。  
修正提案: FR-003 に、`colmap/database.db` の `images.image_id,name` と `colmap/sparse_custom/images.txt` の `IMAGE_ID,NAME` が全20枚で一致することを必須条件として追加する。不一致なら 3.11 併設案では不合格とし、feat-010 の A' 方式へ戻す。

**中**

1. [colmap.sh](/data/sakagawa/4DGaussians/colmap.sh:15) は `set -e` なしで複数の `colmap` コマンドを続けます。一方、設計はログリダイレクト付きで `bash colmap.sh ...` を実行し、後から「各 colmap コマンドの exit」で判定するとしていますが、具体的に exit を捕捉できません: [design.md:183](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/design.md:183), [design.md:200](/data/sakagawa/4DGaussians/docs/issues/feat-011-colmap-3.11/design.md:200)。  
修正提案: 本体非改変のまま `bash -e colmap.sh ...` で実行する手順に変え、最初の失敗で停止させる。併せて `fused.ply` 点数確認は維持する。

**低**

致命的な点に絞ったため、低重要度の指摘はありません。