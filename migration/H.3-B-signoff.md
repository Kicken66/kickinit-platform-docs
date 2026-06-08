# H.3-B — Signoff: Domän/redirect-verifiering

**Status:** ✅ PASS  
**Datum:** 2026-06-08  
**Signerare:** Lovable-agent (automated verification + drift-fix)  

## Referenser

| Artefakt | Commit / Plats |
|---|---|
| Verification-rapport | `docs/migration/H.3-B-domain-redirect-verification.md` (lokal) |
| Drift-fix + inline-kommentar | `a4cfe6b` — `supabase/functions/provision-private-org/index.ts` |
| CI-regression-guard | `a4cfe6b` — `redirectTo`-assert i `provision-private-org/index_e3_test.ts` + `supabaseMock.ts`-utökning |

## Sammanfattning

Alla A2- (custom domain) och A3-krav (magic-link redirect-separation) från `H.3-cutover-prep.md` är intakta i produktion efter en identifierad och åtgärdad drift.

### Utfall per kontrollpunkt

| Kontrollpunkt | Utfall | Notering |
|---|---|---|
| Domänupplösning + SSL | ✅ PASS | `tipstjanst.online`, `www.tipstjanst.online`, `app.kickinit.se` — alla 200, giltigt cert (LE, auto-renew) |
| SPA-fallback `/auth/callback` | ✅ PASS | 200 på båda domänerna |
| Deep links | ✅ PASS | `/quiz/<round>` verifierad |
| `provision-private-org` redirect | ✅ PASS (efter fix) | Regredierad till `app.kickinit.se`; återställd till `tipstjanst.online/auth/callback` |
| `sso-from-launchpad` redirect | ✅ PASS | `app.kickinit.se/auth/callback` oförändrad |
| CI-regression-guard | ✅ PASS | `redirectTo`-assert i `index_e3_test.ts` — 11/11 tester gröna |

### Identifierad drift (åtgärdad)

`provision-private-org/index.ts:290` hade regredierat från A3-kontraktets `https://tipstjanst.online/auth/callback` till `https://app.kickinit.se/auth/callback`. Konsekvens om kvarlämnad: nya privatkunder från Launchpad hade landat på admin-portalen istället för publik tipspromenad. Fixad i samma turn som upptäckten; inline-kommentar tillagd för att förhindra tyst regression.

## Uppföljning

- SSL-cert `tipstjanst.online` förfaller `2026-08-30`. Lovable auto-förnyar; manuell verifiering `2026-08-25` som försäkring.
- `redirectTo`-assert i CI fångar samma klass av drift automatiskt vid nästa deploy.

## Godkännande

H.3-B anses komplett. Nästa steg: H.3-C (live cutover-smoke) enligt `H.3-cutover-checklist.md`.
