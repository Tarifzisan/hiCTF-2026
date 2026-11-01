# Hall of Mirrors 

**Event:** hiCTF 2026  
**Category:** Steganography  
**Challenge:** Hall of Mirrors  
**Flag:** `hiCTF{h1dd3n_numb3r_l4ttice_le4k}`

---

## 1. Challenge Overview

The challenge consists of multiple layers of steganography, reverse engineering, and cryptanalysis.

The complete solution requires five stages:

1. Recover a hidden key from an audio spectrogram.
2. Extract a bytecode payload using an LSB-parity steganography scheme.
3. Reverse engineer and emulate a custom virtual machine.
4. Perform an ECDSA Hidden Number Problem (HNP) attack using lattice reduction.
5. Recover the private key and use it to decrypt the final sealed flag.

The challenge initially looks like a simple audio steganography task, but the extracted payload reveals a much more interesting cryptographic problem.

## 2. Stage 1 — Finding the Key in the Spectrogram

The challenge provides a stereo WAV recording approximately 26 minutes long, sampled at 48 kHz.

The hint suggests:

> If you look at the recording instead of listening to it.

This immediately points toward spectrogram analysis rather than traditional audio playback.

### 2.1 Inspecting the WAV Structure

Before analyzing the audio, I inspected its RIFF/WAV structure.

An important discovery was that the file contained an additional `LIST` chunk.

Because of this, the audio data did not begin at the standard 44-byte offset.

Instead:

```text
Actual data offset = 216 bytes
```

This detail became important later when extracting the hidden carrier.

Using the wrong offset caused my initial brute-force attempts to fail.

### 2.2 Spectrogram Analysis

I generated a spectrogram and examined the image for unusual visual patterns.

A region containing text appeared approximately around:

- Overview image columns: 6900–7300
- FFT bins: 300–600
- Initial FFT size: 2048

However, the text was difficult to read directly.

To improve visibility, I processed both audio channels using:

1. A 1024-point FFT.
2. Row-mean subtraction.
3. Column-median subtraction.
4. Averaging of both channels.
5. Gaussian blur.

After normalization and enhancement, the hidden text became readable:

```text
MIRRORS1 SM64/.AP/PAR..
K=9C4F1E7A3B5???82
```

The text revealed important information:

- `MIRRORS1` identifies the carrier format.
- `SM64` refers to SplitMix64.
- `PAR` indicates parity-based extraction.
- The key is partially hidden.

Three hexadecimal digits were unreadable.

### 2.3 Recovering the Missing Key

The partially recovered key was:

```text
9C4F1E7A3B5???82
```

Since three hexadecimal digits were missing, there were:

\[
16^3 = 4096
\]

possible combinations.

I wrote a C brute-force program to test all candidates against the carrier extraction algorithm.

Initially, none of the candidates worked.

The problem was not the key itself—it was the incorrect WAV data offset.

After correcting the offset to 216 bytes, the brute-force successfully recovered:

```text
K = 0x9c4f1e7a3b50d682
```

This key was required for the next stage.

---

## 3. Stage 2 — Extracting the Hidden Carrier

The challenge uses a custom steganographic format called:

```text
MirrorStego v1
```

The carrier extraction process is based on SplitMix64 and LSB parity.

For every hidden bit, the algorithm performs the following operations.

### 3.1 Sample Position

Two values are generated from SplitMix64 using the recovered key.

The first value determines how far to advance through the audio samples:

\[
p \leftarrow p + 1 + (d_1 \bmod 4096)
\]

### 3.2 Parity Mask

The second value determines an 8-bit mask:

\[
w = (d_2 \gg 17) \mathbin{\&} 0xFF
\]

### 3.3 Extracting the Hidden Bit

The hidden bit is calculated using the parity of the masked sample byte:

\[
bit = \operatorname{popcount}
\left(
\operatorname{lowbyte}(S[p]) \oplus w
\right) \mathbin{\&} 1
\]

The extracted bits are packed most-significant-bit first (MSB-first).

### 3.4 Payload Structure

The extracted frame follows this structure:

```text
MIRRORS1 | length | CRC32 | payload
```

The resulting payload had:

```text
Payload size: 6266 bytes
CRC32:        0xe2046fb1
```

The CRC32 matched the expected value, confirming that the carrier extraction was successful.

The payload was not the final flag. Instead, it contained a custom virtual-machine program.

---

