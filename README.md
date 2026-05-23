# Practical AI Agent Integration with Amazon Connect via MCP

## 1. Overview

This document describes a minimal, reusable architecture that allows any text-based AI agent (Claude Desktop, Cursor, custom apps) to perform practical actions in AWS through standard tools. The primary example is **placing an outbound reminder call via Amazon Connect**.

By using **Amazon Bedrock AgentCore Gateway**, we expose AWS actions as standard Model Context Protocol (MCP) tools. This eliminates the need to build and host custom MCP servers.

### Key Principles
- **Simplicity over streaming:** We use API commands, not real-time bidirectional audio.
- **No self-hosted MCP infrastructure:** AgentCore Gateway handles protocol translation, auth, and routing.
- **Agent-agnostic:** Any MCP-compatible client can discover and invoke these tools.

---

## 2. Architecture

```
[ AI Agent / MCP Client ]
            │
            │ MCP (tools/list, tools/call)
            ▼
[ Amazon Bedrock AgentCore Gateway ]
            │
            │ AWS Lambda Invocation
            ▼
[ AWS Lambda: Connect Actions ]
            │
            ├──────► [ Amazon Connect ]
            │              │
            │              ▼
            │      [ Outbound Call ]
            │              │
            │              ▼
            │      [ Amazon Polly TTS ]
            │
            └──────► [ Amazon DynamoDB ] (Optional: Call logs)
```

### How It Works
1. A user asks their AI agent to "Call Mike and remind him about his appointment."
2. The AI agent discovers the `place_outbound_call` tool via MCP.
3. The agent calls the tool with a phone number and message.
4. **AgentCore Gateway** authenticates the request and invokes an AWS Lambda function.
5. The Lambda function calls the Amazon Connect API to start an outbound contact.
6. Amazon Connect dials the number and uses **Amazon Polly** to speak the dynamic reminder message.

---

## 3. Prerequisites

- An AWS Account.
- An **Amazon Connect Instance** (you only need the Instance ID; the template handles the rest).
- IAM permissions to create Lambda, DynamoDB, SNS, and Connect resources.
- The [AWS AgentCore CLI](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-quick-start.html) installed:
  ```bash
  npm install -g @aws/agentcore
  ```

---

## 4. Step-by-Step Setup

### Step 1: Deploy the CloudFormation Template

The template automatically provisions the backend and can optionally create a Connect Contact Flow and claim a phone number for you.

**Option A: Fully Automated (Recommended for New Projects)**

Provide only your Connect Instance ID. The template will create a simple outbound reminder flow and claim a DID phone number.

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name connect-mcp-agent \
  --parameter-overrides \
    ConnectInstanceId=your-instance-id \
    Environment=dev \
  --capabilities CAPABILITY_NAMED_IAM
```

**Option B: Use Existing Flow and Number**

If you already have an outbound contact flow and a claimed phone number:

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name connect-mcp-agent \
  --parameter-overrides \
    ConnectInstanceId=your-instance-id \
    ConnectContactFlowId=your-flow-id \
    SourcePhoneNumber=+15551234567 \
    Environment=dev \
  --capabilities CAPABILITY_NAMED_IAM
```

**What the template deploys:**

| Resource | Purpose |
|----------|---------|
| **Lambda Function** (`connect-outbound-caller`) | Handles `place_outbound_call`, `send_sms`, and `log` actions. |
| **IAM Role** | Grants least-privilege access to Connect, DynamoDB, SNS, and CloudWatch Logs. |
| **DynamoDB Table** (`connect-mcp-logs`) | Stores call/SMS logs with a 30-day TTL for automatic cleanup. |
| **SNS Topic** | Backend for SMS reminders. |
| **Contact Flow** (optional) | A pre-built outbound flow that speaks `$.Attributes.ReminderMessage` via Polly. |
| **Phone Number** (optional) | A claimed DID associated with the outbound flow. |

After deployment, grab the outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name connect-mcp-agent \
  --query 'Stacks[0].Outputs'
