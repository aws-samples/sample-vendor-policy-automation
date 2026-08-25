# Serverless GenAI Pattern for Automating Contractual Terms Monitoring

A reusable, serverless pattern that automatically monitors vendor contractual terms (Terms of
Service, Privacy Policies, Data Processing Agreements, SLAs, and more) across cloud and SaaS
providers. It detects material changes, summarizes them with a generative AI model, and emails
compliance, legal, and risk stakeholders — with a versioned audit trail of every change.

The workflow is orchestrated by AWS Step Functions using direct service integrations, with a single
AWS Lambda function handling content fetching, main-content extraction, model request preparation,
and email rendering.

## Architecture

```
┌──────────────┐
│   Amazon     │
│ EventBridge  │  Scheduled trigger (rate/cron)
│  Scheduler   │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   AWS Step Functions (Standard Workflow)             │
│                                                                      │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐          │
│   │ Amazon   │──▶│   AWS    │──▶│  Amazon  │──▶│  Amazon  │          │
│   │ DynamoDB │   │  Lambda  │   │ Bedrock  │   │   SNS    │          │
│   │(Registry)│   │ (Fetch,  │   │(Compare &│   │ (Notify) │          │
│   │          │   │ Extract, │   │Summarize)│   │          │          │
│   │          │   │ Render)  │   │          │   │          │          │
│   └──────────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘          │
│                       │              │              │                │
└───────────────────────┼──────────────┼──────────────┼────────────────┘
                        │              │              │
                        ▼              ▼              ▼
                  ┌──────────────────────────┐  Email subscribers
                  │   Amazon S3 (Versioned)  │  (compliance, legal,
                  │  HTML · JSON · req/resp  │   risk)
                  └──────────────────────────┘
```

## How It Works

1. **Amazon EventBridge Scheduler** triggers the Step Functions state machine on a configurable
   cadence (for example, daily or weekly).
2. Step Functions **scans the Amazon DynamoDB registry** of vendor documents and fans out with a
   **Map state**, processing each document independently.
3. For each document, Step Functions **invokes the Lambda**, which reads the previously stored
   version, fetches the current page, extracts the main legal text (dropping navigation, headers,
   footers, and marketing chrome), and writes new versions to **Amazon S3** only when the extracted
   text changed.
4. A **Choice state** routes on the flags returned by the Lambda: first run (store baseline), no
   change (update timestamp only), or changed (continue to analysis).
5. Step Functions invokes **Amazon Bedrock**, passing the request and response bodies by Amazon S3
   reference so full document text never enters the workflow state. The model compares both versions
   and returns a structured, grounded analysis.
6. If material changes are detected, the Lambda renders the analysis into a professional report,
   which Step Functions publishes to **Amazon SNS** for email delivery.
7. Step Functions **updates the DynamoDB registry** with the last-checked timestamp and change
   summary.

## AWS Services Used

- **Amazon EventBridge Scheduler** — schedules the monitoring runs.
- **AWS Step Functions** — orchestrates the workflow via direct service integrations.
- **AWS Lambda** — fetches pages, extracts main content (using BeautifulSoup via a Lambda layer),
  builds the Bedrock request, and renders the notification email.
- **Amazon S3** — versioned storage for raw HTML, extracted text, and model request/response bodies.
- **Amazon Bedrock** — foundation model that compares versions and summarizes material changes.
- **Amazon DynamoDB** — stores the vendor monitoring registry and per-document state.
- **Amazon SNS** — delivers formatted change alerts to stakeholders.

## Prerequisites

1. An AWS account with permissions to create Step Functions, S3, DynamoDB, SNS, EventBridge, Lambda,
   and IAM resources.
2. [Amazon Bedrock model access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)
   enabled for your chosen model or inference profile in the target Region.
3. The [AWS CLI](https://aws.amazon.com/cli/) installed and configured, or access to the
   [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation).
4. A BeautifulSoup Lambda layer ARN for your Region and runtime (the template defaults to a public
   community layer; for production, consider building and owning the layer).

## Deployment

Deploy the CloudFormation template `contractual-terms-monitoring.yaml`:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=$(aws configure get region)

aws cloudformation deploy \
  --stack-name contractual-terms-monitoring \
  --template-file contractual-terms-monitoring.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ScheduleExpression="rate(1 day)" \
    BedrockModelId="us.anthropic.claude-haiku-4-5-20251001-v1:0" \
    NotificationEmail="you@example.com" \
    BucketName="policy-monitoring-${ACCOUNT_ID}-${REGION}"
```

> **Note:** Choose a bucket name that is unique to your account and Region (for example, by including
> your account ID and Region as shown above). Avoid using the literal example names from this
> documentation, which could already be taken or targeted for bucket squatting.

After deployment, confirm the SNS subscription email that is sent to `NotificationEmail`.

### Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `ScheduleExpression` | How often to check for updates | `rate(1 day)` |
| `BedrockModelId` | The model or inference profile ID | `us.anthropic.claude-haiku-4-5-20251001-v1:0` |
| `NotificationEmail` | Email address for change alerts | `you@example.com` |
| `BucketName` | Globally unique S3 bucket name (include account ID/Region) | `policy-monitoring-111122223333-us-east-2` |
| `BeautifulSoupLayerArn` | Lambda layer providing beautifulsoup4 | (Region-specific ARN) |

## Adding Vendors to Monitor

Add vendor documents to the registry by inserting DynamoDB items — no code changes or redeployment
required:

```bash
aws dynamodb put-item \
  --table-name PolicyMonitoringRegistry \
  --item '{
    "vendor": {"S": "AWS"},
    "document_type": {"S": "Service Terms"},
    "url": {"S": "https://aws.amazon.com/service-terms/"},
    "check_frequency": {"S": "weekly"},
    "active": {"BOOL": true}
  }'
```

## Clean Up

To avoid ongoing charges:

1. Empty the S3 versioning bucket (including all object versions and delete markers).
2. Delete the CloudFormation stack.

## Disclaimer

Change analyses are generated by a foundation model from vendors' published text and do not constitute
legal advice. Review by qualified legal counsel is required before acting on any recommendation.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
