# Headless Markets Workers

Cloudflare Workers for [Headless Markets](https://github.com/dutchiono/headless-markets) - automated event indexing, graduation monitoring, and notification services.

## Overview

This repository contains three Cloudflare Workers that handle background tasks for the Headless Markets platform:

1. **Event Indexer**: Syncs on-chain events to Vendure database
2. **Graduation Monitor**: Checks bonding curves for graduation threshold
3. **Notification Service**: Sends webhooks for important events

## Workers

### 1. Event Indexer

**Purpose**: Polls Base L2 RPC for contract events and syncs them to Vendure.

**Schedule**: Every 1 minute (cron)

**Events Monitored**:
- `QuorumCreated`: New quorum formation on-chain
- `QuorumVoted`: Agent votes on quorum
- `QuorumExecuted`: Quorum reaches unanimous vote
- `TokenLaunched`: New token deployed via bonding curve
- `TokenGraduated`: Token graduates to Uniswap V2

**Flow**:
```
1. Fetch latest block from Base RPC
2. Get last synced block from KV storage
3. Query events from (lastSynced + 1) to latest
4. Parse events and extract relevant data
5. Call Vendure API to update database
6. Update last synced block in KV
```

**Environment Variables**:
- `BASE_RPC_URL`: Base L2 RPC endpoint
- `VENDURE_API_URL`: Vendure GraphQL endpoint
- `VENDURE_API_TOKEN`: Admin API token
- `QUORUM_MANAGER_ADDRESS`: QuorumManager contract address
- `BONDING_CURVE_ADDRESS`: BondingCurveFactory contract address
- `GRADUATOR_ADDRESS`: TokenGraduator contract address

### 2. Graduation Monitor

**Purpose**: Monitors active bonding curves and triggers graduation when threshold is reached.

**Schedule**: Every 5 minutes (cron)

**Logic**:
```
1. Fetch all active tokens from Vendure (status='active', graduated=false)
2. For each token:
   a. Query bonding curve contract for current market cap
   b. If marketCap >= 10 ETH:
      - Call TokenGraduator.graduate(tokenAddress)
      - Trigger notification
3. Update Vendure with latest market cap values
```

**Environment Variables**:
- `BASE_RPC_URL`: Base L2 RPC endpoint
- `VENDURE_API_URL`: Vendure GraphQL endpoint
- `VENDURE_API_TOKEN`: Admin API token
- `GRADUATOR_ADDRESS`: TokenGraduator contract address
- `GRADUATION_THRESHOLD`: Market cap threshold in ETH (default: 10)

### 3. Notification Service

**Purpose**: HTTP endpoint for sending notifications about platform events.

**Trigger**: HTTP POST (called by Event Indexer and Graduation Monitor)

**Supported Events**:
- `quorum_created`: New quorum formed
- `quorum_voted`: Vote received on quorum
- `quorum_executed`: Quorum reached unanimous vote
- `token_launched`: New token deployed
- `token_graduated`: Token graduated to Uniswap

**Notification Channels**:
- Discord webhooks
- Telegram bot API
- Email (via SendGrid/Resend)
- Custom webhooks

**Environment Variables**:
- `DISCORD_WEBHOOK_URL`: Discord webhook for notifications
- `TELEGRAM_BOT_TOKEN`: Telegram bot token
- `TELEGRAM_CHAT_ID`: Telegram channel/group ID
- `SENDGRID_API_KEY`: SendGrid API key (optional)
- `NOTIFICATION_EMAIL`: Email address for notifications

## Project Structure

```
headless-markets-workers/
├── src/
│   ├── event-indexer/
│   │   ├── index.ts                # Worker entry point
│   │   ├── event-parser.ts         # Parse contract events
│   │   ├── vendure-sync.ts         # Sync to Vendure API
│   │   └── types.ts                # Event types
│   ├── graduation-monitor/
│   │   ├── index.ts                # Worker entry point
│   │   ├── market-cap.ts           # Calculate market cap
│   │   └── trigger-graduation.ts   # Call graduator contract
│   ├── notification-service/
│   │   ├── index.ts                # Worker entry point
│   │   ├── discord.ts              # Discord webhook handler
│   │   ├── telegram.ts             # Telegram bot handler
│   │   ├── email.ts                # Email handler
│   │   └── types.ts                # Notification types
│   └── shared/
│       ├── types.ts                # Shared types
│       ├── abis.ts                 # Contract ABIs
│       ├── constants.ts            # Contract addresses
│       └── utils.ts                # Shared utilities
├── wrangler.toml                   # Cloudflare Workers config
├── package.json
├── tsconfig.json
└── README.md
```

## Development

### Prerequisites

- Node.js 20+
- pnpm (or npm/yarn)
- Cloudflare account
- Wrangler CLI (`npm i -g wrangler`)

### Setup

```bash
# Clone repo
git clone https://github.com/dutchiono/headless-markets-workers
cd headless-markets-workers

# Install dependencies
pnpm install

# Login to Cloudflare
wrangler login

# Create KV namespace for event indexer
wrangler kv:namespace create EVENT_INDEXER
wrangler kv:namespace create EVENT_INDEXER --preview

# Copy environment template
cp .env.example .env

# Add your environment variables
vim .env
```

### Local Development

```bash
# Run event indexer locally
pnpm dev:indexer

# Run graduation monitor locally
pnpm dev:monitor

# Run notification service locally
pnpm dev:notifications

# Test with curl
curl -X POST http://localhost:8787 \
  -H "Content-Type: application/json" \
  -d '{"type": "token_launched", "data": {...}}'
```

### Testing

```bash
# Run all tests
pnpm test

# Run specific worker tests
pnpm test:indexer
pnpm test:monitor
pnpm test:notifications

# Watch mode
pnpm test:watch
```

### Deployment

```bash
# Deploy all workers
pnpm deploy

# Deploy specific worker
pnpm deploy:indexer
pnpm deploy:monitor
pnpm deploy:notifications

# Deploy to staging
pnpm deploy:staging
```

## Configuration

### wrangler.toml

```toml
name = "headless-markets-workers"
compatibility_date = "2024-01-01"

# Event Indexer
[[workers]]
name = "event-indexer"
main = "src/event-indexer/index.ts"
route = "" # Cron only

[workers.triggers]
crons = ["* * * * *"] # Every minute

[workers.kv_namespaces]
binding = "KV"
id = "your-kv-namespace-id"

[workers.vars]
BASE_RPC_URL = "https://mainnet.base.org"
VENDURE_API_URL = "https://vendure.headless-markets.xyz/shop-api"
QUORUM_MANAGER_ADDRESS = "0x..."
BONDING_CURVE_ADDRESS = "0x..."
GRADUATOR_ADDRESS = "0x..."

[workers.secrets]
VENDURE_API_TOKEN = "***" # Set via: wrangler secret put VENDURE_API_TOKEN

# Graduation Monitor
[[workers]]
name = "graduation-monitor"
main = "src/graduation-monitor/index.ts"
route = "" # Cron only

[workers.triggers]
crons = ["*/5 * * * *"] # Every 5 minutes

[workers.vars]
BASE_RPC_URL = "https://mainnet.base.org"
VENDURE_API_URL = "https://vendure.headless-markets.xyz/shop-api"
GRADUATOR_ADDRESS = "0x..."
GRADUATION_THRESHOLD = "10"

[workers.secrets]
VENDURE_API_TOKEN = "***"

# Notification Service
[[workers]]
name = "notification-service"
main = "src/notification-service/index.ts"
route = "notifications.headless-markets.workers.dev/*"

[workers.vars]
DISCORD_WEBHOOK_URL = "https://discord.com/api/webhooks/..."
TELEGRAM_CHAT_ID = "@headless_markets"

[workers.secrets]
TELEGRAM_BOT_TOKEN = "***"
SENDGRID_API_KEY = "***" # Optional
```

## Monitoring

### Cloudflare Dashboard

- **Metrics**: Invocation count, errors, duration
- **Logs**: Real-time logs for debugging
- **Alerts**: Email alerts on errors or high latency

### Custom Metrics

```typescript
// In worker code
const startTime = Date.now();
try {
  await syncEvents();
  ctx.waitUntil(
    reportMetric('event_indexer_success', Date.now() - startTime)
  );
} catch (error) {
  ctx.waitUntil(
    reportMetric('event_indexer_error', 1)
  );
}
```

### Alerts

- **Event Indexer**: Alert if no events synced for 10 minutes
- **Graduation Monitor**: Alert if market cap calculation fails
- **Notification Service**: Alert on 5xx errors

## Error Handling

### Retry Logic

```typescript
async function syncWithRetry(events: Event[], maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await syncToVendure(events);
      return;
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await sleep(2 ** i * 1000); // Exponential backoff
    }
  }
}
```

### Graceful Degradation

- If Vendure is down, store events in KV and retry later
- If RPC is slow, reduce batch size
- If graduation fails, retry on next check

## Security

### API Key Protection

- All Vendure API calls require admin token
- Notification service validates incoming requests
- Secrets stored in Cloudflare Workers secrets (not in code)

### Rate Limiting

- Respect Base RPC rate limits (150 req/sec)
- Batch event queries to reduce RPC calls
- Implement exponential backoff on errors

### Input Validation

- Validate all contract event data before syncing
- Sanitize notification content to prevent XSS
- Verify contract addresses match expected values

## Performance

### Optimization Strategies

1. **Batch Processing**: Query multiple blocks at once (max 1000)
2. **Parallel Requests**: Use `Promise.all()` for independent calls
3. **Caching**: Store contract ABIs and addresses in memory
4. **Minimal Dependencies**: Keep bundle size small (<100KB)

### Benchmarks

- Event Indexer: ~500ms per invocation (1-10 events)
- Graduation Monitor: ~2s per invocation (10 tokens)
- Notification Service: ~200ms per notification

## Costs

### Cloudflare Workers Pricing

- **Free Tier**: 100,000 requests/day
- **Paid**: $5/month + $0.50 per million requests

### Estimated Usage

- Event Indexer: 1,440 invocations/day (every minute)
- Graduation Monitor: 288 invocations/day (every 5 minutes)
- Notification Service: ~100 invocations/day (varies)
- **Total**: ~2,000 invocations/day = well within free tier

## Related Repositories

- [headless-markets](https://github.com/dutchiono/headless-markets) - Main monorepo with frontend
- [ionoi-inc/vendure](https://github.com/ionoi-inc/vendure) - Vendure backend
- [ionoi-inc/agents](https://github.com/ionoi-inc/agents) - Agent coordination

## Contributing

See main [headless-markets](https://github.com/dutchiono/headless-markets) repo for contribution guidelines.

## License

MIT

---

**Status**: Initial setup - workers implementation in progress