scope repository: @bnpl

background:
Veefin will be integrating with BukuWarung via fs-brick to obtain merchant enrichment data that has been consolidated according to LOps formula.
BNPL currently only stores merchant enrichment data status and it's S3 path that contains the response in form of json.

The task is:
- Get the data needed from AWS S3 according to the S3 path in `merchant_enrichment_data.s3Path`
- Consolidate the data according to LOps formula.
- Store consolidated merchant enrichment data into a new table `MERCHANT_DATA_ENRICHED`. referencing to `merchant_data` by merchant_id. 

Make sure:
- .
- .

requirements:
- 

References:
1. rfc/template.md
2. {ticket-number} ticket
3. etc.

**OUTPUT**
Only after the user asks to create the rfc, then :
1. output RFC (3000–5000 words) following rfc/template.md → save as `<ticket-number>.md` in `RFC/` folder  .
2. output scoped RFC in specific repository impacted by the change.
