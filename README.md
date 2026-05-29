# git-regulation

團隊 Git 操作規範與分支策略文件集。

## 文件清單

| 文件 | 說明 |
|---|---|
| [Hit 分支策略與 GitHub 操作手冊](docs/hit-branch-strategy-and-github-manual.md) | `main` + `release` 雙長期分支的 **Sync-Staged Merge v2.0** 模型:命名規範、六大紀律、專案初始化、日常開發流程、正/反向同步與 Hotfix、角色權限、檢核清單。 |

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
