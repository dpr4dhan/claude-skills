---
name: req-to-md
description: Extracts and converts raw requirement documents (PDF, .docx, .doc files) into a structured Markdown requirements file, then saves it to the project's docs/ folder. Use this skill whenever the user wants to convert, parse, or extract requirements from a document into Markdown — even if they just say "generate md from this file", "parse requirements", "convert this doc", "make a requirements file from pdf", "extract requirements from docx", or hands you a file and asks for a doc or md output. Always invoke this before attempting to manually parse a document.
---

# Requirement Document → Markdown Converter

Convert a PDF or DOCX requirements document into a clean, structured Markdown file. The output must faithfully represent **only the content present in the source file** — no assumptions, additions, or hallucinated content.

## Step 1 — Identify the source file

If the user has not provided a file path, ask for it. Accept:
- `.pdf`
- `.docx` / `.doc`

Resolve the absolute path. If relative, resolve from the current working directory.

## Step 2 — Extract raw content

Use Python to extract text. Run the appropriate script based on file type.

**PDF:**
```python
import sys, pathlib
path = sys.argv[1]
try:
    import pdfplumber
    with pdfplumber.open(path) as pdf:
        pages = [page.extract_text() or "" for page in pdf.pages]
    text = "\n\n--- Page Break ---\n\n".join(pages)
except ImportError:
    from pypdf import PdfReader
    reader = PdfReader(path)
    text = "\n\n--- Page Break ---\n\n".join(
        page.extract_text() or "" for page in reader.pages
    )
print(text)
```

**DOCX:**
```python
import sys
from docx import Document
doc = Document(sys.argv[1])
lines = []
for para in doc.paragraphs:
    if para.text.strip():
        lines.append(para.text)
print("\n\n".join(lines))
```

If the required library is missing, install it first:
```
pip install pdfplumber   # for PDF
pip install python-docx  # for DOCX
```

If extraction fails or produces empty output, tell the user and ask them to paste the content directly.

## Step 3 — Analyze structure

Before writing the Markdown, identify what structural elements are present in the extracted text:

| What you see in the source | How to render in MD |
|---|---|
| Document title / project name | `# Title` |
| Numbered section headings | `## 1. Section Name` |
| Sub-sections | `### 1.1 Sub-section` |
| Numbered requirement lines | `1. Requirement text` |
| Bullet/dash lists | `- item` |
| Tables | Markdown table |
| Code / technical specs | ` ```code block``` ` |
| Definitions / glossary | `**Term**: Definition` |

Do **not** invent section headings that aren't present. If the source is a flat wall of text with no visible structure, render it as paragraphs without adding fake headings.

## Step 4 — Write the Markdown

The golden rule: **every sentence in the output must trace back to a sentence in the source**.

- Preserve the original wording of requirements — do not rephrase or summarise.
- Preserve numbering if the source uses it (REQ-001, 3.2.1, etc.).
- If a section has a heading in the source, use a heading in the output.
- If content spans pages, merge it naturally (remove "--- Page Break ---" markers).
- Tables, notes, and warnings should be rendered as close to the original layout as possible.
- Omit page headers/footers, watermarks, and repeated boilerplate (e.g., "CONFIDENTIAL — Page N of M") that are not part of the requirements content.

## Step 5 — Determine output filename

Derive from the source filename (no extension), replacing spaces with hyphens, lowercasing:

| Source file | Output file |
|---|---|
| `Project Brief.pdf` | `project-brief.md` |
| `FD_Requirements_v2.docx` | `fd_requirements_v2.md` |
| `requirements.pdf` | `requirements.md` |

## Step 6 — Resolve project root and save

Find the project root by walking up from the current directory until you find any of: `.git`, `composer.json`, `package.json`, `pyproject.toml`. If none found, use the current working directory.

Save the output file to:
```
<project-root>/docs/<output-filename>.md
```

Create `docs/` if it does not exist.

## Step 7 — Report

Tell the user:
- ✓ Source file processed (name + type)
- ✓ Output saved to: `docs/<filename>.md`
- Any content that could not be extracted (images, scanned pages, complex embedded objects) — be specific so the user knows what to manually add
