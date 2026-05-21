# issues

案件（feat / bug）ごとのドキュメントを格納するディレクトリ。詳細は `CLAUDE.md` の「機能追加フロー」「不具合修正フロー」「案件ディレクトリ構成」を参照。

## 構成

```
docs/issues/
└── {type}-{number}-{slug}/    # 例: feat-001-env-setup, bug-001-rasterizer-build-fail
    ├── README.md              # 概要、ステータス、再現手順
    ├── requirements.md        # 要求仕様書（機能追加時、REQUIREMENTS_STANDARD.md 準拠）
    ├── design.md              # 機能設計書（機能追加時、DESIGN_STANDARD.md 準拠）
    └── investigation.md       # 不具合の調査・修正計画（BUGFIX_STANDARD.md 準拠）
```

## ルール

- フォルダ名は英語で統一する
- 案件フォルダは完了後も削除・移動しない
- 各案件のステータスは `docs/BACKLOG.md` で管理する
