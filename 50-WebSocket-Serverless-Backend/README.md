# Serverless WebSocket Fan-Out: API Gateway + DynamoDB Connection Registry That Never Pushes to Dead Connections

A production-ready serverless push backend—four enumerated WebSocket route handlers, a self-healing connection registry, and SQS-sharded fan-out to 10,000 clients—so you ship live features instead of running socket servers.

## The Problem

Real-time features get faked with polling or botched with half-built WebSocket stacks. Polling is brutal: 10,000 clients hitting an HTTP API every 30 seconds is 864M requests/month—roughly $864 in API Gateway HTTP API fees alone before Lambda and database reads, just to learn "nothing changed." So teams reach for API Gateway WebSocket APIs, then hit the part nobody documents well: **connection lifecycle**. API Gateway enforces a 2-hour maximum connection duration and a 10-minute idle timeout, so at 10,000 concurrent clients you see at least 120,000 churn events per day. `$disconnect` is best-effort—it is not guaranteed to fire. If even 5% of disconnects are missed, you orphan 6,000 dead `connectionId` rows per day. The first broadcast after that throws `GoneException` (HTTP 410), a naive `Promise.all` loop rejects mid-flight, and half your real users get nothing while the registry keeps bloating. This prompt builds the whole thing correctly the first time: enumerated `$connect` / `$disconnect` / `$default` / `sendmessage` handlers, delete-on-410 self-healing with a DynamoDB TTL backstop, SQS-sharded fan-out that survives any single dead client, and an honest cost model computed for 10,000 connections.

## Who This Is For

Startup and platform teams adding live dashboards, chat, presence indicators, notifications, auctions, or collaborative editing to an existing serverless stack—who refuse to run socket.io on Fargate with sticky sessions and a 3 AM pager. Comfortable with basic Lambda and DynamoDB; no WebSocket-specific experience required.

## How to Use

1. Copy the System Prompt below into a Kiro CLI session, or save it as `websocket-backend.md` and run `claude "Build the stack specified in @websocket-backend.md"` in Claude Code. Works with any AI coding assistant that can run shell commands.
2. Answer the assistant's kickoff questions: target region, stage name, channel model (single broadcast channel vs. per-room), and expected peak concurrent connections.
3. Let the assistant run its verify-reality checks (`aws sts get-caller-identity`, `aws apigatewayv2 get-apis`) before it writes a single line. Review the ASCII architecture diagram and the cost table it produces, then approve code generation.
4. Deploy: `sam build && sam deploy --guided`. Target: live in under 15 minutes.
5. Run the generated verification script: two `wscat` sessions exchanging a message, then the forced-GoneException test that proves the registry self-heals.

**Prerequisites**

- **Required Access:** AWS account with permissions to create API Gateway v2 APIs, Lambda functions, DynamoDB tables, SQS queues, IAM roles, and CloudWatch alarms (PowerUserAccess plus `iam:CreateRole`/`iam:PassRole`, or AdministratorAccess in a sandbox account).
- **Recommended Background:** Basic Lambda and DynamoDB; familiarity with the browser WebSocket API helps.
- **Tools Required:** AWS CLI v2 (configured), AWS SAM CLI, Node.js 20.x, `wscat` (`npm i -g wscat`), and the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) for limit/price verification.

**Key Parameters:** Heartbeat interval (5 min), client reconnect backoff (1s→30s + jitter), TTL horizon (connect + 3h), fan-out wave size (100 parallel posts), SQS shard size (1,000 connection IDs), worker memory/timeout (512 MB / 30s), payload cap (28 KB), log retention (30d), stage (prod), table mode (on-demand).

**Troubleshooting:** If your `wscat` session drops after exactly 10 minutes, that is expected—API Gateway closes idle WebSocket connections at the 10-minute mark. `wscat` sends nothing unless you type; the generated browser client avoids this by pinging every 5 minutes.

## System Prompt

