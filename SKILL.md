---
name: bucket-special-terms
description: Fact bucket for the NEGOTIATED terms - every clause of the Special Terms / Additional Terms sections captured verbatim in order and routed to the tracker columns it affects, rights over future purchases (renewal price caps and holds, future discounts, fixed-price options, MFN, free periods, credits toward future orders), and non-standard modifications found outside those sections (body insertions, overrides of the master agreement, side-letter references, customer-paper deviations). Feeds the Special Terms comparison with Salesforce, future-discount / material-right checks, Other Modifications and Non-Standard Contract.
metadata:
  bucket_id: special_terms
  schema_version: 2.0.0
  output_model: special_terms_models.py (SpecialTermsOutput)
  brd_refs: Sec 4.2.1, 4.2.2, 6.1.2, 6.1.3, 6.3.1, 6.4.1
  tracker_rows: 26, 32, 40, 58 (via Other Modifications); routes clauses to 7, 18, 21, 25, 29, 30, 31, 33, 35, 36, 39, 49, 50, 55, 57
---

# Bucket: Special Terms

This skill is self-contained: Part A is how to extract (shared by every bucket), Part B is what
this bucket extracts, Part C is the exact output structure.

# Part A - How to extract

You extract facts for **one bucket** (Special Terms) from **one document**. Part B says which facts,
Part C gives the exact output structure, and the output is validated by `special_terms_models.py`
(`SpecialTermsOutput`). Your output feeds three kinds of downstream checks:

| Output section | Downstream consumer | Your job |
|---|---|---|
| `values` | Fact rules in code (exact match, tolerance, date difference) | Transcribe the single value exactly + light parse |
| `records` | Calculation rules in code (sums, joins, period comparisons) | Transcribe every row of a repeating structure, completely, with stable keys |
| `evidence` | Contextual agent (policy overrides, deviations, judgements) | Capture verbatim clauses and signals; give non-binding hints only |

**Golden rule: you transcribe; code and the contextual agent decide.**

## A1. Input
- An OCR reconstruction of the PDF in **markdown**. Pages are separated by page markers
  `<<<PAGE n>>>`, where `n` is the 1-based physical page of the PDF.
