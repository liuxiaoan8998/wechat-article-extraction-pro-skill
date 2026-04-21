---
name: wechat-article-extraction-pro
description: >
  微信公众号文章提取 Pro 版完整执行流程。
  基于 wechat-article-for-ai 增强，实现5文件输出 + RapidOCR 识别 + Hermes 总结回填 + 飞书 Base 同步。
  Hybrid 模式：工具负责提取+OCR，Hermes 负责总结，支持内容管理和二创工作流。
---

# 微信公众号文章提取 Pro 版 - 执行流程

> **⚠️ 工具维护提醒**：如果 `/tmp/wechat-article-for-ai-pro/` 目录为空或损坏，请重新克隆：
> ```bash
> cd /tmp && rm -rf wechat-article-for-ai-pro && git clone https://github.com/liuxiaoan8998/wechat-article-for-ai-pro.git
> ```

## 完整流程图

```
用户提供 URL
    ↓
┌─────────────────────────────────────────────────────────┐
│  阶段 1：工具提取（本地执行）                              │
│  ─────────────────────────────                          │
│  1. 启动 Camoufox 浏览器访问文章                          │
│  2. 提取标题、作者、正文内容                               │
│  3. 下载所有图片到 images/ 目录                           │
│  4. OCR 识别（RapidOCR）：                                │
│     - 普通图片：直接 OCR                                  │
│     - 长图(>2000px)：切片 → 分段 OCR                     │
│  5. 生成 5 文件输出：                                     │
│     - article.md（基础 Markdown）                        │
│     - article.html（HTML 格式）                          │
│     - article-ocr.md（OCR 结果，第三部分待填充）          │
│     - metadata.json（元数据）                            │
│     - images/（原始图片）                                │
│     - slices/（长图切片）                                │
│  6. 输出：文件路径 + "需要 Hermes 总结" 提示               │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│  阶段 2：Hermes 总结（AI 处理）                           │
│  ─────────────────────────────                          │
│  1. 读取 article-ocr.md 的 OCR 内容                      │
│  2. 结构化分析：                                          │
│     - 公司/组织概况                                       │
│     - 招聘/活动亮点                                       │
│     - 关键信息（时间、地点、岗位等）                       │
│     - 投递/参与方式                                       │
│  3. 生成 Markdown 格式的第三部分                          │
│  4. 回填到 article-ocr.md                                │
│  5. 同步更新 ~/.hermes/output/ 目录                      │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│  阶段 3：飞书 Base 同步（默认执行，无需询问）              │
│  ─────────────────────────────                          │
│  1. 提取关键信息（标题、公众号、发布时间等）                │
│  2. 调用 Lark CLI 录入多维表格                            │
│  3. 更新状态为"待处理"                                   │
│  ⚠️ 此步骤为默认操作，不再询问用户                        │
└─────────────────────────────────────────────────────────┘
    ↓
完成
```

> **💡 状态保存提示**：任务完成后，自动更新 `~/.hermes/.wechat_workflow_state` 标记文件。如对话即将压缩，我会提前提醒你保存进度。

## 定期状态提醒

**机制**：每进行50轮对话时，自动提示：
```
[对话轮次提醒] 当前已进行XX轮对话，接近上下文压缩阈值（约358轮）。
建议：如有重要任务或状态需要保存，请告知我"保存当前进度"。
```

**用户指令**：
- 说"**保存当前进度**" → 我立即将任务状态写入标记文件
- 说"**查看状态**" → 我读取标记文件并汇报当前进度

## 技术栈

| 组件 | 技术 |
|------|------|
| 浏览器 | Camoufox（Playwright） |
| OCR 引擎 | RapidOCR（本地 ONNX） |
| 备用 OCR | AI Vision（云端） |
| 图片处理 | Pillow |
| 输出格式 | Markdown + HTML + JSON |

## 关键修复

| 问题 | 修复 |
|------|------|
| RapidOCR 返回坐标而非文字 | 修正结果解析：`item[1]` 才是文字内容（格式是 `[[box, text, confidence], ...]`） |
| RGBA 图片保存失败 | 切片前自动转换为 RGB 模式 |

## 执行命令

### 1. 工具提取

```bash
cd /tmp/wechat-article-for-ai-pro
python3 -c "
from wechat_to_md.cli import main
import sys
sys.argv = ['wechat_to_md', 'URL', '-o', 'OUTPUT_DIR']
main()
"
```

### 2. Hermes 总结回填

```python
# 读取 OCR 结果
read_file: path/to/article-ocr.md

# 分析 OCR 内容，生成结构化总结

# 回填到第三部分
patch: path/to/article-ocr.md
  old_string: "## 三、完整文字内容...\n\n*待 OCR 完成后自动更新*"
  new_string: "## 三、完整文字内容...\n\n### 公司概况\n..."

# 同步到 ~/.hermes/output/
cp -r /tmp/test_output/文章标题 ~/.hermes/output/

# 同步到飞书 Base（内容管理）
export PATH="$HOME/.npm-global/lib/node_modules/@larksuite/cli/bin:$PATH"
lark-cli base +record-upsert \
  --base-token "E9y1bxjHGa9LeGs9q3Tc3J41nmf" \
  --table-id "tblYIqHtHrWUlVnP" \
  --json '{
    "文章标题": "文章标题",
    "公众号": "公众号名称",
    "发布时间": "2026-04-20",
    "文章内容概要总结": "AI生成的结构化总结...",
    "投递方式": "投递渠道...",
    "文章链接": "https://mp.weixin.qq.com/s/xxx",
    "行业/领域": {"text": "互联网/科技"},
    "岗位类型": {"text": "实习"},
    "工作地点": {"text": "北京"},
    "学历要求": {"text": "本科"},
    "优先级": {"text": "中"},
    "状态": {"text": "待处理"},
    "标签": [{"text": "大厂"}],
    "备注": ""
  }' \
  --as bot
```

## 输出结构

```
~/.hermes/output/文章标题/
├── article.md              # 基础文字
├── article.html            # 网页格式
├── article-ocr.md          # OCR + 二维码识别 + 总结 ✅
├── metadata.json           # 元数据
├── images/                 # 原始图片
└── slices/                 # 长图切片（如有）
```

### article-ocr.md 结构（v1.7+）

```markdown
# 文章内容（含图片 OCR 识别）

## 一、原文文字内容
[原文 Markdown]

## 二、二维码识别内容
检测到 **N** 张包含二维码的图片

### img_XXX.jpg
**图片路径**: `images/img_XXX.jpg`

**二维码 1**:
- **类型**: 📝 招聘/报名链接 / 🔗 链接 / 📞 联系方式 / 📝 文本内容
- **内容**: {URL或文本}

---

## 三、图片 OCR 识别内容
[OCR 结果]

## 四、完整文字内容（原文 + OCR + 二维码）
*待 OCR 完成后自动更新*

### 二维码关键信息（供整合参考）
- **招聘/报名链接**: `http://weixin.qq.com/r/xxx`（来自 img_007.jpg）

> ⚠️ **注意**：检测到招聘/报名链接，请在总结中整合此信息

