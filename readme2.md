# health.kr drug crawler

This package collects Korean drug information from the public pages of the Korea Pharmaceutical Information Center (`약학정보원`, health.kr). It searches by ingredient keyword, discovers each matching product, fetches the product detail page, extracts the requested fields, and writes UTF-8 JSON Lines to `records.jsonl`.

The first line of `records.jsonl` is a schema header. Each later line contains one product. Product lines store field values as strings or `null`; they do not repeat Korean source-label wrappers.

The crawler keeps source text in Korean. It does not translate, summarize, or infer missing medical content.

## Source and permission requirements

- The crawler uses public rendered search pages and public `/drug/{code}` detail pages.
- It does not call health.kr `/api/` routes. The site blocks those routes in `robots.txt`.
- health.kr restricts unauthorized copying and reuse. Obtain permission from KPIC before collecting full text.
- The `crawl` command requires `--permission-confirmed`. The `discover` command collects product names, codes, and URLs without copying the detail text.
- A detail page may identify `식품의약품안전처` as its data source. The crawler stores that page attribution in the product's positional `record_metadata` array.

## Input configuration

The default configuration file is `261106crawldata.json`. It contains a JSON array with one object per disease and ingredient search:

```json
[
  {
    "disease": "고혈압",
    "search_keyword": "Doxazosin",
    "expected_count": 11
  },
  {
    "disease": "당뇨병",
    "search_keyword": "Insulin aspart",
    "expected_count": 7
  }
]
```

| Input field | Type | Meaning |
| --- | --- | --- |
| `disease` | string | Dataset group copied into each matching output record. |
| `search_keyword` | string | Ingredient or product keyword sent to the health.kr search page. |
| `expected_count` | non-negative integer | Expected product count used by strict discovery validation. |

Keep `search_keyword` unique in the configuration. The full crawl joins discovery results back to the input through this value.

The loader rejects empty disease names, empty keywords, booleans, negative counts, and non-integer counts.

## Commands

### Discover product URLs

```bash
cd /home/smrc/yejoon/ragflow
.venv/bin/python -m tools.healthkr_crawler discover \
  --config tools/healthkr_crawler/261106crawldata.json \
  --output tools/healthkr_crawler/output/demo_discovery.json
```

The command creates `discovery.json`. It does not fetch full detail text.

### Crawl full detail text

```bash
cd /home/smrc/yejoon/ragflow
.venv/bin/python -m tools.healthkr_crawler crawl \
  --config tools/healthkr_crawler/261106crawldata.json \
  --permission-confirmed \
  --output-dir tools/healthkr_crawler/output/demo_full
```

Useful options:

| Option | Default | Effect |
| --- | ---: | --- |
| `--headful` | off | Shows the Chromium search browser. |
| `--browser-timeout` | 60 seconds | Sets the rendered search-page timeout. |
| `--request-timeout` | 45 seconds | Sets the detail-page HTTP timeout. |
| `--request-delay` | 1.5 seconds | Sets the delay between detail requests. The code enforces a 0.5-second floor. |
| `--allow-count-mismatch` | off | Continues when configured, reported, and discovered counts differ. |

The crawl command opens `records.jsonl` in write mode. A rerun with the same output directory replaces the previous JSONL file.

## Collection pipeline

### 1. Load and validate search specifications

`load_search_specs()` reads the configuration as UTF-8 JSON and creates a `SearchSpec` for each entry. The crawler retains the disease, keyword, and configured count through discovery and detail extraction.

### 2. Render health.kr search results

`HealthKrSearchBrowser` launches Chromium through Playwright with locale `ko-KR`. For each keyword it opens:

```text
https://www.health.kr/drugs?search_word=<keyword>&page=<page>&limit=100
```

health.kr renders the product list with JavaScript. The crawler uses these checks before accepting a page:

1. It opens `about:blank` before each result page. This clears a stale Next.js DOM from the previous pagination request.
2. It waits for `DOMContentLoaded` and the page `<main>` element.
3. It polls the DOM every 250 milliseconds until `검색 중...` disappears and `총 N건` appears.
4. It counts unique links that match `/drug/{alphanumeric-code}`.
5. It checks the active pagination button and expects 100 links on a full page. The last page must contain the remaining count.
6. It requires the reported total to stay constant across pagination pages.

The browser retries a failed result page up to three times. The first retry waits one second; the second retry waits two seconds.

Discovery extracts:

- product code from the `/drug/{code}` path;
- Korean product name from the first line of the result link;
- absolute product URL;
- one-based rank within that keyword's result set.

