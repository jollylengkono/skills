# ORDS Concert Sample App Bootstrap Skill

## 1. Overview

This skill helps customers and maintainers configure and validate the ORDS Concert sample app across identity provider setup, ORDS authorization, OCI Database API Gateway / OCS runtime configuration, and cloud metadata migration.

The modernized target architecture is:

`Oracle IAM + ORDS JWT validation + OCI Database API Gateway / OCS + metadataSource:CLOUD`

Auth0 is retained only as a baseline setup path for reproducing original sample behavior, automation compatibility, or migration comparison.

Use this skill to produce reviewable setup plans, commands, configuration guidance, and validation steps without inventing tenant-specific values.

## 2. Supported Modes

### auth0-baseline

Use when the user wants to reproduce, automate, or validate the original Auth0-based setup.

### oci-iam-target

Use when the user wants to configure Oracle IAM / OCI IAM as the target identity provider.

### ords-authorization

Use when the user wants to configure JWT claim, scope for ORDS protected resources.

### ocs-runtime-config

Use when the user wants to configure ORDS using OCI Database API Gateway / OCS and start ORDS using a configuration OCID.

### cloud-metadata-migration

Use when the user wants to migrate ORDS PL/SQL definitions such as `ords.define_module`, `ords.define_template`, `ords.define_handler`, `ords.define_parameter`, or `ords.enable_object` into `ords_config_service/apispec.json` and `ords_config_service/oci_requests.sh`.

### validate-bootstrap

Use after setup to validate identity, JWT claims, ORDS authorization, OCS configuration, generated API specs, and endpoint behavior.

## 3. Skill Behavior Contract

Every response should include:

1. Detected scenario
2. Assumptions
3. Required inputs
4. Files to create or update
5. Commands to run
6. Validation checks
7. Expected results
8. Security warnings
9. Troubleshooting notes
10. Next recommended action

Rules:

- Do not invent missing configuration values.
- Treat existing skill files in this repository as the only source of truth.
- Do not use external data, web searches, connected tools, assumptions, or newly invented content.
- Use placeholders for missing values.
- Ask for missing required values only when they block the next step.
- Prefer reviewable commands over hidden automation.
- Separate explanation from commands.
- Explain what each command changes.
- Always include validation steps.
- Always include negative validation checks when authentication or authorization is involved.

## 4. Required Inputs

### Common inputs

- Local app URL
- ORDS base URL
- Target schema name
- ORDS pool name
- Authorization model: SBAC 
- Environment: local, OCI, or other

### Auth0 baseline inputs

- Auth0 domain
- Auth0 Management API client ID
- Auth0 Management API client secret
- API identifier
- Callback URL
- Logout URL
- Required scopes
- Test user
- Test admin user

### Oracle IAM target inputs

- OCI IAM domain URL
- OIDC client ID
- OIDC client secret
- Authorization endpoint
- Token endpoint
- Userinfo endpoint
- JWKS URL
- JWT issuer
- JWT audience
- Required scopes (SBAC)
- Callback URL
- Logout URL

### ORDS authorization inputs

- Protected endpoint paths
- Required scopes
- JWT profile name
- Privilege names
- Claim mapping strategy: scope-based 

### OCS runtime configuration inputs

- OCI region
- Compartment OCID
- Database API Gateway configuration OCID
- Database Tools connection OCID
- Pool key or pool name
- DBTools endpoint
- OCI CLI profile or authentication method
- ORDS runtime startup command

### Cloud metadata migration inputs

- Source files containing ORDS PL/SQL definitions
- Target `ords_config_service/apispec.json`
- Target `ords_config_service/oci_requests.sh`
- Pool key
- API spec display name
- Auto API Spec object list
- Enabled schema or object names

## 5. Output Format

Use this template:

### Detected Scenario

State the selected mode.

### Assumptions

List assumptions explicitly.

### Required Inputs

| Input | Status | Notes |
| --- | --- | --- |
| `JWT_ISSUER` | provided / missing | Must match token `iss` |
| `JWT_AUDIENCE` | provided / missing | Must match token `aud` |
| `JWKS_URL` | provided / missing | Used by ORDS to validate JWT |
| `CONFIG_OCID` | provided / missing | Used to start ORDS with OCS |

### Files to Create or Update

List only files relevant to the detected scenario.

Examples:

- `.env.example`
- `app/utils/auth.server.ts`
- `app/routes/callback.tsx`
- `app/routes/login.tsx`
- `app/routes/logout.tsx`
- `app/routes/constants/index.server.ts`
- `ords_config_service/apispec.json`
- `ords_config_service/oci_requests.sh`

### Commands

Provide commands in execution order.

### Validation

Include positive checks and negative checks.

Positive checks:

- Login succeeds.
- Access token contains expected issuer.
- Access token contains expected audience.
- Access token contains expected scope.
- Public endpoint works.
- Protected user endpoint works for authenticated user.
- Admin endpoint works for admin user.
- ORDS starts using the OCS configuration OCID.
- Generated API spec is valid JSON.

Negative checks:

- Request without token fails.
- Request with expired token fails.
- Request with wrong audience fails.
- Request with missing scope fails.
- Non-admin user cannot access admin endpoint.
- Wrong OCS config OCID fails clearly.
- Wrong pool key fails clearly.

### Expected Results

Explain what success looks like.

### Troubleshooting

Include likely causes and fixes.

### Security Notes

Warn about:

- Not committing secrets
- Not printing tokens in documentation
- Using least privilege
- Validating issuer, audience, JWKS URL, and scopes 
- Separating sample placeholders from real customer values

### Next Recommended Action

Give one clear next step.

## 6. Auth0 Baseline Setup

This section is the **Auth0 baseline flow**.

Use when you need to reproduce the original setup behavior, run legacy automation, or compare migration changes against Oracle IAM target behavior.

### Required Auth0 values

- `AUTH0_DOMAIN`
- Auth0 Management API app `CLIENT_ID` / `CLIENT_SECRET`
- `API_IDENTIFIER`
- callback URL
- logout URL
- required scopes

### Auth0 API and application setup

#### Auth0 Management API Upsert Rules (Important)