---
```

**第四章特点**：
- 标题明确包含"二维码"，提示 AI 需要整合
- 自动提取关键信息（招聘链接、联系方式等）
- 检测到招聘链接时显示警告提示，确保不被遗漏

## Hybrid 模式说明

### 为什么分离？

| 层级 | 职责 | 原因 |
|------|------|------|
| 工具 (Python) | 提取 + OCR | 适合网页抓取、本地 OCR 批量处理 |
| Hermes (AI) | 总结 + 回填 | 擅长理解上下文、结构化输出 |

### 协作流程

#### 单篇文章

1. **工具完成** → 输出文件路径
2. **用户通知 Hermes** → "请总结这篇文章"
3. **Hermes 读取** → article-ocr.md
4. **Hermes 生成** → 结构化总结
5. **Hermes 回填** → 更新 article-ocr.md
6. **Hermes 同步** → ~/.hermes/output/

#### 批量处理（多篇文章）

使用 `delegate_task` 并行处理：

```python
delegate_task(tasks=[
    {
        "context": "文件路径：/tmp/test_output/文章1/article-ocr.md",
        "goal": "读取 OCR 内容，生成结构化总结并回填",
        "toolsets": ["file", "terminal"]
    },
    {
        "context": "文件路径：/tmp/test_output/文章2/article-ocr.md", 
        "goal": "读取 OCR 内容，生成结构化总结并回填",
        "toolsets": ["file", "terminal"]
    },
    # ... 更多文章
])
```

**优势**：多篇文章并行处理，大幅提升效率

## 版本历史

| 版本 | 变更 |
|------|------|
| v1.0 | 标准4文件输出 |
| v1.2 | 添加 article-ocr.md 占位符 |
| v1.3 | AI Vision OCR 集成 |
| v1.4 | 集成 RapidOCR 本地识别 |
| v1.5 | 可配置 OCR 引擎（rapidocr/vision/auto）|
| v1.5.1 | 修复 RGBA 图片切片保存问题 |
| v1.5.2 | 修复 RapidOCR 返回格式解析 bug |
| v1.6 | **Hybrid 模式**：工具提取+OCR，Hermes 总结回填 |

---

# 历史开发记录（参考）

## 项目架构

```
wechat-article-for-ai-pro/
├── main.py                 # 入口
├── requirements.txt        # 依赖
├── README.md              # 中文文档
├── wechat_to_md/          # 核心模块
│   ├── cli.py            # 命令行接口
│   ├── scraper.py        # Camoufox 抓取
│   ├── parser.py         # 内容解析
│   ├── converter.py      # Markdown 转换
│   ├── downloader.py     # 图片下载
│   ├── formatter.py      # 4文件标准化输出
│   ├── ocr_adapter.py    # OCR 适配器 ⭐
│   └── ocr_processor.py  # OCR 处理模块
└── ...
```

基于 wechat-article-for-ai 的增强版本，实现标准化5文件输出 + AI Vision OCR 识别。

## 项目架构

```
wechat-article-for-ai-pro/
├── main.py                 # 入口
├── requirements.txt        # 依赖
├── README.md              # 中文文档
├── wechat_to_md/          # 核心模块
│   ├── cli.py            # 命令行接口
│   ├── scraper.py        # Camoufox 抓取
│   ├── parser.py         # 内容解析
│   ├── converter.py      # Markdown 转换
│   ├── downloader.py     # 图片下载
│   ├── formatter.py      # 4文件标准化输出
│   └── ocr_processor.py  # OCR 处理模块
└── ...
```

## 开发流程

### 1. 项目初始化

```bash
# 克隆原项目
git clone https://github.com/bzd6661/wechat-article-for-ai.git /tmp/wechat-article-for-ai-pro
cd /tmp/wechat-article-for-ai-pro
rm -rf .git
git init
git add .
git commit -m "Initial commit: fork from bzd6661/wechat-article-for-ai"
```

### 2. 添加标准4文件输出

创建 `wechat_to_md/formatter.py`：
- 生成 `article.md`（YAML frontmatter）
- 生成 `article.html`（原图优先展示）
- 生成 `metadata.json`（结构化元数据）
- 整理 `images/` 文件夹

修改 `cli.py` 集成 formatter。

### 3. 添加 OCR 功能

**方案演进**：
- v1.2: 仅生成占位符，依赖 Hermes 外部识别
- v1.4: 集成 RapidOCR 本地识别，实现全自动 OCR

创建 `wechat_to_md/ocr_adapter.py`（可配置 OCR 引擎）：
```python
class RapidOCREngine(OCREngine):
    def ocr(self, image_path: Path) -> str:
        result, elapse = self._engine(str(image_path))
        # ⚠️ 关键：RapidOCR 返回格式是 [[box, text, confidence], ...]
        # 不是 [text, confidence, box]！
        for item in result:
            text = item[1]  # item[1] 才是文字内容
```

创建 `wechat_to_md/ocr_processor.py`：
- `process_all_images()` - 自动检测长图、切片、OCR
- `slice_long_image()` - 超长图切片（>2000px）
- `create_ocr_markdown_v2()` - 生成包含 OCR 结果的完整文档

**OCR 流程**：
1. 检测图片高度，超长图自动切片
2. 调用 OCRAdapter（默认 RapidOCR）
3. 生成 article-ocr.md（原文 + OCR 结果）

**RapidOCR 返回格式陷阱**（重要！已踩坑）：
```python
# 错误理解：[[text, confidence, box], ...]
# 正确格式：[[box, text, confidence], ...]
# box = [[x1,y1], [x2,y2], [x3,y3], [x4,y4]]  # 四个角坐标
# text = "识别的文字"
# confidence = 0.99  # 置信度

# 正确解析代码：
for item in result:
    if isinstance(item, (list, tuple)) and len(item) >= 2:
        text = item[1]  # item[1] 才是文字，不是 item[0]！
```

### 4.1 OCR 适配器设计（推荐）

创建 `ocr_adapter.py` 实现多引擎支持：

```python
class OCRAdapter:
    """统一 OCR 适配器，支持 rapidocr/vision/auto"""
    
    def __init__(self, config: Optional[Dict] = None):
        self.config = config or {}
        self.engine_name = self.config.get('engine', 'rapidocr')
        self.engines = {
            'rapidocr': RapidOCREngine(),
            'vision': AIVisionEngine(),
        }
    
    def ocr(self, image_path: Path) -> str:
        engine = self.engines.get(self.engine_name)
        return engine.ocr(image_path)
```

**引擎选择策略**：
- `rapidocr`：本地运行，快速，适合批量处理
- `vision`：云端 AI，准确度高，适合复杂排版
- `auto`：先 rapidocr，失败时降级到 vision

### 4. Git 管理

**Commit 规范（中文）**：
```bash
git commit -m "v1.3: 添加图片 OCR 识别功能

- 新增 ocr_processor.py 模块
- 集成 AI Vision 识别图片文字
- 生成 article-ocr.md 包含原文+OCR结果"
```

**推送**：
```bash
git remote add origin https://github.com/username/repo.git
git push -u origin main
```

### 5. README 同步

每次代码修改后同步更新：
- 核心特性列表
- 输出文件结构
- 使用示例
- 与原版对比表

## 使用流程

### 提取文章

```bash
python main.py "https://mp.weixin.qq.com/s/..." -o ./output -v
```

### OCR 识别（Hermes 层）

```python
# 1. 获取图片列表
images = list(Path("output/文章标题/images").glob("img_*"))

