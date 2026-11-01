# No Bit Sings Alone — Audio Steganography & Finite-Field Cryptography Writeup

**Event:** hiCTF 2026  
**Category:** Steganography  
**Challenge:** No Bit Sings Alone  
**Flag:** `hiCTF{m4th3m4t!c5_g4v3_m3_4_l0t_Of_p4!n_8d52ae513}`

---

## 1. Challenge Overview

The challenge provides an audio file named:

```text
suspicious.wav
```

At first glance, the file appears to contain ordinary stereo audio. However, multiple hidden layers are embedded within the recording.

The complete solution involves:

1. Extracting encrypted information from a spectrogram.
2. Analyzing ultrasonic tones to recover sample-selection parameters.
3. Extracting hidden bits using a Hamming-style parity syndrome.
4. Reconstructing the AES key through an inverse Number Theoretic Transform (NTT) over GF(257).
5. Decrypting the final payload using AES-256-GCM.

The important observation is that the audio contains multiple layers, and each layer provides information required to decode the next one.

## 2. Initial Reconnaissance

I started by inspecting the WAV file.

The recording has the following properties:

| Property | Value |
|---|---|
| File | `suspicious.wav` |
| Format | WAV |
| Channels | 2 (Stereo) |
| Sample Rate | 44.1 kHz |
| Bit Depth | 16-bit |
| Duration | 59.8 seconds |

The file size matched the WAV header plus the audio samples, indicating that there was no additional appended payload.

I also checked the Least Significant Bits (LSBs) of both channels.

Approximately 50% of the LSBs were set to 1, which did not reveal any obvious direct LSB-encoded message.

However, spectrogram analysis revealed two suspicious regions.

### Interesting Audio Regions

| Time Range | Channel | Observation |
|---|---|---|
| 0.75–5 seconds | Both | Text-like patterns in the 7–18 kHz range |
| 54.3 seconds–end | Right | Burst of pure tones around 14–22 kHz |

These two regions suggested that the recording contained both a visible spectrogram message and a second hidden data layer.

---

## 3. Stage 1 — Extracting the Spectrogram Message

The first hidden layer was visible when inspecting the audio spectrogram.

The text appeared as a 5×7 pixel font.

Each character occupied approximately five time columns and seven frequency rows.

### 3.1 Character Layout

The approximate dimensions were:

- Character pitch: 0.1393 seconds
- Glyph width: 0.115 seconds
- Frequency-row spacing: two FFT bins
- FFT configuration: 512-point FFT
- Bin spacing: approximately 172.3 Hz

Each text line was vertically offset from the previous one.

I sampled the individual pixels and applied a threshold of -6 dB.

Then I matched the resulting cells against templates for 16 hexadecimal characters.

The matching produced exact results without ambiguity.

### 3.2 Recovered Spectrogram Text

```text
AES256GCM
N  = 99CB9C49E4C83527CE6F89D9
C0 = 8A845BA500DE25B749F1FF63CCFF
C1 = 9D2D9FD8D2614E19BCA42CC7E438
C2 = B31407AE896311C3AADCF9327610
C3 = 478444A09EF6FE3650DF7EBEC3B7
C4 = 1FF18CF0EB728DBFA791B7485D76
C5 = 7172AE8493A8715210B5
```

The recovered information identified:

- Encryption algorithm: AES-256-GCM
- Nonce: 12 bytes
- Ciphertext: 64 bytes
- Authentication tag: 16 bytes

The ciphertext and authentication tag were split across the `C0`–`C5` fields.

The total encrypted payload size was 80 bytes.

However, the AES-256 key was missing.

Therefore, the spectrogram provided the encrypted message and nonce, but not the actual decryption key.

---

## 4. Stage 2 — Analyzing the Ultrasonic Tone Burst

The next interesting region appeared near the end of the recording, exclusively in the right channel.

The tone burst began at sample:

```text
2,396,160
```

This value is exactly:

\[
585 \times 4096
\]

The burst ended at block 643.

Therefore, the recording contained 58 relevant blocks, each containing 4096 samples.

### 4.1 Extracting the Tone Frequencies

I divided the right-channel audio into 4096-sample blocks and calculated the FFT of each block.

For every block, I extracted the frequency bin containing the strongest peak.

This produced 58 integer values:

\[
k_0,k_1,\ldots,k_{57}
\]

Some initial values were:

```text
1781, 1583, 1463, 1625, 1757,
1543, 1409, 1801, ...
```

All 58 values were odd and fell within the range:

```text
1307–1801
```

### 4.2 Why Odd Numbers Matter

