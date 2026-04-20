---
name: wechat-article-extraction-pro
description: >
  微信公众号文章提取 Pro 版完整执行流程。
  基于 wechat-article-for-ai 增强，实现5文件输出 + RapidOCR 识别 + Hermes 总结回填。
  Hybrid 模式：工具负责提取+OCR，Hermes 负责总结。
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
完成
```

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
```

## 输出结构

```
~/.hermes/output/文章标题/
├── article.md              # 基础文字
├── article.html            # 网页格式
├── article-ocr.md          # OCR + 总结 ✅
├── metadata.json           # 元数据
├── images/                 # 原始图片
└── slices/                 # 长图切片（如有）
```

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
