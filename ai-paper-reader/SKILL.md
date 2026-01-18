---
name: ai-paper-reader
description: 深度解析AI论文的创新点与实现细节，提取关键图表，帮助理解核心思想并迁移到实际工程项目中
---

# AI 论文深度解析器

## MCP 工具配置（推荐）

为了获得最佳体验，建议配置以下 MCP 工具：

### 1. PDF 直接读取 - 让 Claude 直接解析 PDF

在 `~/.claude/settings.json` 或项目 `.claude/settings.json` 中添加：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@anthropic/mcp-server-filesystem",
        "/path/to/your/papers"
      ]
    }
  }
}
```

**或者使用专用 PDF MCP（更强大）：**

```json
{
  "mcpServers": {
    "pdf-reader": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-pdf"]
    }
  }
}
```

配置后，你可以直接说：
```
请解析 ~/papers/attention-is-all-you-need.pdf
```

### 2. Notion 同步 - 自动保存笔记到 Notion

```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-notion"],
      "env": {
        "NOTION_API_KEY": "your-notion-integration-token"
      }
    }
  }
}
```

**获取 Notion API Key：**
1. 访问 https://www.notion.so/my-integrations
2. 创建新的 Integration
3. 复制 Internal Integration Token
4. 在 Notion 中将目标数据库/页面与 Integration 共享

配置后，你可以说：
```
请解析这篇论文，并将笔记同步到我的 Notion 论文库
```

### 3. 完整配置示例

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-filesystem", "/Users/你的用户名/papers"]
    },
    "notion": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-notion"],
      "env": {
        "NOTION_API_KEY": "secret_xxxxx"
      }
    }
  }
}
```

### MCP 工具使用流程

```
┌─────────────────────────────────────────────────────────────┐
│  用户: 请解析 ~/papers/transformer.pdf 并同步到 Notion      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  1. [filesystem MCP] 读取 PDF 文件内容                      │
│  2. [Claude] 深度解析论文，提取创新点和实现细节              │
│  3. [filesystem MCP] 提取图片到���地目录                     │
│  4. [notion MCP] 创建笔记页面，上传图片，同步内容            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  完成！笔记已同步到 Notion，本地也保留了图片备份             │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心目标

**不是生成展示报告，而是让你真正理解论文并能将其思想应用到实际工作中。**

重点关注：
- 这篇论文到底解决了什么问题？为什么之前的方法解决不了？
- 核心创新是什么？它的本质思想是什么？
- 具体是怎么实现的？关键代码/公式怎么写？
- 关键图表是什么？它们说明了什么？
- 我能在自己的项目中怎么用？

---

## 笔记目录结构

解析论文时，自动创建以下目录结构：

```
paper-notes/
├── [论文名称]/
│   ├── README.md           # 主笔记文件
│   ├── images/             # 提取的图片
│   │   ├── fig1_architecture.png
│   │   ├── fig2_method.png
│   │   ├── fig3_results.png
│   │   └── ...
│   └── tables/             # 关键表格（如需要）
│       └── table1_comparison.md
```

---

## 图片提取与管理

### 自动提取论文图片

```python
import fitz  # PyMuPDF
import os

def extract_images_from_pdf(pdf_path, output_dir):
    """
    从 PDF 中提取所有图片
    """
    os.makedirs(output_dir, exist_ok=True)
    doc = fitz.open(pdf_path)

    image_list = []
    for page_num in range(len(doc)):
        page = doc[page_num]
        images = page.get_images(full=True)

        for img_idx, img in enumerate(images):
            xref = img[0]
            base_image = doc.extract_image(xref)
            image_bytes = base_image["image"]
            image_ext = base_image["ext"]

            # 保存图片
            image_name = f"page{page_num+1}_img{img_idx+1}.{image_ext}"
            image_path = os.path.join(output_dir, image_name)

            with open(image_path, "wb") as f:
                f.write(image_bytes)

            image_list.append({
                "path": image_path,
                "page": page_num + 1,
                "name": image_name
            })

    doc.close()
    return image_list

