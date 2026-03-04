# auction-service - CLAUDE.md

## What This Service Does

Core auction management service. Creates, retrieves, and manages auctions stored in DynamoDB. Accepts bids, uploads pictures to S3, and closes expired auctions via a scheduled Lambda. Sends email notification triggers to the `notification-service` via SQS.

## Key Patterns

### Middleware

All handlers use `commonMiddleware` from `src/lib/commonMiddleware.js`, which stacks:

- `@middy/json-body-parser`
- `@middy/event-normalizer`
- `@middy/http-error-handler`
- `@middy/http-cors`

Add any new handler by wrapping: `export const handler = commonMiddleware(myHandler)`

### Input Validation

Use `@middy/validator` with JSON schemas in `src/lib/schemas/`. All schemas use JSON Schema draft-07. Add a new schema file and import it into the handler.

### Error Handling

Use `http-errors` for all HTTP errors:

```js
import createError from "http-errors";
throw new createError.NotFound("Auction not found!");
```

`http-error-handler` middleware converts these to proper HTTP responses automatically.

### User Identity

The Auth0 JWT authorizer injects the user's email into the Lambda context:

```js
const { email } = event.requestContext.authorizer;
```

Use this for ownership checks (seller/bidder comparisons).

### DynamoDB Access

DynamoDB client is initialized per-handler. The table name comes from `process.env.AUCTIONS_TABLE_NAME`.

Common operations pattern:

```js
const params = {
  TableName: process.env.AUCTIONS_TABLE_NAME,
  Key: { id },
};
const result = await dynamodb.get(params).promise();
```

### SQS Notifications

To send a notification, post to the MailQueue:

```js
const params = {
  QueueUrl: process.env.MAIL_QUEUE_URL,
  MessageBody: JSON.stringify({ subject, body, recipient }),
};
await sqs.sendMessage(params).promise();
```

The `notification-service` consumes these messages and sends emails via SES.

## File Map

| File                                   | Purpose                                                  |
| -------------------------------------- | -------------------------------------------------------- |
| `serverless.yml`                       | All Lambda functions, events, resources, IAM, env vars   |
| `src/handlers/createAuction.js`        | POST /auction                                            |
| `src/handlers/getAuctions.js`          | GET /auctions (GSI query by status)                      |
| `src/handlers/getAuction.js`           | GET /auction/{id}                                        |
| `src/handlers/placeBid.js`             | PATCH /auction/{id}/bid                                  |
| `src/handlers/uploadAuctionPicture.js` | PATCH /auction/{id}/picture                              |
| `src/handlers/processAuctions.js`      | Scheduled: close expired auctions, trigger notifications |
| `src/lib/commonMiddleware.js`          | Shared Middy middleware stack                            |
| `src/lib/closeAuction.js`              | DynamoDB update to set status=CLOSED                     |
| `src/lib/getEndedAuctions.js`          | GSI query for expired OPEN auctions                      |
| `src/lib/uploadPictureToS3.js`         | S3 PutObject helper                                      |
| `src/lib/setAuctionPictureUrl.js`      | DynamoDB update to set pictureUrl                        |
| `resources/AuctionsTable.yml`          | CloudFormation DynamoDB table definition                 |
| `resources/AuctionsBucket.yml`         | CloudFormation S3 bucket definition                      |
| `iam/AuctionsTableIAM.yml`             | IAM for DynamoDB access                                  |
| `iam/MailQueueIAM.yml`                 | IAM for SQS send access                                  |
| `iam/AuctionsBucketIAM.yml`            | IAM for S3 put access                                    |

## Environment Variables

| Variable               | Source                                         |
| ---------------------- | ---------------------------------------------- |
| `AUCTIONS_TABLE_NAME`  | CloudFormation ref to AuctionsTable            |
| `MAIL_QUEUE_URL`       | `cf:notification-service-{stage}.MailQueueUrl` |
| `AUCTIONS_BUCKET_NAME` | CloudFormation ref to AuctionsBucket           |

## Cross-Service Dependencies

- **notification-service** must be deployed first - this service reads `cf:notification-service-{stage}.MailQueueArn` and `cf:notification-service-{stage}.MailQueueUrl` from CloudFormation outputs.
- **auth-service** must be deployed - the authorizer ARN is hardcoded as `arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:auth-service-${stage}-auth`.

## Adding a New Endpoint

1. Create handler in `src/handlers/myHandler.js` wrapped with `commonMiddleware`
2. Add JSON schema in `src/lib/schemas/myHandlerSchema.js` if input validation is needed
3. Add the function definition to `serverless.yml` under `functions:`
4. If new AWS resources or permissions are needed, add them in `resources/` and `iam/`

## Bid Validation Logic (placeBid.js)

Order of checks:

1. Fetch auction by ID (404 if not found)
2. Auction must be OPEN (403 if CLOSED)
3. Bidder email != seller email (403)
4. Bidder email != current highest bidder email (403)
5. Amount > current highestBid.amount (422)
6. Update DynamoDB with new bid amount and bidder email

## Deployment

```bash
npm install
sls deploy --stage dev
sls deploy --stage prod
```

Remove a deployment:

```bash
sls remove --stage dev
```
