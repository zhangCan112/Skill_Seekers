# Skill Seekers 工作原理

## 概述

Skill Seekers 是一个自动化工具，将文档网站、GitHub 仓库和 PDF 文件转换为 LLM 可用的技能包，支持 Claude AI、Google Gemini、OpenAI ChatGPT 和通用 Markdown 四个平台。

---

## 核心工作流程 (5 个阶段)

### 1. 数据采集 📥
根据不同输入源使用专门的爬虫：

| 输入源 | 爬虫模块 | 关键特性 |
|--------|-----------|----------|
| 文档网站 | `doc_scraper.py` | BeautifulSoup + llms.txt 优先检测 |
| GitHub 仓库 | `github_scraper.py` | API 分析代码、issues、releases |
| PDF 文件 | `pdf_scraper.py` | PyMuPDF 提取文本、表格、图像 |
| 本地代码库 | `codebase_scraper.py` | C3.x 深度分析（模式、示例、配置） |

**性能优化**:
- 异步/多线程爬取（2-3x 速度提升）
- 智能检查点机制（支持中断后恢复）
- 代码语言自动检测

---

### 2. 内容构建 🏗️
将采集的数据组织成结构化的技能目录：

```
output/{skill}/
├── SKILL.md          # 技能主文件
├── references/        # 分类参考文档
│   ├── index.md
│   ├── api.md
│   ├── guides.md
│   └── ...
├── scripts/          # 用户脚本
└── assets/           # 用户资源
```

**构建过程**:
1. 加载爬取的 JSON 文件
2. 智能分类到不同主题（基于 URL、标题、关键词）
3. 提取代码示例和模式
4. 生成参考文件（每个分类一个 .md）
5. 创建基础 SKILL.md 模板

---

### 3. AI 增强 ✨
使用本地编码代理分析参考文档，智能生成高质量 SKILL.md。

#### 3.1 读取和预处理参考文档

**步骤 1: 扫描 references/ 目录**
```python
references = read_reference_files(
    skill_dir, 
    max_chars=30000,        # 每个文件最多 30K 字符
    preview_limit=5000       # 预览前 5K 字符
)
```

**步骤 2: 智能总结（当内容 >30K 字符时自动触发）**
如果总内容超过 30K 字符，应用智能总结策略：

```
原始内容: 100,000 字符
    ↓
保留前 20% (20,000 字符): 介绍、概述、前言
    ↓
提取最佳代码块 (最多 5 个): 优先实际使用示例
    ↓
保留关键标题 + 首段 (最多 10 个): 核心概念
    ↓
总结后: ~30,000 字符 (30% 原始大小)
```

**总结策略细节**:
- **优先级 1**: 保留介绍/概述部分（前 20%）
- **优先级 2**: 提取代码块（5 个最佳，带上下文）
- **优先级 3**: 保留章节标题和第一段（10 个主要章节）
- **结果**: 从 100K → 30K 字符，保留最关键信息

---

#### 3.2 创建增强提示词 (Prompt)

**提示词结构**（约 2,000-5,000 字符）:

```markdown
SKILL OVERVIEW:
- Name: react
- Source Types: documentation, github
- Multi-Source: Yes
- Conflicts Detected: Yes

CURRENT SKILL.MD:
--- (现有 SKILL.md 内容，或空) ---

SOURCE ANALYSIS:
--- (引用文件统计: 类型、数量、大小) ---

REFERENCE DOCUMENTATION:
--- (智能总结后的参考内容) ---

REFERENCE PRIORITY (when sources differ):
1. Code patterns (codebase_analysis): Ground truth - 代码实际做什么
2. Official documentation: 官方 API 和使用模式
3. GitHub issues: 真实世界使用和已知问题
4. PDF documentation: 额外上下文和教程

YOUR TASK:
--- (详细指令，见下方) ---
```

**提示词关键指令**:

