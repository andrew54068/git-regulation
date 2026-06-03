# git-regulation

團隊 Git 操作規範與分支策略文件集。

## 文件清單

| 文件 | 說明 |
|---|---|
| [Hit 分支策略與 GitHub 操作手冊](docs/hit-branch-strategy-and-github-manual.md) | `main` 單一長期主幹 + 依版本切出 `release/v<X.Y.Z>` 發佈分支的 **Release-Branch per Version v3.0** 模型(`main` 與 `release/*` 皆受保護):命名規範、五大紀律、專案初始化、日常開發流程、發佈、Hotfix 與跨版本 cherry-pick 傳播、角色權限、檢核清單。 |

## 結構

```
.
├── README.md
└── docs/
    ├── hit-branch-strategy-and-github-manual.md   # 規範本文
    └── images/                                    # 文件內嵌圖片(13 張)
```

## 來源

`docs/` 內的手冊記錄自私有 repo 的 GitHub Issue
[`hantoptw/Document-HitRD#1`](https://github.com/hantoptw/Document-HitRD/issues/1)(`[doc]-Hit 分支策略與 Git Hub 操作手冊`)。
內文逐字保留,原 `user-attachments` 圖片連結已下載至 `docs/images/` 並改為本地相對路徑,使文件可離線完整閱讀。
