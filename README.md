# Qoin

**Single-file Ethereum wallet. Face ID is the key. Agent-first, human-signed.**

One HTML file. Host it anywhere. Open it on iPhone. Create as many wallets as you want — one per agent, one per purpose. Each wallet is a standard Ethereum address. Agents send transaction proposals as links. You confirm with Face ID.

No accounts. No passwords. No seed phrases. No backend. No recovery. One face unlocks every wallet you will ever create.

---

## The idea

Every AI agent that touches money today faces the same dead end: either it holds custody of your funds (you have to trust it), or it cannot spend at all. Neither is good enough for a world where agents run work on your behalf around the clock.

Qoin is the third path. The agent holds nothing. It knows your address, it knows what the wallet can afford, and when it needs to pay for something, it builds a transaction and sends you a link. You tap the link, Face ID confirms, the money moves. The agent never touches keys. You never type a password.

The wallet itself is a single HTML file. You can host it on GitHub Pages, your own domain, a USB stick, or any static host. There is no server, no database, no subscription. Read the source. Mirror it. Fork it. The file is interchangeable; your wallets are not.

Wallets are multi. One Face ID, many addresses — one per agent, one per envelope, as many as you want. Each is a silo. A compromised agent loses its own envelope, not the house.

---

## How it works

### Face ID becomes the private key

iOS 18+ Safari and modern Chrome support a WebAuthn feature called the **PRF extension**. When you register a passkey on Qoin, the iPhone's Secure Enclave creates a cryptographic key that never leaves the device. Every time you unlock that passkey with Face ID, you can ask it to compute `HMAC-SHA256(secret, salt)` for any 32-byte salt you supply. Same face, same salt, same output — always.

Qoin uses this as a deterministic key derivation function. For each wallet name you create, Qoin computes:

```
salt          = SHA-256("qoin-eth-v1:" + walletName)
privateKey    = passkey-PRF(salt)                      // 32 bytes, requires Face ID
publicKey     = secp256k1(privateKey)
address       = keccak256(publicKey) last 20 bytes
```

The private key exists in JavaScript memory only during the signing moment, and is zeroed after. No file stores it. No cloud holds it. The only thing that can reproduce it is your face plus your iPhone plus the wallet's name.

### Multi-wallet from one Face ID

Different wallet names produce different salts, which produce different keys, which produce different addresses. One passkey, unlimited wallets. You can name them anything: `atom`, `tako`, `main`, `experiments`, `pocket`, `savings`. The name is part of the derivation — change the spelling, you get a different wallet.

```
wallet "atom" → HMAC-SHA256(passkey, "qoin-eth-v1:atom") → 0xA1B2…
wallet "tako" → HMAC-SHA256(passkey, "qoin-eth-v1:tako") → 0xC3D4…
wallet "main" → HMAC-SHA256(passkey, "qoin-eth-v1:main") → 0xE5F6…
```

Wallet names live only on your device, in `localStorage`, for display. They are not published on-chain. Two Qoin users can both have a wallet called "atom" and their addresses will be completely different.

### Just an Ethereum address

Each Qoin wallet is a standard Externally Owned Account. No smart contract, no paymaster, no bundler. You pay your own gas in the chain's native token. Every wallet you make can be used on Ethereum mainnet, Base, or any other EVM chain — same address works on all of them, since the derivation is chain-agnostic.

Qoin ships with support for ETH (native) and USDC on both Ethereum and Base. Any other ERC-20 works via paste-contract-address.

---

## How agents use it

An agent does not register with Qoin. It has no account, no API key, no session. The protocol is the URL.

When an agent needs to move money from a wallet you have assigned to it:

1. It builds a transaction as a JSON object.
2. It encodes the JSON as `base64url`.
3. It sends you the link: `https://your-qoin-host/wallet.html#w=<walletName>&tx=<base64url>`
4. You tap the link on your iPhone.
5. Qoin shows you the proposal. Face ID. Broadcast.

### Proposal JSON format

```json
{
  "v": 1,
  "chain": "base",
  "to": "0xRecipient...",
  "value": "100000",
  "token": "USDC",
  "memo": "API subscription for October",
  "from": "0xYourAtomWallet..."
}
```

| Field | Required | Description |
|---|---|---|
| `v` | yes | Protocol version. Currently `1`. |
| `chain` | yes | `ethereum` or `base` (more chains coming). |
| `to` | yes | Recipient address. |
| `value` | yes | Amount in smallest units, as a decimal string. Wei for native ETH, raw for ERC-20. |
| `token` | no | Symbol (`USDC`) or full ERC-20 contract address. Omit for native ETH. |
| `memo` | no | Human-readable note. Shown on the approval screen. |
| `data` | no | Raw calldata for advanced contract calls. |
| `from` | no | Expected sender. Qoin refuses to sign if the derived address does not match. Strongly recommended. |

