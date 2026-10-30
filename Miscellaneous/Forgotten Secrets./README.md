# Forgotten Secrets – hiCTF 2026 Write-up

**Category:** Miscellaneous  
**Difficulty:** Easy  
**Points:** 78  
**Challenge Author:** @archnexus707  
**Flag Format:** `hiCTF{}`

---

## Challenge Description

A binary dump was recovered from a compromised server's memory. The flag is obfuscated and cannot be found through a simple string search.

The challenge hint suggests looking for patterns, markers, and transformations that could hide readable text in plain sight.

**Provided File:** `forgotten_secrets.bin`  
**File Size:** 262,144 bytes  
**SHA-256:** `a05e06237e0b614e3a6e37159995eed0c222aa5c07ccb745ddcbe4e0561494cd`

## 1. Initial Triage

First, I checked the file type:

```bash
file forgotten_secrets.bin
```

The output identified the file as an ELF with an unknown architecture. However, this was a false identification caused by random bytes at the beginning of the dump accidentally matching an ELF magic signature.

Next, I searched for readable strings:

```bash
strings forgotten_secrets.bin | less
```

Several realistic-looking server artifacts appeared, including:

- `/etc/shadow`
- `root:x:0:0:root:/root:/bin/bash`
- Bearer tokens
- SSH public keys
- Session cookies
- SQL queries

There were also some obvious decoys:

```text
This is not the flag you are looking for
flag_not_here_keep_looking
base64_encoded_nothing_useful_here
```

The Base64-looking strings did not reveal anything useful, so I continued investigating the binary structure rather than focusing on the decoys.

## 2. Identifying the Markers

While examining the extracted strings, one unusual pattern caught my attention:

```text
CAFEBABEDEADBEEF
```

These values looked like deliberate markers rather than ordinary server artifacts.

I searched for both markers:

```bash
strings forgotten_secrets.bin | grep -E 'DEADBEEF|CAFEBABE'
```

Both ASCII markers appeared repeatedly throughout the dump.

Looking at the surrounding binary data revealed a consistent structure:

```text
DEADBEEF | 27 bytes of non-printable data | CAFEBABE
```

This suggested that the actual hidden payload was stored between two recognizable delimiters.

## 3. Analyzing the Hidden Payload

The extracted payload was 27 bytes long.

Considering the expected flag format:

```text
hiCTF{...}
```

a 27-byte payload was a useful clue.

Since the data was short and appeared consistently between the markers, I suspected a single-byte XOR transformation.

Single-byte XOR is straightforward to brute-force because there are only 256 possible keys (`0x00`–`0xFF`).

## 4. Writing the XOR Brute-Force Script

I used Python to extract every payload between `DEADBEEF` and `CAFEBABE`, then tested all 256 possible XOR keys.

```python
import re

with open("forgotten_secrets.bin", "rb") as f:
    data = f.read()

for match in re.finditer(
    rb'DEADBEEF(.{0,60}?)CAFEBABE',
    data,
    re.S
):
    payload = match.group(1)

    for key in range(256):
        decoded = bytes(b ^ key for b in payload)

        if b'hiCTF' in decoded:
            print(
                f"Offset: {match.start()}"
                f" | Key: 0x{key:02x}"
            )
            print(f"Decoded: {decoded.decode(errors='replace')}")
```

## 5. Recovering the Flag

The brute-force script revealed that XOR key `0x42` successfully decoded the payload.

```text
Key: 0x42
```

The decoded content was:

```text
hiCTF{m3m0ry_dump_d1g_d33p}
```

Interestingly, the same encoded payload was embedded 22 times throughout the binary dump, and every occurrence produced the same flag.

This confirmed that the hidden data was deliberately repeated rather than appearing as an isolated accidental match.

## Flag

```text
hiCTF{m3m0ry_dump_d1g_d33p}
```

## Key Takeaways

- **Don't rely only on `strings`:** Readable strings may contain intentional decoys.
- **Look for structural patterns:** Repeated markers such as `DEADBEEF` and `CAFEBABE` can reveal hidden data boundaries.
- **Inspect payload length:** The size of an extracted blob can provide useful clues about its encoding or encryption.
- **Try simple XOR brute-force:** A single-byte XOR cipher has only 256 possible keys, making exhaustive testing practical.
- **Verify repeated occurrences:** Identical decoded results across multiple locations increase confidence in the finding.

## Tools Used

- Linux CLI
- `file`
- `strings`
- `grep`
- Python 3
- Regular Expressions (`re`)
- Single-byte XOR brute-force

---

**Conclusion:** The flag was hidden inside repeated binary structures, protected by a single-byte XOR transformation. By identifying the delimiters and brute-forcing the XOR key, I successfully recovered the flag.

**Challenge:** Forgotten Secrets  
**Event:** hiCTF 2026
