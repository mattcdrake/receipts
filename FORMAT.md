# receipts — On-Disk Format

**Format version: 1**

This document specifies the on-disk format used by `receipts`. The format is the source of truth; the tool is one implementation. A vault must remain readable as plain JSON and image files, decades from now, without this tool.

---

## Vault Layout

```
vault/
  vault.json
  reimbursements.jsonl
  2026/
    2026-04-22-001/
      receipt.json
      image-1.jpg
    2026-04-22-002/
      receipt.json
      image-1.pdf
      image-2.pdf
  2027/
    2027-01-03-001/
      receipt.json
      image-1.png
```

- `vault.json` — vault metadata. Its presence identifies the folder as a vault. Required.
- `reimbursements.jsonl` — append-only log of reimbursement events. Required (may be zero bytes).
- `YYYY/` — one directory per calendar year. Year MUST equal the year of the contained receipts' `date`.
- `YYYY-MM-DD-NNN/` — one receipt per directory. Name is the receipt's ID.

A folder lacking `vault.json` is not a vault.

---

## Receipt IDs

```
YYYY-MM-DD-NNN
```

- `YYYY-MM-DD` is the receipt's `date`.
- `NNN` is a per-day sequence, zero-padded to at least three digits, starting at `001`. May exceed three digits if a date has more than 999 receipts.

The directory name, the `id` field in `receipt.json`, and the enclosing year directory MUST all agree. Disagreement is a hard error.

---

## `vault.json`

```json
{
  "format_version": 1,
  "created": "2026-04-22"
}
```

| Field | Type | Notes |
|---|---|---|
| `format_version` | integer | The format version this vault is written in. |
| `created` | string (date) | ISO 8601 date the vault was initialized. |

All fields required.

---

## `receipt.json`

```json
{
  "format_version": 1,
  "id": "2026-04-22-001",
  "date": "2026-04-22",
  "provider": "CVS Pharmacy",
  "amount_cents": 4287,
  "currency": "USD",
  "notes": "Rx — atorvastatin"
}
```

| Field | Type | Notes |
|---|---|---|
| `format_version` | integer | Currently `1`. |
| `id` | string | Receipt ID; equals enclosing directory name. |
| `date` | string (date) | ISO 8601 date shown on the receipt. |
| `provider` | string | Freeform merchant name. Not normalized; reports group by exact-match string. |
| `amount_cents` | integer | Positive integer in the smallest unit of `currency`. Refunds and zero-value receipts are not representable in v1. |
| `currency` | string | ISO 4217 three-letter code. |
| `notes` | string | What the expense was for. May be empty (`""`) but the field MUST be present. |

All fields required.

---

## Image Files

Files inside a receipt directory are named:

```
image-N.<ext>
```

Where `N` is a positive integer and `<ext>` is the original extension, lowercased.

- `N` MUST be contiguous from `1`. A receipt with `image-1.jpg` and `image-3.jpg` but no `image-2.jpg` is malformed.
- Pages MAY have mixed extensions (`image-1.jpg`, `image-2.pdf`).
- Order is numeric on `N`. Tools handling more than nine pages must sort numerically, not lexicographically.

Image filenames are not listed in `receipt.json`; they are discovered by globbing the directory.

A receipt directory MAY have zero image files (e.g., entered before the scan is available); `validate` warns. Files other than `receipt.json` and `image-N.<ext>` are preserved but warned about during `validate`.

---

## `reimbursements.jsonl`

JSON Lines: one compact JSON object per line, no array wrapper.

```jsonl
{"date":"2027-03-15","receipt_id":"2026-04-22-001","amount_cents":4287,"currency":"USD","notes":"Q1 batch"}
{"date":"2027-03-15","receipt_id":"2026-04-22-002","amount_cents":1500,"currency":"USD"}
{"date":"2027-09-01","receipt_id":"2026-04-22-001","amount_cents":2000,"currency":"USD","notes":"partial top-up"}
```

| Field | Type | Notes |
|---|---|---|
| `date` | string (date) | ISO 8601 date the reimbursement occurred. |
| `receipt_id` | string | ID of an existing receipt. |
| `amount_cents` | integer | Positive integer in the smallest unit of `currency`. |
| `currency` | string | ISO 4217 code. MUST match the referenced receipt's `currency`. |
| `notes` | string | Optional. |

Each line is one event; a receipt may appear on multiple lines if reimbursed in parts. The file may be empty; a missing file is malformed and `init` creates it. The tool's `reimburse` command appends; users may hand-edit, but lines are events — removing one undoes that event.

### Computing Balance

For a receipt with amount `a` and `r` = sum of `amount_cents` from log entries referencing it:

- Reimbursed: `r`.
- Unreimbursed: `a − r`. Negative values (over-reimbursement) MUST be flagged by `validate`.
- Headline vault balance: sum of unreimbursed amounts across all receipts.

---

## Encoding

- **JSON files** (`vault.json`, `receipt.json`): pretty-printed, 2-space indent, fields in the order documented above for stable diffs. Readers MUST NOT depend on field order.
- **JSONL files** (`reimbursements.jsonl`): one compact JSON object per line.
- **All text files:** UTF-8 (no BOM), LF line endings, trailing newline.
- **Dates:** ISO 8601 calendar dates (`YYYY-MM-DD`). No times, no timezones.
- **Money:** integer minor units with explicit `currency`. Never float, never decimal string.
- **JSON:** RFC 8259. No comments, no trailing commas. Unknown fields MUST be preserved on rewrite.

---

## Versioning

A reader encountering a `format_version` higher than it understands MUST refuse to operate on that file rather than guess. The `vault.json` `format_version` is the floor for the vault; per-file values may exceed it during a partial migration.
