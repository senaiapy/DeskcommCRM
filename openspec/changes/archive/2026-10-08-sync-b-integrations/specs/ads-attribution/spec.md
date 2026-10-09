## ADDED Requirements

### Requirement: Google Ads callback is bound to the browser and single-use
`GET /api/v1/plataformas-de-anuncio/google/connect` SHALL put a random nonce and the authenticated session id in the signed `state` and set the httpOnly cookie `crm_oauth_bind` (path of the callback, 10 minutes) signed with `INTERNAL_SECRET`; the callback SHALL redirect to `/app/settings/conversoes?erro=estado_invalido` without exchanging the code when the `state` cannot be verified, the cookie does not match the nonce, `supportCallbackWriteAllowed` refuses, or the nonce cannot be inserted into `calendar_oauth_nonces` (already used), and SHALL clear the cookie on every exit.

#### Scenario: Replayed callback URL
- **WHEN** the same callback URL is opened a second time
- **THEN** the nonce insert conflicts, the browser lands on `/app/settings/conversoes?erro=estado_invalido` and no token is stored

#### Scenario: Callback opened in another browser
- **WHEN** the callback arrives without the `crm_oauth_bind` cookie set at connect
- **THEN** the code is not exchanged and the redirect carries `erro=estado_invalido`
