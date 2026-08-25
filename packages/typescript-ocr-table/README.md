# @mightydatainc/ocr-table

AI-powered OCR for extracting structured tables and metadata from PDF pages and other document images. Render PDFs into page images, identify tables, transcribe their rows, extract notes and aggregations, and find user-specified fields across multi-page documents.

## Installation

```bash
npm install @mightydatainc/ocr-table
```

## Quick Start

### `ocrTablesFromPDFs`

Process one or more PDF files, or recursively process PDFs in directories. The result contains the extracted tables and any requested document metadata:

```ts
import OpenAI from 'openai';
import { ocrTablesFromPDFs } from '@mightydatainc/ocr-table';

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const results = await ocrTablesFromPDFs(
  client,
  './documents',
  {
    'Invoice Number': '',
    Vendor: '',
    'Billing Month': 'The month and year covered by the invoice',
  },
  'Preserve the wording and capitalization from the source document.'
);

const invoice = results['documents/invoice.pdf'];
console.log(invoice?.metadata); // { "Invoice Number": "...", Vendor: "...", ... }
console.log(invoice?.tables[0]?.data); // [{ "Item Name": "...", Quantity: "..." }]
```

The `fieldsToExtract` object maps each output field name to an optional description or question. Fields can be explicitly labeled in the document or inferred from its content. An optional progress callback receives the number of completed and remaining files:

```ts
const results = await ocrTablesFromPDFs(
  client,
  ['./invoices/january.pdf', './invoices/february.pdf'],
  { 'Invoice Number': '', Total: '' },
  undefined,
  (filesDone, filesRemaining) => {
    console.log(`${filesDone} complete, ${filesRemaining} remaining`);
  }
);
```

### `ocrStructuredFields`

Extract selected fields from PNG page buffers. The scan proceeds through the document page by page and carries context forward so fields can be found in later pages:

```ts
import { readFileSync } from 'node:fs';
import { ocrStructuredFields } from '@mightydatainc/ocr-table';

const pages = [
  readFileSync('./document-page-1.png'),
  readFileSync('./document-page-2.png'),
];

const fields = await ocrStructuredFields(client, pages, {
  'School Name': '',
  Principal: '',
  'Contact Phone Number':
    'The phone number for the department that handles questions',
});

console.log(fields);
```

### Table extraction functions

The lower-level functions can be used when table processing needs to be controlled explicitly:

```ts
import {
  ocrIdentifyTablesOnPage,
  ocrTableColumnHeaders,
  ocrTranscribeTableFromPages,
} from '@mightydatainc/ocr-table';

const page = readFileSync('./page-1.png');

const tables = await ocrIdentifyTablesOnPage(client, page);
const tableName = tables[0]?.name ?? 'Purchases';
const columns = await ocrTableColumnHeaders(client, tableName, page);

const table = await ocrTranscribeTableFromPages(
  client,
  tableName,
  tables[0]?.description ?? '',
  1,
  [page]
);

console.log(columns);
console.log(table.data);
```

Table detection can handle multiple tables on a page, tables embedded in prose, orphaned titles, and rows that continue across page breaks. `ocrTranscribeTableFromPages` follows a table across subsequent page images and records its `page_end`.

Use `additionalInstructions` to provide document-specific guidance, such as excluding a column, supplying names for a headerless table, controlling table names, or clarifying ambiguous visual text.

## `OcrTable`

An extracted table has this shape:

```ts
interface OcrTable {
  name: string;
  description: string;
  columns: string[];
  page_start: number;
  page_end: number;
  data: Array<Record<string, string>>;
  aggregations: string;
  notes: string;
}
```

The `data` array contains one object per body row, keyed by the ordered column names. Empty cells are represented as empty strings. `aggregations` contains visible totals or summary information, while `notes` contains relevant notes associated with the table.

## PDF rendering

Convert every page of a PDF into high-resolution PNG buffers before sending the images to OCR:

```ts
import { renderPdfPagesToPngBuffers } from '@mightydatainc/ocr-table';

const pageBuffers = await renderPdfPagesToPngBuffers('./report.pdf');
```

## Metadata utilities

Use `getAllOcrMetadatas` to collect metadata from multi-file extraction results. Empty metadata entries are omitted by default:

```ts
import { getAllOcrMetadatas } from '@mightydatainc/ocr-table';

const metadataByFile = getAllOcrMetadatas(results);
const metadataIncludingEmpty = getAllOcrMetadatas(results, true);
```

## Local dev

From `packages/typescript-ocr-table`:

```bash
npm ci
npm test
npm run build
```

Live OCR tests require `OPENAI_API_KEY` to be available in the test environment. The test setup loads variables from `.env`.

## Notes

- Package name for `npm install` is `@mightydatainc/ocr-table`.
- Requires an OpenAI-compatible client supported by `@mightydatainc/llm-conversation`.
- OCR functions are asynchronous and return Promises.
- PDF input is rendered with `pdf-to-png-converter`; PNG page buffers can also be processed directly.
- Model-based OCR can occasionally produce small transcription differences, especially for blurry or ambiguous source text.
