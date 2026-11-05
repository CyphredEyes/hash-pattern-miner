# hash-pattern-miner
A brute-force hash pattern searcher for finding user-defined sequences in hash outputs (e.g., hexadecimal patterns like "F0F0F0" or Base64 substrings like "GitHub" and "Python").

> Note: This is not cryptocurrency mining — it's a brute-force string-pattern search over hash outputs.

### Searching Methods:

### Base64 (This was the first method I explored, and the one I had the most fun with, because it can form human readable strings)

  Search is performed on Base64-encoded hash outputs. This method is interesting because the alphabet allows partial word formation, making patterns like "GitHub" or "Python" possible.

### Hexadecimal

  Search is performed directly on the raw hexadecimal representation of the hash output, it lacks readable structure compared to Base64.

---

### Quick start

```txt
Usage: hash-pattern-miner [OPTIONS] <PATTERN>
  
  Arguments:
    <PATTERN>  Pattern to search for (substring of the encoded hash)
  
  Options:
    -m, --mode <MODE>        Encoding mode [default: base64] [possible values: base64, hex]
    -t, --threads <THREADS>  Number of threads (0 = auto = logical cores) [default: 0]
        --start <START>      Start value of n [default: 0]
    -c, --count <COUNT>      Stop after N matches (0 = infinite, Ctrl+C to stop) [default: 0]
        --csv <FILE>         Append matches to this CSV file (input,raw_hash,encoded_hash)
        --no-color           Disable ANSI colors in match output
    -h, --help               Print help
```

```bash



# Build (requires Rust 1.75+)
cargo build --release

# Find "GitHub" in base64-encoded SHA-256 outputs
./target/release/hash-pattern-miner GitHub --count 1

# Find "5feceb" in hex-encoded SHA-256 outputs
./target/release/hash-pattern-miner 5feceb --mode hex --count 1

# Run forever, Ctrl+C to stop, append matches to CSV
./target/release/hash-pattern-miner Python --csv matches.csv
```

Matches print to stdout with the pattern highlighted.

