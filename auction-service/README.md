# auction-service

Core auction management service. Handles creating auctions, placing bids, uploading pictures, and automatically closing expired auctions. Protected by the `auth-service` Lambda authorizer.

## API Endpoints

All endpoints require `Authorization: Bearer <token>` header.

| Method  | Path                    | Description            |
| ------- | ----------------------- | ---------------------- |
| `POST`  | `/auction`              | Create a new auction   |
| `GET`   | `/auctions`             | List auctions          |
| `GET`   | `/auction/{id}`         | Get auction by ID      |
| `PATCH` | `/auction/{id}/bid`     | Place a bid            |
| `PATCH` | `/auction/{id}/picture` | Upload auction picture |

### POST /auction

Creates a new auction with a 1-hour duration.

**Body:**

```json
{ "title": "My item" }
```

**Response:**

```json
{
  "id": "uuid",
  "title": "My item",
  "status": "OPEN",
  "createdAt": "2024-01-01T00:00:00.000Z",
  "endingAt": "2024-01-01T01:00:00.000Z",
  "highestBid": { "amount": 0 },
  "seller": "user@example.com"
}
```

### GET /auctions

Query auctions by status using the DynamoDB GSI.

**Query params:** `status` - `OPEN` (default) or `CLOSED`

### PATCH /auction/{id}/bid

Place a bid on an open auction.

**Body:**

```json
{ "amount": 150 }
```

**Validation rules:**

- Auction must be `OPEN`
- Bidder cannot be the seller
- Same user cannot bid twice
- Amount must exceed the current highest bid

### PATCH /auction/{id}/picture

Upload an auction picture. Caller must be the seller.

**Body:** Raw base64-encoded image string (must end with `=`)

Returns the updated auction with a public S3 picture URL.

## Scheduled Function

**processAuctions** runs every minute via CloudWatch Events. It:

1. Queries the `statusAndEndDate` GSI for `OPEN` auctions past their `endingAt` time
2. Updates each auction's status to `CLOSED`
3. Sends SQS messages to the notification service:
   - Seller: "item sold" or "no bids" notification
   - Winning bidder: "you won" notification

## AWS Resources

### DynamoDB - AuctionsTable

| Attribute  | Type   | Key            |
| ---------- | ------ | -------------- |
| `id`       | String | HASH (primary) |
| `status`   | String | HASH (GSI)     |
| `endingAt` | String | RANGE (GSI)    |

**GSI:** `statusAndEndDate` - enables efficient querying of OPEN auctions by end date.

**Billing:** Pay-per-request

### S3 - AuctionsBucket

- Public read access
- Objects auto-expire after 1 day
- Bucket name: `auctions-bucket-sj19asxm-{stage}`

### SQS - MailQueue

Referenced from `notification-service` CloudFormation outputs:

- `cf:notification-service-{stage}.MailQueueArn`
- `cf:notification-service-{stage}.MailQueueUrl`

## Project Structure

```
auction-service/
├── serverless.yml
├── package.json
├── iam/
│   ├── AuctionsBucketIAM.yml
│   ├── AuctionsTableIAM.yml
│   └── MailQueueIAM.yml
├── resources/
│   ├── AuctionsBucket.yml
│   └── AuctionsTable.yml
└── src/
    ├── handlers/
    │   ├── createAuction.js
    │   ├── getAuction.js
    │   ├── getAuctions.js
    │   ├── placeBid.js
    │   ├── processAuctions.js
    │   └── uploadAuctionPicture.js
    └── lib/
        ├── commonMiddleware.js
        ├── closeAuction.js
        ├── getEndedAuctions.js
        ├── setAuctionPictureUrl.js
        ├── uploadPictureToS3.js
        └── schemas/
            ├── createAuctionSchema.js
            ├── getAuctionsSchema.js
            ├── placeBidSchema.js
            └── uploadAuctionPictureSchema.js
```

## Dependencies

This service **requires** `notification-service` to be deployed first. It reads CloudFormation stack outputs for the SQS queue ARN and URL.

## Deployment

```bash
cd auction-service
npm install
sls deploy --stage dev
```

**Configuration:**

- Default stage: `dev`
- Default region: `us-east-1`
- Memory: 256MB
- Runtime: Node.js 12.x
