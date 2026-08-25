# ocr-table

AI-powered OCR for extracting structured tables and metadata from PDF pages and other document images, powered by OpenAI-compatible vision models. Render PDFs into page images, identify tables, transcribe rows, extract notes and aggregations, and pull out document fields across multi-page workflows.

This repo contains cross-language implementations from Mighty Data Inc. that can be dropped straight into real projects.

## Design goals

- Minimal abstractions
- Predictable behavior
- Cross-language parity (Python + TypeScript)
- Easy to drop into real projects

Rather than relying on brittle template matching alone, the OCR tooling uses LLM-driven extraction to recognize table structure, headings, notes, and document metadata across multiple pages and formats.

## Packages

- TypeScript: `@mightydatainc/ocr-table` (npm) in `packages/typescript-ocr-table`
- Python: `mightydatainc-ocr-table` (PyPI, import as `mightydatainc_ocr_table`) in `packages/python-ocr-table`

Package-specific docs:

- TypeScript: [packages/typescript-ocr-table/README.md](packages/typescript-ocr-table/README.md)
- Python: [packages/python-ocr-table/README.md](packages/python-ocr-table/README.md)

## Feature overview

Core capabilities (Python + TypeScript):

- `ocr_tables_from_pdfs` / `ocrTablesFromPDFs` -- process one or more PDFs, or recursively process directories of PDFs, and return extracted tables plus requested document metadata
- `ocr_structured_fields` / `ocrStructuredFields` -- extract named fields from page PNG buffers while carrying context from page to page
- Table-level extraction helpers -- identify tables, infer column headers, and transcribe rows across page breaks
- Structured table output -- returns a row-based data model with columns, notes, aggregations, and page start/end metadata
- `fieldsToExtract` / `fields_to_extract` -- map each required field to an optional description or question, supporting explicit labels and inference from document content

## Quick start

### Python

```python
from openai import OpenAI
from mightydatainc_ocr_table import ocr_tables_from_pdfs

client = OpenAI()

results = ocr_tables_from_pdfs(
    client,
    "./documents",
    {
        "Invoice Number": "",
        "Vendor": "",
        "Billing Month": "The month and year covered by the invoice",
    },
    "Preserve the wording and capitalization from the source document.",
)

invoice = results["documents/invoice.pdf"]
print(invoice["metadata"])  # { "Invoice Number": "...", "Vendor": "...", ... }
print(invoice["tables"][0]["data"])  # [{ "Item Name": "...", "Quantity": "..." }]
```

### TypeScript

```ts
import OpenAI from "openai";
import { ocrTablesFromPDFs } from "@mightydatainc/ocr-table";

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const results = await ocrTablesFromPDFs(
  client,
  "./documents",
  {
    "Invoice Number": "",
    Vendor: "",
    "Billing Month": "The month and year covered by the invoice",
  },
  "Preserve the wording and capitalization from the source document.",
);

const invoice = results["documents/invoice.pdf"];
console.log(invoice?.metadata);
console.log(invoice?.tables[0]?.data);
```

## Local dev (Windows)

### Python

From `packages/python-ocr-table`, activate the package venv and run tests:

```powershell
.\.venv\Scripts\Activate.ps1
python -m pytest tests/ -v
```

### TypeScript

From `packages/typescript-ocr-table`:

```powershell
npm ci
npm test
npm run build
```

Live OCR tests require `OPENAI_API_KEY` to be available in the environment. The test setup loads variables from `.env` when present.

## Unit testing with live API calls

Some tests intentionally call the real OpenAI-compatible API instead of mocking model responses.

This is by design: the core contract includes prompt wording, structured output parsing, document interpretation, and model behavior working together. Mock-only tests cannot verify whether production prompts still elicit the required structured output on real PDFs and page images.

These tests do have tradeoffs:

- They require `OPENAI_API_KEY` in the test environment.
- They incur a small API cost when run.
- They can be slower than pure unit tests.

Deterministic assertions are still intentional here: tests are written with tightly scoped inputs and clearly defined expected outcomes, so stable structured output is treated as a baseline requirement. If those tests fail, we treat it as a bug in prompt design, output handling, or integration behavior.

## Release process

This repo ships two public packages with aligned versions:

- npm: `@mightydatainc/ocr-table` from `packages/typescript-ocr-table`
- PyPI: `mightydatainc-ocr-table` from `packages/python-ocr-table`

GitHub release automation publishes each package automatically on push to `main` when its package version changes:

- TypeScript checks `packages/typescript-ocr-table/package.json`
- Python checks `packages/python-ocr-table/pyproject.toml`

Before publishing, ensure both versions are updated, then authenticate once locally:

- npm: `npm login`
- PyPI: configure `~/.pypirc` or use `python -m twine upload --repository pypi dist/*`

After publish, tag and push a release tag (example):

```powershell
git tag v1.2.0
git push origin v1.2.0
```

## Notes

- Package name for `npm install` is `@mightydatainc/ocr-table`.
- Package name for `pip install` is `mightydatainc-ocr-table`.
- Python import package is `mightydatainc_ocr_table`.
- Requires an OpenAI-compatible client supported by the shared model conversation package.
- OCR functions are asynchronous and return Promises in TypeScript; Python equivalents are synchronous wrappers around async-compatible model calls in the package implementation.
- PDF input is rendered with `pdf-to-png-converter`; PNG page buffers can also be processed directly.
- Model-based OCR can occasionally produce small transcription differences, especially for blurry or ambiguous source text.
