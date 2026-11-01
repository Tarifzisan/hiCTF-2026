# Cold Circuit — Reverse Engineering Writeup

**Event:** hiCTF 2026  
**Category:** Reverse Engineering  
**Challenge:** Cold Circuit  
**Flag:** `hiCTF{6e305f6a7461675f6e65656465642121}`

---

## 1. Challenge Overview

The challenge provides an ELF binary named `check`. Our goal is to reverse engineer its validation mechanism and recover the correct flag.

The binary implements a custom 20-round substitution-permutation network (SPN)-style cipher. It also includes an anti-debugging mechanism and opaque predicates to make analysis slightly more challenging.

Instead of brute-forcing the input, we can reverse the encryption process because all required cryptographic tables and round keys are embedded in the binary.

## 2. Initial Reconnaissance

First, identify the binary type and inspect its printable strings.

```bash
file check
strings check
```

Output:

```text
check: ELF 64-bit LSB pie executable, x86-64, dynamically linked, stripped
```

Some interesting strings:

```text
hiCTF{
no input
Bad format.
unreachable
Access granted.
Incorrect.
```

The imported functions also provide useful hints:

- `ptrace` — suggests anti-debugging protection.
- `memcmp` — suggests a comparison against expected data.
- `strlen` — likely used for input-length validation.
- `fgets` — suggests input can also be read from stdin.

Since the binary is stripped and contains no useful symbols, I proceeded with static analysis using `objdump`.

```bash
objdump -d -M intel check
```

## 3. Reverse Engineering the Validation Logic

### 3.1 Input Handling and Format Validation

The program accepts input through `argv[1]` or reads it from stdin.

At address `0x12d6`, the validation routine performs several checks:

1. Removes trailing `\r` and `\n` characters.
2. Requires the input length to be exactly `0x27` (39) bytes.
3. Checks that the input begins with `hiCTF{`.
4. Checks that the final character is `}`.
5. Parses the 32 hexadecimal characters inside the braces into a 16-byte state.

Therefore, the expected format is:

```text
hiCTF{32_hexadecimal_characters}
```

The resulting 16-byte value becomes the initial cipher state.

### 3.2 Anti-Debugging Mechanism

The binary uses `ptrace(PTRACE_TRACEME, ...)` to detect whether it is already being traced.

The relevant logic is equivalent to:

```c
errno = 0;

if (ptrace(PTRACE_TRACEME, 0, 1, 0) == -1 && errno == EPERM) {
    debugged = 1;
}
```

If a debugger is already attached, `ptrace` fails with `EPERM`.

The program responds by modifying the first byte of the first round key:

```c
roundkey[0][0] ^= 1;
```

This changes the encryption result and causes the correct flag to fail validation when the anti-debugging condition is triggered.

For solving, there is no need to execute the binary under a debugger. We can extract the original round keys directly from the file.

### 3.3 Opaque Predicates

The binary also contains two conditions designed to complicate control-flow analysis.

Let:

```c
v = state[0] + argc;
```

The first condition is:

```c
(v * v + 1) % 4 == 0
```

This condition is impossible because the square of an integer modulo 4 can only be 0 or 1.

Therefore:

```text
(v² + 1) mod 4 ∈ {1, 2}
```

The branch is unreachable.

The second condition is:

```c
(v * v) % 4 != 2
```

This condition is always true because a square modulo 4 can never equal 2.

Both conditions are distractions and do not affect the actual solution.

## 4. Understanding the Core Cipher

The main validation routine implements a custom 20-round substitution-permutation network.

Each round performs three operations on the 16-byte state.

```python
for r in range(20):

    # 1. Add round key
    for j in range(16):
        s[j] = (s[j] + RK[r][j]) & 0xff

    # 2. Substitute each byte
    for j in range(16):
        s[j] = SBOX[s[j]]

    # 3. Permute the state
    t = [0] * 16

    for j in range(16):
        t[PERM[j]] = s[j]

    s = t
```

The binary contains all the required data:

