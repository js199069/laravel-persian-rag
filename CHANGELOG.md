# Changelog

## [Unreleased] — 0.1.0

### Added
- Text and OCR extraction for Persian PDF/DOCX, with per-page best-method selection and conservative OCR noise cleanup.
- Persian normalization (presentation forms, Arabic yeh/kaf, digits, reversed lam-alef, spelling variants).
- Hybrid search: pgvector + PostgreSQL full-text, Reciprocal Rank Fusion, per-document cap, best vector match always kept.
- Role-based access control inside the search query.
- Cited answers with two-layer refusal, citation validation, and retry on transient provider errors.
- Commands: `rag:extract`, `rag:quality`, `rag:refine`, `rag:index`, `rag:reindex-text`, `rag:search`, `rag:ask`, `rag:eval`, `rag:eval-answers`, `rag:health`.
- 41 tests, including access-control tests on real PostgreSQL; CI on PHP 8.3/8.4 × PostgreSQL 17/18.
- Fictional sample corpus with evaluation set (`examples/sample-corpus`): text PDF, scanned PDF, DOCX, four roles.
- `rag:eval` shows the configured threshold and how many answerable questions it would refuse.

### Changed
- Default `min_similarity` lowered from 0.55 to 0.50: 0.55 (tuned on one corpus) refused a correct answer on another; 0.50 refuses none on either, and the language model correctly rejects the extra unanswerable questions it now receives.

### Fixed
- `rag:eval` shows `-` instead of `0%` for page accuracy when no question has an expected page.