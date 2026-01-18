# Claude Code 项目安全约定

## 仓库信息

- **仓库**: `frostant/awesome-claude-skills`
- **用途**: Claude Skills 合集，fork 自 ComposioHQ
- **权限**: 允许完全读写操作

## 允许的 Git 操作

- `git add` / `git commit` - 允许
- `git push origin` - 允许（仅限本仓库）
- `git pull` / `git fetch` - 允许
- 创建分支 - 允许
- 创建 PR - 允许

## 禁止的操作

- `git push --force` 到 master/main 分支
- 修改 `.git/config` 中的 remote 指向其他仓库
- 向 `upstream` (ComposioHQ/awesome-claude-skills) push 任何内容

## Remote 配置

```
origin   -> https://github.com/frostant/awesome-claude-skills.git (可写)
upstream -> https://github.com/ComposioHQ/awesome-claude-skills.git (只读，用于同步)
```

## 敏感文件

以下文件不应提交到仓库：

- `.env` / `.env.local`
- 包含 API Key 的配置文件
- `**/secrets/**`
- `**/*credential*`

---

*此文件由 Claude Code 自动生成，用于定义项目级安全边界*
