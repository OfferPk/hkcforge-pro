# CCSDS 124.0-B-1: Robust Compression of Fixed-Length Housekeeping Data Specification

**Document Standard**: CCSDS 124.0-B-1 (Blue Book, Issue 1, February 2023\)  
**Space Agency Working Group**: Space Link Services Area / Data Compression Working Group (SLS-DC)  
**Algorithm Origin**: European Space Agency (ESA) POCKET+ Algorithm  
**Classification**: Recommended Standard / Public Domain

---

## 1\. Executive Summary & Purpose

The **Consultative Committee for Space Data Systems (CCSDS) 124.0-B-1** Recommended Standard establishes a low-complexity, packet-loss-resilient lossless data compression algorithm applied to fixed-length binary vectors (\$I\_t\$).

In space missions, spacecraft housekeeping telemetry (e.g., voltages, temperatures, relay switches, reaction wheel speeds, payload sensor statuses) represents a mission-critical stream that competes for severely constrained RF downlink bandwidth (\$\<9.6\\text{ kbps}\$ on UHF/S-band, or brief ground visibility passes). Traditional lossless compression algorithms (e.g., gzip, Deflate, LZ4, zstd) are unsuitable for spaceborne flight computers because:

1. They require significant RAM dictionary buffers and high CPU cycle budgets that conflict with real-time flight software.  
2. They introduce multi-packet latency buffers incompatible with live real-time "fire-and-forget" downlink streams.  
3. A single dropped packet causes catastrophic decompressor failure across the entire subsequent stream.

CCSDS 124.0-B-1 (POCKET+) resolves these limitations through a bitwise, zero-latency, packet-loss-resilient architecture that operates on fundamental processor instructions (XOR, OR, AND, bit shifts) and provides mathematical guarantees for automatic synchronization recovery following burst packet losses.

---

## 2\. Technical Architecture & Dataflow

\+-----------------------------------------------------------------------------+

\|               CCSDS 124.0-B-1 (POCKET+) Dataflow Architecture               \|

\+-----------------------------------------------------------------------------+

       Input Vector I\_t (Fixed Length F bits, e.g., 256 bits)

                           │

                           ▼

          Difference Vector E\_t \= I\_t XOR I\_{t-1}

                           │

            ┌──────────────┴──────────────┐

            ▼                             ▼

   Mask Vector M\_t               Build Vector B\_t

   (Tracks Current Predictability) (Accumulates Candidate Masks)

   M\_t \= M\_{t-1} OR E\_t           B\_t \= B\_{t-1} OR E\_t

   \[If p\_t=1: M\_t \= B\_{t-1} OR E\_t\] \[If p\_t=1: B\_t \= 0\]

            │                             │

            ▼                             ▼

   Change Vector D\_t \= M\_t XOR M\_{t-1}   (Tracks State Evolution)

            │

            ▼

   Cumulative Change Vector over Robustness Window V\_t:

   C\_cum \= D\_t OR D\_{t-1} OR ... OR D\_{t-V\_t}

            │

            ├─────────────────────────────┬───────────────────────────┐

            ▼                             ▼                           ▼

   Stream 1: h\_t                 Stream 2: q\_t               Stream 3: u\_t

   (Mask Changes)                (Full Mask)                 (Payload)

   • 4-bit Robustness V\_t        • Sent if s\_t=1             • Extracted

   • Flags (p\_t, s\_t, u\_t, c\_t)  • Preprocessed                Unpredictable Bits

   • Run-to-Count RLE(C\_cum)       RLE(M\_t)                    (M\_t \== 1\)

   • Mask Values at '1' bits                                 • Or raw if u\_t=1

            │                             │                           │

            └─────────────────────────────┼───────────────────────────┘

                                          ▼

                Output Vector o\_t \= h\_t \|\| q\_t \|\| u\_t (Variable Length)

---

## 3\. Mathematical Formulation

### 3.1 Mask Update Formulation (§4.2)

Let \$I\_t\$ be the input vector of length \$F\$ at time index \$t \\ge 0\$.

- For \$t \= 0\$: \$B\_0 \= 0\$, \$M\_0\$ is user-specified initial mask (default all 1s).  
- For \$t \\ge 1\$:  
  - Difference: \$E\_t \= I\_t \\oplus I\_{t-1}\$  
  - If \$p\_t \= 1\$ (new mask requested): \$\$M\_t \= B\_{t-1} \\vee E\_t\$\$ \$\$B\_t \= 0\$\$  
  - If \$p\_t \= 0\$ (normal cycle): \$\$M\_t \= M\_{t-1} \\vee E\_t\$\$ \$\$B\_t \= B\_{t-1} \\vee E\_t\$\$  
  - Change Vector: \$\$D\_t \= M\_t \\oplus M\_{t-1}\$\$

### 3.2 Run-to-Count Encoding (§5.2.3)

Given binary vector \$a\$, \$\\text{RLE}(a)\$ is defined as: \$\$\\text{RLE}(a) \= \\text{COUNT}(C\_0) \\parallel \\text{COUNT}(C\_1) \\parallel \\dots \\parallel \\text{COUNT}(C\_{H(a)-1}) \\parallel \\text{'10'}\$\$ where \$C\_i\$ is **one more than the number of consecutive '0' bits preceding the \$i\$-th '1' bit in \$a\$**, starting at MSB.

### 3.3 Counter Encoding Table (Table 5-1)

For integer \$A \\ge 1\$:

- If \$A \= 1\$: `'0'` (1 bit)  
- If \$2 \\le A \\le 33\$: `'110'` \$\\parallel \\text{BIT}\_5(A \- 2)\$ (8 bits total)  
- If \$A \\ge 34\$: `'111'` \$\\parallel \\text{BITE}(A \- 2)\$ where \$E \= 2\\lfloor \\log\_2(A \- 2\) \+ 1 \\rfloor \- 6\$ (comprising \$(k-6)\$ zeros followed by \$k\$-bit binary representation of \$A-2\$).

---

## 4\. Packet-Loss Resynchronization Mechanism (§1.2 & §2.2)

By incorporating the cumulative mask change vector over past \$V\_t \\le R\$ cycles: \$\$\\text{Effective Robustness Guarantee: } \\Delta t\_{\\text{lost}} \\le V\_t\$\$ If up to \$R\$ consecutive packets are lost over a noisy space RF channel, the decompressor re-synchronizes its mask immediately upon receiving the next packet using the cumulative change vector in \$h\_t\$ and resumes 100% lossless decompression without requiring a retransmission handshake.