When making setup scripts rerunnable, apply these rules:

- `POST /api/v2/resource-servers` may include `identifier`.
- `PATCH /api/v2/resource-servers/{id}` must **not** include `identifier` (immutable field).
- A `400 invalid_body` with message similar to `Additional properties not allowed: identifier` means your PATCH payload includes immutable fields.

#### 1. Export automation app credentials

The automation app is a **Machine to Machine Application** authorized for Auth0 Management API scopes.

In Auth0:

1. Go to **Applications -> Applications**
2. Click **Create Application**
3. Name it something like `Management_App`
4. Choose **Machine to Machine Applications**
5. Authorize it for the **Auth0 Management API**
6. Grant only the scopes required by the automation

Recommended scopes:

- `create:resource_servers`
- `read:resource_servers`
- `update:resource_servers`
- `create:clients`
- `read:clients`
- `update:clients`
- `create:client_grants`
- `read:client_grants`
- `update:client_grants`
- `delete:client_grants`

After that, note the following values from the application:

```bash
export AUTH0_DOMAIN="your-tenant.us.auth0.com"
export CLIENT_ID="your_management_app_client_id"
export CLIENT_SECRET="your_management_app_client_secret"
```

#### 2. Obtain Management API access token

```bash
ACCESS_TOKEN=$(
  curl -s --request POST \
    --url "https://$AUTH0_DOMAIN/oauth/token" \
    --header "content-type: application/json" \
    --data "{
      \"client_id\": \"$CLIENT_ID\",
      \"client_secret\": \"$CLIENT_SECRET\",
      \"audience\": \"https://$AUTH0_DOMAIN/api/v2/\",
      \"grant_type\": \"client_credentials\"
    }" | jq -r '.access_token'
)
```

#### 3. Create the Auth0 API (resource server)

```bash
API_IDENTIFIER="https://concert.sample.app"
API_NAME="ORDS Concert API"

RESOURCE_SCOPES='[
  {"value":"read:general_user_content","description":"Read all of the general user endpoints"},
  {"value":"concert_app_authuser","description":"Provides access to the user specific endpoints"},
  {"value":"concert_app_admin","description":"Provides access to the concert app admin endpoints"}
]'
```

#Find existing resource server by identifier
```bash
RS_LIST=$(
  curl -s --request GET \
    --url "https://$AUTH0_DOMAIN/api/v2/resource-servers?per_page=100" \
    --header "authorization: Bearer $ACCESS_TOKEN"
)

RS_ID=$(printf '%s' "$RS_LIST" | jq -r --arg idf "$API_IDENTIFIER" '.[] | select(.identifier==$idf) | .id' | head -n1)
```

#Create payload (identifier allowed)
```bash
CREATE_PAYLOAD=$(
  jq -cn --arg name "$API_NAME" --arg identifier "$API_IDENTIFIER" --argjson scopes "$RESOURCE_SCOPES" '{
    name: $name,
    identifier: $identifier,
    signing_alg: "RS256",
    scopes: $scopes,
    subject_type_authorization: {
      user: { policy: "require_client_grant" },
      client: { policy: "deny_all" }
    }
  }'
)
```

#Update payload (identifier NOT allowed)
```bash
UPDATE_PAYLOAD=$(
  jq -cn --arg name "$API_NAME" --argjson scopes "$RESOURCE_SCOPES" '{
    name: $name,
    signing_alg: "RS256",
    scopes: $scopes,
    subject_type_authorization: {
      user: { policy: "require_client_grant" },
      client: { policy: "deny_all" }
    }
  }'
)

if [ -z "$RS_ID" ]; then
  API_RESPONSE=$(
    curl -s --request POST \
      --url "https://$AUTH0_DOMAIN/api/v2/resource-servers" \
      --header "authorization: Bearer $ACCESS_TOKEN" \
      --header "content-type: application/json" \
      --data "$CREATE_PAYLOAD"
  )
else
  API_RESPONSE=$(
    curl -s --request PATCH \
      --url "https://$AUTH0_DOMAIN/api/v2/resource-servers/$RS_ID" \
      --header "authorization: Bearer $ACCESS_TOKEN" \
      --header "content-type: application/json" \
      --data "$UPDATE_PAYLOAD"
  )
fi

API_ID=$(printf '%s' "$API_RESPONSE" | jq -r '.id')
API_IDENTIFIER=$(printf '%s' "$API_RESPONSE" | jq -r '.identifier')
```

#### 4. Create the sample app client (Regular Web Application)

```bash
APP_RESPONSE=$(
  curl --silent --show-error --fail \
    --request POST \
    --url "https://${AUTH0_DOMAIN}/api/v2/clients" \
    --header "authorization: Bearer ${ACCESS_TOKEN}" \
    --header "content-type: application/json" \
    --data '{
      "name": "ORDS Remix JWT Sample",
      "app_type": "regular_web",
      "grant_types": ["authorization_code"],
      "callbacks": ["http://localhost:3000/callback"],
      "allowed_logout_urls": ["http://localhost:3000"],
      "web_origins": ["http://localhost:3000"]
    }'
)

APP_CLIENT_ID=$(printf '%s' "$APP_RESPONSE" | jq -r '.client_id')
APP_CLIENT_SECRET=$(printf '%s' "$APP_RESPONSE" | jq -r '.client_secret')
```

#### 5. Authorize app to call API (client grant)

```bash
CLIENT_GRANT_RESPONSE=$(
  curl -s --request POST \
    --url "https://$AUTH0_DOMAIN/api/v2/client-grants" \
    --header "authorization: Bearer $ACCESS_TOKEN" \
    --header "content-type: application/json" \
    --data "{
      \"client_id\": \"$APP_CLIENT_ID\",
      \"audience\": \"$API_IDENTIFIER\",
      \"scope\": [
        \"read:general_user_content\",
        \"concert_app_authuser\",
        \"concert_app_admin\"
      ],
      \"subject_type\": \"user\"
    }"
)
```

#### 6. Generate sample app environment values

