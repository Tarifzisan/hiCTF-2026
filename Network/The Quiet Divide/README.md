# The Quiet Divide — Network Forensics & Cryptography Writeup

**Event:** hiCTF 2026  
**Category:** Network 
**Challenge:** The Quiet Divide  
**Flag:** `hiCTF{c8c46fdbd6e5e4eacabfba53a156ff9ab10567f2}`

---

## 1. Challenge Overview

The challenge provides a network capture containing approximately 150 MB of data and 110,000 packets.

At first glance, most of the traffic appears to be noise:

- Health probes
- DNS queries
- Repeated 8 MB `AVLT2` archive uploads and downloads
- Multiple HTTP and QUIC sessions
- Diagnostic escrow transfers

However, the actual investigation revolves around a MIRRORBREAK custody and release workflow.

The relevant incident details are:

| Field | Value |
|---|---|
| Incident | `INC-3142` |
| Device | `ARCH-14` |
| Shard | `IDX-S9` |
| Workflow | MIRRORBREAK |
| Case Authority | `10.77.40.66:8122` |

The final flag is encrypted inside an AEAD final seal returned by the case authority through:

```http
GET /v8/audit
```

The challenge contains several decoys, so simply extracting interesting traffic is not enough. We must identify the authoritative records at every stage before reconstructing the final cryptographic material.

## 2. Reassembling Network Streams

I started by reassembling TCP streams using Python's `dpkt` library.

The capture was too large to inspect manually, so I also deduplicated requests using their request lines to reduce repeated traffic.

The investigation revealed several relevant services:

| Host | Role |
|---|---|
| `.33:8083` | Custody committee (C0–C4, threshold 3) |
| `.34:8084` | Device trust and attestation |
| `.35:8085` | Firmware distribution |
| `.36:8086` | Frame collector |
| `.37:4433` | QUIC observation collector |
| `.38/.39:8093` | Diagnostic escrow (slot A/B) |
| `.66:8122` | Case authority |
| `.69:8123` | Witness checkpoint |
| `.61:8110` | Archive vault |

The most important endpoints were the custody committee, device trust service, diagnostic escrow, QUIC collector, witness, archive vault, and case authority.

This established the infrastructure and helped separate operational noise from evidence relevant to the incident.

## 3. Firmware Analysis

The device `ARCH-14` downloaded:

```text
pulse-codec_7.7.0_amd64.elf
```

The firmware was transferred through four HTTP Range requests.

I extracted each response body and concatenated them in the correct order.

The resulting file had:

```text
Size: 14528 bytes
SHA-256: 0835dfce...
```

This hash matched both the signed firmware manifest and the device attestation.

Later, another device, `OPS-17`, downloaded firmware version 7.7.1 and submitted a qualification-passed report.

However, the policy explicitly stated:

```text
firmware_qualification_is_not_installation
```

This distinction is important: passing qualification does not prove that firmware was installed on the target device.

Therefore, the deployed firmware for `ARCH-14` was version **7.7.0**, not 7.7.1.

The 7.7.1 report was a decoy.

## 4. Recovering TLS Keylogs from Diagnostic Escrow

The next step was recovering the TLS session secrets.

`ARCH-14` submitted two diagnostic-share documents:

```text
mb-diagnostic-share/3
```

The documents were:

- Slot A: 760 bytes
- Slot B: 760 bytes

The two shares were XORed together to reconstruct the original plaintext NSS keylog.

```python
keylog = bytes(
    a ^ b
    for a, b in zip(shareA, shareB)
)
```

The reconstructed keylog's SHA-256 matched the declared `keylog_sha256`:

```text
fbe54727...
```

A second pair of diagnostic shares was submitted at `07:20:16` by:

```text
FleetDR-Gateway
```

This produced another keylog.

Both keylogs were needed because the capture contained two separate QUIC observation sessions.

## 5. Decrypting QUIC Traffic

