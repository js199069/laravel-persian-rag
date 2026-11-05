# Laravel Persian RAG

[![tests](https://github.com/js199069/laravel-persian-rag/actions/workflows/tests.yml/badge.svg)](https://github.com/js199069/laravel-persian-rag/actions/workflows/tests.yml)
[![Latest Version](https://img.shields.io/packagist/v/js199069/laravel-persian-rag.svg)](https://packagist.org/packages/js199069/laravel-persian-rag)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**Answer questions from your organization's Persian documents, inside your Laravel app — with citations, role-based access control, and no answer when the documents don't have one.**

[فارسی](README.fa.md)

```php
$answer = app(Rag::class)->answer('یک واحد درس نظری چند جلسه است؟', role: $user->rag_role);

$answer['answer'];   // "یک واحد درس نظری طی ۱۶ جلسه یک ساعتی آموزش داده می‌شود [2]."
$answer['sources'];  // [2] => doc-04, "آیین‌نامه آموزشی ...", page 8
$answer['refused'];  // false — true when the documents don't contain the answer
```

- **Real Persian documents, as they are:** text PDFs, scanned PDFs (OCR), DOCX — including broken fonts, Arabic presentation forms, and reversed ligatures.
- **Hybrid search** on PostgreSQL: pgvector + full-text, fused with Reciprocal Rank Fusion.
- **Access control inside the SQL query**, not in the prompt. A student never sees a faculty-only document, however the question is phrased.
- **Cited answers** with page numbers, two-layer refusal, and detection of citations to sources that don't exist.
- **Provider-independent:** embeddings run locally (Ollama); answers come from any OpenAI-compatible endpoint — a local model, a GPU server, or a hosted gateway. Switch with one environment variable.
- **Measured, not claimed:** evaluation commands, 41 tests (including access control on real PostgreSQL), CI on PHP 8.3/8.4 × PostgreSQL 17/18.

---

## Results

Evaluated on two corpora. The first is real; the second you can reproduce yourself.

**Real corpus** — 31 public regulations from a large Iranian university (28 PDF + 3 DOCX, ~200 pages, 5 audiences; **55% of the documents were scanned images with no text layer**). The documents are not redistributed.

| Retrieval (30 questions) | | Answers (22 answerable, 5 unanswerable, 3 access) | |
|---|---|---|---|
| Correct document ranked first | 95% | Answers with the expected keyword | 10 / 11 |
| Correct document in top 5 | 100% | Citations to the correct document | 19 / 20 |
| Correct page | 93% | Unanswerable questions refused | 5 / 5 |
| MRR | 0.98 | Citations to non-existent sources | 0 |
| Access-control leaks | 0 | Access-control leaks | 0 |

**Sample corpus** — six fictional documents shipped in [`examples/sample-corpus`](examples/sample-corpus) (text PDF, scanned PDF, DOCX, four roles):

| Metric | Result |
|---|---|
| Correct document ranked first | 12 / 12 |
| Answers with the expected keyword | 12 / 12 (including 3 phrasings of one question, and 2 answered only by the scanned document) |
| Unanswerable questions refused | 3 / 3 |
| Access-control leaks / fake citations | 0 / 0 |

Setup: embeddings with `bge-m3` on a 2016 laptop CPU (i7-6700HQ, no GPU); answers with a Qwen 35B-A3B model through an OpenAI-compatible gateway. Search takes under 0.5 s; a full answer 2–9 s.

> A local 3B model (Qwen2.5-3B) was also tested for answers: retrieval found the right document for 100% of questions, but the model produced the expected answer for only 1 of 11. Retrieval runs well on a laptop; answer generation needs a larger model.

---

## Requirements

- PHP 8.3+ with `intl`, `mbstring`, `zip`, `pdo_pgsql`
- Laravel 13
- PostgreSQL 17 or 18 with [pgvector](https://github.com/pgvector/pgvector)
- [Ollama](https://ollama.com) with `bge-m3` (embeddings; must be running whenever you search)
- An OpenAI-compatible chat endpoint for answers
- For extraction: [Poppler](https://poppler.freedesktop.org) (`pdftotext`, `pdftoppm`) and [Tesseract](https://github.com/tesseract-ocr/tesseract) with the Persian (`fas`) language data

---

## Installation

**1. Install the package**

```bash
composer require js199069/laravel-persian-rag
```

**2. Enable pgvector** — once, as a PostgreSQL superuser, on your application's database:

```sql
CREATE EXTENSION vector;
```

The migration checks for the extension and tells you exactly this if it's missing.

**3. Publish the configuration files**

```bash
php artisan vendor:publish --tag=persian-rag-config
php artisan vendor:publish --tag=ai-config
```

> Use `--tag=ai-config`, not `--tag=ai` — the latter also publishes migrations from `laravel/ai` that this package doesn't need.

**4. Define two providers** in `config/ai.php`, inside `'providers' => [`:

```php
'ollama_local' => [
    'driver' => 'openai-compatible',
    'url' => env('OLLAMA_URL'),
    'models' => [
        'text' => ['default' => env('OLLAMA_CHAT_MODEL')],
        'embeddings' => ['default' => env('OLLAMA_EMBED_MODEL')],
    ],
],

'answers' => [
    'driver' => 'openai-compatible',
    'url' => env('RAG_ANSWER_URL'),
    'key' => env('RAG_ANSWER_KEY'),
    'models' => [
        'text' => ['default' => env('RAG_ANSWER_MODEL')],
    ],
],
```

> `config/ai.php` already contains a provider named `ollama`. **Don't use it here**: it doesn't accept a custom URL. The `openai-compatible` driver works with Ollama's `/v1` endpoint.

**5. Environment**

```ini
OLLAMA_URL=http://localhost:11434/v1
OLLAMA_CHAT_MODEL=qwen2.5:3b
OLLAMA_EMBED_MODEL=bge-m3

RAG_ANSWER_URL=https://your-openai-compatible-endpoint/v1
RAG_ANSWER_KEY=...
RAG_ANSWER_MODEL=...

RAG_LLM_PROVIDER=answers
```

`RAG_LLM_PROVIDER` is the name of the provider that writes answers. To use a local model instead, set it to `ollama_local`.

**6. Migrate and check**

```bash
ollama pull bge-m3
php artisan migrate
php artisan config:clear
php artisan rag:health --llm
```

`rag:health` checks every dependency and tells you what to do about anything that fails:

```
| intl       | ✅ | Unicode normalization enabled                 |
| database   | ✅ | PostgreSQL, connected                         |
| pgvector   | ✅ | version 0.8.7                                 |
| embedding  | ✅ | ollama_local, 1024 dimensions, 130ms          |
| answers    | ✅ | answers responded (1800ms): OK                |
| tesseract  | ✅ | tesseract                                     |
| roles      | ✅ | 4 roles; every document audience is covered   |
```

---

## Quick start with the sample corpus

The sample documents live in the repository, not in the Composer package. Clone the repository (or download `examples/sample-corpus` from GitHub), then from your Laravel app:

```bash
mkdir -p storage/app/corpus
cp /path/to/laravel-persian-rag/examples/sample-corpus/{*.pdf,*.docx,manifest.csv} storage/app/corpus/
cp /path/to/laravel-persian-rag/examples/sample-corpus/eval.csv storage/app/eval.csv

php artisan rag:extract
php artisan rag:quality
php artisan rag:refine
php artisan rag:index --fresh

php artisan rag:ask "یک واحد درس نظری چند جلسه است؟" --role=bachelor
php artisan rag:eval-answers
```

The sample uses the package's default roles, so no configuration is needed.

---

## Indexing your own documents

Put the files in `storage/app/corpus/` with a `manifest.csv`:

```csv
id,file,original_name,audience,title
doc-01,doc-01.pdf,آیین‌نامه آموزشی,all_students,آیین‌نامه آموزشی دوره کارشناسی
doc-02,doc-02.pdf,آیین‌نامه تخلفات,faculty,آیین‌نامه رسیدگی به تخلفات اعضای هیئت علمی
```

- `audience` decides who can see the document (see [Access control](#access-control)).
- `title` is optional but matters: it is embedded with every chunk. A descriptive title moved one question from a wrong result to rank 1 in our evaluation. File names like `scan003.pdf` hurt retrieval.
- The manifest may be UTF-8 or Windows-1256, with or without BOM, comma- or semicolon-separated (Excel).

Then run the pipeline:

| Command | What it does |
|---|---|
| `rag:extract` | Extracts text; runs OCR automatically on PDFs with no usable text layer |
| `rag:quality` | Builds a lexicon from your own documents and scores each one |
| `rag:refine` | Picks the better of text or OCR for each page; conservatively removes OCR noise |
| `rag:index` | Chunks, embeds, and stores in PostgreSQL (`--fresh` to rebuild) |
| `rag:reindex-text` | Rebuilds only the full-text column (seconds; no re-embedding) |

**Expect it to take a while on CPU:** OCR is 3–6 seconds per page; embedding ~1.7 seconds per chunk. A 200-page corpus took about 30 minutes end to end on a 2016 laptop.

> `rag:refine` skips noise cleanup when its lexicon has fewer than 400 words (small corpora), because a small lexicon mistakes real words for noise. You'll see a warning; this is intended.

---

## Usage

```php
use Js199069\PersianRag\Rag;

public function __construct(private Rag $rag) {}

// Search only
$found = $this->rag->search($question, role: 'bachelor', k: 5);

// Full answer with citations
$answer = $this->rag->answer($question, role: $request->user()->rag_role);
// $answer['answer'], ['sources'], ['cited'], ['refused'], ['refused_by'],
// ['invalid_citations'], ['best_similarity'], ['search_ms'], ['llm_ms']

// Streaming for the web: show sources first, then stream the answer
$prepared = $this->rag->prepare($question, role: $request->user()->rag_role);
// $prepared['sources'] is available immediately (under a second)
abort_if($prepared['refused'], 422);
return $this->rag->stream($prepared)->usingVercelDataProtocol();
```

**The role must come from the authenticated user**, never from request input — otherwise anyone can claim to be faculty. An unknown role throws `InvalidArgumentException`. `role: null` disables filtering and is meant for admin tools only.

From the command line: `php artisan rag:search` and `php artisan rag:ask` (both accept `--role` and `--file` with one question per line).

A complete web demo with streaming and clickable citations: [laravel-persian-rag-demo](https://github.com/js199069/laravel-persian-rag-demo).

---

## Access control

Each document has one `audience`; each role lists the audiences it may see. In `config/persian-rag.php`:

```php
'roles' => [
    'bachelor' => ['bachelor', 'all_students'],
    'phd'      => ['phd', 'all_students'],
    'faculty'  => ['faculty', 'phd', 'bachelor', 'all_students'],
    'staff'    => ['staff', 'phd', 'bachelor', 'all_students'],
],
```

**These defaults are an example from a university.** Replace them with your organization's roles. Also update `audience_labels` (the human-readable audience name embedded with each chunk), then rebuild with `rag:index --fresh`.

The filter is applied inside both the vector and the full-text query, so a document outside the user's audiences can never reach the model. A document whose audience matches no role is visible to nobody: `rag:index` warns about it, and so does `rag:health`.

---

## Tuning the refusal threshold

Answers are refused in two layers:

1. **Retrieval:** if the best similarity is below `min_similarity` (default `0.50`), the model is not called at all. This only catches clearly unrelated questions.
2. **The model:** it reads the sources and says when the answer isn't in them. This is the main defence.

Similarity alone cannot separate answerable from unanswerable questions: in our evaluation, unanswerable but on-topic questions scored up to 0.650, while answerable ones started at 0.599. **Tune the threshold on your own data:**

```bash
php artisan rag:eval
```

prints the current threshold, how many answerable questions it would wrongly refuse, and a table of alternatives. Set it so that *false refusals* is zero. A threshold of 0.55 tuned on one corpus refused a correct answer on another; that's why the default is 0.50.

---

## Evaluation

Write questions in `storage/app/eval.csv`:

```csv
id,question,role,expected_doc,expected_page,must_not_doc,expected_answer
1,یک واحد درس نظری چند جلسه است؟,bachelor,doc-04,8,,16|۱۶
2,حقوق هیئت علمی چه روزی پرداخت می‌شود؟,faculty,,,,
3,مصرف مواد مخدر توسط عضو هیئت علمی چه تخلفی است؟,bachelor,,,doc-08,
```

- Empty `expected_doc` → an unanswerable question (must be refused).
- `must_not_doc` → an access-control check (that document must never appear).
- Several correct documents: `doc-04|doc-12`. Several accepted keywords: `16|۱۶`.

| Command | Measures |
|---|---|
| `rag:eval` | Retrieval: rank of the correct document, page, MRR, leaks, threshold table. Deterministic, no LLM calls. |
| `rag:eval-answers` | Answers: false refusals, keywords, citations, refusals on unanswerable questions, fake citations. Saves results after every question and keeps going if one fails. |

Include **several phrasings of the same question**. A ranking bug in this package was invisible until a shorter phrasing of an existing question was asked.

---

## Why this architecture

Each decision below was driven by something that failed first.

**OCR is not optional.** 16 of 31 documents from a major university had no text layer at all. Others had broken fonts that look fine to simple checks. Extraction picks text or OCR per page, preferring text on ties because its numbers are more reliable.

**Noise removal is conservative.** An aggressive filter removed real content ("possession or use of narcotics", course names). The rule now: deleting real text is worse than keeping noise. Text-layer pages are never touched; numbered clauses always stay; every removed line is logged.

**Hybrid search, with the best semantic match guaranteed.** Vector search handles paraphrase; full-text handles exact terms and numbers. A per-document cap keeps one long document from filling all results — but that cap once dropped the single most relevant chunk. Now the chunk that the refusal threshold is based on is always passed to the model.

**Access control in SQL, not in the prompt.** A model can be talked out of an instruction; a `WHERE` clause cannot. Seven tests run against real PostgreSQL, and they were verified by deliberately disabling the filter: exactly the three tests that should fail, failed.

**Local embeddings, swappable answers.** Embeddings and search stay on your server, so full documents never leave it. Only the 3 retrieved chunks are sent to the answer model — and that model can be local too. Be aware of that trade-off for confidential documents.

**PostgreSQL instead of a separate vector database.** Vector search, full-text search, and access filtering in one query, in a database your team already runs.

**Changes are measured, and reverted when they don't help.** Splitting digits from words glued together by OCR ("16جلسه") seemed obviously right, but it pushed a wrong document up and dropped rank-1 accuracy from 95% to 90%. It was reverted, and a test records the decision.

---

## Limitations

- **Full-text ranking doesn't weight rare words properly.** Better tokenization can therefore hurt (see above). A proper IDF weighting or a reranker is the next step.
- **Tables in scanned PDFs** lose their structure.
- **Latin words inside Persian lines** can be reordered by `pdftotext` and garbled by OCR with `fas` only.
- **The quality lexicon depends on corpus size.** Small corpora skip noise cleanup; after `rag:refine`, re-running `rag:quality` on a corpus without DOCX files can produce an empty lexicon.
- **Ollama must be running** for search — the question itself is embedded locally.
- **Streaming responses don't retry** on provider errors (non-streaming answers retry twice).
- `rag:quality` and `rag:search` still assume the default corpus path (`storage/app/corpus`).
- The evaluation sets are small (30 and 18 questions): the numbers show direction, not precision.

---

## Testing

```bash
composer install
cp phpunit.xml.dist phpunit.xml   # set RAG_TEST_DB_PASSWORD
vendor/bin/phpunit
```

Integration tests need a PostgreSQL database with pgvector whose name ends in `_test` — they drop tables, and refuse to run against any other database.

---

## License

MIT. See [LICENSE](LICENSE).