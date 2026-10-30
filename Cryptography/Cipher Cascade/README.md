# Cipher Cascade — hiCTF 2026 Writeup

**Category:** Cryptography  
**Challenge:** Cipher Cascade  
**Author:** @archnexus707svg  
**Flag:** `hiCTF{l4y3r_by_l4y3r_unw1nd}`

## Challenge Description

An encrypted transmission was intercepted from a rogue operator. The message was encrypted using multiple layers of encoding.

The challenge provides the following hint:

> The outermost layer is reversed.

The encryption layers, from outermost to innermost, are:

`Reverse → Base64 → Hex → ???`

Our task is to reverse the operations and recover the original flag.

## Solution

### Step 1: Reverse the Encoded String

Since the outermost layer is Reverse, we first reverse the entire encoded message.

```python
encoded = "=Q2NxcTM2EzMhZTM2gjNmVTN2MzMjZDNzkzNmVzY2YmNmVTN2MzMjZDNzkzNidzM1cDNwUjN3UzN"

reversed_text = encoded[::-1]
print(reversed_text)
```

Output:

```text
NzU3NjUwNDc1MzdiNzkzNDZjMzM2NTVmNmY2YzVmNzkzNDZjMzM2NTVmNjg2MTZhMzE2MTcxN2Q=
```

### Step 2: Base64 Decoding

Next, we decode the reversed string using Base64.

```python
import base64

decoded = base64.b64decode(reversed_text).decode()
print(decoded)
```

Output:

```text
75765047537b79346c33655f6f6c5f79346c33655f68616a3161717d
```

The output looks like hexadecimal data.

### Step 3: Hexadecimal Decoding

We convert the hexadecimal string into readable text.

```python
hex_decoded = bytes.fromhex(decoded).decode()
print(hex_decoded)
```

Output:

```text
uvPGS{y4l3e_ol_y4l3e_haj1aq}
```

The result still does not look like a valid flag. However, the prefix `uvPGS` suggests another substitution cipher.

### Step 4: ROT13 Decoding

We identify the final layer as ROT13.

ROT13 shifts every alphabetical character by 13 positions while leaving numbers and special characters unchanged.

```python
import codecs

flag = codecs.decode(hex_decoded, "rot_13")
print(flag)
```

Output:

```text
hiCTF{l4y3r_by_l4y3r_unw1nd}
```

## Complete Python Solver

The following script automates the entire decoding process:

```python
import base64
import codecs

encoded = "=Q2NxcTM2EzMhZTM2gjNmVTN2MzMjZDNzkzNmVzY2YmNmVTN2MzMjZDNzkzNidzM1cDNwUjN3UzN"

# Layer 1: Reverse
step1 = encoded[::-1]

# Layer 2: Base64
step2 = base64.b64decode(step1).decode()

# Layer 3: Hexadecimal
step3 = bytes.fromhex(step2).decode()

# Layer 4: ROT13
flag = codecs.decode(step3, "rot_13")

print("[+] Recovered Flag:", flag)
```

**Output:**

```text
[+] Recovered Flag: hiCTF{l4y3r_by_l4y3r_unw1nd}
```

## Flag

```text
hiCTF{l4y3r_by_l4y3r_unw1nd}
```

## Conclusion

This challenge demonstrates how multiple encoding layers can be combined to hide a message.

The key to solving it was following the reverse order of the applied transformations:

1. Reverse the encoded string.
2. Decode Base64.
3. Decode hexadecimal.
4. Apply ROT13.

The important lesson is to inspect the output after every decoding step rather than assuming that one successful decoding reveals the final plaintext.

**Challenge solved!**
