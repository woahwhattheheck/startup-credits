# BoTTube #1102 functional bug — manual Coinbase wallet accepts non-hex EVM addresses

Bounty: https://github.com/Scottcjn/rustchain-bounties/issues/1102

Target: `Scottcjn/bottube`

Verified source revision: `6af6b63f7a5a87353a30cd4552dc5c0e183a96af`

Exact source blob: `bottube_x402.py` @ `ddce70af0cad78ea5356032d900cdee8ffc34b1a`

## Summary

`POST /api/agents/me/coinbase-wallet` claims to validate a manually supplied Ethereum/Base address, but its validation checks only two properties:

```python
if not (manual_address.startswith("0x") and len(manual_address) == 42):
    return _jsonify({"error": "Invalid Ethereum address format"}), 400
```

That accepts any 42-character string beginning with `0x`, even when the remaining 40 characters are not hexadecimal. The accepted value is then persisted to `agents.coinbase_address` and returned with `ok: true` / `method: "manual_link"`.

A concrete invalid value that passes the current predicate is:

```text
0xgggggggggggggggggggggggggggggggggggggggg
```

It is length 42 and starts with `0x`, but `g` is not a hexadecimal digit, so it is not a valid EVM address.

## Reproduction

This is an authenticated API route. With any valid BoTTube agent bearer key:

```http
POST /api/agents/me/coinbase-wallet
Authorization: Bearer <valid-agent-api-key>
Content-Type: application/json

{"coinbase_address":"0xgggggggggggggggggggggggggggggggggggggggg"}
```

Current source behavior is deterministic:

1. `manual_address` receives the supplied string.
2. `manual_address.startswith("0x")` is `True`.
3. `len(manual_address) == 42` is `True`.
4. No hex-character validation is performed.
5. The route executes `UPDATE agents SET coinbase_address=?, coinbase_wallet_created=0 WHERE id=?`.
6. It commits and returns HTTP 200 with `ok: true`, the invalid address, and `method: "manual_link"`.
7. A later `GET /api/agents/me/coinbase-wallet` returns the persisted invalid value.

Minimal predicate check:

```python
bad = "0x" + "g" * 40
assert len(bad) == 42
assert bad.startswith("0x")
# The current route therefore accepts `bad`, although it is not hex.
```

## Expected

A manual Ethereum/Base address should be rejected with HTTP 400 unless it is exactly `0x` followed by 40 hexadecimal characters (or passes a stricter EVM address parser/checksum policy).

## Actual

A non-hex 42-character `0x...` string is accepted as a valid Coinbase/EVM wallet, persisted to the agent record, and reported back as a successful manual link.

## Impact

This lets an authenticated user put their account into a persistently invalid wallet state while BoTTube reports success. Any downstream flow that relies on `coinbase_address` as an EVM destination or identity can later fail at transaction/address parsing time, far away from the configuration request that introduced the bad value. It also makes the endpoint's `Invalid Ethereum address format` response misleading because the advertised format is not actually enforced.

## Distinctness from the previously reported x402 JSON-type bug

Issue #1102 history contains an accepted report for this endpoint's handling of non-object JSON and non-string `coinbase_address` values. This finding is different: the input is already a normal JSON object and `coinbase_address` is already a string. It passes the prior type-safety boundary and fails only because the string's character set is never validated.

The current `main` source still contains this prefix-plus-length-only check at the revision above.

## Suggested fix

Validate the full lexical form before writing the database row, for example:

```python
import re

if not isinstance(manual_address, str) or not re.fullmatch(r"0x[0-9a-fA-F]{40}", manual_address):
    return _jsonify({"error": "Invalid Ethereum address format"}), 400
```

If checksum enforcement is desired, use an EVM address parser/checksum utility instead of regex alone.

Regression cases should include:

- valid lowercase hex address -> accepted;
- valid uppercase hex digits after `0x` -> accepted if checksum is not required;
- one non-hex character (`g`, `_`, space, etc.) -> 400;
- too short / too long -> 400;
- non-string address -> 400 (existing type-safety regression boundary).

## Environment / evidence class

- API surface: BoTTube x402 wallet-link endpoint
- Evidence: exact current `main` source and deterministic predicate/database path
- Revision: `6af6b63f7a5a87353a30cd4552dc5c0e183a96af`
- Source blob: `ddce70af0cad78ea5356032d900cdee8ffc34b1a`
- Browser/device: not browser-specific; HTTP API behavior

No payout is assumed until maintainer acceptance.