| Component | Address | Size |
|---|---:|---:|
| S-box | `0x2020` | 256 bytes |
| Permutation | `0x2120` | 16 bytes |
| Round keys | `0x2140` | 320 bytes |
| Target state | `0x2280` | 16 bytes |

Since the binary is a PIE executable, these are file offsets used to extract the embedded data.

The important observation is that every operation is reversible:

- Addition modulo 256 can be reversed using subtraction.
- The S-box can be reversed using an inverse S-box.
- The permutation can be reversed using the inverse mapping.

Therefore, instead of searching for the original input, we can decrypt the target state.

## 5. Recovering the Original State

### 5.1 Constructing the Inverse S-box

The original S-box maps an input byte to an output byte.

Because it is a permutation of all 256 possible byte values, we can construct its inverse:

```python
inv_sbox = [0] * 256

for i, value in enumerate(sbox):
    inv_sbox[value] = i
```

Now:

```text
INV_SBOX[SBOX[x]] = x
```

### 5.2 Reversing the Cipher

The original encryption performs these operations:

1. Add round key.
2. Apply S-box.
3. Permute bytes.

To reverse them, we must perform the inverse operations in the opposite order:

1. Undo permutation.
2. Apply inverse S-box.
3. Subtract round key.

The inverse round is:

```python
for r in range(19, -1, -1):

    # Undo permutation
    s = [s[perm[j]] for j in range(16)]

    # Undo substitution
    s = [inv_sbox[x] for x in s]

    # Undo round-key addition
    s = [
        (s[j] - rk[r * 16 + j]) & 0xff
        for j in range(16)
    ]
```

Starting from the target state and applying all 20 inverse rounds gives us the original 16-byte input.

## 6. Complete Python Solver

The following script extracts the cryptographic data directly from the binary and reverses the cipher.

```python
# Read binary
d = open("check", "rb").read()

# Extract embedded tables
sbox = d[0x2020:0x2120]
perm = d[0x2120:0x2130]
rk = d[0x2140:0x2140 + 320]
target = d[0x2280:0x2290]

# Construct inverse S-box
inv = [0] * 256

for i, v in enumerate(sbox):
    inv[v] = i

# Start from target state
s = list(target)

# Reverse all 20 rounds
for r in range(19, -1, -1):

    # Undo permutation
    s = [s[perm[j]] for j in range(16)]

    # Undo S-box
    s = [inv[x] for x in s]

    # Undo round-key addition
    s = [
        (s[j] - rk[r * 16 + j]) & 0xff
        for j in range(16)
    ]

# Reconstruct flag
flag = "hiCTF{" + bytes(s).hex() + "}"

print(flag)
```

Run it:

```bash
python3 solve.py
```

Output:

```text
hiCTF{6e305f6a7461675f6e65656465642121}
```

## 7. Flag Verification

Finally, execute the original binary with the recovered flag:

```bash
./check 'hiCTF{6e305f6a7461675f6e65656465642121}'
```

Output:

```text
Access granted.
```

I also verified the result by running the recovered state through the original forward encryption algorithm. The resulting state matched the embedded target constant.

The recovered hexadecimal bytes decode to:

```text
n0_jtag_needed!!
```

## 8. Final Flag

```text
hiCTF{6e305f6a7461675f6e65656465642121}
```

## 9. Key Takeaways

This challenge demonstrates several useful reverse-engineering concepts:

- **Static binary analysis:** Recovering program logic from a stripped ELF using disassembly.
- **Anti-debugging:** Understanding how `ptrace` can detect an attached debugger and alter program behavior.
- **Opaque predicates:** Identifying mathematically impossible or always-true conditions.
- **Substitution-Permutation Networks:** Recognizing a custom block-cipher-like construction.
- **Inverse transformations:** Reversing modular addition, S-box substitution, and byte permutation.
- **Static data extraction:** Recovering embedded cryptographic tables and round keys directly from the binary.

The main lesson is that encryption does not necessarily provide security when the complete key material, transformation logic, and expected output are embedded in the executable.

Rather than brute-forcing a 128-bit input, reversing the 20-round transformation gives us the answer directly.

---