Each block contains 4096 samples.

Since:

\[
4096=2^{12}
\]

and every extracted step size \(k_j\) is odd:

\[
\gcd(k_j,4096)=1
\]

Therefore, multiplication by \(k_j\) modulo 4096 produces a permutation of all 4096 positions.

This was an important clue.

The tone frequencies were not simply audio artifacts. They encoded the sample-selection steps used to extract the hidden data from the left channel.

---

## 5. Stage 3 — Recovering the Hidden Bits Using Syndrome Extraction

The next layer was hidden in the LSBs of the left channel.

The hints suggested that the bits were encoded using a parity-based construction.

The key observation was that the 58 blocks could produce:

\[
58 \times 5 = 290
\]

bits.

The expected key representation could contain 32 elements from GF(257).

Each element requires 9 bits because:

\[
2^8=256<257<512=2^9
\]

Therefore:

\[
32 \times 9=288
\]

The 290 available bits were sufficient to encode the 288 required bits, with two remaining bits.

This suggested a Hamming-style syndrome extraction mechanism.

### 5.1 Hamming(31,26) Structure

A Hamming(31,26) code produces a 5-bit syndrome.

For each audio block \(j\), I extracted 31 sample LSBs.

The sample positions were determined by:

\[
p_i=((i+1)k_j)\bmod4096
\]

where:

\[
i=0,\ldots,30
\]

The parity-column assignment was:

\[
col_i=((i-k_j)\bmod31)+1
\]

This means the tone-derived step size also rotates the parity arrangement between blocks.

### 5.2 Computing the Syndrome

For every selected position, I checked the corresponding LSB.

If the LSB was set, its parity-column value was XORed into the syndrome.

Therefore:

\[
s_j=
\bigoplus_{\text{LSB}(L[p_i])=1}col_i
\]

Each block generated one 5-bit syndrome.

I discovered this extraction rule by brute-forcing families of possible start offsets, step rules, and parity-column arrangements.

### 5.3 Using GF(257) as a Validation Oracle

A key could potentially be represented as:

- 32 bytes (256 bits), or
- 32 elements of GF(257), requiring 288 bits.

The second representation matched the available syndrome bit count much better.

After concatenating the syndromes MSB-first and dividing them into 9-bit chunks, every decoded value had to be in the range:

\[
0\leq x\leq256
\]

A random 9-bit value has a probability of:

\[
\frac{257}{512}
\]

of falling in this range.

For 32 independent chunks, the probability that all are valid is:

\[
\left(\frac{257}{512}\right)^{32}
\]

which is approximately \(2^{-31.5}\).

This made the field-range condition a powerful filter for testing candidate extraction rules.

Only one non-degenerate candidate passed the validation.

### 5.4 Recovered GF(257) Elements

The extracted 32 values were:

```python
[
    205, 208, 154, 114, 115, 240, 123, 108,
    245, 172, 73, 192, 256, 177, 62, 202,
    153, 7, 77, 138, 175, 0, 4, 68,
    208, 32, 76, 169, 103, 36, 147, 216
]
```

All values were valid elements of GF(257).

However, these were not the AES key bytes yet.

They were the key represented in the transform domain.

---

## 6. Stage 4 — Recovering the AES Key Using an Inverse NTT

The next hint suggested that the finite-field values had passed through a frequency-like transform.

This pointed toward a Number Theoretic Transform (NTT).

An NTT is the finite-field equivalent of a Discrete Fourier Transform (DFT).

The extracted values represented the transformed key over:

\[
GF(257)
\]

The transform length was:

\[
N=32
\]

The primitive 32nd root of unity was:

\[
\omega=136
\]

### 6.1 Inverse NTT

To recover the original sequence, I applied the inverse NTT.

The inverse transform is:

\[
a_m=
N^{-1}
\sum_{i=0}^{N-1}
A_i\omega^{-im}
\pmod{257}
\]

where:

- \(A_i\) are the extracted transform-domain values.
- \(N=32\).
- \(\omega=136\).
- \(N^{-1}\) is the modular inverse of 32 modulo 257.

The required modular inverses are:

\[
\omega^{-1}\pmod{257}
\]

and:

\[
32^{-1}\pmod{257}
\]

Applying the inverse NTT produced 32 values, all smaller than 256.

These values could therefore be converted directly into bytes.

### 6.2 Recovered AES-256 Key

The resulting key was:

```text
8da8dca0aa79a8a2b1e1aef181e98e63
fcca7da26d49d6dbffdb461e3ecd544d
```