Multithreaded with [rayon](https://github.com/rayon-rs/rayon), SHA-NI accelerated via [sha2](https://github.com/RustCrypto/hashes), zero allocations in the hot loop. Typical throughput: **200–600 MH/s** on a modern CPU.

---




### A simple demonstration of the idea, searching for "GitHub" inside the base64-encoded sha256 hash:

```rust
use sha2::{Digest, Sha256};
use base64::{Engine, engine::general_purpose::STANDARD};

let pattern = "GitHub";
let mut n: u64 = 0;
loop {
    let digest = Sha256::digest(n.to_string().as_bytes());
    let b64 = STANDARD.encode(&digest);
    if b64.contains(pattern) {
        println!("found {pattern} in {b64} (n={n})");
        break;
    }
    n += 1;
}
```
### Example output

For `n = 385121501`, we compute `SHA-256(str(n))` — which produces a raw 32-byte digest — and then Base64-encode **those raw bytes** directly:

```
n           = 385121501
raw digest  = 26278bd5cc38fba4179e6c557fc6461a2b47babbbf508ad1dfddc1cbba536a7e
Base64(raw) = JieL1cw4+6QXnmxVf8ZGGitHubsPUIrR393By7pTan4=
```

The `raw digest` line is just a human-readable hex display of the bytes for convenience — the Base64 encoder receives the **32 actual bytes**, not this hex string. Encoding the hex string would produce a different (and longer) result.

Approximately 385 million hashes to get the first "GitHub"

**Note that the more characters your target string has, the harder it will be to find it**

---

### The math behind

We're using SHA-256, which produces a fixed 256-bit output. Encoded in Base64, it becomes a 44-character string using a 64-symbol alphabet (A–Z, a–z, 0–9, + and /).

Note: the final Base64 character before the `=` padding has only 16 possible values, not 64, because only 2 bits of entropy feed into it. This is a minor correction that doesn't meaningfully affect the approximations used below.

A Base64-encoded SHA-256 hash is 44 characters long. For a pattern of length X, the positions available are **44 - X + 1**.
So, for a 6 character pattern like "GitHub": 44 - 6 + 1 = 39 possible positions, each hash gives us 39 possible positions where the pattern could appear.

The probability per hash is **P = 1 - (1 - 1/64^X)^(44 - X + 1)**.
For larger patterns (X ≥ 4), overlap probability is negligible, so the approximation **P ≈ (44 - X + 1) / 64^X** is very close.
For a 6 character pattern like "GitHub": P ≈ 39 / 64⁶ ≈ 5.67 × 10⁻¹⁰, so each hash has a **~0.0000000567%** chance of containing the pattern and the average number of hashes to find a match is 64⁶ / 39 ≈ 1.76 billion.

$$
P = 1 - \left(1 - \frac{1}{64^X}\right)^{44-X+1}
$$

---

#### Practical results

Here is some research i have done with "GitHub"

| n | Hash |
|---|------|
| 385,121,501 | `JieL1cw4+6QXnmxVf8ZGGitHubsPUIrR393By7pTan4=` |
| 1,863,243,251 | `50GitHubvKkJlF58gM317iBczllWOJCOPIUf53YU/VA=` |
| 4,459,354,449 | `qzx9SYnlPqwAbdj1QGitHubxqqq2MKaLc/7Il2g6ZO0=` |
| 5,071,751,051 | `cOLJDn3Ox1XY4hvLOZEy8EOn+lLOg7+GGitHub5EvS4=` |
| 9,722,760,142 | `DvCGitHubvrtGs7q+sS2iYOjyQQmM/2xRBqYWbbODm4=` |
| 10,482,513,114 | `mNUY5GitHubRBvtkdHRPoq/Mromc7YGiHdoR+yaLnTg=` |
| 10,517,355,940 | `+wxKW80OJhvOenQZOQAl7xYGitHubYtozCKigmQvN1I=` |
| 12,413,501,635 | `VTP9jGitHub0OtVdPvrki1rv7I1mmewb2fHlKl0y9j0=` |
| 12,560,796,672 | `T+rKxN/uP8utF+CAg1c6SDODggMnHYCIwvnGitHubUk=` |

The gaps between consecutive matches varied from ~35M to ~4.65B, but the practical average of the 8 inter-match gaps — 1.52 billion — is close to the theoretical 1.76 billion...

---

### Difficulty scaling by pattern length

| Pattern Length | Positions | Total Combinations | Per-Hash Probability (approx.) | Expected Hashes |
|----------------|-----------|-------------------|-------------------------------|-----------------|
| 1 char | 44 | 64 | 1 - (63/64)^44 ≈ 0.500 | ~2 |
| 2 chars | 43 | 4,096 | 1 - (1 - 1/4096)^43 ≈ 0.0105 | ~95 |
| 3 chars | 42 | 262,144 | 42/262,144 | ~6,242 |
| 4 chars | 41 | 16,777,216 | 41/16,777,216 | ~409,200 |
| 5 chars | 40 | 1,073,741,824 | 40/1,073,741,824 | ~26.8 million |
| 6 chars | 39 | 68,719,476,736 | 39/68,719,476,736 | ~1.76 billion |
| 7 chars | 38 | 4,398,046,511,104 | 38/4.4T | ~116 billion |
| 8 chars | 37 | 281,474,976,710,656 | 37/281T | ~7.6 trillion |

Each extra character multiplies difficulty by ~64, while positions decrease by 1.

---

### More patterns tested

#### Python (6 chars — expected: ~1.76 billion)

| n | Hash |
|---|------|
| 1,209,165,140 | `vrQPython58GtZzt4nSVlLc7C11Ui3rOHBkCoOJqFj0=` |
| 2,040,080,585 | `LODq/rEJovLQi1mHBelXQskTGPythonjYjOYGBH5mxo=` |
| 3,461,072,520 | `kQPJIkUATUKkqn2Vq88y5/vYS6tYPythonfDTUVVyXQ=` |
| 4,613,362,899 | `y7syeeo5WPython/E3SZVdculD+rSBMuZOmI68IjY5w=` |
| 4,753,600,887 | `760j3/+FBzkvj6QmuKPython9zro0bFsY781mhLN9sQ=` |
| 6,431,978,464 | `eoHUnyZQ5wPythonUo8YZCMl7ZV5vLpEfzRMLf6mK34=` |

average gap: 1.04B

---

#### XXXXXX (6 chars — expected: ~1.76 billion)

| n | Hash |
|---|------|
| 1,795,204,091 | `piRhZw9xJ0lw0wnDHAQKCNPZXaAyDO5516XXXXXXOAI=` |
| 4,817,114,945 | `O+MOyNGWrLUlhFpXXXXXX5xnrZodwiwHnSt+A2lZM7E=` |
| 6,944,174,827 | `Oo30pjkqCDXXXXXXDPHx0G2crKhgKfxjqlgd/7Ybddg=` |
| 7,644,672,584 | `0YpdmdQwmAQWefSqUNrdXXXXXXHi0pkpMxhdh5B09Vk=` |
| 10,865,295,694 | `Jvzzi1fydC8vE9DVCSCwPfde2fdQlkJXXXXXXZJTbE8=` |

average gap: 2.27B

---

#### XXXXX (5 chars — expected: ~26.8 million)

| n | Hash |
|---|------|
| 11,128,525 | `xbzJEmJLGXXXXXsBKgGL8vbfLTHEKw+NNA9pswtb92g=` |
| 44,192,272 | `Jg+1rHocod7bQ0y61khncgpFlkqgfDVFl+XXXXXSS1E=` |
| 64,377,159 | `LA737SPmF4JsejHsvj4QlXXXXXOY76G0Xa9dGd7lKGo=` |
| 96,720,303 | `l1CfyLXXXXXB7O/nosicKvI0ELkSnUEmlHdoCRsmK6c=` |
| 125,267,232 | `wRXXXXXWG1OsHCAHC5UhVbp09seCpEfJcMekGqvPJ+g=` |
| 134,280,852 | `S2zfycXjDJ0A0IpfTCipBX8vrvnzwNGIMXXXXX+vhvU=` |
| 211,447,873 | `5GFpsVkzMvpF7fm0dw4bJ8iG/BAVV6D+gXXXXXMz5ro=` |
| 223,630,127 | `bBODnMKTIJMwSHFUAF77EAyqK45tlXXXXXwYxmSJq/c=` |
| 279,342,016 | `P2s3ZeSrtYUXXXXXYoR2auAFMWA5UDeVwzuzMNSJxBM=` |
| 290,984,470 | `ViYo8DvSZvGsvoYpNXoXXXXXQT9RnLrnYFW4jbMn1CQ=` |
| 363,703,930 | `IDEiNlm8PaiF+m/mDtyXXXXXU7ycZwDcgZFf7rc43/o=` |
| 369,540,462 | `b7ruAiA7PNfsj1rUXXXXXtxtB9CuFagrI3zbUrZ6BeI=` |

average gap: 32.6M

---

I'm still working on this project.

I think all experimental results align with the mathematical model, but since I'm not so good at math, I still have some questions about it. Feel free to help me out!
If you think some math is wrong and you wanna explain it better, or if you have your own version of the hash miner, you're welcome!

Also, if you find any nice patterns on the hashes you mine, please share it!

That's it for now, happy mining out there!
