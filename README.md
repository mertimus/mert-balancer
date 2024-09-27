# Solana RPC Load Balancer with Cloudflare Workers

This project sets up a load balancer for Solana RPC nodes using Cloudflare Workers. It allows you to distribute requests across multiple Solana RPC endpoints, improving reliability and performance for your Solana dApps.

## Prerequisites

- A Cloudflare account (free tier is sufficient to start)
- Node.js and npm installed on your local machine
- Basic familiarity with JavaScript and command-line operations

## Setup Instructions

### 1. Cloudflare Account Setup

1. If you don't have a Cloudflare account, sign up at [https://dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up).
2. Log in to your Cloudflare dashboard.

### 2. Create a KV Namespace

1. In the Cloudflare dashboard, click on "Workers & Pages" in the sidebar.
2. Go to the "KV" tab.
3. Click "Create namespace".
4. Name it "SOLANA_RPC_KV" and click "Add".

### 3. Create the Load Balancer Worker

1. In the Cloudflare dashboard, go to "Workers & Pages".
2. Click "Create application".
3. Choose "Create Worker".
4. Give your Worker a name (e.g., "solana-rpc-load-balancer").
5. Click "Deploy".
6. You'll be taken to the editor. Replace the default code with the following:

```javascript
export default {
  async fetch(request, env, ctx) {
    // Handle CORS preflight requests
    if (request.method === "OPTIONS") {
      return new Response(null, {
        headers: {
          "Access-Control-Allow-Origin": "*",
          "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
          "Access-Control-Allow-Headers": "Content-Type",
        },
      });
    }

    // Get the list of healthy RPC endpoints from KV
    const healthyRPCs = await env.SOLANA_RPC_KV.get('healthy_rpcs', 'json')
    
    if (!healthyRPCs || healthyRPCs.length === 0) {
      return new Response('No healthy RPC endpoints available', { status: 503 })
    }

    // Simple round-robin selection
    const selectedRPC = healthyRPCs[Math.floor(Math.random() * healthyRPCs.length)]

    // Forward the request to the selected RPC
    const response = await fetch(selectedRPC, {
      method: request.method,
      headers: request.headers,
      body: request.body
    })

    // Clone the response so that we can modify headers
    const modifiedResponse = new Response(response.body, response)

    // Add CORS headers to the response
    modifiedResponse.headers.set("Access-Control-Allow-Origin", "*")

    // Return the modified response
    return modifiedResponse
  }
}
```

7. Click "Save and deploy".

### 4. Bind the KV Namespace to your Worker

1. In your Worker's dashboard, go to the "Settings" tab.
2. Scroll down to "Variables".
3. Under "KV Namespace Bindings", click "Add binding".
4. Set the Variable name to "SOLANA_RPC_KV" and select your KV namespace.
5. Click "Save".

### 5. Add RPC Endpoints to KV

1. Go to the "KV" tab in the Workers & Pages section.
2. Click on your "SOLANA_RPC_KV" namespace.
3. Click "Add key-value pair".
4. Set the key to "healthy_rpcs" and the value to a JSON array of your RPC endpoints, e.g.:
   ```json
   ["https://api.mainnet-beta.solana.com", "https://solana-api.projectserum.com"]
   ```
5. Click "Add".

### 6. Create the Health Check Worker (Optional but Recommended)

1. Repeat steps 3-4, but name this worker "solana-rpc-health-check".
2. Use the following code for this worker:

```javascript
export default {
  async scheduled(event, env, ctx) {
    const allRPCs = await env.SOLANA_RPC_KV.get('all_rpcs', 'json')
    const healthyRPCs = []

    for (const rpc of allRPCs) {
      try {
        const response = await fetch(rpc, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            jsonrpc: '2.0',
            id: 1,
            method: 'getHealth'
          })
        })

        if (response.ok) {
          const result = await response.json()
          if (result.result === 'ok') {
            healthyRPCs.push(rpc)
          }
        }
      } catch (error) {
        console.error(`Health check failed for ${rpc}:`, error)
      }
    }

    await env.SOLANA_RPC_KV.put('healthy_rpcs', JSON.stringify(healthyRPCs))
  }
}
```

3. Set up a Cron Trigger for this worker:
   - In the worker's settings, go to the "Triggers" tab.
   - Under "Cron Triggers", click "Add Cron Trigger".
   - Set it to run every minute: `* * * * *`
   - Click "Add Trigger".

### 7. Using the Load Balancer in Your Project

1. Install the Solana web3.js library in your project:
   ```
   npm install @solana/web3.js
   ```

2. Use the following code to interact with your load balancer:

```javascript
import { Connection, PublicKey, LAMPORTS_PER_SOL } from '@solana/web3.js';

// Replace with your Cloudflare Worker URL
const WORKER_URL = 'https://your-worker.your-subdomain.workers.dev';

// Create a connection to your Cloudflare Worker
const connection = new Connection(WORKER_URL);

async function getAccountBalance(publicKeyString) {
  const publicKey = new PublicKey(publicKeyString);
  const balance = await connection.getBalance(publicKey);
  return balance / LAMPORTS_PER_SOL;
}

async function getLatestBlockHeight() {
  return await connection.getBlockHeight();
}

// Example usage
async function main() {
  try {
    // Get account balance
    const publicKeyString = 'YourSolanaPublicKeyHere';
    const balance = await getAccountBalance(publicKeyString);
    console.log(`Account balance: ${balance} SOL`);

    // Get latest block height
    const blockHeight = await getLatestBlockHeight();
    console.log(`Latest block height: ${blockHeight}`);

  } catch (error) {
    console.error('Error:', error);
  }
}

main();
```

Replace `'https://your-worker.your-subdomain.workers.dev'` with your actual Cloudflare Worker URL.

## Testing

1. Run your script using Node.js:
   ```
   node your-script-name.js
   ```

2. You should see the account balance and latest block height printed to the console.

## Troubleshooting

- If you encounter CORS issues, ensure that the CORS handling code in the load balancer worker is correct.
- If you're not getting responses, check that your RPC endpoints are correctly added to the KV store.
- Verify that the KV namespace is correctly bound to your workers.

## Next Steps

- Implement more sophisticated load balancing algorithms.
- Add error handling and automatic failover.
- Implement caching for frequently requested data.

Congratulations! You now have a functioning Solana RPC load balancer using Cloudflare Workers. This setup will help distribute your RPC requests across multiple endpoints, improving reliability and performance for your Solana applications.