# 使用示例
images = extract_images_from_pdf("paper.pdf", "./notes/images/")
print(f"提取了 {len(images)} 张图片")
```

### 按页面截取（保留完整布局）

```python
import fitz

def pdf_pages_to_images(pdf_path, output_dir, dpi=150):
    """
    将 PDF 每页转为图片，保留完整布局
    适合截取包含图+标题的完整 Figure
    """
    os.makedirs(output_dir, exist_ok=True)
    doc = fitz.open(pdf_path)

    for page_num in range(len(doc)):
        page = doc[page_num]
        # 设置缩放比例
        zoom = dpi / 72
        mat = fitz.Matrix(zoom, zoom)
        pix = page.get_pixmap(matrix=mat)

        output_path = os.path.join(output_dir, f"page_{page_num+1}.png")
        pix.save(output_path)

    doc.close()

# 使用示例
pdf_pages_to_images("paper.pdf", "./notes/pages/", dpi=200)
```

### 截取指定区域（精确裁剪）

```python
import fitz

def crop_figure_from_pdf(pdf_path, page_num, rect, output_path, dpi=200):
    """
    从 PDF 指定页面裁剪特定区域
    rect: (x0, y0, x1, y1) 坐标，单位为点(pt)
    """
    doc = fitz.open(pdf_path)
    page = doc[page_num - 1]  # 页码从1开始

    # 设置裁剪区域
    clip = fitz.Rect(rect)
    zoom = dpi / 72
    mat = fitz.Matrix(zoom, zoom)

    pix = page.get_pixmap(matrix=mat, clip=clip)
    pix.save(output_path)
    doc.close()

# 使用示例：裁剪第2页的 Figure 1
# 坐标可以通过 PDF 阅读器查看，或者先转成图片再确定
crop_figure_from_pdf(
    "paper.pdf",
    page_num=2,
    rect=(50, 100, 550, 400),  # 左上角(50,100) 到 右下角(550,400)
    output_path="./notes/images/fig1_architecture.png"
)
```

---

## 论文解析框架

### 一、问题定义（简略）

> 快速回答：这篇论文要解决什么问题？

```
问题: [一句话描述]
现有方法的痛点: [为什么之前的方法不够好]
```

---

### 二、核心创新点（重点）

这是论文最核心的部分，需要深度剖析：

#### 2.1 创新点识别

对每个创新点，回答以下问题：

```markdown
## 创新点 1: [名称]

### 本质思想
> 用最直白的话解释：这个创新的核心 idea 是什么？
> 不要用论文的术语，用自己的话重新表述

### 为什么有效
> 从原理层面解释：为什么这个方法能 work？
> 它利用了什么性质/假设/先验知识？

### 与现有方法的本质区别
| 维度 | 之前的方法 | 本文方法 | 为什么更好 |
|------|-----------|---------|-----------|
| [维度1] | | | |
| [维度2] | | | |

### 关键洞察 (Key Insight)
> 作者发现了什么别人没注意到的东西？
> 这个洞察可以迁移到其他问题吗？
```

#### 2.2 创新点之间的关系

```
[创新点1] --解决了--> [问题A] --但引入了--> [问题B] --通过--> [创新点2] 解决
```

---

### 三、实现细节（重点）

#### 3.1 整体架构

```
输入 --> [模块1] --> [模块2] --> ... --> 输出
           |            |
           v            v
      [关键操作]    [关键操作]
```

#### 3.2 核心模块拆解

对每个关键模块：

```markdown
## 模块: [名称]

### 输入输出
- 输入: [shape, 含义]
- 输出: [shape, 含义]

### 核心公式
$$
[公式]
$$

**公式解读:**
- [符号1]: 表示什么，为什么这样设计
- [符号2]: 表示什么，取值范围，敏感度

