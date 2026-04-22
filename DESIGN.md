# receipts — Scope and Path

*A CLI for long-horizon HSA and charitable-donation receipt tracking. HSA first; donations to follow.*

---

## What this is

A command-line tool that stores receipts as plain JSON + image files in a local folder, tracks reimbursements, and reports the running unreimbursed balance. Built in Go. Operates on a local vault folder; cloud sync, if desired, is handled by whatever folder-sync tool the user already has.

Initial implementation covers HSA receipts. Charitable donations are planned as a follow-on, sharing the same architecture.

## Why this shape

The long-horizon HSA strategy requires saving receipts for 30–40 years. Existing tools are either vendor-locked SaaS (will not exist in 30 years), HSA-provider built-ins (lost on provider switch), or DIY spreadsheets (tedious).

A CLI operating on a documented file format is the most durable possible shape for this data. The tool can rot; the format cannot, because it is specified independently and the files are plain JSON and images in a normal folder hierarchy. The vault is readable without the tool.

Go fits the project well: it's mostly "read files, write files, parse JSON, format output," which the standard library covers almost entirely, and the single-static-binary distribution is clean.

## Scope

### In scope for v1

- Vault initialization (`init`)
- Adding receipts from an image file (`add`)
- Listing and viewing receipts (`list`, `show`)
- Marking receipts reimbursed, full or partial (`reimburse`)
- Headline unreimbursed-balance calculation (`balance`)
- Batch import from a folder (`import`)
- Vault validation (`validate`) — schema, orphans, duplicates
- Reports and CSV export (`report`, `export`)
- Documented on-disk format (`FORMAT.md`)

### Out of scope for v1

- Donations (planned follow-on, not in v1)
- OCR
- Cloud sync built into the tool
- Integration with any specific cloud provider
- Mobile app, web UI, or TUI
- Config files (CLI flags + one env var only)
- Interfaces or abstractions introduced speculatively

## Commands

```
receipts init <path>
receipts add <image-path> --date <YYYY-MM-DD> --provider <name> --amount <n> [--notes <text>]
receipts list [--year <n>] [--unreimbursed] [--provider <name>]
receipts show <id>
receipts reimburse <id> --amount <n> --date <YYYY-MM-DD> [--notes <text>]
receipts balance
receipts import              # batch-add receipts from a folder
receipts validate            # schema, orphans, duplicates
receipts report --year <n> [--format text|json|csv]
receipts export --format csv > out.csv
```

Vault location via `RECEIPTS_VAULT` env var with `--vault` flag override.

Currency defaults to USD in v1; the format records it explicitly per receipt and per reimbursement (see `FORMAT.md`), but the CLI does not expose a `--currency` flag yet.

## Build path

Rough ordering. Ship and tag at every milestone.

1. **`FORMAT.md` first.** Write the spec before writing code. Resolve ambiguities in prose. Tag v0.0.1 as spec-only.
2. **Vertical slice: `init` + `add`.** Smallest thing that exercises the full format end-to-end. Confirm the resulting folder matches `FORMAT.md`. Fix whichever is wrong. Tag v0.1.0.
3. **Read path: `list` + `show`.** Close the round-trip loop. Dogfood on real receipts. Fix what hurts. Tag v0.2.0.
4. **Domain logic: `reimburse` + `balance`.** First point where the tool is useful for its actual purpose. Tag v0.3.0.
5. **Inbox flow: `import`.** Tag v0.4.0.
6. **Polish: `validate` + `report` + `export` + tests + CI + release binaries.** Tag v1.0.0.

## Tech choices

- **Language:** Go.
- **CLI framework:** `cobra`.
- **JSON:** stdlib `encoding/json` with struct tags. Custom marshaler for date-only values.
- **Filesystem:** stdlib `filepath.WalkDir`, atomic writes via temp file + rename.
- **Errors:** stdlib `errors` with wrapping.
- **Testing:** stdlib `testing`, integration tests using `t.TempDir()`.
- **CI/release:** GitHub Actions + GoReleaser for cross-platform binaries.

No caching layer in v1. Walk the vault on every command; fast enough for realistic volumes. A cache can be added later if needed.

## Things to resist

- Adding a config file.
- Introducing interfaces before there's a second implementation.
- A TUI, web UI, or phone app.
- Building OCR before the non-OCR version is boring to use.
- Implementing donations before HSA is solid.