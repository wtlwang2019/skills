---
name: minimax-pdf-cn-font-use
description: |
  生成支持中文显示的 PDF 文档。当用户要求生成 PDF 且内容包含中文时使用。
  触发场景：
  - 用户说"生成中文 PDF"、"创建中文报告"
  - 用户说"PDF 中文显示为空格"、"中文显示不正常"
  - 用户提到使用 minimax-pdf 但中文有问题
  - 用户要求"中文字体"、"中文 PDF 支持"

  本技能不能独立使用，需要配合 minim-pdf 技能工作。
---

# minimax-pdf-cn-font-use

## 概述

本技能为 [minimax-pdf](../minimax-pdf/SKILL.md) 提供**中文字体支持**。

当文档内容以中文为主时，直接使用 minimax-pdf 会导致中文显示为空格（方块），因为默认字体（Helvetica/Times）不支持中文。本技能说明如何通过修改 tokens 配置注入系统中文字体来解决此问题。

## 前置条件

系统已安装中文字体。验证命令：

```bash
fc-list :lang=zh family
```

常用中文字体：
- `WenQuanYi Zen Hei`（文泉驿正黑）- 推荐
- `WenQuanYi Micro Hei`（文泉驿微米黑）
- `Noto Sans CJK SC`（思源黑体）

## 核心问题

minimax-pdf 的 `palette.py` 生成的 tokens.json 中：

```json
{
  "font_display_rl": "Times-Bold",
  "font_body_rl": "Helvetica",
  "font_body_b_rl": "Helvetica-Bold",
  "font_paths": {}
}
```

这些字体是 ReportLab 系统字体，不支持中文。需要替换为支持中文的 TTF 字体。

## 使用方法

### Step 0: 选择风格类型

在生成PDF之前，必须先询问用户选择风格类型。支持的类型：

| 类型 | 封面样式 | 视觉特点 |
|------|----------|----------|
| `report` | 全出血背景 | 深色背景+点阵网格，Playfair Display字体 |
| `proposal` | 左右分割 | 左侧面板+右侧几何图形，Syne字体 |
| `resume` | 文字排版 | 超大首词，DM Serif Display字体 |
| `portfolio` | 氛围感 | 近黑色+径向光晕，Fraunces字体 |
| `academic` | 文字排版 | 浅色背景+古典衬线，EB Garamond字体 |
| `general` | 全出血背景 | 深石板色，Outfit字体 |
| `minimal` | 极简 | 白色+8px强调条，Cormorant Garamond字体 |
| `stripe` | 条纹 | 3条粗彩色横带，Barlow Condensed字体 |
| `diagonal` | 对角线 | SVG斜切，明暗两半，Montserrat字体 |
| `frame` | 边框 | 内嵌边框+角落装饰，Cormorant字体 |
| `editorial` | 编辑风格 | 幽灵字母+全大写标题，Bebas Neue字体 |
| `magazine` | 杂志 | 暖米色背景+居中堆叠+主图，Playfair Display字体 |
| `darkroom` | 暗房 | 海军蓝背景+居中堆叠+灰度图，Playfair Display字体 |
| `terminal` | 终端 | 近黑色+网格线+等宽字体+霓虹绿 |
| `poster` | 海报 | 白色背景+粗侧边栏+超大标题，Barlow Condensed字体 |

**必须询问用户**："请选择一个风格类型（如 report、proposal、magazine 等）"

### Step 1: 生成 tokens

使用 minimax-pdf 的 palette.py 生成基础 tokens，使用用户选择的类型：

```bash
python3 minimax-pdf/scripts/palette.py \
  --title "中文标题" \
  --type <用户选择的类型> \
  --author "作者" \
  --date "2026年3月" \
  --accent "#2D5F8A" \
  --out tokens.json
```

### Step 2: 注入中文字体配置

编辑 `tokens.json`，替换字体配置：