### 伪代码实现

```python
def core_module(input):
    """
    核心逻辑的简化实现
    """
    # Step 1: [做什么]
    x = operation_1(input)

    # Step 2: [做什么] - 这是关键步骤
    # 为什么这样做: [解释]
    y = key_operation(x)

    # Step 3: [做什么]
    output = operation_3(y)

    return output
```

### 实现中的 tricks
1. **[trick 名称]**: [具体做法] → [为什么有效]
2. **[trick 名称]**: [具体做法] → [为什么有效]

### 常见实现陷阱
- [ ] [陷阱1]: [描述] → [正确做法]
- [ ] [陷阱2]: [描述] → [正确做法]
```

#### 3.3 关键超参数

| 超参数 | 论文取值 | 作用 | 调参建议 |
|--------|---------|------|---------|
| | | | |

#### 3.4 训练细节

```markdown
### 损失函数设计
- 总损失: L = λ1*L1 + λ2*L2 + ...
- L1 的作用: [解释]
- L2 的作用: [解释]
- λ 的设置依据: [解释]

### 训练策略
- 学习率调度: [具体方案]
- 数据增强: [具体方法]
- 其他 tricks: [列举]
```

---

### 四、关键图表解析（重点）

论文中的图表往往是理解的关键，需要逐一分析：

```markdown
## 图表清单

| 图表 | 位置 | 类型 | 重要性 | 本地路径 |
|------|------|------|--------|---------|
| Figure 1 | p.2 | 架构图 | ⭐⭐⭐ | images/fig1_architecture.png |
| Figure 2 | p.4 | 方法流程 | ⭐⭐⭐ | images/fig2_method.png |
| Figure 3 | p.6 | 实验结果 | ⭐⭐ | images/fig3_results.png |
| Table 1 | p.7 | 性能对比 | ⭐⭐ | tables/table1.md |

---

## Figure 1: [标题]

![架构图](images/fig1_architecture.png)

### 图示内容
> 这张图展示了什么？

### 关键组件解读
1. **[组件A]**: 作用是...
2. **[组件B]**: 作用是...
3. **箭头/连接**: 表示...

### 数据流向
```
输入 --> [处理1] --> [处理2] --> 输出
```

### 为什么这样设计
> 这个架构设计的巧妙之处在于...

---

## Figure 2: [标题]

![方法流程](images/fig2_method.png)

### 这张图说明的核心问题
> 作者通过这张图想传达什么？

### 图中的关键信息
- 信息1: ...
- 信息2: ...

### 与正文的对应关系
> 这张图对应论文第X节的内容...
```

### 图片命名规范

为了便于后续检索，建议按以下规则命名：

```
fig{序号}_{类型}_{简述}.png

类型包括:
- arch: 架构图
- method: 方法流程
- result: 实验结果
- ablation: 消融实验
- compare: 对比图
- viz: 可视化示例

示例:
- fig1_arch_overall.png
- fig2_method_attention.png
- fig3_result_main.png
- fig4_ablation_components.png
```

---

### 五、实验洞察（简略但要提炼）

不需要列举所有实验结果，只关注：

```markdown
### 消融实验关键结论
1. [组件A] 贡献了 X% 的提升 → 说明 [什么]
2. [组件B] 在 [场景] 下特别有效 → 说明 [什么]
3. 去掉 [组件C] 后 [现象] → 说明 [什么]

### 边界条件 / 失效场景
- 在 [条件] 下效果不好，因为 [原因]
- 对 [类型] 的数据敏感

### 与 baseline 对比的核心结论
> 不是列数字，而是：为什么比 baseline 好？本质原因是什么？
```

---

### 五、工程落地指南（重点）

这是将论文应用到实际工作的关键：

```markdown
## 复现清单

### 必要组件
- [ ] [组件1]: [复杂度评估: 简单/中等/复杂]
- [ ] [组件2]: [复杂度评估]
- [ ] [组件3]: [复杂度评估]

