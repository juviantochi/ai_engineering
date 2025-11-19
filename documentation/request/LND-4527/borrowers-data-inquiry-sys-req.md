repository scope: @fs-brick-service @bnpl

integration overview:
- Veefin calls `fs-brick-service`, which proxies or aggregates data from BNPL.
- BNPL now persists enrichment-derived credit summaries in `merchant_credit_summary`, keyed by `merchant_enrichment_data` (`batch_id`, `merchant_id`, `enrichment_type`).
- Each enrichment payload (FDC/PEFINDO) is processed by BNPL when fs-brick’s callback arrives; the service stores scalar fields (`total_facilities`, `worst_overdue_days`, `total_outstanding_balance`, etc.) alongside metadata (`ktp_number`, `source_provider`, `reference_id`, `data_timestamp`).
- `merchant_enrichment_data` retains the S3 path, status, and links to the merchant; it serves as the parent record for each summary row.

data sources:
- `merchant_data` table: supplies core merchant info (KTP number and profile).
- Enrichment JSON from S3 (FDC/PEFINDO shapes documented under `sample_FDC.json`, `sample_PEFINDO.json`).
- Persisted summary in `merchant_credit_summary` eliminates the need for fs-brick to derive metrics on the fly.

integration contract expectations:
- fs-brick queries BNPL over the existing `bnplPort` to fetch merchant credit summaries for Veefin’s borrower data inquiry.
- API response should map persisted metrics to Veefin fields (`jmlFasilitas`, `jmlHariTunggakanTerburuk`, `jmlSaldoTerutang`) and include status/reference metadata as required.

design decision (clarified):
- We store the derived data in BNPL to avoid re-computation latency and memory spikes in fs-brick. Storage overhead is acceptable given insert-once semantics and limited payload size.
