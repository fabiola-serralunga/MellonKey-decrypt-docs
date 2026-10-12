## 🛠️ MellonKey◉fs◉ — v2.1.0 — Portfolio Project
> MellonKey◉fs◉, inspired by the gate of Moria.

Document originally drafted in Spanish by https://github.com/fabiola-serralunga. Translated into English using a large language model (LLM) and hosted in README.md

### v2.1.0 *MellonKeyDecrypt* — *The Narya Keepers* - Documentation

Successor to *Mellon-Key-Simple* v1.0.0: same mint/verification logic, but the deliverable file no longer lives in R2 — it is **encrypted with AES-256-GCM on Arweave**, and a **portable Cloudflare Worker** fetches it (with gateway redundancy), decrypts it on the fly, and delivers it solely to the verified SBT holder.
<br><br>

🌐 https://mellonkeyfs.pages.dev — site of *MellonKeyDecrypt*

🔗 https://mellonkeyfs.pages.dev/deliver — decryption and download site. The file containing the digital asset is encrypted and uploaded to Arweave. In `deliver.html`, a Cloudflare Worker verifies ownership of the Soulbound Token, decrypts the file, and delivers it. It also displays the Acquisition Certificate and shows the contract's state.

🔗 https://base-sepolia.blockscout.com/address/0x9DF53bB4F3A56b7927560C2638074f55F5DDD20E?tab=tokens_nfts — to view the SBTs minted in the project on its testnet space. In *Cirdan Narya Keeper*, the contract was deployed in such a way that the vectorized image built during contract deployment can be observed: its base is vectorized code plus the printed data of the digital asset.

📜⛓️🔷 Solidity contracts follow the *abstract contract* pattern `SBTDescriptor...` + the inheriting contract. For example, `SBTDescriptorCirdan.sol` + `TestSBTCirdan.sol`.
They were coded keeping in mind the goal of generating a token that, as long as the network where it was minted exists (in this case Base Sepolia), will always preserve and display its metadata for name, description, image, and attributes.

## Table of Contents

