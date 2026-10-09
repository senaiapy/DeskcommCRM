## ADDED Requirements

### Requirement: The OAuth callback state is single-use
After verifying `state`, `GET /api/v1/integrations/nuvemshop/callback` SHALL insert the state nonce with the organization and user into `calendar_oauth_nonces` before exchanging the code, and when the insert fails or the state carries no user SHALL audit `nuvemshop.oauth_failed` with reason `state_reused` (unique violation) or `nonce_unavailable` and redirect to `/app/integrations/nuvemshop?error=invalid_state`.

#### Scenario: Second use of the same callback
- **WHEN** the browser replays a callback URL that already connected the store
- **THEN** the redirect is `/app/integrations/nuvemshop?error=invalid_state`, the audit reason is `state_reused` and the code is not exchanged again
