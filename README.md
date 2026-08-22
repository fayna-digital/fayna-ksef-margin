# l10n_pl_ksef_margin — Odoo 17 × KSeF 2.0 (FA(3))

![Odoo Version](https://img.shields.io/badge/Odoo-17.0%20Community-purple)
![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![KSeF](https://img.shields.io/badge/KSeF-2.0%20FA(3)-red)
![License](https://img.shields.io/badge/License-LGPL--3-green.svg)
![Status](https://img.shields.io/badge/status-production-brightgreen)

**Opracowane przez [Fayna Digital](https://www.fayna.agency) dla [CampScout](https://campscout.eu)**
**Autor: Volodymyr Shevchenko**

---

## Problem

Od 1 lutego 2026 polski **Krajowy System e-Faktur (KSeF)** staje się obowiązkowy
dla podatników VAT — wszystkie faktury B2C/B2B muszą być wysyłane przez państwowe
API w standaryzowanym formacie XML **FA(3)**, z szyfrowaniem (RSA-OAEP +
AES-256-CBC), oficjalnym potwierdzeniem (**UPO**) i śledzeniem statusu. Odoo
Community z pudełka tego nie umie — a dla biznesu na **VAT Marża** (turystyczny
schemat marży, art. 119 ustawy o VAT — faktyczna podstawa opodatkowania nie
równa się kwocie faktury) standardowe polskie moduły lokalizacyjne w ogóle nie
liczą poprawnego XML.

## Rozwiązanie

`l10n_pl_ksef_margin` — moduł Odoo 17, który integruje księgowość bezpośrednio z
KSeF 2.0:

- **Auto-wysyłka** — cron co 30 min podchwytuje potwierdzone faktury i wysyła
  partiami po 20; nic ręcznie.
- **Generacja XML FA(3)** z poprawnym `P_PMarzy` / `P_PMarzy_3_3` dla wierszy
  Marża (rozpoznaje słowa kluczowe po łacinie **i cyrylicy** — `marż` /
  `маржа` / `маржі`) oraz zwykłym `P_13_1..P_13_10` dla reszty.
- **Sprawdzanie statusu** — cron co 15 min: `waiting` → `accepted` (+ Nr KSeF)
  lub `rejected` z **realną przyczyną** (XPath + kod z `get_session_invoices`,
  nie tylko generic-fallback).
- **Duplikat kod 440** — ponowna wysyłka już przyjętej faktury nie spada już w
  `rejected`: parsowany jest oryginalny Nr KSeF i status pozostaje `accepted`.
- **Faktury korygujące (KOR)** — faktury korygujące z poprawnym
  `DaneFaKorygowanej` / `NrKSeFFaKorygowanej`.
- **Bufor przychodzących** — synchronizacja faktur dostawców z KSeF do
  izolowanego bufora, bez bezpośredniego wpływu na księgowość do ręcznego
  opracowania.
- **UPO** — Urzędowe Potwierdzenie Odbioru pobierane automatycznie dla
  przyjętych faktur i zapisywane jako attachment.
- **KSeF Dashboard** — komponent Owl: przyjęto / oczekuje / odrzucono / nie
  wysłano, z wykresem aktywności.
- **Bulk wizard** — ręczna masowa wysyłka dowolnego zestawu faktur.

Każde wywołanie zewnętrznego API KSeF jest izolowane — błąd pojedynczej faktury
nie blokuje reszty partii, faktura pozostaje `draft`/`waiting` i zostanie
podchwycona w następnym cyklu crona.

## Rezultat

- **57 testów** — generacja XML (`test_xml_builder.py`) + parsowanie UPO/statusów/
  VAT Marża (`test_xml_parser.py`), pokrywając regression-case'y realnych
  KSeF-rejectów (kod 450 — wiszący `<P_13_1>` bez `<P_14_1>`; cyryliczna
  «Маржа»; duplikaty kod 440).
- Użycie produkcyjne: tysiące faktur VAT Marża przeszło walidację XSD FA(3)
  i zostało przyjętych przez KSeF bez ręcznej ingerencji.
- Cyryliczne nazwy towarów logowane `WARNING` zamiast cichego reject —
  operator widzi problem zanim KSeF go zwróci.

## Stack

Odoo 17.0 Community · `ksef-client` (KSeF 2.0 SDK, RSA-OAEP + AES-256-CBC) ·
schema FA(3) `crd.gov.pl/wzor/2025/06/25/13775` · lxml/ElementTree · Owl
(dashboard UI) · pytest (Odoo `BaseCase`).

## Architektura

```
l10n_pl_ksef_margin/
├── models/
│   ├── account_move.py          # pola KSeF, akcje użytkownika, auto-send cron, sprawdzanie statusu
│   ├── ksef_bulk_send_wizard.py # ręczna masowa wysyłka
│   ├── ksef_vendor_buffer.py    # bufor przychodzących faktur od dostawców
│   └── res_company.py           # token KSeF + środowisko (test/prod)
├── services/
│   ├── ksef_auth.py             # autoryzacja (challenge → token → session, RSA-OAEP)
│   ├── ksef_xml_builder.py      # generacja XML FA(3) — główna logika biznesowa (VAT Marża, KOR)
│   └── ksef_xml_parser.py       # parsowanie UPO, statusów, detekcja VAT Marża
├── views/                       # account_move, res_company, vendor_buffer, dashboard, bulk_send_wizard
├── data/ksef_cron.xml           # 3 crony: auto-send, sprawdzanie statusu, synchronizacja przychodzących
├── static/src/components/       # Owl: KSeF dashboard (strona) + karta alertu
├── security/  i18n/  docs/
└── tests/                       # 57 testów (XML builder + parser)
```

## Flow

```
Faktura potwierdzona (posted)
        ↓
status = 'draft' (Not Sent)
        ↓  ← cron co 30 min, batch 20
status = 'waiting'
        ↓  ← cron co 15 min
    ┌── accepted ──→ Nr KSeF + chatter
    └── rejected  ──→ przyczyna XPath + chatter
            ↓
       Ponów → 'draft' → cron podchwyci
            ↓
       lub kod 440 → automatycznie 'accepted'
```

---

## Instalacja

```bash
pip install ksef-client

cd /path/to/odoo/extra-addons
git clone https://github.com/fayna-digital/fayna-ksef-margin.git

odoo -c /etc/odoo/odoo.conf -d your_db -i l10n_pl_ksef_margin --stop-after-init
```

`odoo.conf`:
```ini
addons_path = ...,/path/to/odoo/extra-addons/l10n-pl-ksef-margin
```

Moduł podłączany jako **extra-addons** (nie custom-addons), bo to plugin
lokalizacyjny. Repo ma zagnieżdżony katalog `l10n_pl_ksef_margin/` —
Odoo podchwyci go automatycznie, jeśli `addons_path` wskazuje na repozytorium.

## Konfiguracja

**Ustawienia → Firmy → [firma] → zakładka KSeF:**

| Pole | |
|------|-|
| Token API KSeF | Token z ksef.mf.gov.pl (Tokeny → Generuj token → InvoiceWrite) |
| Środowisko | `Test` (`ksef-test.mf.gov.pl`) lub `Production` (`ksef.mf.gov.pl`) |

## Użycie

**Automatycznie (zalecane):** po potwierdzeniu faktury — nie trzeba nic robić,
cron co 30 min wyśle do KSeF.

**Ręcznie:** zakładka **KSeF** na fakturze → **Wyślij do KSeF**.

**Bulk:** lista faktur → zaznacz → **Akcja → Masowa wysyłka do KSeF**.

**Faktura odrzucona:** zakładka KSeF pokazuje realną przyczynę (kod + XPath).
Popraw dane → **Ponów wysyłkę** → cron podchwyci.

## VAT Marża

Moduł wykrywa wiersze Marża po słowach kluczowych w nazwie podatku:
- Łacina: `marż`, `marza`, `margin`
- Cyrylica: `маржа`, `маржі`, `маржу`

**Nazwy produktów w KSeF (`P_7`) muszą być po polsku.** Jeśli produkt nie ma
tłumaczenia `pl_PL` — w logach pojawi się `WARNING: P_7 zawiera cyrylicę`.

## Testy

```bash
pre-commit run --all-files    # ruff + ruff-format + mypy + gitleaks + no-ai-signature

# Pełny przebieg Odoo BaseCase — wymaga żywego Odoo; lokalnie bez niego
# testy automatycznie się skipują (patrz tests/conftest.py). W CI są
# uruchamiane realnie przez docker odoo:17.0 + postgres (.github/workflows/ci.yml).
pytest l10n_pl_ksef_margin/tests/ -v
```

## Dokumentacja

- [docs/TZ.md](l10n_pl_ksef_margin/docs/TZ.md) — specyfikacja techniczna (6 obszarów wg REPO_STANDARD)
- [docs/PLAN.md](l10n_pl_ksef_margin/docs/PLAN.md) — dependency graph + fazy + checkpointy
- [CHANGELOG.md](CHANGELOG.md) — historia zmian

---

## Licencja

LGPL-3 — patrz [LICENSE](LICENSE). © Fayna Digital.

---

*Opracowane przez [Fayna Digital](https://www.fayna.agency) · Volodymyr Shevchenko*