```
# Serverless Real-Time WebSocket Backend — Architecture & Implementation Request

## Project Overview
Build a production-ready, fully serverless real-time push backend on AWS: an Amazon API Gateway WebSocket API in front of AWS Lambda route handlers, a DynamoDB connection registry, and SQS-sharded fan-out. Target: 10,000 concurrent browser clients, p95 broadcast delivery under 1 second, zero servers to manage.

## Verify Reality First (before generating anything)
1. Run `aws sts get-caller-identity` and `aws configure get region`; state the account and region you are building for.
2. Run `aws apigatewayv2 get-apis` and confirm no API named `realtime-ws` already exists. If it does, stop and ask.
3. Confirm Node.js 20.x Lambda runtime availability and current WebSocket API limits in this region with the AWS Documentation MCP server (aws_read_documentation). Never state a quota, limit, or price from memory when you can verify it.

## Detailed Requirements

### 1. Routes and Handlers (all four, enumerated)
- WebSocket API with route selection expression `$request.body.action`.
- `$connect`: REQUEST-type Lambda authorizer validates a short-lived JWT passed as `?token=` (browsers cannot set custom WebSocket headers); on allow, PutItem the connection record.
- `$disconnect`: DeleteItem the connection record. Treat as best-effort — it is not guaranteed to fire.
- `$default`: return a structured JSON error for unknown actions; never silently drop.
- `sendmessage` (custom route): validate payload <= 28 KB, then write one SQS message per 1,000 target connection IDs.

### 2. Connection Registry (DynamoDB)
- Table `ws-connections`, on-demand mode, partition key `connectionId` (S); GSI `byChannel` keyed on `channelId` for targeted fan-out.
- Attributes: connectionId, channelId, userId, connectedAt, expiresAt. Keep items under 200 bytes.
- Enable TTL on `expiresAt` = connect time + 3 hours (covers the 2-hour hard maximum connection duration plus drift). TTL is the BACKSTOP only — TTL deletion can lag expiry by hours, so it must never be the primary cleanup path.

### 3. Fan-Out and Error Management
- Broadcast worker: Lambda (512 MB, 30 s timeout, SQS event source) posts via the management endpoint `https://{api-id}.execute-api.{region}.amazonaws.com/{stage}` using @aws-sdk/client-apigatewaymanagementapi, in waves of 100 parallel PostToConnection calls.
- On GoneException (410): DeleteItem the stale record and continue. This is the PRIMARY cleanup path. One dead client must never fail a broadcast.
- On ThrottlingException or LimitExceededException: exponential backoff, max 3 retries, then dead-letter queue.
- IAM: the worker gets only `execute-api:ManageConnections` scoped to `arn:aws:execute-api:{region}:{account}:{api-id}/{stage}/POST/@connections/*`. Every function gets its own least-privilege role.

### 4. Honest Limits (encode in client code and docs)
- 10-minute idle timeout: generated browser client sends a heartbeat ping every 5 minutes.
- 2-hour maximum connection duration: client auto-reconnects with exponential backoff and jitter, then re-subscribes to its channel.
- 128 KB maximum message size, 32 KB maximum frame size; messages are METERED in 32 KB increments, so reject oversize payloads at `sendmessage` instead of paying 4x.
- Document the default quota of 500 new connections per second per account per region and how to raise it via Service Quotas.

### 5. Cost Targets
- Produce a line-item cost table for 10,000 always-connected clients receiving 1 broadcast per minute for 30 days in us-east-1: 432M connection minutes ($108.00), 432M messages ($432.00), plus Lambda, DynamoDB, and SQS lines, with a per-connection-month figure. Flag any design choice pushing the total past $600/month.
- The dev stage must fit inside the 12-month free tier (1M messages, 750K connection minutes).

### 6. Observability
- Stage access logging to CloudWatch Logs (30-day retention). Alarms: IntegrationError rate > 1% over 5 minutes, broadcast DLQ depth > 0, and anomaly detection on ConnectCount.

## Deliverables Requested
1. ASCII architecture diagram.
2. Complete IaC: AWS SAM `template.yaml` using AWS::ApiGatewayV2::Api, ::Route, ::Integration, ::Stage, and ::Authorizer resources.
3. Six Node.js 20.x functions (AWS SDK v3 only): authorizer, connect, disconnect, default, sendmessage, broadcast-worker.
4. Browser client snippet with heartbeat + reconnect, plus a wscat smoke-test script.
5. The Section 5 cost table.
6. A Verification section with exact commands and expected output: two wscat sessions exchanging a message end-to-end, a forced-GoneException test proving the registry deletes the stale row, and the CloudWatch metrics to confirm.

Output everything without preamble. Prefix every assumption with ASSUMPTION:. The stack must deploy with `sam build && sam deploy --guided` in under 15 minutes and pass the Verification section on the first run.
```

## What You Get

1. `template.yaml` — full SAM/CloudFormation stack: WebSocket API, 4 routes + authorizer, 6 functions, `ws-connections` table with TTL + `byChannel` GSI, SQS broadcast queue + DLQ, scoped IAM roles, CloudWatch alarms.
2. `src/authorizer.mjs` — REQUEST authorizer validating the query-string JWT.
3. `src/connect.mjs`, `src/disconnect.mjs`, `src/default.mjs`, `src/sendmessage.mjs` — the enumerated route handlers.
4. `src/broadcast-worker.mjs` — SQS-driven fan-out with delete-on-GoneException and backoff on throttles.
5. `client/realtime-client.js` — browser client with 5-minute heartbeat and jittered auto-reconnect.
6. `scripts/smoke-test.sh` — wscat connect/send/receive plus the forced-GoneException self-healing test.
7. A line-item cost table for the 10k-connection scenario with a per-connection-month figure.
8. A README runbook: deploy, verify, raise the 500 conn/sec quota, roll back.

## Example Output

```
VERIFICATION
$ wscat -c "wss://a1b2c3d4e5.execute-api.us-east-1.amazonaws.com/prod?token=eyJh..."
Connected (press CTRL+C to quit)
> {"action":"sendmessage","channelId":"demo","data":"hello"}
  [session 2] < {"channelId":"demo","data":"hello"}   (delivered in 410 ms)

