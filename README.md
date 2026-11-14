# Obsidian 笔记库

个人知识库，通过 Git 在电脑和 Android 平板之间同步。

## 目录结构

| 目录 | 用途 |
| --- | --- |
| `00-索引` | 笔记入口与索引 |
| `10-日程安排` | Daily Notes 日志 |
| `20-学习` | Python / 嵌入式 / 编程与工具 |
| `30-读书笔记` | 读书笔记 |
| `40-书架` | 电子书（**PDF 不入库**，见下） |
| `60-文章` | 长文与剪藏 |
| `90-附件与示例` | 附件、Canvas / Bases 示例 |

## 什么进 Git，什么不进

- **进**：`.md` 笔记、`.canvas`、`.base`、图片附件、`.obsidian` 配置与插件
- **不进**：`*.pdf` 等大文件、旧备份目录、`.obsidian/workspace.json`（本机窗口状态）

规则见 [.gitignore](.gitignore)。PDF 只留在电脑本地，需要时用网盘单独同步。

## 同步约定

- 分支：`main`
- 行尾：不做转换（`core.autocrlf=false`），避免跨平台批量改写
- 冲突高发文件：`10-日程安排/` 下的日记。建议每次编辑前先 `pull`

## 常用命令

```bash
git pull --rebase      # 拉取（平板上先执行这个）
git add -A && git commit -m "update" && git push
```
