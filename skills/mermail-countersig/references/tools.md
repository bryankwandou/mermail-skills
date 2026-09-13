# Countersig tool map

This skill owns no MCP tools. It uses tools owned by other skills, under their contracts, plus one bundled script.

## Mermail tools used

| Tool | Owner | Why | Approval |
| --- | --- | --- | --- |
| `list_mailboxes` | `mermail-administer-workspace` | Resolve the mailbox and its `public_id` | none |
| `search_emails` | `mermail-manage-inbox` | Find the change request, earlier authenticated mail, and earlier `[Countersig]` receipts. Pass `query` as a native JSON object; never stringify it | none |
| `get_email_context` | `mermail-manage-inbox` | Read the change request with sanitized, scan-gated thread context | none |
| `get_email` | `mermail-manage-inbox` | Read `sender_authentication` and date on a candidate trusted-channel message, `metadata_only` first | none |
| `send_email` | `mermail-compose-email` | Deliver the challenge to the trusted channel once. Requires `from` plus `html` and/or `text` | external-effect |
| `save_draft` | `mermail-compose-email` | Store the receipt in the mailbox. Uses the `body` string field | write-preview |
| `list_folders`, `create_folder`, `move_email` | `mermail-manage-inbox` | Park the change request in `Countersig Hold` or `Countersig Verified` | write-preview |

There is no MCP tool that manually assigns a custom label to one email, so Countersig records state with folders and receipts, not labels.

Free plan external sends are limited to 10 recipient units per minute, 50 per hour and 200 per day. One challenge uses one recipient unit.

## Handoff only

`get_paybox_connection` and `paybox_request_transfer` belong to `mermail-agent-wallet` and need full-profile OAuth. Countersig never calls them. It hands over a verified address, the proven cluster, and the receipt hash.

## Bundled script: `scripts/countersig.mjs`

Node.js 22 or newer, no dependencies. Reads public Solana JSON-RPC. Writes to chain only in `sign` and `anchor`, and both refuse `mainnet-beta` without `--allow-mainnet`.

| Command | Purpose | Chain write |
| --- | --- | --- |
| `challenge --claimed --channel [--prior] [--counterparty] [--cluster] [--ttl-hours 72] [--out]` | Create nonce, memo strings, signer link, and the email subject, text and HTML | no |
| `verify --challenge-file [--known a,b] [--rpc] [--out]` | Search the claimed and prior wallets for exact memos inside the window and return a verdict | no |
| `receipt --verdict-file --challenge-file [--out]` | Build a canonical receipt, its sha256, and draft text | no |
| `anchor --receipt-file --keypair [--cluster devnet]` | Write `countersig:v1:receipt:<sha256>` as a memo | yes |
| `sign --keypair --memo [--cluster devnet]` | Payee-side helper for testing; only sends `countersig:v1:` memos | yes |
| `keygen --out` | Throwaway devnet keypair for rehearsal | no |

Exit codes: `0` verified or success, `2` pending, `3` hard stop, `1` usage or network error.

### Verdicts

| Verdict | Meaning | Payable |
| --- | --- | --- |
| `VERIFIED_CONTINUITY` | Claimed wallet signed the nonce and the prior wallet endorsed it | yes, on the proven cluster |
| `VERIFIED_CHANNEL` | First contact; claimed wallet signed the nonce | after `coolOffUntil` or explicit user override |
| `PENDING` | A required memo has not landed; `missing` lists which | no |
| `MISMATCH` | The prior wallet endorsed a different address for this nonce | no, hard stop |
| `LOOKALIKE` | Claimed wallet shares first and last four characters with a different known wallet | no, hard stop |
| `EXPIRED` | Window closed before every required proof landed | no |
| `UNCHANGED` | Claimed equals prior | not a change |
| `INVALID_ADDRESS` / `INVALID_INPUT` | Malformed input | no |

### What the verifier checks

- Transaction succeeded (`meta.err` is null).
- The address is a signer of that transaction.
- An SPL Memo instruction (`MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr`) equals the expected memo exactly.
- Block time is between `issuedAt` minus 5 minutes and `expiresAt`.
- The newest 200 signatures of each wallet are scanned in full, so a conflicting endorsement is found even when a correct one also exists.
