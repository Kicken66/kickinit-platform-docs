# Migration 0004 — SSO-exchange smoke test (Tipspromenad)

## Sammanfattning
Verifiering av Flöde B (`/sso-exchange`) från Tipspromenad mot Hub.

## Utfall
- Edge-funktion `sso-initiate` deployad i Tipspromenad.
- HMAC-SHA256-signering i v1.0-format (`b64u(payload).b64u(sig)`) validerar korrekt mot
  Hubs `/sso-exchange`.
- Hub svarar `404 code_not_found` vid smoke-anrop — förväntat eftersom koden är syntetisk.
- Nonce-replay-skydd och felenvelopp (`{ error: { code, message } }`) bekräftade.

## Kontrakt
- `contracts/v1/sso-exchange.md` v1.0.0 (i docs-repot).

## Beroenden
- `HUB_EXCHANGE_SECRET` delad mellan Hub och Tipspromenad.

## Sign-off
- ✅ Tipspromenad-sida verifierad 2026-06-08.
- ⏳ Hub-sida (full `/sso-exchange`-implementation med riktig kod-utgivare) separat spår.
