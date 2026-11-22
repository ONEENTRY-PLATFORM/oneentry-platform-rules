# Admin client certificate mode

Some instances require a client certificate on top of the password for the Admin API. In that mode every call, every sign-in and every token renewal must arrive over TLS with the client certificate issued to the same administrator account; a valid certificate of another administrator is refused.

Read this when an Admin API call or a sign-in answers `403 Client certificate required` or `403 Client certificate does not belong to this admin`, or before switching an instance to this mode.

→ `mcp/docs/api/admin-sign-in#sign-in-answers-a-challenge-instead-of-tokens` · `mcp/docs/api/admin-account-ownership#no-operation-sets-another-admin-password`

## How to tell that an instance is in certificate mode

`GET /api/admin/settings-general` reports it read-only as `system.common.adminMtls`: `true` means every Admin API call needs the account's own client certificate. It is `false` unless the instance operator turned the mode on, and no Admin API operation changes it.

This flag is separate from `system.common.mtls`, which concerns public reads only. An instance can have either one without the other.

## Every call answers 403 Client certificate required

The call reached the instance without a verified client certificate. The access token, the login and the password are not looked at yet, so retrying with another token or password changes nothing.

Present the certificate in the TLS handshake to the Admin API host: the `.cert` and `.key` pair, or the `.p12` file from the same archive (it has no password).

A liveness probe of the instance answers without a certificate. Nothing else does.

## Answers 403 when the certificate belongs to another admin

`403 Client certificate does not belong to this admin` means a valid certificate was presented, but it is not the one issued to the account whose token, login, sign-in challenge or refresh token the call carries. The same answer comes back for a login that does not exist, so it never tells whether an account exists.

Switch to the certificate of that account, or sign in as the account the certificate belongs to. A revoked certificate answers the same way for its own account.

## Sign in and renewal with a certificate

`POST /api/admin/auth/login` checks the certificate before the password. A wrong certificate answers `403` whether or not the password is right; with the right certificate a wrong password still answers `401 Login or password incorrect`.

The second step of a two-step sign-in, `POST /api/admin/auth/login/mfa`, checks the certificate after the code. The code is spent even when the certificate is refused, so start the sign-in again with the right certificate.

`POST /api/admin/auth/refresh` and `POST /api/admin/auth/change-password` check it as well.

## Issuing downloading and revoking an admin certificate

Each administrator has at most one active certificate. Three operations act on it, by administrator id:

```http
POST   /api/admin/system/certs/admins/7   → 201 { "id": 512 }
GET    /api/admin/system/certs/admins/7   → 200 zip: .key .cert .csr .p12
DELETE /api/admin/system/certs/admins/7   → 200 true
```

An administrator may manage their own certificate. Another administrator's needs the project owner or `admins.manageCredentials`; the owner's certificate only the owner. Otherwise the answer is `403`.

A second issue while one is active answers `409`; revoke first. Download and revoke with no active certificate answer `404`. Revocation is asynchronous. Once it completes, the account is signed out of every session and can be issued a new certificate.

The archive holds a private key without a password. Hand it only to the administrator it belongs to.

## Switching an instance to certificate mode safely

Issue and install a certificate for every administrator, the owner first, and have each one load the panel with it before the mode is switched on. An administrator without a certificate cannot reach the Admin API at all once the mode is on, including to issue one.

Unattended clients that call the Admin API need an administrator account of their own with its own certificate.

## What this means for this server

This server does not present a client certificate. Against an instance in certificate mode every call it makes answers `403 Client certificate required`, whatever `ONEENTRY_CMS_TOKEN` or login it is configured with.

Give it an administrator account of its own, issue that account a certificate, and put a TLS client in front of the server that presents it to the Admin API host. Without that, keep such an instance out of certificate mode for as long as the server must reach it.

→ `mcp/docs/server/configuration#the-base-url-must-point-at-the-admin-api` · `mcp/docs/server/authentication#two-ways-to-authenticate` · `mcp/docs/api/settings#general-settings`