```bash
cat <<EOF
AUTH0_DOMAIN=$AUTH0_DOMAIN
AUTH0_LOGOUT_URL=https://$AUTH0_DOMAIN/v2/logout
JWT_ISSUER=https://$AUTH0_DOMAIN/
AUTH0_CLIENT_ID=$APP_CLIENT_ID
AUTH0_CLIENT_SECRET=$APP_CLIENT_SECRET
AUTH0_RETURN_TO_URL=http://localhost:3000
AUTH0_CALLBACK_URL=http://localhost:3000/callback
JWT_AUDIENCE=$API_IDENTIFIER
JWT_VERIFICATION_KEY=https://$AUTH0_DOMAIN/.well-known/jwks.json
EOF
```

### Validation commands

```bash
npm run drop
npm run migrate
npm run seed
npm run dev
```

### Common Auth0 errors

- `400 invalid_body` during API update because `identifier` was included in PATCH payload.
- Login callback mismatch because callback URL does not match Auth0 app configuration.
- Missing scopes in issued token because client grant does not include required scopes.

## 7. Oracle IAM Target Setup

This section is the **Oracle IAM target flow**.

Use when migrating to the modernized target architecture.

The Oracle IAM authorization model uses Scope-Based Access Control.

### OCI IAM confidential application setup and endpoints

#### 1. Create or select the Identity Domain

In OCI IAM, create or select the target Identity Domain for the ORDS Concert sample app.

Then create one confidential application:

- `ords-sample-app`

This application is used as both:

- the OAuth/OIDC client for the Remix sample app
- the resource server definition that exposes the ORDS Concert API scopes

#### 2. Configure `ords-sample-app`

##### Service Configuration

Configure the service/resource side of the application.

Use the following values:

| Setting | Value |
| --- | --- |
| Access token expiration | `3600` seconds |
| Primary audience | `ords/sample-app/` |
| Supported scopes | `concert_app_authuser`, `concert_app_admin` |

Define these scopes:

| Scope | Purpose |
| --- | --- |
| `concert_app_authuser` | Allows access to authenticated-user ORDS endpoints |
| `concert_app_admin` | Allows access to admin-only ORDS endpoints |


##### Client Configuration

Configure the client side of the application.

Use the following values:

| Setting | Value |
| --- | --- |
| Client type | `Confidential` |
| Allowed grant type | `Authorization code` |
| Redirect URL | `http://localhost:3000/callback` |
| Optional testing redirect | `https://insomnia.rest` |
| Optional testing redirect | `https://oauth.pstmn.io/v1/callback` |
| Token issuance policy | `Specific` |
| Allowed resource scopes | `concert_app_authuser`, `concert_app_admin` |

Under **Resources**, add the ORDS Concert API resource and allow the following scopes:

- `concert_app_authuser`
- `concert_app_admin`

The Remix server must keep the client secret server-side. Do not expose the client secret in browser code, documentation, screenshots, or committed files.

#### 3. Configure SBAC scope mapping

Use Scope-Based Access Control as the default Oracle IAM authorization model.

The ORDS Concert sample app should map IAM/OIDC scopes to ORDS privileges:

| OCI IAM Scope | ORDS Mapping |
| --- | --- |
| `concert_app_authuser` | ORDS authenticated-user privilege |
| `concert_app_admin` | ORDS admin privilege |


### Environment variables and credential handling

```env
# We refer to some variables as Autonomous Database specific
# but you can use whichever ORDS URL you want/have as well as the user,
# as long as this user is capable of creating and REST Enabling other schemas.
ADB_ORDS_URL=http://localhost:8080/ords/pool_name/
ADB_ADMIN_USER=admin
ADB_ADMIN_PASSWORD=ADMIN_PASSWORD

# The name of the schema that will be created to host all of the
# ORDS Concert App database objects.
SCHEMA_NAME=ORDS_CONCERT_APP
SCHEMA_PASSWORD=YOUR_PASSWORD

# OCI IAM JWT credentials, used by ORDS to validate request to protected endpoints.
JWT_ISSUER=https://identity.oraclecloud.com/
JWT_VERIFICATION_KEY=https://<domain-url>:443/admin/v1/SigningCert/jwk
JWT_AUDIENCE=ords/sample-app/

# OCI IAM OIDC application configuration parameters specific to the sample app.
OIDC_RETURN_TO_URL=http://localhost:3000
OIDC_REDIRECT_URI=http://localhost:3000/callback
OIDC_CLIENT_ID=<CLIENT_ID>
OIDC_CLIENT_SECRET=<CLIENT_SECRET>
OIDC_AUTHORIZATION_ENDPOINT=https://<domain-url>:443/oauth2/v1/authorize
OIDC_TOKEN_ENDPOINT=https://<domain-url>:443/oauth2/v1/token
OIDC_USERINFO_ENDPOINT=https://<domain-url>:443/oauth2/v1/userinfo
OIDC_SCOPES=openid,email,profile,ords/sample-app/concert_app_authuser,ords/sample-app/concert_app_admin
```

> ⚠️ Unverified: validate exact scope names/format in your tenant before production rollout.

### Remix app changes for Oracle IAM target

Apply these updates in the sample app after setting OCI IAM `.env` values.

#### 1. Required packages

```bash
npm install remix-auth@3.7.0 remix-auth-oauth2@1.10.0
```

#### 2. Auth strategy migration (`app/utils/auth.server.ts`)

Use `OAuth2Strategy` and switch app auth flow key to `oidc`.

```ts
import { createFileSessionStorage } from '@remix-run/node';
import { Authenticator } from 'remix-auth';
import { OAuth2Strategy } from 'remix-auth-oauth2';
import {
  COOKIE_MAX_AGE,
  OIDC_AUTHORIZATION_ENDPOINT,
  OIDC_AUDIENCE,
  OIDC_CLIENT_ID,
  OIDC_CLIENT_SECRET,
  OIDC_REDIRECT_URI,
  OIDC_SCOPES,
  OIDC_TOKEN_ENDPOINT,
  OIDC_USERINFO_ENDPOINT,
} from '~/routes/constants/index.server';
import type { OIDCProfile } from '~/models/OIDCProfile';
```

#### 3. Session storage fix (`app/utils/auth.server.ts`)

Use server-side session storage to avoid cookie overflow when token payloads are large.