- [MellonKey◉fs◉](#mellonkeyfs)
- [MellonKeyDecrypt — v2.1.0 — *The Narya Keepers*](#mellonkeydecrypt--v210--the-narya-keepers)
- [1. Complete flow](#1-complete-flow)
- [2. Arweave gateway redundancy (inside the worker)](#2-arweave-gateway-redundancy-inside-the-worker)
- [3. Worker portability](#3-worker-portability)
- [4. Encrypting and uploading the file (per edition)](#4-encrypting-and-uploading-the-file-per-edition)
- [5. Worker and monorepo deployment](#5-worker-and-monorepo-deployment)
  - [5.1 How `worker/` coexists with the Pages files in `public/`](#51-how-worker-coexists-with-the-pages-files-in-public)
  - [5.2 Commands](#52-commands)
  - [5.3 Testing locally (VSCode Live Server + wrangler dev) — and the 405 error](#53-testing-locally-vscode-live-server--wrangler-dev--and-the-405-error)
- [6. Contracts on Sepolia: 100% on-chain metadata](#6-contracts-on-sepolia-100-on-chain-metadata)
  - [6.1 Inside that JSON, the critical data is guaranteed by construction:](#61-inside-that-json-the-critical-data-is-guaranteed-by-construction)
  - [6.2 The art: controlled vectorization](#62-the-art-controlled-vectorization)
- [7. A `descriptor.sol` per edition](#7-a-descriptorsol-per-edition)
  - [Inside each descriptor there are two classes of data, with different natures:](#inside-each-descriptor-there-are-two-classes-of-data-with-different-natures)
  - [Deployment: abstract contract + final contract](#deployment-abstract-contract--final-contract)
- [8. Registering an edition (summary)](#8-registering-an-edition-summary)
- [9. Security — notes](#9-security--notes)


## MellonKey◉fs◉ 


MellonKey◉fs◉ is a development ecosystem created as a technical portfolio. It gathers the Proof of Concepts (PoCs) and the outcomes of an iterative design process originated in the ·sarocha· Project. During the research phase, the latter evolved toward an architecture that fit its final objectives and scope more efficiently, allowing MellonKey◉fs◉ to remain a valuable repository of solutions, resolved dilemmas, and use cases transferable to other projects.


## MellonKeyDecrypt — v2.1.0 — *The Narya Keepers*


Successor to *Mellon-Key-Simple* v1.0.0: same mint/verification logic, but the
deliverable file no longer lives in R2 — it is **encrypted with AES-256-GCM on Arweave**,
and a **portable Cloudflare Worker** fetches it (with gateway redundancy),
decrypts it on the fly, and delivers it solely to the verified SBT holder.

<br>

```
mellonkey-decrypt/
├── public/                        # Static site — hostable ANYWHERE
│   ├── index.html                 #   Mint of the 3 Narya Keepers (Base Sepolia)
│   ├── deliver.html               #   Certificate + decrypted download
│   ├── editions.json              #   SINGLE source of truth (worker imports it at deploy)
│   └── css/                       #   Project's own stylesheets
├── worker/                        # Portable worker (replaces functions/api/download.js)
│   ├── src/index.js               #   All the logic: signature, ownerOf, gateways, AES-GCM
│   ├── wrangler.toml              #   Deploy config
│   ├── package-lock.json          #   
│   └── package.json               #   ethers v6 (fetch transport, native to Workers)
└── README.md
```

---

## 1. Complete flow

1. The user mints the SBT (`claim()`) in `index.html` — Base Sepolia, 0.02 ETH, 1 per wallet.
2. In `deliver.html` they connect their wallet: the page looks up their tokenId on-chain
   (`Transfer 0x0 -> wallet` event + double check with `ownerOf()`).
3. They sign a message that binds **edition + tokenId + timestamp** (anti-replay, 15-min TTL).
4. The page sends `{wallet, signature, message, tokenId, edition}` to the worker
   (`delivery.apiBase` + `/api/download`).
5. The worker:
   - validates the signature's format and TTL,
   - recovers the address with `verifyMessage` and compares it to `wallet`,
   - queries `ownerOf(tokenId)` on the edition's contract via RPC,
   - downloads the **encrypted** file from Arweave following the gateway chain,
   - **decrypts** it (AES-256-GCM) with a key stored as a secret,
   - responds with the plain binary using `Content-Disposition: attachment`.
6. The browser saves the file using the edition's `downloadName`.

The key NEVER travels to the client: decryption happens entirely within the worker.

## 2. Arweave gateway redundancy (inside the worker)

`editions.json` no longer hardcodes any gateway: neither the deliverable file
(`arweaveTx` field) nor the preview image (`image` field) reference a
gateway — both use the `ar://TXID` scheme. The worker resolves the file's
gateway like this (and `index.html` uses the SAME chain, in the browser, to
resolve the card images):

| Order | Source | Configuration |
|-------|--------|---------------|
| 1st | Canonical gateway `https://arweave.net` | last-resort constant (the value takes precedence in `delivery.canonicalGateway`) |
| 2nd | Up to **3 tested fallback gateways** | `delivery.fallbackGateways` (empty ones are skipped) |
| 3rd | **Last resort**: API on Vercel (AR.IO SDK, list already filtered by performance) | `delivery.resolverApi` → the **first** one is taken |

**Images in the browser (`image: "ar://TXID"`)**: no browser understands
`ar://`, so `index.html` resolves each `<img>` on the client with the same order: canonical gateway → fallbacks → `resolverApi`. The mechanism is
a `src` swap on error (`onerror`), i.e., pure `<img>` loading, no CORS.
A legacy `https` URL in `image` keeps working as-is (backward
compatibility). It works from any hosting, including the clone on Arweave.

Robustness details already implemented:

- The Vercel resolver is queried **only** if canonical and fallbacks fail (lazy).
- Each attempt has its own timeout (`delivery.gatewayTimeoutMs`, default 10 s).
- A suspiciously small body (error pages with HTTP 200) is discarded.
- If a copy is obtained but **does not decrypt**, the chain continues with the next
  gateway; only when everything is exhausted is the error returned (502 if nothing was obtained,
  500 if it was obtained but does not decrypt).

## 3. Worker portability

The worker **does not depend on the page's hosting**:

- `editions.json` → `delivery.apiBase`:
  - `""` → same origin (the worker is on a route of the same domain, e.g., Pages).
  - `"https://mellonkey-decrypt.00k.workers.dev"` → the page can
    be hosted anywhere (Pages, localhost, **a clone served from Arweave**).
- CORS open by default (authentication is via wallet signature, there are no
  cookies or sessions). To restrict it: secret `ALLOWED_ORIGINS` with comma-separated
  origins.
- The worker's `GET /` responds with a JSON health check (active editions, whether a key is present).

> Regarding a clone on Arweave, the considerations are obvious but worth insisting on: in web2 `editions.json` is updated with each new edition ("Gil-Galad Narya Keeper", "Cirdan Narya Keeper", etc.), the worker is built and automatically redeployed, while on Arweave it will be frozen at a TXID. The **authoritative configuration for delivery** will be bundled into the worker at deploy time, and the worker will look for a TXID, not an `editions.json`. Each new edition requires updating the TXID and redeploying the worker so that the clone keeps serving that delivery.

## 4. Encrypting and uploading the file (per edition)

```bash
# Encrypts and generates a new key
node tools/encrypt-for-arweave.mjs my-file.png

# Or use your own key (openssl rand -hex 32)
node tools/encrypt-for-arweave.mjs my-file.png -k 3f2a...64-hex-chars...9b1c

# Verify integrity before uploading
node tools/encrypt-for-arweave.mjs --verify my-file.png.enc -k <key> my-file.png
```

Blob format (identical to what the worker decrypts with WebCrypto):

```
[ IV: 12 bytes ][ ciphertext: N bytes ][ GCM tag: 16 bytes ]
```

Then, for now manually (automation is being worked on):

1. Host the `.enc` on Arweave **without modifying it** (arweave.app, Irys, etc.).
2. In `editions.json`, load the edition:
   - `"contract"`: deployed SBT address,
   - `"arweaveTx": "ar://<TXID>"`,
   - `"active": true`.
3. Configure the worker secret (once per key):
   `wrangler secret put ***_KEY` with 64 hex chars.
4. `wrangler deploy` (from `worker/`) — the worker **imports editions.json at
   build time**, so after editing it, it must always be redeployed.

## 5. Worker and monorepo deployment

### 5.1 How `worker/` coexists with the Pages files in `public/`

They are two separate pieces, each published on its own:

- **`public/` is the SITE** (`index.html`, `deliver.html`, `editions.json`,
  `css/`). That is what gets published to the hosting — for example, on Cloudflare
  Pages, the project's output directory points to `public`.
- **`worker/` is the BACKEND** (replaces `functions/api/download.js`). It is NOT
  uploaded to the page's hosting: it is deployed separately with wrangler and lives at
  `https://mellonkey-decrypt.00k.workers.dev`. The page finds it via `delivery.apiBase` from `editions.json`:
  `""` = same origin, or the worker's full URL = page hosted wherever preferred: Cloudflare, Vercel, Arweave, etc.

**Node: installed ONCE on the system** (from nodejs.org, LTS version). Inside
`worker/` you do not "install node": `npm install` only runs there the first time,
downloading the project's dependencies (wrangler + ethers) to `node_modules/`
inside that folder.

**The relative structure matters**: the worker IMPORTS `../public/editions.json`
at build time, so `worker/` must remain in the same hierarchy as
`public/`. Since updates are performed via a GitHub repository, the cleanest approach is to keep both folders at the root, in the same hierarchy:


### 5.2 Commands

```bash
cd worker
npm install
wrangler login                        # first time
wrangler secret put ***_KEY           # required: 64 hex chars (32 bytes)
wrangler secret put ALCHEMY_API_KEY   # optional: premium RPC; without it, sepolia.base.org is used
wrangler deploy
```

Local testing → see section 5.3.

### 5.3 Testing locally (VSCode Live Server + wrangler dev) — and the 405 error

Two servers run in parallel, and `editions.json` must point to the worker:

```bash
cd worker
npx wrangler dev          # terminal 1: worker at http://localhost:8787
# terminal 2: serve the site (e.g., VSCode Live Server, or:)
npx serve public          # or any static server
```

In `editions.json` (the one served by the page):

```json
"delivery": { "apiBase": "http://localhost:8787" }
```

- With `""` it does NOT work locally because the html/css/js would go to the static server
  (Live Server :5500), which responds to a POST with **405 Method Not Allowed** and
  an empty body — and the browser shows `Unexpected end of JSON input`.
  If the POST does not appear in the console as `:8787/api/download`,
  `apiBase` is incorrect.
- Quick worker check: `http://localhost:8787/health` → JSON
  with `"keyConfigured": true` confirms that `.dev.vars` loaded the key.
- `.dev.vars` lives in `worker/`, next to `wrangler.toml`, in `KEY=value` format
  with no quotes or spaces.
- If `ALLOWED_ORIGINS` is set locally, add the page's origins:
  `http://127.0.0.1:5500,http://localhost:5500`.
- The SAME `editions.json` feeds both sides (the page reads it via HTTP and the
  worker imports it at build time): after editing it, `wrangler dev` reloads on its own.
- When moving to production: `apiBase` is set to the worker's URL
  (`https://mellonkey-decrypt.00k.workers.dev`); `""` only if the
  worker runs on the SAME domain as the page.

**If the POST already reaches the worker but the download fails (502 "Could not fetch
the encrypted file from any Arweave gateway")**, the `wrangler dev` terminal (`[gateway-chain] ...`) explains the causes:

- **timeout on some fallback** — that gateway is slow or down from the network.
  The chain continues with the next candidate.
- **HTTP 404 on a fallback** — that gateway does not have that TXID indexed (AR.IO
  ones do not replicate everything instantly). The chain continues.
- **`[ar-io-resolver]` timeout** — the Vercel API took longer than
  `gatewayTimeoutMs` (default 10 s). It is the last resort: if the canonical one
  worked, it is never queried. The timeout can be set with
  `"gatewayTimeoutMs": 20000` in `delivery`.

**If the gateways download the file but all of them say `undecryptable blob`**,
the envelope arrived fine and what fails is opening it. Two possible causes:

- **The key is not the same** one used to encrypt the file (or it has an extra/missing
  character when pasted). The key is a 64-hex-char K — with or without `0x`, the code
  handles both possibilities, foolproof against a distracted developer.
- **The container format**: since v2.0.3 the worker auto-detects the two supported
  formats and picks the correct one on its own:
  - `SRCHENC1` — toolchain (`gen-master-key.mjs` + `encrypt-artwork.mjs`):
    `["SRCHENC1" 8B][version 1B][reserved 3B][IV 12B][ciphertext+tag]`
  - `RAW` — project's `encrypt-for-arweave.mjs`: `[IV 12B][ciphertext+tag]`
  In both, the inside is the same AES-256-GCM (12-byte IV, GCM tag at the
  end). The worker terminal logs which container it detected
  (`[decrypt] container: ...`), so if a third one ever appears,
  it shows up instantly.

The final error reaching `deliver.html` lists the attempts with their
diagnosis (e.g., `Attempts: https://arweave.net -> HTTP 403; ...`).

## 6. Contracts on Sepolia: 100% on-chain metadata
Each SBT describes itself without depending on anything external. The tokenURI does not point to IPFS, to Arweave, nor to any server: it is a data:application/json;base64,... that the contract builds on its own, every time an explorer requests it. The SVG lives as an internal constant of the contract: the art is bytecode. If the site goes down, if the hosting disappears, the token keeps showing its own.

The contracts were coded keeping in mind the goal of generating a token that, as long as the network where it was minted exists (in this case Base Sepolia), will always preserve and display its metadata for name, description, image, and attributes.

This implies avoiding dependence on an external `metadata.json` which suffers from at least these potential failures:
- if incorporated by hardcoding gateways to be read by any platform, these can disappear, and if there is no one to update the contract setters, its content will be lost
- if incorporated with a Uri scheme (especially `ar://`, which for instance is not read by Metamask, although `ipfs://` is more widely accepted), they may not be readable by certain platforms
- if `IPFS://` is chosen —which is widely accepted by different platforms— it forces the creator to sustain expensive accounts to keep their files alive on the nodes. This developer understands that nothing guarantees that free services will continue to exist, not even tomorrow: a situation at odds with data permanence.
- if `ar://` is chosen, it is not readable by various platforms.

### 6.1 Inside that JSON, the critical data is guaranteed by construction:

- name — "<asset name> #<tokenId>", generated with the token's ID.
- description — assembled by the contract with the asset name, creator, and collection, plus the reveal URL.
- image / image_data — the art is exposed in two formats at once: as data:image/svg+xml;base64 (ERC-721 standard, what wallets, marketplaces, and explorers read) and as raw SVG in image_data (on-chain art convention). Two doors so that any viewer always finds the image. Additionally, the SVG itself prints name, collection, and creator.
- There is no state in which the token is shown without a name, without a description, or without an image: there is no external file that can be lost.

### 6.2 The art: controlled vectorization
The token's image is the vectorization of an image strictly related to the digital asset being delivered: the token's face anticipates what will be received upon claiming it.

To generate it, a Python script was programmed (`generador_svg_sol.py`) that does the full path: reduces and quantizes the image to 4 colors (K-Means), vectorizes it with vtracer, and delivers an SVG between 4 and 6 KB, a `file.svg` so the creator can observe how it will look. It also delivers a `file.txt` with rules: all attributes use single quotes, because the SVG is embedded raw inside the contract's JSON; before writing the final file, it simulates the exact JSON the contract would produce and validates it — if something does not check out, the .txt file is not generated.

That size is the operational key: the compiled contract takes up ~8 KB of bytecode against EIP-170's 24.5 KB limit (about 16 KB of margin), so deployment goes through without difficulties.

## 7. A `descriptor.sol` per edition
Each edition has its own descriptor: a dedicated Solidity contract that exclusively carries its identity and visual memory — the asset name, creator, collection, the vectorized SVG as a constant, and the Arweave TXIDs for that edition. Gil-Galad's descriptor does not share a single line of art with Cirdan's or Mithrandir's: three descriptors, three identities, three SVGs baked into three distinct bytecodes. The editions are impermeable to each other: regenerating or correcting the art of one does not touch the others.

### Inside each descriptor there are two classes of data, with different natures:

- Bytecode constants (art and identity): `digitalAssetName`, `creatorName`, `collectionName`, and the base SVG. Immutable, no storage cost, read directly from the code.
- Deployment data in storage: the Arweave TXIDs (animation manifest, encrypted file), which are passed in the constructor and have setters.
- The descriptor also assembles the complete token JSON: `name`, `description`, `image`, `image_data`, `animation_url`, `external_url`, `attributes`, and `encrypted_media` — the entire certificate, not just the image.

### Deployment: abstract contract + final contract

The descriptor is written as an abstract contract: it cannot be deployed on its own — the compiler rejects it. It is half of a piece that only exists joined to the other. On top of it, the contract that is actually deployed is written, which inherits from the descriptor and provides the token's machinery:

```solidity
contract SBTCirdan is SBTDescriptorCirdan, ERC721, Ownable, Pausable, ReentrancyGuard {
    // claim() with direct payment to the owner, soulbound enforcement, pause,
    // reentrancy control, mint dates registry...
}
```

Upon compiling, Solidity merges both into a single bytecode: descriptor (art + identity + metadata) and logic (mint, soulbound, security) are fused into the same deployed contract. There are no external calls to display the metadata, no proxies, no pieces that can become out of sync.

The practical consequence is elegant: the token machinery is identical across editions; what changes is the soul — and the soul travels in the descriptor. A single contract template, three unrepeatable identities.


## 8. Registering an edition (summary)

1. Deploy the SBT contract for each open edition. Import the address in editions.json.
2. `editions.json`: `contract`, `arweaveTx: "ar://<TXID of the encrypted file>"`, `image: "ar://<TXID of the image>"`, `active: true`, `downloadName`.
3. `wrangler deploy`.
4. If the site lives on another hosting (e.g., Arweave clone): also update the
   clone's `editions.json` (or re-upload it) so the card stops saying *Coming soon*.

## 9. Security — notes

- The `***_KEY` lives only in the worker's secrets; the client never receives it.
- The message signature is valid for 15 minutes and binds edition + tokenId: it works against replay.
- Delivery requires `ownerOf(tokenId) == wallet` on-chain on THAT edition's contract.
- The token is soulbound (`_update` blocks transfers): the key can be minted and burned.
- Noted roadmap: worker fallback (multiple mirror workers) and additional fallback
  logic for file delivery.

*Stay human... be kind* 🌿

--------------------------------------------------------
This is how each token renders inside a MetaMask wallet given the Fully On-Chain NFT architecture.

<img width="591" height="828" alt="2026-10-12 10_35_37-gilgalad-metamask" src="https://github.com/user-attachments/assets/9524c78d-727f-49b3-bf09-b81e14bd5001" />

<img width="586" height="814" alt="2026-10-12 10_36_54-cirdan-metamask" src="https://github.com/user-attachments/assets/e5f95438-0cbe-43cf-b233-6c0c610979ff" />

<img width="589" height="815" alt="2026-10-12 10_37_29-mithrandir-metamask" src="https://github.com/user-attachments/assets/9592eb78-ac07-4116-8be0-1711a3f180d3" />