# 2. 调用 vision_analyze 识别每张图片
for img in images:
    result = vision_analyze(image_url=str(img), question="提取所有文字")
    
# 3. 更新 article-ocr.md
```

## 输出结构

```
output/
└── 文章标题/
    ├── article.md          # Markdown 原文
    ├── article.html        # HTML 查看器
    ├── metadata.json       # 结构化元数据
    ├── article-ocr.md      # OCR 识别结果 ⭐
    └── images/             # 下载的图片
        ├── img_001.jpg
        └── ...
```

## 关键技术决策

### 为什么不用 PaddleOCR？
- 安装复杂，依赖多
- 环境兼容性问题
- 改用 AI Vision，无需本地安装

### 二维码识别方案（v1.7+）

**引擎选择**：
- **首选**：zbar-py（识别能力最强，支持复杂背景）
- **备选**：OpenCV QRCodeDetector（无需额外依赖）

**安装 zbar-py**：
```bash
pip3 install zbar-py
```

**智能分类**：基于URL关键词匹配（apply/job/career/校招/招聘/报名/投递等）自动标记招聘链接

**数据结构分离**：
- `process_all_images()` 返回 `(ocr_results, qr_results)` 元组
- 二维码内容独立成章（第二章节），而非附在每张图片OCR结果后
- 便于快速定位和批量处理

**第四章整合**：
- 标题：`完整文字内容（原文 + OCR + 二维码）`
- 自动提取关键信息（招聘链接、联系方式等）
- 检测到招聘链接时显示警告提示，确保 AI 总结时不遗漏

### 为什么分离 OCR 和提取？
- Python 环境无法直接调用 Hermes 工具
- 分离后流程清晰：提取 → 识别 → 整合
- 便于调试和替换 OCR 方案

## 常见问题

### 图片路径特殊字符
文件名中的引号等特殊字符可能导致路径问题，需要处理。

### OCR 识别失败
- 装饰图标无文字
- 二维码需解码
- 复杂排版识别不全

## 版本历史

| 版本 | 变更 |
|------|------|
| v1.0 | 标准4文件输出 |
| v1.2 | 添加 article-ocr.md 占位符 |
| v1.3 | AI Vision OCR 集成 |
| v1.3.1 | README 同步更新 |
| v1.4 | 集成 RapidOCR 本地识别，修复返回格式解析 bug |
| v1.5 | 可配置 OCR 引擎（rapidocr/vision/auto）|
| v1.5.1 | 修复 RGBA 图片切片保存问题 |
| v1.6 | **hybrid 模式**：工具提取+OCR，Hermes 负责总结回填 |
| v1.7 | **新增二维码识别功能**：OpenCV QRCodeDetector 自动检测，智能分类招聘/报名链接 |
| v1.7.1 | **第四章整合二维码信息**：完整文字内容（原文+OCR+二维码），自动提取关键链接供 AI 总结 |
| v1.7.2 | **集成 zbar-py**：识别能力更强，解决 OpenCV 无法识别的复杂背景二维码问题 |

---

## Hybrid 模式（推荐）

### 流程设计

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   工具层     │ ──→ │   Hermes    │ ──→ │   最终输出   │
│ (Python)    │     │   (AI)      │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
      │                   │                   │
   1. 提取文章         4. 读取 OCR 结果      6. 完整文档
   2. 下载图片         5. AI 总结          (article-ocr.md)
   3. RapidOCR 识别
      生成 article-ocr.md
      (OCR 结果 + 占位符)
```

### 为什么分离？

| 层级 | 职责 | 原因 |
|------|------|------|
| 工具 | 提取 + OCR | Python 环境适合网页抓取和本地 OCR |
| Hermes | 总结 + 回填 | AI 擅长理解上下文、结构化总结 |

### 触发机制

工具生成 `article-ocr.md` 后，创建标记文件通知 Hermes：

```python
# 工具端
with open(f"{output_dir}/.hermes_summary_pending", "w") as f:
    f.write(f"article_dir={output_dir}\n")

# Hermes 检测到标记后，自动读取 OCR 结果并总结回填
```

### 回填内容示例

```markdown
## 三、完整文字内容（原文 + OCR）

### 公司概况
**九号公司**是一家...

### 招聘亮点
| 亮点 | 说明 |
|------|------|
| 留用通道 | 提前锁定秋招Offer |
...

---
*由 Hermes Agent 基于 OCR 结果自动总结生成*
```

---

## 故障排除与维护

### Skill 丢失/损坏恢复

如果 Skill 目录被误删或损坏：

```bash
# 1. 从 GitHub 恢复 Skill
cd ~/.hermes/skills/web
git clone https://github.com/liuxiaoan8998/wechat-article-extraction-pro-skill.git wechat-article-extraction-pro

# 2. 重新部署 Python 项目
cd /tmp
rm -rf wechat-article-for-ai-pro
git clone https://github.com/liuxiaoan8998/wechat-article-for-ai-pro.git

# 3. 运行安装脚本
bash ~/.hermes/skills/web/wechat-article-extraction-pro/scripts/setup.sh
```

### 双仓库管理策略

| 仓库 | 地址 | 用途 | 更新时机 |
|------|------|------|----------|
| Python 源码 | `wechat-article-for-ai-pro` | 核心提取工具 | 功能迭代 |
| Hermes Skill | `wechat-article-extraction-pro-skill` | 调用指南和脚本 | 流程优化 |

**为什么分离？**
- Python 项目：关注功能实现（OCR、下载、解析）
- Skill 项目：关注调用流程（参数、步骤、示例）
- 不同迭代周期，避免互相干扰

### 版本控制工作流

**Skill 更新：**
```bash
cd ~/.hermes/skills/web/wechat-article-extraction-pro
git add .
git commit -m "v2.0.x: 描述"
git push origin main
```

**Python 源码更新：**
```bash
cd /tmp/wechat-article-for-ai-pro
git pull origin main  # 先同步远程
git add .
git commit -m "v1.x: 描述"
git push origin main
```

### 快速诊断命令

```bash
# 检查 Skill 是否存在
ls ~/.hermes/skills/web/wechat-article-extraction-pro/

# 检查 Python 项目是否存在
ls /tmp/wechat-article-for-ai-pro/

# 检查 Git 状态
cd ~/.hermes/skills/web/wechat-article-extraction-pro && git status
cd /tmp/wechat-article-for-ai-pro && git status

# 查看技能列表
hermes skills list web
```

---

## v2.1 新增：飞书多维表格集成（内容管理）

### 功能概述
将提取的公众号文章自动同步到飞书多维表格，支持选题管理和二创工作流。

### 完整流程图（v2.1）

