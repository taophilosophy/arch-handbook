# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库概览

这是 FreeBSD Architecture Handbook（FreeBSD 架构手册）的中文翻译项目。

源内容为 Markdown 文件，由 GitBook 平台自动构建和部署，无需本地构建步骤。

## 版本与流程

英文原版来源为 `en/` 目录下的 **AsciiDoc（`.adoc`）文件**，从上游仓库 https://github.com/freebsd/freebsd-doc 的 `documentation/content/en/books/arch-handbook/` 路径同步而来。

当前翻译基准上游 commit：`c9d98c0134864d8bc69006b92d34a3b22c940870`

### ADOC 文件结构（唯一权威源）

每个章节子目录下有 `_index.adoc`，根目录另有 `_index.adoc`、`book.adoc`、`parti.adoc`、`partii.adoc`、`partiii.adoc`。

- **`_index.adoc` 是英文原版的唯一权威源**，校对翻译时只读取 `_index.adoc`
- **禁止参考 `en/` 下的 `.po`（gettext）文件**——`.po` 是衍生产物，可能滞后或与 adoc 不一致，一律无视
- ADOC 语法要点：
  - `= 标题` = H1，`== 标题` = H2，`=== 标题` = H3，`==== 标题` = H4
  - `[[anchor-id]]` = 锚点
  - `man:命令[节号]` = man 页交叉引用
  - `[.filename]#path#` = 文件名标记
  - `[.programlisting]` = 代码块
  - `link:url[文本]` = 链接
  - `'''` = 水平分隔线
  - 文件头 `---` 之间是 YAML front matter（元数据，不翻译）

如果 `en/` 目录为空或缺少 adoc 文件，需要从上游同步：