- Markdown formatting in the input (`#` headings, `**bold**`, `_italic_`, `|` table cells, `-` bullets,
  `>` quotes, backticks, `<br>`, `\` escapes) is layout produced by the OCR step, not document text.

Use nothing else: not Salesforce, not Anaplan, not other documents, not general knowledge about the
customer. If a fact is not in this document, it is `not_found`.

## A2. Applicability
Read Part B section 1 and the document.
- Bucket does not apply -> `applicable: false`, a one-sentence `applicability_reason`, and
  `values`, `records`, `evidence` set to `null`. `observations` and `notes` are `[]`.
- Bucket applies -> `applicable: true` and fill **every** field Part B defines.

"Applies" means the document is the kind that could contain these facts, not that it does.
Silence in an applicable document is evidence (no opt-out clause, no budget cap), so it is reported
as `not_found`, never skipped.

## A3. Citations
Every `found`, `inferred` or `unreadable` fact carries `sources`, a list of
`{"page_number": int, "quote": [string, ...]}`. The quotes are used to **highlight the fact in the
original PDF**, so they must match the OCR text exactly.

**Rules for each string in `quote`:**
1. It is a verbatim span from **one single line** of the page. Never join two lines in one string.
2. It contains **plain text only**: no markdown syntax (`#`, `**`, `__`, backticks, `>`, leading
   `-` / `*` / `+` bullets, `<br>`, HTML tags, `\` escapes) and **never a `|` character**. Drop the
   formatting characters and keep the words between them: `**Order Schedule Number:** ACM-1`
   is quoted as `Order Schedule Number: ACM-1`.
3. Copy the characters exactly as the OCR text has them, including OCR typos, casing, punctuation,
   currency symbols and number formatting. No ellipses, no paraphrase, no translation.
4. Make it long enough to identify the fact unambiguously on the page (label + value, or the
   sentence that states it), but no longer than the line.

**When a fact spans several lines or cells, use one string per line or cell, in reading order:**
- Address block on 5 lines -> 5 strings.
- Label on one line, value on the next -> 2 strings.
- Table row -> one string per relevant cell (`| Solution Architect | 200 |` ->
  `["Solution Architect", "200"]`).
- A printed value that itself contains `|` (e.g. DocuSign `10-Apr-26 | 5:35:23 PM EDT`) -> split at
  the pipe: `["10-Apr-26", "5:35:23 PM EDT"]`. The `raw` field still keeps the full printed value.
- A fact on two pages -> two source objects, one per page.

**By status:**
- `inferred`: quote the text you inferred from. Never invent text for something that is absent.
- `unreadable`: quote the partially legible text as the OCR gives it; if nothing is legible, use
  `["[illegible]"]` with the page number.
- `not_found` / `not_applicable`: `sources` is `[]`.

## A4. Status and confidence
| Status | Use when | raw / parsed |
|---|---|---|
| `found` | Stated in the document | raw = as printed (formatting removed); parsed = light parse |
| `inferred` | Not stated directly but follows from specific text | raw = the text relied on; parsed = your reading; confidence at most `medium` |
| `not_found` | Bucket applies, document is silent | both null, no sources, no candidates |
| `not_applicable` | This field cannot exist in this document type | both null, no sources, no candidates |
| `unreadable` | Text is there but illegible | raw = what is legible; parsed = null |

`confidence`: `high` = explicit and unambiguous; `medium` = explicit but wording or formatting
leaves some doubt, or inferred; `low` = conflicting or barely legible.

## A5. Light parsing
`raw` is the truth (as printed, markdown removed); `parsed` is a convenience that code re-checks.
Put `parsed: null` rather than guessing.

| Type in Part C | parsed format | Rules |
|---|---|---|
| Text | string | Trim whitespace and line breaks; do not rewrite. |
| Number | number | Remove thousands separators; respect decimal commas (`1.234,56` -> 1234.56). Percentages in percent units (`99.5%` -> 99.5). |
| Boolean | true / false | Only from explicit wording; otherwise null. |
| Date | `YYYY-MM-DD` | Only if unambiguous. `03/04/2026` with no locale evidence -> null + `normalization` note. |
| Time | `HH:MM:SS` (24 h) | Seconds `00` if not printed. |
| Timestamp | `YYYY-MM-DDTHH:MM:SS` | Local time as printed; **no conversion, no offset**. |
| UtcOffset | `+HH:MM` / `-HH:MM` | Only when printed as an offset or an unambiguous zone listed in Part B. |
| Money | `{"amount": number, "currency_iso": "USD"}` | ISO 4217. `$` alone is ambiguous -> `currency_iso: null` unless the document names the currency. |
| CountryCode | ISO 3166-1 alpha-2 (`US`, `GB`) | |
| SubdivisionCode | ISO 3166-2 (`US-IL`, `CA-ON`) | |
| LanguageCode | ISO 639-1 (`en`, `fr`) | |
| CurrencyCode | ISO 4217 (`USD`, `EUR`) | Only when the document names the currency or uses an unambiguous symbol (`€`, `£`). |
| enum (`"a" \| "b"`) | one listed value | If nothing fits, use `other` where listed and keep the wording in raw. Never force a fit. |

**Never compute.** No totals, sums, differences, term lengths, execution dates, discounts, FX or
time-zone conversions. If the document prints a total, transcribe it. Code does the arithmetic.

## A6. Conflicts -> candidates
If the document states the same fact differently in two places, put the reading you judge primary
in the value, every other reading in `candidates` (each with its own sources), set
`confidence: low`, and add a `conflict` note. Do not reconcile.

## A7. Records (repeating structures)
- One item per row / entry **as printed**. Do not merge, split, de-duplicate or re-order.
- `key`: follow the key rule in Part B. Keys must be unique within the collection.
- Leaves are `{"raw", "parsed"}`; `status` and `sources` sit on the item (quote the row's line or cells).
- A column that is blank for a row -> `{"raw": null, "parsed": null}`.
- `completeness`: `items_extracted` = number of items you output; `items_declared` = a count the
  document itself prints, else null; `continues_beyond_extraction` = true if the table visibly
  continues somewhere not in this document; `status` = `complete` / `partial` / `unknown`.
- `declared_totals`: every total / subtotal printed for the collection, transcribed.
- Empty collection: `items: []`, `items_extracted: 0`, status `complete`.

## A8. Evidence (for the contextual agent)
**Clauses**: capture each clause matching one of Part B's clause categories.
- `clause_id`: `C1`, `C2`, … in document order.
- `text_verbatim`: the clause text with markdown formatting removed and line breaks kept.
- Length: full text for categories marked ★ (full-verbatim). Otherwise up to 1,500 characters; if
  cut, set `text_truncated: true` and stop at a sentence end.
- `sources`: one string per line of the clause (and one source object per page).
- `overrides_hint`: fill it when the clause says it modifies, replaces, supplements or conflicts with
  another document or clause; otherwise null.

**Signals**: short factual pointers explaining why something may matter.
- `signal_id`: `S1`, `S2`, … Link `clause_id` when the signal comes from a captured clause.
- Record weak signals too. `description` states what is printed, not a conclusion.

**Hints**: one entry for every hint Part B defines, every time the bucket applies.
- `likely` must cite at least one `signal_id`; `unlikely` / `unclear` may have an empty list.
- Hints are leanings, not verdicts.

## A9. Observations and notes
- `observations`: useful facts this bucket does not define but that matter to contract review
  (cited). Do not repeat facts already in a field.
- `notes`: `conflict`, `normalization`, `missing` (expected but absent), `multi_document`,
  `truncation`, `other`, with `field_path` when about a specific field.

## A10. Fact paths
Downstream checks, agent findings and human overrides address facts by path:
`special_terms.values.<field>`, `special_terms.records.<collection>[<key>].<leaf>`,
`special_terms.evidence.clauses[<clause_id>]`, `special_terms.evidence.signals[<signal_id>]`.

## A11. What the validator checks
`special_terms_models.py` rejects the output if any of these fail; it is retried once with the errors,
then routed to a human.
1. Structure, field names, types and enum values match Part C exactly (no extra fields).
2. Every quote string is plain text (no `|`, no markdown syntax, no line break) and is found on a
   single line or table cell of its page.
3. Status rules: `found` / `inferred` / `unreadable` have sources; `not_found` / `not_applicable` have
   none and null raw / parsed; `inferred` is never `high` confidence.
4. `values` / `records` / `evidence` are null exactly when `applicable` is false.
5. `items_extracted` equals the number of items; record keys are unique; every item has a source.
6. Clause and signal IDs are unique and every reference to them exists.
7. Every hint appears exactly once; `likely` hints cite a signal; ★ clauses are not truncated.

# Part B - Special Terms

**Core rule: copy the negotiated terms completely and word for word, then route each clause.**
The Special Terms section is compared clause by clause with Salesforce `Extra_Special_Terms__c`
(#26): any added, removed or reworded clause means the document changed during signing. So every
clause is captured in full, never truncated or summarised, in printed order. Routing tells code and
the agent which tracker columns each clause affects; it is a pointer, not a verdict.

## 1. Applicability
**Applies** to any document that can carry negotiated terms: OS, SOW, change order, amendment,
addendum, SSA / customer-paper master agreement, marketplace private offer, partner agreement.

**Does not apply** to documents that cannot carry negotiated terms (a PO with no terms, a Deal Desk
email, a standard SLA exhibit or DPA incorporated by reference).

When it applies but the document has no special terms, still return the bucket with
`special_terms_status` = `none_stated` or `section_absent` and empty collections.

## 2. What counts as a special-terms section
A section headed "Special Terms", "Additional Terms", "Special Conditions", "Custom Terms",
"Addendum - Special Terms" or similar, or an addendum / exhibit whose purpose is to add or change
terms for this order. Standard printed form terms (the order's normal billing, term and product
sections) are **not** special terms; those belong to their own buckets.

## 3. Ownership boundaries
Other buckets extract the **structured facts** inside these clauses (opt-out rights in Term &
Termination, uplift in Pricing, workspace terms in Products & Entitlements, withholding in Billing,
and so on). This bucket owns:
- the **complete verbatim text** and order of the special-terms sections (#26);
- the **routing** of each clause to tracker columns;
- **future-purchase rights** (renewal price caps and holds, future discounts, fixed-price options,
  MFN, free periods, credits toward future orders), which no other bucket extracts;
- **non-standard wording outside** the special-terms sections (evidence clauses).

## 4. values (fact rules)
| Field | Type | Extraction notes |
|---|---|---|
| `special_terms_status` | enum `has_terms`, `none_stated`, `section_absent` | `none_stated` when the section exists but reads "None", "N/A" or is blank (quote the heading and that line). `section_absent`: status `found`, quoting the document title or last heading to show where you looked. |
| `special_terms_heading_raw` | Text | Heading as printed. Several sections -> the first here, others in `candidates`. |

## 5. records

### special_terms: one item per clause, in printed order
| Leaf | Type | Notes |
|---|---|---|
| `section_label` | Text | The section the clause sits in ("Special Terms", "Addendum A"). |
| `clause_number` | Text | As printed ("1", "4.2", "(c)"), null if unnumbered. |
| `heading` | Text | Clause heading if any ("Opt-Out."). |
| `text` | Text | raw = the **whole clause** with markdown removed and line breaks kept, never truncated; parsed = the same text on one line with single spaces (used for the Salesforce comparison). Include the heading words if they are part of the sentence. |
| `routing_categories` | enum[] | One or more categories from the table below. |
| `is_boilerplate` | Boolean | true only for the standard wording in section 6. |
| `override_target_raw` | Text | The document / clause this clause modifies, if it says so ("Section 6.1 of the Agreement"). |
| `override_relationship` | enum `modifies`, `replaces`, `supplements`, `conflicts`, `none` | `none` when the clause does not refer to another term. |

Key: `ST1`, `ST2`, … in printed order across all special-terms sections.
`sources`: one string per line of the clause (one source object per page).
Sub-clauses ((a), (b)) stay inside their parent clause unless they are printed as separate numbered clauses.

**Routing categories -> tracker column fed**
| Category | Use for | Feeds |
|---|---|---|
| `opt_out_termination` | Opt-out, termination for convenience, cancellation, non-appropriation | #21, #31 |
| `supersede` | Supersedes / cancels / replaces earlier orders | #7, Rule 6.1.2 |
| `discount_or_price_hold` | Current-deal discounts, price holds within the term | #33 |
| `future_purchase_right` | Renewal caps / holds, future discounts, fixed-price options, MFN, free renewal periods, credits toward future orders | #32, Rule 6.3.1 |
| `uplift` | Price increases during the term | #36 |
| `workspace` | Workspace size / allocation terms | #39 |
| `use_case` | Use-case limitations | #55 |
| `user_exchange` | Licence swaps / conversions | #49 |
| `payment_billing` | Payment days, invoicing, billing schedule, stubs | #30, #35 |
| `refund_credit_acceptance` | Refunds, credits, acceptance, right of return, dispute holds | #31 |
| `sla` | Uptime, SLA credits, termination for downtime | #50 |
| `withholding_tax` | Withholding / gross-up / tax terms | #57 |
| `marketplace` | Marketplace / private offer terms | #25 |
| `purchase_order` | PO requirements | #18 |
| `free_deliverable` | Free services, workshops, training, custom work | #29 |
| `services_delivery` | Services delivery mechanics | SOW review |
| `entity_or_party` | Affiliates, resellers, end users, entity changes | #8, #48 |
| `master_terms_override` | Changes the SSA / master agreement for this order | #28, Rule 4.2.2 |
| `other_rev_rec_relevant` | Anything that could affect revenue recognition and fits no category above (contingent fees, free months, customer-specific restrictions, unusual IP or deliverable terms) | #40 Other Modifications |
| `boilerplate_no_impact` | Only for the boilerplate in section 6 | none |

### future_purchase_rights: one item per right over future or optional purchases
| Leaf | Type | Notes |
|---|---|---|
| `right_type` | enum `renewal_price_cap`, `renewal_price_hold`, `future_discount`, `fixed_price_option`, `mfn`, `free_period`, `future_order_credit`, `other` | |
| `special_term_key` | Text | The `special_terms` key it comes from, null if it comes from elsewhere. |
| `product_key` | Text | Same rule as Pricing / Products (SKU upper-cased, else normalised name), if a product is named. |
| `percent` | Number | Cap / discount %, percent units. |
| `price` | Money | Fixed / option price per unit. |
| `quantity` | Number | Units the right covers. |
| `valid_until` | Date | Only if a date is printed. |
| `applies_to_raw` | Text | "upon renewal", "during the Subscription Term", "any future Order Schedule". |

Record rights from anywhere in the document, not only the special-terms section.

## 6. Boilerplate (`is_boilerplate` = true, routing `boilerplate_no_impact`)
Only wording equivalent to:
- "The discount … is a one-time discount and does not apply to future Order Schedules / renewals."
- "Pricing in this Order Schedule is confidential."
- Standard references to the Anaplan DPA, SLA or SSA incorporated without changes.
Anything else is not boilerplate, even if it looks routine.

## 7. evidence (contextual agent)
**Clauses** here are ONLY non-standard wording **outside** the special-terms sections. Clause text
inside those sections is already in `records.special_terms`; do not repeat it.
Categories (all ★ full-verbatim): `off_section_modification` (custom terms typed into a normal
section), `master_terms_override` ("notwithstanding Section 6.1 of the Agreement…"),
`side_letter_reference` (refers to a separate letter / email / agreement that changes terms),
`customer_paper_deviation` (customer paper terms that depart from the Anaplan standard reference
clauses, Rule 4.2.2), `handwritten_or_inserted_text`, `other_modification`.

**Signal types**:
| Type | Record when |
|---|---|
| `future_discount_language` | Discount promised on future orders or renewals |
| `renewal_cap_language` | Renewal price capped or held |
| `mfn_language` | Most-favoured-customer / price-matching wording |
| `fixed_price_option_language` | Option to buy more at a fixed or discounted price |
| `free_period_language` | Free months / free renewal period |
| `free_deliverable_language` | Free services, workshops, training or custom work |
| `master_terms_override` | A clause changes the master agreement for this order |
| `side_letter_reference` | A separate letter / email / agreement changes terms |
| `inserted_text` | Text typed or handwritten into the form outside special terms |
| `unclassified_rev_rec_term` | A term that may affect revenue recognition and fits no routing category |
| `translated_terms` | Special terms appear in more than one language |
| `other` | Anything else a reviewer should see |

**Hints** (each exactly once):
| Hint | Lean on… | Feeds |
|---|---|---|
| `future_purchase_rights` | The customer has rights over future or optional purchases | #32 (likely -> 'N' + clause + %) |
| `other_modifications` | A rev-rec-relevant modification is not captured by any specific tracker column | #40 Other Modifications -> #58 Non-Standard Contract |
| `master_terms_overridden` | This document changes the master agreement's terms | #28 SSA review, Rule 4.2.2 |

## 8. Code-side derivations (never done by the model)
- `section_text` = `special_terms[*].text.parsed` joined in key order.
- Clause-by-clause comparison with SF `Extra_Special_Terms__c` (split into clauses, normalised for
  whitespace and punctuation): MATCH, or MISMATCH with added / removed / reworded clauses.
- Routing table: for each tracker column, the special-terms keys routed to it.
- Material-right test: each `future_discount` / `fixed_price_option` compared with the current-deal
  discount (Pricing + SF list prices).

## 9. Validations fed (rule registry)
| Rule | Type | Check | Fact paths |
|---|---|---|---|
| #26 Special terms | calc | Clause-by-clause comparison with SF `Extra_Special_Terms__c` -> MATCH / MISMATCH + differences; `none_stated` / `section_absent` vs blank SF = MATCH | `records.special_terms[*].text`, `values.special_terms_status` |
| #26 Classification | fact | Each clause pushed to its tracker column(s) for the reviewer | `records.special_terms[*].routing_categories` |
| #32 No significant future discounts | fact + context | Any future-purchase right -> 'N' + description + %; boilerplate one-time discount -> no future discount | `records.future_purchase_rights`, hint `future_purchase_rights` |
| Rule 6.3.1 Material right | calc | Future discount exceeds the current-deal discount by > 15 pts, or a net-new tier is offered -> Commissions / Bookings escalation (open point: basis of the 15%) | `records.future_purchase_rights` + Pricing + SF list prices |
| #40 Other Modifications | context | TRUE if any clause is routed `other_rev_rec_relevant`, any evidence clause exists, or the contract is customer paper with deviations; boilerplate never triggers | `records.special_terms[routing=other_rev_rec_relevant]`, `evidence.clauses`, hint `other_modifications` |
| #58 Non-Standard Contract | calc | OR roll-up including #40 | via #40 |
| Rule 4.2.2 Customer paper / overrides | context | Overrides of the master agreement and customer-paper deviations -> mandatory HITL | `records.special_terms[override_relationship != none]`, `evidence.clauses[master_terms_override, customer_paper_deviation]`, hint `master_terms_overridden` |
| Preparer summary (#63) | context | Draft agent summary for the preparer from routed clauses (e.g. "Renewal. Month-12 opt-out. Renewal cap 5%. No other rev-rec terms") | `records.special_terms` |

# Part C - Output structure

Return **one JSON object** exactly in this shape. Every key shown is required (use `null` /
`[]` as the rules allow); no other keys are allowed.

## C1. Shared shapes

```jsonc
Source        = { "page_number": int >= 1, "quote": [string, ...] }        // >= 1 plain single-line string
Value<T>      = { "raw": string | null, "parsed": T | null,
                  "status": "found" | "not_found" | "not_applicable" | "inferred" | "unreadable",
                  "confidence": "high" | "medium" | "low",
                  "sources": [Source], "candidates": [Candidate] }
Candidate     = { "raw": string, "parsed": string | number | boolean | Money | null, "sources": [Source] }
Leaf<T>       = { "raw": string | null, "parsed": T | null }
Money         = { "amount": number | null, "currency_iso": string | null }   // ISO 4217, 3 letters
Item          = { "key": string, "status": "found" | "inferred" | "unreadable", "sources": [Source] (>= 1), ...leaves }
Collection<I> = { "completeness": Completeness, "declared_totals": [DeclaredTotal], "items": [I] }
Completeness  = { "items_extracted": int, "items_declared": int | null,
                  "continues_beyond_extraction": boolean, "status": "complete" | "partial" | "unknown" }
DeclaredTotal = { "label_raw": string, "value": Leaf<Money>, "sources": [Source] }
Clause        = { "clause_id": string, "clause_number_raw": string | null, "heading_raw": string | null,
                  "text_verbatim": string, "text_truncated": boolean, "category_hints": [category],
                  "overrides_hint": OverridesHint | null, "sources": [Source] }
OverridesHint = { "target_document_raw": string | null, "target_clause_raw": string | null,
                  "relationship": "modifies" | "replaces" | "supplements" | "conflicts" | "unclear" }
Signal        = { "signal_id": string, "signal_type": signal_type, "description": string, "entity_raw": string | null,
                  "strength": "strong" | "moderate" | "weak", "clause_id": string | null, "sources": [Source] }
Hint          = { "hint": hint_name, "value": "likely" | "unlikely" | "unclear", "signal_ids": [string] }
Observation   = { "label": string, "value_raw": string, "sources": [Source] }
Note          = { "type": "conflict" | "normalization" | "missing" | "multi_document" | "truncation" | "other",
                  "field_path": string | null, "message": string }

Scalar types: Text = string, Number = number, Boolean = boolean, Date = "YYYY-MM-DD",
Time = "HH:MM:SS", Timestamp = "YYYY-MM-DDTHH:MM:SS", UtcOffset = "+HH:MM" | "-HH:MM",
CountryCode = "US", SubdivisionCode = "CA-ON", LanguageCode = "en", CurrencyCode = "USD", Money as above.
```

## C2. `special_terms` output

```jsonc
{
  "bucket": "special_terms",
  "schema_version": "2.0.0",
  "applicable": true,                // false -> values, records, evidence are null
  "applicability_reason": string,

  "values": {
    "special_terms_status": Value<"has_terms" | "none_stated" | "section_absent">,
    "special_terms_heading_raw": Value<Text>
  },

  "records": {
    "special_terms": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "section_label": Leaf<Text>,
      "clause_number": Leaf<Text>,
      "heading": Leaf<Text>,
      "text": Leaf<Text>,
      "routing_categories": [Leaf<"opt_out_termination" | "supersede" | "discount_or_price_hold" | "future_purchase_right" | "uplift" | "workspace" | "use_case" | "user_exchange" | "payment_billing" | "refund_credit_acceptance" | "sla" | "withholding_tax" | "marketplace" | "purchase_order" | "free_deliverable" | "services_delivery" | "entity_or_party" | "master_terms_override" | "other_rev_rec_relevant" | "boilerplate_no_impact">],
      "is_boilerplate": Leaf<Boolean>,
      "override_target_raw": Leaf<Text>,
      "override_relationship": Leaf<"modifies" | "replaces" | "supplements" | "conflicts" | "none">
    }>,
    "future_purchase_rights": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "right_type": Leaf<"renewal_price_cap" | "renewal_price_hold" | "future_discount" | "fixed_price_option" | "mfn" | "free_period" | "future_order_credit" | "other">,
      "special_term_key": Leaf<Text>,
      "product_key": Leaf<Text>,
      "percent": Leaf<Number>,
      "price": Leaf<Money>,
      "quantity": Leaf<Number>,
      "valid_until": Leaf<Date>,
      "applies_to_raw": Leaf<Text>
    }>
  },

  "evidence": {
    "clauses": [Clause],
    "signals": [Signal],
    "hints": [Hint]
  },

  "observations": [Observation],
  "notes": [Note]
}
```

Allowed lists for this bucket:

- `category` (clause `category_hints`): `off_section_modification` ★, `master_terms_override` ★, `side_letter_reference` ★, `customer_paper_deviation` ★, `handwritten_or_inserted_text` ★, `other_modification` ★
  (★ = full-verbatim, never truncated)
- `signal_type`: `future_discount_language`, `renewal_cap_language`, `mfn_language`, `fixed_price_option_language`, `free_period_language`, `free_deliverable_language`, `master_terms_override`, `side_letter_reference`, `inserted_text`, `unclassified_rev_rec_term`, `translated_terms`, `other`
- `hint_name` (each exactly once): `future_purchase_rights`, `other_modifications`, `master_terms_overridden`
