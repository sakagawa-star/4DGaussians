**指摘事項**

**高: `cgal` 有無が要求と設計で割れている。**  
[requirements.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:40) は `colmap[core,cuda,cgal]:x64-linux` を要求し、受け入れ基準も同じです。一方 [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:74) / [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:98) / ADR-3 は `colmap[core,cuda]` で CGAL 省略です。vcpkg の COLMAP port でも `cgal` は独立 feature なので、この差は実ビルド内容・依存量・合格判定を変えます。  
修正提案: 今の設計意図に合わせるなら requirements 側の [40](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:40), [44](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:44), [112](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/requirements.md:112) を `colmap[core,cuda]:x64-linux` に統一し、CGAL は「meshing が必要になった場合のみ再ビルド」に落とす。逆に CGAL 必須なら design/ADR/BACKLOG を全部 `core,cuda,cgal` に寄せる。

**中: FR-003 が既存 `open3d==0.19.0` を無視して 0.18.0 へ降格する手順になっている。**  
現 lock は [requirements.lock.txt](/data/sakagawa/4DGaussians/requirements.lock.txt:43) で `numpy==1.23.5`、[44](/data/sakagawa/4DGaussians/requirements.lock.txt:44) で `open3d==0.19.0` 済みです。TECH_STACK も open3d 0.19.0 を検証済み扱いです。一方 design は [design.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:78) と [177](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:177) で `open3d==0.18.0` を入れるため、「追加的」「既存環境非破壊」と矛盾します。実際、現 `.venv` では `open3d 0.19.0` import と `downsample_point.py` の IndexError 到達は確認できました。  
修正提案: open3d は「既存 0.19.0 を維持」に変更し、導入コマンドは `uv pip install "numpy==1.23.5" "open3d==0.19.0" "scikit-image==0.22.0"` か、より明確に scikit-image だけ追加する。requirements も「open3d 導入」ではなく「open3d 維持確認 + scikit-image 追加」に直す。

**中: CUDA 12.8 と driver 565.57.01 の前提が強く書かれすぎている。**  
[design.md](/data/sakagawa/4DGaussians/docs/issues/feat-008-colmap/design.md:321) は CUDA 12.8 が driver 565.57.01 で動作する前提ですが、NVIDIA release notes 上の CUDA 12.8 GA の対応 driver は 570.26+、ただし CUDA 12.x minor compatibility は 525以上で限定的に成立します。`native`/`sm_80` 指定なら成立余地はありますが、PTX や新 driver 機能に触ると `cudaErrorCallRequiresNewerDriver` 系で落ち得ます。  
修正提案: 「driver 565 では CUDA 12.x minor compatibility に依存する」と明記し、`CMAKE_CUDA_ARCHITECTURES=80` 固定を fallback ではなく推奨寄りにする。GPU 実行で driver 要求エラーが出た場合の fallback を CUDA 12.6/12.3/11.8 で再ビルド、または CPU SIFT 採用として明記する。

**低: なし。**  
今回の範囲では、低重要度の文言レベルは意図的に省略しました。

前提全体として、vcpkg ソースビルドへ切り替える判断自体は妥当です。一次情報でも COLMAP vcpkg port は 3.12.6、default feature は `gui`、`cuda`/`cgal` は feature として分離されています。参照: [vcpkg colmap port](https://raw.githubusercontent.com/microsoft/vcpkg/master/ports/colmap/vcpkg.json), [vcpkg feature docs](https://learn.microsoft.com/en-us/vcpkg/concepts/features), [NVIDIA CUDA release notes](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html), [CUDA minor compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html)。