Stale-connection test: killed session 2 without $disconnect, broadcast again →
broadcast-worker log: GoneException for K2xQzcfBoAMCJgw= → DeleteItem ws-connections ✓
Re-broadcast: 1/1 delivered, 0 errors. Registry self-healed.

COST — 10,000 connections, 1 broadcast/min, 30 days (us-east-1)
Connection minutes  432M × $0.25/M  = $108.00
Messages            432M × $1.00/M  = $432.00
Lambda + DynamoDB + SQS             = $ 16.43
TOTAL ≈ $556/mo → $0.056 per connected client/month
```

## AWS Services Used

Amazon API Gateway (WebSocket APIs), AWS Lambda, Amazon DynamoDB (TTL + GSI), Amazon SQS, Amazon CloudWatch, AWS IAM, AWS SAM, AWS Service Quotas.

## Well-Architected Alignment

- **Operational Excellence:** stage access logs, named alarms (IntegrationError > 1%, DLQ depth > 0, ConnectCount anomaly), a verification script as the definition of done, runbook included.
- **Security:** deny-by-default JWT authorizer on `$connect`; per-function least-privilege roles; `execute-api:ManageConnections` scoped to a single API/stage ARN; no long-lived credentials anywhere.
- **Reliability:** GoneException delete-on-410 as primary cleanup with DynamoDB TTL as backstop; SQS sharding keeps every worker under its 30 s timeout; retries with backoff and a DLQ so one dead client never fails a broadcast.
- **Performance Efficiency:** on-demand DynamoDB absorbs connection churn spikes; `byChannel` GSI targets fan-out instead of table scans; 100-wide parallel post waves hit p95 < 1 s to 10k clients.
- **Cost Optimization:** full cost table as a required deliverable; 28 KB payload cap avoids 32 KB-increment metering surprises; dev stage sized to the free tier.

## Cost Notes

Heavy scenario — 10,000 always-on connections, 1 broadcast/minute, 30 days, us-east-1: connection minutes 10,000 × 43,200 min = 432M × $0.25/M = **$108.00**; messages 432M × $1.00/M = **$432.00**; Lambda fan-out (~648,000 GB-s) ≈ **$10.90**; DynamoDB on-demand (~7.2M writes from 2-hour churn + ~5.4M GSI reads) ≈ **$5.18**; SQS ≈ **$0.35**. **Total ≈ $556/month, i.e. $0.056 per connected client per month**, and message fees for a single broadcast to all 10k clients are exactly **$0.01**. Light scenario — 500 concurrent connections, 50k messages/day: ≈ $5.40 connection minutes + $1.50 messages = **under $10/month**. Dev stays at ~$0 inside the 12-month free tier (1M messages + 750K connection minutes). Watch the metering rule: messages bill in 32 KB increments, so a 100 KB payload costs 4 message units—the prompt caps payloads at 28 KB for exactly this reason.

## Troubleshooting

1. **Clients drop after exactly 10 minutes.** Cause: API Gateway's 10-minute idle timeout and no traffic on the socket. Fix: the browser client's 5-minute heartbeat ping; for wscat tests, send any frame periodically.
2. **Broadcasts throw GoneException (410).** Cause: stale connectionIds from missed `$disconnect` events. Fix: expected and handled—the worker deletes the row and continues; confirm the TTL backstop is enabled on `expiresAt`.
3. **PostToConnection returns 403 Forbidden.** Cause: missing `execute-api:ManageConnections` on the worker role, or calling the `wss://` URL instead of the `https://` management endpoint. Fix: scope the policy to `POST/@connections/*` on your API/stage ARN and use the HTTPS endpoint with the ApiGatewayManagementApi client.
4. **Everyone disconnects at the 2-hour mark.** Cause: the hard maximum connection duration—no setting changes this. Fix: jittered auto-reconnect plus re-subscribe in the client; size the TTL horizon at connect + 3 hours.
5. **Connection storms get throttled at launch.** Cause: the default quota of 500 new connections/second per account per region. Fix: request an increase in Service Quotas before launch events, and keep reconnect jitter so 10k clients never reconnect in the same second.

Deployable in under 15 minutes. The registry heals itself; the bill stays computed, not discovered.
