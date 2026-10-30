# Cold Beacon – hiCTF 2026 Write-up

**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Challenge:** Cold Beacon  
**Flag Format:** `hiCTF{}`

---

## Challenge Description

Evidence tag IR-2231-A contains a single Linux x86-64 ELF binary named `cold-beacon`.

The binary accepts one command-line argument:

```bash
./cold-beacon <key>
```

With the correct key, it reveals the configuration it carries. Otherwise, it displays:

```text
config sealed
```

The objective was to reverse engineer the binary, understand its key validation mechanism, and recover the correct key to unlock the hidden configuration.

## 1. Initial Triage

I started by identifying the binary:

```bash
file cold-beacon
```

The output confirmed that it was a 64-bit Linux ELF executable, dynamically linked and stripped.

Next, I inspected its strings:

```bash
strings cold-beacon | less
```

A few interesting strings immediately stood out:

```text
syncio.nim
cmdline.nim
IndexDefect
TracerPid:
/proc/self/status
```

These findings provided some useful clues:

- The `.nim` source-related names and runtime messages indicated that the binary was compiled using **Nim**.
- `TracerPid:` and `/proc/self/status` suggested an anti-debugging mechanism.
- The actual flag and the `config sealed` message were not directly visible through `strings`.

Since the binary was stripped and contained Nim runtime code, I decided to follow important data references rather than trying to understand the entire disassembly.

## 2. Locating the Main Logic

I used `objdump` to disassemble the executable:

```bash
objdump -d -M intel cold-beacon
```

Following references to `.rodata` helped identify the main routine at:

```text
0x404271
```

The main execution flow revealed several important conditions:

1. The program requires exactly one command-line argument.
2. The supplied key must be exactly `0x18` bytes (24 bytes) long.
3. Two separate key validators must succeed.
4. An anti-debugging check examines `/proc/self/status`.
5. If any validation fails, the program prints `config sealed`.
6. Otherwise, it derives a keystream, decrypts the embedded configuration, and verifies the resulting plaintext using a hash comparison.

The failure handler was located at:

```text
0x4041f9
```

It XORs 14 bytes stored at `0x405e00` with `0x5a`, producing the message:

```text
config sealed
```

The binary also contained a SHA-256 round-constant table in `.rodata` at:

```text
0x405e80
```

This suggested that SHA-256 was involved in the key processing and configuration verification.

## 3. Understanding the Key Validators

The most important part of the challenge was understanding how the program validated the supplied key.

### Validator A – `0x403cff`

This function checked 23 constraints using several tables stored in `.rodata`.

| Table | Address | Purpose |
|---|---|---|
| Target | `0x405fa0` | Expected byte values |
| Indices | `0x405fc0` | Key positions involved in each constraint |
| Count | `0x406050` | Number of indices used |
| Type | `0x406070` | Operation type |

After examining the branch logic, I identified the following operations:

| Type | Operation |
|---|---|
| 0, 3, 5 | XOR of selected key bytes |
| 1, 4 | Sum of selected key bytes modulo 256 |
| 2 | Product of two key bytes modulo 256 |

Essentially, the binary was not storing the complete key directly. Instead, it stored mathematical relationships that the correct key had to satisfy.

### Validator B – `0x403dc4`

The second validator introduced three additional relationships:

```text
k[9] + k[10]  = 0xa8
k[12] + k[13] = 0xe3
k[20]         = k[12] ^ k[4]
```

These constraints duplicated relationships already represented in Validator A.

This meant that the main challenge was solving the constraints extracted from the first validator.

## 4. Solving the Key Using Z3

Manually solving 23 constraints involving XOR, addition, and multiplication would be unnecessarily complicated.

Instead, I used the Z3 SMT solver.

I represented the 24-byte key as 8-bit bit-vectors:

```python
from z3 import *

K = [BitVec(f'k{i}', 8) for i in range(24)]

solver = Solver()

# Restrict every key byte to printable ASCII.
for k in K:
    solver.add(k >= 0x20, k <= 0x7e)

# Add the extracted constraints:
# XOR operations -> ^
# Sum operations -> +
# Product operations -> *
# Compare each result against its target byte.
```

Since these were 8-bit bit-vectors, arithmetic naturally operated modulo 256.

After encoding all extracted constraints, I enumerated the possible solutions.

Z3 returned exactly four candidates:

```text
Cpl$Bstc0n::unl0ck::7a3f
C`l4Bcdc0n::unl0ck::7a3f
C ltB#$c0n::unl0ck::7a3f
C0ldB34c0n::unl0ck::7a3f
```

The last candidate immediately looked meaningful:

```text
ColdBeacon::unlock::7a3f
```

However, the constraints alone were insufficient to uniquely identify the correct key.

All four candidates satisfied the extracted mathematical relationships.

## 5. Verifying the Correct Key

The binary performed an additional hash-based verification after the key validators passed.

This final check was important because it rejected candidates that satisfied the constraints but were not the actual decryption key.

I tested all four candidates against the executable:

```bash
./cold-beacon 'Cpl$Bstc0n::unl0ck::7a3f'

./cold-beacon 'C`l4Bcdc0n::unl0ck::7a3f'

./cold-beacon 'C ltB#$c0n::unl0ck::7a3f'

./cold-beacon 'C0ldB34c0n::unl0ck::7a3f'
```

Only the final candidate successfully decrypted the embedded configuration.

## 6. Recovering the Flag

Running the binary with the correct key:

```bash
./cold-beacon 'C0ldB34c0n::unl0ck::7a3f'
```

Produced:

```text
hiCTF{c0ld_b34c0n_cfg_uns34l3d_fr0m_th3_s4mpl3}
```

## Flag

```text
hiCTF{c0ld_b34c0n_cfg_uns34l3d_fr0m_th3_s4mpl3}
```

## Key Takeaways

- **Identify the compiler:** Recognizing Nim-specific strings helped explain the noisy runtime code.
- **Follow data references:** In stripped binaries, important logic can often be located by tracing references to `.rodata`.
- **Understand validation logic:** A binary may store mathematical constraints instead of the actual secret key.
- **Use SMT solvers:** Z3 makes solving complex combinations of XOR, addition, and multiplication much easier.
- **Account for ambiguity:** Satisfying the key constraints does not necessarily mean a candidate is correct. Additional verification may still be required.
- **Static analysis is powerful:** Understanding the binary's logic statically helped avoid unnecessary interaction with its anti-debugging mechanism.

---

**Conclusion:** Cold Beacon demonstrated how a secret key can be protected through multiple layers of validation rather than being stored directly. By identifying the Nim binary, reverse engineering its validation routines, and using Z3 to solve the extracted constraints, I recovered the correct key and successfully unlocked the hidden configuration.

**Event:** hiCTF 2026  
**Category:** Reverse Engineering
