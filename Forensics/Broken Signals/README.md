# Broken Signals — hiCTF 2026

**Category:** Forensics  
**Challenge Author:** @archnexus707svg

## Challenge Description

Our signals intelligence team intercepted a file during a routine sweep, but it appears corrupted. The file cannot be opened by any standard application. Something is wrong with the file itself — can you repair it and recover the hidden message?

## Initial Analysis

After downloading and extracting `broken-signals.zip`, I found a file named `broken_signal.dat`.

First, I checked the file type:

```bash
file broken_signal.dat
```

The file was not recognized as a valid PNG image.

Since the challenge involved recovering a corrupted file, I decided to inspect its hexadecimal header.

```bash
xxd -l 32 broken_signal.dat
```

## Identifying the Problem

A valid PNG file always starts with the following 8-byte signature:

```text
89 50 4E 47 0D 0A 1A 0A
```

However, the provided file started with:

```text
00 00 00 00 0D 0A 1A 0A
```

This indicated that the first four bytes of the PNG signature had been overwritten.

## Repairing the PNG Header

To repair the file, I replaced the corrupted first four bytes with the correct PNG signature.

I used Python:

```python
with open("broken_signal.dat", "rb") as f:
    data = bytearray(f.read())

# Restore the PNG signature
data[:8] = bytes.fromhex("89 50 4E 47 0D 0A 1A 0A")

with open("repaired_signals.png", "wb") as f:
    f.write(data)

print("[+] PNG header repaired successfully!")
```

Run the script:

```bash
python3 solve.py
```

This generated a repaired image named `repaired_signals.png`.

## Recovering the Flag

After opening the repaired PNG image, I found the hidden message containing the flag.

## Flag

```text
hiCTF{h34d3r_r3p41r_m4st3r}
```

## Conclusion

This challenge demonstrated the importance of file signatures (magic bytes) in digital forensics.

Even when a file appears corrupted, inspecting its header with tools like `file`, `xxd`, and Python can help identify and repair the underlying issue.

**Key Takeaway:** Always check file magic bytes when investigating corrupted files.
