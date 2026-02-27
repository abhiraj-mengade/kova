***

# `KOVA` — The Polkadot Super App
### Product, Technical & UX Document
**MontaQ Labs | v1.0 | February 2026**

***

## Why This Name

**KOVA** — derived from "Kovač" (Slovak/Croatian for *blacksmith*, someone who forges things together). Polkadot itself was named by a Slovenian founder. KOVA forges the two sides of Polkadot — Substrate and EVM — into one. It is short, strong, unique, and has no existing trademark in crypto.

Tagline: ***"Your complete Polkadot life."***

***

## The One-Line Product Brief

KOVA is not a wallet. It is the **Polkadot home screen** — the single place where a person lives their entire Polkadot life: assets, identity, governance, NFTs, contracts, and social activity — scoped entirely to the Relay Chain and Asset Hub/Polkadot Hub, without any cross-chain complexity.

***

## Framework Decision — Tauri v2

### Tauri v2 is the right call. Here is why.

Tauri v2 is stable, ships to iOS and Android from a single codebase, uses Rust for all business logic, and lets you write the UI in any JavaScript framework (React, Svelte, Vue). The minimum app binary is **600KB** vs 100MB+ for Electron. [v2.tauri](https://v2.tauri.app)

| Factor | Tauri v2 | Dioxus | Swift (iOS only) |
|---|---|---|---|
| iOS support | ✅ Stable  [v2.tauri](https://v2.tauri.app/blog/tauri-20/) | ✅ First-class  [github](https://github.com/DioxusLabs/dioxus) | ✅ Native |
| Android support | ✅ Same codebase  [v2.tauri](https://v2.tauri.app) | ✅ Same codebase  [github](https://github.com/DioxusLabs/dioxus) | ❌ Need Kotlin rewrite |
| Rust business logic | ✅ Full | ✅ Full | ⚠️ Via FFI |
| UI language | JS/TS (React/Svelte) | Pure Rust | Swift |
| Production ready | ✅ Yes  [v2.tauri](https://v2.tauri.app/blog/tauri-20/) | ⚠️ Maturing  [reddit](https://www.reddit.com/r/rust/comments/1p9vhln/is_dioxus_already_being_used_in_production/) | ✅ Yes |
| Native device APIs | ✅ Swift/Kotlin plugins  [v2.tauri](https://v2.tauri.app/blog/tauri-20/) | ⚠️ Limited | ✅ Full |
| Secure Enclave (iOS) | ✅ Via Swift plugin  [v2.tauri](https://v2.tauri.app/blog/tauri-20/) | ⚠️ Manual | ✅ Native |
| UI quality ceiling | High (React + CSS) | Medium | Highest |

**Decision: Tauri v2 + React (TypeScript) frontend + Rust backend logic.**

Rust handles all chain interaction, cryptography, and business logic. React handles all UI. The same codebase ships to iOS and Android. When you need native hardware (Secure Enclave, biometrics, NFC), Tauri v2 lets you drop into Swift/Kotlin plugins for exactly those pieces. [v2.tauri](https://v2.tauri.app/blog/tauri-20/)

### What This Looks Like in Practice

```
KOVA/
├── src/           → React + TypeScript (all UI)
├── src-tauri/     → Rust (all chain logic, crypto, API calls)
│   └── plugins/
│       ├── ios/   → Swift plugin for Secure Enclave, FaceID
│       └── android/ → Kotlin plugin for fingerprint, keystore
```

You write the wallet/chain logic once in Rust. You write the UI once in React. Both iOS and Android get it.

***

## Scope — What KOVA Covers

### In Scope
- **Polkadot Relay Chain** — DOT balance, staking, OpenGov voting
- **Polkadot Asset Hub** — fungible assets (USDT, USDC, etc.), NFTs (pallet_nfts), on-chain identity
- **Polkadot Hub Smart Contracts** — REVM (Solidity/EVM) and PVM (ink!/RISC-V) interaction and browsing [docs.polkadot](https://docs.polkadot.com/smart-contracts/overview/)

### Explicitly Out of Scope (v1)
- ❌ XCM cross-chain transfers
- ❌ Parachains (Moonbeam, Astar, Hydration, etc.)
- ❌ Kusama
- ❌ Staking pools, parachain crowdloans

This scope ships in 4 weeks and has a clear, focused value proposition. Parachains can be v2.

***

## Architecture Overview

### Three-Layer System

**Layer 1 — Tauri Rust Core (on device)**
All private key operations, transaction signing, and cryptography live here — never leave the device. This layer talks directly to Polkadot RPC nodes.

**Layer 2 — KOVA Cloud Backend (Hetzner €4/month)**
Non-sensitive enrichment only: AI governance summaries (cached), identity resolution cache, push notification delivery, price feeds, security/scam database. The backend never sees private keys.

**Layer 3 — Polkadot Network**
Three RPC endpoints — Relay Chain (Substrate), Asset Hub (Substrate), Polkadot Hub EVM endpoint. All free, publicly hosted by Parity/Dwellir.

### Why Split This Way

Private keys stay on-device inside hardware security (Secure Enclave on iOS, Android Keystore on Android). The cloud backend only handles data that is already public on-chain or cheap to regenerate (AI summaries, price data). This means even if the KOVA backend goes down, the wallet still works completely.

***

## The Dual Identity System

This is KOVA's most important technical concept and must be explained clearly in the UX.

Every Polkadot user now has two addresses that point to the same funds:

- A **Substrate address** (SS58 format, starts with `1` for Polkadot) — used for DOT transfers, staking, governance, Asset Hub
- An **EVM address** (0x format, 20 bytes) — used for Solidity contracts, ERC-20 tokens, ERC-721 NFTs on Polkadot Hub

These are mapped via `pallet_revive.map_account` — a one-time on-chain operation that links both addresses to the same underlying key. [openguild](https://openguild.wtf/blog/polkadot/polkadot-how-to-set-up-pallet-revive-in-substrate-parachain-template)

**KOVA's job:** Make this completely invisible to 95% of users. Show one identity. Use the right address format silently when needed. Let power users toggle to see both.

***

## Feature Specification

### Feature 1 — Unified Home Feed

**Purpose:** Give users a daily reason to open the app. Not a transaction list — a living feed of everything happening in their Polkadot life.

**What it shows:**
- Incoming and outgoing DOT transfers (humanized — "Received 12 DOT from alex.dot")
- Asset Hub token movements (USDT, USDC received/sent)
- NFT activity (received, sold, listed)
- Governance events (voted, referendum passed/failed, delegation received)
- Staking rewards received
- Smart contract events (if user has interacted with any contracts)
- Security alerts (flagged address interaction, unusual activity)

**UX principles:**
- Every item uses plain human language — zero jargon, zero hex strings
- Each item has a clear icon indicating type (💸 transfer, 🗳️ governance, 🖼️ NFT, ⚡ contract, ⚠️ security)
- Tap any item → full detail view with contextual explanation
- Real-time via WebSocket — new items slide in from top without full refresh
- Pull to refresh as fallback

**Differentiator from Nova:** Nova shows a transaction list. KOVA shows a narrative of your Polkadot activity — telling you what happened, not just that something happened.

***

### Feature 2 — Assets

**Purpose:** Show every asset the user holds, clearly, with USD value, across both Substrate and EVM.

**Asset types shown:**
- Native DOT (Relay Chain)
- Asset Hub fungible tokens (USDT, USDC, and any other listed assets)
- ERC-20 tokens on Polkadot Hub EVM

**UX principles:**
- Total portfolio value prominently at top in both DOT and USD/INR
- Assets sorted by USD value descending
- Each asset shows: logo, name, balance, USD value, 24h change
- Subtle tag on each: "Substrate" or "EVM" — small, secondary, not prominent
- Tap asset → full history + send/receive for that specific asset

**The Gas Trap Prevention — KOVA's Killer Feature:**

Before any send action, KOVA proactively checks if the user has enough DOT for fees. If not:

```
UX flow:
User taps "Send USDT"
→ KOVA checks DOT balance for fees
→ DOT is insufficient
→ BEFORE showing the send form, KOVA shows:

"You need a little DOT for network fees
 You have enough USDT to cover it.
 [Get DOT for fees — takes 10 seconds]"

→ User taps → KOVA automatically swaps 0.10 USDT → DOT
→ Then proceeds to the Send form

The user never hits an error.
The user never gets stuck.
```

This single feature fixes the #1 complaint in every Polkadot wallet App Store review. [apps.apple](https://apps.apple.com/us/app/nova-polkadot-wallet/id1597119355?l=ko)

***

### Feature 3 — Governance (The Social Voting Room)

**Purpose:** Make governance participation feel like a community activity, not bureaucratic homework.

**The Swipe Interface:**

Cards are shown one at a time. Each card has:

1. **Track badge** — colour-coded (Root is red, Treasury is gold, Fellowship is purple, etc.)
2. **Referendum number + time remaining** — shown as human time ("3 days left"), not block numbers
3. **AI summary** — exactly 2 sentences, plain language, neutral, generated by Claude Haiku and cached for 24 hours. Cost: $0.003 per summary
4. **Live tally bar** — visual AYE vs NAY progress
5. **Social context bar** ← This is what Nova doesn't have
   - "Web3 Foundation voted AYE"
   - "3 validators you follow voted NAY"
   - Shows on-chain identity names, not addresses
6. **Conviction selector** — horizontal slider below the card (1x to 6x), shows lock duration in plain language ("6x = your DOT locked for 6 months")

**Gestures:**
- Swipe right → AYE with selected conviction
- Swipe left → NAY with selected conviction
- Swipe up → Skip (goes to "decide later" pile)
- Tap card → Full proposal text + Polkassembly discussion link

**Governance Portfolio view:**
- All your active votes in one screen
- How much DOT is locked, when it unlocks (in days, not blocks)
- "You've voted on 14 referenda this month — top 8% of participants" (streak/gamification)
- One-tap to remove vote if you change your mind (before referendum ends)

**Delegation:**
- Simple delegation UI: choose a track → pick from a curated list of known delegates (showing their voting history)
- "Alex.dot votes on Treasury track 94% of the time — last 30 days"

***

### Feature 4 — NFT Gallery

**Purpose:** Show all NFTs the user owns in one beautiful gallery, regardless of whether they live on Substrate or EVM.

**Sources:**
- `pallet_nfts` collections on Asset Hub (Substrate NFTs)
- `pallet_uniques` legacy collections on Asset Hub
- ERC-721 contracts on Polkadot Hub EVM

**UX:**
- Pinterest-style masonry grid layout
- Each NFT has a small chain badge: "Substrate" or "EVM" — subtle bottom-right corner tag
- Tap → full screen view with: name, collection, description, trait list, floor price if available
- Long press → Share, Send options
- Filter by: All / Substrate / EVM / Collection

**No cross-chain moves** (out of scope for v1 — no XCM).

***

### Feature 5 — Contracts Browser

**Purpose:** Let users discover and interact with smart contracts deployed on Polkadot Hub — both Solidity (REVM) and ink!/RISC-V (PVM) — safely.

**Discovery:**
- List of verified contracts on Polkadot Hub (pulled from block explorer API)
- Filter: All / Solidity (REVM) / ink! (PVM) / Verified / New
- Each contract shows: name (if verified), type, deployment age, risk score

**Safety-first interaction flow:**

Every time a user taps a contract or a dApp requests connection:

```
Contract Details screen shows:
- Name (if verified) or "Unknown Contract"
- Type: Solidity on REVM / ink! on PVM
- Verified source code: Yes ✓ / No ✗
- Deployed: X days ago
- Risk score: LOW / MEDIUM / HIGH
- Risk reasons listed plainly:
  "Source code not published"
  "Deployed 2 hours ago"
  "No previous user interactions"

Action buttons only appear AFTER user reads this screen.
```

**WalletConnect integration:**
- When a dApp connects via WalletConnect, KOVA auto-detects if it is an EVM dApp or Substrate dApp
- Injects the correct provider silently
- Shows banner: "Connected as your Ethereum address" or "Connected as your Polkadot address"
- User never manually manages this

***

### Feature 6 — Identity Profile

**Purpose:** Give users a sense of their Polkadot presence and reputation.

**What it shows:**
- Display name from `pallet_identity` (if set)
- Both addresses (Substrate SS58 + EVM 0x) — tappable to copy
- Judgement status (Verified / Known Good / etc.)
- "Set on-chain identity" CTA if not set — guides through the flow
- Governance activity score
- Staking status (nominating, validator set position, unbonding)
- Follow button for watching other addresses

**Social layer — Follow addresses:**
- Follow any address → their activity appears in a "Following" section of the home feed
- See what governance decisions key community members make
- Follow the Web3 Foundation address, treasury, known teams
- This is KOVA's social moat — over time it becomes the social graph of Polkadot

***

### Feature 7 — Security Layer (Always-On)

This is not a feature users interact with directly — it runs silently beneath everything.

**Clipboard attack prevention:**
When user opens the Send screen, KOVA captures the current clipboard address. If the clipboard changes while they switch apps and come back, KOVA shows a clear warning before they paste.

**Suspicious address flagging:**
Every address the user interacts with is checked against a public scam database (ScamSniffer API + community-maintained lists). If flagged, a red warning banner appears before any action is possible.

**New contract warnings:**
Any contract deployed less than 7 days ago gets a visible "New — unproven" badge. Not blocked — just clearly labelled.

**Transaction simulation:**
Before signing, show a plain-language summary of what the transaction will do:
```
"This will:
 → Send 50 USDT to 0x742d...
 → Approve this contract to spend your USDT (limit: 50 USDT)
 → Cost: 0.003 DOT in fees"
```
Not hex data. Not raw calldata. Human language.

***

## UX Principles — The KOVA Design Language

### 1. Outcomes over Operations
Never show the user what the blockchain is doing. Show them what *they* are achieving.
- ❌ "Submit extrinsic to pallet_balances::transfer"
- ✅ "Send 10 DOT to alex.dot"

### 2. Proactive over Reactive
KOVA anticipates problems before they happen.
- No transaction fails because of missing gas fees
- No referendum closes without the user knowing
- No scam address goes unwarned

### 3. Context Cards — Explain Everything, Exactly Once
Every confusing Polkadot concept gets a tappable "?" that opens a Context Card:
- Conviction → "Locking DOT longer multiplies your vote power. You choose the tradeoff."
- Unbonding → "Your DOT is being released. Ready in 28 days."
- ED (Existential Deposit) → "Polkadot requires a tiny minimum balance to keep your account alive."
Context Cards appear inline, not in a help section nobody reads.

### 4. Progressive Disclosure
Normal users see a clean, simple interface. Power users tap deeper:
- Home → default: clean feed. Tap account name → see both Substrate + EVM addresses
- Send → default: simple amount + address. Tap "Advanced" → raw extrinsic builder
- Governance → default: swipe cards. Tap "Full Details" → raw proposal text

### 5. Human Time, Not Block Time
- ❌ "Referendum ends at block 22,845,100"
- ✅ "Referendum ends in 3 days, 4 hours"
- ❌ "Unbonding period: 604,800 blocks"
- ✅ "Your DOT unlocks on March 28, 2026"

***

## Navigation Structure

```
Bottom Navigation Bar (5 tabs):

🏠 HOME     — Unified activity feed, portfolio value at top

💰 ASSETS   — All fungible tokens, Send/Receive actions

🗳️ VOTE     — Governance swipe cards, portfolio, delegation

🖼️ COLLECT  — NFT gallery (Substrate + EVM unified)

👤 PROFILE  — Identity, addresses, settings, followed accounts
```

No tab for "Contracts" — contracts are accessed contextually (via dApp browser or directly from the Assets/Feed screen when relevant). Keeping 5 clean tabs keeps the navigation simple.

***

## Onboarding Flow

```
Screen 1: "Welcome to KOVA"
  Tagline: "Your complete Polkadot life."
  [Create New Wallet]  [Import Existing]

Screen 2 (Import): "Paste your seed phrase or private key"
  → Accepts: 12/24-word mnemonic, raw private key, JSON keystore
  → Detects key type automatically (SR25519, ECDSA, ed25519)

Screen 3: "Securing your wallet"
  Animation showing key going into a lock
  "Your key is stored in your phone's secure chip.
   It never leaves your device."
  → FaceID/TouchID enrollment prompt

Screen 4: "Back up to iCloud" (optional but strongly encouraged)
  "If you lose your phone, your wallet is recoverable.
   Encrypted with your PIN — Apple cannot read it."
  [Back up to iCloud]  [I'll do this later]

Screen 5: "Your Polkadot identity"
  Shows: Polkadot address (5Grw...)
  Small toggle: "Also show Ethereum address"
  → Explains the dual address in one sentence:
    "You also have an Ethereum address for new Polkadot smart contracts.
     Same funds, both are you."

Screen 6: Home feed loads → Done.
```

Total onboarding: under 90 seconds.

***

## Framework & Stack Summary

| Component | Choice | Reason |
|---|---|---|
| **App framework** | Tauri v2 | iOS + Android from one codebase, Rust logic, production stable  [v2.tauri](https://v2.tauri.app/blog/tauri-20/) |
| **UI language** | React + TypeScript | Fastest UI iteration, best component ecosystem, Tauri-first  [v2.tauri](https://v2.tauri.app) |
| **Chain logic** | Rust (subxt + alloy-rs) | Your existing expertise, best Substrate client |
| **Key storage iOS** | Swift plugin → Secure Enclave | Hardware security, free, no TEE needed  [v2.tauri](https://v2.tauri.app/blog/tauri-20/) |
| **Key storage Android** | Kotlin plugin → Android Keystore | Android hardware security equivalent |
| **Backend** | Rust (Axum) on Hetzner CX21 | €4.15/month, non-sensitive enrichment only |
| **AI summaries** | Claude Haiku via OpenRouter | $0.003/call, cached 24h in Redis |
| **Push notifications** | APNs (iOS) + FCM (Android) via backend | Native, free |
| **Database** | Supabase free tier (PostgreSQL) | User prefs, notification settings, watchlists |
| **Price feeds** | CoinGecko API free tier | DOT, USDT, asset prices in USD/INR |
| **Identity** | Subscan API + on-chain `pallet_identity` | Free, comprehensive |

***

## 4-Week Milestone Plan

| Week | What Gets Built | User Can Do This |
|---|---|---|
| **Week 1** | Tauri v2 setup, key import/creation, Secure Enclave plugin, dual address derivation, DOT balance query | Import a wallet, see DOT balance, see both addresses |
| **Week 2** | Home feed (Substrate + EVM events), Asset Hub token balances, Send/Receive with gas trap prevention | See full activity feed, send DOT/USDT safely, never get gas trapped |
| **Week 3** | Governance swipe cards with AI summaries + social context, NFT gallery (pallet_nfts + ERC-721), identity resolution | Vote on governance, view NFT collection, see human names not addresses |
| **Week 4** | Security layer (clipboard protection, scam detection, contract risk scores), push notifications, iCloud backup, WalletConnect, UI polish | Connect to dApps safely, get governance deadline alerts, recover wallet from iCloud |

**Total infrastructure cost:** €4.15/month compute + ~$10/month AI summaries + $0 database + $0 RPCs = **under $20/month total.**

***

## Revenue & Grant Path

**Polkadot Treasury Referendum** — Submit after Week 4 TestFlight beta with a demo video. Community votes on-chain. Historically funds wallet/tooling projects between $50K–$200K. The existing $30K UX audit that changed nothing  makes the community ready to fund something that actually ships. [forum.polkadot](https://forum.polkadot.network/t/a-case-study-in-misaligned-incentives-the-polkadot-ux-bounty/16275)

**Web3 Foundation Grants** — Apply in parallel for the open-source UX/tooling track. $20K–$50K range.

**Pro subscription ($3.99/month)** — Advanced governance analytics, unlimited watchlists, custom price alerts, hardware wallet support.

**Parachain SDK licensing (v2)** — Once XCM is added in v2, parachains like Hydration and Moonbeam pay MontaQ for deep integration into KOVA.

Kilo Code
about 2 hours ago
18,924 tokens · $0.0000

I'll start by exploring the workspace to understand what's already there before scaffolding the KOVA project.


ls -la
Kilo Code
about 2 hours ago
319 tokens · $0.0000


node --version && npm --version && rustc --version && car...
Kilo Code
about 2 hours ago
228 tokens · $0.0000


which cargo || echo "cargo not found" && which rustup || ...
Kilo Code
about 2 hours ago
600 tokens · $0.0000


todowrite
13 pending
Kilo Code
about 2 hours ago
1,518 tokens · $0.0000


todowrite
12 pending, 1 active
Kilo Code
about 2 hours ago
1,186 tokens · $0.0000


curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs...
Kilo Code
about 2 hours ago
739 tokens · $0.0000


source "$HOME/.cargo/env" && rustc --version && cargo --v...
Kilo Code
about 2 hours ago
623 tokens · $0.0000


todowrite
11 pending, 1 active, 1 done
Kilo Code
about 2 hours ago
1,316 tokens · $0.0000

Now I'll install the Tauri CLI and set up all prerequisites in parallel:


source "$HOME/.cargo/env" && cargo install tauri-cli --ve...

apt-get install -y libwebkit2gtk-4.1-dev libssl-dev libgt...
Kilo Code
about 2 hours ago
9,656 tokens · $0.0000


apt-get update 2>&1 | tail -5 && apt-get install -y build...
Kilo Code
about 2 hours ago
643 tokens · $0.0000


source "$HOME/.cargo/env" && cargo install tauri-cli --ve...
Kilo Code
about 2 hours ago
34,246 tokens · $0.0000


todowrite
10 pending, 1 active, 2 done
Kilo Code
about 2 hours ago
1,197 tokens · $0.0000

Now I'll scaffold the entire project structure. Since cargo tauri init requires interactive prompts, I'll build the structure manually — which also gives full control over layout:


npm create vite@latest kova -- --template react-ts 2>&1
Kilo Code
about 2 hours ago
602 tokens · $0.0000


cd kova && npm install && npm install -D tailwindcss @tai...
Kilo Code
about 2 hours ago
806 tokens · $0.0000

Now I'll set up the Tauri v2 Rust core. First let me write all the project files in parallel batches:


source "$HOME/.cargo/env" && cargo init --name kova-core ...

mkdir -p kova/src-tauri/{capabilities,icons,plugins/{ios,...
Kilo Code
about 2 hours ago
945 tokens · $0.0000

Now I'll write all the files. Starting with configuration files:


vite.config.ts
31 lines, 779B
Kilo Code
about 2 hours ago
724 tokens · $0.0000


read
vite.config.ts

read
Cargo.toml
Kilo Code
about 2 hours ago
808 tokens · $0.0000


read
main.rs

read
src
Kilo Code
about 2 hours ago
835 tokens · $0.0000


read
App.tsx

read
index.css

read
main.tsx
Kilo Code
about 2 hours ago
2,270 tokens · $0.0000


todowrite
8 pending, 1 active, 4 done
Kilo Code
about 2 hours ago
1,553 tokens · $0.0000

Now writing all the files in large parallel batches:


edited
vite.config.ts
Kilo Code
about 2 hours ago
1,932 tokens · $0.0000


edited
index.css
Kilo Code
about 2 hours ago
1,730 tokens · $0.0000


edited
main.tsx
Kilo Code
about 2 hours ago
2,708 tokens · $0.0000

Now write the TypeScript types, then app router, then all pages and components:


wrote
index.ts
252 lines, 7.7KB
Kilo Code
about 2 hours ago
6,120 tokens · $0.0000


wrote
tauri.ts
372 lines, 12.7KB
Kilo Code
about 2 hours ago
5,470 tokens · $0.0000


wrote
format.ts
150 lines, 5.2KB
Kilo Code
about 2 hours ago
2,381 tokens · $0.0000


wrote
wallet.ts
61 lines, 1.6KB
Kilo Code
about 2 hours ago
1,701 tokens · $0.0000


cd kova && npm install zustand 2>&1 | tail -5

wrote
assets.ts
35 lines, 766B

wrote
feed.ts
52 lines, 1.1KB
Kilo Code
about 2 hours ago
2,289 tokens · $0.0000


wrote
governance.ts
60 lines, 1.4KB

wrote
nfts.ts
44 lines, 965B
Kilo Code
about 2 hours ago
2,194 tokens · $0.0000

Now write the shared UI components:


wrote
ContextCard.tsx
92 lines, 2.8KB
Kilo Code
about 2 hours ago
1,534 tokens · $0.0000


wrote
ChainBadge.tsx
41 lines, 982B
Kilo Code
about 2 hours ago
1,339 tokens · $0.0000


wrote
SecurityBanner.tsx
62 lines, 2.0KB
Kilo Code
about 2 hours ago
1,581 tokens · $0.0000


wrote
Button.tsx
60 lines, 1.6KB
Kilo Code
about 2 hours ago
1,416 tokens · $0.0000


wrote
TransactionSimulator.tsx
48 lines, 1.7KB
Kilo Code
about 2 hours ago
1,659 tokens · $0.0000

Now write the feed, assets, governance, NFT and contract components, then the five pages:


wrote
ActivityFeedItem.tsx
86 lines, 2.5KB
Kilo Code
about 2 hours ago
2,004 tokens · $0.0000


wrote
AssetRow.tsx
78 lines, 2.5KB
Kilo Code
about 2 hours ago
2,012 tokens · $0.0000


wrote
GasTrapGuard.tsx
83 lines, 2.6KB
Kilo Code
about 2 hours ago
4,260 tokens · $0.0000


wrote
ReferendumCard.tsx
272 lines, 9.4KB
Kilo Code
about 2 hours ago
3,955 tokens · $0.0000


wrote
NftCard.tsx
57 lines, 1.6KB
Kilo Code
about 2 hours ago
2,327 tokens · $0.0000


wrote
ContractRiskCard.tsx
125 lines, 4.7KB
Kilo Code
about 2 hours ago
2,301 tokens · $0.0000

Now write the navigation and the five main pages:


wrote
BottomNav.tsx
54 lines, 1.6KB
Kilo Code
about 2 hours ago
1,826 tokens · $0.0000


wrote
Home.tsx
89 lines, 3.2KB
Kilo Code
about 2 hours ago
3,404 tokens · $0.0000


wrote
Assets.tsx
200 lines, 7.1KB
Kilo Code
about 2 hours ago
Streaming...
write...
Scroll to 