With the recovered keylogs, I used Wireshark's command-line tool, `tshark`, to decrypt the TLS and QUIC traffic.

The relevant command was:

```bash
tshark \
  -o tls.keylog_file:KEYLOG_FILE \
  -d udp.port==4433,quic \
  -d udp.port==4434,quic
```

The decrypted traffic revealed two different observation sessions.

### Session comparison

| Keylog | Session | Observation |
|---|---|---|
| 1 | `.14 → .37` (`collector.asterion.test`) | Signed by `ARCH-14`, production domain `fleet-prod` |
| 2 | `.59 → .58` (`gateway-recovery...`) | Signed by `REC-14`, authority domain `dr-ops` |

The first session contained the production observation:

```text
device: ARCH-14
authority_domain: fleet-prod
```

Its signed hash was:

```text
60369e7f...
```

The second session contained a recovery observation associated with:

```text
device: REC-14
authority_domain: dr-ops
```

The signed policy established that:

- The production authority domain is `fleet-prod`.
- Recovery observations cannot finalize production.
- `recovery_may_finalize` is `false`.

Therefore, the observation from keylog 1 was authoritative.

The DR recovery session was a decoy.

## 6. Verifying the Custody Chain

The case authority's `/v8/audit` endpoint returned a five-event custody chain.

The sequence was:

```text
ROOT
  |
PREP 733
  |
COMMIT 733
  |
PREP 734
  |
COMMIT 734
```

Each event was signed by three of the five custody committee members:

```text
C0, C1, C2, C3, C4
```

The committee required a threshold of three signatures.

However, the replica log at host `.54` contained two additional events from epoch 735, submitted by:

```text
ctrl-west-2
```

At first, these additional events appeared to be a continuation of the custody chain.

But the policy required an:

```text
independently witnessed terminal checkpoint
```

The witness `W0` had frozen the authoritative chain at position 734.

The corresponding terminal event hash was:

```text
b34dc728...
```

Consequently, the epoch-735 branch was rejected because it was not part of the independently witnessed terminal checkpoint.

This terminal hash was particularly important because it was later used as the AEAD Additional Authenticated Data (AAD) when decrypting the archives.

## 7. Recovering the Evidence Key

The case authority provided a reference SDK through:

```http
GET /v10/sdk
```

An important detail in the SDK documentation was that it did not determine which records were eligible.

That responsibility remained with the investigator.

The evidence components required by the SDK were:

| Input | Source |
|---|---|
| `custody_records` | Five authoritative audit entries |
| `firmware` | Deployed 7.7.0 ELF |
| `attestation` | `ARCH-14` device attestation |
| `quic_observation` | Production observation from keylog 1 |
| `frame` | 32-byte PLS2 frame |

Using the SDK, I reconstructed the evidence and archive key.

```python
ev, codec = sdk.evidence_components(
    events,
    fw,
    att,
    qobs,
    frame
)

key = sdk.archive_key(
    ev,
    seal_body['archive_salt_hex']
)
```

The archive salt was:

```text
6d562eb7...
```

Next, I attempted to decrypt the archive using the witnessed terminal hash:

```python
pt = sdk.decrypt_archive(
    blob,
    key,
    terminal_hash
)
```

### Archive selection

The archive vault contained nine `AVLT*` uploads.

The older `AVLT2` archives were decoys.

The three `AVLT10` objects associated with the `finalize-batch` candidates successfully decrypted into SQLite databases.

An important observation was that only the `events_full` representation of the custody records produced successful archive decryption.

This made the AES-GCM authentication tag useful as a cryptographic validation mechanism: incorrect evidence combinations failed authentication.

## 8. Identifying the Correct Transaction

Each decrypted SQLite database contained four important tables:

- `transfers`
- `receipts`
- `fragments`
- `operations`

The `operations` table contained approximately 10,000 filler rows.

Instead of examining every row manually, I followed the release requirements exposed by:

```http
GET /v10/interop
```

The required checks were:

