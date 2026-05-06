# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

All commands run from the `frontend/` directory:

```bash
# Install dependencies (--legacy-peer-deps required due to peer conflicts)
npm install --legacy-peer-deps

# Start dev server (port 3000)
npm run dev

# Production build (outputs to frontend/dist/)
npm run build

# Preview production build
npm run preview
```

**Deployment**: Vercel. Build command is `cd frontend && npm install --legacy-peer-deps && npm run build`. Output: `frontend/dist`.

## Environment Variables

Create `frontend/.env`:

```bash
VITE_CHAT_API_BASE=http://localhost:8000   # Backend LLM API URL
VITE_DEFAULT_NETWORK=base-sepolia          # base-sepolia | base | sepolia
VITE_ONCHAINKIT_API_KEY=...               # Optional: Coinbase CDP API key
VITE_DEFAULT_CHAIN_ID=84532              # Optional: override chain ID
```

## Architecture

This is a blockchain-enabled AI chat UI for leasing infrastructure (LXC containers via Proxmox) using x402 micropayments (USDC on Base).

**Three-service system**:
1. **This frontend** — React 18 + Vite + TypeScript, no Tailwind (plain CSS)
2. **backend-llm** (port 8000) — LLM agent that decides when payment is required
3. **backend-proxmox** (port 4021) — behind the payment wall; provisions containers

**Blockchain stack**: Wagmi 2 + Viem 2 + OnchainKit 1 (Coinbase wallet components). Single-chain at build time; chain determined by `VITE_DEFAULT_NETWORK`.

**Farcaster MiniApp**: The app runs as a Farcaster/Base MiniApp. `index.html` has the MiniApp meta tag; `public/.well-known/farcaster.json` has the app manifest (served with CORS headers via `vercel.json`).

## Key Files

| File | Role |
|------|------|
| `frontend/src/App.tsx` | Main component: chat UI, wallet auto-connect, payment flow |
| `frontend/src/Providers.tsx` | Wagmi + OnchainKit provider config, chain selection |
| `frontend/src/x402.ts` | Payment protocol: `encodeX402Header()`, EIP-712 typed data, `generateNonce()` |
| `frontend/src/main.tsx` | Entry point |
| `frontend/src/styles.css` | All styling |
| `frontend/public/.well-known/farcaster.json` | Farcaster MiniApp manifest |

## Payment Flow

1. User sends a chat message → POST to `VITE_CHAT_API_BASE/chat`
2. If backend returns `402 Payment Required`, frontend stores the pending message
3. Wallet auto-connects (or was already connected); EIP-712 USDC `TransferWithAuthorization` signature is requested from user
4. Payment proof encoded as Base64 JSON in `X-Payment` header
5. Original request resent with payment header

**Critical: duplicate payment prevention** uses `useRef<boolean>` (not `useState`) so the flag survives re-renders without causing them. Pattern in `App.tsx`:

```typescript
const isProcessingPayment = useRef<boolean>(false);
// Guard at top of handlePayment():
if (isProcessingPayment.current) return;
isProcessingPayment.current = true;
try { /* sign + send */ } finally { isProcessingPayment.current = false; }
```

## Farcaster MiniApp Detection

```typescript
const isInMiniApp = typeof window !== 'undefined' && window.parent !== window;
```

When running in MiniApp context, wallet auto-connect is skipped (Farcaster handles it). `sdk.actions.ready()` must be called to signal the host that the app is ready.