The crawler removes duplicate product codes within one keyword. It does not remove a product that appears under two different keywords or diseases. `total_products` and `records.jsonl` can therefore contain the same product code more than once when the configuration searches overlapping terms.

### 3. Validate discovery counts

Strict mode requires all three counts to match:

```text
configured expected_count == health.kr reported_count == unique discovered products
```

The crawler stops before full-text collection when the counts differ. `--allow-count-mismatch` disables this stop while retaining the three counts in `discovery.json`.

The crawler writes discovery output to a temporary file and replaces the destination after the JSON document completes.

### 4. Fetch each product detail page

`HealthKrDetailClient` accepts URLs with this shape:

```text
https://www.health.kr/drug/<product-code>
```

It rejects another scheme, host, or path. The HTTP session sends the crawler user agent and `Accept-Language: ko-KR,ko;q=0.9`.

The client applies these request controls:

- a 1.5-second delay by default, with a 0.5-second minimum;
- a 45-second request timeout by default;
- three retries for connection, read, and status failures;
- retry status codes `429`, `500`, `502`, `503`, and `504`;
- exponential backoff with `Retry-After` support.

After a successful response, the client records `crawled_at` in ISO 8601 format using the `Asia/Seoul` time zone.

### 5. Extract the requested Korean fields

`extract_drug_fields()` parses HTML with Beautiful Soup and the built-in `html.parser`.

The extractor reads product metadata from exact Korean labels such as `제품명` and the adjacent value node. It reads `효능 · 효과` and `용법 · 용량` from their matching `h3` document sections. It reads the medication guidance from the block headed `이 약은 어떻게 복용합니까?`.

The extractor divides `사용상의 주의사항` by `cautionGroup` headings. It removes leading ordinals such as `1.` and `2)` before matching headings. It recognizes source variants for these sections:

- adverse reactions: `이상반응`, `이상사례`, or `부작용`;
- pregnancy and lactation: headings containing `임부` or `임신`;
- children: headings containing `소아` or `어린이`;
- elderly patients: headings containing `고령자` or `노인`;
- overdose: headings containing `과량투여` or `과다복용`, including spacing variants.

If multiple caution groups match one output field, the crawler joins their text values with a blank line. It does not store the matched source titles in the product record.

Text cleanup removes scripts, styles, `noscript` content, and matched navigation links such as `상세보기` and `동일성분 의약품`. It converts non-breaking spaces, collapses horizontal whitespace, retains line breaks, and removes adjacent duplicate lines. The extractor ignores page content outside the matched fields, including menus and images.

`extract_source_attribution()` reads the value after `자료출처:`. A page without that label produces `null`.

### 6. Write durable JSONL records

The crawler writes one compact schema header as the first line. The header contains the file-wide values and defines the order of the product metadata array. It then writes one compact JSON object per product. Serialization uses `ensure_ascii=False`, so Korean text remains readable.

The crawler flushes the file and calls `fsync()` after the header and after each product.

If a crawl stops midway, `records.jsonl` contains the records that completed before the failure. The crawler writes `manifest.json` only after it processes all discovered products.

## Output files

A full crawl directory contains:

| File | Format | Purpose |
| --- | --- | --- |
| `discovery.json` | formatted JSON document | Search inputs, live counts, product names, product codes, product URLs, and ranks. |
| `records.jsonl` | JSON Lines | One schema header followed by one extracted record per product. |
| `manifest.json` | formatted JSON document | Run completion time, product count, physical JSONL line count, search count, language, and output path. |

`discovery.json` includes the live health.kr `reported_count`. A product's positional `expected_product_count` value comes from the configuration, so it describes the configured expectation rather than a second live count.

## `records.jsonl` encoding and framing

- Encoding: UTF-8 without ASCII escaping.
- Framing: one JSON object per physical line. Line 1 is the file header; lines 2 onward are products.
- Top-level container: none. JSONL does not wrap records in an array.
- Embedded text line breaks: JSON encodes them as `\n` inside the string, so one product still occupies one physical line.
- Record order: configuration order, then health.kr result rank within each keyword.
- Null handling: a missing source field uses `null` as the leaf value.
- Missing-field index: `missing_fields` lists each null leaf as `division.field_name`.
- Product count: subtract the one header line from a raw `wc -l` result, or filter by `record_type` with `jq`.

Example commands for reading the file:

