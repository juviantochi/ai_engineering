Borrower onboarding form diff between `documentation/Borrower onboarding - Application input fields(Current).csv` and `documentation/Borrower onboarding - Application input fields(Proposed).csv` (see supporting diff dump in `thoughts/form-diff.json`).

## Snapshot
- 9 new inputs, 1 field removed, and 51 existing inputs with behaviour/metadata changes.
- Compliance tagging was introduced: 32 fields are now explicitly “Yes”, 27 are “Not required”, and 1 is “Yes – but calculable internally”.
- Most demographic rows now state “pre-filled from KYC/KYB” plus UI hints (e.g. read-only), signalling tighter integration with Janus checks from `LENDING 2025 Q4 - Case_ New User Until Repayment.csv`.

## Newly introduced inputs
- **Family Card Number (Nomer KK)** – text, pre-filled via KYC/KYB, editable if OCR fails; marked compliance-critical.
- **Domicile Length of Stay** – optional `Text/Dropdown` for years lived at the current address (non-compliance).
- **Business Location** – dropdown describing industrial vs residential areas (mandatory + compliance).
- **Years running payment agent business** – numeric text (mandatory + compliance).
- **Monthly Income slider** – replaces manual entry with slider control (mandatory + compliance).
- **Employee Salaries** – numeric capture of payroll spend (mandatory but tagged “Remove!” pending confirmation).
- **Rental Costs** – optional numeric field for store rent.
- **Risk acknowledgement & Personal-data consent** – two new checkboxes under T&C, both digitally signed (compliance-required).

## Removed or deprecated
- **Estimated net monthly income** is dropped; use the new monthly income slider.
- Email, Religion, and NPWP question/number are still in the sheet but annotated “Remove!” + “Not required”, implying eventual deletion once backend tolerates missing values.

## Behaviour and validation shifts
- **Identity fields (Name, NIK, DOB, Gender, etc.)** are now read-only, pre-filled from Janus/kyc and marked compliance. DOB adds a POJK 18+ eligibility gate.
- **Loan limit request** switches from discrete dropdown (10/20/40/60M) to a 3–50M slider, while **loan purpose** becomes a dropdown of six enumerated options (working capital, salary, rent, renovation, expansion, inventory).
- **Emergency contact 2** entries (name/phone/relationship) become mandatory, with fixed relationship picklist for both contacts.
- **Address blocks (KTP & domicile)** expect OCR-prefill but stay editable; every row except the domicile equality toggle is now compliance-tagged.
- **Business metadata** (name, type, years, outlets, employees, etc.) and **bank details** now declare compliance coverage; growth metrics (store photos, NIB, Laku Pandai proof) become optional with rules like “prefill if KYB < 6 months”.
- **Document uploads** (front/selfie/interior photos, NIB) downshift from mandatory to optional when KYB recency is sufficient, or await Fadhly’s confirmation.
- **Psychometric** step marked “phase 2”, so expected later.
- **Terms & conditions checkbox** now explicitly “Can be ticked” and must include the GandengTangan link plus the two new consent clauses.

## Additional notes / nice-to-haves
- The compliance column clarifies what must stay surfaced to borrowers vs what can be auto-derived. Only one item (“Business age in months”) is “Yes – but can be calculated internally”.
- Highlighted “Remove!” annotations (email, religion, NPWP, employee salary) need backend alignment before UI hides them to avoid schema drift.
- Conditional statements (e.g. KYB < 6 months → pre-fill attachments) should be wired to Janus/LOS data so merchants aren’t re-uploading verified docs.
- Consider telemetry for the new slider/dropdowns to ensure defaults honour historical approval bands.

Next steps: align DTO/back-end schema with the above, decide whether to drop the “Remove!” fields now or guard them server-side, and confirm how Janus feeds the new read-only values.
