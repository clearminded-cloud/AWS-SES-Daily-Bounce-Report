# Amazon SES Daily Bounce & Complaint Email Messages Report
Setup a daily email report of all AWS SES bounced emails and compliants.

This report is extremely useful when using AWS SES to send critical emails and you must know which AWS SES emails bounce.
Email is an asynchronous system.  Imagine you're sending a critical email to partners.  If one day a partner no longer receives your email and it bounces, you'll find out from the daily report that this script will setup for you.

This project is a single Python setup script that deploys a fully automated daily email report of Amazon SES bounce and complaint failures — including recipient, sender, subject, bounce type, and diagnostic code — delivered to one or more recipients every morning.

Designed to run from **AWS CloudShell** with no dependencies beyond `boto3`.

---

## What It Does

- Creates two SQS queues to collect SES bounce and complaint notifications
- Creates two SNS topics and wires them to your SES identity
- Enables original message headers (`commonHeaders`) on SES notifications so that sender and subject are captured
- Deploys a Lambda function that drains the queues daily and sends an HTML email report
- Schedules the Lambda via EventBridge to run every day at 07:00 UTC
- All resources are created idempotently — safe to re-run if interrupted or when updating

---

## Report Contents

The daily email report includes three sections:

**Hard Bounces (Permanent)** — sorted by sender
| Sender | Recipient | Subject | Subtype | Diagnostic Code | Timestamp |

**Soft Bounces (Transient)** — sorted by sender
| Sender | Recipient | Subject | Subtype | Diagnostic Code | Timestamp |

**Complaints**
| Recipient | Feedback Type | Timestamp |

The email subject line summarises counts at a glance, e.g.:
```
SES Daily Report 2025-04-21 — 5 failure(s) (3 hard, 1 soft, 1 complaints)
```

If sender or subject is missing from a notification, the SES Message ID is shown in its place.

---

## Prerequisites

- AWS CloudShell access (or any environment with `boto3` and appropriate IAM permissions)
- A verified SES identity (domain or email address)
- SES available in your target region
- The report sender address must be a verified SES identity

### Required IAM Permissions

The principal running the setup script needs permissions to create and manage:

- SQS queues and queue policies
- SNS topics and subscriptions
- SES notification topics and header settings
- IAM roles and inline policies
- Lambda functions
- EventBridge rules and targets

---

## Configuration

Edit the four lines at the top of the script before running:

```python
REGION           = "us-east-2"                                     # Your AWS region
SES_IDENTITY     = "yourdomain.com"                                     # Verified SES domain or email
REPORT_RECIPIENT = ["you@yourdomain.com", "colleague@yourdomain.com"]   # One or more report recipients
REPORT_SENDER    = "you@yourdomain.com"                                 # Must be a verified SES identity
```

---

## Usage

Run from AWS CloudShell:

```bash
python3 setup-aws-ses-daily-bounce-report.py
```

### Example Output

```
~ $ python setup-aws-ses-daily-bounce-report.py

[1/7] Creating SQS queues...
  ✓ ses-bounces-queue    → https://sqs.us-east-2.amazonaws.com/012345678901/ses-bounces-queue
  ✓ ses-complaints-queue → https://sqs.us-east-2.amazonaws.com/012345678901/ses-complaints-queue

[2/7] Creating SNS topics...
  ✓ ses-bounces    → arn:aws:sns:us-east-2:012345678901:ses-bounces
  ✓ ses-complaints → arn:aws:sns:us-east-2:012345678901:ses-complaints

[3/7] Attaching SQS access policies for SNS...
  ✓ Policy set on ses-bounces-queue
  ✓ Policy set on ses-complaints-queue

[4/7] Subscribing SQS queues to SNS topics...
  ✓ ses-bounces-queue    subscribed to ses-bounces
  ✓ ses-complaints-queue subscribed to ses-complaints

[5/7] Wiring SES identity to SNS topics...
  ✓ Bounce notifications    → arn:aws:sns:us-east-2:012345678901:ses-bounces
  ✓ Complaint notifications → arn:aws:sns:us-east-2:012345678901:ses-complaints
  ✓ Original headers enabled for Bounce + Complaint notifications

[6/7] Creating Lambda IAM role...
  ✓ Role already exists → arn:aws:iam::012345678901:role/ses-daily-report-lambda-role
  ✓ Inline policy attached

[7/7] Deploying Lambda function...
  Waiting 15s for IAM role to propagate...
  ✓ Lambda function created → ses-daily-report

[+] Creating EventBridge daily trigger (07:00 UTC)...
  ✓ EventBridge rule → arn:aws:events:us-east-2:012345678901:rule/ses-daily-report-trigger
  ✓ EventBridge target set

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Setup complete!

  Bounce queue:    https://sqs.us-east-2.amazonaws.com/012345678901/ses-bounces-queue
  Complaint queue: https://sqs.us-east-2.amazonaws.com/012345678901/ses-complaints-queue
  Lambda:          ses-daily-report
  Schedule:        Daily at 07:00 UTC
  Report sent to:  me@example.com, coworker@example.com, partner@example.com
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Architecture

```
SES Identity
    │
    ├─── Bounce notifications ──► SNS (ses-bounces)
    │                                    │
    │                                    ▼
    │                            SQS (ses-bounces-queue)
    │                                    │
    └─── Complaint notifications ─► SNS (ses-complaints)
                                         │
                                         ▼
                                 SQS (ses-complaints-queue)
                                         │
                              ┌──────────┘
                              │
                    EventBridge (07:00 UTC daily)
                              │
                              ▼
                    Lambda (ses-daily-report)
                              │
                              ▼
                    SES SendEmail → Report Recipients