```bash
# Count product records, excluding the header
jq -c 'select(.record_type != "healthkr_records_header")' \
  tools/healthkr_crawler/output/demo_full/records.jsonl | wc -l

# Print the first product name
sed -n '2p' tools/healthkr_crawler/output/demo_full/records.jsonl \
  | jq -r '.fields.identification_of_drugs.product_name'

# Count records by disease
jq -r 'select(.record_type != "healthkr_records_header") | .disease' \
  tools/healthkr_crawler/output/demo_full/records.jsonl \
  | sort | uniq -c
```

## JSONL schema version 2.0

Schema version 2.0 removes repeated file-level keys and field-label wrappers. The file has two line types: one header and one or more product records.

### Line 1: records header

```json
{"record_type":"healthkr_records_header","schema_version":"2.0","language":"ko","source_site":"약학정보원","record_metadata_fields":["expected_product_count","result_rank","product_code","source_attribution"],"field_value_format":"scalar_or_null"}
```

| Header field | Type | Description |
| --- | --- | --- |
| `record_type` | string | Identifies the line as the `healthkr_records_header`. |
| `schema_version` | string | File schema version. Current value: `2.0`. |
| `language` | string | Source language. Current value: `ko`. |
| `source_site` | string | Source site. Current value: `약학정보원`. |
| `record_metadata_fields` | array of strings | Defines the position and meaning of each value in every product's `record_metadata` array. |
| `field_value_format` | string | Declares that leaves under `fields` contain a string or `null`. |

The crawler writes `schema_version`, `language`, and `source_site` once because these values apply to the full file.

The values for `expected_product_count`, `result_rank`, `product_code`, and `source_attribution` can differ between products. The header writes those four names once. Each product stores the corresponding values in `record_metadata`, in the same order:

| Array index | Header name | Value type | Meaning |
| ---: | --- | --- | --- |
| `0` | `expected_product_count` | integer | Configured expected count for the product's search keyword. |
| `1` | `result_rank` | integer | One-based product rank within that keyword's results. |
| `2` | `product_code` | string | Code parsed from the health.kr `/drug/{code}` URL. |
| `3` | `source_attribution` | string or null | Text extracted after `자료출처:` on the detail page. |

For example, this array:

```json
[11, 1, "2016122200009", "식품의약품안전처"]
```

maps to:

```text
expected_product_count = 11
result_rank            = 1
product_code           = 2016122200009
source_attribution     = 식품의약품안전처
```

Read metadata by pairing the two arrays instead of hard-coding the current positions:

```python
metadata = dict(zip(header["record_metadata_fields"], product["record_metadata"], strict=True))
print(metadata["product_code"])
```

### Lines 2 onward: product records

| Product field | Type | Description |
| --- | --- | --- |
| `disease` | string | Disease group copied from the input specification. |
| `search_keyword` | string | Keyword that discovered this product. |
| `record_metadata` | array | Four values defined by the header's `record_metadata_fields` order. |
| `product_url` | string | Public product detail URL. |
| `crawled_at` | string | Successful detail-fetch time in ISO 8601 `Asia/Seoul` format. |
| `fields` | object | Korean drug text grouped by division. Each leaf is a string or `null`. |
| `missing_fields` | array of strings | Dotted paths for requested fields whose source value was absent. |

### Field values

Each leaf under `fields` contains the Korean value itself:

```json
"efficacy_effects": "고혈압 치료"
```

A source page without the requested section produces:

```json
"warning": null
```

The JSON path supplies the field meaning, so product records do not repeat `source_label_ko` or `value_ko`.

### Requested field map

| Division | JSON path under `fields` | health.kr label or match |
| --- | --- | --- |
| Identification | `identification_of_drugs.product_name` | `제품명` |
| Identification | `identification_of_drugs.ingredients_content` | `성분 / 함량` |
| Identification | `identification_of_drugs.professional_general` | `전문 / 일반` |
| Identification | `identification_of_drugs.single_complex` | `단일 / 복합` |
| Dosage | `dosage_instructions.efficacy_effects` | `효능 · 효과` |
| Dosage | `dosage_instructions.usage_dosage` | `용법 · 용량` |
| Dosage | `dosage_instructions.how_to_take` | `이 약은 어떻게 복용합니까?` |
| Safety | `safety_information.contraindicated_patients` | heading containing `투여하지 말` |
| Safety | `safety_information.cautious_administration_patients` | heading containing `신중히 투여` |
| Safety | `safety_information.warning` | `경고` |
| Safety | `safety_information.adverse_reactions` | `이상반응`, `이상사례`, or `부작용` |
| Safety | `safety_information.interaction` | heading containing `상호작용` |
| Safety | `safety_information.pregnant_lactating_women` | heading containing `임부` or `임신` |
| Safety | `safety_information.children` | heading containing `소아` or `어린이` |
| Safety | `safety_information.elderly` | heading containing `고령자` or `노인` |
| Safety | `safety_information.drug_safety_information_dur` | `의약품안전성 정보(DUR)` |
| Emergency | `emergency_information.overdose_treatment` | heading containing `과량투여` or `과다복용` |