```json
{
  "font_display_rl": "WenQuanYiZenHei",
  "font_body_rl": "WenQuanYiZenHei",
  "font_body_b_rl": "WenQuanYiZenHei",
  "font_heading": "WenQuanYiZenHei",
  "font_body_b": "WenQuanYiZenHei",
  "font_paths": {
    "WenQuanYiZenHei": "/usr/share/fonts/truetype/wqy/wqy-zenhei.ttc"
  }
}
```

### Step 3: 继续 minimax-pdf 流程

```bash
# 渲染封面
python3 minimax-pdf/scripts/cover.py --tokens tokens.json --out cover.html
node minimax-pdf/scripts/render_cover.js --input cover.html --out cover.pdf

# 渲染正文
python3 minimax-pdf/scripts/render_body.py --tokens tokens.json --content content.json --out body.pdf

# 合并
python3 minimax-pdf/scripts/merge.py --cover cover.pdf --body body.pdf --out output.pdf
```

## 中文字体路径参考

| 字体 | 路径 |
|------|------|
| WenQuanYi Zen Hei | `/usr/share/fonts/truetype/wqy/wqy-zenhei.ttc` |
| WenQuanYi Micro Hei | `/usr/share/fonts/truetype/wqy/wqy-microhei.ttc` |

查找系统中文字体路径：

```bash
fc-list :lang=zh -f "%{family}\t%{file}\n"
```

## 常见问题

**Q: 中文仍然显示为空格怎么办？**
A: 检查：
1. `fc-list :lang=zh` 是否能找到字体
2. `font_paths` 中的路径是否正确指向 .ttf 或 .ttc 文件
3. 字体名称（key）必须与 `font_body_rl` 的值一致

**Q: 封面中文正常但正文中文是空格？**
A: 这是因为封面使用 Google Fonts（不支持中文），而正文使用 ReportLab 系统字体。确保 `font_body_rl` 和 `font_paths` 正确配置。

**Q: 字体文件不存在怎么办？**
A: 安装字体：
```bash
# Ubuntu/Debian
sudo apt-get install fonts-wqy-zenhei
fc-cache -f
```

## 完整示例

```bash
# 0. 询问用户选择风格类型（示例：用户选择 report）

# 1. 生成 tokens（使用用户选择的类型）
python3 minimax-pdf/scripts/palette.py \
  --title "国产大模型对比报告" \
  --type report \
  --author "OpenCode AI" \
  --date "2026年3月" \
  --out /tmp/tokens.json

# 2. 注入中文字体（使用 Python）
python3 -c "
import json
with open('/tmp/tokens.json') as f:
    t = json.load(f)
t['font_display_rl'] = 'WenQuanYiZenHei'
t['font_body_rl'] = 'WenQuanYiZenHei'
t['font_body_b_rl'] = 'WenQuanYiZenHei'
t['font_heading'] = 'WenQuanYiZenHei'
t['font_body_b'] = 'WenQuanYiZenHei'
t['font_paths'] = {'WenQuanYiZenHei': '/usr/share/fonts/truetype/wqy/wqy-zenhei.ttc'}
with open('/tmp/tokens.json', 'w') as f:
    json.dump(t, f, indent=2)
"

# 3. 渲染
python3 minimax-pdf/scripts/cover.py --tokens /tmp/tokens.json --out /tmp/cover.html
node minimax-pdf/scripts/render_cover.js --input /tmp/cover.html --out /tmp/cover.pdf
python3 minimax-pdf/scripts/render_body.py --tokens /tmp/tokens.json --content content.json --out /tmp/body.pdf
python3 minimax-pdf/scripts/merge.py --cover /tmp/cover.pdf --body /tmp/body.pdf --out output.pdf
```

## 依赖

与 minimax-pdf 相同：
- Python 3.9+
- reportlab
- pypdf
- Node.js 18+（用于渲染封面）
- playwright

安装依赖：
```bash
bash minimax-pdf/scripts/make.sh fix
