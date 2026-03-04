# notification-service

Asynchronous email notification service. An SQS queue receives messages from the `auction-service`, and a Lambda function processes them to send emails via AWS SES.

## How It Works

1. `auction-service` sends a message to the `MailQueue` SQS queue when an auction closes
2. SQS triggers the `sendMail` Lambda (batch size: 1)
3. The Lambda sends an email via AWS SES

## SQS Message Format

Messages must follow this structure:

```json
{
  "subject": "Your auction has ended",
  "body": "Your item \"Widget\" did not receive any bids.",
  "recipient": "seller@example.com"
}
```

## Email Scenarios

The `auction-service` sends the following types of notifications when an auction closes:

| Scenario                  | Recipient      | Message                |
| ------------------------- | -------------- | ---------------------- |
| Auction closed, no bids   | Seller         | Item received no bids  |
| Auction closed, item sold | Seller         | Item sold for X amount |
| Auction won               | Highest bidder | You won the auction    |

## AWS Resources

### SQS - MailQueue

- Created by this service and exported for cross-service use
- **Exported outputs:**
  - `MailQueueArn` - consumed by `auction-service` for IAM permissions
  - `MailQueueUrl` - consumed by `auction-service` to send messages

### SES - Email Sending

- Region: `eu-west-1` (hardcoded in handler)
- From address: `ariel@codingly.io`
- **SES must be configured in your AWS account** for the sender address (and recipient addresses in sandbox mode)

## Project Structure

```
notification-service/
├── serverless.yml
├── package.json
├── iam/
│   ├── MailQueueIAM.yml    # SQS receive permissions
│   └── SendMailIAM.yml     # SES send permissions
├── resources/
│   └── MailQueue.yml       # SQS queue CloudFormation definition
└── src/
    └── handlers/
        └── sendMail.js     # SQS event handler
```

## Deployment

**Deploy this service first** - `auction-service` depends on the CloudFormation stack outputs from this service.

```bash
cd notification-service
npm install
sls deploy --stage dev
```

**Configuration:**

- Default stage: `dev`
- Default region: `us-east-1`
- Memory: 256MB
- Runtime: Node.js 12.x

## AWS SES Setup

Before emails can be sent, configure SES in `eu-west-1`:

1. Verify the sender address `ariel@codingly.io` (or update to your address in `sendMail.js`)
2. In SES sandbox mode, also verify all recipient email addresses
3. Request production access to send to unverified addresses

To change the sender address, update `sendMail.js`:

```js
Source: 'your-email@example.com',
```