### Complete product record shape

The shortened values below show the full product structure. The schema header precedes this line in the JSONL file.

```json
{
  "disease": "고혈압",
  "search_keyword": "Doxazosin",
  "record_metadata": [
    11,
    1,
    "2016122200009",
    "식품의약품안전처"
  ],
  "product_url": "https://www.health.kr/drug/2016122200009",
  "crawled_at": "2026-11-06T17:45:01+09:00",
  "fields": {
    "identification_of_drugs": {
      "product_name": "제품명 원문",
      "ingredients_content": "성분 및 함량 원문",
      "professional_general": "전문",
      "single_complex": "단일"
    },
    "dosage_instructions": {
      "efficacy_effects": "효능 · 효과 원문",
      "usage_dosage": "용법 · 용량 원문",
      "how_to_take": "복약정보 원문"
    },
    "safety_information": {
      "contraindicated_patients": "투여 금기 원문",
      "cautious_administration_patients": "신중 투여 원문",
      "warning": null,
      "adverse_reactions": "이상반응 원문",
      "interaction": "상호작용 원문",
      "pregnant_lactating_women": "임부 및 수유부 원문",
      "children": "소아 투여 원문",
      "elderly": "고령자 투여 원문",
      "drug_safety_information_dur": "DUR 원문"
    },
    "emergency_information": {
      "overdose_treatment": "과량투여 처치 원문"
    }
  },
  "missing_fields": [
    "safety_information.warning"
  ]
}
```

### Schema 1.0 files

Existing crawl output does not change in place. Files produced before the schema 2.0 code change still contain `schema_version: "1.0"` on each product and use `{source_label_ko, value_ko}` leaf objects. Run a new crawl to produce schema 2.0 output. Do not combine schema 1.0 and 2.0 product lines in one file.

## Post-processing

### Split one crawl into five disease files

`split_records.py` reads the JSONL stream one line at a time, validates each object and its `disease`, and writes five files through temporary paths. For schema 2.0 input, it copies the header to line 1 of each disease file. It also accepts older headerless JSONL files. It replaces final files after the full split succeeds.

```bash
.venv/bin/python -m tools.healthkr_crawler.split_records \
  --input tools/healthkr_crawler/output/demo_full/records.jsonl \
  --output-dir tools/healthkr_crawler/output/demo_full/by_disease
```

The output names are:

```text
고혈압.jsonl
당뇨병.jsonl
비만.jsonl
이상지질혈증.jsonl
항혈전제_심뇌혈관질환_예방_목적.jsonl
```

### Split large disease files for RAGFlow upload

RAGFlow rejects files above 128 MiB. `split_jsonl_by_size.py` validates each JSON object and starts a new part before the next product would cross the byte limit. For schema 2.0 input, it writes the same header as line 1 of each part and includes the header bytes in its size calculation. Each upload part can therefore be read on its own. The splitter keeps each JSONL line intact and leaves the source file unchanged.

```bash
.venv/bin/python -m tools.healthkr_crawler.split_jsonl_by_size \
  --output-dir tools/healthkr_crawler/output/demo_full/by_disease/upload_parts \
  --max-size-mib 100 \
  tools/healthkr_crawler/output/demo_full/by_disease/고혈압.jsonl \
  tools/healthkr_crawler/output/demo_full/by_disease/당뇨병.jsonl
```

The 100 MiB setting leaves space below RAGFlow's 128 MiB limit. The splitter names parts as `<source-name>.part-001.jsonl`, `<source-name>.part-002.jsonl`, and so on.

## Test and lint

```bash
.venv/bin/pytest -q test/unit/healthkr_crawler
uvx ruff check tools/healthkr_crawler test/unit/healthkr_crawler
```

The tests cover scalar field extraction, legacy caution headings, missing values, source attribution, schema-header framing, positional metadata, search-page size calculations, disease splitting, and byte-limited JSONL splitting.