## 4. Stage 3 — Reverse Engineering the MVM1 Virtual Machine

The extracted payload begins with:

```text
MVM1
```

This identifies a custom virtual machine format.

The header contains:

| Field | Value |
|---|---:|
| Magic | `MVM1` |
| Code length | 6107 bytes |
| Data length | 143 bytes |
| Entry point | 0 |

I implemented a Python emulator based on the challenge's VM specification.

Three implementation details were particularly important.

### 4.1 Rolling Hash Register

The VM uses `R15` as a rolling hash.

It is updated on every instruction fetch, before decoding and executing the instruction.

This means the hash must be updated in the correct order. Updating it after execution would produce incorrect results.

### 4.2 Self-Modifying Code

The VM includes an instruction named:

```text
SMC
```

This instruction modifies the CODE memory region using an XOR-based xorshift keystream.

The keystream is seeded with:

```text
R[rs] ^ R15
```

As a result, the program decrypts portions of its own code while executing.

A static interpretation of the original code bytes would therefore be insufficient.

The emulator needed to support dynamic code modification.

### 4.3 Memory and Arithmetic

The VM maintains separate memory spaces:

| Memory | Size |
|---|---:|
| CODE | 64 KiB |
| DATA | 64 KiB |

All arithmetic operates modulo:

\[
2^{64}
\]

After implementing the instruction set, memory handling, rolling hash, and self-modifying behavior, the emulator successfully executed the program.

### 4.4 VM Execution Result

The program halted after:

```text
20,395 steps
```

It emitted:

```text
5083 bytes
```

The output was a `MIRRORS1` TLV block containing:

- secp256k1 curve parameters: \(n,p,a,b,G\)
- Public key \(Q\)
- Parameter \(L=12\)
- Sample count: 48
- 48 tuples, each containing three 32-byte values
- A 33-byte sealed blob

This revealed the actual cryptographic challenge.

---

## 5. Stage 4 — ECDSA Hidden Number Problem

The VM output contained ECDSA-related data using the secp256k1 elliptic curve.

The public key satisfies:

\[
Q=dG
\]

where:

- \(Q\) is the public key.
- \(d\) is the unknown private key.
- \(G\) is the curve generator.

The challenge provides 48 tuples containing three 32-byte values.

The parameter:

```text
L = 12
```

and the challenge title strongly suggest an ECDSA biased-nonce attack.

### 5.1 Understanding the Biased Nonces

In normal ECDSA signing, the nonce \(k_i\) should be unpredictable.

However, in this challenge, the top 12 bits of each nonce are zero.

Therefore:

\[
k_i < 2^{244}
\]

Define:

\[
B=2^{244}
\]

The nonces are significantly smaller than the full 256-bit range.

This leaks information about each nonce and makes a Hidden Number Problem (HNP) attack possible.

### 5.2 Deriving the ECDSA Equation

The ECDSA signing equation is:

\[
s_i \equiv k_i^{-1}(h_i+r_i d)\pmod n
\]

Multiplying both sides by \(s_i^{-1}\):

\[
k_i \equiv s_i^{-1}(h_i+r_i d)\pmod n
\]

Define:

\[
t_i=r_i s_i^{-1}\pmod n
\]

\[
u_i=h_i s_i^{-1}\pmod n
\]

Then:

\[
k_i \equiv t_i d+u_i\pmod n
\]

Since \(k_i\) is small, each equation constrains the unknown private key \(d\).

With enough biased signatures, these constraints can be combined into a lattice.

### 5.3 Constructing the Lattice

The attack uses 36 samples from the available 48 tuples.

Let:

\[
B=2^{244}
\]

and define the scaling factor:

\[
c=2^{12}
\]

To center the nonce interval, define:

\[
u_i'=u_i-\frac{B}{2}
\]

The lattice basis uses the following structure:

\[
\begin{bmatrix}
nc & 0 & \cdots & 0 & 0 & 0\\
0 & nc & \cdots & 0 & 0 & 0\\
\vdots & \vdots & \ddots & \vdots & \vdots & \vdots\\
0 & 0 & \cdots & nc & 0 & 0\\
t_1c & t_2c & \cdots & t_mc & \frac{Bc}{n} & 0\\
u_1'c & u_2'c & \cdots & u_m'c & 0 & \frac{Bc}{2}
\end{bmatrix}
\]

Here:

