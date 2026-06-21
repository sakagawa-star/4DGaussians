> **メタ（Claude Code 追記）**
> - 日付: 2026-06-19
> - 対象: `requirements.md` / `design.md`（vcpkg ソースビルド版）
> - 種別: 再レビュー（codex-05、`resume` で codex-04 セッション継続）
> - session id: `019edd77-433a-7f31-8af0-3b470a38a168`
> - 結果: **高0・中0・低0 → 収束**（codex-04 の高1・中2 を design.md 修正後に再確認し解消）
> - Claude 対応方針: 高・中がゼロに収束したため **Codex 再帰レビューを終了**し、人（ユーザー）レビューに進む（CLAUDE.md レビュー終了条件）。
> - 備考: codex-05 初回投入は CLI フラグ位置誤り（`-C` を `resume` の後に置いた）で exit 2。`codex exec -C <dir> resume ...` に修正して再投入し成功。

---

**レビュー結果**

高: なし  
中: なし  
低: なし

前回指摘3件は解消されています。

1. `cgal` 有無の不整合  
要求は [requirements.md:40](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:40) / [requirements.md:44](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:44)、設計は [design.md:74](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:74) / [design.md:98](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:98) / ADR [design.md:304](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:304) で `colmap[core,cuda]` に統一されています。`cgal` は想定外時の再ビルド分岐としてだけ残っており、主経路の不整合ではありません。

2. open3d 0.18.0 降格問題  
要求は [requirements.md:60](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:60) 以降で「open3d 0.19.0 維持 + scikit-image 新規導入」に変更済みです。設計も [design.md:167](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:167) / [design.md:173](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:173) / [design.md:178](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:178) で open3d を install コマンドから外しています。現 lock の `open3d==0.19.0` とも整合しています。

3. CUDA 12.8 第一候補問題  
要求は [requirements.md:41](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:41)、設計は [design.md:75](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:75) / [design.md:96](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:96)、ADR-4 は [design.md:320](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:320) 以降で、11.6 第一・12.8 代替に訂正済みです。12.8 の driver 565 制約も明記されています。

要求と設計の整合も、致命的な不整合は見つかりませんでした。vcpkg 側の前提も、現在の COLMAP port が 3.12.6、default feature が `gui`、`cuda` と `cgal` が別 feature である点と合っています。確認元: [vcpkg colmap port](https://raw.githubusercontent.com/microsoft/vcpkg/master/ports/colmap/vcpkg.json), [vcpkg features docs](https://learn.microsoft.com/en-us/vcpkg/concepts/features), [NVIDIA CUDA release notes](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html), [CUDA minor compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html)。