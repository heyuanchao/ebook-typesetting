# PDF 修复目录跳转工作流

适用于 PDF 中“正文目录页链接”或“左侧书签/大纲”点击跳转不正确的情况，尤其是带有 PART / 章节 / 子标题层级的电子书 PDF。

下次可直接对 Claude 说：

```text
按《pdf修复目录跳转工作流.md》修复 XXX.pdf 的目录跳转。
```

---

## 1. 准备与备份

1. 明确输入 PDF 文件名。
2. 修复前先备份原始 PDF，不直接覆盖。

推荐备份名：

```text
原文件名_目录跳转修复前备份.pdf
```

推荐输出名：

```text
原文件名_目录和书签跳转修复.pdf
```

如果需要保留版本，可使用：

```text
原文件名_目录和书签跳转修复_v1.pdf
原文件名_目录和书签跳转修复_v2.pdf
```

---

## 2. 先区分两套目录系统

PDF 里常见有两套“目录”：

1. **正文目录页链接**  
   即 PDF 第几页里的“目录”页面，点击页面文字跳转。

2. **左侧书签 / 大纲 / 导航目录**  
   即阅读器左侧栏显示的章节树，点击书签跳转。

重要：

```text
正文目录页链接正确，不代表左侧书签正确。
左侧书签正确，也不代表正文目录页链接正确。
两套都要检查、都要修复。
```

---

## 3. 检查问题类型

逐项检查：

```text
1. 是否存在重叠重复链接
2. 正文目录页点击是否跳错
3. 左侧书签点击是否跳错
4. PART 本身是否跳错
5. PART 下面的子标题是否跳错
6. 点击后是否只到页，不到标题位置
7. 目录打开时是否需要全部展开或默认折叠
```

常见问题：

```text
目录页中同一个文字区域有两个重复 Link 注释
子标题跳到 PART 页，而不是子标题页
子标题跳到前一页
标题文字在正文中重复出现，导致搜索定位错
只修了页面目录，忘了修左侧书签
重建 PDF 时误改了书签层级或展开状态
```

---

## 4. 推荐工具

优先使用 Python + PyMuPDF：

```bash
python -m pip install PyMuPDF
```

PyMuPDF 的优势：

```text
可以读取页面文字坐标
可以删除和重新插入页面链接
可以读取和重写 PDF 书签 / 大纲
可以设置书签默认展开或折叠状态
```

如只需要初步检查 PDF 注释，也可使用 pypdf，但复杂目录跳转修复建议使用 PyMuPDF。

---

## 5. 提取正文目录条目

从正文目录页提取可见目录文字，例如第 2-3 页：

```python
import fitz

doc = fitz.open("input.pdf")
toc_lines = []
for pno in [1, 2]:  # 0-based，第 2、3 页
    for line in doc[pno].get_text().splitlines():
        line = line.strip()
        if line and line != "目录":
            toc_lines.append(line)
```

注意：

```text
页码是 0-based。
PDF 第 1 页在代码中是 0。
如果目录页不是第 2-3 页，要相应调整。
```

---

## 6. 定位正文标题的真实位置

不要只按“某个标题文字在哪一页出现”来判断，因为标题可能在正文、引语或目录中重复出现。

推荐做法：

1. 跳过封面和目录页；
2. 按页面的文字行定位；
3. 使用标题文字的归一化匹配；
4. 获取标题所在行的坐标；
5. 链接跳转到标题上方一点的位置。

示例：

```python
import re
import fitz

def norm(s):
    return re.sub(r"\s+", "", s).replace("“", "\"").replace("”", "\"")

def find_title(doc, title):
    nt = norm(title)
    # 跳过封面和目录页，从正文开始找
    for pno in range(3, doc.page_count):
        page = doc[pno]
        data = page.get_text("dict")
        for block in data.get("blocks", []):
            if block.get("type") != 0:
                continue
            for line in block.get("lines", []):
                text = "".join(span.get("text", "") for span in line.get("spans", []))
                if nt == norm(text) or nt in norm(text):
                    rect = fitz.Rect(line["bbox"])
                    point = fitz.Point(max(rect.x0 - 2, 0), max(rect.y0 - 18, 0))
                    return pno, point, rect
    raise RuntimeError(f"cannot locate title: {title}")
```