```ts
const sessionStorage = createFileSessionStorage({
  dir: '/tmp/remix-sessions',
  cookie: {
    name: '_remix_session',
    sameSite: 'lax',
    path: '/',
    httpOnly: true,
    secrets: ['foobar'],
    secure: process.env.NODE_ENV === 'production',
    maxAge: COOKIE_MAX_AGE,
  },
});
```

#### 4. Login route (`app/routes/oidc.tsx`)

```ts
import type { ActionFunctionArgs } from '@remix-run/node';
import { redirect } from '@remix-run/node';
import { auth } from '~/utils/auth.server';

export const loader = async () => redirect('/');
export const action = async ({ request }: ActionFunctionArgs) =>
  auth.authenticate('oidc', request);
```

#### 5. Callback route (for example `app/routes/callback.tsx`)

```ts
export const loader = async ({ request }: LoaderFunctionArgs) =>
  auth.authenticate('oidc', request, {
    successRedirect: '/private/profile',
    failureRedirect: '/error',
  });
```

#### 6. OIDC constants (`app/routes/constants/index.server.ts`)

```ts
export const OIDC_RETURN_TO_URL = process.env.OIDC_RETURN_TO_URL || 'http://localhost:3000';
export const OIDC_REDIRECT_URI = process.env.OIDC_REDIRECT_URI!;
export const OIDC_CLIENT_ID = process.env.OIDC_CLIENT_ID!;
export const OIDC_CLIENT_SECRET = process.env.OIDC_CLIENT_SECRET!;
export const OIDC_AUTHORIZATION_ENDPOINT = process.env.OIDC_AUTHORIZATION_ENDPOINT!;
export const OIDC_TOKEN_ENDPOINT = process.env.OIDC_TOKEN_ENDPOINT!;
export const OIDC_USERINFO_ENDPOINT = process.env.OIDC_USERINFO_ENDPOINT!;
export const OIDC_LOGOUT_ENDPOINT = process.env.OIDC_LOGOUT_ENDPOINT || '';
export const OIDC_AUDIENCE = process.env.OIDC_AUDIENCE || '';
```

#### 7. Provider-neutral profile model (`app/models/OIDCProfile.ts`)

```ts
import type { OAuth2Profile } from 'remix-auth-oauth2';

export interface OIDCProfile extends OAuth2Profile {
  id: string;
  displayName: string;
  emails: Array<{ value: string; type?: string }>;
  photos: Array<{ value: string }>;
  _json?: {
    nickname?: string;
    [key: string]: unknown;
  };
}
```

#### 8. Update login form actions to `/oidc`

- `app/components/navbar/NavBar.tsx`
- `app/components/error/ErrorPage.tsx`
- `app/components/artists/HeroBanner.tsx`
- `app/components/concerts/ConcertBanner.tsx`

```tsx
<Form method="post" action="/oidc">
```

#### 9. Logout route OIDC constants import

```ts
import {
  OIDC_CLIENT_ID,
  OIDC_LOGOUT_ENDPOINT,
  OIDC_RETURN_TO_URL,
} from '~/routes/constants/index.server';
```

### ORDS JWT validation requirements

- `JWT_ISSUER` must match token `iss`.
- `JWT_AUDIENCE` must match token `aud`.
- `JWT_VERIFICATION_KEY` (JWKS URL) must be reachable by ORDS.
- Required scopes must be present in access token for SBAC.

### Validation commands

```bash
npm run drop
npm run migrate
npm run seed
npm run dev
```

### Common IAM/OIDC errors

- Wrong authorization/token endpoint values.
- Wrong `OIDC_CLIENT_ID` or `OIDC_CLIENT_SECRET`.
- Callback URL mismatch with confidential application redirect URI.

## 8. ORDS Authorization Mapping

Use this section for **ORDS authorization** design and validation.

### Scope-based access control (SBAC)

This is the default and recommended model for Oracle IAM target setup in this skill.

Use JWT scopes such as:

- `concert_app_authuser`
- `concert_app_admin`

A user with only `concert_app_authuser` should be unauthorized for admin endpoints.


### Claim mapping strategy guidance

- `scope-based`: default path; use token scopes for privilege checks.


### Example expected JWT payload

```json
{
  "iss": "<JWT_ISSUER>",
  "aud": "<JWT_AUDIENCE>",
  "exp": 1735689600,
  "scope": "concert_app_authuser"
}
```

### Positive and negative tests

- Normal authenticated user: user endpoint allowed; admin endpoint denied.
- Admin user: admin endpoint allowed.
- User without required scope: denied.
- Token with wrong audience: denied.
- Token with wrong issuer: denied.

## 9. OCI Database API Gateway / OCS Runtime Configuration

This is the target ORDS runtime configuration model for the modernized sample app.

Use this section when ORDS should load runtime configuration from OCI Database API Gateway / OCS instead of local static config.

### Purpose

- Centralize ORDS runtime configuration in OCI.
- Map ORDS pools to OCI Database Tools connections.
- Run ORDS using a configuration OCID.
- Support `metadataSource: CLOUD` for cloud-managed API metadata.

### Required OCI values

- OCI region
- compartment OCID
- config OCID
- Database Tools connection OCID
- pool key / pool name
- DBTools endpoint

### 1. Create a compartment

- Use `us-phoenix-1` and the beta endpoint: `https://dbtools.us-phoenix-1.oci.oraclecloud.com`.
- Create a non-production compartment and save its OCID as `<COMPARTMENT_OCID>`.

### 2. Create VCN and subnet rules

- Create a VCN with internet connectivity.
- In the public subnet/security list, add ingress needed for your flow:
  - DB access port, for example TCP `1521`
  - ORDS HTTP test port, for example TCP `8088`

### 3. Create VM and install ORDS

- Launch a compute instance in `us-phoenix-1` in that VCN/subnet.
- Install ORDS and verify Java `17+`.

Example install path:

```bash
sudo yum install ords
```

If your YUM image does not include required dbtools fixes, use a newer ORDS build.

### 4. Create a dynamic group for the instance

Use a matching rule for your VM OCID:

