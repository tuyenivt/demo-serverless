# Demo Serverless

A serverless auction platform built on AWS using the Serverless Framework. The project follows a microservices architecture with three independent services that communicate asynchronously via SQS.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                  API Gateway (REST)                  │
└──────┬───────────────────────────────────────────────┘
       │
       ├─► auth-service       JWT verification (Auth0)
       │
       └─► auction-service    Core auction API (protected)

┌──────────────────┐    ┌──────────────────────────────┐
│  DynamoDB        │    │  S3 Bucket                   │
│  AuctionsTable   │    │  (auction pictures)          │
└──────────────────┘    └──────────────────────────────┘

┌──────────────────┐    ┌──────────────────────────────┐
│  SQS MailQueue   │───►│  notification-service        │
│  (async events)  │    │  AWS SES email delivery      │
└──────────────────┘    └──────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  CloudWatch Events (every 1 min)                     │
│  → processAuctions: closes expired auctions          │
└──────────────────────────────────────────────────────┘
```

## Services

| Service                                         | Description                       | Docs                                       |
| ----------------------------------------------- | --------------------------------- | ------------------------------------------ |
| [auction-service](./auction-service/)           | Core auction management API       | [README](./auction-service/README.md)      |
| [auth-service](./auth-service/)                 | JWT authentication via Auth0      | [README](./auth-service/README.md)         |
| [notification-service](./notification-service/) | SQS-triggered email notifications | [README](./notification-service/README.md) |

## Tech Stack

- **Runtime:** Node.js 12.x
- **Framework:** [Serverless Framework](https://www.serverless.com/)
- **Cloud:** AWS (Lambda, API Gateway, DynamoDB, S3, SQS, SES)
- **Auth:** Auth0 (JWT RS256)
- **Middleware:** [Middy](https://middy.js.org/)
- **Build:** serverless-bundle (Webpack + Babel)

## Prerequisites

- Node.js 12+
- [Serverless Framework CLI](https://www.serverless.com/framework/docs/getting-started/) installed globally
- AWS CLI configured with appropriate credentials
- An [Auth0](https://auth0.com/) account for authentication

## Deployment Order

Services have cross-service dependencies via CloudFormation outputs. Deploy in this order:

```bash
# 1. Deploy notification service first (exports SQS queue ARN/URL)
cd notification-service && sls deploy

# 2. Deploy auth service (provides the Lambda authorizer)
cd auth-service && sls deploy

# 3. Deploy auction service last (depends on both above)
cd auction-service && sls deploy
```

## Environment & Configuration

Each service is configured via `serverless.yml`. No `.env` files are required - all environment variables are resolved at deploy time.

**Auth service requires** a `secret.pem` file containing the Auth0 RS256 public key placed in `auth-service/src/handlers/secret.pem`.

## API Overview

All auction endpoints require a Bearer token in the `Authorization` header (issued by Auth0).

| Method  | Path                    | Description                                      |
| ------- | ----------------------- | ------------------------------------------------ |
| `POST`  | `/auction`              | Create a new auction                             |
| `GET`   | `/auctions`             | List auctions (filter by `?status=OPEN\|CLOSED`) |
| `GET`   | `/auction/{id}`         | Get a single auction                             |
| `PATCH` | `/auction/{id}/bid`     | Place a bid                                      |
| `PATCH` | `/auction/{id}/picture` | Upload auction picture (base64)                  |

## Auction Lifecycle

1. Auction created with 1-hour duration, status `OPEN`
2. Users place bids (validated: no self-bidding, no double-bidding, must exceed current bid)
3. Seller can upload a picture (base64-encoded)
4. Every minute, a scheduled function closes expired auctions (status → `CLOSED`)
5. Notifications are sent via SQS → SES to seller and winning bidder
