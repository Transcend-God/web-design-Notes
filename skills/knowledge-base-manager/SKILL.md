---
name: knowledge-base-manager
description: "知识库管理技能，适用于 Joe 向 Joplin 本地知识库录入新知识点的场景。触发词包括：记录知识点、录入知识库、新知识、记录问题、补上、要（知识库上下文）。涵盖接收知识点、客观判断正确性、格式化写入本地笔记、git提交并推送到 GitHub。自动同步到 GitHub 仓库 Transcend-God/web-design-Notes。"
agent_created: true
---

# knowledge-base-manager — 知识库管理技能

## 用途

本技能用于 Joe 的 Joplin 本地知识库的日常维护，实现：
1. 接收用户输入的知识点（设计规范、项目流程、工作经验等）
2. 客观判断内容是否正确，必要时补充说明
3. 格式化写入本地 Markdown 笔记文件
4. 自动 git commit + push 到 GitHub 备份

---

## 关键配置

| 参数 | 值 |
|------|----|
| 本地笔记根目录 | `E:\notes` |
| 笔记存储子目录 | `E:\notes\default\` |
| GitHub 仓库 | `https://github.com/Transcend-God/web-design-Notes` |
| 分支 | `master` |
| Git 推送命令 | `git -c http.proxy= -c https.proxy= push`（需绕过本机 7897 代理） |
| GitHub Token | 存储在远程 URL 中（`git remote -v` 可查看），有效期约 1 年 |

---

## 工作流程

### Step 1：接收并判断知识点

收到用户输入的知识点后：
- 简要复述核心内容（1-2句话）
- 给出客观判断：**正确** / **需补充** / **建议修正**，并说明理由
- 如内容有缺口（例如只说了"不能做X"但未给替代方案），主动补全

### Step 2：生成笔记 ID

使用 Bash 命令生成唯一 ID（Unix 时间戳）：

```bash
date +%s
```

### Step 3：写入笔记文件

使用 PowerShell 写入，避免编码乱码（`joplin.sh edit` 对特殊字符有 bug）：

```powershell
powershell -Command "
\$content = @'
---
id: <ID>
title: 知识库：<标题>
created: <YYYY-MM-DD HH:mm>
tags: []
notebook: default
---
# <标题>

## 核心规范

<正文内容>
'@
Set-Content -Path 'E:\notes\default\<ID>.md' -Value \$content -Encoding UTF8
Write-Output 'Done'
"
```

**注意**：PowerShell here-string 中不能包含 `'@` 起始的单独行，如需包含单引号内容，改用变量拼接。

### Step 4：Git 提交并推送

```bash
cd "E:/notes" && git add -A && git commit -m "<简短描述>" && git -c http.proxy= -c https.proxy= push 2>&1
```

**注意**：`-c http.proxy= -c https.proxy=` 是绕过本机代理（127.0.0.1:7897）的必要参数，不能省略。

### Step 5：回复用户

完成后告知：
- 笔记 ID 和标题
- 客观判断结论
- 当前知识库总条数

---

## 笔记格式规范

```markdown
---
id: <数字ID>
title: 知识库：<主题简述>
created: <YYYY-MM-DD HH:mm>
tags: []
notebook: default
---

# <主题>

## 核心规范

<主要内容，用简洁的列表或表格>

## 注意事项（可选）

<补充说明，边界条件，反例>

## 适用场景（可选）

<说明在什么项目/情境下适用>
```

---

## 批量推送（已有积压时）

当用户说"上传到 GitHub"或积压了多条未推送的笔记时：

```bash
cd "E:/notes" && git status 2>&1
cd "E:/notes" && git log --oneline -5 2>&1
cd "E:/notes" && git add -A && git commit -m "<描述本批次内容>" && git -c http.proxy= -c https.proxy= push 2>&1
```

---

## 知识库缺口分析

当用户说"帮我补充"或"客观分析一下"时，读取现有笔记列表并按以下维度评估：
- 是否有没有对应规范的常见工作环节
- 现有规范之间是否有"孤岛"（前后缺乏联系）
- 是否有需要配套的反例或边界条件未记录

分析完后列出优先级（🔴强烈建议 / 🟡建议），等用户确认后再写入。

---

## 更新 README

当笔记总数超过上次 README 更新时的数量（每新增约 3-5 条），或用户明确要求时，更新 `E:\notes\README.md` 并推送。

README 模板见 `references/readme_template.md`。

---

## 已知问题和注意事项

- **编码问题**：`joplin.sh edit` 命令对包含竖线 `|`、反引号、中文引号的内容会乱码，必须用 PowerShell `Set-Content -Encoding UTF8` 写文件
- **代理干扰**：本机配置了 `127.0.0.1:7897` 代理，git push 必须加 `-c http.proxy= -c https.proxy=` 参数绕过
- **GitHub Token**：存在远程 URL 中（`ghp_` 开头 classic token，repo 权限）；旧的 `github_pat_` 开头的 token 已失效
- **.sock 文件报错**：`E:\notes` 下有 `.marscode/` 目录会导致 `git add -A` 失败，`.gitignore` 已排除此目录