检查点：

```text
PART 标题应跳到 PART 标题位置
PART 子标题应跳到对应文章标题位置
不要让子标题跳到 PART 页
不要让子标题跳到正文中再次出现该短句的位置
```

---

## 7. 修复正文目录页链接

流程：

1. 读取目录页现有链接；
2. 按视觉顺序排序；
3. 去掉重复重叠链接；
4. 删除目录页旧链接；
5. 按目录文字逐条插入新链接，跳到真实标题坐标。

示例：

```python
link_rects = []
for pno in [1, 2]:
    page = doc[pno]
    links = sorted(page.get_links(), key=lambda l: (l["from"].y0, l["from"].x0))
    seen = set()
    for link in links:
        rect = link["from"]
        key = tuple(round(v, 2) for v in (rect.x0, rect.y0, rect.x1, rect.y1))
        if key in seen:
            continue
        seen.add(key)
        link_rects.append((pno, rect))

    for link in list(page.get_links()):
        page.delete_link(link)

if len(link_rects) != len(toc_lines):
    raise RuntimeError(f"link count mismatch: rects={len(link_rects)} toc={len(toc_lines)}")

for (src_pno, rect), line in zip(link_rects, toc_lines):
    target_pno, point, bbox = targets[line]
    doc[src_pno].insert_link({
        "kind": fitz.LINK_GOTO,
        "from": rect,
        "page": target_pno,
        "to": point,
        "zoom": 0,
    })
```

---

## 8. 修复左侧书签 / 大纲跳转

正文目录页修好后，还要同步修复左侧书签。

示例：

```python
old_toc = doc.get_toc(simple=False)
new_toc = []

for item in old_toc:
    level, title, page_num = item[0], item[1], item[2]

    matched = None
    for line in toc_lines:
        if norm(title) == norm(line):
            matched = line
            break

    if matched:
        target_pno, point, bbox = targets[matched]
        new_toc.append([
            level,
            title,
            target_pno + 1,
            {
                "kind": fitz.LINK_GOTO,
                "page": target_pno,
                "to": point,
                "zoom": 0,
            },
        ])
    else:
        new_toc.append(item)

# collapse 参数控制初始展开状态，见下一节
doc.set_toc(new_toc, collapse=99)
```

注意：

```text
不要随意重建书签层级。
尽量保留原 level / title，只改跳转目标。
```

---

## 9. 设置书签默认展开状态

根据用户需求设置：

### 全部展开

```python
doc.set_toc(new_toc, collapse=99)
```

### 只显示一级目录，子目录默认折叠

```python
doc.set_toc(new_toc, collapse=1)
```

说明：

```text
不同 PDF 阅读器对默认展开/折叠状态支持可能略有不同。
但应尽量在 PDF 大纲数据中写入期望状态。
```

---

## 10. 保存输出

```python
doc.save("output.pdf", garbage=4, deflate=True)
doc.close()
```

不要直接覆盖原 PDF，除非用户确认。

---

## 11. 验证

至少验证：

```text
1. 正文目录页链接数量是否等于目录条目数量
2. 是否还有重复链接
3. 页面目录跳转目标是否正确
4. 左侧书签跳转目标是否正确
5. PART 子标题是否跳到文章标题，而不是 PART 页
6. 打开 PDF 时书签展开状态是否符合要求
```

验证示例：

```python
rd = fitz.open("output.pdf")

links_after = []
for pno in [1, 2]:
    for link in sorted(rd[pno].get_links(), key=lambda l: (l["from"].y0, l["from"].x0)):
        links_after.append(link)

page_errors = []
for i, (line, link) in enumerate(zip(toc_lines, links_after), 1):
    target_pno, point, bbox = targets[line]
    if link.get("page") != target_pno:
        page_errors.append((i, line, link.get("page") + 1, target_pno + 1))

outline_errors = []
for level, title, page_num, dest in rd.get_toc(simple=False):
    matched = next((line for line in toc_lines if norm(title) == norm(line)), None)
    if matched:
        target_pno, point, bbox = targets[matched]
        if page_num != target_pno + 1:
            outline_errors.append((title, page_num, target_pno + 1))

print("page_link_errors=", len(page_errors))
print("outline_errors=", len(outline_errors))
rd.close()
```