- \(m\) is the number of selected samples.
- \(n\) is the secp256k1 group order.
- \(B\) is the nonce upper bound.
- \(c\) is the scaling factor.
- \(t_i\) and \(u_i'\) encode the ECDSA relationships.

The lattice is constructed so that a sufficiently short vector reveals the private key.

### 5.4 Lattice Reduction with LLL

I used the LLL algorithm through SymPy's `DomainMatrix` implementation.

After reduction, the resulting basis contained a short vector whose final coordinate was:

\[
\pm \frac{Bc}{2}
\]

The second-to-last coordinate contained the information needed to recover \(d\).

The key idea is that the biased nonces produce unusually short lattice vectors compared with random full-size nonce values.

### 5.5 Identifying the Correct Tuple Field Order

One complication was that the tuple field order was not immediately known.

Each tuple contained three values, but their roles as \(r\), \(s\), and \(h\) were unclear.

There are only:

\[
3! = 6
\]

possible permutations.

I tested all six assignments and validated each recovered private key using the elliptic-curve public-key relationship:

\[
dG=Q
\]

Using 36 samples, the correct assignment was:

| Tuple element | Meaning |
|---|---|
| Element 0 | \(h\) |
| Element 1 | \(r\) |
| Element 2 | \(s\) |

This validation eliminated incorrect field-order assumptions.

### 5.6 Recovered Private Key

The lattice attack recovered:

```text
d = 0x278c1c0ea08921ba895d3d4c971e4e0a8c8b29d8016f42d9861fe49c6fa826c6
```

The recovered private key satisfied:

\[
dG=Q
\]

This confirmed that the HNP attack had recovered the correct ECDSA private key.

---

## 6. Stage 5 — Unsealing the Final Flag

The final VM output also contained a 33-byte sealed blob.

The recovered private key was used as input to the challenge's custom key derivation function.

### 6.1 Key Derivation

First, the private key was converted into a 32-byte big-endian representation.

Then it was passed through the datasheet's four-round mixer:

```text
Private key (32 bytes, big-endian)
                |
                v
        Four-round mixer
                |
                v
             key32
```

The result was a 32-byte encryption key.

### 6.2 Stream Cipher

The derived key was loaded as four little-endian 64-bit words.

The challenge then used an ARX-style double-round generator to produce a keystream.

Finally, the keystream was XORed with the sealed blob.

```text
Recovered private key
          |
          v
       KDF mixer
          |
          v
         key32
          |
          v
   ARX double-round
          |
          v
       Keystream
          |
          v
    XOR sealed blob
          |
          v
       Plaintext
```

The decrypted plaintext revealed the final flag.

---

## 7. Final Flag

```text
hiCTF{h1dd3n_numb3r_l4ttice_le4k}
```

---

## 8. Key Takeaways

This challenge combines several different technical disciplines into one investigation.

### Steganography

- Inspecting spectrograms instead of relying only on audio playback.
- Understanding WAV chunk layouts.
- Recovering incomplete cryptographic keys through constrained brute force.
- Extracting hidden information using deterministic PRNG-based parity.

### Reverse Engineering

- Implementing a custom virtual-machine emulator.
- Tracking instruction-fetch side effects.
- Handling self-modifying code.
- Reproducing 64-bit arithmetic and separate memory spaces.

### Cryptography

- Understanding ECDSA signing equations.
- Recognizing biased nonces as an HNP vulnerability.
- Constructing a lattice for private-key recovery.
- Using LLL to find short vectors.
- Verifying recovered private keys through elliptic-curve multiplication.

### Practical Lessons

- **Always inspect file structure:** The additional WAV `LIST` chunk shifted the audio data offset to 216 bytes. A wrong offset can silently invalidate later extraction attempts.
- **Tune spectrogram parameters:** A 2048-point FFT was difficult to read. A shorter 512–1024-point window and channel averaging improved visibility.
- **Implement VM semantics precisely:** Rolling hashes and self-modifying code make instruction execution order critical.
- **Do not assume tuple field order:** Testing all six permutations and verifying \(dG=Q\) provided a reliable way to identify the correct assignment.
- **Nonce bias is dangerous:** Even a small amount of predictable structure in ECDSA nonces can expose the entire private key.

The most interesting part of this challenge was how each stage revealed the next one. What started as an audio steganography problem eventually became a lattice-based cryptographic key recovery attack.

**A hidden message led to a VM, the VM led to biased ECDSA signatures, and the lattice attack unlocked the final flag.**