```
用户提供 URL
    ↓
┌─────────────────────────────────────────────────────────┐
│  阶段 1：工具提取（本地执行）                              │
│  1. 启动 Camoufox 浏览器访问文章                          │
│  2. 提取标题、作者、正文内容                               │
│  3. OCR 识别（RapidOCR）                                  │
│  4. 生成 5 文件输出                                        │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│  阶段 2：Hermes 总结（AI 处理）                           │
│  1. 读取 article-ocr.md 的 OCR 内容                      │
│  2. 结构化分析并生成总结                                   │
│  3. 回填到 article-ocr.md 第三部分                        │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│  阶段 3：飞书 Base 同步（内容管理）                        │
│  1. 提取关键字段（标题、公众号、投递方式等）                │
│  2. 调用 Lark CLI 录入多维表格                             │
│  3. 标记状态为"待处理"                                     │
└─────────────────────────────────────────────────────────┘
    ↓
完成（可在飞书表格中进行选题和二创管理）
```

### 前置条件

1. **安装 Lark CLI**
```bash
npm install -g @larksuite/cli
export PATH="$HOME/.npm-global/lib/node_modules/@larksuite/cli/bin:$PATH"
```

2. **配置 Lark CLI 认证**
```bash
lark-cli config init --app-id "cli_xxx" --app-secret-stdin
lark-cli doctor  # 验证配置
```

3. **创建 Base（首次使用）**
```bash
# 创建 Base
lark-cli base +base-create --name "公众号文章选题库" --as bot

# 创建表格
lark-cli base +table-create --base-token "YOUR_BASE_TOKEN" --name "文章列表" --as bot

# 添加字段（参考下方字段结构）
```

### Base 字段结构

| 字段 | 类型 | 说明 |
|------|------|------|
| 文章标题 | 文本 | 主标题 |
| 公众号 | 文本 | 来源账号 |
| 发布时间 | 日期 | 原文发布时间 |
| 文章内容概要总结 | 文本 | AI生成的结构化总结 |
| 投递方式 | 文本 | 简历投递渠道 |
| 更新时间 | 日期 | 数据录入时间 |
| 文章链接 | 文本 | 原文URL |
| 行业/领域 | 单选 | 互联网/金融/能源/传媒等 |
| 岗位类型 | 单选 | 实习/校招/社招/兼职 |
| 工作地点 | 单选 | 城市列表 |
| 学历要求 | 单选 | 本科/硕士/博士/不限 |
| 截止日期 | 日期 | 招聘截止 |
| 优先级 | 单选 | 高/中/低 |
| 状态 | 单选 | 待处理/已选题/撰写中/已二创/已发布 |
| 标签 | 单选 | 热门/急招/大厂/国企/外企/可内推 |
| 备注 | 文本 | 编辑备注 |

### 标准同步流程（含字段检查）

```python
import subprocess
import json
import os
from datetime import datetime

def sync_article_to_feishu(article_dir: str) -> dict:
    """
    完整的文章同步流程，包含字段检查和自动补充
    
    Args:
        article_dir: 文章输出目录路径（如 ~/.hermes/output/文章标题/）
        
    Returns:
        dict: 包含 success, record_id, missing_fields 的结果
    """
    # 1. 读取metadata.json
    metadata_path = os.path.join(article_dir, 'metadata.json')
    with open(metadata_path, 'r', encoding='utf-8') as f:
        metadata = json.load(f)
    
    # 2. 读取article-ocr.md获取OCR内容
    ocr_path = os.path.join(article_dir, 'article-ocr.md')
    with open(ocr_path, 'r', encoding='utf-8') as f:
        ocr_content = f.read()
    
    # 3. 准备基础数据（从metadata提取）
    base_token = "E9y1bxjHGa9LeGs9q3Tc3J41nmf"
    table_id = "tblYIqHtHrWUlVnP"
    
    # 格式化发布时间
    published_at = metadata.get('published_at', '')
    if published_at:
        # 尝试解析并格式化为 YYYY/MM/DD
        try:
            dt = datetime.strptime(published_at, "%Y-%m-%d %H:%M:%S")
            formatted_date = dt.strftime("%Y/%m/%d")
        except:
            formatted_date = published_at
    else:
        formatted_date = "/"
    
    # 4. 构建完整记录数据
    record_data = {
        # ❗ 必填字段（从metadata提取）
        "文章标题": metadata.get('title', ''),
        "公众号": metadata.get('author', ''),
        "发布时间": formatted_date,
        "文章链接": metadata.get('url', ''),
        
        # AI分析字段（从OCR内容提取）
        "行业": analyze_industry(ocr_content),  # 需要实现
        "领域": analyze_field(ocr_content),
        "岗位类型": analyze_job_types(ocr_content),
        "工作地点": analyze_location(ocr_content),
        "学历要求": analyze_education(ocr_content),
        "截止日期": analyze_deadline(ocr_content),
        "投递方式": analyze_apply_method(ocr_content),
        "原文亮点": analyze_highlights(ocr_content),
        "文章概要": generate_summary(ocr_content),
        "选题方向": determine_topic_direction(ocr_content),
        
        # 固定值字段
        "素材状态": "待选题",
        "文章来源": "链接",
        "适配账号": match_accounts(ocr_content),
        "优先级": "中",
        "标签": analyze_tags(ocr_content),
        "采集时间": int(datetime.now().timestamp() * 1000),
    }
    
    # 5. 检查必填字段
    required_fields = ['文章标题', '公众号', '发布时间', '文章链接']
    missing_fields = [f for f in required_fields if not record_data.get(f) or record_data.get(f) == '/']
    
    if missing_fields:
        print(f"⚠️ 警告：以下必填字段缺失或为空: {', '.join(missing_fields)}")
    
    # 6. 写入临时文件并同步
    with open('sync_data.json', 'w', encoding='utf-8') as f:
        json.dump(record_data, f, ensure_ascii=False)
    
    try:
        cmd = (
            f'lark-cli base +record-upsert '
            f'--base-token {base_token} '
            f'--table-id {table_id} '
            f'--json @sync_data.json '
            f'--as bot'
        )
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
        
        if result.returncode == 0:
            response = json.loads(result.stdout)
            if response.get('ok'):
                record_id = response['data']['record']['record_id_list'][0]
                print(f"✅ 同步成功！记录ID: {record_id}")
                if missing_fields:
                    print(f"⚠️ 但以下字段缺失，建议后续补充: {', '.join(missing_fields)}")
                return {
                    'success': True,
                    'record_id': record_id,
                    'missing_fields': missing_fields
                }
            else:
                return {
                    'success': False,
                    'error': response.get('error'),
                    'missing_fields': missing_fields
                }
        else:
            return {
                'success': False,
                'error': result.stderr,
                'missing_fields': missing_fields
            }
    finally:
        if os.path.exists('sync_data.json'):
            os.remove('sync_data.json')

# 使用示例
result = sync_article_to_feishu("~/.hermes/output/Peet's第二届暑期实习项目火热招聘中！/")
if result['success']:
    print(f"记录ID: {result['record_id']}")
    if result['missing_fields']:
        print(f"缺失字段: {result['missing_fields']}")
        # 可以选择立即补充
        # update_missing_fields(result['record_id'], metadata)
```

### 字段检查清单

同步前必须确认以下字段：

