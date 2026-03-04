# notification-service - CLAUDE.md

## What This Service Does

Minimal email notification service. Owns the SQS `MailQueue` and exposes its ARN/URL as CloudFormation outputs for other services. The `sendMail` Lambda is triggered by SQS events and sends emails via AWS SES.

## Key Files

| File                       | Purpose                                              |
| -------------------------- | ---------------------------------------------------- |
| `serverless.yml`           | Function definition, SQS trigger, IAM, stack outputs |
| `src/handlers/sendMail.js` | SQS event handler - parses message, sends via SES    |
| `resources/MailQueue.yml`  | CloudFormation SQS queue definition + stack exports  |
| `iam/MailQueueIAM.yml`     | IAM for SQS receive/delete messages                  |
| `iam/SendMailIAM.yml`      | IAM for SES send email                               |

## sendMail Handler Pattern

```js
// SQS event structure (batch size: 1)
const { subject, body, recipient } = JSON.parse(event.Records[0].body);

await ses
  .sendEmail({
    Source: "ariel@codingly.io",
    Destination: { ToAddresses: [recipient] },
    Message: {
      Subject: { Data: subject },
      Body: { Text: { Data: body } },
    },
  })
  .promise();
```

## CloudFormation Exports

These outputs are what other services depend on:

```yaml
# In resources/MailQueue.yml
Outputs:
  MailQueueArn:
    Export:
      Name: !Sub "notification-service-${self:provider.stage}.MailQueueArn"
  MailQueueUrl:
    Export:
      Name: !Sub "notification-service-${self:provider.stage}.MailQueueUrl"
```

**Do not rename these exports** - `auction-service` references them via `cf:notification-service-{stage}.MailQueueArn`.

## SES Configuration

- Region: `eu-west-1` (hardcoded in `sendMail.js` SES client initialization)
- Sender: `ariel@codingly.io`

To change sender or region, update `sendMail.js` directly. No env var is used for these.

## Adding New Notification Types

The message schema (`{ subject, body, recipient }`) is a convention, not enforced by this service. Any service that sends a correctly-formed SQS message to the MailQueue will trigger an email. To extend notification logic, modify `sendMail.js`.

## SQS Trigger Config

```yaml
# In serverless.yml
events:
  - sqs:
      arn: !GetAtt MailQueue.Arn
      batchSize: 1
```

`batchSize: 1` means one email per Lambda invocation. If SES throttling becomes an issue, consider batching and adding retry logic.

## Deployment

Deploy before `auction-service`:

```bash
npm install
sls deploy --stage dev
sls deploy --stage prod
```

Remove:

```bash
sls remove --stage dev
```