```

---

## AWS Resources Created

| Resource | Name |
|---|---|
| SQS Queue | `ses-bounces-queue` |
| SQS Queue | `ses-complaints-queue` |
| SNS Topic | `ses-bounces` |
| SNS Topic | `ses-complaints` |
| IAM Role | `ses-daily-report-lambda-role` |
| Lambda Function | `ses-daily-report` |
| EventBridge Rule | `ses-daily-report-trigger` |

---

## Re-running the Script

The script is fully idempotent. Re-running it will:

- Return existing SQS queue URLs and SNS topic ARNs without error
- Overwrite SQS access policies and SES notification settings with identical values
- Skip IAM role creation if it already exists
- Update the Lambda function code and environment variables in place
- Overwrite the EventBridge rule and target with identical values

The only meaningful change on a re-run (versus a fresh run) is that any updated Lambda code or configuration changes will be applied.

> **Note:** When updating the Lambda, the script waits for the code update to complete before applying configuration changes, avoiding a `ResourceConflictException` from AWS.

---

## Changing the Report Schedule

The default schedule is **07:00 UTC daily**. To change it, update the `ScheduleExpression` in the EventBridge `put_rule` call:

```python
ScheduleExpression="cron(0 7 * * ? *)"  # 07:00 UTC daily
```

Examples:
```
cron(0 12 * * ? *)   # 12:00 UTC (noon) daily
cron(0 6 * * ? *)    # 06:00 UTC daily
cron(0 7 ? * MON *)  # 07:00 UTC Mondays only
```

---

## Triggering the Report Manually

To send a report on demand from CloudShell:

```bash
aws lambda invoke \
  --function-name ses-daily-report \
  --region us-east-2 \
  --payload '{}' \
  /tmp/out.json && cat /tmp/out.json
```
---

## Notes
- The lambda logs contain additional bounce email details and more SES email headers which may be useful to you, see more below.
- Bounce and complaint notifications are generated by SES regardless of whether emails were sent via the SES API or the SES SMTP interface, and will appear in this report.
- Hard bounces should be removed from your mailing lists immediately to protect your SES sender reputation.

---

## Viewing Lambda Logs

```bash
aws logs tail /aws/lambda/ses-daily-report \
  --region us-east-2 \
  --since 1h \
  --format short
