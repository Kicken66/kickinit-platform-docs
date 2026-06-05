# Contract: sso-from-launchpad

**Version: 1.0.0** · **Status: FROZEN**
**Direction:** Launchpad → Tipspromenad
**Endpoint:** `POST {TIPSPROMENAD_FUNCTIONS_URL}/sso-from-launchpad`

Genererar en kortlivad magic-link så att en kund som är inloggad i Launchpad
kan klicka "Öppna admin" och landa inloggad i tipspromenad-appen utan att
ange lösenord. Ingen delad session, ingen JWT-bridging — bara en signerad,
kortlivad token.

## Auth

Bearer-token i body, signerad med shared secret.

## Request

```json
{
  "token": "<jwt-eller-hmac-token>",
  "email": "kalle@example.com"
}
```

### Token-format (HMAC-SHA256, base64url)

Payload som tecknas:

```
{
  "email": "kalle@example.com",
  "iat": 1716900000,
  "exp": 1716900060,
  "nonce": "random-uuid"
}
```

Token = `base64url(payload) + "." + base64url(hmac_sha256(secret, base64url(payload)))`.

- Secret: `LAUNCHPAD_SSO_SHARED_SECRET` (i båda projekten).
- TTL: ≤ 60s (`exp - iat`).
- `nonce` lagras i tipspromenad-DB 5 min för replay-skydd.

## Response 200

```json
{
  "status": "ok",
  "magic_link": "https://app.kickinit.se/auth/callback?token=...",
  "expires_at": "2026-05-28T13:00:00Z"
}
```

Launchpads klient gör `302` redirect till `magic_link`.

## Error responses

| Status | `code` | Betydelse |
|---|---|---|
| 400 | `invalid_token` | Felaktigt format / signaturfel. |
| 401 | `token_expired` | `exp` har passerat. |
| 401 | `replay_detected` | `nonce` redan använt. |
| 404 | `unknown_user` | E-post finns inte som auth-user. |
| 500 | `internal_error` | |

## Säkerhet

- Ingen användarinmatning passerar genom token (utöver email).
- Token har TTL ≤ 60s — kund måste klicka snabbt eller token genereras om.
- Magic link har egen TTL 1h (Supabase-default).
- `email`-fältet i body används bara för debug-logg; den auktoritativa identiteten kommer från `token.email`.
