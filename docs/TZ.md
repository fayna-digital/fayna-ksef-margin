# docs/TZ.md — fayna-ksef-margin

Specification per Fayna REPO_STANDARD (6 areas).

## 1. Objective

`l10n_pl_ksef_margin` is an Odoo 17 module that integrates accounting directly
with the Polish **Krajowy System e-Faktur (KSeF 2.0)**. From 1 Feb 2026 KSeF is
mandatory for VAT taxpayers — all B2C/B2B invoices must be sent through the
state API in the standardized **FA(3)** XML format, with encryption
(RSA-OAEP + AES-256-CBC), official confirmation (**UPO**) and status tracking.
Odoo Community does not support this out of the box, and standard Polish
localization modules do not compute correct XML for the **VAT Marża** scheme
(art. 119 VAT Act — the taxable base differs from the invoice amount).

**Success =** confirmed invoices are auto-sent in batches; FA(3) XML is
generated with correct `P_PMarzy` / `P_PMarzy_3_3` for Marża rows (recognizing
Latin **and Cyrillic** keywords — `marż` / `маржа` / `маржі`) and plain
`P_13_1..P_13_10` for the rest; statuses are tracked (`waiting` → `accepted` /
`rejected` with the real reason); duplicate code 440 is handled; correcting
invoices (KOR) carry correct `DaneFaKorygowanej` / `NrKSeFFaKorygowanej`;
incoming supplier invoices are synced to an isolated buffer; UPO is downloaded
automatically; a KSeF Dashboard (Owl) and a bulk wizard are provided.

## 2. Commands

```bash
# Install the module into an Odoo 17 addons path (custom-addons)
# and update the module list, then install l10n_pl_ksef_margin.

# Run the test suite (Odoo BaseCase, no live KSeF API)
pytest -q l10n_pl_ksef_margin/tests

# Lint / types / coverage (dev tooling in pyproject.toml)
ruff check .
mypy --ignore-missing-imports l10n_pl_ksef_margin
coverage run -m pytest -q && coverage report
```

## 3. Project Structure

```
fayna-ksef-margin/
├── LICENSE              # LGPL-3
├── CHANGELOG.md         # version history
├── README.md            # PL — product overview
├── pyproject.toml       # ruff / pytest / mypy / coverage config
├── docker-compose.yml   # example Odoo service
├── docs/                # TZ.md, PLAN.md
└── l10n_pl_ksef_margin/ # Odoo module
    ├── __manifest__.py
    ├── data/            # cron jobs, sequences
    ├── i18n/            # translations
    ├── models/          # account.move, ksef session, buffer
    ├── security/        # ir.model.access
    ├── services/        # ksef-client wrapper, XML builder/parser
    ├── static/          # Owl dashboard assets
    ├── tests/           # test_xml_builder.py, test_xml_parser.py, test_environment.py
    └── views/           # dashboard, wizard, menu
```

## 4. Non-Functional Requirements

- **Isolation:** every external KSeF API call is isolated — a single invoice
  error does not block the rest of the batch; the invoice stays `draft`/`waiting`
  and is picked up in the next cron cycle.
- **Reliability:** auto-send cron every 30 min (batches of 20); status cron
  every 15 min.
- **Observability:** Cyrillic product names are logged `WARNING` instead of a
  silent reject — the operator sees the problem before KSeF returns it.
- **Security:** credentials stored via Odoo `ir.config_parameter`; no secrets
  in the repo; RSA-OAEP + AES-256-CBC handled by the `ksef-client` SDK.
- **Compatibility:** Odoo 17.0 Community, Python 3.10+.

## 5. Acceptance Criteria

- [x] FA(3) XML generation with correct Marża rows (Latin + Cyrillic keywords).
- [x] Auto-send of confirmed invoices in batches of 20.
- [x] Status tracking with real rejection reason (XPath + code from
      `get_session_invoices`, not generic fallback).
- [x] Duplicate code 440 handled — original Nr KSeF parsed, status stays `accepted`.
- [x] Correcting invoices (KOR) with `DaneFaKorygowanej` / `NrKSeFFaKorygowanej`.
- [x] Incoming supplier invoice buffer (isolated from accounting).
- [x] UPO auto-download as attachment.
- [x] KSeF Dashboard (Owl) + bulk wizard.
- [x] 57 tests covering XML generation, UPO/status/VAT Marża parsing, and
      regression cases of real KSeF rejects (code 450, Cyrillic «Маржа», code 440).

## 6. Out of Scope

- Live KSeF API calls in CI (requires production credentials).
- Non-Polish e-invoicing schemes.
- Odoo Enterprise-specific features.
