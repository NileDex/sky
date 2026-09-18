# sky

Product Requirements Document
Tagline: Let AI shop. Only your wallet pays.

1. Problem
AI agents (browser agents, Codex-style coding agents, autonomous shopping assistants) are starting to act on websites on a person's behalf. But nobody wants to hand an AI agent direct control of their money — a prompt injection, a bug, or a misread instruction could redirect a payment to the wrong item, the wrong price, or the wrong recipient.

Today's agentic payment standards (like x402) mostly assume the agent itself holds funded keys and pays autonomously. That's fine for small, low-stakes, machine-to-machine payments. It is not fine for a real purchase a person cares about.

The gap: there's no simple pattern where an AI agent can browse and initiate a purchase, but a human's own wallet is the only thing that can ever actually authorize the money movement — with the blockchain itself refusing anything that doesn't match what was agreed.

2. Solution
A merchant website exposes its catalog and checkout actions as WebMCP tools, so any AI agent visiting the page can browse and create a purchase intent. That intent is turned into a signed, tamper-proof invoice by the merchant's backend. The invoice is shown as a QR code. Only the buyer's own wallet can scan, review, and confirm it. Payment settles on Arbitrum, where a smart contract checks the real transaction against the sealed invoice — right item, right price, right payer — before marking the order paid. Any mismatch reverts on-chain.

The AI can propose. It can never authorize or move money.

3. Goals
Prove that an AI agent can complete a real purchase flow on a website using WebMCP tools, with zero custom UI built for the AI.
Prove that the payment is cryptographically bound to what the merchant actually offered — the AI cannot alter price, item, or recipient after the fact.
Prove that payment can be restricted to a specific buyer wallet, enforced on-chain.
Demonstrate, live, that a prompt-injection attack on the AI does not result in a successful bad payment.
4. Non-Goals (out of scope for this build)
Multi-merchant marketplace or product discovery across sites.
Cross-chain payments or bridging.
The AI holding or managing a wallet/keys itself.
Refund/dispute automation (a stub or slide is enough).
A reusable, published SDK/npm package (may follow later, not part of the core demo).
Building a custom chat interface — the AI agent used in the demo is an existing agent (browser AI / Codex / Claude in Chrome), not something built for this project.
5. Users / Actors
Actor	Role
Buyer	The person who wants something purchased. Owns a wallet with testnet funds.
AI agent	An existing outside agent (not built by this project) that visits the merchant site and calls its exposed tools based on the buyer's request.
Merchant website	Exposes the product catalog and order actions as WebMCP tools; issues signed invoices.
Arbitrum smart contract	The referee — validates incoming payments against the sealed invoice; settles or reverts.
6. Core Flow
Buyer tells their AI agent what they want (e.g. "buy the $30 course").
AI agent visits the merchant site and calls get_products / check_price to find the item.
AI agent calls create_order — this is only a purchase intent: item, price, buyer's saved wallet address.
Merchant backend builds a signed invoice (EIP-712 typed data) containing: order ID, item, price, token, seller address, expected payer address, expiry.
Merchant backend renders the invoice as a QR code and calls pay_order/returns a payment link to the AI, which surfaces it to the buyer.
Buyer scans the QR with their wallet app. Wallet decodes and displays the invoice in plain terms. Buyer confirms.
Wallet submits a transaction to the Arbitrum settlement contract.
Contract checks: does msg.sender match expectedPayer? Does the amount/token/recipient match the sealed invoice? Has it expired?
Match → payment accepted, order marked paid, event emitted.
Mismatch → transaction reverts on-chain. No funds move except gas.
Merchant site listens for the on-chain event (via a watcher/websocket) and flips the order to "Paid" automatically, no refresh.
AI agent calls get_order_status, sees "paid," reports back to the buyer.
7. WebMCP Tool Specification
Exposed via document.modelContext.registerTool(...) on the merchant site (with polyfill/fallback for browsers without native support).

