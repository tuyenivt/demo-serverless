# auth-service - CLAUDE.md

## What This Service Does

Provides JWT authentication for the platform. The `auth` Lambda is used as an API Gateway custom authorizer - it validates Auth0 RS256 JWTs and returns IAM policy documents. Also exposes two test endpoints (`/public`, `/private`).

## Key Files

| File                      | Purpose                                                         |
| ------------------------- | --------------------------------------------------------------- |
| `serverless.yml`          | Function definitions, authorizer config, CORS gateway responses |
| `src/handlers/auth.js`    | Lambda authorizer - verifies JWT, returns IAM policy            |
| `src/handlers/public.js`  | Public test endpoint (no auth)                                  |
| `src/handlers/private.js` | Private test endpoint (uses authorizer)                         |
| `src/handlers/secret.pem` | Auth0 public key - **gitignored, must be added manually**       |

## JWT Verification Logic (auth.js)

1. Extract token from `event.authorizationToken` (format: `"Bearer <token>"`)
2. Verify with `jsonwebtoken.verify(token, process.env.AUTH0_PUBLIC_KEY, { algorithms: ['RS256'] })`
3. On success: build and return `Allow` policy with decoded claims in context
4. On failure: `throw new Error('Unauthorized')` - API Gateway returns 401

### Policy Document Structure

```js
{
  principalId: decoded.sub,           // Auth0 user ID
  policyDocument: {
    Statement: [{
      Action: 'execute-api:Invoke',
      Effect: 'Allow' | 'Deny',
      Resource: 'arn:aws:execute-api:*:*:*'  // entire API
    }]
  },
  context: { ...decoded }             // full JWT claims passed downstream
}
```

Downstream services access the user's email via:

```js
const { email } = event.requestContext.authorizer;
```

## Adding the Auth0 Public Key

```bash
# Get the PEM from Auth0 dashboard and save it:
# Applications → Your App → Settings → Advanced → Certificates → Copy PEM
echo "-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----" > src/handlers/secret.pem
```

## How Other Services Reference This Authorizer

In `auction-service/serverless.yml`, protected functions use:

```yaml
authorizer: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:auth-service-${self:custom.stage}-auth
```

The `serverless-pseudo-parameters` plugin resolves `#{AWS::Region}` and `#{AWS::AccountId}` at deploy time.

## Authorizer Caching

The authorizer result is cached for **300 seconds** (`resultTtlInSeconds: 300`). During this window, subsequent requests with the same token skip the Lambda invocation. Keep this in mind when testing - you may need to wait for the cache to expire or use a new token.

## CORS Gateway Responses

Two custom API Gateway responses are defined so that browsers receive CORS headers even on auth failures:

```yaml
GatewayResponseExpiredToken: # EXPIRED_TOKEN → 401
GatewayResponseUnauthorized: # UNAUTHORIZED → 401
```

Both include `Access-Control-Allow-Origin: '*'`. If you add new response types, follow the same pattern in `serverless.yml`.

## Deployment

```bash
npm install
sls deploy --stage dev
```

Deploy this before `auction-service` since it must be live for the authorizer ARN to resolve.