```text
All {instance.id = '<INSTANCE_OCID>'}
```

Save the dynamic group name as `<DYNAMIC_GROUP_NAME>`.

### 5. Create IAM policy for the dynamic group (least privilege)

Current beta guidance may temporarily require broader policy scope:

```text
Allow dynamic-group <DYNAMIC_GROUP_NAME> to manage all-resources in compartment id <COMPARTMENT_OCID>
```

When finer-grained verbs are available, tighten this policy immediately to only required Database Tools permissions.

### 6. Create Database Tools connection to local Base DB

Create a Database Tools connection that points to your Base DB local listener.

- Local connection string example from config steps: `:1540/FREEPDB1`
- This shorthand behaves like a local connection on that host.

CLI skeleton:

```bash
oci dbtools connection create-oracle-database \
  --compartment-id <COMPARTMENT_OCID> \
  --endpoint https://dbtools.us-phoenix-1.oci.oraclecloud.com \
  --display-name BASE_DB_LOCAL \
  --user-name <db_user> \
  --user-password-secret-id <vault-secret-ocid>
```

Record the resulting Database Tools connection OCID as `<DBTOOLS_CONNECTION_OCID>`.

### 7. Create Database API Gateway Config

Choose metadata mode:

- `DATABASE`: metadata resolved from ORDS metadata in DB.
- `CLOUD`: metadata managed in the config service (`apiSpecs` / `autoApiSpecs`).

```bash
DBTOOLS_ENDPOINT="https://dbtools.us-phoenix-1.oci.oraclecloud.com"
COMPARTMENT_OCID="<your-compartment-ocid>"

oci raw-request --http-method POST \
  --target-uri "$DBTOOLS_ENDPOINT/20201005/databaseToolsDatabaseApiGatewayConfigs" \
  --request-body '{
    "type": "DEFAULT",
    "displayName": "ORDS DEMO CONFIG",
    "metadataSource": "CLOUD",
    "compartmentId": "'"$COMPARTMENT_OCID"'"
  }' \
  --profile DEFAULT
```

Retrieve and export the config OCID:

```bash
export CONFIG_OCID="<your-config-ocid>"
oci raw-request --http-method GET \
  --target-uri "$DBTOOLS_ENDPOINT/20201005/databaseToolsDatabaseApiGatewayConfigs/$CONFIG_OCID" \
  --profile DEFAULT
```

### 8. Update global settings

```bash
oci raw-request --http-method PUT \
  --target-uri "$DBTOOLS_ENDPOINT/20230222/databaseToolsDatabaseApiGatewayConfigs/$CONFIG_OCID/globals/SETTINGS" \
  --request-body '{
    "type": "DEFAULT",
    "poolRoute": "PATH",
    "databaseApiStatus": "ENABLED",
    "httpPort": 8088,
    "httpsPort": 0,
    "documentRoot": "/var/www/html",
    "advancedProperties": {
      "cache.metadata.enabled": "true"
    }
  }' \
  --profile DEFAULT
```

### 9. Create pools mapped to Database Tools connections

```bash
DBTOOLS_CONNECTION_OCID="<DBTOOLS_CONNECTION_OCID>"

oci raw-request --http-method POST \
  --target-uri "$DBTOOLS_ENDPOINT/20230222/databaseToolsDatabaseApiGatewayConfigs/$CONFIG_OCID/pools" \
  --request-body '{
    "type": "DEFAULT",
    "mapping": "ords-pool",
    "displayName": "BASE_DB pool in dev Environment",
    "databaseToolsConnectionId": "'"$DBTOOLS_CONNECTION_OCID"'",
    "maxPoolSize": 100,
    "minPoolSize": 20,
    "initialPoolSize": 20,
    "jwtProfileJwkUrl": "<your-jwt-profile-url>",
    "jwtProfileIssuer": "<your-jwt-profile-issuer>",
    "jwtProfileAudience": "<your-jwt-profile-audience>",
    "restEnabledSqlStatus": "ENABLED"
  }' \
  --profile DEFAULT
```

### Starting ORDS with configuration OCID

```bash
./ords --java-options "-Dconfig.url=https://dbtools.us-phoenix-1.oci.oraclecloud.com" \
  serve --ocid "$CONFIG_OCID"
```

You can also use:

```bash
./ords --java-options "-Dconfig.url=https://dbtools.us-phoenix-1.oci.oraclecloud.com -Doci.profile=DEFAULT" \
  serve --ocid "$CONFIG_OCID"
```

### Runtime validation checks

```bash
curl -i -u HR:'<hr-password>' http://<compute-host>:8088/ords/ords-pool/api/dev/pets
```

### Troubleshooting pointers

- Wrong configuration OCID causes startup or fetch failure.
- Wrong pool key or mapping causes route resolution failure.
- Missing dynamic group/policy prevents config retrieval.
- Missing Database Tools connection OCID causes pool creation failure.

## 10. Cloud Metadata Migration

Use this section for **cloud metadata migration** from ORDS PL/SQL definitions to cloud-managed artifacts.

### Migration target artifacts

- `ords_config_service/apispec.json`
- `ords_config_service/oci_requests.sh`

### Scope

- Convert PL/SQL metadata definitions (`ords.define_module`, `ords.define_template`, `ords.define_handler`, `ords.define_parameter`) into API Spec JSON.
- Convert qualifying auto REST object definitions (`ords.enable_object`, `ORDS_METADATA.ORDS.ENABLE_OBJECT`) into `autoApiSpecs` OCI requests.

### Required deliverables

- `ords_config_service/apispec.json` for standard ORDS definitions (`ords.define_module`, `ords.define_template`, `ords.define_handler`, `ords.define_parameter`).
- `ords_config_service/oci_requests.sh` for OCI raw requests (always include the API Spec create request and include one Auto API Spec request per qualifying `ords.enable_object`).

When this skill is used to generate those two migration artifacts, you must also apply the required sample-app code updates from this document:

- Apply section **7. Oracle IAM Target Setup** (`app/utils/auth.server.ts`, OIDC routes/constants/profile model, login form action changes, logout constants import).
- Apply the **Code baseline update for sample app endpoint composition** shown in this section (normalize `BASE_ENDPOINT` and set `STATS_ENDPOINT` from it).
- Treat these as implementation changes in app source files, not documentation-only notes.
- If a referenced file/path does not exist in the target workspace, report it explicitly and continue generating migration artifacts for available inputs.

### Migration guardrails

- Accepted migration input is **PL/SQL definition source only** from `.sql` artifacts.
- Allowed statements for extraction are only ORDS PL/SQL definitions such as `ords.define_module`, `ords.define_template`, `ords.define_handler`, `ords.define_parameter`, `ords.define_privilege`, `ords.define_role`, `ords.define_client`, and SQL-side `ORDS_METADATA.ORDS.ENABLE_OBJECT` / `ords.enable_object`.
- Do not use JavaScript constants, generated JSON, exported metadata-catalog documents, runtime endpoint discovery, or any non-PL/SQL source as migration input.
- For auto REST objects, derive qualifying `autoApiSpecs` entries only from SQL `ENABLE_OBJECT` calls where `p_enabled => TRUE` and `p_auto_rest_auth => TRUE`.
- Build `apispec.json` from ORDS source metadata only. Do not generate it by copying any existing OpenAPI file.
- Do not use any OpenAPI definition (including metadata-catalog exports, Swagger/OpenAPI files, or prior `openapi.json`) as migration source input under any circumstance. API Spec migration must be derived only from `ords.define_*` metadata so `x-dbtools-operation` and `x-dbtools-properties` are preserved correctly.
- Scope boundary for migration artifacts: always update `ords_config_service/apispec.json` and `ords_config_service/oci_requests.sh`. In addition, apply the required sample-app code changes listed above when the target app files are present. Do not modify Auth0 baseline or Oracle IAM target setup guidance unless explicitly requested.
- Do not read from ORDS metadata-catalog OpenAPI (for example `/metadata-catalog/openapi.json`) during migration. This skill is PL/SQL-definition-only for migration input.
- Keep custom REST and Auto REST artifacts separate.
- Map `ords.define_*` metadata to OpenAPI paths and preserve vendor metadata (`x-dbtools-operation` / `x-dbtools-properties`).
- For qualifying auto REST objects (`ENABLE_OBJECT` with `p_enabled => TRUE` and `p_auto_rest_auth => TRUE`), generate separate OCI `autoApiSpecs` request artifacts and keep those objects out of OpenAPI `paths`.
- Keep conversion deterministic. When metadata is ambiguous, use the safest minimal valid API Spec output that can be derived from ORDS PL/SQL metadata and document the gap.

### ORDS sample app compatibility profile

- Use OpenAPI `3.1.0`.
- Normalize collection endpoint paths without trailing slash (for example `/authuser/v1/events`, `/euser/v1/artists`, `/euser/v1/cities`, `/euser/v1/events`, `/euser/v1/eventsHome`, `/euser/v1/eventStatus`, `/euser/v1/musicGenres`, `/euser/v1/venues`, `/euser/v1/landing_page_global_stats`).
- Keep application endpoint constants aligned to the normalized handler path exactly.
- Use operation-level tags and include top-level module tags (`concert_app.euser.v1`, `concert_app.authuser.v1`, `concert_app.adminuser.v1`).
- For this sample app compatibility profile, document operations with `200` response keys and include explicit `application/json` schemas (`object` or `array` with `additionalProperties: true`) so OCI config-service accepts and renders operations consistently.
- For public `euser` endpoints set `security: []` explicitly.
- For protected endpoints use operation-level bearer scopes exactly as follows: `authuser` => `[{"bearerAuth":["concert_app_authuser"]}]`, `adminuser` => `[{"bearerAuth":["concert_app_admin"]}]`.
- Use flattened `x-dbtools-properties` shape: `{"itemsPerPage": <number>, "published": "PUBLISHED"}`.
- Validation rule: `x-dbtools-properties.itemsPerPage` must be an integer `>= 1`.
- Keep `x-dbtools-operation.source` semantically intact while normalizing SQL string escaping for JSON output (single SQL quotes in text, not doubled quote artifacts).

### Mapping rules

- `ords.define_module`: use module/base path for path grouping, do not invent a standalone module object.
- `ords.define_template`: append template to base path and normalize `:id` tokens to `{id}`.
- `ords.define_handler`: create operation by HTTP method and preserve `sourceType` and `source` in `x-dbtools-operation`; preserve `parameters` when present. In the ORDS sample app compatibility profile, do not emit `moduleName`, `pattern`, or `method` unless the target runtime explicitly requires them.
- `p_source_type`: copy exactly to `x-dbtools-operation.sourceType`; never infer from SQL text.
- `p_source`: preserve verbatim in `x-dbtools-operation.source` (JSON escaping only).
- Parser correctness requirement: extract `ORDS.DEFINE_HANDLER` blocks with parenthesis-depth and SQL string-literal awareness; never terminate parsing on `);` found inside SQL/PLSQL text.
- Validation requirement: every generated operation must include a non-empty `x-dbtools-operation.source`; fail generation when any source is empty.
- `ords.define_parameter`: map URI params to OpenAPI parameters and preserve binding metadata under `x-dbtools-operation.parameters`.
- `ords.enable_object` / `ORDS_METADATA.ORDS.ENABLE_OBJECT`: produce separate `autoApiSpecs` OCI requests (not OpenAPI paths) for qualifying objects only, using SQL definitions as the only source.
- Never invent request body fields, response schema fields, scopes, or roles that are not explicit in source metadata.
- If metadata is missing, omit fields rather than invent values, then document the gap.

### Security rules

- Only migrate security proven by source metadata (`ords.define_privilege`, `ords.define_role`, and when relevant `ords.define_client`).
- API Spec security may be root-level or operation-level; operation-level overrides root-level for that operation.
- For this sample app, prefer operation-level security declarations so mixed public/protected endpoints remain explicit.
- API Spec allowed models: HTTP Basic, HTTP Bearer (JWT role-based pools), OAuth2 (scope-based pools), OpenID Connect (scope-based pools), or public access when security is empty/omitted.
- For this sample app JWT pool profile, define `components.securitySchemes.bearerAuth` as `{ "type": "http", "scheme": "bearer", "bearerFormat": "JWT" }` (do not use OAuth2 for this bearer scheme).
- Auto API Spec uses `securitySchemes` values `BASIC` or `BEARER` (or public access by omission/empty array), aligned to the target pool JWT profile.
- Scope-based security must map to scope-based pool profiles; role-based security must map to role-based pool profiles.
- Do not mix scopes and roles in the same Auto API Spec security definition.