get_products()
Input: none
Output: list of { id, name, price, token, description }
check_price(product_id)
Input: { product_id }
Output: { price, token }
create_order(product_id, buyer_wallet)
Input: { product_id, buyer_wallet }
Output: { order_id, invoice_signature, qr_payload, expected_payer, expiry }
Side effect: backend generates and signs the EIP-712 invoice, stores the pending order.
pay_order(order_id)
Input: { order_id }
Output: { qr_code_url or payment_uri }
Returns the scannable payment request for the given order (may be combined with create_order's output for the demo).
get_order_status(order_id)
Input: { order_id }
Output: { status: "pending" | "paid" | "reverted" | "expired" }
8. Signed Invoice (EIP-712 payload)
{
  orderId: bytes32,
  item: string,
  price: uint256,
  token: address,        // e.g. USDC on Arbitrum testnet
  seller: address,
  expectedPayer: address, // the buyer's saved wallet
  expiry: uint256,
  nonce: uint256
}
Signed by the merchant's backend key. The signature travels with the QR payload so the smart contract (or a verifying step before settlement) can confirm the invoice hasn't been altered.

9. Smart Contract Requirements (Arbitrum, testnet)
settlePayment(invoice, signature) — payable/token-transfer function.
Verifies the merchant's signature over the invoice matches a known merchant key.
Requires msg.sender == invoice.expectedPayer. Reverts otherwise.
Requires amount and token transferred match invoice.price and invoice.token. Reverts otherwise.
Requires block.timestamp <= invoice.expiry. Reverts otherwise.
On success: transfers funds to invoice.seller, marks orderId as settled, emits OrderPaid(orderId, payer).
Prevents replay: an orderId can only be settled once.
10. Buyer Profile (minimal)
Name (display only)
Wallet address — saved once, reused as expectedPayer on every order created for this buyer.
No other profile data is required for the demo.

11. Demo Script
Setup (10s): Show the merchant page — one product, $30 course — and the buyer's saved wallet in profile.
Happy path (60s): Prompt the AI agent: "buy the $30 course." Watch it call the tools live. QR appears. Scan with testnet wallet, confirm. Page flips to "Paid" automatically. AI reports success.
Attack demo (60s): Reveal a hidden instruction planted in the product description trying to redirect the AI into creating an order paying a different address. Show the AI gets misled and creates a bad-looking order — then show the contract call reverts live, on-chain, because it doesn't match the sealed invoice/expected payer.
Close (15s): One slide — "The AI proposes. Your wallet authorizes. The chain enforces. It can never do the last two."
12. Success Criteria
End-to-end happy path completes without manual backend intervention.
Mismatched-wallet or tampered-invoice payment attempt visibly reverts on-chain in the demo.
No custom chat UI was built — an existing AI agent product performed the shopping.
Judges can articulate the security property back in one sentence after the demo.
13. Risks / Open Questions
WebMCP maturity: spec is a draft, recently moved from navigator.modelContext to document.modelContext, and no mainstream agent yet consumes WebMCP tools by default — mitigate with a polyfill and a pre-recorded fallback clip.
Wallet UX for EIP-712 on mobile: confirm the chosen wallet app renders the decoded invoice legibly before relying on it live.
Token choice (USDC vs USDG): verify testnet liquidity/availability before the day of the demo; default to USDC if uncertain.
Which AI agent to demo with: needs one that reliably discovers and calls WebMCP tools today — confirm compatibility ahead of time rather than assuming.
14. Build Order
Merchant page + hardcoded product data.
Five WebMCP tool functions wired directly into the page (no SDK abstraction yet).
Backend: order creation → EIP-712 invoice signing → QR generation.
Smart contract on Arbitrum testnet: signature check, expected-payer check, amount check, revert logic.
Event listener/websocket flipping order status to "Paid" live.
Wire up a real AI agent against the page; test the happy path end to end.
Build and test the prompt-injection attack scenario and confirm the on-chain revert.
(Only if time remains) Extract tool registration into a small reusable module and demo adding it to a second page.