1. Scope
2. Signed freeze
3. Deployed observation
4. Durable receipt
5. Transaction ancestry

### 8.1 Scope

The transaction's `case_commit` had to match the witnessed terminal hash:

```text
b34dc728...
```

### 8.2 Signed Freeze

The `release_checkpoint` field had to match the witnessed checkpoint hash:

```text
5e4f78ad...
```

Three transactions passed this check.

### 8.3 Deployed Observation

The transaction's `observed_state` had to match the state derived from the deployed codec.

This eliminated transactions associated with incorrect observations.

### 8.4 Durable Receipt

The receipt disposition had to be:

```text
accepted
```

Transactions marked:

```text
queued
```

were not considered durable receipts.

### 8.5 Transaction Ancestry

Finally, two additional values had to match:

**Device nonce:**

```text
12060f75...
```

This was obtained from the device attestation challenge.

**Counter:**

```text
10450910
```

This was decoded from the PLS2 frame sequence.

Only one transaction satisfied all five release requirements:

```text
Transaction:
6dd9ec588c99...

Archive:
5a2c8ade...
```

This was the authoritative transaction.

## 9. Recovering the Final Seal

The remaining task was reconstructing the final cryptographic material.

The archive's `fragments` table contained XOR-masked fragments.

These fragments were masked using an HMAC-derived pad keyed by the codec state.

The SDK provided a helper to reverse this masking:

```python
mat = b''.join(
    sdk.unmask_fragment(codec, tx, offset, payload)
    for offset, payload, _ in frags
)
```

Before using the recovered material, I verified its integrity through the transaction material commitment:

```python
sdk.transaction_material_commit(
    mat,
    terminal,
    tx,
    codec
) == fragment_commit
```

The result was:

```text
True
```

This confirmed that the reconstructed transaction material matched the expected commitment.

Next, I derived the final seal key using:

- Reconstructed transaction material
- Evidence object
- Witness signature
- Final seal salt

```python
key = sdk.final_key(
    mat,
    ev,
    witness_signature,
    seal['salt_hex']
)
```

Finally, I decrypted the final AEAD seal:

```python
flag = sdk.decrypt_seal(
    b64(ciphertext_b64),
    key,
    nonce_hex,
    seal_body
)
```

I repeated the process across all 21 candidate transactions.

Only transaction `6dd9ec58...` successfully authenticated the final seal.

The recovered flag was:

```text
hiCTF{c8c46fdbd6e5e4eacabfba53a156ff9ab10567f2}
```

## 10. Final Flag

```text
hiCTF{c8c46fdbd6e5e4eacabfba53a156ff9ab10567f2}
```

**Verification note:** I did not verify the flag against the CTF scoring page. However, successful final-seal AEAD authentication for only the matching transaction provides strong cryptographic confirmation of the result.

## 11. Key Takeaways

This challenge combined network forensics, cryptographic analysis, and evidence-chain verification.

Some of the most important lessons were:

- **Traffic reconstruction:** Reassembling large PCAP captures and identifying relevant application-layer requests.
- **TLS and QUIC decryption:** Recovering NSS keylogs from XOR-split diagnostic escrow shares.
- **Firmware verification:** Distinguishing firmware qualification from actual deployment.
- **Digital evidence custody:** Following signed records and independently witnessed checkpoints.
- **Decoy identification:** Rejecting recovery-domain observations and unwitnessed replication branches.
- **Authenticated encryption:** Using AES-GCM authentication failures to reject incorrect archive candidates.
- **Transaction validation:** Combining scope, checkpoint, observation, receipt, and ancestry requirements.
- **Cryptographic reconstruction:** Recovering masked fragments and deriving the final seal key.

The biggest lesson was that cryptographic validity alone is not enough. A candidate may decrypt successfully, but the underlying evidence must also satisfy the authoritative custody, deployment, and release policies.

**The correct solution required reconstructing the entire chain of trust—not simply finding an interesting encrypted packet.**