| 优先级 | 字段 | 来源 | 检查方法 |
|--------|------|------|----------|
| P0 | 文章标题 | metadata.json | `metadata.get('title')` |
| P0 | 公众号 | metadata.json | `metadata.get('author')` |
| P0 | 发布时间 | metadata.json | `metadata.get('published_at')` → 格式化为 YYYY/MM/DD |
| P0 | 文章链接 | metadata.json | `metadata.get('url')` |
| P1 | 行业 | AI分析 | 根据内容判断 |
| P1 | 岗位类型 | AI分析 | 实习/校招/社招/兼职 |
| P1 | 工作地点 | AI提取 | 从OCR内容提取 |
| P2 | 其他字段 | AI分析 | 截止日期、投递方式等 |

**注意**：P0字段缺失时必须警告，可选择立即补充或后续更新。

使用 Python 脚本标准化同步流程：

```python
import subprocess
import json
import os

def sync_to_feishu(record_data: dict) -> dict:
    """
    同步单条记录到飞书Base
    
    Args:
        record_data: 符合Base字段格式的字典
        
    Returns:
        dict: 包含 success, record_id, error 的结果
    """
    base_token = "E9y1bxjHGa9LeGs9q3Tc3J41nmf"
    table_id = "tblYIqHtHrWUlVnP"
    
    # 写入临时文件（必须使用相对路径）
    with open('sync_data.json', 'w', encoding='utf-8') as f:
        json.dump(record_data, f, ensure_ascii=False)
    
    try:
        # 执行同步命令
        result = subprocess.run(
            f'lark-cli base +record-upsert --base-token {base_token} --table-id {table_id} --json @sync_data.json --as bot',
            shell=True, capture_output=True, text=True
        )
        
        # 解析结果
        if result.returncode == 0:
            response = json.loads(result.stdout)
            if response.get('ok'):
                record_id = response['data']['record']['record_id_list'][0]
                return {'success': True, 'record_id': record_id}
            else:
                return {'success': False, 'error': response.get('error')}
        else:
            return {'success': False, 'error': result.stderr}
    finally:
        # 清理临时文件
        if os.path.exists('sync_data.json'):
            os.remove('sync_data.json')

# 使用示例
record = {
    "文章标题": "文章标题",
    "行业": "消费品",  # 单选字段直接使用字符串
    "岗位类型": ["实习"],  # 多选字段使用数组
    # ... 其他字段
}

result = sync_to_feishu(record)
if result['success']:
    print(f"同步成功！记录ID: {result['record_id']}")
else:
    print(f"同步失败: {result['error']}")
```

### 更新已有记录

当需要补充或修改已同步记录时，使用 `--record-id` 参数：

```python
import subprocess
import json
import os

def update_feishu_record(record_id: str, update_data: dict) -> dict:
    """
    更新飞书Base中已有的记录
    
    Args:
        record_id: 记录ID（如 recvhq1MWUhyc5）
        update_data: 要更新的字段字典
        
    Returns:
        dict: 包含 success, record_id, error 的结果
    """
    base_token = "E9y1bxjHGa9LeGs9q3Tc3J41nmf"
    table_id = "tblYIqHtHrWUlVnP"
    
    # 写入临时文件
    with open('update_data.json', 'w', encoding='utf-8') as f:
        json.dump(update_data, f, ensure_ascii=False)
    
    try:
        # 执行更新命令（添加 --record-id 参数）
        cmd = (
            f'lark-cli base +record-upsert '
            f'--base-token {base_token} '
            f'--table-id {table_id} '
            f'--record-id {record_id} '  # 指定要更新的记录
            f'--json @update_data.json '
            f'--as bot'
        )
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
        
        if result.returncode == 0:
            response = json.loads(result.stdout)
            if response.get('ok'):
                return {'success': True, 'record_id': record_id}
            else:
                return {'success': False, 'error': response.get('error')}
        else:
            return {'success': False, 'error': result.stderr}
    finally:
        if os.path.exists('update_data.json'):
            os.remove('update_data.json')

# 使用示例：补充缺失字段
record_id = "recvhq1MWUhyc5"  # Peet's的记录ID
update_data = {
    "文章标题": "Peet's第二届暑期实习项目火热招聘中！",
    "公众号": "皮爷招聘",
    "文章链接": "https://mp.weixin.qq.com/s/xxx"
}

result = update_feishu_record(record_id, update_data)
if result['success']:
    print(f"更新成功！记录ID: {result['record_id']}")
else:
    print(f"更新失败: {result['error']}")
```

**常见更新场景**：
1. **补充缺失字段**：初次同步时某些字段未填写，后续补充
2. **修正错误数据**：发现同步的数据有误，需要更正
3. **更新状态**：如从"待选题"更新为"已选题"
4. **添加标签**：为文章添加或修改标签
    print(f"同步失败: {result['error']}")
```

### 批量同步多篇文章

```python
import subprocess
import json
import os

def batch_sync_to_feishu(records: list) -> list:
    """
    批量同步多篇文章到飞书Base
    
    Args:
        records: 记录数据列表
        
    Returns:
        list: 每条记录的同步结果
    """
    results = []
    for i, record in enumerate(records):
        print(f"正在同步第 {i+1}/{len(records)} 篇文章...")
        result = sync_to_feishu(record)
        results.append(result)
        
        if result['success']:
            print(f"  ✅ 成功: {result['record_id']}")
        else:
            print(f"  ❌ 失败: {result['error']}")
    
    return results

# 使用示例
articles = [
    {"文章标题": "文章1", "行业": "互联网", "岗位类型": ["实习"]},
    {"文章标题": "文章2", "行业": "金融", "岗位类型": ["校招"]},
    {"文章标题": "文章3", "行业": "能源", "岗位类型": ["实习"]},
]

results = batch_sync_to_feishu(articles)
success_count = sum(1 for r in results if r['success'])
print(f"\n同步完成: {success_count}/{len(articles)} 篇成功")
```

### 命令行直接录入（备用）

```bash
export PATH="$HOME/.npm-global/lib/node_modules/@larksuite/cli/bin:$PATH"

# 先创建数据文件
cat > data.json << 'EOF'
{
  "文章标题": "文章标题",
  "行业": "消费品",
  "岗位类型": ["实习"],
  "工作地点": "/",
  "学历要求": "本科",
  "截止日期": "/",
  "投递方式": "扫码投递",
  "原文亮点": "亮点内容",
  "文章概要": "概要内容",
  "选题方向": "消费品行业实习机会",
  "素材状态": "待选题",
  "文章来源": "链接",
  "适配账号": ["Joblinker"],
  "优先级": "中",
  "标签": ["大厂"],
  "采集时间": 1752571200000
}
EOF

# 执行同步
lark-cli base +record-upsert \
  --base-token "E9y1bxjHGa9LeGs9q3Tc3J41nmf" \
  --table-id "tblYIqHtHrWUlVnP" \
  --json @data.json \
  --as bot

# 清理临时文件
rm data.json
```

### 工作流建议

**二创管理流程：**
1. **待处理** → 新提取的文章默认状态
2. **已选题** → 编辑确认要二创的文章
3. **撰写中** → 正在撰写二创内容
4. **已二创** → 二创完成，待发布
5. **已发布** → 已发布到目标平台

**筛选视图：**
- 按行业筛选（金融/能源/传媒）
- 按岗位类型筛选（实习/校招/社招）
- 按状态筛选（待处理/已选题/已发布）
- 按优先级筛选（高/中/低）

### 自动化脚本（可选）

创建 `scripts/sync_to_feishu.sh`：

```bash
#!/bin/bash
# 将提取的文章同步到飞书 Base

