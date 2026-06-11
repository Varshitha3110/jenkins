# AI-Powered Candidate Verification Platform
## Complete Setup & Operations Guide

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 – Meta Developer App & WhatsApp Setup](#step-1--meta-developer-app--whatsapp-setup)
4. [Step 2 – AWS Account Preparation](#step-2--aws-account-preparation)
5. [Step 3 – Enable Amazon Bedrock Claude Access](#step-3--enable-amazon-bedrock-claude-access)
6. [Step 4 – Deploy Infrastructure](#step-4--deploy-infrastructure)
7. [Step 5 – Register Webhook with Meta](#step-5--register-webhook-with-meta)
8. [Step 6 – Load Candidates & Run Campaign](#step-6--load-candidates--run-campaign)
9. [DynamoDB Schema Reference](#dynamodb-schema-reference)
10. [Testing](#testing)
11. [Troubleshooting](#troubleshooting)
12. [Cost Estimate](#cost-estimate)

---

## Architecture Overview

```
CSV/DynamoDB → [Lambda: Campaign Sender] → [WhatsApp Cloud API] → Candidate
                                                                       ↓ (reply)
[DynamoDB: Candidates + History + Audit] ← [Lambda: Response Processor] ← [Lambda: Webhook Receiver] ← [API Gateway]
                              ↑ AI extraction
                    [Amazon Bedrock – Claude 3 Sonnet]
```

**Flow:**
1. Campaign Sender loads PENDING candidates and sends a WhatsApp message asking for profile updates.
2. Candidate replies with updated details (name, email, phone, skills, etc.).
3. API Gateway receives the WhatsApp webhook POST and invokes Webhook Receiver.
4. Webhook Receiver parses the payload and asynchronously invokes Response Processor.
5. Response Processor sends the raw reply to Amazon Bedrock (Claude 3 Sonnet) for AI extraction.
6. Extracted fields are compared with existing record; changed fields are updated in DynamoDB.
7. Full audit trail is written to VerificationHistory and AuditLog tables.
8. Candidate receives a WhatsApp confirmation message.

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| AWS CLI | v2.x | `aws --version` |
| Node.js | 20.x | `node --version` |
| npm | 10.x | `npm --version` |
| zip | any | for Lambda packaging |
| AWS Account | — | With billing enabled |
| Meta Developer Account | — | Free at developers.facebook.com |

---

## Step 1 – Meta Developer App & WhatsApp Setup

> ⚠️ **MANUAL STEP** – Must be done in your browser. Takes ~20 minutes.

### 1.1 Create a Meta App
1. Go to [https://developers.facebook.com/apps](https://developers.facebook.com/apps)
2. Click **Create App** → choose **Business** type
3. Fill in app name (e.g., `CandidateVerification`) and contact email
4. Click **Create App**

NOTE : https://developers.facebook.com/documentation/business-messaging/whatsapp/get-started

### 1.2 Add WhatsApp Product
1. In your app dashboard, scroll to **Add a Product**
2. Find **WhatsApp** → click **Set up**
3. You'll be taken to the WhatsApp Getting Started page

### 1.3 Get Your Credentials
Under **WhatsApp > API Setup**:

| Value | Where to Find | Used In |
|-------|--------------|---------|
| **Phone Number ID** | "From" phone number section | `WHATSAPP_PHONE_NUMBER_ID` env var |
| **Temporary Access Token** | "Temporary access token" box | `WHATSAPP_TOKEN` env var (use System User token for prod) |

> 💡 **For production**: Create a **System User** under Business Settings and generate a permanent token with `whatsapp_business_messaging` permission.

### 1.4 Add a Test Phone Number
1. Under **WhatsApp > API Setup**, click **Add phone number** (test recipient)
2. Add your own mobile number and verify with OTP
3. You can now receive WhatsApp messages from the test number

### 1.5 Note Down Your Verify Token
Choose any string (e.g., `my_verify_token_abc123`) — you'll need it in Step 4 and Step 5.

---

## Step 2 – AWS Account Preparation

### 2.1 Create an S3 Bucket for Lambda Deployments
```bash
# Replace with your preferred bucket name and region
aws s3 mb s3://my-candidate-verification-deploy --region ap-south-1
```

### 2.2 Ensure IAM Permissions
The user/role running the deployment needs:
- `cloudformation:*`
- `lambda:*`
- `dynamodb:*`
- `apigateway:*`
- `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PassRole`
- `s3:PutObject`, `s3:GetObject`

---

## Step 3 – Enable Amazon Bedrock Claude Access

> ⚠️ **MANUAL STEP** – Bedrock model access must be requested manually.

1. Go to [AWS Console → Amazon Bedrock](https://console.aws.amazon.com/bedrock)
2. Select region **us-east-1** (N. Virginia) — Claude models are most widely available here
3. In left sidebar: **Model access** → **Manage model access**
4. Find **Anthropic → Claude 3 Sonnet** → check the box → **Request access**
5. Wait for approval (usually instant for Claude Sonnet)

> 💡 The Lambda `BEDROCK_REGION` env var defaults to `us-east-1`. Change it if you request access in a different region.

---

## Step 4 – Deploy Infrastructure

### 4.1 Install Lambda Dependencies & Deploy
```bash
cd candidate-verification-platform

# Make scripts executable
chmod +x scripts/deploy.sh

# Run deployment
./scripts/deploy.sh my-candidate-verification-deploy ap-south-1 dev
```

The script will:
- `npm install` each Lambda's dependencies
- Zip and upload each Lambda to S3
- Deploy the CloudFormation stack with all resources
- Print the **WebhookURL** you need for Step 5

### 4.2 Note the Webhook URL
After deployment, copy the `WebhookURL` output — it looks like:
```
https://abc123xyz.execute-api.ap-south-1.amazonaws.com/dev/webhook
```

---

## Step 5 – Register Webhook with Meta

> ⚠️ **MANUAL STEP**

1. In Meta Developer Console → Your App → **WhatsApp > Configuration**
2. Under **Webhook**, click **Edit**
3. Fill in:
   - **Callback URL**: the WebhookURL from Step 4.2
   - **Verify Token**: the same string you chose in Step 1.5
4. Click **Verify and Save**
   - Meta will call your API Gateway GET endpoint with the verify token
   - Your Lambda returns the challenge — verification succeeds ✅
5. Under **Webhook Fields**, enable **messages**
6. Click **Save**

---

## Step 6 – Load Candidates & Run Campaign

### 6.1 Option A: Load from CSV (one-time seed)
```bash
# Install local deps first
npm install

# Seed sample data
CANDIDATES_TABLE=Candidates-dev \
AWS_REGION=ap-south-1 \
node scripts/seed-candidates.js sample-data/sample-candidates.csv
```

### 6.2 Option B: Load CSV via Lambda (S3 trigger)
Upload your CSV to an S3 bucket and invoke the Campaign Sender directly:
```bash
# Upload CSV
aws s3 cp your-candidates.csv s3://my-bucket/candidates/input.csv

# Trigger Campaign Sender with S3 event
aws lambda invoke \
  --function-name CampaignSender-dev \
  --payload '{"source":"S3","bucket":"my-bucket","key":"candidates/input.csv"}' \
  --region ap-south-1 \
  response.json

cat response.json
```

### 6.3 Run a Campaign (send WhatsApp messages to all PENDING)
```bash
aws lambda invoke \
  --function-name CampaignSender-dev \
  --payload '{}' \
  --region ap-south-1 \
  response.json

cat response.json
```

---

## DynamoDB Schema Reference

### Candidates Table
| Attribute | Type | Description |
|-----------|------|-------------|
| `candidate_id` | String (PK) | Unique ID, e.g. `CAND_001` |
| `name` | String | Full name |
| `email` | String | Email address |
| `phone` | String | International format, e.g. `+919876543210` |
| `skills` | String | Comma-separated skills |
| `experience` | String | Years/description |
| `current_employer` | String | Company name |
| `verification_status` | String | `PENDING` / `VERIFIED` / `OPT_OUT` |
| `created_at` | String | ISO timestamp |
| `updated_at` | String | ISO timestamp |

### VerificationHistory Table
| Attribute | Type | Description |
|-----------|------|-------------|
| `history_id` | String (PK) | Unique log entry ID |
| `candidate_id` | String (GSI) | Reference to candidate |
| `field_name` | String | Which field changed |
| `old_value` | String | Previous value |
| `new_value` | String | Updated value |
| `confidence_score` | String | AI confidence (0.0–1.0) |
| `timestamp` | String | ISO timestamp |

### AuditLog Table
| Attribute | Type | Description |
|-----------|------|-------------|
| `log_id` | String (PK) | Unique log entry ID |
| `candidate_id` | String (GSI) | Reference to candidate |
| `action_performed` | String | e.g. `PROFILE_UPDATED`, `OPT_OUT`, `VERIFICATION_MESSAGE_SENT` |
| `old_value` | String | JSON of old values |
| `new_value` | String | JSON of new values |
| `updated_by` | String | `AI`, `SYSTEM`, or `HUMAN` |
| `timestamp` | String | ISO timestamp |

---

## Testing

### Local Test (no AWS needed)
```bash
node scripts/local-test.js
```
This simulates all 4 steps with mock data and prints the final DynamoDB state, history, and audit log.

### Manual Webhook Test
```bash
# Simulate a WhatsApp webhook POST
curl -X POST https://YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "object": "whatsapp_business_account",
    "entry": [{
      "changes": [{
        "value": {
          "messages": [{
            "from": "919876543210",
            "id": "msg_test_001",
            "timestamp": "1700000000",
            "type": "text",
            "text": { "body": "Name: Test User\nEmail: test@new.com\nSkills: Python, AWS" }
          }],
          "metadata": { "display_phone_number": "1234567890" }
        }
      }]
    }]
  }'
```

### View DynamoDB Records
```bash
# All candidates
aws dynamodb scan --table-name Candidates-dev --region ap-south-1

# Audit log for a specific candidate
aws dynamodb query \
  --table-name AuditLog-dev \
  --index-name candidate-index \
  --key-condition-expression "candidate_id = :id" \
  --expression-attribute-values '{":id":{"S":"CAND_001"}}' \
  --region ap-south-1
```

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Webhook verification fails | Wrong verify token | Ensure `WHATSAPP_VERIFY_TOKEN` Lambda env var matches Meta console |
| Messages not sending | Invalid token or phone ID | Check `WHATSAPP_TOKEN` and `WHATSAPP_PHONE_NUMBER_ID` |
| Bedrock returns error | Model access not approved | Go to Bedrock console → enable Claude 3 Sonnet in correct region |
| Candidate not found | Phone number mismatch | Ensure phone in DynamoDB matches WhatsApp sender format |
| Lambda timeout | Response Processor too slow | Increase timeout to 120s (already set in CloudFormation) |
| CloudFormation deploy fails | Missing IAM permissions | Ensure deploying IAM user has cloudformation:* and iam:* permissions |

### Check Lambda Logs
```bash
# Campaign Sender logs
aws logs tail /aws/lambda/CampaignSender-dev --follow --region ap-south-1

# Response Processor logs
aws logs tail /aws/lambda/ResponseProcessor-dev --follow --region ap-south-1
```

---

## Cost Estimate (POC, low volume)

| Service | Usage | Est. Monthly Cost |
|---------|-------|------------------|
| AWS Lambda | 10,000 invocations, 256MB | ~$0.50 |
| Amazon DynamoDB | On-demand, <1M ops | ~$1.00 |
| API Gateway | 10,000 requests | ~$0.04 |
| Amazon Bedrock (Claude 3 Sonnet) | 1,000 requests, ~500 tokens each | ~$1.50 |
| CloudWatch Logs | Minimal | ~$0.10 |
| **Total** | | **~$3–5/month** |

> WhatsApp Cloud API: Free for the first 1,000 user-initiated conversations/month. Business-initiated conversations are charged per Meta's pricing.

---

## Security Recommendations for Production

1. **Store secrets in AWS Secrets Manager** instead of Lambda env vars.
2. **Add signature validation** for WhatsApp webhooks (verify `X-Hub-Signature-256` header).
3. **Enable DynamoDB Point-in-Time Recovery** (PITR) for all tables.
4. **Restrict IAM roles** to specific table ARNs (already done in CloudFormation).
5. **Enable AWS WAF** on API Gateway for rate limiting.
6. **Use a permanent System User token** from Meta Business Manager instead of the temporary token.
