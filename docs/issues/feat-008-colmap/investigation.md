# feat-008 investigation（実装中に判明した問題の調査記録）

本書は `docs/BUGFIX_STANDARD.md` に準拠する。実装フロー（CLAUDE.md feat ステップ6→問題発見）で判明した阻害要因を、再検討と再レビューが自己完結できる形で記録する。**上書きせずイテレーション番号を付けて追記する。**

---

## イテレーション1（2026-06-19）: vcpkg `lapack-reference` ビルド失敗（Fortran コンパイラ欠如）

### 現象
FR-001 実装で `vcpkg install 'colmap[core,cuda]:x64-linux'`（CUDA 11.6）を実行したところ、依存 `lapack-reference:x64-linux` のビルドで `BUILD_FAILED`。clone / bootstrap / vcpkg 本体生成 / cmake 自動取得は成功し、依存ビルド段で停止した（glog 等の一部依存はビルド成功済み）。

### エラー要点（build.log より）
```
-- The Fortran compiler identification is unknown
CMake Error at scripts/cmake/vcpkg_find_fortran.cmake:65 (message):
  Unable to find a Fortran compiler using 'CMakeDetermineFortranCompiler'.
  Please install one (e.g.  gfortran) and make it available on the PATH!
  ...
error: building lapack-reference:x64-linux failed with: BUILD_FAILED
```

### 再現情報
- ビルドログ: `/data/sakagawa/tmp/feat008-colmap/build.log`
- ビルドスクリプト: `/data/sakagawa/tmp/feat008-colmap/build_colmap.sh`
- vcpkg commit: `36393d1ca008d0086488a9041afac26ed3b8edb9`、vcpkg 本体版 `2026-05-27-d5b6777d`
- ビルド CUDA: 11.6（`/usr/local/cuda-11.6`, V11.6.124）

### 原因（実機確認済み）
1. **本機に Fortran コンパイラが一切無い**: `gfortran` / `gfortran-11` / `gfortran-12` / `gfortran-10` / `f95` / `flang` すべて不在。`gcc` も Fortran フロントエンド `f951` を持たない（`gcc -x f95` → `cannot execute 'f951'`）。
2. COLMAP は SuiteSparse(CHOLMOD) / Ceres 経由で **BLAS/LAPACK が必須**。vcpkg の Linux 向け LAPACK 提供ポート `lapack-reference` は **Reference LAPACK を Fortran ソースからビルド**するため Fortran 必須。
3. **設計・要求の盲点**: requirements.md「非機能要求」と design.md は C++ コンパイラ（gcc/g++ 11.4）のみを前提とし、Fortran 依存を見落としていた。

### 環境事実（再検討の前提）
| 項目 | 値 |
|------|-----|
| OS / arch | Ubuntu 22.04.5 LTS / amd64 |
| gcc-11 | 11.4.0-1ubuntu1~22.04.3（`f951` は未配備） |
| libgfortran ランタイム | `libgfortran.so.5` **存在**（/lib/x86_64-linux-gnu/） |
| system BLAS/LAPACK | `liblapack.so.3` / `libblas.so.3` **存在**（実行時 `.so.3` のみ。`-dev` の `.so` シンボリックリンク・ヘッダは未確認/恐らく無し） |
| sudo | パスワード要求（アカウントは sudoer。非対話 `sudo -n` は不可だが、人が手で打てば可） |
| apt-get | 存在（`apt-get download` で **非 root** でも .deb 取得可） |
| vcpkg 部分成果 | clone/bootstrap/cmake自動取得 完了、一部依存ビルド済み（キャッシュ）。Fortran 解決後の再ビルドは継続可能 |