**1. 多源综合**
```
- 承认此技能结合多个源
- 突出源之间的共识（建立信心）
- 透明记录差异（如存在）
- 使用源优先级综合冲突信息
```

**2. 清晰的"何时使用此技能"部分**
```
- 具体的触发条件
- 具体用例列表
- 文档 + 真实使用的视角
```

**3. 优秀的快速参考部分**
```
- 提取 5-10 个最佳、最实用的代码示例
- 优先高置信度源
- 优先实际使用的代码示例（来自 codebase）
- 优先官方文档示例（来自 docs）
- 选择简短、清晰的示例（5-20 行）
- 使用正确的语言标签 (cpp, python, javascript)
- 添加清晰描述，注明来源
```

**4. 详细的参考文件描述**
```
- 解释每个参考文件的内容
- 注意源类型和置信度
- 帮助用户导航多源文档
```

**5. 实用的"使用此技能"部分**
```
- 初学者、中级、高级用户的清晰指导
- 多源参考的导航提示
- 如存在，如何解决冲突
```

**6. 关键概念部分**
```
- 解释核心概念
- 定义重要术语
- 如需要，调和源之间的差异
```

**7. 冲突处理**
```
添加"已知差异"部分:
- 透明解释主要冲突
- 提供每种情况信任哪个源的指导
```

**关键约束**:
```
- 从上面的参考文档中提取真实示例
- 综合时优先高置信度源
- 有帮助时注明源归属（如"官方文档说 X，但代码显示 Y"）
- 使差异透明，不隐藏
- 优先简短、清晰的示例
- 使其可操作和实用
- 保持前置元数据完整 (---\nname: ...\n---)
- 使用正确的 Markdown 格式
```

---

#### 3.3 运行本地 AI 代理

**Headless 模式** (默认):
```bash
# 1. 创建临时提示词文件
prompt_file = /tmp/enhance_prompt_xxxxxx.txt

# 2. 运行本地 CLI（等待完成，最多 10 分钟）
claude --dangerously-skip-permissions $prompt_file
# 或
codex exec --full-auto - < $prompt_file

# 3. 验证 SKILL.md 被更新
initial_mtime = skill_md_path.stat().st_mtime
... 等待完成 ...
new_mtime = skill_md_path.stat().st_mtime
if new_mtime > initial_mtime:
    ✅ 成功
```

**Terminal 模式**:
```bash
# 1. 在 macOS 自动检测终端应用
if sys.platform == 'darwin':
    terminal_app = detect_terminal_app()  # Ghostty, iTerm, Terminal
    
# 2. 创建 shell 脚本
cat > /tmp/enhance_xxxxxx.sh << 'EOF'
#!/bin/bash
claude --dangerously-skip-permissions $prompt_file
echo ""
echo "✅ Enhancement complete!"
echo "Press any key to close..."
read -n 1
rm $prompt_file
EOF

# 3. 在新终端窗口打开
open -a $terminal_app /tmp/enhance_xxxxxx.sh
```

**Background 模式**:
```python
# 1. 启动后台线程
thread = threading.Thread(target=background_worker, daemon=True)
thread.start()

# 2. 立即返回
print("✅ Background enhancement started!")
print("监控状态文件:", status_file)

# 3. 在后台执行
# - 读取参考文件
# - 创建提示词
# - 运行 AI 代理
# - 写入状态到 .enhancement_status.json
```

**Daemon 模式**:
```python
# 1. 创建独立 Python 脚本
daemon_script = f'''
import os
import subprocess
from datetime import datetime

def write_status(status, message, progress):
    status_data = {{
        "status": status,
        "message": message,
        "progress": progress,
        "timestamp": datetime.now().isoformat()
    }}
    # 写入 .enhancement_status.json

# 运行增强
enhancer = LocalSkillEnhancer(...)
prompt = enhancer.create_enhancement_prompt()
result = subprocess.run(...)

# 写入最终状态
'''

# 2. 使用 nohup 启动（完全分离）
subprocess.Popen([
    "nohup", "python3", daemon_script_path
], stdout=log_file, stderr=log_file, start_new_session=True)

# 3. 守护进程继续运行，即使父进程退出
```