BASE_TOKEN="YOUR_BASE_TOKEN"
TABLE_ID="YOUR_TABLE_ID"
ARTICLE_DIR="$1"

# 读取 metadata
TITLE=$(cat "$ARTICLE_DIR/metadata.json" | jq -r '.title')
AUTHOR=$(cat "$ARTICLE_DIR/metadata.json" | jq -r '.author')
URL=$(cat "$ARTICLE_DIR/metadata.json" | jq -r '.url')

# 读取总结内容（前500字）
SUMMARY=$(head -c 500 "$ARTICLE_DIR/article-ocr.md")

# 录入 Base
lark-cli base +record-upsert \
  --base-token "$BASE_TOKEN" \
  --table-id "$TABLE_ID" \
  --json "{
    \"文章标题\": \"$TITLE\",
    \"公众号\": \"$AUTHOR\",
    \"文章内容概要总结\": \"$SUMMARY\",
    \"文章链接\": \"$URL\"
  }" \
  --as bot
```

---

## 飞书 Base 内容管理

### 表格配置

| 属性 | 值 |
|------|-----|
| **Base 名称** | 公众号文章选题库 |
| **Base 链接** | https://rqtvt0xmrql.feishu.cn/base/E9y1bxjHGa9LeGs9q3Tc3J41nmf |
| **Base Token** | `E9y1bxjHGa9LeGs9q3Tc3J41nmf` |
| **Table ID** | `tblYIqHtHrWUlVnP` |

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| 文章标题 | 文本 | 主标题 |
| 公众号 | 文本 | 来源账号 |
| 发布时间 | 日期 | 原文发布时间 |
| 文章内容概要总结 | 文本 | AI生成的结构化总结 |
| 投递方式 | 文本 | 简历投递渠道 |
| 文章链接 | 文本 | 原文URL |
| **行业** | **单选** | **互联网/金融/能源/传媒/制造/消费品/其他** |
| **领域** | **单选** | **科技/投资/电力/娱乐/互联网/零售/其他** |
| 岗位类型 | 单选 | 实习/校招/社招/兼职 |
| 工作地点 | 单选 | 城市列表 |
| 学历要求 | 单选 | 本科/硕士/博士/不限 |
| 截止日期 | 日期 | 招聘截止 |
| 优先级 | 单选 | 高/中/低 |
| 状态 | 单选 | 待处理/已选题/撰写中/已二创/已发布 |
| 标签 | 单选 | 热门/急招/大厂/国企/外企/可内推 |
| **亮点** | **文本** | **招聘信息亮点总结** |

### 字段值格式（重要）

**单选字段格式**：直接使用字符串值，**不要**包装成 `{"text": "值"}`

| 字段类型 | 正确格式 | 错误格式 |
|---------|---------|---------|
| 文本 | `"文章标题": "标题内容"` | - |
| 单选 | `"行业": "消费品"` | `{"行业": {"text": "消费品"}}` ❌ |
| 多选 | `"标签": [{"text": "大厂"}]` | - |

**空字段填充规则**：信息未获取到的字段，统一填充 `"/"`

| 场景 | 处理方式 | 示例 |
|------|---------|------|
| 有明确信息 | 填充实际值 | `"工作地点": "深圳"` |
| 信息未提及/不确定 | **填充 `"/"`** | `"截止日期": "/"` |
| 字段不适用 | **填充 `"/"`** | `"亮点": "/"` |

> **目的**：区分"确实没有信息" vs "可能解析出了问题"
> 
> **禁止**：留空字符串或省略字段，必须显式标记 `"/"`

**完整示例**：
```bash
lark-cli base +record-upsert \
  --base-token "E9y1bxjHGa9LeGs9q3Tc3J41nmf" \
  --table-id "tblYIqHtHrWUlVnP" \
  --json '{
    "文章标题": "珀莱雅2026校招启动",
    "公众号": "珀莱雅招聘",
    "行业": "消费品",
    "领域": "消费品",
    "岗位类型": "校招",
    "工作地点": "杭州",
    "学历要求": "本科",
    "截止日期": "/",
    "优先级": "中",
    "状态": "待处理",
    "标签": "大厂",
    "亮点": "九大品牌矩阵，实习生有留用机会"
  }' \
  --as bot
```
| **最后更新时间** | **系统字段** | **自动记录最后修改时间** |

### 二创工作流

```
待处理 → 已选题 → 撰写中 → 已二创 → 已发布
```

**筛选视图建议：**
- 按「状态」筛选：查看待处理文章
- 按「优先级」筛选：优先处理高优先级
- 按「行业」筛选：互联网/金融/能源等
- 按「领域」筛选：科技/投资/电力/娱乐等
- 按「岗位类型」筛选：实习/校招/社招

### 自动录入脚本

```bash
#!/bin/bash
# 将提取的文章同步到飞书 Base

BASE_TOKEN="E9y1bxjHGa9LeGs9q3Tc3J41nmf"
TABLE_ID="tblYIqHtHrWUlVnP"
ARTICLE_DIR="$1"

# 读取 metadata
TITLE=$(cat "$ARTICLE_DIR/metadata.json" | jq -r '.title')
AUTHOR=$(cat "$ARTICLE_DIR/metadata.json" | jq -r '.author')
URL=$(cat "$ARTICLE_DIR/metadata.json" | jq -r '.url')
PUBLISHED=$(cat "$ARTICLE_DIR/metadata.json" | jq -r '.published_at')

# 读取总结内容（前500字）
SUMMARY=$(grep -A 50 "### 招聘概览\|### 公司介绍\|### 招募概览" "$ARTICLE_DIR/article-ocr.md" | head -c 500)

# 录入 Base
export PATH="$HOME/.npm-global/lib/node_modules/@larksuite/cli/bin:$PATH"
lark-cli base +record-upsert \
  --base-token "$BASE_TOKEN" \
  --table-id "$TABLE_ID" \
  --json "{
    \"文章标题\": \"$TITLE\",
    \"公众号\": \"$AUTHOR\",
    \"发布时间\": \"$PUBLISHED\",
    \"文章内容概要总结\": \"$SUMMARY\",
    \"文章链接\": \"$URL\",
    \"状态\": {\"text\": \"待处理\"}
  }" \
  --as bot
```

---

## 关键经验教训（必读）

### 1. 飞书 Base 字段格式陷阱

**发现过程**：多次尝试后发现 API 报错 "Match one of the supported request payload shapes exactly"

**正确格式**：
```bash
# ✅ 正确：单选字段直接使用字符串
{"行业": "消费品", "领域": "科技"}

