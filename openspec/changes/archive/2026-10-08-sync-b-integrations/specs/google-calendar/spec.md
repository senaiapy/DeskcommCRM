## ADDED Requirements

### Requirement: The installation client secret must look like a Google secret
The `updateGoogleOAuth` platform-admin action SHALL reject a non-empty `client_secret` that does not match `FORMATO_DO_CLIENT_SECRET` (`^[A-Za-z0-9_-]+$` after trimming) with an error telling the admin to copy only the value starting with `GOCSPX-`, and the `/admin/google` form SHALL disable saving while the typed secret fails `clientSecretTemFormato`.

#### Scenario: Secret pasted with the rest of the JSON line
- **WHEN** the admin saves `GOCSPX-abc","redirect_uris` as client secret
- **THEN** nothing is stored in `platform_google_oauth` and the error asks for the value only

### Requirement: The privacy policy declares the Google data use
`/legal/privacy` SHALL render the section `id="dados-do-google"` stating which Google Calendar and Google Ads data is read and for what, that it is not sold, used for advertising or used to train AI models, that it follows the Google API Services User Data Policy including Limited Use, and how to revoke access (Agenda › Desconectar and `https://myaccount.google.com/permissions`).

#### Scenario: Google verification review
- **WHEN** a reviewer opens `/legal/privacy#dados-do-google`
- **THEN** the page lists the Calendar and Ads uses, the Limited Use statement and the revocation link