期望结果：

```text
page_link_errors=0
outline_errors=0
```

---

## 12. 本次经验教训

本次修复中出现过的问题：

```text
1. 只删除重复链接，不足以修复跳转目标错误。
2. 只修正文目录页，不足以修复左侧书签。
3. 只按页码修，不如按标题坐标修准确。
4. 标题文字可能在正文中重复出现，不能简单取最后一次或第一次匹配。
5. 重建书签结构可能误改 PART 层级，应保留原结构，只改目标。
6. 默认展开/折叠状态是独立需求，需要单独设置。
```

最终正确策略：

```text
备份原文件
提取正文目录条目
按标题行坐标定位真实正文标题
删除目录页旧链接并重建页面链接
保留原书签层级，只修书签目标
按用户要求设置全部展开或折叠
保存为新文件
验证页面目录和左侧书签均无错误
```

---

## 13. 推荐完整脚本模板

```python
import fitz
from pathlib import Path
import re

src = Path("input.pdf")
out = Path("input_目录和书签跳转修复.pdf")
backup = Path("input_目录跳转修复前备份.pdf")
if not backup.exists():
    backup.write_bytes(src.read_bytes())

def norm(s):
    return re.sub(r"\s+", "", s).replace("“", "\"").replace("”", "\"")

def find_title(doc, title):
    nt = norm(title)
    for pno in range(3, doc.page_count):
        page = doc[pno]
        data = page.get_text("dict")
        for block in data.get("blocks", []):
            if block.get("type") != 0:
                continue
            for line in block.get("lines", []):
                text = "".join(span.get("text", "") for span in line.get("spans", []))
                if nt == norm(text) or nt in norm(text):
                    rect = fitz.Rect(line["bbox"])
                    point = fitz.Point(max(rect.x0 - 2, 0), max(rect.y0 - 18, 0))
                    return pno, point, rect
    raise RuntimeError(f"cannot locate title: {title}")

doc = fitz.open(src)

# 修改为实际目录页页码，0-based
toc_pages = [1, 2]

toc_lines = []
for pno in toc_pages:
    for line in doc[pno].get_text().splitlines():
        line = line.strip()
        if line and line != "目录":
            toc_lines.append(line)

targets = {line: find_title(doc, line) for line in toc_lines}

# 修正文目录页链接
link_rects = []
for pno in toc_pages:
    page = doc[pno]
    links = sorted(page.get_links(), key=lambda l: (l["from"].y0, l["from"].x0))
    seen = set()
    for link in links:
        rect = link["from"]
        key = tuple(round(v, 2) for v in (rect.x0, rect.y0, rect.x1, rect.y1))
        if key in seen:
            continue
        seen.add(key)
        link_rects.append((pno, rect))
    for link in list(page.get_links()):
        page.delete_link(link)

if len(link_rects) != len(toc_lines):
    raise RuntimeError(f"link count mismatch: rects={len(link_rects)} toc={len(toc_lines)}")

for (src_pno, rect), line in zip(link_rects, toc_lines):
    target_pno, point, bbox = targets[line]
    doc[src_pno].insert_link({
        "kind": fitz.LINK_GOTO,
        "from": rect,
        "page": target_pno,
        "to": point,
        "zoom": 0,
    })

# 修复左侧书签 / 大纲
old_toc = doc.get_toc(simple=False)
new_toc = []
for item in old_toc:
    level, title, page_num = item[0], item[1], item[2]
    matched = next((line for line in toc_lines if norm(title) == norm(line)), None)
    if matched:
        target_pno, point, bbox = targets[matched]
        new_toc.append([
            level,
            title,
            target_pno + 1,
            {"kind": fitz.LINK_GOTO, "page": target_pno, "to": point, "zoom": 0},
        ])
    else:
        new_toc.append(item)

# 全部展开：collapse=99；只展开一级：collapse=1
doc.set_toc(new_toc, collapse=99)

doc.save(out, garbage=4, deflate=True)
doc.close()

print(f"wrote {out}")
print(f"backup {backup}")
```