# ❌ 错误：不要包装成对象格式
{"行业": {"text": "消费品"}}  # 会导致验证失败
```

**适用规则**：
| 字段类型 | 格式 | 示例 |
|---------|------|------|
| 文本 | 直接字符串 | `"文章标题": "标题"` |
| 单选 | **直接字符串** | `"行业": "消费品"` ✅ |
| 多选 | 对象数组 | `"标签": [{"text": "大厂"}]` |

### 2. 空字段填充规则

**用户明确要求**：信息未获取到的字段，统一填充 `"/"`

**目的**：区分"确实没有信息" vs "可能解析出了问题"

**执行标准**：
| 场景 | 处理方式 | 示例 |
|------|---------|------|
| 有明确信息 | 填充实际值 | `"工作地点": "深圳"` |
| 信息未提及 | **填充 `"/"`** | `"截止日期": "/"` |
| 信息不确定 | **填充 `"/"`** | `"学历要求": "/"` |
| 字段不适用 | **填充 `"/"`** | `"亮点": "/"` |

**禁止**：留空字符串、null、或省略字段

### 3. 上下文压缩处理

**问题**：对话达到358轮时，上下文被压缩，导致"失忆"

**解决方案**：
1. **Memory存储**：关键配置写入长期Memory（飞书Base Token、字段格式规则）
2. **标记文件**：创建 `~/.hermes/.wechat_workflow_state` 保存任务状态
3. **定期提醒**：每50轮对话主动提醒保存进度

**用户指令**：
- "保存当前进度" → 立即更新标记文件
- "查看状态" → 读取标记文件汇报

### 4. 双仓库管理策略

| 仓库 | 用途 | 更新时机 |
|------|------|----------|
| `wechat-article-for-ai-pro` | Python提取工具 | 功能迭代 |
| `wechat-article-extraction-pro-skill` | Hermes调用指南 | 流程优化 |

**为什么分离**：不同迭代周期，避免互相干扰

---

### 原文亮点提炼规范（v2.6 新增）

**核心原则**：突出吸引学生查看和投递的关键点，按优先级排序

#### 关键词优先级

| 优先级 | 类型 | 关键词 | 示例 |
|--------|------|--------|------|
| **P0** | 硬性福利 | 包食宿、提供住宿、六险二金、补充医疗 | `"包食宿"`、`"六险二金"` |
| **P1** | 发展机会 | 可转正、提前锁定Offer、留用机会、实习转正 | `"可转正"`、`"提前获Offer"` |
| **P2** | 企业背书 | 央企A级、国企、大厂、世界500强、行业第一 | `"央企A级"`、`"国企"`、`"大厂"` |
| **P3** | 流程优势 | 免笔试、直通面试、无笔试、快速通道 | `"免笔试"`、`"直通面试"` |
| **P4** | 特殊政策 | 落户政策、优先落户、北京户口、上海户口 | `"可落户"`、`"优先落户"` |
| **P5** | 薪资优势 | 薪资上浮、竞争力薪酬、高薪、待遇优厚 | `"薪资上浮"`、`"待遇优厚"` |

#### 亮点格式规范

1. **使用分号分隔**，每个亮点独立成句
2. **关键词前置**，吸引眼球的内容放在前面
3. **控制长度**，总字数不超过50字
4. **避免冗余**，不重复同类信息

#### 优化示例

| 原标题亮点 | 优化后 |
|-----------|--------|
| 提前锁定27届校招Offer；六大招聘方向；近4000件授权专利；终端网点14万家；2016年上市 | **可转正**；**大厂**；**包食宿**；六大方向；4000+专利 |
| 提前批次录用机会；国家卓越工程师团队；全球五大电力保护设备企业之一；提供住宿；专业导师辅导 | **央企背景**；**可转正**；**包食宿**；国家卓越工程师团队 |
| 免笔试直通面试；管理规模20亿；三种实习类型(校招/暑期/日常)；六险二金；高频量化策略 | **免笔试**；**可转正**；**六险二金**；三种实习类型 |
| 央企A级；清洁能源装机世界第一(2.88亿千瓦)；六险二金；落户政策；68家二级单位；业务覆盖47个国家 | **央企A级**；**可落户**；**六险二金**；**包食宿**；清洁能源世界第一 |
| 11个组别招募（编剧/舞台/真人秀/统筹/音乐/艺人/听审/宣传/视觉/技术/后期）；实习到8月中；湖南长沙；大厂实习机会 | **大厂实习**；**可转正**；11个组别；实习到8月中 |

#### 提取流程

```
阅读OCR内容
    ↓
识别福利关键词（P0-P1优先）
    ↓
识别企业背书（P2）
    ↓
识别流程/政策优势（P3-P4）
    ↓
按优先级排序，去重
    ↓
生成亮点字符串（分号分隔）
```

#### 完整示例

**欧普照明**：
```
原文信息：
- 提前锁定27届校招Offer
- 六大招聘方向(研发/产品/算法/营销/品牌/职能)
- 近4000件授权专利
- 终端网点14万家
- 2016年上市
- 上海总部及多个生产办公园区

优化后亮点：
"可转正；大厂；包食宿；六大方向；4000+专利"
```

**南瑞继保**：
```
原文信息：
- 提前批次正式录用的机会
- 国家卓越工程师团队
- 全球五大电力保护设备企业之一
- 提供住宿
- 专业导师辅导
- 具有竞争力的实习工资

优化后亮点：
"央企背景；可转正；包食宿；国家卓越工程师团队"
```

---

## 飞书 Base 字段填充规范（v2.5 新增）

### 字段填充逻辑

| 字段 | 类型 | 填充规则 | 示例 |
|------|------|---------|------|
| 文章标题 | 文本 | 从metadata提取 | `"实习 \| 欧普照明27届实习生招聘正式开启！"` |
| 公众号 | 文本 | 从metadata提取 | `"欧普照明微招聘"` |
| 发布时间 | 日期 | 从metadata提取 | `"2026-04-21 15:01:39"` |
| 文章链接 | 文本 | 从metadata提取 | `"https://mp.weixin.qq.com/s/xxx"` |
| 行业 | 单选 | AI分析文章内容判断 | `"消费品"` / `"金融"` / `"能源"` |
| 领域 | 单选 | AI分析文章内容判断 | `"消费品"` / `"投资"` / `"电力"` |
| 岗位类型 | 多选 | AI分析招聘类型，**最多2个** | `["实习"]` / `["校招"]` / `["实习", "校招"]` |
| 工作地点 | 单选 | AI提取地点信息 | `"上海"` / `"北京"` / `"深圳"` |
| 学历要求 | 单选 | AI提取学历要求 | `"本科"` / `"硕士"` / `"博士"` |
| 截止日期 | 日期 | AI提取截止时间，无则填`"/"` | `"2026-05-31"` / `"/"` |
| 投递方式 | 文本 | AI提取投递渠道，无则填`"/"` | `"关注公众号在线投递"` / `"/"` |
| 原文亮点 | 文本 | AI总结核心亮点 | `"提前锁定Offer；六大招聘方向；近4000件专利"` |
| 文章概要 | 文本 | AI生成结构化总结（500字内） | 公司概况+招聘亮点+关键信息 |
| 选题方向 | 文本 | AI根据行业和账号匹配生成 | `"消费品行业实习机会"` |
| 素材状态 | 单选 | 固定值 `"待选题"` | `"待选题"` |
| 文章来源 | 单选 | 固定值 `"链接"` | `"链接"` |
| 适配账号 | 多选 | 根据行业/领域/岗位类型匹配，**规则见下方** | `["Joblinker"]` / `["研究生求职圈", "行研实习"]` |
| 优先级 | 单选 | 默认 `"中"`，重要可标 `"高"` | `"中"` |
| 标签 | 多选 | AI判断标签，**最多3个** | `["央企", "大厂", "六险二金"]` |
| 采集时间 | 日期 | 当前时间Unix时间戳（毫秒） | `1776756180000` |

### 空字段填充规则

**必须填充 `"/"` 的字段**：
- 截止日期（未明确提及）
- 投递方式（未明确提及）
- 原文亮点（无法提取）
- 工作地点（未明确提及）
- 学历要求（未明确提及）

**目的**：区分"确实没有信息" vs "可能解析出了问题"

**禁止**：留空字符串、null、或省略字段

### 日期字段格式

**飞书Base日期字段需使用Unix时间戳（毫秒）**：

```python
from datetime import datetime

