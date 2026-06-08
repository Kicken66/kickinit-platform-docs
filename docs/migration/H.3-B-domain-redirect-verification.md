# H.3-B — Domän/redirect-verifiering (live smoke)

**Status:** ✅ PASS (efter 1 drift-fix)
**Datum:** 2026-06-08
**Scope:** Verifiera att A2 (custom domain) och A3 (magic-link redirect-separation)
från `H.3-cutover-prep.md` fortfarande är intakta i produktion.

## 1. Domän + SSL

| URL | HTTP | Remote IP | Notering |
|-----|------|-----------|----------|
| `https://tipstjanst.online/` | 200 | 185.158.133.1 (Lovable edge) | ✅ |
| `https://www.tipstjanst.online/` | 200 | 185.158.133.1 | ✅ alias |
| `https://app.kickinit.se/` | 200 | 2a07:8240::1 | ✅ admin |
| `https://tipstjanst.online/auth/callback` | 200 | — | ✅ SPA-fallback |
| `https://app.kickinit.se/auth/callback` | 200 | — | ✅ SPA-fallback |
| `https://tipstjanst.online/quiz/<sanity-round>` | 200 | — | ✅ deep link |

### SSL-cert `tipstjanst.online`
- Subject: `CN=tipstjanst.online`
- Issuer: `Google Trust Services / WE1`
- `notBefore=2026-06-01`, `notAfter=2026-08-30` (90-dagars LE-rotation aktiv).
- Lovable hanterar förnyelse automatiskt — ingen åtgärd.

## 2. Magic-link redirect-separation

Kontrollerades mot `supabase/functions/<fn>/index.ts`:

| Edge-fn | Förväntad redirect | Faktisk redirect (före fix) | Resultat |
|---------|---------------------|------------------------------|----------|
| `provision-private-org` | `https://tipstjanst.online/auth/callback` | ❌ `https://app.kickinit.se/auth/callback` (rad 290) | **DRIFT — fixad** |
| `sso-from-launchpad` | `https://app.kickinit.se/auth/callback` | ✅ `https://app.kickinit.se/auth/callback` (rad 92) | OK |

### Drift-finding (åtgärdad i samma turn)
`provision-private-org/index.ts:290` hade regredierat till `app.kickinit.se`-callbacken
någon gång efter A3-signoff (2026-06-01). Konsekvens om den varit kvar: nya
privatkunder från Launchpad hade landat på admin-portalen istället för publik
tipspromenad — magic-link funkar men UX bryter mot kontraktet i `H.3-cutover-prep.md` A3.

**Fix:** En-rad-ändring tillbaka till `https://tipstjanst.online/auth/callback` +
inline-kommentar som pekar på A3 så regressen inte sker tyst igen.

## 3. Inga ytterligare avvikelser
- `app.kickinit.se` + `tipstjanst.online` körs parallellt (inga 301-redirects mellan dem) — matchar beslutet i `H.3-blockers-resolved.md` rad 64.
- Lovable SPA-fallback levererar `/auth/callback` korrekt på båda domänerna (200, inte 404).
- `VITE_REDIRECT_TIPSTJANST` är inte längre aktiv i `App.tsx` (default `false` enligt H.3-blockers-resolved rad 30) — bekräftat via `rg`.

## 4. Uppföljning
- Lägg in `redirectTo`-asserts i `provision-private-org/index_test.ts` så samma drift fångas i CI nästa gång. (Backlog — inte blocker för H.3-B.)
- SSL-cert förfaller `2026-08-30`. Lovable auto-förnyar; sätt kalender-påminnelse `2026-08-25` för manuell verifiering om automation skulle fallera.