1. 拉取 git 项目（注意：部分文件名含英文冒号，Windows 下无法直接 checkout，需下载 zip 压缩包）：https://github.com/freebsd/freebsd-doc
2. 路径：**documentation/content/en/books/arch-handbook/**
3. 将该路径下的 `.adoc` 文件及子目录完全复制到 `en/` 路径下
4. 更新本文件「当前翻译基准上游 commit」字段

需要进一步核查时，才参考 <https://docs.freebsd.org/en/books/arch-handbook/>。

## 内容架构

### 核心文件

- **`SUMMARY.md`** — 全书唯一数据源，定义完整目录结构和导航树。GitBook 用它来生成侧边栏。**第一行 `# Table of contents` 绝对不能变更**，否则 GitBook 同步失效。
- **`mu-lu.md`** — 由 `mulu.yml` CI 工作流从 `SUMMARY.md` 自动复制生成，**不要手动编辑**。

### 目录结构

- 中文章节为**扁平 `.md` 文件**，位于 `di-yi-bu-fen-nei-he/`（第 1–16 章）与 `di-er-bu-fen-fu-lu/`（参考文献）目录下，**不使用** `di-X-zhang/` 子目录
- 文件命名遵循拼音 slug 模式，如 `di-1-zhang-yin-dao-he-nei-he-chu-shi-hua.md`
- 顶层另有 `README.md`（首页）、`mu-lu.md`（目录）、`shang-biao.md`（商标）、`gai-shu.md`（概述）
- 全书共 **16 章 + 附录**，中文章号与英文原版 1:1 对应
- 英文原版三部分结构：
  - **Part I Kernel（第一部分 内核）**：第 1–8 章
  - **Part II Device Drivers（第二部分 设备驱动程序）**：第 9–16 章
  - **Part III Appendices（第三部分 附录）**：参考文献
- `en/` 目录有 17 个子目录（16 章 + `bibliography`），每个含 `_index.adoc`；另有根级 `_index.adoc`、`book.adoc`、`parti.adoc`、`partii.adoc`、`partiii.adoc`。`en/` 为英文原版归档，**不要修改或翻译其中的内容**

### 章节与 adoc 目录对照

| 章号 | 中文文件 | 英文 adoc 目录 |
| ---- | -------- | -------------- |
| 1 | di-1-zhang-yin-dao-he-nei-he-chu-shi-hua.md | en/boot/ |
| 2 | di-2-zhang-nei-he-zhong-de-suo.md | en/locking/ |
| 3 | di-3-zhang-nei-he-dui-xiang.md | en/kobj/ |
| 4 | di-4-zhang-jail-zi-xi-tong.md | en/jail/ |
| 5 | di-5-zhang-sysinit-kuang-jia.md | en/sysinit/ |
| 6 | di-6-zhang-trustedbsd-mac-kuang-jia.md | en/mac/ |
| 7 | di-7-zhang-xu-ni-nei-cun-xi-tong.md | en/vm/ |
| 8 | di-8-zhang-smpng-she-ji-wen-dang.md | en/smp/ |
| 9 | di-9-zhang-bian-xie-freebsd-she-bei-qu-dong-cheng-xu.md | en/driverbasics/ |
| 10 | di-10-zhang-isa-she-bei-qu-dong-cheng-xu.md | en/isa/ |
| 11 | di-11-zhang-pci-she-bei.md | en/pci/ |
| 12 | di-12-zhang-gong-gong-cun-qu-mo-xing-scsi-kong-zhi-qi.md | en/scsi/ |
| 13 | di-13-zhang-usb-she-bei.md | en/usb/ |
| 14 | di-14-zhang-newbus.md | en/newbus/ |
| 15 | di-15-zhang-sheng-yin-zi-xi-tong.md | en/sound/ |
| 16 | di-16-zhang-pc-ka.md | en/pccard/ |
| 附录 | di-er-bu-fen-fu-lu/can-kao-wen-xian.md | en/bibliography/ |

### 标题管理（关键约束）

`sync-headers.yml` 工作流会在每次 push 时自动将 `SUMMARY.md` 中的标题同步到对应 `.md` 文件的 H1。**这意味着直接编辑 `.md` 文件的 `# 标题` 会被 CI 覆盖。** 修改章节标题时，只改 `SUMMARY.md` 中对应的 `* [标题](路径)` 条目即可。

## CI/CD 工作流

所有工作流位于 `.github/workflows/`：

| 工作流 | 触发条件 | 说明 |
| ------ | -------- | ---- |
| `sync-headers.yml` | push | 从 SUMMARY.md 同步一级标题到各 .md 文件 |
| `mulu.yml` | SUMMARY.md 变更时 push | 复制 SUMMARY.md → mu-lu.md |
| `markdown-lint2.yml` | workflow_dispatch | markdownlint 检查，规则见 `.github/.markdownlint.json` |
| `md-padding.yml` | workflow_dispatch | 自动在 CJK 与英文/数字间添加空格 |
| `AutoCorrect.yml` | --- | 自动修正常见中文笔误与格式问题 |
| `check-images.yml` | push（`.gitbook/assets/**`）+ workflow_dispatch | 检查 Markdown 图片引用，脚本 `.github/scripts/check_images.py` |
| `file-name-check.yml` | --- | 检查 SUMMARY.md 中引用的文件是否存在 |
| `create-pdf.yml` | --- | 导出 PDF |

## 编写规范

### 格式

- **命令行前缀：** `#` 表示 root 权限，`$` 表示普通用户。不要使用 `sudo`。
- **提示块：** tip/important/note/warning/caution 使用 `>` 缩进引用，关键词 **加粗**。
- **代码块：** 无法判断的，再使用 ` ```sh ` 兜底，禁止使用 text 作为代码块标记，不窜改既有的 ` ```ini ` 标记。
- **表格：** 一律居中。
- **禁止 HTML：** 本项目不支持任何 HTML 语法。
- **文件命名：** 使用拼音 slug，文件名中不得包含空格、中文字符或英文冒号 `:`，必须兼容 Windows 操作系统对文件名的要求。
- **路径与 IP：** 全书正文中的路径（带 `/` 或 `\` 的，如 `/etc/rc.conf`、`/usr/local/etc/`）和 IP 地址一律使用 **加粗**，不使用反引号 `` ` `` 包裹；不要混合使用 `*` 和 `` ` ``（如 `*`path`*` 是错误的）。
- **命令、选项、参数：** 命令、选项、可调选项、可调参数等格式使用 `行间代码`（反引号）包裹。注意：不带选项和参数的裸命令，如 mv cp 等，一般不加反引号，不进行包裹。
- **转义字符：** 除非是命令、选项和参数，否则含转义字符 `\` 的元素一律使用 **加粗** 包裹整个元素，不在正文中直接使用转义字符（如写 **PROTO_TYPE** 而非 `PROTO\_TYPE` 在正文里裸露）。逐个手动修改，禁止批量替换。
- **避免滥用"已"字：** 如"XX 已新增"应改为"新增 XX"；"已修复"应改为"修复完成"。禁止机械替换。
- **禁止篡改：** 不要篡改软件版本号、用户名、带圈数字（如 ①②③ 等），确保其位置、数量和英语原文一致。
- **避免章节交叉引用：** 正文不要出现具体的章节交叉引用（如"参见第 5 章"），改用语义化链接。
- **代码块注释翻译：** 代码块（` ``` ` 围栏）内的英文注释必须翻译为中文（如 shell 注释 `# This is a comment` → `# 这是一个注释`）。只翻译注释部分，不修改实际命令或代码。保持代码结构和格式不变。
- **fstab 不翻译：** fstab 文件表头（如 `# Device Mountpoint FStype Options Dump Pass#`）及其相关内容保持英文原样，不翻译。

### 术语

- `Ports` 保持英文不翻译，且保持首字母大写（注意区分真正的"端口"）禁止机械替换。
- "Jail" 保持英文（不翻译为"监狱"、"监牢"）禁止机械替换。
- "拷贝" → "复制"，"壳/外壳" → "shell"。禁止机械替换。
- 第二人称一律使用"你"而非"您"

### man 页交叉引用格式

英文原版 AsciiDoc 的 `man:命令[节号]` 在中文 Markdown 中转换为：

```markdown
[命令(节号)](https://man.freebsd.org/cgi/man.cgi?query=命令&sektion=节号&format=html)
```

示例：`man:boot[8]` → `[boot(8)](https://man.freebsd.org/cgi/man.cgi?query=boot&sektion=8&format=html)`

### ADOC 元素到 Markdown 的转换

| ADOC 语法 | Markdown 写法 |
| --------- | ------------- |
| `= 标题` | `# 标题`（H1，由 CI 同步，不手改） |
| `== 标题` | `## 标题` |
| `=== 标题` | `### 标题` |
| `==== 标题` | `#### 标题` |
| `[.filename]#path#` | **path**（加粗） |
| `[.programlisting]` 代码块 | ` ```语言 ` 围栏代码块 |
| `man:cmd[N]` | `[cmd(N)](https://man.freebsd.org/...)` |
| `link:url[文本]` | `[文本](url)` |
| `'''` | `---`（或省略） |
| `[NOTE]` / `[TIP]` / `[WARNING]` | `> **注意**` / `> **提示**` / `> **警告**` |
| `[[anchor]]` | 省略（Markdown 无锚点语法） |

### CJK 空格

中英文、中文与数字之间必须加半角空格。文件和命令名用 `` ` `` 括起来。`md-padding.yml` 和 `AutoCorrect.yml` 工作流会自动检测修复。

### 图片

在正文中插入图片，使用 markdown 格式。

### 翻译流程

1. 参考官方英文手册原文（`en/<chapter>/_index.adoc`）进行人工校对
2. 提交 PR 到 main 分支

### 翻译校对工作流程（Claude Code）

对已翻译章节进行质量审校时，参考以下步骤：

1. **获取原文**：读取 `en/<chapter>/_index.adoc`（**唯一权威源**）。`<chapter>` 为章节子目录名，如 `boot`、`locking`、`kobj`、`jail`、`sysinit`、`mac`、`vm`、`smp`、`driverbasics`、`isa`、`pci`、`scsi`、`usb`、`newbus`、`sound`、`pccard`、`bibliography`。**禁止参考 `en/` 下的 `.po` 文件**。

2. **逐句对照**：将中文翻译与英文原文逐句比对，重点检查以下问题类别：

   | 问题类型 | 示例 |
   | -------- | ---- |
   | 事实性错误 | "large parts" 误译为"三个文件" |
   | 漏译 | 英文原版有但中文缺失的句子 |
   | 机翻腔/表达生硬 | "在阅读本章后，你将会收获" → "通过阅读本章，你将了解" |
   | 用词不当 | "独家优势"应为"主要优势"（原文 "particular strengths"） |
   | 版本过时 | 15.0-CURRENT 未同步至 16.0-CURRENT |
   | 语法/文字错误 | 重复字词、"非常地"应为"非常" |
   | 欧化汉语、倒装句、后置句、偷换主语、不必要的被动句 | 自行联网制定标准，避免语法错误和非地道汉语表述 |

3. **提交修改**：
   - 基于 `main` 创建分支 `fix/arch-handbook-chX-translation`（X 为章号）
   - 仅提交修改的 `.md` 文件，不要包含无关文件
   - 提交信息格式：`fix: 修正第 X 章翻译错误及同步英文原版更新`
   - 用 `gh pr create` 创建 PR，目标分支为 `main`
   - PR 描述中列出每处修改及原因

4. **注意事项**：
   - 不要修改 `# 标题`——会被 `sync-headers.yml` CI 覆盖
   - 保持术语一致性
   - 英文原文可能比翻译版本更新，版本号差异（如 15.0 vs 16.0）应以英文原版为准
   - 添加新内容（如英文原版新增的段落）时，确保翻译风格与上下文一致
   - 禁止篡改带圈数字，如 ①②③ 等，确保其位置、数量和英语原文一致
   - 禁止篡改代码块（包括 shell 命令、配置文件内容、ini 段等），仅修改正文翻译
   - 禁止篡改用户名、邮箱、URL、IP 地址等技术参数
   - 禁止篡改软件版本号
   - 以 `en/<chapter>/_index.adoc` 为**唯一权威源**；禁止以中文版已有翻译作为修改依据，避免错误沿袭

5. **逐句校对递归工作流**（核心约束）：
   - **逐行逐句子**遍历内容，检查逻辑一致性，联网复核后修改
   - **三轮 + 一轮复查**：完整执行三轮逐句校对，然后严格、完整地复查一轮；若仍有问题，继续执行三轮，递归直至无问题
   - **逐个修改**：修改时禁止机械修改、禁止批量；必须逐个手动修改，禁止使用软件工具批量替换或批量逐个复核
   - **修改前后复核**：每次修改前确认问题真实存在、修改后确认改动准确且未引入新错误
   - **不复查旧错**：不得复查以前已经完成三轮的错误；每轮查询内容必须是新的（若与之前的复查重复则跳过、不改，只看是否有新增内容）
   - **联网复查**：对于不存在的实际访问查询（如失效链接、版本号、API 端点），需联网复查三次后修改
   - **禁止绕过审查**：不得以任何方式搜索、批量、绕过逐句子审查流程

## Lint 配置

- **markdownlint**（`.github/.markdownlint.json`）：本地已安装 `markdownlint-cli2 v0.22.1`（基于 `markdownlint v0.40.0`），可在本地直接运行检查与修复
  - **本地运行命令**（在仓库根目录执行）：

    ```sh
    # 检查全部中文 .md（排除 en/、.github/、.gitbook/、node_modules/、script/）
    markdownlint-cli2 "**/*.md" "!en/**" "!.github/**" "!.gitbook/**" "!node_modules/**" "!script/**" --config .github/.markdownlint.json

    # 自动修复可修复的问题（MD012 多余空行、MD047 末尾换行、MD034 裸 URL 等）
    markdownlint-cli2 "**/*.md" "!en/**" "!.github/**" "!.gitbook/**" "!node_modules/**" "!script/**" --config .github/.markdownlint.json --fix
    ```

  - **MD060 表格列风格规则（重要）**：配置为 `"style": "compact"` + `"aligned_delimiter": true`
    - **compact 风格**：管道符两侧各保留**单个空格**，例如 `| cell1 | cell2 |`；空单元格写作 `| |`（不是 `||` 或 `|  |`）
    - **aligned_delimiter**：分隔行 `|---|` 的管道符位置必须与表头行的管道符位置对齐
    - **CJK 显示宽度**：markdownlint 按**显示宽度**计算列位置，**每个中文字符按 2 宽度**计算（与英文 1 宽度不同）。因此含 CJK 的列宽 = CJK 字符数 × 2 + ASCII 字符数 × 1
    - **分隔行 dash 数量公式**：若某列内容显示宽度为 W，则分隔行该列总宽（含两侧空格）也应为 W，dash 数量 = W − 4（减去前后各 1 空格 + 前后各 1 冒号）。例如列内容 `FreeBSD 版本` 显示宽度 = 7+1+4 = 12，分隔行应为 ` :--------: `（8 个 dash）
    - **常见错误修复**：当 markdownlint 报 `MD060/table-column-style [Table pipe does not align with header for option "aligned_delimiter"]` 时，需手动扩展分隔行 dash 数量，使管道符位置与表头按显示宽度对齐；`--fix` 无法自动修复此类问题
  - **MD056 表格列数规则**：表头声明几列，所有数据行都必须有几列；末尾缺空单元格时需补 `| |`，而非省略管道符
- **textlint**（`.textlintrc`）：仅启用 `ja-space-between-half-and-full-width` 规则，用于 CJK/英文空格检查
- **lychee**（`.github/lychee.toml`）：6 线程、30 并发、30 秒超时、最多 3 次重试、Chrome UA、排除私有 IP 和 `ftp.freebsd.org`

## 日后更新办法（AI 执行流程）

当上游 freebsd-doc 仓库有新提交，需同步至本项目时，按以下 8 步执行：

### 步骤 1：获取上游 commit
- 访问 https://github.com/freebsd/freebsd-doc/commit/<commit-hash> 确认变更范围
- 记录新 commit hash

### 步骤 2：下载 ADOC 文件
- 从 freebsd-doc 仓库 `documentation/content/en/books/arch-handbook/` 下载最新 `_index.adoc` 文件
- 替换 `en/` 对应子目录下的 `_index.adoc`
- 注意：部分文件名含英文冒号，Windows 下需用 zip 下载方式
- **禁止下载或参考 `.po` 文件**，只同步 `.adoc`

### 步骤 3：逐章 diff
- 对每个章节，运行 `git diff en/<chapter>/_index.adoc` 找出英文原文变更
- 仅关注正文 `=`/`==`/`===` 标题与段落文本的变更

### 步骤 4：同步翻译（逐句手动，禁止批量）
- 针对变更的 adoc 段落，逐句修正中文 `.md` 文件
- 遵循「翻译校对工作流程」的逐句对照规则
- 禁止使用 sed/awk/脚本批量替换

### 步骤 5：结构核对
- 比较 `en/` 目录与 `SUMMARY.md`
- 若上游新增章节：在 `di-yi-bu-fen-nei-he/` 新建 `.md`，翻译，插入 SUMMARY.md
- 若上游删除章节：移除对应 `.md` 与 SUMMARY.md 条目
- 验证三部分归属正确（内核/设备驱动程序/附录）

### 步骤 6：更新基准 commit
- 将本文件「版本与流程」章节的「当前翻译基准上游 commit」更新为新 commit hash

### 步骤 7：三轮复核
- 按「逐句校对递归工作流」执行三轮逐句校对 + 一轮严格复查
- 递归直至无问题

### 步骤 8：提交
- 创建分支 `fix/arch-handbook-sync-<commit短>`
- 仅提交修改的 `.md` 文件与 `en/` 下更新的 ADOC 文件
- 提交信息：`fix: 同步上游 arch-handbook 至 <commit短>`
- 用 `gh pr create` 创建 PR，目标分支 `main`
- PR 描述列出每处变更及原因
