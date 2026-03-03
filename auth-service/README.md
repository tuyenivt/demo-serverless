# auth-service

JWT authentication service for the auction platform. Provides a Lambda authorizer that validates Auth0 Bearer tokens for API Gateway, plus two test endpoints (public and private).

## How It Works

API Gateway is configured to use the `auth` Lambda as a **custom authorizer**. When a protected endpoint is called:

1. API Gateway extracts the `Authorization: Bearer <token>` header
2. Invokes the `auth` Lambda
3. The Lambda verifies the JWT against the Auth0 RS256 public key (`secret.pem`)
4. Returns an IAM policy document - `Allow` or `Deny`
5. The policy is cached for 300 seconds
6. On success, JWT claims (including `email`) are passed to downstream Lambdas via `event.requestContext.authorizer`

## Functions

### auth (Custom Authorizer)

- **Trigger:** API Gateway custom authorizer (not HTTP)
- **Input:** `event.authorizationToken` - the `Authorization` header value
- **Output:** IAM policy document
- **On invalid token:** Throws `"Unauthorized"` (triggers 401)

The authorizer grants access to the entire API Gateway instance (`arn:aws:execute-api:*:*:*`), so authorization is evaluated once per token (not per route).

### publicEndpoint

- **Method:** `POST /public`
- **Auth:** None
- Used for testing connectivity.

### privateEndpoint

- **Method:** `POST /private`
- **Auth:** Custom authorizer (`auth`)
- Used for verifying the authorization flow works end-to-end.

## Setup

### Auth0 Public Key

This service requires an Auth0 RS256 public key to verify JWTs.

1. In your Auth0 dashboard, go to **Applications → Your API → Settings → Advanced Settings → Certificates**
2. Copy the PEM certificate
3. Save it as `src/handlers/secret.pem`

The file is referenced in `serverless.yml`:

```yaml
AUTH0_PUBLIC_KEY: ${file(src/handlers/secret.pem)}
```

**The `secret.pem` file is gitignored and must be added manually before deploying.**

## Project Structure

```
auth-service/
├── serverless.yml
├── package.json
└── src/
    └── handlers/
        ├── auth.js           # Lambda authorizer
        ├── public.js         # Public test endpoint
        ├── private.js        # Private test endpoint
        └── secret.pem        # Auth0 public key (gitignored, add manually)
```

## Deployment

```bash
cd auth-service
npm install
sls deploy --stage dev
```

**Configuration:**

- Default stage: `dev`
- Default region: `us-east-1`
- Memory: 128MB
- Runtime: Node.js 12.x

## CORS on Auth Failures

Custom API Gateway responses are configured so that CORS headers are returned even on authorization failures (e.g., expired token, unauthorized). This prevents browser-side CORS errors masking authentication errors.

Handled responses:

- `EXPIRED_TOKEN` → 401
- `UNAUTHORIZED` → 401
