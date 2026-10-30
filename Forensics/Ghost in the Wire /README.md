# Ghost in the Wire — HiCTF 2026

**Category:** Forensics  
**Author:** @archnexus707svg

## Challenge Description

Our network monitoring team detected anomalous ICMP traffic during a routine packet capture. Something is being exfiltrated through the wire — but the data blends in with normal network noise. Analyze the capture and extract the hidden message before the ghost disappears.

**File:** `ghost_traffic.pcap`

## Solution

### 1. Inspect the PCAP file

First, I checked the capture file metadata using `capinfos`.

```bash
capinfos ghost_traffic.pcap
```

The capture contained:
- 30 packets
- 1,857 bytes
- 52 seconds capture duration

Next, I inspected the packets using `tshark`.

```bash
tshark -r ghost_traffic.pcap
```

Most packets were ICMP Echo Requests. However, packets 3–28 stood out because they were sent continuously from `10.13.37.1` to `10.13.37.2`.

### 2. Extract ICMP payloads

I used the following command to extract important ICMP fields, including packet numbers, source and destination IPs, sequence numbers, and payload data.

```bash
tshark -r ghost_traffic.pcap -Y "icmp" \
-T fields \
-e frame.number \
-e ip.src \
-e ip.dst \
-e icmp.type \
-e icmp.seq \
-e data.data
```

The output revealed something interesting: packets 3–28 contained exactly one byte of hexadecimal payload data each.

For example:

| Packet | Hex | ASCII |
|---|---|---|
| 3 | `68` | h |
| 4 | `69` | i |
| 5 | `43` | C |
| 6 | `54` | T |
| 7 | `46` | F |
| 8 | `7b` | { |
| 9 | `31` | 1 |

This indicated that the ICMP payloads were being used to transmit a hidden message.

### 3. Decode the hidden message

Since each packet contained a hexadecimal ASCII character, I extracted the payloads and converted them back into readable text.

```bash
tshark -r ghost_traffic.pcap \
-Y "icmp.type == 8 && ip.src == 10.13.37.1" \
-T fields -e data.data | tr -d '\n' | xxd -r -p
```

The command returned:

```text
hiCTF{1cmp_3xf1l_d3t3ct3d}
```

## Flag

```text
hiCTF{1cmp_3xf1l_d3t3ct3d}
```

## Conclusion

The challenge used ICMP Echo Request packets to hide data inside their payloads. By inspecting the packet sequence and extracting the hexadecimal payload bytes, I was able to reconstruct the original message.

**Key takeaway:** ICMP traffic should not always be considered harmless. Even simple Echo Requests can be abused as a covert channel for data exfiltration.