### What makes the link safe

The agent signs with its own private key (optional but recommended) and embeds no secret of yours. The link contains only what needs to be approved. If anyone intercepts it, they still cannot sign — only your face can. If the agent is compromised, the worst it can do is propose a bad transaction, which you will reject when you see it.

### Examples

Pocket money for an Atom agent:
```
https://qoin.io/wallet.html#w=atom&tx=eyJ2IjoxLCJjaGFpbiI6ImJhc2UiLCJ0byI6IjB4YWJjLi4uIiwidmFsdWUiOiIxMDAwMDAiLCJ0b2tlbiI6IlVTREMiLCJtZW1vIjoiQVBJIHN1Yi Oct","ZnJvbSI6IjB4WW91ckF0b20ifQ
```

Agent-to-agent settlement:
```
https://qoin.io/wallet.html#w=tako&tx=...  // Tako pays Atom, you approve
```

---

## How humans use it

Qoin works as a regular wallet too.

**First launch:** Open the HTML on your iPhone. One Face ID prompt creates a passkey. No sign-up, no email, no recovery questions.

**Create a wallet:** Tap `+`. Type a name ("main", "savings", "atom", whatever you'll remember). Face ID. An Ethereum address appears. Copy it. Fund it from an exchange or another wallet.

**Receive:** Every wallet shows its address on its detail page. Tap to copy. Any incoming transaction arrives normally — no action required from you.

**Send:** Tap a wallet. Tap Send. Pick chain, pick asset (ETH or USDC), enter amount and recipient, add a note. Face ID. Broadcast.

**Switch wallets:** Just tap another one in the list. Same Face ID unlocks all of them.

**On a new device:** Open Qoin on any iPhone signed into your Apple ID. Tap Set Up. Your iCloud Keychain syncs the passkey. Re-create each wallet by name. Same name produces the same address every time. Your wallets return.

**Wipe:** Clears `localStorage` on the current device. Does not touch your wallets on-chain — they are always recoverable from the same face + the same names.

---

## Security

### What Qoin protects against

- **Phishing sites copying the Qoin HTML.** Passkeys are bound to origin. A fake Qoin at a different domain cannot touch your real wallet.
- **Network interception of proposal links.** The link contains no secret. Only Face ID can sign.
- **Compromised agents.** An agent can propose but not sign. The worst case is a rejected proposal.
- **Malicious browser extensions stealing seed phrases.** Qoin has no seed phrase to steal.
- **Cloud breaches.** There is no cloud. Keys live only in Secure Enclave.
- **Losing one wallet to a bad actor.** Each wallet is siloed. An envelope compromised does not expose the rest.

### What Qoin does not protect against

- **Losing your iPhone and wiping iCloud.** There is no recovery. This is a wallet, not a bank.
- **Someone with your face and your unlocked iPhone.** Physical access plus biometric access defeats any personal wallet.
- **Bad proposals you approve without reading.** Qoin shows you the amount, recipient, chain, memo, and from-address before you sign. Read them.
- **Bugs in the HTML itself.** Audit the file before trusting it with real money. The commit hash is the thing to verify — publish it, pin it, check it.
- **Malicious mirrors hosting a modified file.** The domain is the wallet's identity; if you type a new domain, you are creating a new wallet there, not accessing your old one. Check the URL before you enter your name.

### The trust model

- You trust Apple's Secure Enclave to protect the passkey.
- You trust iOS Safari to implement WebAuthn PRF correctly.
- You trust the Qoin HTML file you loaded.
- You do **not** trust any agent, any server, any cloud, any person.

Everything in Qoin is designed so that the only things that can move your money are your face plus the correct HTML plus your device. Remove any one of the three and signing is impossible.

---

## Requirements

- iOS 18+ Safari, or a recent Chrome/Edge on a device with a platform authenticator (Face ID, Touch ID, Windows Hello)
- HTTPS origin (WebAuthn does not work over `file://` or plain HTTP)
- Internet connectivity for the public RPC read calls and for broadcasting transactions

---

## Hosting

Drop `wallet.html` on any HTTPS static host. GitHub Pages works. Vercel works. Your own server works.

The HTML file itself is freely replicable, but passkeys are bound to origin. A wallet created on `qoin.io` is not the same wallet as one created on `other.com` — the derived keys differ by domain, even with the same name. Pick a canonical domain and stick with it. That domain is the wallet's identity.

---

## Supported

- **Chains:** Ethereum mainnet, Base
- **Native assets:** ETH
- **Tokens:** USDC on both chains (more on request)
- **Arbitrary ERC-20:** by contract address

---

## Not included, by design

- No seed phrase export
- No recovery phrase
- No email signup
- No backend
- No analytics
- No tracking
- No customer support
- No "forgot password"

This is a wallet, not a bank.

---

## License

MIT

---

Built as part of [Labs](https://labs.vc).