### 候補対応（比較）
| 案 | 内容 | sudo | リスク | 備考 |
|----|------|------|--------|------|
| **A. userspace gfortran** | `apt-get download gfortran-11` → `dpkg-deb -x` で `/data/sakagawa/opt/gfortran` 等に展開、`PATH` と `GCC_EXEC_PREFIX` を設定して `gfortran` を可視化。libgfortran5・gcc-11 支援ライブラリは既存。 | 不要 | 中 | 要求「sudo不可」を厳守。Claude が完結可能。driver→`f951` の解決設定（`GCC_EXEC_PREFIX`）に検証が要る。導入 deb 版を記録すれば再現可。 |
| **B. sudo で gfortran 導入** | ユーザーが `sudo apt-get install -y gfortran` を実行。 | 必要 | 低 | 最簡・クリーン・標準的・再現性高。ただし要求の「sudo不可」を緩める判断が要る。システム変更がリポジトリ外に残る。 |
| C. system LAPACK 利用 / lapack-reference 回避 | system の `liblapack.so.3`/`libblas.so.3` を使い vcpkg のビルドを回避。 | 不要 | 高 | vcpkg は依存ポートを自前ビルドする方針で system ライブラリへの切替が困難。COLMAP の ceres/suitesparse が `lapack`/`blas` ポートを要求。custom triplet でも回避は煩雑・不確実。`.so.3` のみで dev リンク不可の懸念。 |
| （参考）D. ceres を suitesparse 無し・LAPACK off でビルド | LAPACK 依存自体を外す。 | 不要 | 高 | colmap vcpkg port が suitesparse 前提のため **port 改変**が必要。「本体・上流非改変」方針に反する寄りで、機能（CHOLMOD）も落ちる。 |

注: OpenBLAS をビルドして LAPACK を得る案も検討したが、**OpenBLAS の LAPACK 部分も Fortran を要する**ため本ブロッカーの回避にはならない。

### 推奨と未決事項
- **推奨**: 案 **A（userspace gfortran）**。要求の「sudo不可」を厳守でき、Claude の手で完結できる。案 B はより簡単だが「sudo不可」制約の緩和というプロジェクト方針判断を伴う。
- **未決（ユーザー判断待ち）**: A / B（または C/D 再考）のいずれを採るか。
- **決定後の手順**: design.md に Fortran ツールチェーン導入手順（§1.4.1 の前段に新ステップ、または新節）と ADR（例: ADR-6 Fortran 導入方式）を追記 → requirements.md の非機能要求/制約に Fortran 依存を反映 → **Codex 再レビュー（高・中ゼロ収束）** → 人レビュー → 実装再開（FR-001 から。vcpkg ツリーは保持済みでキャッシュ再利用）。

### 現時点のステータス
- 実装は **中断**（ユーザー指示「今は中断して再検討」2026-06-19）。
- FR-003（scikit-image 導入・open3d 0.19.0 維持）は**実装・検証済み**（本ブロッカーと独立。`.venv` 反映・`requirements.lock.txt` 再生成済み）。
- FR-004 用検証データ（south-building 先頭18枚）は scratch に取得済み。

---

## イテレーション2（2026-06-19）: 対応方針決定 = 案 B（sudo で gfortran 導入、管理者承認）

### 決定
GPUサーバー管理者より **`sudo` での gfortran インストール許可**を取得（2026-06-19、ユーザー報告）。候補 **B** を採用する。

### 根拠
- 案 A（userspace 展開）より簡単・クリーン・再現性が高く、gfortran ドライバ→`f951` 解決の脆さが無い。
- 管理者承認により、要求の「sudo 不可」を **gfortran 導入（ビルド必須のシステムツール）に限り例外的に緩和**する正当性が得られた。これ以外の依存は引き続き vcpkg のユーザー権限ビルドで完結する。
- gfortran は標準的なビルドツールチェーンであり、システム導入は妥当。

### 反映する文書変更
1. `requirements.md`: 非機能要求「権限」に gfortran のみ sudo 例外を明記、「ビルドツール」に Fortran 必須を追記、FR-001 に gfortran 前提を追記、制約条件を整合。
2. `design.md`: §1.3 に gfortran（system, sudo）追加、§1.4.1 に gfortran 導入ステップ追加、ADR-6（Fortran 導入方式）追加。
3. 本 investigation.md（本イテレーション）。

### 次の手順
1. 上記文書更新 → **Codex 再レビュー（高・中ゼロ収束）** → 人レビュー。
2. 実装再開: **ユーザーが `sudo apt-get install -y gfortran` を実行**（パスワード入力。Claude は sudo 不可）→ Claude が `vcpkg install 'colmap[core,cuda]:x64-linux'` を再開（vcpkg キャッシュ再利用で lapack-reference から続行）。
