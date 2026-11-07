# RustChain API Walkthrough

This guide walks you through making your first API calls to RustChain.

## Base URL

```
https://rustchain.org
```

> ⚠️ **Note**: The node uses a self-signed certificate. Use `-k` or `--insecure` with curl.

---

## 1. Check Node Health

The simplest way to verify the node is running:

```bash
curl -k "https://rustchain.org/health"
```

**Response:**
```json
{
  "ok": true,
  "version": "2.2.1-rip200",
  "uptime_s": 223,
  "backup_age_hours": 19.7,
  "db_rw": true,
  "tip_age_slots": 0
}
```

---

## 2. Check Wallet Balance

Query any wallet balance using the `miner_id` parameter:

```bash
curl -k "https://rustchain.org/wallet/balance?miner_id=tomisnotcat"
```

**Response:**
```json
{
  "amount_i64": 0,
  "amount_rtc": 0.0,
  "miner_id": "tomisnotcat"
}
```

### Understanding the Response

| Field | Type | Description |
|-------|------|-------------|
| `amount_i64` | integer | Raw amount (in smallest units) |
| `amount_rtc` | float | Human-readable RTC amount |
| `miner_id` | string | The wallet ID queried |

---

## 3. Check Mining Eligibility

If you're mining, check your eligibility status:

```bash
curl -k "https://rustchain.org/lottery/eligibility?miner_id=tomisnotcat"
```

**Response (not eligible):**
```json
{
  "eligible": false,
  "reason": "not_attested",
  "rotation_size": 27,
  "slot": 13839,
  "slot_producer": null
}
```

**Response (eligible):**
```json
{
  "eligible": true,
  "reason": null,
  "rotation_size": 27,
  "slot": 13840,
  "slot_producer": "miner_name"
}
```

---

## 4. List Active Miners

```bash
curl -k "https://rustchain.org/api/miners"
```

**Response (truncated):**
```json
[
  {
    "miner": "stepehenreed",
    "hardware_type": "PowerPC G4",
    "antiquity_multiplier": 2.5,
    "device_arch": "powerpc_g4",
    "last_attest": 1773010433
  },
  {
    "miner": "nox-ventures", 
    "hardware_type": "x86-64 (Modern)",
    "antiquity_multiplier": 1.0,
    "device_arch": "modern",
    "last_attest": 1773010407
  }
]
```

---

## 5. Signed Transfer (Advanced)

To send RTC from one wallet to another, you need to create a signed transfer.

### Understanding Signed Transfers

RustChain uses Ed25519 signatures for transfers. You need:

1. **Your private key** (from `beacon identity new`)
2. **The transfer payload**
3. **Sign the payload with your key**

### Transfer Endpoint

```
POST /wallet/transfer/signed
```

### Transfer Payload Structure

```json
{
  "from_address": "RTC_sender_address",
  "to_address": "RTC_recipient_address",
  "amount_rtc": 100,
  "nonce": "unique_value",
  "chain_id": "rustchain-mainnet-v2",
  "public_key": "sender_ed25519_public_key_hex",
  "signature": "ed25519_signature_hex"
}
```

### Example (Python)

```python
import requests
import json
import nacl.signing
import nacl.encoding

# Load your private key
with open("/path/to/your/agent.key", "rb") as f:
    private_key = nacl.signing.SigningKey(f.read())

# Derive RTC address from public key
import hashlib
public_key_hex = private_key.verify_key.encode().hex()
from_address = "RTC" + hashlib.sha256(bytes.fromhex(public_key_hex)).hexdigest()[:40]

# Create the canonical message to sign.
# The node rebuilds these exact bytes, so four details are load-bearing:
#   * keys are from/to/amount/fee/memo/nonce (+chain_id), not the outer field names
#   * `amount` is a float on the wire: 100 is rebuilt as 100.0, and "100" signs
#     different bytes than "100.0" -> {"error": "Invalid signature"}
#   * `fee` is part of the current form and must be written as 0.0 (the node
#     rejects any nonzero signed fee with SIGNED_TRANSFER_FEE_UNSETTLED, and a
#     fee-less message is only accepted through the legacy form)
#   * `nonce` inside the message is a STRING, even when the body sends a number
transfer_msg = {
    "from": from_address,
    "to": "RTC_recipient_address",
    "amount": 100.0,
    "fee": 0.0,
    "nonce": "1234567890",
    "memo": "",
    "chain_id": "rustchain-mainnet-v2"
}

# Sign the canonical message
message = json.dumps(transfer_msg, sort_keys=True, separators=(",", ":")).encode()
signed = private_key.sign(message)
signature_hex = signed.signature.hex()

# Build outer payload (uses from_address/to_address/amount_rtc)
payload = {
    "from_address": from_address,
    "to_address": "RTC_recipient_address",
    "amount_rtc": 100,
    "nonce": "1234567890",
    "memo": "",
    "chain_id": "rustchain-mainnet-v2",
    "public_key": public_key_hex,
    "signature": signature_hex
}

# Send transfer
response = requests.post(
    "https://rustchain.org/wallet/transfer/signed",
    json=payload,
    # Do not disable TLS verification on a request that carries a signed transfer.
    # If the node uses a self-signed cert, point verify at its CA file instead:
    verify="/path/to/node-ca.pem"
)
print(response.json())
```

### Important Notes

- **RustChain Addresses**: Signed transfers require `RTC...` addresses (43 chars: `RTC` + 40 hex), not simple wallet IDs or ETH/SOL addresses
- **Private Key**: Your Ed25519 key from `beacon identity new`
- **Nonce**: Must be unique per transfer (use timestamp or counter)
- **Public Key**: Required in outer payload; must match the `from_address`
- **Chain ID**: Optional for backward compatibility, but recommended. If supplied, it is verified and included in the signed message.

---

## Common API Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `{"ok":false,"reason":"admin_required"}` | Endpoint requires admin | Use appropriate endpoint |
| `404 Not Found` | Wrong URL | Check endpoint path |
| Connection refused | Node down | Check node status |

---

## SDK Alternative

Instead of raw API calls, use the Python SDK:

```bash
pip install rustchain-sdk
```

```python
from rustchain_sdk import Client

client = Client("https://rustchain.org")

# Check balance
balance = client.get_balance("tomisnotcat")
print(balance)

# Get miners
miners = client.get_miners()
print(miners)
```

---

## Next Steps

- Explore the [RustChain GitHub](https://github.com/Scottcjn/Rustchain)
- Check [Bounties](https://github.com/Scottcjn/rustchain-bounties) for earning opportunities
- Join the community for help
