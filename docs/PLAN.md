# docs/PLAN.md — fayna-ksef-margin

## Phase 0 — Extract & sanitize ✅

Extracted the KSeF 2.0 module from the internal project, de-identified
(production infra, deploy scripts and client-specific data excluded), brought
to REPO_STANDARD, fresh git history without AI signatures.

## Phase 1 — Senior-grade pass ✅

- Package structure (`l10n_pl_ksef_margin` as an Odoo addon).
- Full type coverage of public functions, specific `except`, `logging` in
  library code.
- `pytest` test suite (57 tests) covering XML generation and parsing without a
  live KSeF API; CI (ruff + gitleaks + mypy + pytest).

## Phase 2 — Production hardening ✅

- Auto-send cron (30 min, batches of 20) and status cron (15 min).
- Duplicate code 440 handling; correcting invoices (KOR).
- Incoming supplier invoice buffer; UPO auto-download.
- KSeF Dashboard (Owl) + bulk wizard.
- Production use: thousands of VAT Marża invoices passed FA(3) XSD validation
  and were accepted by KSeF without manual intervention.

## Phase 3 — Documentation & i18n

- [x] README (PL) — product overview.
- [x] docs/TZ.md (EN) — specification per REPO_STANDARD.
- [x] docs/PLAN.md (EN) — this plan.
- [x] i18n: module strings are Polish-first (source); full `pl_PL` + `uk_UA`
      translations shipped in `i18n/`.

## Checkpoints

- End of Phase 1 → ready for public portfolio showcase.
- End of Phase 2 → production-proven, CI green.
- End of Phase 3 → documentation complete and language-standard compliant.
