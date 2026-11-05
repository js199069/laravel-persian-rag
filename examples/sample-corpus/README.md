# Sample corpus

Six **fictional** Persian documents for a made-up institution ("مؤسسه آموزش عالی نمونه"), written for testing `laravel-persian-rag`. Any resemblance to real regulations is coincidental.

> شش سند **ساختگی** فارسی برای یک مؤسسه‌ی خیالی، فقط برای آزمون این پکیج.

## What it covers

| id | Document | Audience | Format |
|---|---|---|---|
| sample-01 | Undergraduate academic regulations | all_students | text PDF |
| sample-02 | Student disciplinary rules | all_students | **scanned PDF** (OCR path) |
| sample-03 | PhD regulations | phd | text PDF |
| sample-04 | Faculty misconduct rules | faculty | text PDF |
| sample-05 | Staff leave policy | staff | DOCX |
| sample-06 | Internship guide | bachelor | DOCX |

The audiences match the package's default roles, so it works without any configuration.

`eval.csv` has 18 questions: 12 answerable (including three phrasings of the same question, and two answered only by the scanned document), 3 unanswerable but on-topic, and 3 access-control checks (a bachelor student must never see faculty, staff, or PhD documents).

`source/` holds the editable DOCX originals.

## Try it

From a Laravel app with the package installed and `php artisan rag:health` green:

```bash
mkdir -p storage/app/corpus
cp vendor/js199069/laravel-persian-rag/examples/sample-corpus/{*.pdf,*.docx,manifest.csv} storage/app/corpus/
cp vendor/js199069/laravel-persian-rag/examples/sample-corpus/eval.csv storage/app/eval.csv

php artisan rag:extract
php artisan rag:quality
php artisan rag:refine
php artisan rag:index --fresh
php artisan rag:eval
php artisan rag:eval-answers
```

> `examples/` is excluded from the Composer package (`export-ignore`). If you installed from Packagist, clone the repository to get these files.

## Expected results

Measured with `bge-m3` (local, Ollama) for embeddings and a Qwen 35B-A3B model for answers:

| Metric | Result |
|---|---|
| Correct document ranked first | 12 / 12 |
| Answers with the expected keyword | 12 / 12 |
| Citations to the correct document | 12 / 12 |
| Unanswerable questions refused | 3 / 3 |
| Access-control leaks | 0 |
| Citations to non-existent sources | 0 |

`rag:refine` will warn that the lexicon is small and skip noise cleanup. That is expected for a six-document corpus.