---

#### 3.4 代理执行和验证

**支持的 AI 代理命令模板**:

| 代理 | 命令模板 | 支持权限跳过 |
|------|----------|----------------|
| Claude Code | `claude {prompt_file}` | ✅ |
| OpenAI Codex | `codex exec --full-auto --skip-git-repo-check -` | ❌ |
| GitHub Copilot | `gh copilot chat` | ❌ |
| OpenCode | `opencode` | ❌ |
| 自定义 | 用户自定义 `{prompt_file}` 或 stdin | ⚠️ |

**执行流程**:
```python
# 1. 构建命令
if "{prompt_file}" in command_template:
    cmd = command_template.replace("{prompt_file}", prompt_file)
    # 从文件读取
    result = subprocess.run(cmd, capture_output=True, timeout=600)
else:
    # 通过 stdin 传递
    prompt = Path(prompt_file).read_text()
    result = subprocess.run(
        cmd, 
        input=prompt,
        capture_output=True, 
        timeout=600
    )

# 2. 检查返回码
if result.returncode == 0:
    # 3. 验证 SKILL.md 被更新
    if skill_md_path.exists():
        new_size = skill_md_path.stat().st_size
        if new_size > initial_size:
            print(f"✅ Enhancement complete! ({elapsed:.1f} seconds)")
            print(f"   SKILL.md: {new_size:,} bytes")
            return True

# 4. 清理临时文件
os.unlink(prompt_file)
```

**验证检查**:
- ✅ SKILL.md 文件存在
- ✅ 修改时间比初始时间新
- ✅ 文件大小增加（通常是 200-1000+ 行）
- ✅ 包含所有增强内容

---

#### 3.5 状态监控

**状态文件结构** (`output/{skill}/.enhancement_status.json`):
```json
{
  "status": "running",  // pending | running | completed | failed
  "message": "Enhancement in progress...",
  "progress": 0.5,    // 0.0-1.0
  "timestamp": "2026-02-08T10:30:00",
  "skill_dir": "output/react/",
  "error": null
}
```

**监控命令**:
```bash
# 查看当前状态
cat output/react/.enhancement_status.json

# 实时监控
skill-seekers enhance-status output/react/ --watch

# JSON 输出
skill-seekers enhance-status output/react/ --json
```

**后台/守护进程监控**:
```python
# 检查状态
status = read_status()

if status["status"] == "completed":
    print("✅ Enhancement completed!")
elif status["status"] == "running":
    progress = status["progress"] * 100
    print(f"⏳ In progress: {progress}%")
elif status["status"] == "failed":
    print(f"❌ Failed: {status['error']}")
```

---

#### 3.6 增强结果示例

**增强前** (基础模板，~75 行):
```markdown
---
name: react
description: Use when working with React
---

# React Skill

Use when working with React, generated from official documentation.

## Quick Reference

*Quick reference patterns will be added as you use this skill.*

## Reference Files

- getting_started.md - Getting started documentation
- api.md - API reference

## Working with This Skill

Use `view` to read specific reference files when detailed information is needed.
```

