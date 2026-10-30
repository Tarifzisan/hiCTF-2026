# The Foxglove Trace – hiCTF 2026 Write-up

**Category:** Miscellaneous  
**Difficulty:** Easy  
**Challenge:** The Foxglove Trace  
**Flag Format:** `hiCTF{}`

---

## Challenge Description

The challenge provides four files recovered from an abandoned project. The objective is to reconstruct an unknown scoring algorithm that combines phone step counts and Wi-Fi signal strength.

The scoring system is designed to identify the access point associated with the location the device owner frequently returns to.

### Provided Files

| File | Description |
|---|---|
| `notes.txt` | Explains the scoring concept but omits the movement cutoff |
| `aps.conf` | Contains trust weights for four access points |
| `scan_export.csv` | Contains 33 Wi-Fi scan records |
| `run.log` | Contains execution logs and scoring subtotals |

The main challenge was to recover the missing movement threshold, understand the scoring formula, and identify the highest-scoring access point.

## 1. Understanding the Scoring Rules

I started by examining `notes.txt`.

The notes described four important conditions:

1. Only readings recorded while the device owner was moving should be counted.
2. Only access points listed in `aps.conf` are eligible.
3. Each access point has a trust weight.
4. The final score is calculated using the signal strength multiplied by the corresponding trust weight.

However, the exact movement threshold was intentionally omitted.

The notes also mentioned two potential traps:

- Selecting the access point with the strongest signal.
- Selecting the access point appearing most frequently.

This indicated that simply counting records or comparing signal strengths would not produce the correct result.

## 2. Recovering the Movement Threshold

Next, I examined `run.log` to understand how the original scoring program classified movement.

The log contained the following decisions for `CafeDelMundo`:

| Signal | Step Delta | Decision |
|---:|---:|---|
| 62 | 40 | Dropped |
| 58 | 41 | Kept |

These two records established the boundary.

A reading with `step_delta = 40` was rejected, while a reading with `step_delta = 41` was accepted.

Therefore, the correct movement condition was:

```python
step_delta >= 41
```

Other entries in the log supported this conclusion:

- 22, 11 and 5 were dropped.
- 44, 58 and 77 were kept.

This recovered the missing movement threshold.

## 3. Reconstructing the Scoring Formula

The next step was to verify how the trust weights and signal strengths were combined.

The execution log provided useful subtotals.

### CafeDelMundo

Trust weight:

```text
w = 2
```

Accepted signal readings:

```text
58, 60, 64
```

Calculation:

```text
58 + 60 + 64 = 182

182 × 2 = 364
```

<badge color="success">Verified score: 364</badge>

### HomePlex_5G

Trust weight:

```text
w = 2
```

Accepted signal reading:

```text
87
```

Calculation:

```text
87 × 2 = 174
```

<badge color="success">Verified score: 174</badge>

Both calculations matched the subtotals recorded in `run.log`.

Therefore, the scoring formula was:

\[
\text{Score}(AP)=W(AP)\times
\sum_{\text{step\_delta}\geq41}\text{signal}
\]

Where:

- \(W(AP)\) is the trust weight of the access point.
- `signal` represents the Wi-Fi signal strength.
- Only moving readings are included.

## 4. Implementing the Scoring Algorithm

With the threshold and scoring formula recovered, I wrote a Python script to process the complete CSV export.

```python
import csv

W = {
    "FoxgloveCoffee_Guest": 3,
    "HomePlex_5G": 2,
    "CafeDelMundo": 2,
    "NETGEAR-8841": 1
}

score = {}

with open("scan_export.csv", newline="") as f:
    for r in csv.DictReader(f):
        ap = r["ap_name"]

        if ap in W and int(r["step_delta"]) >= 41:
            score[ap] = (
                score.get(ap, 0)
                + int(r["signal"]) * W[ap]
            )

print(sorted(score.items(), key=lambda x: -x[1]))
```

The script performs three operations:

1. Filters out access points that are not present in the trust-weight configuration.
2. Rejects readings where `step_delta` is below 41.
3. Adds the weighted signal strength to the corresponding access point's total score.

## 5. Analyzing the Results

After processing all 33 records, the final scores were:

| Access Point | Accepted Signals | Weight | Score |
|---|---|---:|---:|
| **FoxgloveCoffee_Guest** | 68, 71, 66, 73, 70 | 3 | **1044** |
| CafeDelMundo | 58, 60, 64 | 2 | 364 |
| NETGEAR-8841 | 80, 78, 82 | 1 | 240 |
| HomePlex_5G | 87 | 2 | 174 |

The highest-scoring access point was:

```text
FoxgloveCoffee_Guest
```

Its combination of strong signal readings and the highest trust weight produced a score of 1044.

## 6. Understanding the Traps

The challenge included several misleading indicators.

### Trap 1: The Loudest Access Point

The strongest individual signal belonged to:

```text
xfinitywifi
```

Its signal strength was 96.

However, it was not listed in `aps.conf`, meaning it was not eligible for scoring.

### Trap 2: The Most Frequent Access Point

`HomePlex_5G` appeared in 10 records, making it the most frequently observed access point.

However, nine of those records were collected while the device was stationary.

Only one reading qualified for scoring, resulting in a total of 174.

### Trap 3: Incorrect Threshold Boundary

Using:

```python
step_delta > 41
```

instead of:

```python
step_delta >= 41
```

would incorrectly exclude the reading with a step delta of 41.

That would break the CafeDelMundo subtotal of 364, which was independently confirmed by the execution log.

This demonstrates why carefully reconstructing comparison operators matters during forensic analysis.

## 7. Recovering the Flag

The scoring results identified `FoxgloveCoffee_Guest` as the highest-ranked access point.

The corresponding flag was:

```text
hiCTF{FoxgloveCoffee_Guest}
```

## Flag

```text
hiCTF{FoxgloveCoffee_Guest}
```

## Key Takeaways

- **Reconstruct missing logic:** Execution logs can reveal parameters that are intentionally omitted from documentation.
- **Verify assumptions mathematically:** Intermediate subtotals are useful for validating a reconstructed algorithm.
- **Understand filtering conditions:** A high signal strength does not matter if the access point is not trusted.
- **Watch for frequency traps:** The most frequently observed access point is not necessarily the most relevant one.
- **Pay attention to boundary conditions:** Small differences between `>` and `>=` can change the final result.

---

**Conclusion:** The Foxglove Trace was an offline forensic analysis challenge focused on reconstructing an unknown scoring algorithm. By examining the project notes, recovering the movement threshold from execution logs, and implementing the weighted scoring formula in Python, I identified `FoxgloveCoffee_Guest` as the winning access point.

**Event:** hiCTF 2026  
**Category:** Miscellaneous
