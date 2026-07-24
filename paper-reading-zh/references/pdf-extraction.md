# PDF 文本提取实践

当 `paper-reading-zh` 已触发，且用户以本地 PDF 文件作为论文锚点时读取本文件。本文件解决"如何拿到可读原文"这一前置步骤，不替代深读、工程拆解或证据审计流程。

## 核心问题

`analyze_multimedia` 工具不支持 PDF 格式。因此当用户提供本地 PDF 路径时，不能直接用多媒体工具读取，必须先做文本层提取。

## 推荐工具链

优先使用 PyMuPDF（`fitz`），原因：提取质量高、能保留分页结构、对公式和表格的文本层容忍度较好。

## 提取流程

### 1. 确认文件存在

先检查路径是否有效，避免后续脚本因路径错误中断。

### 2. 安装依赖（按需）

如果环境中没有 `fitz`，安装 PyMuPDF （如果没有安装）：

```
python -m pip install PyMuPDF
```

安装是后台长任务，建议用 `run_in_background`，避免阻塞会话。安装完成后继续后续步骤。

### 3. 提取全文并写入 UTF-8 文件

直接 `print` 提取的文本在 Windows 上常因 GBK 编码报错（UnicodeEncodeError），尤其是含数学符号（如 ∗、≤、∈）的论文。正确做法是把文本写入 UTF-8 文件，再用 `read_file` 读取。

推荐脚本模式：

```python
import fitz
import sys
import io

sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')

doc = fitz.open(r'<PDF 绝对路径>')
output = []
output.append(f'Total pages: {doc.page_count}')
for i, page in enumerate(doc):
    text = page.get_text()
    output.append(f'\n===== PAGE {i+1} =====')
    output.append(text)
doc.close()

with open(r'<输出 txt 绝对路径>', 'w', encoding='utf-8') as f:
    f.write('\n'.join(output))

print('Done. Text saved to <输出路径>')
```

关键点：
- 用 `sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')` 解决标准输出编码问题。
- 脚本文件本身用 UTF-8 写入（`write_file` 默认 UTF-8）。
- 输出文件也用 `encoding='utf-8'` 写入。
- PDF 路径用 `r'...'` 原始字符串，避免反斜杠转义问题。

### 4. 读取提取后的文本

用 `read_file` 读取输出 txt 文件。如果文件较大（超过 2000 行），分批读取：
- 第一次读前 2000 行（默认）。
- 后续用 `offset` 参数继续读取。
- 到达文件末尾时工具会提示。

### 5. 清理临时文件

提取完成后删除临时脚本和 txt 文件（不必删除原始的论文PDF），保持工作目录干净：

```
Remove-Item "<脚本路径>", "<txt 路径>" -Force
```

## 提取质量与证据边界

提取后的文本可能出现以下问题，必须在深读回答的"材料范围"里说明：

- **公式乱码或丢失**：LaTeX 公式提取后常变成碎片化文本或特殊字符序列。不要基于乱码重构公式；只解释上下文可确认的含义，必要时提示用户"公式以原文截图为准"。
- **表格错位**：多列表格可能被提取为单行文本，列对齐丢失。引用表格数字时优先以正文叙述为锚点。
- **图 caption 与图内容混淆**：`get_text()` 提取的是文本层，图本身的视觉内容不可读；只能获得 caption 文本，不能描述图的视觉元素。
- **页眉页脚噪声**：页码、会议名、版权声明会混入正文，需要人工识别并忽略。
- **断词断行**：PDF 排版可能导致单词被连字符断开或行尾断开，影响搜索。

## 何时不能依赖文本提取

以下情况文本提取不可靠，应降级处理：
- PDF 是扫描件（无文本层）：提取结果为空或乱码，需 OCR 工具。
- PDF 是图片导出：同上。
- 公式以图片形式嵌入：提取不到，只能从上下文推断。
- 表格以图片形式嵌入：同上。

遇到这些情况时，在材料范围中明确说明"文本提取无法获得 X"，只基于可提取文本回答，不编造不可读部分的内容。

## 不要做的事

- 不要用 `read_file` 直接读 PDF：该工具只处理文本文件，无法解析 PDF 二进制格式。
- 不要在提取脚本里直接 `print` 全文：Windows GBK 编码会报错。
- 不要把临时 txt 和脚本留在用户工作目录：用完即删。
- 不要因为提取质量差就跳过材料范围说明：必须如实告知用户当前材料的限制。