**增强后** (AI 生成，~500-1000 行):
```markdown
---
name: react
description: Use when building modern web applications with React
---

# React Skill

Use when building modern web applications with React, managing state, handling events, or implementing component architecture.

## When to Use This Skill

This skill should be triggered when:
- Building React applications with functional or class components
- Managing application state with hooks (useState, useEffect, useContext)
- Handling events and user interactions
- Implementing component composition and props passing
- Debugging React applications or optimizing performance
- Working with React Router for navigation
- Integrating with third-party libraries

## Key Concepts

### Components
React applications are built from components—self-contained UI elements that accept inputs (props) and return UI descriptions (JSX). Components can be:
- **Functional Components**: Modern approach using JavaScript functions
- **Class Components**: Legacy approach using ES6 classes (still supported)

### Hooks
Hooks are functions that let you "hook into" React state and lifecycle features from function components:
- **useState**: Manages local state within a component
- **useEffect**: Handles side effects (API calls, subscriptions)
- **useContext**: Accesses context values without prop drilling

## Quick Reference

### Component Pattern

**Pattern 1: Functional Component with State**

From official docs:
```javascript
import React, { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

### API Pattern

**Pattern 2: useEffect for Data Fetching**

From codebase examples:
```javascript
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data));
  }, [userId]); // Dependency array
  
  if (!user) return <div>Loading...</div>;
  return <div>{user.name}</div>;
}
```

### State Management Pattern

**Pattern 3: Context API**

From official documentation:
```javascript
const ThemeContext = React.createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// Consuming context
function ThemedButton() {
  const { theme, setTheme } = useContext(ThemeContext);
  return <button style={{ background: theme }}>Click</button>;
}
```

## Reference Files

This skill combines knowledge from documentation and codebase analysis:

### Documentation Sources (24 files)
- **getting_started.md** - Installation and first steps setup
- **components.md** - Component patterns and composition
- **hooks.md** - Complete hooks reference and examples
- **state_management.md** - State patterns and context API
- **events.md** - Event handling and synthetic events

### Codebase Analysis (12 files)
- **patterns.md** - Design patterns extracted from React source
- **examples.md** - Real-world usage examples from React codebase
- **architecture.md** - Component rendering and reconciliation

## Working with This Skill

### For Beginners
Start with getting_started.md and components.md to understand React fundamentals:
- Component structure and JSX syntax
- Props and state basics
- First introduction to hooks

### For Intermediate Users
Focus on hooks.md and state_management.md for practical patterns:
- Master useState, useEffect, and other core hooks
- Learn context API for prop drilling solutions
- Implement custom hooks for reusable logic

### For Advanced Users
Explore patterns.md and architecture.md to understand React internals:
- Virtual DOM and reconciliation
- Performance optimization techniques
- Advanced patterns like render props and higher-order components

### Multi-Source Navigation
This skill combines official documentation with real-world usage from codebase:
- **Priority 1**: Code patterns (actual implementation)
- **Priority 2**: Official documentation (intended usage)
- **Conflicts**: Refer to conflicts.md for discrepancies

When encountering differences between docs and code, trust codebase_analysis patterns as ground truth.

## Known Discrepancies

This skill detected conflicts between documentation and codebase:

### useEffect Cleanup Pattern

⚠️ **Conflict**: Documentation recommends different cleanup approach

**Documentation says:**
```javascript
useEffect(() => {
  // effect logic
  return () => cleanup(); // Return cleanup function
}, []);
```

**Codebase shows:**
```javascript
useEffect(() => {
  // effect logic
}, []); // Cleanup handled separately
```

**Resolution**: For new code, follow documentation pattern. Legacy code may use codebase approach.

## Resources

### references/
Organized documentation from multiple sources:
- High confidence sources (codebase patterns)
- Official documentation with API references
- Real-world usage examples

### scripts/
Add helper scripts for common React automation tasks.

### assets/
Add React boilerplate, templates, or example projects.

## Notes

- This skill was automatically generated and enhanced with AI
- Combines official documentation with codebase analysis
- Conflicts are transparently reported
- Code examples include language detection for better syntax highlighting
- Quality improved from 3/10 → 9/10 through AI enhancement
```

**增强对比**:
- ✅ 行数: 75 → 800+ (10x 增加)
- ✅ 内容: 模板 → 综合指导
- ✅ 示例: 0 → 5-10 个实用示例
- ✅ 结构: 单一 → 多层级（初学者/中级/高级）
- ✅ 源归因: 无 → 明确标注（官方文档/代码库）
- ✅ 冲突处理: 无 → 透明报告和解决建议

---

#### 3.7 增强时间估算

| 技能大小 | 参考内容 | 总结后 | AI 处理时间 | 总时间 |
|----------|----------|--------|------------|--------|
| 小 (<100 页) | 5K 字符 | 不需要 | 10-20 秒 | <1 分钟 |
| 中 (100-500 页) | 15K 字符 | 不需要 | 20-40 秒 | <1 分钟 |
| 大 (500-2000 页) | 30K 字符 | 30K 字符 | 40-60 秒 | 1-2 分钟 |
| 超大 (2000-10000 页) | 50K 字符 | 15K 字符 | 60-120 秒 | 2-4 分钟 |

**性能优化**:
- ✅ 智能总结减少 AI 输入大小（50-70% 减少）
- ✅ 并行代码块提取
- ✅ 缓存参考文件元数据
- ✅ 超时保护（默认 10 分钟）

---

### 3.8 多源综合增强

**当技能包含多个源时的特殊处理**:

**场景**: React 技能 = 文档 + GitHub 代码库 + issues

**增强策略**:
```python
# 1. 按源分组参考文件
by_source = {
    ("documentation", None): [getting_started.md, api.md, ...],
    ("github", "facebook/react"): [patterns.md, examples.md, ...],
    ("github", "issues"): [common_problems.md, ...]
}

# 2. 为每个源生成总结
for (source, repo_id), files in by_source.items():
    if repo_id:
        prompt += f"\n### {source.upper()} - {repo_id} ({len(files)} files)\n"
    else:
        prompt += f"\n### {source.upper()} ({len(files)} files)\n"
    
    for filename, metadata in files[:5]:  # 前 5 个文件
        prompt += f"- {filename} (confidence: {metadata['confidence']})\n"
```

**源优先级规则**（在综合时）:
```
1. 代码模式: 最高优先级 - 代码实际做什么
2. 官方文档: 次优先级 - 官方 API 和使用模式
3. GitHub Issues: 补充优先级 - 真实使用和已知问题
4. PDF 文档: 最低优先级 - 额外上下文和教程
```

**多仓库处理**:
```
检测到多个仓库: httpx, httpcore

在增强时:
- 清晰标识每个内容来自哪个仓库
- 比较和对比跨仓库模式
  例如: "httpx 使用策略模式 50 次，httpcore 使用 32 次"
- 突出关系
  例如: "httpx 是构建在 httpcore 之上的客户端库"
- 展示两个仓库的示例以展示不同用例
```

---

### 3.9 备份和恢复

**自动备份机制**:
```python
# 在增强前备份原始 SKILL.md
original = skill_md_path.read_text()
backup_path = skill_md_path.with_suffix('.md.backup')

backup_path.write_text(original)
print(f"✅ Backed up to: {backup_path}")
```

**恢复命令**（如对增强结果不满意）:
```bash
# 恢复原始版本
mv output/react/SKILL.md.backup output/react/SKILL.md

# 重新增强（使用不同代理）
skill-seekers enhance output/react/ --agent codex
```

---

**增强阶段总结**:

```
1. 📖 读取 references/ 目录中的所有参考文件
2. 🧠 智能总结（如内容 >30K，减少到 30%）
3. 📝 创建 2-5K 字符的详细增强提示词
4. 🤖 运行本地 AI 代理（Claude/Codex/Copilot/OpenCode）
5. ✅ 验证 SKILL.md 被更新且质量提升
6. 💾 自动备份原始 SKILL.md
7. 📊 写入状态到 .enhancement_status.json（后台/守护进程）
```

**关键指标**:
- ⏱️  处理时间: 10秒 - 4分钟（取决于技能大小）
- 📊  内容增加: 75 行 → 500-1000 行（10x+）
- 🎯  质量提升: 3/10 → 9/10
- 🔄  备份保护: 自动备份 SKILL.md.backup
- 📈  成功率: >95%（智能总结 + 超时保护）
### 3. AI 增强 ✨
使用本地编码代理分析参考文档，智能生成高质量 SKILL.md。

**运行模式**:
- **Headless** (默认): 直接运行 CLI，等待完成
- **Background**: 后台线程运行
- **Daemon**: 独立守护进程
- **Terminal**: 打开新终端窗口

**支持的代理**:
- Claude Code（默认，推荐）
- OpenAI Codex CLI
- GitHub Copilot CLI
- OpenCode CLI
- 自定义代理
### 3. AI 增强 ✨
使用本地编码代理分析参考文档，智能生成高质量 SKILL.md。

---

### 4. 打包 📦
通过 **Adaptor 模式**（策略模式）支持多平台打包：

```python
from skill_seekers.cli.adaptors import get_adaptor

adaptor = get_adaptor('claude')  # 或 'gemini', 'openai', 'markdown'
package_path = adaptor.package(skill_dir, output_dir)
```

**平台特定格式**:

| 平台 | 格式 | 说明 |
|------|------|------|
| Claude AI | ZIP + YAML | 标准 zip，包含 SKILL.md 前置元数据 |
| Google Gemini | tar.gz | 压缩归档格式 |
| OpenAI ChatGPT | ZIP + Vector Store | 包含向量索引的 zip |
| Generic Markdown | ZIP | 通用 Markdown 格式 |

**打包前质量检查**:
- SKILL.md 存在且有效
- references/ 目录有文件
- 必需字段完整（name, description）
- 质量评分（errors/warnings）

---

### 5. 上传 ☁️
自动上传到目标平台（可选）：

**方式 1: API 自动上传**（需要 API Key）
```bash
skill-seekers package output/react/ --upload
```

**方式 2: 手动上传**
- 生成 .zip 文件
- 打开文件浏览器
- 显示上传指南

**支持的平台 API**:
- `ANTHROPIC_API_KEY` - Claude AI
- `GOOGLE_API_KEY` - Google Gemini
- `OPENAI_API_KEY` - OpenAI ChatGPT

---

## 架构设计

### 1. Adaptor 模式（策略模式）
所有平台特定逻辑封装在独立的 adaptor 类中：

```python
class SkillAdaptor(ABC):
    @abstractmethod
    def package(self, skill_dir, output_dir) -> Path:
        """打包技能为平台特定格式"""
        pass
    
    @abstractmethod
    def upload(self, package_path, api_key) -> dict:
        """上传技能到平台"""
        pass
```

**Adaptor 实现**:
- `base.py` - 抽象基类
- `claude.py` - Claude AI 实现
- `gemini.py` - Google Gemini 实现
- `openai.py` - OpenAI ChatGPT 实现
- `markdown.py` - 通用 Markdown 实现

**优势**: 新增平台只需添加新的 adaptor 类，无需修改核心代码。

---

### 2. Git-style CLI
统一入口点 `main.py` 使用子命令：

```bash
skill-seekers scrape --config configs/react.json      # 文档爬取
skill-seekers github --repo facebook/react           # GitHub 爬取
skill-seekers pdf --pdf docs/manual.pdf             # PDF 提取
skill-seekers enhance output/react/                    # AI 增强
skill-seekers package output/react/                    # 打包
skill-seekers upload react.zip                         # 上传
```

**优势**: 用户熟悉的 git 风格命令结构，易于学习和记忆。

---

### 3. One-Command Install
一键完成整个流程：

```bash
skill-seekers install --config react
```

**自动执行**:
1. 获取配置（从 API 或本地）
2. 爬取文档（AI 增强是强制的）
3. 打包技能为平台特定格式
4. 上传到 Claude（需要 API Key）

**时间**: 20-45 分钟（取决于文档大小）

---

## 多源统一处理

**Unified Multi-Source** 模式结合多个源：
- 文档网站（官方文档，意图）
- GitHub 仓库（实际代码实现，现实）
- PDF 文件（补充教程）

**冲突检测**:
自动发现 4 种差异并透明报告：

| 冲突类型 | 描述 | 严重性 |
|----------|------|---------|
| 代码中缺失 | 文档记录了 API 但代码未实现 | 高 |
| 文档中缺失 | 代码实现了 API 但文档未记录 | 中 |
| 签名不匹配 | 参数/类型不同 | 中 |
| 描述不匹配 | 解释不同 | 低 |

**示例**:
```markdown
#### `move_local_x(delta: float)`

⚠️ **Conflict**: Documentation signature differs from implementation

**Documentation says:**
```
def move_local_x(delta: float)
```

**Code implementation:**
```python
def move_local_x(delta: float, snap: bool = False) -> None
```
```

**优势**: 单一技能展示意图（文档）和现实（代码），提供可信来源优先级。

---

## MCP 集成

**MCP Server** 支持 5 个 AI 编码代理：

| 代理 | 传输模式 | 配置难度 |
|------|----------|----------|
| Claude Code | stdio | 简单 |
| VS Code + Cline | stdio | 简单 |
| Cursor | HTTP | 中等 |
| Windsurf | HTTP | 中等 |
| IntelliJ IDEA | HTTP | 中等 |

**18 个可用工具**:
- **配置工具** (3): `list_configs`, `generate_config`, `validate_config`
- **爬取工具** (8): `scrape_docs`, `scrape_github`, `scrape_pdf`, `unified_scrape`, `estimate_pages`, `extract_test_examples`, `analyze_codebase`, `merge_sources`
- **打包工具** (4): `package_skill`, `upload_skill`, `split_config`, `generate_router`
- **源管理** (3): `add_config_source`, `fetch_config`, `list_config_sources`

**优势**: 自然语言交互，无需手动 CLI 命令。

---

## 配置管理

**存储位置**: `~/.config/skill-seekers/config.json` (600 权限)

**支持**:
- 多 GitHub Token 管理（个人、工作、OSS）
- API Keys 存储
- Rate limit 策略（prompt, wait, switch, fail）
- Profile 配置

**获取配置优先级**:
1. CLI 参数（最高）
2. 环境变量
3. 配置文件
4. 用户提示（最低）

**Rate Limit 策略**:
- **prompt** (默认): 询问用户操作
- **wait**: 自动等待倒计时
- **switch**: 自动切换到下一个 profile
- **fail**: 立即失败（CI/CD 专用）

---

## 总结

**Skill Seekers** 的核心工作原理：

```
1. 📥 采集: 从文档、GitHub、PDF 或本地代码库提取原始数据
   ↓
2. 🏗️ 构建: 整理和分类内容到参考文件
   ↓
3. ✨ 增强: 使用 AI 代理生成高质量 SKILL.md
   ↓
4. 📦 打包: 通过 Adaptor 模式创建平台特定包
   ↓
5. ☁️ 上传: 自动或手动上传到目标 LLM 平台
```

**核心优势**:
- ✅ **多源支持**: 文档 + 代码 + PDF，冲突检测
- ✅ **多平台**: Claude AI、Gemini、OpenAI、Markdown
- ✅ **AI 增强**: 确保技能质量（质量从 3/10 → 9/10）
- ✅ **透明度**: 多源冲突清晰报告
- ✅ **MCP 集成**: 支持 5 个 AI 编码代理
- ✅ **性能优化**: 异步/并行，检查点恢复
- ✅ **模块化设计**: Adaptor 模式轻松扩展新平台

**设计理念**: 通过策略模式、模块化和清晰的关注点分离，创建一个可扩展、可维护的系统，轻松支持新平台和数据源。