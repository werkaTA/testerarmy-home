# auth.md — agent registration

You are an agent. This service allows you to register for an identity and exchange it for credentials.

Run these commands in order. **Don't print secret values and don't write your own scripts** — the commands below persist secrets and surface only non-secret fields. Commands are macOS (`curl`, `security`, `jq`); adapt as needed for Linux (`secret-tool`, reads stdin) or Windows (PowerShell `Set-Secret`/`Get-Secret`). Only `security` takes the secret on argv — `secret-tool`/`Set-Secret` don't, so don't carry its "rotate if exposed" caveat to them.

Set once:

```bash
ISSUER="https://auth.tester.army"
SVC="${ISSUER#https://}"        # keychain service label (issuer host)
EMAIL="<user-email>"           # login_hint, and keychain account for email-based methods
```

## 1. Check for an existing registration — reuse before creating a new one

```bash
if   security find-generic-password -s "$SVC" -a "$EMAIL"  -w >/dev/null 2>&1; then ACCT="$EMAIL"
elif security find-generic-password -s "$SVC" -a anonymous -w >/dev/null 2>&1; then ACCT="anonymous"
fi
# ACCT set -> skip to step 4 (Exchange). ACCT unset -> continue to step 2.
```

## 2. Register — pick the method that fits; capture into `$REG`, never print it

Prefer the lightest method that does the job — don't reach for the user's email by default. A new user can start **anonymous** to trial the service: it grants its pre-claim scopes immediately, with no email and no human step. If that covers what you're doing, stay anonymous. Step up to **service_auth** (and its claim ceremony) only when you need broader access (the post-claim scopes) or the user already has an account and wants to sign in.

```bash
# service_auth — you have the user's email (claim ceremony required):
REG="$(curl -sS "$ISSUER/agent/identity" -H 'Content-Type: application/json' \
  -d "{\"type\":\"service_auth\",\"login_hint\":\"$EMAIL\"}")"
ACCT="$EMAIL"
# errors: invalid_request -> fix the body; invalid_login_hint -> fix EMAIL; service_auth_registration_disabled -> method not enabled for this environment.

# anonymous — no identity / defer it (claim ceremony optional):
REG="$(curl -sS "$ISSUER/agent/identity" -H 'Content-Type: application/json' \
  -d '{"type":"anonymous"}')"
ACCT="anonymous"
# errors: invalid_request -> fix the body; anonymous_registration_disabled -> method not enabled for this environment.
```

Persist the whole response now for **anonymous** (carries `identity.assertion`); for **service_auth** keep `$REG` in memory and persist after the claim verifies (step 3):

```bash
security add-generic-password -U -s "$SVC" -a "$ACCT" -w "$REG"   # not after a service_auth registration
```

## 3. Claim ceremony — service_auth (required), anonymous (optional)

Mint an attempt and give the user its link. They sign in there and the page shows them a code; once they read it back to you, complete the claim with it.

```bash
CLAIM_TOKEN="$(printf '%s' "$REG" | jq -r .claim.token)"
ATT="$(curl -sS "$ISSUER/agent/identity/claim" -H 'Content-Type: application/json' \
  -d "{\"type\":\"service_auth\",\"claim_token\":\"$CLAIM_TOKEN\",\"login_hint\":\"$EMAIL\"}")"
# errors: invalid_claim_token -> restart at step 2; invalid_login_hint -> fix EMAIL; claim_expired | claim_revoked | already_claimed -> restart at step 2; auth_method_disabled -> method was disabled; too_many_attempts -> wait for a pending attempt to expire.

# give the user this link (non-secret); the code is shown to them on the page, not here:
printf '%s' "$ATT" | jq '{verification_uri:.attempt.verification_uri}'

# the user reads the code off that page and gives it to you; submit it with the claim token:
USER_CODE="<code the user read off the claim page>"
VER="$(curl -sS "$ISSUER/agent/identity/claim/complete" -H 'Content-Type: application/json' \
  -d "{\"claim_token\":\"$CLAIM_TOKEN\",\"user_code\":\"$USER_CODE\"}")"

# success returns the verified identity once — persist it; an error returns {code,message}:
printf '%s' "$VER" | jq -e .identity.assertion >/dev/null && security add-generic-password -U -s "$SVC" -a "$ACCT" -w "$VER"
# errors: claim_not_confirmed -> user hasn't finished on the page yet, wait and retry; invalid_user_code -> wrong code, ask the user again; user_code_expired -> re-run this step for a fresh link; claim_expired | already_claimed -> restart at step 2; claim_denied -> user denied the claim, restart at step 2; auth_method_disabled -> method was disabled; organization_selection_required | stale_organization_selection -> user must re-select an organization on the claim page.
```

## 4. Exchange the assertion for an access token

```bash
ASSERTION="$(security find-generic-password -s "$SVC" -a "$ACCT" -w | jq -r .identity.assertion)"
RESOURCE="<resource-uri>"   # optional — binds the token's aud to a resource this service recognizes; omit both resource lines for the default audience
CRED="$(curl -sS "$ISSUER/oauth2/token" -H 'Content-Type: application/x-www-form-urlencoded' \
  -d grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer \
  --data-urlencode "assertion=$ASSERTION" \
  --data-urlencode "resource=$RESOURCE")"
ACCESS_TOKEN="$(printf '%s' "$CRED" | jq -r .access_token)"
# errors: invalid_request -> assertion could not be decoded; invalid_grant -> assertion expired/revoked, restart at step 2; invalid_target -> resource URI not recognized; unsupported_grant_type -> use the grant above.
```

Send it as `Authorization: Bearer $ACCESS_TOKEN`. **Don't persist the access token** — it's short-lived (~5 minutes). When it expires, re-run this step from the stored assertion; when the assertion itself expires, refresh (step 5).

## 5. Refresh — when the stored assertion nears expiry

```bash
RT="$(security find-generic-password -s "$SVC" -a "$ACCT" -w | jq -r .identity.refresh_token.value)"
REG="$(curl -sS "$ISSUER/agent/identity" -H 'Content-Type: application/json' \
  -d "{\"type\":\"refresh\",\"refresh_token\":\"$RT\"}")"
security add-generic-password -U -s "$SVC" -a "$ACCT" -w "$REG"   # rotates the refresh token; overwrite
# errors: invalid_refresh_token -> restart at step 2.
```

Then re-run step 4 with the fresh assertion.

## Any request

`5xx` -> back off and retry the same request. `rate_limit_exceeded` -> wait `retry_after` seconds. A `4xx` not listed above -> fix per the response body; don't replay.