```
## Example Lambda Log entry
```
2026-05-01T07:00:36 [INFO]      2026-05-01T07:00:36.935Z        0fe60246-77ce-46de-8a3a-2b0695c01234    RAW NOTIFICATION: {"notificationType": "Bounce", "bounce": {"feedbackId": "011a019ddee01234-35201234-0012-4012-b012-a012adb01234-000000", "bounceType": "Permanent", "bounceSubType": "General", "bouncedRecipients": [{"emailAddress": "partner@example.com", "action": "failed", "status": "5.4.1", "diagnosticCode": "SMTP; 550 5.4.1 Recipient address rejected: Access denied. For more information see https://aka.ms/EXOSmtpErrors [MTA.prod.example.com 2026-04-30T14:55:40.863Z 012346BEE8C01234]"}], "timestamp": "2026-04-30T15:05:57.000Z", "reportingMTA": "dns; mx0f-0123456.destination.host.example.com"}, "mail": {"timestamp": "2026-04-30T14:55:36.782Z", "source": "myapp@example.com", "sourceArn": "arn:aws:ses:us-east-2:012345678901:identity/example.com", "sourceIp": "199.255.192.111", "callerIdentity": "smtp-user", "sendingAccountId": "012345678901", "messageId": "011a019ddee01234-7c301234-a0123-40123-90123-362470123456-000000", "destination": ["destinationuser@example.com"], "headersTruncated": false, "headers": [{"name": "Received", "value": "from myapp.example.com (ec2-18-252-252-111.us-east-2.compute.amazonaws.com [199.255.192.111]) by email-smtp.amazonaws.com with SMTP (SimpleEmailService-d-XEF4O0FFF) id 0zjZZZeKEzXP01234567 for partner@example.com; Thu, 30 Apr 2026 14:55:36 +0000 (UTC)"}, {"name": "Date", "value": "Thu, 30 Apr 2026 10:55:36 -0400"}, {"name": "To", "value": "\"partner@example.com\" <partner@example.com>"}, {"name": "From", "value": "My App <myapp@example.com>"}, {"name": "Reply-To", "value": "My HelpDesk <MyHelpdesk@example.com>"}, {"name": "Subject", "value": "Example Subject Line"}, {"name": "Message-ID", "value": "<zO5fPhnfjescTxXkXAwLZSrlXrorESnmxgSd29OXo@myapp.example.com>"}, {"name": "MIME-Version", "value": "1.0"}, {"name": "Content-Type", "value": "text/html; charset=UTF-8"}], "commonHeaders": {"from": ["My App <myapp@example.com>"], "replyTo": ["My HelpDesk <MyHelpdesk@example.com>"], "date": "Thu, 30 Apr 2026 10:55:36 -0400", "to": ["\"partner@example.com\" <partner@example.com>"], "messageId": "<zO5fPhcdjwikTxXkWBzLZSrlXrorESnm012345678@myapp.example.com>", "subject": "Example Subject Line"}}}
```
---
# Available SES Bounce Notification Fields

Reference of all fields available in the SES bounce notification payload, based on a live notification sample. These fields can be extracted and included in the daily report.

---

## `bounce` Object

| Field | Example Value |
|---|---|
| `bounceType` | `Permanent` |
| `bounceSubType` | `General` |
| `timestamp` | `2026-04-30T15:05:57.000Z` |
| `feedbackId` | `011a019d...` |
| `reportingMTA` | `dns; mx0f-0123456.destination.host.example.com` |

---

## `bounce.bouncedRecipients[]`

| Field | Example Value |
|---|---|
| `emailAddress` | `partner@example.com` |
| `action` | `failed` |
| `status` | `5.4.1` |
| `diagnosticCode` | `SMTP; 550 5.4.1 Recipient address rejected...` |

---

## `mail` Object

| Field | Example Value |
|---|---|
| `timestamp` | `2026-04-30T14:55:36.782Z` *(send time — distinct from bounce timestamp)* |
| `source` | `myapp@example.com` |
| `sourceArn` | `arn:aws:ses:us-east-2:012345678901:identity/example.com` |
| `sourceIp` | `199.255.192.111` |
| `callerIdentity` | `smtp-user` |
| `sendingAccountId` | `012345678901` |
| `messageId` | `011a019d...` |
| `destination` | `["destinationuser@example.com"]` |

---

## `mail.commonHeaders`

| Field | Example Value |
|---|---|
| `from` | `My App <myapp@example.com>` |
| `to` | `"partner@example.com" <partner@example.com>` |
| `subject` | `Example Subject Line` |
| `date` | `Thu, 30 Apr 2026 10:55:36 -0400` |
| `replyTo` | `My HelpDesk <helpdesk@example.com>` |
| `messageId` | `<zO5fPh...@myapp.example.com>` |

---

## Fields Currently Used in the Report

| Field | Source |
|---|---|
| Sender | `mail.commonHeaders.from[0]` |
| Recipient | `bounce.bouncedRecipients[].emailAddress` |
| Subject | `mail.commonHeaders.subject` |
| Subtype | `bounce.bounceSubType` |
| Diagnostic Code | `bounce.bouncedRecipients[].diagnosticCode` |
| Timestamp | `bounce.timestamp` |

---

## Additional Fields Worth Considering

| Field | Source | Notes |
|---|---|---|
| `status` | `bounce.bouncedRecipients[].status` | SMTP status code (e.g. `5.4.1`, `5.1.1`) — quick failure categorisation without reading the full diagnostic |
| `reportingMTA` | `bounce.reportingMTA` | Which mail server reported the bounce — useful if sending through multiple MTAs |
| `mail.timestamp` | `mail.timestamp` | Original send time vs `bounce.timestamp` which is when the bounce was reported |
| `callerIdentity` | `mail.callerIdentity` | Shows which IAM identity or SMTP user submitted the send — useful in multi-app environments |
| `sourceIp` | `mail.sourceIp` | IP address that submitted the message to SES — useful in multi-server environments |
| `replyTo` | `mail.commonHeaders.replyTo[0]` | Reply-To address if set by the sending application |
| `action` | `bounce.bouncedRecipients[].action` | Always `failed` for bounces — included for completeness |

## License

MIT
