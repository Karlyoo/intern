- [ch4 Frame structure and physical resources ](https://github.com/Karlyoo/LDPCinOAI/blob/main/3GPP%20TR%2038.211.md#4-frame-structure-and-physical-resources)
- [ch5 Generic functions](https://github.com/Karlyoo/LDPCinOAI/blob/main/3GPP%20TR%2038.211.md#5-generic-functions)
- [ch6 uplink](https://github.com/Karlyoo/LDPCinOAI/blob/main/3GPP%20TR%2038.211.md#6-uplink)
- [ch7 downlink](https://github.com/Karlyoo/LDPCinOAI/blob/main/3GPP%20TR%2038.211.md#7-downlink)
- [ch8 sidelink](https://github.com/Karlyoo/LDPCinOAI/blob/main/3GPP%20TR%2038.211.md#8-sidelink)

# 4 Frame structure and physical resources 
## 4.1 General
- Usually,Time units are defined as:
  - Tc = 1 / (Δf_max × Nf),
  - where Δf_max = 480,000 Hz and Nf = 4096.
- Another commonly used time unit is:
  - Ts = Tc × (Nf_ref / Δf_ref),
  - where Δf_ref = 15,000 Hz and Nf_ref = 2048.
## 4.2 Numerologies
- The subcarrier spacing is defined as:
  - Δf = 2^μ × 15 kHz,  where μ is the numerology index.
  - Supported configurations:
    
    | μ | Subcarrier spacing (kHz) | Cyclic Prefix      |
    | - | ------------------------ | ------------------ |
    | 0 | 15                       | Normal             |
    | 1 | 30                       | Normal             |
    | 2 | 60                       | Normal or Extended |
    | 3 | 120                      | Normal             |
    | 4 | 240                      | Normal             |

## 4.3 Frame Structure
### 4.3.1 Frames and Subframes
- One frame is 10 milliseconds long and consists of 10 subframes (each 1 ms).
- Each frame is split into two half-frames:
  - Half-frame 0: subframes 0 to 4
  - Half-frame 1: subframes 5 to 9
- Downlink (DL) and uplink (UL) have separate frame structures on a carrier.
- The UL frame number i starts earlier than the corresponding DL frame by:
  - TA = (N_TA + N_TA_offset) × Tc
### 4.3.2 Slots
- For each numerology μ:
  - Slots are numbered within subframes and frames.
  - Each slot contains a fixed number of OFDM symbols depending on the cyclic prefix.
 
| μ | Symbols per slot | Slots per frame | Slots per subframe |
| - | ---------------- | --------------- | ------------------ |
| 0 | 14               | 10              | 1                  |
| 1 | 14               | 20              | 2                  |
| 2 | 14               | 40              | 4                  |
| 3 | 14               | 80              | 8                  |
| 4 | 14               | 160             | 16                 |

- OFDM symbols in a slot are categorized as:
  - Downlink
  - Uplink
  - Flexible
- In a downlink slot, the UE should expect transmissions only in downlink or flexible symbols.
- In an uplink slot, the UE may only transmit in uplink or flexible symbols.
#### Timing Requirements for Half-Duplex UEs
- A UE that does not support full-duplex or simultaneous transmission and reception must observe certain timing gaps.

| Transition | FR1 (Tc units) | FR2 (Tc units) |
| ---------- | -------------- | -------------- |
| Tx to Rx   | 25600          | 13792          |
| Rx to Tx   | 25600          | Not specified  |

- Such a UE must:
  - Wait Rx-to-Tx time before transmitting after receiving.
  - Wait Tx-to-Rx time before receiving after transmitting.
 
# 5 Generic functions
## 5.1 Modulation mapper
- The modulation mapper takes binary digits, 0 or 1, as input and produces complex-valued modulation symbols as
output.
<img width="938" height="165" alt="image" src="https://github.com/user-attachments/assets/d100f01d-4867-48f8-af10-da2d8392f7e0" />
<img width="537" height="574" alt="image" src="https://github.com/user-attachments/assets/d40729a3-46c5-4636-b43b-cb33ba702f65" />

### 5.3.1 Baseband Signal for All Channels Except PRACH and RIM-RS
#### Time-domain Baseband Signal Definition
- The time-domain signal on antenna port p, symbol index l, and subcarrier spacing configuration mu is:
   - s_l^(mu)(t, p) = s̄_l^(mu)(t, p), if t_start,l^(mu) ≤ t < t_start,l^(mu) + T_symb,l^(mu)
   - s_l^(mu)(t, p) = 0, otherwise
- OFDM Signal Formula
   - s̄_l^(mu)(t, p) = sum from k = k_min to k_max of [a_k,l^(mu)(p) * exp(j * 2 * pi * k * Delta_f * (t - T_CP,l^(mu)))]
- Symbol Start Time
  - t_start,l^(mu) =0, if l = 0
  - t_start,l-1^(mu) + T_symb,l-1^(mu), if l > 0
- Cyclic Prefix Extension (First Symbol)
  - For PUSCH, SRS, or PUCCH with extended CP:
     - s_ext,0^(mu)(t, p) = s̄_0^(mu)(t, p), for t in [t_start,0^(mu) - T_ext, t_start,0^(mu))

### 5.3.2 Baseband Signal Generation for PRACH
- Time-domain PRACH Signal
   - s_RA^(mu)(t, p) = sum from m = 0 to L_RA - 1 of [x_RA(m) * exp(j * 2 * pi * m * Delta_f_RA * t)]
   - x_RA(m): Preamble sample
   - Delta_f_RA: PRACH subcarrier spacing (e.g., 1.25, 5 kHz)
   - L_RA: PRACH sequence length
-  Time and Frequency Mapping for PRACH
   - t_start,RA = 0, if l = 0
   - t_start,l-1^(mu) + T_symb,l-1^(mu), if l > 0
Notes:
   - mu = 0 assumed for 1.25 or 5 kHz PRACH
   - Higher mu used for 15/30/60/120 kHz PRACH
   - Parameters:
     - start_NBWP,i: Lowest RB index of the uplink BWP
     - f_RA_start: PRACH frequency offset
     - n_RA: Frequency-domain occasion index
     - N_RB_RA: Number of RBs allocated for PRACH
     - L_RA: Preamble length
     - T_CP_RA: CP duration for PRACH

| Feature            | All Channels (excl. PRACH/RIM-RS) | PRACH          |
| ------------------ | --------------------------------- | -------------- |
| IFFT-based         | Yes                               | No             |
| Data Type          | Modulated data symbols            | Preamble       |
| CP Handling        | Standard or Extended              | Special rule   |
| Subcarrier spacing | 15 \~ 120 kHz                     | 1.25 \~ 60 kHz |
| Application        | Data/Control channels             | Random Access  |

# 6 Uplink
## 6.3 Physical channels
### 6.3.1 Physical uplink shared channel
#### 6.3.1.1 Scrambling – Physical Uplink Shared Channel (PUSCH)
- Before modulation, the bit sequence in the PUSCH codeword (q = 0) is scrambled to improve the robustness of transmission against interference and to separate UEs.
- Scrambling Logic (Pseudocode)
- Let:
  - b_q(i): i-th input bit of codeword q
  - b̃_q(i): i-th scrambled output bit
  - c_q(i): i-th scrambling sequence bit
```
    i = 0
while i < N_q^bit:
    if x(i) == q:                // UCI placeholder bits
        b̃_q(i) = b_q(i)
    else if y(i) == q:           // UCI placeholder bits
        b̃_q(i) = 1 - b_q(i)
    else:
        b̃_q(i) = (b_q(i) + c_q(i)) mod 2
    i = i + 1
```
- N_q^bit is the number of bits in codeword q
**Scrambling Sequence Initialization**
 ``` 
  c_init = {
   RNTI × 2^15 + RAPID × 2^14 + ID         // if it's msgA on PUSCH
   RNTI × 2^16 + ID                        // otherwise
}
```
| Parameter | Meaning                                                                                    |
| --------- | ------------------------------------------------------------------------------------------ |
| **RNTI**  | Radio Network Temporary Identifier associated with the PUSCH (e.g., C-RNTI, RA-RNTI, etc.) |
| **RAPID** | Random Access Preamble Index (used for msgA triggered by random access)                    |
| **ID**    | Depends on context:                                                                        |
|           | - `msgA-dataScramblingIdentity` (if configured and msgA is used)                           |
|           | - `dataScramblingIdentityPUSCH` (if configured and RNTI is C-RNTI / SP-CSI-RNTI / etc.)    |
|           | - Else: `ID = N_ID_cell`, the physical cell ID                                             |

### 6.3.1 Physical Uplink Shared Channel (PUSCH)
#### 6.3.1.2 Modulation
- Input: Scrambled bit sequence for codeword q = 0
- Output: Modulated complex-valued symbols d_q(0), ..., d_q(N_symb(q) - 1)
  
| Transform Precoding | Modulation Scheme | Modulation Order (Q<sub>m</sub>) |
| ------------------- | ----------------- | -------------------------------- |
| **Disabled**        | π/2-BPSK          | 1                                |
|                     | QPSK              | 2                                |
|                     | 16QAM             | 4                                |
|                     | 64QAM             | 6                                |
|                     | 256QAM            | 8                                |
| **Enabled**         | QPSK              | 2                                |
|                     | 16QAM             | 4                                |
|                     | 64QAM             | 6                                |
|                     | 256QAM            | 8                                |

#### 6.3.1.3 Layer Mapping
- Input: Complex-valued modulation symbols d_q(0)...d_q(N_symb(q)-1)
- Output: Symbols mapped onto up to 4 transmission layers
- Each symbol is assigned to layer λ, where:
  - λ ∈ {0, 1, ..., ν−1} with ν being the number of layers
- Symbols are mapped to:
  - x_λ = [x_λ(0), ..., x_λ(N_symb_layer - 1)]^T
- (Mapping follows Table 7.3.1.3-1 in TS 38.211.)
#### 6.3.1.4 Transform Precoding (DFT-spread OFDM)
- Condition for Use:
  - Enabled or Disabled according to TS 38.214 Section 6.1.3
  - If enabled, only 1 layer (ν = 1) is allowed.
**Transform Precoding Process:**
- If transform precoding is disabled:
   - y_λ(i) = x_λ(i) for all λ
- If enabled and PT-RS is NOT used:
   - The input x_0(i) is divided per OFDM symbol (based on number of subcarriers).
- Transform precoding is applied directly.
- If PT-RS IS used:
   - Symbols are divided into sets by OFDM symbol index l
   - Set l contains:
      - Nsc_PUSCH - ε_l * Msamp_group
      - ε_l = 1 if the symbol contains PT-RS samples
      - Msamp_group: number of PT-RS samples per group
   - The remaining (non-PT-RS) symbols are input to the DFT.
**DFT-based Precoding Equation**
- The DFT (transform precoding) applied to each set:
  ```
  y_0(l ⋅ Nsc_PUSCH + k) = 
      ∑_{i=0}^{Nsc_PUSCH - 1} x_0(l ⋅ Nsc_PUSCH + i) ⋅ 
      exp(-j ⋅ 2π ⋅ i ⋅ k / Nsc_PUSCH)
  ```
- Result: ỹ_0(0), ..., ỹ_0(N_symb_layer - 1)
- Nsc_PUSCH = N_RB_PUSCH × 12 (Number of subcarriers = number of RBs × 12)
- N_RB_PUSCH: number of allocated resource blocks for PUSCH

#### 6.3.1.5  Precoding 
- Precoding is applied to the modulated symbols before transmission across multiple antenna ports.
- It transforms a block of modulation symbols for each layer into a new set of symbols for each antenna port using a precoding matrix W.
- Let:
  - M_symb_layer be the number of modulation symbols per layer
  - M_symb_ap be the number of modulation symbols per antenna port
  - rho_p be the number of antenna ports
  - W be the precoding matrix
```    
[z_0(i), ..., z_(rho_p - 1)(i)]^T = W * [y_0(i), ..., y_(number_of_layers - 1)(i)]^T
```
  -  y_l(i) is the symbol for layer l at symbol index i
  -  z_p(i) is the precoded output symbol for antenna port p
##### Precoding Matrix Tables
- Precoding matrices depend on:
  - Number of layers
  - Number of antenna ports
  - Transform precoding status (enabled/disabled)
  - TPMI index
- Single-layer transmission
  - 2 antenna ports: Table 6.3.1.5-1
  - 4 antenna ports:
    - Transform precoding enabled: Table 6.3.1.5-2
    - Transform precoding disabled: Table 6.3.1.5-3
- Two-layer transmission
  - 2 antenna ports: Table 6.3.1.5-4
  - 4 antenna ports (no transform precoding): Table 6.3.1.5-5
- Three-layer transmission
  - 4 antenna ports (no transform precoding): Table 6.3.1.5-6
- Four-layer transmission
  - 4 antenna ports (no transform precoding): Table 6.3.1.5-7

##### Precoding Matrix Selection
- Matrix W  is selected based on:
  - Number of antenna ports
  - Number of layers
  - TPMI index
  - Transform precoding enable/disable
  - Configurations in TS 38.214

### 6.3.2 Physical Uplink Control Channel (PUCCH)
#### 6.3.2.1 General Description
- The PUCCH supports multiple formats as listed in the table below.

| Format | Number of OFDM Symbols (Nsymb\_PUCCH) | Number of Bits |
| ------ | ------------------------------------- | -------------- |
| 0      | 1 to 2                                | ≤ 2            |
| 1      | 4 to 14                               | ≤ 2            |
| 2      | 1 to 2                                | > 2            |
| 3      | 4 to 14                               | > 2            |
| 4      | 4 to 14                               | > 2            |

#### 6.3.2.2 Sequence and Cyclic Shift Hopping
- Formats using sequence hopping:
  - Formats 0, 1, 3, and 4.
  - Sequences follow clause 5.2.2.
  - Group and sequence hopping are determined by higher-layer parameter ``` pucch-GroupHopping.```

##### 6.3.2.2.1 Group and Sequence Hopping
- Let n_ID be:
  - If hoppingId is configured: n_ID = hoppingId
  - Otherwise: n_ID = physicalCellId
- Let v be the sequence group number.
- Group Hopping Modes:
  - GroupHopping = 'neither'
    - v = 0
    - Group index = 0 mod 30
  - GroupHopping = 'enable'
    - v = (2^7 * c(0) + 2^6 * c(1) + ... + c(6)) mod 30
    - Sequence c(i) is pseudo-random, defined in clause 5.2.1
    - Initialized at the beginning of each radio frame using n_ID mod 30
  - GroupHopping = 'disable'
    - v = (2 * f + c(f + m0)) mod 30
    - Sequence c(i) is the same as above
    - Initialization: init = floor(n_ID / 30) + (n_ID mod 30)
- Frequency Hopping Index:
  - hop = 0 if intra-slot frequency hopping is disabled
  - hop = 0 for first hop, hop = 1 for second hop if enabled

##### 6.3.2.2.2 Cyclic Shift Hopping
The cyclic shift value n_cs is calculated per symbol and slot.

```
n_cs = (2 * m0 + cs + interlace_offset + c_cs(sym, slot)) mod N_sc_RB
```
- slot is the slot number
- sym is the OFDM symbol index (starting at 0)
- sym' is the index of the first OFDM symbol in the slot (defined in TS 38.213)
- m_cs = 0, except for Format 0 (depends on the content being transmitted)
- interlace_offset:
  - = 5 * IRB if interlaced mapping is enabled by higher-layer parameters
  - = 0 otherwise
- IRB is the interlace Resource Block index
- Function c_cs(sym, slot):
  
```
c_cs(sym, slot) = sum over k=0 to sym of [2^k * c(k + 8 * slot)]
```
#### 6.3.2.3 PUCCH Format 0
##### 6.3.2.3.1 Sequence Generation
- The sequence x(n) is generated as follows:
  - For single-symbol PUCCH: only one symbol is generated.
  - For double-symbol PUCCH: a two-symbol sequence is generated.
    
 ```
  x(n) = r_alpha(l, n_RBsc * n + delta_alpha)
 ```
  - r_alpha(...) is the sequence defined in clause 6.3.2.2
  - n = 0 to N_sc_RB - 1 (N_sc_RB = 12 subcarriers)
  - l is the OFDM symbol index
  - m_cs is determined by clause 9.2 in TS 38.213, depending on UCI content
##### 6.3.2.3.2 Mapping to Physical Resources
- The sequence x(n) is:
  - Scaled by a power factor β_PUCCH,0 (for transmit power normalization)
  - Mapped to REs (Resource Elements) starting from x(0)
- Mapping rules:
  - Order: increasing subcarrier index k → then OFDM symbol index l on antenna port p = 2000
  - For interlaced transmission, the mapping is done per resource block (RB) in the interlace and bandwidth part.

#### 6.3.2.4 PUCCH Format 1
##### 6.3.2.4.1 Sequence Modulation
- The UCI bits b(0)...b(Mbit-1) are modulated:
  - BPSK for 1 bit
  - QPSK for 2 bits
  - Result: one complex-valued symbol d(0)
```    
y(n) = d(0) * r_alpha(l, n)
```
  - r_alpha(...) is from clause 6.3.2.2
  - Length: N_sc_RB (12 subcarriers)
**Blockwise Spreading with Orthogonal Sequences**
- Spreading is applied depending on slot configuration and hopping.
- Resulting spread sequence z(n) is:
```
z(n + m′ * N_sc_RB) = y(n) * w_m(i)
```
  - w_m(i) is an orthogonal sequence (from Table 6.3.2.4.1-2)
  - i is determined from higher-layer parameters (TS 38.213 clause 9.2.1)
  - m′ is 0 or 1 depending on whether intra-slot frequency hopping is used
  - The total number of sequences (PUCCH_1_N_SF,m') depends on PUCCH symbol length

| PUCCH Length | No hopping (m′ = 0) | Intra-slot hopping (m′ = 0, 1) |
| ------------ | ------------------- | ------------------------------ |
| 4            | 2                   | 1, 1                           |
| 5            | 2                   | 1, 1                           |
| 6            | 3                   | 1, 2                           |
| 7            | 3                   | 1, 2                           |
| 8            | 4                   | 2, 2                           |
| 9            | 4                   | 2, 2                           |
| 10           | 5                   | 2, 3                           |
| 11           | 5                   | 2, 3                           |
| 12           | 6                   | 3, 3                           |
| 13           | 6                   | 3, 3                           |
| 14           | 7                   | 3, 4                           |

##### 6.3.2.4.2 Mapping to Physical Resources
- The final spread sequence z(n) is:
  - Scaled by β_PUCCH,1
  - Mapped to REs not used by associated DM-RS
- Mapping rule:
  - First by subcarrier index k in the assigned RB
  - Then by OFDM symbol index l on antenna port p = 2000
- For interlaced transmission, mapping is repeated per RB in interlace region.
- Sequence r_alpha(...) remains RB-dependent (as in clause 6.3.2.2)

#### 6.3.2.5 PUCCH format 2 
- Scrambling (§6.3.2.5.1):
  - Scrambling with scrambled_bit = bit XOR c(i).
  - Init = RNTI * 2^16 + nID (dataScramblingIdentityPUSCH or cell ID).
- Modulation (§6.3.2.5.2):
  - QPSK modulation.
- Spreading (§6.3.2.5.2A):
  - Optional spreading if OCC-Length-r16 is configured.
  - Spreading factor ∈ {2,4}, orthogonal sequence depends on OCC-Index-r16.
- Mapping (§6.3.2.5.3):
  - Multiply with β<sub>PUCCH,2</sub>.
  - Map to RBs assigned and not used by DMRS.

#### 6.3.2.6 PUCCH Formats 3 and 4
- Support more bits than format 2; difference: format 4 uses intra/inter-slot frequency hopping with transform precoding.
- Scrambling (§6.3.2.6.1):
  - Same method as format 2.
- Modulation (§6.3.2.6.2):
  - QPSK (bit/2 symbols) or π/2-BPSK (1 bit = 1 symbol).
- Block-wise Spreading (§6.3.2.6.3):
  - Required for interlaced format 3 and all format 4.
  - Spreading factors from {2,4}, using orthogonal sequences (Tables 6.3.2.6.3-1 and 6.3.2.6.3-2).
- Transform Precoding (§6.3.2.6.4):
  - Applied after spreading to create a DFT-precoded symbol block.
- Mapping (§6.3.2.6.5):
  - Multiply with β<sub>PUCCH,s</sub>.
  - Map to assigned RBs (exclude DMRS).
- Intra-slot frequency hopping: split symbols evenly between hops.

### 6.3.3 Physical random-access channel
#### 6.3.3.1 PRACH Sequence Generation
**Zadoff-Chu Sequence Generation**
- Time-domain base sequence:
```
x_u,v(n) = x_u((n + Cv + L_RA * i) mod L_RA)
```
  - x_u(n): Zadoff-Chu sequence of length L_RA with root index u
  - Cv: cyclic shift
  - i: repetition index (if needed)
- Frequency-domain representation:
  - Y_u,v(n) = sum over m = 0 to L_RA - 1 of [x_u,v(m) * exp(-j * 2 * pi * m * n / L_RA)]
 
| Format       | L\_RA | Number of Root Sequences | Max Preambles Per Root |
| ------------ | ----- | ------------------------ | ---------------------- |
| Format 0–3   | 839   | 1151                     | Depends on N\_CS       |
| Format B4–B5 | 139   | 571                      | Depends on N\_CS       |

**Cyclic Shift Calculation (Cv)**
- Cv depends on the restricted set type:
- Unrestricted Set
  - Simple and direct:
    - Cv = v * N_CS , where v = 0, 1, ..., floor(L_RA / N_CS) - 1
- Restricted Set Type A
  - Used for longer sequences (L_RA = 839)
  - If d_u < 2: No shift allowed
  - If 3 <= d_u <= L_RA - 1:
    - Compute various shift-related parameters:
  - N_CS_group, N_shift, N_start, etc.
  - Use table-defined formulas for Cv
- Depends on the range of d_u (split into subcases)
- Restricted Set Type B
  - Used for shorter sequences (L_RA = 139)
  - If d_u < 2: No shift allowed
  - Otherwise:
    - Multiple d_u ranges each with their own formulas
    - Includes special handling when d_u >= 72
**Key Parameters**
- L_RA: Length of ZC sequence (839 or 139 depending on preamble format)
- N_CS: Cyclic shift granularity (determines how many shifts can be made)
- u: Root sequence index (logical root)
- v: Preamble index (0 to 63)
- d_u: Derived from L_RA and root index u
- restrictedSetConfig: Tells whether unrestricted, type A, or type B is used
- Tables 6.3.3.1-1 to 6.3.3.1-7 define all needed constants (like N_CS values)

#### 6.3.3.2 Mapping to Physical Resources
- After generating the PRACH preamble sequence, it is mapped to physical resource elements for transmission.
- The sequence y_{u,v}(k) is scaled and mapped as:
```
a_p(k) = β_PRACH × y_{u,v}(k)
```
- β_PRACH: Amplitude scaling factor (for transmit power control as defined in TS 38.213).
- p = 4000: Fixed antenna port for PRACH.
- Mapping follows the baseband signal generation procedure in clause 5.3.
**Frequency-Domain Mapping**
- Index k used for mapping depends on the PRACH format.
- The mapping location in frequency depends on:
- The configured PRACH frequency domain occasion (FDO)
- The starting frequency index
- The msg1-FDM parameter (if used)
**Time-Domain Resource Mapping**
- PRACH preambles are only transmitted in specific time resources defined by:
  - Tables 6.3.3.2-2 to 6.3.3.2-4, depending on:
     - Frequency Range (FR1 or FR2)
     - Spectrum type (paired or unpaired)
     - Subcarrier spacing (SCS)
- Time-domain resources are determined by the PRACH configuration index, which is configured via one of:
  - prach-ConfigurationIndex
  - prach-ConfigurationIndexNew
  - msgA-PRACH-ConfigurationIndex-r16
**Subcarrier Spacing Assumptions for PRACH**
- FR1: 15 kHz SCS
- FR2: 60 kHz SCS
- Used for interpreting PRACH configuration tables (slot numbering).

**Timing Assumptions for Handover**
- For handover in paired/unpaired spectrum (with N_frame_max = 4):
   - UE can assume max time offset < 153600 × Ts (Ts: base time unit)
- For inter-frequency handover to a cell in unpaired spectrum (N_frame_max = 8):
   - UE may assume max time offset < 7680 × Ts

##  6.4 Physical signals
### 6.4.1 Reference signals 
#### 6.4.1.3 PUCCH Demodulation Reference Signal (DM-RS)
**PUCCH Format 1**
- Sequence Generation:
- The DM-RS sequence for PUCCH format 1 is defined as:

```
r̄ₘₙ = z_{m'N_SF,PUCCH1}(n) · wₘ^{(i)}  for n = 0,...,N_RB^PUCCH,1 · N_sc^RB - 1
```
  - z_{m'N_SF,PUCCH1}(n) is defined in Clause 5.2.2 (Zadoff-Chu-like sequence).
  - m' is the DM-RS symbol index defined by PUCCH length and hopping setting (Table 6.4.1.3.1.1-1).
  - wₘ^{(i)} is the orthogonal cover code sequence from Table 6.3.2.4.1-2.

- Resource Mapping:
  - The sequence is scaled by β_PUCCH,1.
  - Mapped to antenna port 2000 across assigned RBs.
**PUCCH Format 2**
- Sequence Generation:
- QPSK symbols based on pseudo-random sequence c(i):
```
r̄ₙ = 1/√2 · [1 - 2·c(2n)] + j·1/√2 · [1 - 2·c(2n+1)]
```
  - c(i) is defined in Clause 5.2.
  - Initialization of the PRBS depends on scramblingID0 or n_ID_cell.
- Resource Mapping:
  - Multiplied by β_PUCCH,2 and mapped to antenna port 2000.
  - Subcarrier index k is relative to CRB0 (common resource block 0).

**PUCCH Formats 3 and 4**
- Sequence Generation:
  - Based on r̄ₘ(n) = r_{l_uv}^{(α, δ)}(n), where the base sequence comes from:
    - Clause 5.2.3 if transform precoding is used and π/2-BPSK applies.
    -Clause 6.3.2.2 otherwise.
- Cyclic shift varies with:
  - Symbol index
  - Intra-slot frequency hopping (from Table 6.4.1.3.3.1-1)
- Resource Mapping:
  - The sequence is scaled by β_PUCCH,s (s ∈ {3,4}).
  - Mapped to port 2000, k is relative to lowest-numbered PUCCH RB.
- Symbol positions depend on hopping and additional DM-RS (Table 6.4.1.3.3.2-1).

#### 6.4.1.4 SRS: Sounding Reference Signal
- An SRS resource includes:
- 1, 2, or 4 antenna ports (p_i = 1000 + i for codebook-based; otherwise follows TS 38.214)
- 1, 2, 4, 8, or 12 OFDM symbols (given by nrofSymbols)
- Time-domain position l₀ = N_symb,slot - 1 - offset, where offset ∈ [0,13]
- Frequency-domain starting position k₀
- The sequence is defined as:
```
r_i(n, l') = r_{u,v}^{(α_i)}(n) for each symbol l' ∈ [0, N_symb,SRS - 1]
```
- Cyclic shift α_i is given by:
```
α_i = 2π · (n_cs_i mod N_cs_max) / N_cs_max
```
- where N_cs_max depends on transmissionComb (see Table 6.4.1.4.2-1).
- Sequence hopping controlled by groupOrSequenceHopping:
  - "neither": v = 0
  - "groupHopping": hopping group changes per slot based on PRBS
  - "sequenceHopping": hopping applied if certain conditions are met

**Resource Mapping**
- Sequence is scaled by β_SRS.
- Mapped to k and l positions for each antenna port.
- Mapping formula:
```
a_p(k, l') = β_SRS · r_i(n, l')
```
- Total subcarriers:
```
N_sc,SRS = m_SRS,b · N_sc^RB / comb
```
- comb = {2, 4, 8}
- m_SRS,b and N_b are taken from Table 6.4.1.4.3-1
- b_SRS is selected via higher-layer parameter freqHopping (or default = 0)

**SRS Slot Configuration**
- For periodic or semi-persistent SRS:
- Periodicity T_SRS and offset Toffset configured by periodicityAndOffset-p or -sp
- Slot selection condition:
```
(slot_index - Toffset) mod T_SRS == 0
```

# 7. Downlink
### 7.1.1 Physical Channels
**Downlink physical channels carry higher layer information and include:**
- Physical Downlink Shared Channel (PDSCH)
- Physical Broadcast Channel (PBCH)
- Physical Downlink Control Channel (PDCCH)
### 7.1.2 Physical Signals
- Downlink physical signals are used by the physical layer but do not carry higher-layer information:
  - Demodulation Reference Signal (DM-RS)
  - Phase Tracking Reference Signal (PT-RS)
  - Positioning Reference Signal (PRS)
  - Channel State Information Reference Signal (CSI-RS)
  - Primary Synchronization Signal (PSS)
  - Secondary Synchronization Signal (SSS)
## 7.2 Physical Resources
- Downlink antenna ports:
  - Ports starting with 1000: PDSCH
  - Ports starting with 2000: PDCCH
  - Ports starting with 3000: CSI-RS
  - Ports starting with 4000: SS/PBCH block
  - Ports starting with 5000: PRS
- UE must not assume two antenna ports are quasi co-located (QCL) unless explicitly specified.

## 7.3 Physical Channels
### 7.3.1 PDSCH (Physical Downlink Shared Channel)
#### 7.3.1.1 Scrambling
- Up to two codewords (q ∈ {0,1}) may be transmitted.
- For each codeword, bits are scrambled before modulation:
```
b_scrambled(q)(j) = b(q)(j) + c(q)(j) mod 2
```
- The scrambling sequence c(q)(j) is initialized using:
```
c_init = RNTI * 2^15 + q * 2^14 + nID
```
#### 7.3.1.2 Modulation
Scrambled bits are modulated using one of the following schemes:

| Modulation | Order (Qm) |
| ---------- | ---------- |
| QPSK       | 2          |
| 16QAM      | 4          |
| 64QAM      | 6          |
| 256QAM     | 8          |

#### 7.3.1.3 Layer Mapping
- Scrambled modulation symbols are mapped to transmission layers based on the number of codewords and layers.
- Mapping follows specific rules (see Table 7.3.1.3-1 in the spec) depending on:
  - Number of layers (1–8)
- Whether single or dual codewords are transmitted

#### 7.3.1.4 Antenna Port Mapping
- Each layer is mapped to a corresponding antenna port (e.g., ports 1000+ for PDSCH).
- The number of antenna ports used equals the number of layers.

#### 7.3.1.5 Mapping to Virtual Resource Blocks
- The mapping of modulation symbols to Resource Elements (REs) follows these rules:
  - Only mapped to REs in assigned virtual resource blocks (VRBs).
  - REs must not:
     - Overlap with associated DM-RS or other UEs' DM-RS
     - Be used by non-zero-power CSI-RS (with some exceptions)
     - Be used by PT-RS
     - Be declared unavailable for PDSCH (per TS 38.214 clause 5.1.4)
  - Symbols are mapped in increasing order by:
     - Subcarrier index k' within VRBs
     - Symbol index l

### 7.3.2 Physical Downlink Control Channel (PDCCH)
#### 7.3.2.1 Control-Channel Element (CCE)
- A PDCCH consists of one or more CCEs.
- Supported aggregation levels: 1, 2, 4, 8, 16 CCEs (see Table 7.3.2.1-1).
- Each CCE consists of 6 REGs (Resource Element Groups).
- One REG = 1 Resource Block (RB) over 1 OFDM symbol.

#### 7.3.2.2 Control-Resource Set (CORESET)
- A CORESET spans a defined number of RBs (frequency) and symbols (time), where symbol ∈ {1, 2, 3}.
- REGs within a CORESET are indexed in time-first order: increasing symbol index, then RB index.
- One CORESET contains a total of N_REG = RB_CORESET × symbol_CORESET REGs.
- CCE-to-REG mapping:
   - One mapping per CORESET.
   - Two mapping types:
      - Non-interleaved: REG bundle size L = 6.
      - Interleaved: L ∈ {2, 3, 6} depending on symbol_CORESET.
- REG bundle i contains REGs:
```
{Li, Li+1, ..., Li+L-1}
```

#### 7.3.2.3 Scrambling
- Bits b(0)...b(bit-1) are scrambled before modulation:
```
b_scrambled(i) = b(i) + c(i) mod 2
```
Scrambling sequence c(i) initialized with:
```
c_init = RNTI × 2^15 + n_ID × 2^0
```
- where:
  - If pdcch-DMRS-ScramblingID is configured: n_ID = that value
  - Otherwise: n_ID = cell ID
  - RNTI = C-RNTI for UE-specific search space; 0 otherwise


#### 7.3.2.4 Modulation
- Modulation is QPSK.
- Resulting in complex-valued symbols: d(0)...d(symb-1)

#### 7.3.2.5 Mapping to Physical Resources
- Symbols d(0)...d(symb-1) are scaled by β_PDCCH and mapped to REs not used by DM-RS.
- Mapped in increasing order of subcarrier (k), then OFDM symbol (l).
- Antenna port = 2000.

### 7.3.3 Physical Broadcast Channel (PBCH)
#### 7.3.3.1 Scrambling
- Bits b(0)...b(Mbit-1) are scrambled:
```
b_scrambled(i) = b(i) + c(i + Mbit × ν) mod 2
```
- Scrambling sequence initialized with:
```
c_init = cell ID
```
- Index ν depends on SS/PBCH candidate index:
  - If Ssb-PositionsInBurst ≤ 4 → use 2 LSBs of index
  - If > 4 → use 3 LSBs
- Max number of SS/PBCH blocks per half-frame defined in TS 38.213 §13

#### 7.3.3.2 Modulation
- Modulation is QPSK:
```
b_scrambled → QPSK → complex symbols d(0)...d(symb-1)
```
## 7.4 Physical signals
### 7.4.1 Reference signals
#### 7.4.1.1 Demodulation reference signals for PDSCH
##### 7.4.1.1.1 Sequence generation
- The complex DM-RS sequence r(n) is generated from a pseudo-random sequence c(i).
  - r(n) = (1/sqrt(2)) * (c(2n) + j * c(2n+1))
- The pseudo-random generator is initialized with a value c_init.
- Initialization Formula c_init:
```
c_init = (2^17 * (n_symb_slot * (n_s,f_u) + l + 1) * (2 * N_ID + 1) * 2^5 + (2 * N_ID) + n_SCID) mod 2^31
```
- l: OFDM symbol number in the slot.
- n_s,f: Slot number in the frame.
- N_ID and n_SCID: Scrambling identities.
- Determining N_ID and n_SCID:
  - N_ID is derived from higher-layer parameters scramblingID0 and scramblingID1 in the DMRS-DownlinkConfig Information Element (IE). The specific N_ID used depends on the Downlink Control Information (DCI) format (1_0, 1_1, or 1_2).
  - n_SCID is typically given by the DM-RS sequence initialization field in the DCI. If that field is not present, n_SCID is 0.

##### 7.4.1.1.2 Mapping to Physical Resources 
- DM-RS is mapped to physical resources based on dmrs-Type (Configuration Type 1 or Type 2).
- The sequence is scaled by a power factor beta_DMRS_PDSCH.
- The mapping to resource elements a_k,l depends on antenna port p, time-domain index l', and frequency-domain index k'.
- Mapping Reference Points:
  - Frequency (k): The reference is subcarrier 0 of a specific Common Resource Block (CRB), typically CRB 0.
  - Time (l): The reference point and starting DM-RS symbol l0 depend on the PDSCH mapping type.
  - Mapping Type A: l is relative to the start of the slot. l0 is position 2 or 3.
  - Mapping Type B: l is relative to the start of the scheduled PDSCH resources. l0 is position 0.
- DM-RS Symbol Positions:
  - The exact positions of the DM-RS symbols are determined by the PDSCH duration (l_d) and additional position settings (dmrs-AdditionalPosition).
  - Tables 7.4.1.1.2-3 (single-symbol) and 7.4.1.1.2-4 (double-symbol) define the specific OFDM symbol locations (l) for different PDSCH durations and configurations.
- Single vs. Double Symbol DM-RS:
  - This is determined by the higher-layer parameter maxLength. If not configured, it's single-symbol. If set to 'len2', the DCI determines if it's single or double.

#### 7.4.1.3 Demodulation Reference Signals for PDCCH 
- The reference signal sequence r(l) is generated from a pseudo-random sequence c(i).
  - Formula: r(l) = (1/sqrt(2)) * (c(2i) + j*c(2i+1))
- The generator is initialized with c_init at the start of each OFDM symbol l.
  ```
   Initialization Formula (c_init): c_init = (2^21 * (n_s,f * N_symb_slot + l + 1) * (2 * N_ID + 1) + 2 * N_ID) mod 2^31
  ```
  - n_s,f is the slot number within a frame.
  - l is the OFDM symbol number within the slot.
  - N_ID is given by the higher-layer parameter pdcch-DMRS-ScramblingID. If not provided, it defaults to the physical cell ID.

**Mapping to Physical Resources**
- The sequence is mapped to resource elements a_k,l with a power scaling factor beta_PDCCH_DMRS.
- Mapping is done every 4th subcarrier (k = n * 4 + k').
- The mapping is confined to the resource-element groups (REGs) within the Control Resource Set (CORESET) where the UE is attempting to decode the PDCCH.
- The frequency reference point (k=0) is subcarrier 0 of the lowest-numbered resource block in the CORESET.
- The antenna port is fixed to p = 2000.
- Quasi Co-location (QCL): PDCCH DM-RS can be assumed to be QCL with the SS/PBCH block for Doppler, delay, and spatial parameters.

#### 7.4.1.4 Demodulation Reference Signals for PBCH 
- The reference signal sequence r(n) is generated from a pseudo-random sequence c(n).
- The generator is initialized with c_init at the start of each SS/PBCH block occasion.
- Initialization Formula (c_init):
 ```
  c_init = 2^11 * (i_SSB + 1) + 2^6 * (i_SSB + 1) + N_ID_cell
 ```
  - N_ID_cell is the physical cell ID.
  - i_SSB is derived from the candidate SS/PBCH block index and the half-frame number. Its calculation depends on whether the maximum number of SS/PBCH blocks is 4 or greater than 4.

#### 7.4.1.5 Channel State Information Reference Signals (CSI-RS) 
- There are two types of CSI-RS:
  - Non-Zero-Power (NZP) CSI-RS: Used for channel measurement. A sequence is generated and transmitted.
  - Zero-Power (ZP) CSI-RS: No signal is transmitted on these resource elements. The UE assumes these resources are not used for PDSCH, allowing it to measure interference.
**Sequence Generation (for NZP CSI-RS)**
- The sequence r(n) is generated from a pseudo-random sequence c(i).
- The generator is initialized with c_init at the start of each OFDM symbol containing CSI-RS.
- Initialization Formula (c_init):
```
c_init = (2^10 * (N_symb_slot * n_s,f + l + 1) * (2 * n_ID + 1) + n_ID) mod 2^31
```
  - n_s,f is the slot number within a radio frame.
  - l is the OFDM symbol number within a slot.
  - n_ID is given by higher-layer parameters scramblingID or sequenceGenerationConfig.

**Mapping to Physical Resources (for NZP CSI-RS)**
- The sequence is mapped to resource elements a_k,l with a power scaling factor beta_CSIRS.
- The mapping is highly flexible and defined by multiple higher-layer parameters.
- Location:
  - Time Domain: The OFDM symbols used are given by ```firstOFDMSymbolInTimeDomain```.
  - Frequency Domain: The resource blocks and subcarriers used are defined by a bitmap in ```frequencyDomainAllocation```.
- Density and Ports:
  - The density parameter controls how sparse or dense the CSI-RS is in the frequency domain.
  - The ```nrofPorts``` parameter specifies the number of antenna ports.
- Code Division Multiplexing (CDM):
  - Multiple antenna ports can be multiplexed onto the same time-frequency resources using orthogonal cover codes (OCC).
  - The cdm-Type parameter (e.g., 'noCDM', 'fd-CDM2', 'cdm4-FD2-TD2') specifies the CDM scheme.
  - Table 7.4.1.5.3-1 defines the resource element patterns for different numbers of ports and CDM types.
- Antenna Ports: CSI-RS ports are numbered starting from p = 3000.
- Periodicity: CSI-RS can be configured to be transmitted periodically based on a slot periodicity and offset (CSI-ResourcePeriodicityAndOffset).

Of course. Here are the notes based on the provided text.

### 7.4.2 Synchronization Signals
#### 7.4.2.1 Physical-layer Cell Identities 
- There are 1008 unique physical-layer cell identities (N_cell_ID).
- The cell ID is composed of two parts:
```
N_cell_ID = 3 * N_ID^(1) + N_ID^(2)
```
  - N_ID^(1) is a value from 0 to 335.
  - N_ID^(2) is a value from 0 to 2.

#### 7.4.2.2 Primary Synchronization Signal (PSS) 
- The PSS sequence is denoted as d_PSS(n) and has a length of 127.
- It is a BPSK-modulated m-sequence.
- Formula: d_PSS(n) = 1 - 2*x(m)
  - The index m depends on the N_ID^(2) component of the cell ID:
  ```
  m = (n + 43 * N_ID^(2)) mod 127
  ```
- The sequence x(i) is a length-127 m-sequence generated by a specific polynomial with a defined initial value. This allows the UE to determine N_ID^(2) by testing the three possible PSS sequences.


#### 7.4.2.3 Secondary Synchronization Signal (SSS) 
- The SSS sequence is denoted as d_SSS(n) and has a length of 127.
- It is generated by multiplying two different length-127 m-sequences (x_0 and x_1).
- Formula: d_SSS(n) = [1 - 2*x_0(m_0)] * [1 - 2*x_1(m_1)]
  - The indices m_0 and m_1 depend on the N_ID^(1) and N_ID^(2) components of the cell ID:
```
m_0 = (n + m') mod 127, where m' is a function of N_ID^(1).
m_1 = (n + N_ID^(1) mod 112) mod 127.
```
- The sequences x_0(i) and x_1(i) are generated by different polynomials with defined initial values. This structure allows the UE to determine the N_ID^(1) part of the cell ID.

### 7.4.3 SS/PBCH Block
#### 7.4.3.1 Time-Frequency Structure 
- An SS/PBCH block is a dedicated resource block for transmitting synchronization signals and the physical broadcast channel.
- Time Domain: The block spans 4 consecutive OFDM symbols, numbered 0 to 3.
- Frequency Domain: The block spans 240 contiguous subcarriers, numbered 0 to 239.
- A single antenna port, p = 4000, is used for all signals (PSS, SSS, PBCH) and channels within the block.
- All components within the block share the same subcarrier spacing and cyclic prefix length.
**Resource Mapping within the Block**
- The mapping of signals and channels to specific resource elements (REs) within the 4x240 grid is fixed, as defined in Table 7.4.3.1-1.
- Primary Synchronization Signal (PSS):
  - Location: Mapped to OFDM symbol 0.
  -Frequency: Occupies the central 127 subcarriers (from subcarrier 56 to 182).
- Secondary Synchronization Signal (SSS):
  - Location: Mapped to OFDM symbol 2.
  - Frequency: Also occupies the central 127 subcarriers (from subcarrier 56 to 182).
- Physical Broadcast Channel (PBCH):  
  - Location: Mapped across OFDM symbols 1 and 3, and the subcarriers in symbol 2 not used by the SSS.
  - The PBCH data is mapped to all REs in its assigned symbols that are not reserved for the PBCH DM-RS.
- PBCH Demodulation Reference Signal (DM-RS):  
  - Location: Interspersed within the PBCH symbols (OFDM symbols 1, 2, and 3).
  - Frequency: Mapped sparsely, occurring on every 4th subcarrier.
  - The starting position of the DM-RS pattern is shifted based on the cell ID: v = N_cell_ID mod 4. This allows a UE to get a rough idea of the cell ID early in the synchronization process.
- Unused Resources ("Set to 0"):
  - The subcarriers at the edges of symbols 0 and 2 (outside the PSS and SSS) are set to zero.

**Mapping Process for Signals and Channels**
- PSS/SSS Mapping:
   - The 127 symbols of the PSS and SSS sequences are scaled by power factors (beta_PSS and beta_SSS).
   - They are then mapped to their respective resource elements in increasing order of the subcarrier index k.
- PBCH/DM-RS Mapping:
   - The PBCH and its DM-RS symbol sequences are also scaled by their respective power factors.
   - They are mapped to their allocated resource elements by first increasing the subcarrier index k across the frequency domain, and then increasing the OFDM symbol index l.
# 8 SideLink
### 8.1.1 Overview of Physical Channels
- Sidelink physical channels carry information from higher layers using specific resource elements.
- Defined sidelink physical channels:
  - Physical Sidelink Shared Channel (PSSCH)
  - Physical Sidelink Broadcast Channel (PSBCH)
  - Physical Sidelink Control Channel (PSCCH)
  - Physical Sidelink Feedback Channel (PSFCH)

### 8.1.2 Overview of Physical Signals
- Sidelink physical signals are used by the physical layer but do not carry higher-layer information.
- Defined sidelink physical signals:
  - Demodulation Reference Signals (DM-RS)
  - Channel-State Information Reference Signal (CSI-RS)
  - Phase-Tracking Reference Signals (PT-RS)
  - Sidelink Primary Synchronization Signal (S-PSS)
  - Sidelink Secondary Synchronization Signal (S-SSS)

### 8.2.2 Numerologies
- Multiple OFDM numerologies are supported, defined by subcarrier spacing (μ) and cyclic prefix (CP), as per Table 8.2.2-1.
- Subcarrier spacing and CP are configured via higher-layer parameters `subcarrierSpacing-SL` and `cyclicPrefix-SL`.
- Table 8.2.2-1: Supported transmission numerologies
  - μ = 0: 15 kHz, Normal CP
  - μ = 1: 30 kHz, Normal CP
  - μ = 2: 60 kHz, Normal or Extended CP
  - μ = 3: 120 kHz, Normal CP

### 8.2.3 Frame Structure
- **Frames and Subframes**: Defined in clause 4.3.1.
- **Slots**: Defined in clause 4.3.2 for sidelink transmission.

### 8.2.4 Antenna Ports
- Antenna ports are defined in clause 4.4.1.
- Sidelink antenna ports:
  - PSSCH: Starting with 1000
  - PSCCH: Starting with 2000
  - CSI-RS: Starting with 3000
  - S-SS/PSBCH: Starting with 4000
  - PSFCH: Starting with 5000
- For DM-RS associated with PSBCH:
  - Channel inference is valid only if the PSBCH and DM-RS symbols are within the same S-SS/PSBCH block, same slot, and same block index (clause 8.4.3.1).
- For DM-RS associated with PSSCH:
  - Channel inference is valid only if the PSSCH and DM-RS symbols are within the same frequency resource and slot.

### 8.2.5 Resource Grid
- Defined in clause 4.4.2.
- Carrier bandwidth grid size (`N_grid^size,μ`) is configured by `carrierBandwidth-SL` for subcarrier spacing configuration μ.
- Starting position (`N_grid^start,μ`) is configured by `offsetToCarrier-SL`.
- DC subcarrier location is indicated by `txDirectCurrentLocation-SL`:
  - Values 0–3299: Specific DC subcarrier number.
  - Value 3300: DC subcarrier is outside the resource grid.
  - Value 3301: DC subcarrier position is undetermined.
  - Includes option for 7.5 kHz offset relative to the subcarrier center.

## 8.3  Physical Channels
### 8.3.1 Physical Sidelink Shared Channel (PSSCH)
##### 8.3.1.1 Scrambling
- For the single codeword (q = 0), the bit block b^(q)(0), ..., b^(q)(M_bit^(q)-1) is scrambled before modulation, where M_bit^(q) = M_bit,SCI2^(q) + M_bit,data^(q) 
- Scrambling follows the pseudo-code:
  ```
  set i = 0
  set k = 0
  while i < M_bit^(q)
      if b^(q)(i) = x // SCI placeholder bits
          b_tilde^(q)(i) = b_tilde^(q)(i-2)
          k = k + 1
      else
          b_tilde^(q)(i) = (b^(q)(i) + c(i-M_k,SCI2^(q))) mod 2
      end if
      i = i + 1
  end while
  ```
- Scrambling sequence c(i) is defined in clause 5.2.1.
- For 0 ≤ i < M_bit,SCI2^(q):
  - M_k,SCI2 = 0
  - Sequence generator initialized with c_init = 2^10 * n_ID + 1010, where n_ID = L_PSCCH mod 2^10, and L_PSCCH is the CRC decimal representation on the associated PSCCH (TS 38.212, clause 8.3.2).
- For M_bit,SCI2^(q) ≤ i < M_bit^(q):
  - M_k,SCI2 = M_bit,SCI2^(q)
  - Same initialization: c_init = 2^10 * n_ID + 1010.

#### 8.3.1.2 Modulation
- Scrambled bits are modulated into complex-valued symbols d^(q)(0), ..., d^(q)(M_symb^(q)-1), where M_symb^(q) = M_symb,1^(q) + M_symb,2^(q).
- For 0 ≤ i < M_bit,SCI2^(q): QPSK modulation (clause 5.1), M_symb,1^(q) = M_bit,SCI2^(q)/2.
- For M_bit,SCI2^(q) ≤ i < M_bit^(q): Modulation per Table 6.3.1.2-1, M_symb,2^(q) = M_bit,data^(q)/log_2(M), where M is the modulation order.
- Table 6.3.1.2-1: Supported modulation schemes:
  - QPSK: Modulation order 2
  - 16QAM: Modulation order 4
  - 64QAM: Modulation order 6
  - 256QAM: Modulation order 8

#### 8.3.1.3 Layer Mapping
- Layer mapping per clause 7.3.1.3 with number of layers ν ∈ {1,2}, resulting in x(i) = [x^(0)(i) ... x^(ν-1)(i)]^T, i = 0,1, ..., M_symb^layer-1.

#### 8.3.1.4 Precoding
- Vectors [x^(0)(i) ... x^(ν-1)(i)]^T are precoded per clause 6.3.1.5, with precoding matrix W as the identity matrix, M_symb^ap = M_symb^layer.

#### 8.3.1.5 Mapping to Virtual Resource Blocks
- For each antenna port, symbols y^(p)(0), ..., y^(p)(M_symb^ap-1) are scaled by amplitude factor β_DMRS^PSSCH (TS 38.213) and mapped to resource elements (k', l)_p,μ in assigned virtual resource blocks, where k' = 0 is the first subcarrier in the lowest-numbered virtual resource block.
- Mapping in two steps:
  1. Symbols for 2nd-stage SCI: Mapped in increasing order of k' (subcarrier index), then l (symbol index), starting from the first PSSCH symbol with DM-RS, excluding resource elements used for DM-RS, PT-RS, or PSCCH.
  2. Non-SCI symbols: Mapped in increasing order of k', then l, starting position per TS 38.214, excluding resource elements used for 2nd-stage SCI, DM-RS, PT-RS, or CSI-RS.
- Resource elements in the first OFDM symbol are duplicated in the immediately preceding symbol.

#### 8.3.1.6 Mapping from Virtual to Physical Resource Blocks
- Non-interleaved mapping: Virtual resource block n maps to physical resource block n.

### 8.3.2 Physical Sidelink Control Channel (PSCCH)

#### 8.3.2.1 Scrambling
- Bit block b(0), ..., b(M_bit-1) is scrambled into b_tilde(0), ..., b_tilde(M_bit-1) using:
  ```
  b_tilde(i) = (b(i) + c(i)) mod 2
  ```
- Scrambling sequence c(i) per clause 5.2.1, initialized with c_init = 1010.

#### 8.3.2.2 Modulation
- Scrambled bits are QPSK modulated (clause 5.1), resulting in symbols d(0), ..., d(M_symb-1), where M_symb = M_bit/2.

##### 8.3.2.3 Mapping to Physical Resources
- Symbols d(0), ..., d(M_symb-1) are scaled by β_PSCCH (TS 38.213) and mapped to resource elements (k, l)_p,μ assigned per clause 16.4 of TS 38.213, starting with d(0), in increasing order of k, then l, on antenna port p = 2000, excluding DM-RS resource elements.
- Resource elements in the first OFDM symbol are duplicated in the immediately preceding symbol.

#### 8.3.3 Physical Sidelink Broadcast Channel (PSBCH)

#### 8.3.3.1 Scrambling
- Bit block b(0), ..., b(M_bit-1) is scrambled into b_tilde(0), ..., b_tilde(M_bit-1) using:
  ```
  b_tilde(i) = (b(i) + c(i)) mod 2
  ```
- Scrambling sequence c(i) per clause 5.2.1, initialized with c_init = n_SL^ID at the start of each S-SS/PSBCH block.

#### 8.3.3.2 Modulation
- Scrambled bits are QPSK modulated (clause 5.1.3), resulting in symbols d_PSBCH(0), ..., d_PSBCH(M_symb-1), where M_symb = M_bit/2.

#### 8.3.3.3 Mapping to Physical Resources
- Mapping to physical resources is described in clause 8.4.3.

### 8.3.4 Physical Sidelink Feedback Channel (PSFCH)

#### 8.3.4.1 General
- Defines the PSFCH structure and operation.

#### 8.3.4.2 PSFCH Format 0

##### 8.3.4.2.1 Sequence Generation
- Sequence y(n) is generated as:
  ```
  y(n) = r_(α,δ)^(n_0, l')(n), n = 0,1, ..., N_sc^RB-1
  ```
- r_(α,δ)^(n_0, l')(n) per clause 6.3.2.2, with:
  - n_cs given by clause 16.3 of TS 38.213.
  - n_0 given by clause 16.3 of TS 38.213.
  - l is the OFDM symbol number in PSFCH transmission (l = 0 for the first symbol).
  - l' is the slot’s OFDM symbol index corresponding to the first PSFCH symbol (TS 38.213).

##### 8.3.4.2.2 Mapping to Physical Resources
- Sequence y(n) is scaled by β_PSFCH (TS 38.213) and mapped to resource elements (k, l)_p,μ assigned per clause 16.3 of TS 38.213, starting with y(0), in increasing order of k, then l, on antenna port p = 5000.
- Resource elements in the first OFDM symbol are duplicated in the immediately preceding symbol.

## 8.4: Physical Signals
### 8.4.1 Reference Signals
#### 8.4.1.1 Demodulation Reference Signals for PSSCH
##### 8.4.1.1.1 Sequence Generation
- The DM-RS sequence r(n) for PSSCH is generated as:
  ```
  r(n) = (1/sqrt(2)) * (1 - 2*c(2n)) + j * (1/sqrt(2)) * (1 - 2*c(2n+1))
  ```
- Pseudo-random sequence c(i) is defined in clause 5.2.1.
- Sequence generator initialized with:
  ```
  c_init = ((2^(N_symb^slot * n_s,f^μ + l + 1) * (2*n_ID + 1) + 2*n_ID) mod 2^31
  ```
  where:
  - l: OFDM symbol number within the slot.
  - n_s,f^μ: Slot number within a frame.
  - n_ID = L_PSCCH mod 2^10, where L_PSCCH is the decimal representation of CRC on the PSCCH associated with PSSCH (TS 38.212, clause 7.3.2).

##### 8.4.1.1.2 Mapping to Physical Resources
- Sequence r(n) is mapped to intermediate quantity a_tilde(k',l)_p,μ per clause 6.4.1.1.3, using configuration type 1 without transform precoding.
- Parameters w_f(k'), w_t(l'), and Δ are given by Table 8.4.1.1.2-2; r(n) per clause 8.4.1.1.1.
- DM-RS pattern is indicated in the SCI (TS 38.212, clause 8.3.1.1).
- Intermediate quantity a_tilde(k',l)_p,μ is precoded, scaled by β_DMRS^PSSCH (clause 8.3.1.5), and mapped to physical resources:
  ```
  [a(k,l)_p,μ, ..., a(k+P'-1,l)_p,μ]^T = β_DMRS^PSSCH * W * [a_tilde(k',l)_p,μ, ..., a_tilde(k'+P'-1,l)_p,μ]^T
  ```
  where:
  - W: Precoding matrix (clause 8.3.1.4).
  - {p_0, ..., p_P'-1}: Antenna ports (clause 8.3.1.4).
  - {p_tilde_0, ..., p_tilde_P'-1}: Antenna ports (TS 38.214).
  - Resource elements a_tilde(k',l)_p,μ are within common resource blocks allocated for PSSCH.
- k is relative to subcarrier 0 in common resource block 0; l is relative to the start of scheduled PSSCH/PSCCH resources, including the duplicated symbol (clauses 8.3.1.5, 8.3.2.3).
- DM-RS symbol positions (l) are given by Table 8.4.1.1.2-1, based on l_d (duration of PSSCH/PSCCH including duplicated symbol) and PSCCH duration (2 or 3 symbols).
- Table 8.4.1.1.2-1: PSSCH DM-RS time-domain location (example entries):
  - l_d = 9 symbols, 2-symbol PSCCH: DM-RS at symbols {3, 8} (2 DM-RS), {1, 4, 7} (3 DM-RS), {4, 8} (4 DM-RS).
  - l_d = 13 symbols, 3-symbol PSCCH: DM-RS at {4, 10} (2 DM-RS), {1, 6, 11} (3 DM-RS), {1, 4, 7, 10} (4 DM-RS).
- Table 8.4.1.1.2-2: Parameters for PSSCH DM-RS:
  - Antenna port 1000: CDM group λ=0, Δ=0, w_f(k')=[+1, +1], w_t(l')=[+1, +1].
  - Antenna port 1001: CDM group λ=0, Δ=0, w_f(k')=[+1, -1], w_t(l')=[+1, +1].

#### 8.4.1.2 Phase-Tracking Reference Signals for PSSCH

##### 8.4.1.2.1 Sequence Generation
- Precoded PT-RS for subcarrier k on layer λ:
  ```
  r(k') = r(n) if λ = λ_0 or λ = λ_1, else 0
  ```
  where:
  - Antenna ports {p_tilde_0, p_tilde_1} for PT-RS (TS 38.214, clause 8.2.3).
  - r(n) per clause 8.4.1.1.1 at DM-RS symbol positions.

##### 8.4.1.2.2 Mapping to Physical Resources
- PT-RS transmitted only in resource blocks used for PSSCH, if enabled (TS 38.214).
- PT-RS mapped to resource elements:
  ```
  [a(k,l)_p,μ, ..., a(k+P'-1,l)_p,μ]^T = β_DMRS^PSSCH * W * [r(k'), ..., r(k'+P'-1)]^T
  k = 4m + 2*k_0 + Δ
  ```
  where:
  - l is within PSSCH transmission symbols.
  - Resource elements (k, l) not used for CSI-RS, PSCCH, or PSSCH DM-RS.
  - k' and Δ correspond to {p_tilde_0, ..., p_tilde_P'-1}.
- Time indices l for PT-RS:
  1. Initialize i = 0, l_ref = 0.
  2. If interval max(l_ref + (i-1)*L_PT-RS + 1, l_ref) overlaps with DM-RS symbol:
     - Set i = 1, l_ref to DM-RS symbol index, repeat.
  3. Add l_ref + i*L_PT-RS to PT-RS time indices.
  4. Increment i, repeat while l_ref + i*L_PT-RS is within PSSCH allocation.
  - L_PT-RS ∈ {1, 2, 4} (TS 38.214, clause 8.4.3).
- Subcarrier mapping:
  ```
  k = k_ref^RE + (i/L_PT-RS + k_ref^RB) * N_sc^RB
  k_ref^RB = 0 if n_RB mod L_PT-RS = 0, else n_ID mod (n_RB mod L_PT-RS)
  ```
  where:
  - i = 0, 1, 2, ...
  - k_ref^RE per Table 8.4.1.2.2-1 (e.g., DM-RS port 0: {0, 2, 6, 8} for offsets).
  - n_RB: Number of scheduled resource blocks.
  - L_PT-RS ∈ {2, 4} (TS 38.214).
  - n_ID = L_PSCCH mod 2^10 (TS 38.212, clause 7.3.2).
- PT-RS not mapped to resource elements with PSCCH or PSCCH DM-RS (puncturing applied).
- No overlap with sidelink CSI-RS.

#### 8.4.1.3 Demodulation Reference Signals for PSCCH

##### 8.4.1.3.1 Sequence Generation
- DM-RS sequence r(n) for PSCCH:
  ```
  r(n) = (1/sqrt(2)) * (1 - 2*c(2n)) + j * (1/sqrt(2)) * (1 - 2*c(2n+1))
  ```
- Pseudo-random sequence c(i) per clause 5.2.1, initialized with:
  ```
  c_init = ((2^(N_symb^slot * n_s,f^μ + l + 1) * (2*n_ID + 1) + n_ID) mod 2^31
  ```
  where:
  - l: OFDM symbol number within the slot.
  - n_s,f^μ: Slot number within a frame.
  - n_ID ∈ {0, 1, ..., 65535} from higher-layer parameter sl-DMRS-ScrambleID.

##### 8.4.1.3.2 Mapping to Physical Resources
- Sequence r(n) scaled by β_DMRS^PSCCH (TS 38.213) and mapped to resource elements (k, l)_p,μ starting with r(0) on antenna port p = 2000:
  ```
  a(k,l)_p,μ = β_DMRS^PSCCH * w_f,k(n') * r(m)
  k = m*N_sc^RB + 4*n' + 1
  n' = 0, 1, 2
  m = 0, 1, ...
  ```
- Conditions:
  - Resource elements within PSCCH allocation.
- w_f,k(n') per Table 8.4.1.3.2-1 (e.g., n'=0: w_f,k=[1, 1, 1]; n'=1: w_f,k=[1, e^(j*2π/3), e^(-j*2π/3)]).
- n' randomly selected by UE from {0, 1, 2}.
- k relative to subcarrier 0 in common resource block 0; l is the OFDM symbol number in the slot.

#### 8.4.1.4 Demodulation Reference Signals for PSBCH

##### 8.4.1.4.1 Sequence Generation
- DM-RS sequence r(n) for S-SS/PSBCH block:
  ```
  r(n) = (1/sqrt(2)) * (1 - 2*c(2n)) + j * (1/sqrt(2)) * (1 - 2*c(2n+1))
  ```
- Pseudo-random sequence c(i) per clause 5.2.1, initialized with:
  ```
  c_init = n_SL^ID
  ```
  at the start of each S-SS/PSBCH block.

##### 8.4.1.4.2 Mapping to Physical Resources
- Mapping described in clause 8.4.3.

#### 8.4.1.5 CSI Reference Signals

##### 8.4.1.5.1 General
- Supports 1 or 2 antenna ports (ν ∈ {1, 2}).
- Density ρ = 1 supported; zero-power CSI-RS not supported.
- Amplitude scaling factor β_CSIRS per clause 8.2.1 of TS 38.214.

##### 8.4.1.5.2 Sequence Generation
- CSI-RS sequence r(n):
  ```
  r(n) = (1/sqrt(2)) * (1 - 2*c(2n)) + j * (1/sqrt(2)) * (1 - 2*c(2n+1))
  ```
- Pseudo-random sequence c(i) per clause 5.2.1, initialized with:
  ```
  c_init = ((2^(N_symb^slot * n_s,f^μ + l + 1) * (2*n_ID + 1) + n_ID) mod 2^31
  ```
  where:
  - n_s,f^μ: Slot number within a radio frame.
  - l: OFDM symbol number within a slot.
  - n_ID = L_PSCCH mod 2^10, where L_PSCCH is the CRC decimal representation for SCI on PSCCH (TS 38.212, clause 7.3.2).

##### 8.4.1.5.3 Mapping to Physical Resources
- Mapping per clause 7.4.1.5.3, with exceptions:
  - Only 1 or 2 antenna ports supported.
  - Density ρ = 1.
  - No zero-power CSI-RS.

### 8.4.2 Synchronization Signals

#### 8.4.2.1 Physical-Layer Sidelink Identities
- 672 unique sidelink identities:
  ```
  n_SL^ID = n_ID,1^SL + 336*n_ID,2^SL
  ```
  where n_ID,1^SL ∈ {0, 1, ..., 335}, n_ID,2^SL ∈ {0, 1}.
- Divided into two sets:
  - id_net: n_SL^ID = 0, 1, ..., 335
  - id_oon: n_SL^ID = 336, 337, ..., 671

#### 8.4.2.2 Sidelink Primary Synchronization Signal (S-PSS)

##### 8.4.2.2.1 Sequence Generation
- S-PSS sequence d_S-PSS(n):
  ```
  d_S-PSS(n) = 1 - 2*x(m)
  m = n + 22 + 43*n_ID,2^SL mod 127
  0 ≤ n < 127
  ```
  where:
  ```
  x(i+7) = (x(i+4) + x(i)) mod 2
  [x(6) x(5) x(4) x(3) x(2) x(1) x(0)] = [1 1 1 0 1 1 0]
  ```

##### 8.4.2.2.2 Mapping to Physical Resources
- Mapping described in clause 8.4.3.

#### 8.4.2.3 Sidelink Secondary Synchronization Signal (S-SSS)

##### 8.4.2.3.1 Sequence Generation
- S-SSS sequence d_S-SSS(n):
  ```
  d_S-SSS(n) = [1 - 2*x_0((n + m_0) mod 127)] * [1 - 2*x_1((n + m_1) mod 127)]
  m_0 = 15*floor(n_ID,SL/112) + 5*n_ID,1^SL
  m_1 = n_ID,SL mod 112
  0 ≤ n < 127
  ```
  where:
  ```
  x_0(i+7) = (x_0(i+4) + x_0(i)) mod 2
  x_1(i+7) = (x_1(i+1) + x_1(i)) mod 2
  [x_0(6) x_0(5) x_0(4) x_0(3) x_0(2) x_0(1) x_0(0)] = [0 0 0 0 0 0 1]
  [x_1(6) x_1(5) x_1(4) x_1(3) x_1(2) x_1(1) x_1(0)] = [0 0 0 0 0 0 1]
  ```

##### 8.4.2.3.2 Mapping to Physical Resources
- Mapping described in clause 8.4.3.

### 8.4.3 S-SS/PSBCH Block

#### 8.4.3.1 Time-Frequency Structure of an S-SS/PSBCH Block
- **Time Domain**:
  - S-SS/PSBCH block consists of N_symb^S-SSB OFDM symbols (0 to N_symb^S-SSB-1):
    - Normal CP: N_symb^S-SSB = 13
    - Extended CP: N_symb^S-SSB = 11
  - First OFDM symbol is the first in the slot.
- **Frequency Domain**:
  - 132 contiguous subcarriers (0 to 131).
  - k and l represent frequency and time indices within the S-SS/PSBCH block.
- **Antenna Port**:
  - Uses antenna port 4000 for S-PSS, S-SSS, PSBCH, and DM-RS.
  - Same cyclic prefix length and subcarrier spacing for all signals.
- **Resource Mapping** (Table 8.4.3.1-1):
  - S-PSS: Symbols 1, 2; subcarriers 2, 3, ..., 127, 128.
  - S-SSS: Symbols 3, 4; subcarriers 2, 3, ..., 127, 128.
  - Set to zero: Symbols 1, 2, 3, 4; subcarriers 0, 1, 129, 130, 131.
  - PSBCH: Symbols 0, 5, 6, ..., N_symb^S-SSB-1; subcarriers 0, 1, ..., 131.
  - DM-RS for PSBCH: Symbols 0, 5, 6, ..., N_symb^S-SSB-1; subcarriers 0, 4, 8, ..., 128.

##### 8.4.3.1.1 Mapping of S-PSS within an S-SS/PSBCH Block
- S-PSS sequence d_S-PSS(0), ..., d_S-PSS(126) scaled by β_S-PSS (TS 38.213) and mapped to resource elements (k, l)_p,μ in increasing order of k for symbols l per Table 8.4.3.1-1.

##### 8.4.3.1.2 Mapping of S-SSS within an S-SS/PSBCH Block
- S-SSS sequence d_S-SSS(0), ..., d_S-SSS(126) scaled by β_S-SSS (TS 38.213) and mapped to resource elements (k, l)_p,μ in increasing order of k for symbols l per Table 8.4.3.1-1.

##### 8.4.3.1.3 Mapping of PSBCH and DM-RS within an S-SS/PSBCH Block
- PSBCH symbols d_PSBCH(0), ..., d_PSBCH(M_symb-1) scaled by β_PSBCH (TS 38.213) and mapped to resource elements (k, l)_p,μ not used for DM-RS, in increasing order of k, then l, per Table 8.4.3.1-1.
- DM-RS symbols r(0), ..., r(33*(N_symb^S-SSB-4)-1) scaled by β_PSBCH^DM-RS (TS 38.213) and mapped to resource elements (k, l)_p,μ in increasing order of k, then l, per Table 8.4.3.1-1.