Combined into one 32-byte hexadecimal value:

```text
8da8dca0aa79a8a2b1e1aef181e98e63fcca7da26d49d6dbffdb461e3ecd544d
```

This was the missing AES-256 key.

---

## 7. Stage 5 — AES-256-GCM Decryption

With the key recovered, the remaining task was decrypting the encrypted payload.

The spectrogram had already provided:

| Parameter | Value |
|---|---|
| Algorithm | AES-256-GCM |
| Key size | 32 bytes |
| Nonce size | 12 bytes |
| AAD | None |
| Ciphertext | 64 bytes |
| Authentication tag | 16 bytes |

The nonce was:

```text
99CB9C49E4C83527CE6F89D9
```

The ciphertext and authentication tag were concatenated before decryption.

I used Python's `AESGCM` implementation from the `cryptography` library.

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

key = bytes.fromhex(
    "8da8dca0aa79a8a2b1e1aef181e98e63"
    "fcca7da26d49d6dbffdb461e3ecd544d"
)

nonce = bytes.fromhex(
    "99CB9C49E4C83527CE6F89D9"
)

ct = bytes.fromhex(
    "8A845BA500DE25B749F1FF63CCFF"
    "9D2D9FD8D2614E19BCA42CC7E438"
    "B31407AE896311C3AADCF9327610"
    "478444A09EF6FE3650DF7EBEC3B7"
    "1FF18CF0EB728DBFA791B7485D76"
    "7172AE8493A8715210B5"
)

pt = AESGCM(key).decrypt(nonce, ct, None)

length = int.from_bytes(pt[:2], "big")
flag = pt[2:2 + length].decode()

print(flag)
```

The AES-GCM authentication tag successfully validated, confirming that the recovered key and nonce were correct.

### 7.1 Plaintext Structure

The decrypted plaintext began with:

```text
00 32
```

This represents a big-endian length field:

\[
0x0032=50
\]

The next 50 bytes contained the flag.

The remaining bytes were random padding.

---

## 8. Complete Extraction Pipeline

The complete solving process can be summarized as:

```text
suspicious.wav
      |
      v
Spectrogram Analysis
      |
      v
AES-GCM Parameters
      |
      v
Ultrasonic Tone Analysis
      |
      v
58 Step Sizes
      |
      v
Hamming-Style Syndrome Extraction
      |
      v
288 Hidden Bits
      |
      v
32 GF(257) Elements
      |
      v
Inverse NTT (ω = 136)
      |
      v
32-Byte AES Key
      |
      v
AES-256-GCM Decryption
      |
      v
Length-Prefixed Plaintext
      |
      v
Final Flag
```

---

## 9. Final Flag

```text
hiCTF{m4th3m4t!c5_g4v3_m3_4_l0t_Of_p4!n_8d52ae513}
```

---

## 10. Key Takeaways

This challenge demonstrates how multiple mathematical and signal-processing techniques can be combined to hide cryptographic material inside an audio recording.

### Audio Forensics

- Inspecting WAV metadata and channel layouts.
- Detecting hidden text through spectrogram analysis.
- Identifying ultrasonic tone sequences using FFT.
- Extracting information from individual audio sample LSBs.

### Steganography

- Understanding parity-based bit extraction.
- Using Hamming-style syndrome calculations.
- Recovering hidden data through deterministic sample selection.
- Recognizing when statistical validation can identify the correct extraction rule.

### Finite-Field Cryptography

- Understanding arithmetic in GF(257).
- Converting bitstreams into finite-field elements.
- Recognizing NTT as a finite-field equivalent of the DFT.
- Reversing an NTT to recover the original AES key.

### Practical Lessons

- **The spectrogram is not always the final answer:** It may reveal only encryption parameters while the actual key is hidden elsewhere.
- **Frequency information can control data extraction:** The ultrasonic tones provided both sample-selection steps and parity-layout rotation.
- **Statistical constraints are useful:** Requiring all 32 decoded values to belong to GF(257) made it possible to distinguish the correct extraction rule from incorrect candidates.
- **Mathematics can hide ordinary bytes:** The AES key was encoded as finite-field elements and transformed through an NTT.
- **Authentication tags provide strong confirmation:** Successful AES-GCM authentication validated the reconstructed key and extracted ciphertext.

The most interesting part of this challenge was the connection between signal processing and finite-field mathematics.

What initially appeared to be an audio steganography problem turned into a combination of FFT analysis, parity syndromes, finite-field transforms, and authenticated encryption.

**The bits were hidden in the audio, but mathematics was the key to recovering them.**
