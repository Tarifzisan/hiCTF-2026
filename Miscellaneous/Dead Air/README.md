# Dead Air — hiCTF 2026 Writeup

**Category:** Miscellaneous  
**Challenge:** Dead Air  
**Flag:** `hiCTF{s1l3nc3_w4s_th3_m3ss4g3_6r3v}`

---

## Challenge Overview

The challenge provided two files:

- `kestrel_last_recording.wav` — Stereo audio recording (44.1 kHz, 47 seconds).
- `kestrel_logbook.7z` — An encrypted 7z archive.

The challenge description hinted that two operators communicated differently:

> "One spoke out loud, the other never made a sound you could hear."

This suggested investigating both audio channels independently rather than treating the recording as a single signal.

## 1. Splitting the Two Operators

The first step was to separate the left and right audio channels.

The channels had almost zero correlation (approximately 0), indicating that they contained independent signals.

### Left Channel — Morse Code

The left channel contained a 700 Hz tone that switched on and off.

By analyzing the timing:

- Dot: approximately 85 ms
- Dash: approximately 266 ms

The signal was identified as Morse code.

Decoding the Morse transmission revealed:

```text
KESTREL STATION OFF AIR KEY IS NIGHTJAR LISTEN WITH YOUR EYES
```

Important clues:

- `NIGHTJAR` — A cryptographic key.
- `LISTEN WITH YOUR EYES` — A hint to inspect the spectrogram.

### Right Channel — Spectrogram Message

The right channel initially sounded like random noise.

However, inspecting its spectrogram using `scipy.signal.spectrogram` with `nperseg=2048` revealed text hidden in the 5–13 kHz frequency range.

<text>Hidden message:</text>

```text
KSTR-7742-VXQ
```

Initially, this looked like another key. However, `NIGHTJAR` and `KSTR-7742-VXQ` served different purposes.

- `NIGHTJAR` was the Vigenère key.
- `KSTR-7742-VXQ` was the password for the encrypted archive.

## 2. Extracting the Encrypted 7z Archive

The archive used 7z header encryption, meaning even its filenames were hidden.

Since 7z extraction tools were unavailable in the environment, I manually analyzed the archive format.

### Header Analysis

The archive's start header pointed to an encoded header containing a single AES coder:

```text
06F10701
```

This identified the 7zAES encryption method.

The coder properties indicated:

- Algorithm: AES-256-CBC
- Key derivation: SHA-256
- Iterations: \(2^{19}\)
- Salt: None
- IV: 16 bytes

### Password Verification

The Morse-derived key `NIGHTJAR` failed to decrypt the archive header.

The spectrogram-derived password `KSTR-7742-VXQ` successfully decrypted it into a valid 7z structure.

The key derivation process was:

```python
key = sha256()

for i in range(1 << 19):
    key.update(
        pw.encode('utf-16le') +
        struct.pack('<Q', i)
    )

derived_key = key.digest()
```

The decrypted data was then processed using AES-CBC and decompressed using raw LZMA2:

```python
out = lzma.LZMADecompressor(
    lzma.FORMAT_RAW,
    filters=[
        {
            'id': lzma.FILTER_LZMA2,
            'dict_size': 1 << 26
        }
    ]
).decompress(dec[:unpack_size])
```

### Recovered Files

The decrypted archive header described one folder with two coders:

1. AES
2. LZMA2 (property `0x18`)

Two files were recovered:

| Filename | Size |
|---|---:|
| `logbook_page.png` | 880,401 bytes |
| `operator_codepad.png` | 1,090,865 bytes |

Both recovered files passed their CRC checks.

**Important parsing detail:** The first file's size was encoded using a three-byte 7z variable-length integer:

```text
CD 11 6F → 0x0D6F11
```

Treating it as a two-byte integer caused an incorrect data split and CRC failures.

## 3. Analyzing the Glyph Cipher

The extracted PNG files revealed another layer of encryption.

### Logbook Page

`logbook_page.png` contained a heading:

```text
LAST LOG ENTRY – 03:17 UTC
```

Below it was a grid containing 28 glyphs.

### Operator Codepad

`operator_codepad.png` contained a reference table of 37 glyphs.

The glyphs represented the following alphabet:

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_
```

Each glyph was assigned an index from 0 to 36.

### Glyph Matching

The logbook glyphs were visually similar to the reference glyphs, but had random variations in:

- Position
- Scale
- Rotation

To recover the ciphertext, I:

1. Cropped individual glyph cells from the logbook.
2. Binarized the extracted glyph images.
3. Compared each glyph against all 37 reference glyphs.
4. Used pixel-distance matching to identify the closest reference glyph.

Every match was unambiguous.

The recovered ciphertext was:

```text
59R_6L3Q9BYGBQ3QZAYZMP3QIZ92
```

## 4. Identifying the Cipher

The recovered ciphertext initially appeared meaningless.

However, two clues pointed toward a Vigenère cipher:

- The Morse transmission explicitly provided the key `NIGHTJAR`.
- The codepad contained 37 symbols indexed from 0 to 36.

This indicated a Vigenère cipher operating over a custom 37-character alphabet.

### Custom Alphabet

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_
```

Character mapping:

```text
A = 0
B = 1
...
Z = 25
0 = 26
...
9 = 35
_ = 36
```

Unlike the traditional Vigenère cipher, which operates modulo 26, this cipher required modulo 37.

## 5. Decrypting the Ciphertext

The decryption formula was:

\[
P_i = (C_i - K_{i \bmod 8}) \bmod 37
\]

Where:

- \(P_i\) = Plaintext character index
- \(C_i\) = Ciphertext character index
- \(K_i\) = Key character index
- Key = `NIGHTJAR`

Applying the formula produced:

```text
S1L3NC3_W4S_TH3_M3SS4G3_6R3V
```

The plaintext was recognizable as leetspeak.

The first part translated to:

```text
SILENCE WAS THE MESSAGE
```

The remaining characters were:

```text
6R3V
```

I verified the final four characters against the corresponding glyphs in the logbook to ensure that they were not matching errors.

## 6. Final Flag

The recovered plaintext used uppercase characters, but the accepted flag format required lowercase.

The final submission was:

```text
hiCTF{s1l3nc3_w4s_th3_m3ss4g3_6r3v}
```

<text color="success" weight="semibold">Flag successfully recovered!</text>

## 7. Rabbit Holes and False Leads

During the investigation, I also tested several possible hiding places that turned out to contain no useful information.

| Investigation | Result |
|---|---|
| Morse timing jitter | Random variation |
| Audio amplitude and phase | No hidden data |
| Audio LSB analysis | No useful information |
| Silent noise segments | Plain Gaussian noise |
| PNG chunks | No hidden payload |
| PNG LSB planes | No useful information |
| Channel differences | No useful information |
| Alternative Vigenère keys | Gibberish |

These checks helped narrow the investigation toward the actual multi-stage encoding mechanism.

## 8. Key Takeaways

1. **Analyze stereo channels independently:** Different channels can contain completely different communication methods.
2. **Use spectrograms:** Information that is inaudible in the waveform may become visible in the frequency domain.
3. **Distinguish keys from passwords:** A clue that provides a cryptographic key does not necessarily provide the archive password.
4. **Understand file formats:** Manual parsing of the 7z structure made it possible to recover encrypted contents without extraction utilities.
5. **Follow the reference alphabet:** The codepad's 37-character indexing revealed the correct Vigenère modulus.
6. **Verify character matching:** Checking ambiguous glyphs individually helped confirm the final plaintext.
7. **Check flag casing:** A correctly decoded flag can still be rejected if its required formatting is incorrect.

---

**Conclusion**

Dead Air was a multi-stage Miscellaneous challenge combining audio forensics, Morse decoding, spectrogram analysis, encrypted archive recovery, image processing, glyph recognition, and custom-alphabet cryptography.

The central lesson was hidden in the challenge's message:

**Silence was the message.**