```

You will need the **LambdaArn**, **ContactFlowId**, and **SourcePhoneNumber** for the next step.

### Step 2: Deploy AgentCore Gateway

1. **Initialize the project:**
   ```bash
   agentcore create --name ConnectReminderAgent --defaults
   cd ConnectReminderAgent
   ```

2. **Add a Gateway:**
   ```bash
   agentcore add gateway \
     --name ConnectGateway \
     --authorizer-type NONE \
     --runtimes ConnectReminderAgent
   ```
   *Note: Use `NONE` for local development. For production, use `IAM` or `OAuth`.*

3. **Create the tool definition file** (`tools.json`):
   ```json
   {
     "tools": [
       {
         "name": "place_outbound_call",
         "description": "Places an outbound reminder call via Amazon Connect. The call will use text-to-speech to speak the provided message.",
         "inputSchema": {
           "type": "object",
           "properties": {
             "phone_number": {
               "type": "string",
               "description": "E.164 formatted phone number, e.g. +15551234567"
             },
             "message": {
               "type": "string",
               "description": "The reminder message to speak to the recipient."
             }
           },
           "required": ["phone_number", "message"]
         }
       },
       {
         "name": "send_sms",
         "description": "Sends an SMS reminder via Amazon SNS.",
         "inputSchema": {
           "type": "object",
           "properties": {
             "phone_number": {
               "type": "string",
               "description": "E.164 formatted phone number, e.g. +15551234567"
             },
             "message": {
               "type": "string",
               "description": "The SMS message to send."
             }
           },
           "required": ["phone_number", "message"]
         }
       }
     ]
   }
   ```

4. **Attach the Lambda as a Gateway target:**
   ```bash
   agentcore add gateway-target \
     --name OutboundCallTool \
     --type lambda-function-arn \
     --lambda-arn <LAMBDA_ARN_FROM_STACK_OUTPUTS> \
     --tool-schema-file tools.json \
     --gateway ConnectGateway
   ```

5. **Deploy:**
   ```bash
   agentcore deploy
   ```

6. **Copy the Gateway MCP endpoint URL** from the deployment output. You will need this to configure your AI agent.

### Step 3: Connect Your AI Agent

Configure your MCP client to use the Gateway endpoint. The example below is for **Claude Desktop** (`claude_desktop_config.json`). Adapt the connection method based on the specific MCP SDK or client you are using.

```json
{
  "mcpServers": {
    "connect_reminders": {
      "command": "npx",
      "args": [
        "-y",
        "@aws/agentcore-mcp-client",
        "--gateway-url",
        "https://YOUR_GATEWAY_ID.agentcore.REGION.amazonaws.com/mcp"
      ]
    }
  }
}
```

*Restart your AI agent client so it can fetch the available tools from the Gateway.*

---

## 5. Example Usage

Once connected, the AI agent naturally discovers and uses the tool:

**User:** "Remind Mike at +1-555-123-4567 that his appointment is tomorrow at 9am."

**Agent Action:**
1. Discovers `place_outbound_call` via the MCP `tools/list` handshake.
2. Invokes `place_outbound_call` with:
   - `phone_number`: `+15551234567`
   - `message`: `Hello Mike, this is a reminder that your appointment is tomorrow at 9 AM.`
3. Receives the JSON result:
   ```json
   {
     "contact_id": "abc-123-def-456",
     "status": "dialing",
     "message": "Reminder call initiated to +15551234567"
   }
   ```
4. Reports to the user: "Calling Mike now with the reminder."

**Result:** Amazon Connect dials the number, and the contact flow speaks the message using Amazon Polly.

---

## 6. Extending the Architecture

You can add more practical tools by creating new Lambda functions and registering them as additional targets in the same Gateway.

| Tool | Lambda Action | AWS Service |
|------|---------------|-------------|
| `send_sms_reminder` | `sns.publish(PhoneNumber=..., Message=...)` | Amazon SNS |
| `log_reminder` | `dynamodb.put_item(...)` | Amazon DynamoDB |
| `email_reminder` | `ses.send_email(...)` | Amazon SES |
| `schedule_future_call` | `eventbridge.put_rule` + `connect.start_outbound_contact` | Amazon EventBridge + Connect |

Because AgentCore Gateway aggregates all targets into a single MCP endpoint, your AI agent sees one unified toolset.

---

## 7. Security & Operations

### Authentication
- **Development:** `NONE` authorizer allows rapid testing.
- **Production:** Switch to `IAM` (SigV4) or `OAuth` (JWT) before deploying to production. Gateway manages both inbound and outbound credential exchange.

### Input Validation
- The Lambda validates that `phone_number` is in strict **E.164 format** (`+` followed by country code and number) to prevent dial errors.
- Add a message length check to avoid excessive Polly usage or TTS costs.

### Rate Limiting & Cost Control
- Set **Lambda reserved concurrency** to prevent accidental runaway calling.
- Use Amazon Connect **queue-based routing** if you need to throttle outbound attempts.
- Enable **CloudTrail** for AgentCore Gateway and **CloudWatch Logs** for Lambda to audit every tool invocation.

### Data Retention
- Call logs in DynamoDB auto-expire after 30 days via TTL.

---

## 8. Cost Estimate (US East)

| Service | Cost Driver | Approximate Cost |
|---------|-------------|------------------|
| **Amazon Connect** | Outbound call time | ~$0.025 / minute |
| **Amazon Polly** | TTS characters spoken | ~$4.00 per 1M characters |
| **AgentCore Gateway** | Per-request pricing | See [AWS Pricing](https://aws.amazon.com/bedrock/pricing/) |
| **AWS Lambda** | Requests & compute | Usually within free tier for low volume |
| **DynamoDB** | On-demand writes | Negligible for low volume |
| **SNS SMS** | Per-message | ~$0.0075 / SMS (US) |

---

## 9. Summary

This architecture provides a **practical, low-complexity bridge** between text-based AI agents and Amazon Connect. By leveraging **Amazon Bedrock AgentCore Gateway** as the managed MCP layer, you avoid building custom protocol adapters. The result is a secure, extensible system where an AI agent can place phone calls, send messages, and interact with AWS services using natural language.