### Code baseline update for sample app endpoint composition

```ts
export const BASE_ENDPOINT = (ADBS_ENDPOINT || "").replace(/\/$/, "");
export const STATS_ENDPOINT = `${BASE_ENDPOINT}/euser/v1/landing_page_global_stats`;
```

Use this normalization before composing API paths to avoid double slashes and route mismatches in UI/API calls.

### Output contract for migration artifacts

- `ords_config_service/apispec.json`
- `ords_config_service/oci_requests.sh` containing OCI raw-request commands only (no headings, notes, or non-OCI commands)
- Migration output is invalid unless both files are present on every run.
- Migration output is incomplete unless required sample-app code changes from section 7 and endpoint normalization are applied (or missing paths are reported explicitly).
- `oci_requests.sh` order: first one create `apiSpecs` request, then one `autoApiSpecs` request per qualifying object (for objects that meet the qualifying rule)
- API spec content size must be between `2` and `102400` characters
- If source contains only qualifying `ords.enable_object` calls and no normal handler definitions, `apispec.json` may have empty `paths`

`ords_config_service/oci_requests.sh` (only OCI raw-request commands):

```bash
oci raw-request \
  --http-method POST \
  --target-uri "https://dbtools.us-phoenix-1.oci.oraclecloud.com/20230222/databaseToolsDatabaseApiGatewayConfigs/$CONFIG_OCID/pools/$POOL_KEY/apiSpecs" \
  --request-body "$(jq -Rs --arg type "DEFAULT" --arg displayName "Demo API Spec" '{type:$type, displayName:$displayName, content:.}' ords_config_service/apispec.json)" \
  --profile DEFAULT

oci raw-request \
  --http-method POST \
  --target-uri "https://dbtools.us-phoenix-1.oci.oraclecloud.com/20230222/databaseToolsDatabaseApiGatewayConfigs/$CONFIG_OCID/pools/$POOL_KEY/autoApiSpecs" \
  --request-body '{
    "type": "DEFAULT",
    "displayName": "The SEARCH_ARTIST_VIEW VIEW",
    "databaseObjectName": "SEARCH_ARTIST_VIEW",
    "databaseObjectType": "VIEW",
    "description": "This is a rest API of SEARCH_ARTIST_VIEW",
    "alias": "search_artist_view",
    "operations": ["READ"]
  }' \
  --profile DEFAULT
```

### Full reference example (PL/SQL -> API Spec + Auto API Spec)

Input ORDS PL/SQL source:

```sql
BEGIN
  ORDS.DEFINE_MODULE(
    p_module_name    => 'music',
    p_base_path      => '/music/',
    p_items_per_page => 25,
    p_status         => 'PUBLISHED'
  );

  ORDS.DEFINE_TEMPLATE(
    p_module_name => 'music',
    p_pattern     => 'artists/:id'
  );

  ORDS.DEFINE_HANDLER(
    p_module_name => 'music',
    p_pattern     => 'artists/:id',
    p_method      => 'GET',
    p_source_type => 'json/collection',
    p_source      => 'select artist_id, name from artists where artist_id = :id'
  );

  ORDS.DEFINE_PARAMETER(
    p_module_name        => 'music',
    p_pattern            => 'artists/:id',
    p_method             => 'GET',
    p_name               => 'id',
    p_bind_variable_name => 'id',
    p_source_type        => 'URI',
    p_param_type         => 'STRING',
    p_access_method      => 'IN'
  );

  ORDS.CREATE_ROLE(p_role_name => 'concert_app_authuser');

  DECLARE
    l_roles    OWA.VC_ARR;
    l_modules  OWA.VC_ARR;
    l_patterns OWA.VC_ARR;
  BEGIN
    l_roles(1) := 'concert_app_authuser';
    l_modules(1) := 'music';
    l_patterns(1) := '/music/*';
    ORDS.DEFINE_PRIVILEGE(
      p_privilege_name => 'music.read',
      p_roles          => l_roles,
      p_patterns       => l_patterns,
      p_modules        => l_modules,
      p_label          => 'music read privilege',
      p_description    => 'Read access for music endpoints'
    );
  END;

  ORDS.ENABLE_OBJECT(
    p_enabled        => TRUE,
    p_schema         => 'ORDS_CONCERT_APP',
    p_object         => 'SEARCH_ARTIST_VIEW',
    p_object_type    => 'VIEW',
    p_object_alias   => 'search_artist_view',
    p_auto_rest_auth => TRUE
  );

  COMMIT;
END;
/
```

Generated `ords_config_service/apispec.json`:

```json
{
  "openapi": "3.1.0",
  "info": {
    "title": "Demo API Spec",
    "version": "1.0.0"
  },
  "paths": {
    "/music/artists/{id}": {
      "get": {
        "operationId": "get_music_artists_id",
        "parameters": [
          {
            "name": "id",
            "in": "path",
            "required": true,
            "schema": {
              "type": "string"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "Response",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "type": "object",
                    "additionalProperties": true
                  }
                }
              }
            }
          }
        },
        "security": [
          {
            "bearerAuth": ["concert_app_authuser"]
          }
        ],
        "x-dbtools-operation": {
          "moduleName": "music",
          "pattern": "artists/:id",
          "method": "GET",
          "sourceType": "json/collection",
          "source": "select artist_id, name from artists where artist_id = :id",
          "parameters": [
            {
              "name": "id",
              "bindVariable": "id",
              "sourceType": "URI",
              "parameterType": "STRING",
              "accessMethod": "IN",
              "description": ""
            }
          ]
        }
      }
    }
  },
  "components": {
    "securitySchemes": {
      "bearerAuth": {
        "type": "http",
        "scheme": "bearer",
        "bearerFormat": "JWT"
      }
    }
  },
  "x-dbtools-properties": {
    "itemsPerPage": 25,
    "published": "PUBLISHED"
  }
}
```