### 可选组件（锦上添花）
- [ ] [组件]: [带来的提升] vs [实现成本]

### 最小可行版本 (MVP)
> 如果只能实现一个核心 idea，应该实现哪个？
> 预期能获得多少收益？

### 依赖项
- 框架: PyTorch / TensorFlow / JAX
- 关键库: [列举]
- 计算资源: [GPU 需求]

## 迁移到我的项目

### 适用场景判断
我的问题是否满足以下条件：
- [ ] [条件1]: [解释为什么需要]
- [ ] [条件2]: [解释为什么需要]
- [ ] [条件3]: [解释为什么需要]

### 需要适配的部分
| 论文设定 | 我的场景 | 适配方案 |
|---------|---------|---------|
| | | |

### 可能的改进方向
1. [改进1]: 论文没做但可能有效
2. [改进2]: 针对我的场景的特殊优化

### 风险评估
- 实现风险: [高/中/低] - [原因]
- 效果风险: [高/中/低] - [原因]
- 建议: [是否值得尝试，优先级]
```

---

### 六、深度思考

```markdown
## 我的理解

### 这篇论文的本质贡献
> 抛开所有技术细节，这篇论文最核心的贡献是什么？
> 如果要用一个词/一句话总结？

### 方法论层面的启发
> 这篇论文的解题思路对我有什么启发？
> 我能用类似的思路解决其他问题吗？

### 批判性思考
- 论文的假设合理吗？在什么情况下会不成立？
- 实验设置有没有问题？结论可信吗？
- 有没有更简单的方法能达到类似效果？

### 遗留问题
> 读完论文后还有哪些没想通的问题？
1. [问题1]
2. [问题2]

### 延伸阅读
- 要理解这篇论文，需要先读: [论文列表]
- 这篇论文的后续工作: [论文列表]
```

---

## 使用方式

### 深度解析单篇论文
```
请深度解析这篇论文，我想理解它的核心创新和实现细节，
以便将其思想应用到我的 [具体项目/问题] 中。
```

### 带具体问题的解析
```
请解析这篇论文，重点帮我理解：
1. [模块A] 具体是怎么实现的？
2. [技术B] 为什么比之前的方法有效？
3. 如果我想在 [我的场景] 中使用，需要怎么改？
```

### 对比解析
```
请对比解析这两篇论文：
- 它们解决同一个问题的不同思路是什么？
- 各自的优劣势？
- 我应该选哪个作为 baseline？
```

## 输出格式

默认输出 Markdown 格式，适合：
- 直接作为笔记保存
- 导入 Notion / Obsidian 等笔记工具
- 后续检索和回顾

```bash
# 如需导出为其他格式
pandoc notes.md -o notes.pdf
pandoc notes.md -o notes.docx
```

## 依赖安装

```bash
# PDF 文本提取
pip install pdfplumber pypdf

# 图���提取（核心）
pip install pymupdf  # 即 fitz

# 可选：导出格式转换
# macOS
brew install pandoc
# Ubuntu
sudo apt-get install pandoc
```

## 快速开始

```bash
# 1. 创建笔记目录
mkdir -p paper-notes/论文名称/images

# 2. 提取图片
python -c "
import fitz
import os

pdf_path = 'paper.pdf'
output_dir = 'paper-notes/论文名称/images'
os.makedirs(output_dir, exist_ok=True)

doc = fitz.open(pdf_path)
for page_num in range(len(doc)):
    page = doc[page_num]
    for img_idx, img in enumerate(page.get_images(full=True)):
        xref = img[0]
        base = doc.extract_image(xref)
        with open(f'{output_dir}/page{page_num+1}_img{img_idx+1}.{base[\"ext\"]}', 'wb') as f:
            f.write(base['image'])
doc.close()
print('图片提取完成')
"

# 3. 开始解析
# 在 Claude Code 中输入：请深度解析这篇论文 paper.pdf
```