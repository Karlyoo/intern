# LDPC  in OAI 5GNR
- [encode](https://github.com/Karlyoo/intern/blob/main/OAI/LDPC.md#encode)
- [decode](https://github.com/Karlyoo/intern/blob/main/OAI/LDPC.md#decode)
## Introduction
  * Low-Density Parity-Check Code(LDPC),it uses a mathematical matrix to check for and correct errors. technique called **iterative message passing**,
  * LDPC codes are defined by a parity-check matrix H, which consists of variable nodes and check nodes . The variable nodes send their  LLR (log-likelihood ratio) messages to the check nodes. The check nodes perform XOR operations to verify the parity constraints. This iterative message passing continues for about 10 to 50 iterations until the codeword satisfies all parity checks.
  * In 5G, LDPC codes have a specially structured parity-check matrix (PCM), which allows for efficient decoding algorithms. [38.212](https://www.etsi.org/deliver/etsi_ts/138200_138299/138212/17.01.00_60/ts_138212v170100p.pdf) page.11 provides a deeper understanding of the decoding process.

# Encode
## File list 
| File                        | Role                                      | Module Level     | Actual Content                                  |
|-----------------------------|-------------------------------------------|------------------|------------------------------------------------|
| `nr_ulsch_coding.c`         | **LDPC Encoding Main Program (Uplink)**    | PHY Layer        | Segmentation, CRC, LDPC encoding, rate matching |
| `nr_dlsch_coding.c`         | **LDPC Encoding Main Program (Downlink)**  | PHY Layer        | Segmentation, CRC, LDPC encoding, rate matching |

| Directory Path                          | Description                              | Function                            |
|-----------------------------------------|------------------------------------------|-------------------------------------|
| `ldpc_encoder.c`                        | Main controller of the LDPC encoding process | `LDPCencoder()`                   |
| `ldpc_encode_parity_check.c`            | Uses mod-2 operations, bit shifts, and cyclic shifts to construct the final codeword | |
| `nr_rate_matching.c`                    | Rate Matching                            | `nr_rate_matching_ldpc()`           |
| `nr_interleaving.c`                     | Bit Interleaving                         | `nr_interleaving_ldpc()`            |
| `nrLDPC_coding_segment_encoder.c`       | Processes segmented data for LDPC encoding | `nrLDPC_coding_encoder()`          |
| `nrLDPC_coding_segment_encoder.c`       | Creates encoding tasks for each Code Block (CB) | `nrLDPC_launch_TB_encoding()` |
| `nrLDPC_coding_segment_encoder.c`       | Performs encoding for a single 8-block    | `ldpc8blocks()`                     |
| `ldpc_encode_parity_check.c`            | Performs parity calculation              | `encode_parity_check_part_optim()`  |

## coding
**openair1/PHY/NR_UE_TRANSPORT/nr_ulsch_coding.c**
- Implements the 5G NR UE Uplink Shared Channel (ULSCH) encoding process.
- Data Preparation: Extracts TB configuration (e.g., size, modulation scheme, coding rate) from input parameters.
  
 ```
int nr_ulsch_encoding(PHY_VARS_NR_UE *ue,
                      NR_UE_ULSCH_t *ulsch,
                      const uint32_t frame,
                      const uint8_t slot,
                      unsigned int *G,
                      int nb_ulsch,
                      uint8_t *ULSCH_ids)
- ue: Points to the UE structure in the PHY layer, containing UE configuration and state.
- ulsch: Points to the ULSCH structure, containing uplink shared channel configuration (e.g., PUSCH PDU).
- frame: Frame number being processed.
- slot: Slot number being processed.
- G: Number of encoded data bits (code block size), with each PUSCH corresponding to a G value.
- nb_ulsch: Number of PUSCH (Physical Uplink Shared Channel) instances to process.
- ULSCH_ids: Array of ULSCH IDs used to search for different ULSCH instances.
```

```
unsigned int crc = 1;
NR_UL_UE_HARQ_t *harq_process = &ue->ul_harq_processes[harq_pid];
const nfapi_nr_ue_pusch_pdu_t *pusch_pdu = &ulsch->pusch_pdu;
uint16_t nb_rb = pusch_pdu->rb_size;
uint32_t A = pusch_pdu->pusch_data.tb_size << 3;
uint8_t Qm = pusch_pdu->qam_mod_order;
// target_code_rate is in 0.1 units
float Coderate = (float)pusch_pdu->target_code_rate / 10240.0f;

LOG_D(NR_PHY, "ulsch coding nb_rb %d, Nl = %d\n", nb_rb, pusch_pdu->nrOfLayers);
LOG_D(NR_PHY, "ulsch coding A %d G %d mod_order %d Coderate %f\n", A, G[pusch_id], Qm, Coderate);
LOG_D(NR_PHY, "harq_pid %d, pusch_data.new_data_indicator %d\n", harq_pid, pusch_pdu->pusch_data.new_data_indicator);
Extracts relevant parameters from ulsch and ue, such as:
- harq_pid: HARQ (Hybrid Automatic Repeat reQuest) ID.
- nb_rb: Number of allocated Resource Blocks.
- Qm: Modulation Order (e.g., QPSK, 16QAM, etc.).
```

- CRC Addition: Adds CRC to the transport block for error detection.

```
if (A > NR_MAX_PDSCH_TBS) {
    crc = crc24a(harq_process->payload_AB, A) >> 8;
    harq_process->payload_AB[A >> 3] = ((uint8_t *)&crc)[2];
    harq_process->payload_AB[1 + (A >> 3)] = ((uint8_t *)&crc)[1];
    harq_process->payload_AB[2 + (A >> 3)] = ((uint8_t *)&crc)[0];
    B = A + 24;
} else {
    crc = crc16(harq_process->payload_AB, A) >> 16;
    harq_process->payload_AB[A >> 3] = ((uint8_t *)&crc)[1];
    harq_process->payload_AB[1 + (A >> 3)] = ((uint8_t *)&crc)[0];
    B = A + 16;
}
**Large blocks add 24 bits, small blocks add 16 bits.**
```

- Segmentation: Splits large transport blocks into smaller blocks to meet LDPC encoding requirements.
  
```
TB_parameters->Kb = nr_segmentation(harq_process->payload_AB,
                                    harq_process->c,
                                    B,
                                    &harq_process->C,
                                    &harq_process->K,
                                    &harq_process->Z,
                                    &harq_process->F,
                                    harq_process->BG);
**Input:**
- payload_AB: Data including CRC.
- B: Total number of bits (including CRC).
- BG: LDPC Base Graph, determining the encoding structure.
**Output:**
- C: Number of segments.
- K: Number of bits per segment.
- Z: Lifting Size of the LDPC segment.
- F: Number of filler bits (used when data bits are less than K for alignment).
- Kb: Number of data bit columns in the Base Graph.
**Segmentation ensures correct LDPC encoding.**
```


#### openair1/PHY/CODING/nrLDPC_coding
**nrLDPC_coding_segment**
- nrLDPC_coding_segment_encoder.c
- nr_rate_matching.c
- Called by nr_dlsch_coding.c/nr_dlsch_encoding()
  - Segmentation
  - CRC Attachment: Calls functions in crc_byte.c
  - Parameter Calculation: Z, K, etc.
  - Filler Bits (F): Number of zero-padding bits (if any).

ldpc_encoder.c
---

![image](https://hackmd.io/_uploads/SJgF9UVEgg.png)

FROM https://www.mdpi.com/1424-8220/21/18/6266

```
#include <stdlib.h>
#include <math.h>
#include <stdio.h>
#include <string.h>
#include "defs.h"
#include "assertions.h"
#include "openair1/PHY/CODING/nrLDPC_defs.h"
#include "openair1/PHY/CODING/nrLDPC_extern.h"
#include "ldpc_generate_coefficient.c"
```
1. `nrLDPC_defs.h` includes all the necessary parameters for the encoding process. eg.block length
2. `ldpc_generate_coefficient.c` 
```
int LDPCencoder(unsigned char **inputArray, unsigned char *outputArray, encoder_implemparams_t *impp)
{
  const unsigned char *input = inputArray[0];
  // channel input is the output of this function!
  unsigned char *output = outputArray;
  const int Zc = impp->Zc;
  const int Kb = impp->Kb;
  const short block_length = impp->K;
  const short BG = impp->BG;
  const uint8_t gen_code = impp->gen_code;
  uint8_t c[22*384]; //padded input, unpacked, max size
  uint8_t d[68 * 384]; // coded output, unpacked, max size
```
**Encoding parameters:**
1. Kb: The number of information bits,the original information bits that you want to transmit before encoding.
2. K=Kb+padding. If the Kb does not match the LDPC structure,we have to put 0s to match it.

  
  ```
const short *Gen_shift_values = choose_generator_matrix(BG, Zc);
  if (Gen_shift_values==NULL) {
    printf("ldpc_encoder_orig: could not find generator matrix\n");
    return(-1);
  }
```

KEY parameters in ldpc encode 
| Parameter | Description                                                  |
|-----------|--------------------------------------------------------------|
| BG, Zc, Kb | Define the structure of the parity-check matrix              |
| K, F      | Define the actual input length and number of filler bits for the LDPC encoder |
| C         | Number of segments (Code Blocks)                            |
| d[]       | Intermediate buffer for encoded output, sized 68*384 (maximum possible encoded bits) |
| c[]       | Output pointer for each segment, written back to the final output |


**Spilt to code word.**
Convert the byte-based input data (input[]) into a bit-level array (c[]) to enable bit-wise LDPC encoding.
```
for (i=0; i<block_length; i++)
  {
    // c[i] = input[i/8]<<(i%8);
    // c[i]=c[i]>>7&1;
    c[i] = (input[i / 8] & (128 >> (i & 7))) >> (7 - (i & 7));
  }
```
**Read cycle shift**
use 4 integer
i1: row index of parity-check matrix
i2: Bit index within Zc (cyclic shift index)
i3: column index of parity-check matrix
i4: Multiple shift values per matrix cell
```
for (i1=0; i1 < nrows-no_punctured_columns; i1++)
      {
        unsigned char channel_temp = 0;

        for (i3 = 0; i3 < Kb; i3++) {
          temp_prime = i1 * ncols + i3;

          for (i4 = 0; i4 < no_shift_values[temp_prime]; i4++) {
            channel_temp = channel_temp ^ c[i3 * Zc + Gen_shift_values[pointer_shift_values[temp_prime] + i4]];
          }
        }
```
**Obtain the Parity Code Block 1 / 2**
- gen_code == 0 → scalar XOR shift ：
- gen_code >= 1 →  SIMD C code（AVX512, AVX2...）
```
else if (gen_code == 1 && (Zc&31)==0) {
      shift=5; // AVX2 - 256-bit SIMD
      mask=31;
      strcpy(data_type,"simde__m256i");
      strcpy(xor_command,"simde_mm256_xor_si256");
    }
```
**Connect Code word**
Combine output and parity bits.
 ```
memcpy(&output[0], &c[2 * Zc], block_length - 2 * Zc);
  memcpy(&output[block_length - 2 * Zc], &d[0], (nrows - no_punctured_columns) * Zc - removed_bit);
  // memcpy(output,c,Kb*Zc*sizeof(unsigned char));
  return block_length - 2 * Zc + (nrows - no_punctured_columns) * Zc - removed_bit;
```


Here is the translation of the Chinese parts into English, integrated into the relevant sections of your provided text:

---

## LDPC File List
| File | Role | Module Level | Actual Content |
|------|--------------|-----------------------|---------------------------------------------|
| `nr_ulsch_coding.c` | **LDPC Encoding Main Program (Uplink)** | PHY Layer | Segmentation, CRC, LDPC encoding, rate matching |
| `nr_dlsch_coding.c` | **LDPC Encoding Main Program (Downlink)** | PHY Layer | Segmentation, CRC, LDPC encoding, rate matching |

| Directory Path | Description | Function |
|----------------|-------------|----------|
| `ldpc_encoder.c` | Main controller of the LDPC encoding process | `LDPCencoder()` |
| `ldpc_encode_parity_check.c` | Uses mod-2 operations, bit shifts, and cyclic shifts to construct the final codeword | |
| `nr_rate_matching.c` | **Rate Matching** | `nr_rate_matching_ldpc()` |
| `nr_interleaving.c` | **Bit Interleaving** | `nr_interleaving_ldpc()` |
| `nrLDPC_coding_segment_encoder.c` | **Processes segmented data for LDPC encoding** | `nrLDPC_coding_encoder()` |
| `nrLDPC_coding_segment_encoder.c` | **Creates encoding tasks for each Code Block (CB)** | `nrLDPC_launch_TB_encoding()` |
| `nrLDPC_coding_segment_encoder.c` | **Performs encoding for a single 8-block** | `ldpc8blocks()` |
| `ldpc_encode_parity_check.c` | **Performs parity calculation** | `encode_parity_check_part_optim()` |

---

### Coding
#### openair1/PHY/NR_UE_TRANSPORT/nr_ulsch_coding.c
Implements the 5G NR UE Uplink Shared Channel (ULSCH) encoding process.

- **Data Preparation**: Extracts TB configuration (e.g., size, modulation scheme, coding rate) from input parameters.
```
int nr_ulsch_encoding(PHY_VARS_NR_UE *ue,
                      NR_UE_ULSCH_t *ulsch,
                      const uint32_t frame,
                      const uint8_t slot,
                      unsigned int *G,
                      int nb_ulsch,
                      uint8_t *ULSCH_ids)
- ue: Points to the UE structure in the PHY layer, containing UE configuration and state.
- ulsch: Points to the ULSCH structure, containing uplink shared channel configuration (e.g., PUSCH PDU).
- frame: Frame number being processed.
- slot: Slot number being processed.
- G: Number of encoded data bits (code block size), with each PUSCH corresponding to a G value.
- nb_ulsch: Number of PUSCH (Physical Uplink Shared Channel) instances to process.
- ULSCH_ids: Array of ULSCH IDs used to search for different ULSCH instances.
```

```
unsigned int crc = 1;
NR_UL_UE_HARQ_t *harq_process = &ue->ul_harq_processes[harq_pid];
const nfapi_nr_ue_pusch_pdu_t *pusch_pdu = &ulsch->pusch_pdu;
uint16_t nb_rb = pusch_pdu->rb_size;
uint32_t A = pusch_pdu->pusch_data.tb_size << 3;
uint8_t Qm = pusch_pdu->qam_mod_order;
// target_code_rate is in 0.1 units
float Coderate = (float)pusch_pdu->target_code_rate / 10240.0f;

LOG_D(NR_PHY, "ulsch coding nb_rb %d, Nl = %d\n", nb_rb, pusch_pdu->nrOfLayers);
LOG_D(NR_PHY, "ulsch coding A %d G %d mod_order %d Coderate %f\n", A, G[pusch_id], Qm, Coderate);
LOG_D(NR_PHY, "harq_pid %d, pusch_data.new_data_indicator %d\n", harq_pid, pusch_pdu->pusch_data.new_data_indicator);
Extracts relevant parameters from ulsch and ue, such as:
- harq_pid: HARQ (Hybrid Automatic Repeat reQuest) ID.
- nb_rb: Number of allocated Resource Blocks.
- Qm: Modulation Order (e.g., QPSK, 16QAM, etc.).
```

- **CRC Addition**: Adds CRC to the transport block for error detection.
```
if (A > NR_MAX_PDSCH_TBS) {
    crc = crc24a(harq_process->payload_AB, A) >> 8;
    harq_process->payload_AB[A >> 3] = ((uint8_t *)&crc)[2];
    harq_process->payload_AB[1 + (A >> 3)] = ((uint8_t *)&crc)[1];
    harq_process->payload_AB[2 + (A >> 3)] = ((uint8_t *)&crc)[0];
    B = A + 24;
} else {
    crc = crc16(harq_process->payload_AB, A) >> 16;
    harq_process->payload_AB[A >> 3] = ((uint8_t *)&crc)[1];
    harq_process->payload_AB[1 + (A >> 3)] = ((uint8_t *)&crc)[0];
    B = A + 16;
}
**Large blocks add 24 bits, small blocks add 16 bits.**
```

- **Segmentation**: Splits large transport blocks into smaller blocks to meet LDPC encoding requirements.
```
TB_parameters->Kb = nr_segmentation(harq_process->payload_AB,
                                    harq_process->c,
                                    B,
                                    &harq_process->C,
                                    &harq_process->K,
                                    &harq_process->Z,
                                    &harq_process->F,
                                    harq_process->BG);
**Input:**
- payload_AB: Data including CRC.
- B: Total number of bits (including CRC).
- BG: LDPC Base Graph, determining the encoding structure.
**Output:**
- C: Number of segments.
- K: Number of bits per segment.
- Z: Lifting Size of the LDPC segment.
- F: Number of filler bits (used when data bits are less than K for alignment).
- Kb: Number of data bit columns in the Base Graph.
**Segmentation ensures correct LDPC encoding.**
```

#### openair1/PHY/CODING/nrLDPC_coding
**nrLDPC_coding_segment**
- `nrLDPC_coding_segment_encoder.c`
- `nr_rate_matching.c`
- **Called by `nr_dlsch_coding.c`/`nr_dlsch_encoding()`**
  - **Segmentation**
  - **CRC Attachment**: Calls functions in `crc_byte.c`
  - **Parameter Calculation**: Z, K, etc.
  - **Filler Bits (F)**: Number of zero-padding bits (if any).

---

### Key Parameters in LDPC Encoding
| Parameter | Description |
|-----------|-------------|
| BG, Zc, Kb | Define the structure of the parity-check matrix |
| K, F | Define the actual input length and number of filler bits for the LDPC encoder |
| C | Number of segments (Code Blocks) |
| d[] | Intermediate buffer for encoded output, sized 68*384 (maximum possible encoded bits) |
| c[] | Output pointer for each segment, written back to the final output |

---

### nrLDPC_coding_segment_encoder.c
- **Mainly responsible for the top-level integration of the entire LDPC encoding process, serving as the master control module for 5G NR uplink/downlink physical layer encoding.**
- **nrLDPC_coding_encoder**  
  └── **nrLDPC_launch_TB_encoding**  
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── **ldpc8blocks**

- **`nrLDPC_coding_encoder`**: Performs LDPC encoding for all Transport Blocks (TBs) in a slot and writes the encoded output to the corresponding buffer.
- **`nrLDPC_launch_TB_encoding`**: Controls the LDPC encoding task flow for a complete Transport Block (TB), creating and dispatching thread tasks to `ldpc8blocks()` to process every 8 segments.
- **`ldpc8blocks()`**: Handles up to 8 LDPC segments in a single thread.
  - Processes all Transport Blocks (TBs) in a slot according to 5G NR standards:
    - **LDPC Encoding**: `LDPCencoder(c, d, impp)`
    - **Rate Matching**: `nr_rate_matching_ldpc()`
    - **Bit Interleaving**: `nr_interleaving_ldpc()`
    - **Output Result**
  - Each task processes 8 segments.

# Decode
- **Program Entry Point**: `nr_dlsch_decoding.c`
### nr_dlsch_decoding.c
```
void nr_ue_dlsch_init(NR_UE_DLSCH_t *dlsch_list, int num_dlsch, uint8_t max_ldpc_iterations) {
  for (int i=0; i < num_dlsch; i++) {
    NR_UE_DLSCH_t *dlsch = dlsch_list + i;
    memset(dlsch, 0, sizeof(NR_UE_DLSCH_t));
    dlsch->max_ldpc_iterations = max_ldpc_iterations;
  }
}
```

- **Initialize DLSCH Channel Structure**
  - **Loop Processing**: Iterates through all DLSCH structures that need initialization.
  - **Clear Memory**: Uses `memset` to set all memory for each structure to 0, ensuring no residual data remains.
  - **Set Parameters**: Writes the input `max_ldpc_iterations` parameter into the structure. This is a critical setting for the LDPC decoding algorithm, affecting decoding performance and computational complexity.

```
void nr_dlsch_unscrambling(int16_t *llr, uint32_t size, uint8_t q, uint32_t Nid, uint32_t n_RNTI)
{
  nr_codeword_unscrambling(llr, size, q, Nid, n_RNTI);
}
```

- **Perform Descrambling**
  - `llr`: Soft bits (Log-Likelihood Ratios, LLRs) output from the demodulator, representing the likelihood of each bit being 0 or 1.
  - `size`: Size of the LLR array.
  - `q`: Index of the codeword, as a transport block may correspond to multiple codewords.
  - `Nid` and `n_RNTI`: Parameters required to generate the same scrambling sequence as the transmitter, representing the cell ID and the user's temporary identifier, respectively.

```
void nr_dlsch_decoding(PHY_VARS_NR_UE *phy_vars_ue,
                       const UE_nr_rxtx_proc_t *proc,
                       NR_UE_DLSCH_t *dlsch,
                       int16_t **dlsch_llr,
                       uint8_t **b,
                       int *G,
                       int nb_dlsch,
                       uint8_t *DLSCH_ids)
```

- **Execute LDPC Decoding**: Performs LDPC decoding on the input LLRs to correct errors.
- **Rate Dematching**: Restores the rate matching operations applied before transmission.
- **CRC Check**: After decoding, calculates the Cyclic Redundancy Check (CRC) for the data block. If the calculated CRC matches the included CRC, decoding is successful; otherwise, it indicates a decoding failure.

![LDPC Decoding Flow](https://github.com/user-attachments/assets/9fa47ab7-b330-4b46-8d48-b4fb31d5da2d)

### LDPC Decoding Process

```
| Process Step                   | Description                                      |
|--------------------------------|--------------------------------------------------|
| Start Decoder                  | Initiate the decoding process                    |
| Load Parameters / Buffers      | Set up C, K, Z, E, F, LLR, and other information |
| BN → CN Initial Processing      | Variable nodes pass information to Check nodes   |
| CN → BN Initial Processing      | Check nodes compute parity and return to variable nodes |
| First Parity Check             | Attempt to decode the correct codeword           |
| Output Bits                    | Output b[] if successful, otherwise retransmit   |
| Iterative Loop                 | Perform up to `max_ldpc_iterations` iterations   |
| CN Processing                  | Update all parity constraints                   |
| CN→BN Return                   | Pass back to variable nodes                     |
| BN Processing                  | Update variable node information (soft LLR)      |
```

- **Main Entry Function**: `nrLDPC_decoder_core` in `nrLDPC_decoder.c`

- **Decoder Setup**
  - **Load LUT (Lookup Tables)**: Used for subsequent CN/BN computations.
  - **Initialize Buffers**: Allocates memory space for CN, BN, and LLRs.
  - **Data Alignment and SIMD Preparation**: Facilitates acceleration using AVX instructions.

#### LUT
- **Definition**: A Lookup Table (LUT) is a pre-constructed table of data that allows algorithms to quickly retrieve corresponding values using an index, avoiding repeated calculations.
- **Why Use LUT?**
  - Fast retrieval speed.
  - Saves computational resources.
  - Suitable for well-defined mapping relationships.

#### SIMD
- **Definition**: Single Instruction, Multiple Data (SIMD) is a CPU vectorization technique that allows a single instruction to process multiple data elements simultaneously.
- **How SIMD is Used in LDPC Decoding**

```
| Stage                     | SIMD Acceleration Usage                          |
|---------------------------|-------------------------------------------------|
| LLR Input Processing      | Accelerates batch computation and access of LLRs for bits or symbols |
| BN→CN Message Passing     | Processes multiple variable node messages simultaneously |
| CN→BN Message Passing     | Compares minimum values (min-sum) using SIMD for parallel min() operations |
| CRC Calculation           | Performs multi-bit parallel XOR operations      |
| LLR Demodulation (Modulator) | Uses SIMD to accelerate inner product calculations for IQ to LLR conversion |
```

- **First Pass Processing**
  - BN Pre-processing with `nrLDPC_bnProcPc_*` (rate & BG-specific).
  - CN Pre-processing via `nrLDPC_bn2cnProcBuf_*`.
  - First CN processing: `nrLDPC_cnProc_*`.
  - Back-propagate to BN via `nrLDPC_cn2bnProcBuf_*`.

- **Iterative Decoding Loop**
  **Execution Conditions**:
  - `numIter < numMaxIter`
  - Parity check (or CRC) failed

  **Steps per Iteration**:
  - CN computation.
  - Pass CN messages back to BN.
  - BN computation (update soft LLR).
  - Prepare input for the next CN round.
  - Perform Parity Check or CRC verification.

- **Early Termination**
  - **If `check_crc` is enabled**:
    - After decoding, uses CRC to verify hard-decision data.
    - If verification passes, terminates the loop early.
  - **If CRC is not enabled**:
    - Performs parity check each time to determine whether to terminate early.

- **Output Formatting**
  - **Output Modes**: `LLRINT8`, `BIT`, or `BITINT8`.