Generated `ords_config_service/oci_requests.sh`:

```bash
oci raw-request \
  --http-method POST \
  --target-uri "https://dbtools.us-phoenix-1.oci.oraclecloud.com/20230222/databaseToolsDatabaseApiGatewayConfigs/$CONFIG_OCID/pools/$POOL_KEY/apiSpecs" \
  --request-body "$(jq -Rs --arg type "DEFAULT" --arg displayName "Demo API Spec" '{type:$type, displayName:$displayName, content:.}' ords_config_service/apispec.json)" \
  --profile DEFAULT

oci raw-request \
  --http-method POST \
  --target-uri "https://dbtools.us-phoenix-1.oci.oraclecloud.com/20230222/databaseToolsDatabaseApiGatewayConfigs/$CONFIG_OCID/pools/$POOL_KEY/autoApiSpecs" \
  --request-body '{
    "type": "DEFAULT",
    "displayName": "The SEARCH_ARTIST_VIEW VIEW",
    "databaseObjectName": "SEARCH_ARTIST_VIEW",
    "databaseObjectType": "VIEW",
    "description": "This is a rest API of SEARCH_ARTIST_VIEW",
    "alias": "search_artist_view",
    "operations": ["READ"]
  }' \
  --profile DEFAULT
```

Example rule checks this reference must satisfy:

- `SEARCH_ARTIST_VIEW` does not appear in OpenAPI `paths`.
- Auto API Spec request exists because `p_enabled=true` and `p_auto_rest_auth=true`.
- `x-dbtools-operation.sourceType` exactly matches `p_source_type`.
- `x-dbtools-operation.source` is preserved verbatim from `p_source`.

### Optional: create auto API specs

Use this only for objects that explicitly meet both conditions:

- `p_enabled => true`
- `p_auto_rest_auth => true`

```bash
POOL_KEY="<pool-ocid>"
oci raw-request --http-method POST \
  --target-uri "$DBTOOLS_ENDPOINT/20230222/databaseToolsDatabaseApiGatewayConfigs/$CONFIG_OCID/pools/$POOL_KEY/autoApiSpecs" \
  --request-body '{
    "type": "DEFAULT",
    "displayName": "The SEARCH_ARTIST_VIEW VIEW",
    "databaseObjectName": "SEARCH_ARTIST_VIEW",
    "databaseObjectType": "VIEW",
    "description": "This is a rest API of SEARCH_ARTIST_VIEW",
    "alias": "search_artist_view",
    "operations": ["READ"]
  }' \
  --profile DEFAULT
```

### Cloud metadata migration validation rules

- `apispec.json` must be valid JSON.
- Paths must match original ORDS module/template structure.
- HTTP methods must match original handlers.
- Parameters must be preserved.
- Auto API Spec objects must map to correct schema objects.
- No personal machine paths should appear in generated commands.

## 11. Validation Workflow

Use this section after baseline or target setup.

### Environment validation

- Verify required environment variables are present.
- Ensure placeholders were replaced where required.

### JWT validation

Check token claims:

- `iss`
- `aud`
- `exp`
- `scope`

### App validation

Check:

- Login
- Logout
- Callback route
- Session handling
- Protected route access

Use this reset sequence when `.env` values change:

```bash
npm run drop
npm run migrate
npm run seed
npm run dev
```

### ORDS validation

Positive checks:

- Public endpoint works.
- Protected user endpoint works with valid user token.
- Protected admin endpoint works with valid admin token.

Negative checks:

- Missing token fails.
- Invalid token fails.
- Expired token fails.
- Wrong audience fails.
- Wrong issuer fails.
- Missing scope fails.
- Non-admin user cannot access admin endpoint.

### OCS validation

Check:

- Configuration OCID exists.
- Pool exists.
- Database Tools connection exists.
- ORDS starts using config OCID.
- Runtime can reach database.
- API spec is published or available.

## 12. Troubleshooting

### Login failures

Common causes:

- Wrong callback URL
- Wrong client ID
- Wrong client secret
- Wrong authorization endpoint
- Wrong token endpoint

### JWT validation failures

Common causes:

- Wrong issuer
- Wrong audience
- Wrong JWKS URL
- Expired token
- Missing scope

### Authorization failures

Common causes:

- User does not have required scope
- ORDS privilege mapping is wrong
- Claim name mismatch

### OCS startup failures

Common causes:

- Wrong configuration OCID
- Wrong OCI region
- Missing permission
- Missing Database Tools connection
- Wrong pool key
- ORDS cannot access OCI configuration

### API spec migration failures

Common causes:

- Invalid JSON
- Incorrect path mapping
- Missing handler method
- Missing parameter mapping
- Schema object not enabled
- Auto API Spec object mismatch

## 13. Style and Cleanup Requirements

While restructuring or maintaining this skill:

- Preserve useful technical details from existing instructions.
- Remove duplicated explanations.
- Replace personal local paths with generic repo-relative paths.
- Use consistent naming:
  - `Auth0 baseline`
  - `Oracle IAM target`
  - `OCI Database API Gateway / OCS`
  - `ORDS authorization`
  - `cloud metadata migration`
- Use clear markdown headings.
- Use tables where helpful.
- Use fenced code blocks for commands and config examples.
- Do not add unrelated project management sections.
- Do not split this skill into multiple markdown files.
- Do not remove important setup logic.
- Do not invent real secrets, OCIDs, domains, or customer-specific values.


## Sources

- [How to Secure Oracle Database REST APIs with OCI IAM (IDCS) JSON Web Tokens and Role-Based Access Claims, Part One](https://blogs.oracle.com/database/ords-apis-iam-jwts-role-based-claims-part-one)
- [Auth0 management API](https://auth0.com/docs/api/management/v2)
- [Deploying ORDS with the OCI Database API Gateway Configuration Service](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/26.1/ordig/configuring-additional-databases.html#GUID-707DE44D-50D3-42CE-8350-FF69E53A6B6C)