# 采集时间
dt = datetime.now()
timestamp_ms = int(dt.timestamp() * 1000)
# 结果: 1776756180000

# API调用
lark-cli api PUT /open-apis/bitable/v1/apps/.../records/RECORD_ID \
  --data '{"fields": {"采集时间": 1776756180000}}' --as bot
```

### 适配账号匹配规则（v2.10 新增）

根据文章内容和账号定位，自动匹配适配账号：

| 账号 | 定位 | 匹配规则 |
|------|------|----------|
| **Joblinker** | 实习岗位为主 | **仅当文章包含实习岗位时适配** |
| **研究生求职圈** | 校招岗位为主 | **校招文章优先适配** |
| **行研实习** | 行研/金融实习 | 金融/投资/行研相关实习岗位 |

**匹配逻辑**：
1. **纯实习文章** → 适配 `["Joblinker"]`
2. **纯校招文章** → 适配 `["研究生求职圈"]`
3. **实习+校招混合文章** → 根据内容侧重判断：
   - 侧重实习 → `["Joblinker", "研究生求职圈"]`
   - 侧重校招 → `["研究生求职圈", "Joblinker"]`
4. **金融/投资/行研类实习** → 同时适配 `["行研实习"]`

**示例**：
| 文章类型 | 适配账号 |
|---------|----------|
| 某大厂暑期实习招聘 | `["Joblinker"]` |
| 某央企2026校招启动 | `["研究生求职圈"]` |
| 某券商实习+校招混合招聘 | `["Joblinker", "研究生求职圈"]` 或 `["研究生求职圈", "Joblinker"]` |
| 某基金公司行研实习招聘 | `["Joblinker", "行研实习"]` |
| 某投行校招+实习 | `["研究生求职圈", "行研实习"]` |

| 类型 | 格式 | 示例 |
|------|------|------|
| 单选 | 字符串 | `"行业": "消费品"` |
| 多选 | 字符串数组 | `"适配账号": ["Joblinker"]` |

### 完整录入示例

```bash
export PATH="$HOME/.npm-global/lib/node_modules/@larksuite/cli/bin:$PATH"

lark-cli base +record-upsert \
  --base-token "E9y1bxjHGa9LeGs9q3Tc3J41nmf" \
  --table-id "tblYIqHtHrWUlVnP" \
  --json '{
    "文章标题": "实习 | 欧普照明27届实习生招聘正式开启！",
    "公众号": "欧普照明微招聘",
    "发布时间": "2026-04-21 15:01:39",
    "文章链接": "https://mp.weixin.qq.com/s/rQVzVfl4FJ8tj74yI8YLuw",
    "行业": "消费品",
    "领域": "消费品",
    "岗位类型": "实习",
    "工作地点": "上海",
    "学历要求": "本科",
    "截止日期": "/",
    "投递方式": "关注\"欧普照明微招聘\"公众号，点击\"加入欧普-实习招聘\"在线投递",
    "原文亮点": "提前锁定27届校招Offer；六大招聘方向(研发/产品/算法/营销/品牌/职能)；近4000件授权专利；终端网点14万家；2016年上市",
    "文章概要": "欧普照明27届实习生招聘，面向2026年11月-2027年10月毕业生。招聘研发类、产品类、算法类、营销类、品牌类、职能类六大方向。4月21日启动网申，5月7日起AI面试，实习表现优异者可提前获得校招Offer。",
    "选题方向": "消费品行业实习机会",
    "素材状态": "待选题",
    "文章来源": "链接",
    "适配账号": ["Joblinker"],
    "优先级": "中",
    "标签": "大厂",
    "采集时间": 1776756180000
  }' \
  --as bot
```

### 更新已有记录

使用 `api PUT` 更新特定字段：

```bash
# 更新采集时间
lark-cli api PUT /open-apis/bitable/v1/apps/E9y1bxjHGa9LeGs9q3Tc3J41nmf/tables/tblYIqHtHrWUlVnP/records/RECORD_ID \
  --data '{"fields": {"采集时间": 1776756180000}}' \
  --as bot

# 更新投递方式
lark-cli api PUT /open-apis/bitable/v1/apps/E9y1bxjHGa9LeGs9q3Tc3J41nmf/tables/tblYIqHtHrWUlVnP/records/RECORD_ID \
  --data '{"fields": {"投递方式": "关注公众号在线投递"}}' \
  --as bot
```

---

## 版本历史

| 版本 | 变更 |
|------|------|
| v1.0 | 标准4文件输出 |
| v1.2 | 添加 article-ocr.md 占位符 |
| v1.3 | AI Vision OCR 集成 |
| v1.4 | 集成 RapidOCR 本地识别 |
| v1.5 | 可配置 OCR 引擎（rapidocr/vision/auto）|
| v1.5.1 | 修复 RGBA 图片切片保存问题 |
| v1.6 | Hybrid 模式：工具提取+OCR，Hermes 总结回填 |
| v2.0 | 标准 Skill 结构，Git 版本控制 |
| v2.1 | **新增飞书 Base 集成**，支持内容管理和二创工作流 |
| v2.2 | 添加飞书Base字段值格式说明 |
| v2.3 | 添加定期状态提醒机制 |
| v2.4 | **添加空字段填充规则**，明确"/"标记规范 |
| v2.5 | **添加字段填充逻辑规范**，明确采集时间/二创深度等默认值 |
| v2.8 | **删除字段**：移除"状态"和"二创深度"字段 |
| v2.9 | **更新标签字段**：改为多选，最多可选3个 |
| v2.10 | **更新岗位类型字段**：改为多选（最多2个）；**新增适配账号匹配规则** |
| v2.11 | **添加飞书Base同步标准方法**：sync_to_feishu()函数、批量同步、命令行方案 |
| v2.12 | **添加飞书同步指南文档** references/feishu-sync-guide.md |
| v2.13 | **添加更新已有记录方法**：update_feishu_record()函数，支持补充缺失字段和修正数据 |
| v2.14 | **添加完整字段检查流程**：sync_article_to_feishu()函数，包含22个字段的必填检查清单 |
| v2.15 | **添加二维码识别功能说明**：v1.7 新增 OpenCV QRCodeDetector，智能分类招聘链接 |
| v2.16 | **更新第四章结构说明**：展示第四章新格式（原文+OCR+二维码） |
| v2.17 | **集成 zbar-py 说明**：v1.7.2 新增 zbar-py 支持，解决复杂背景二维码识别问题 |