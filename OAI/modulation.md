# Modulation
## openair1/PHY/MODULATION

| File Name                | Function                                                                 |
|--------------------------|--------------------------------------------------------------------------|
| `beamforming.c`          | Implements beamforming functionality at the eNB end                      |
| `compute_bf_weights.c`   | Currently contains no data; possibly attempts to estimate DLCSI from ULCSI to calculate beamforming weights (beam_weights) |
| `modulation_UE.h`        | Defines core function interfaces for modulation/demodulation front-end processing at the UE physical layer |
| `modulation_eNB.h`       | Defines key function interfaces related to modulation and beamforming at the base station physical layer |
| `nr_modulation.c`        | Handles modulation, layer mapping, DFT (Discrete Fourier Transform), and precoding |
| `modulation_common.h`    | Common modulation constants used in NR UE                                |
| `modulation_extern.h`    | Declares external functions and variables used in the modulation module   |

### OFDM and Front-End Processing Modules


| File Name              | Function Category   | Description                                                  |
|------------------------|---------------------|--------------------------------------------------------------|
| `ofdm_mod.c`           | OFDM Modulation     | FFT/IFFT, CP insertion (common for NR/LTE)                   |
| `slot_fep.c`           | Slot FEP            | LTE slot front-end processing (used at receiver for FFT, etc.) |
| `slot_fep_mbsfn.c`     | Slot FEP            | MBSFN-specific slot pre-processing                           |
| `slot_fep_nr.c`        | Slot FEP (NR)       | 5G NR slot receiver pre-processing (FFT, SC-FDMA processing, etc.) |
| `slot_fep_ul.c`        | Slot FEP (UL)       | Uplink LTE slot processing logic                             |

```
A[Encoded Codewords] --> B[nr_modulation.c (QAM Mapping)]
B --> C[Resource Mapping]
C --> D[OFDM IFFT + CP insert (ofdm_mod.c)]
D --> E[Slot packing (slot_fep_nr.c)]
E --> F[RF or channel simulator]
```

### nr_modulation.c

- **Modulation (nr_modulation)**: Maps the encoded bitstream to QPSK, 16-QAM, 64-QAM, or 256-QAM complex symbols.
  - **Input Parameters**:
    - `in`: Input bitstream (in `uint32_t` format).
    - `length`: Number of bits.
    - `mod_order`: Modulation order (2 for QPSK, 4 for 16-QAM, 6 for 64-QAM, 8 for 256-QAM).
    - `out`: Output complex symbols (in `int16_t` format, representing real and imaginary parts in Q15 fixed-point format).
  - **Operation Principle**:
    Groups the input bitstream and queries predefined modulation tables (per 3GPP TS 38.211), such as `nr_qpsk_mod_table`, `nr_16qam_mod_table`, etc., to generate corresponding complex symbols.

```c
#if defined(__SSE2__)
case 2:
  nr_mod_table128 = (simde__m128i *)nr_qpsk_byte_mod_table;
  out128 = (simde__m128i *)out;
  for (i = 0; i < length / 8; i++)
    out128[i] = nr_mod_table128[in_bytes[i]];
  i = i * 8 / 2;
  nr_mod_table32 = (int32_t *)nr_qpsk_mod_table;
  while (i < length / 2) {
    const int idx = ((in_bytes[(i * 2) / 8] >> ((i * 2) & 0x7)) & mask);
    out32[i] = nr_mod_table32[idx];
    i++;
  }
  return;
#else
case 2:
  nr_mod_table32 = (int32_t *)nr_qpsk_mod_table;
  for (i = 0; i < length / mod_order; i++) {
    const int idx = ((in[i * 2 / 32] >> ((i * 2) & 0x1f)) & mask);
    out32[i] = nr_mod_table32[idx];
  }
  return;
#endif

/* QPSK maps every 2 bits to a complex symbol (e.g., {1+j, 1-j, -1+j, -1-j}), accelerated with SSE2. Each loop processes 8 bits, generating 4 complex symbols, stored in out. */
```

```c
case 6:
  if (length > (3 * 64))
    for (i = 0; i < length - 3 * 64; i += 3 * 64) {
      uint64_t x = *in64++;
      uint64_t x1 = x & 0xfff;
      *out64++ = nr_64qam_mod_table[x1];
      x1 = (x >> 12) & 0xfff;
      *out64++ = nr_64qam_mod_table[x1];
      // ... (process multiple 12-bit groups)
    }
  while (i + 24 <= length) {
    uint32_t xx = 0;
    memcpy(&xx, in_bytes + i / 8, 3);
    uint64_t x1 = xx & 0xfff;
    *out64++ = nr_64qam_mod_table[x1];
    x1 = (xx >> 12) & 0xfff;
    *out64++ = nr_64qam_mod_table[x1];
    i += 24;
  }
  if (i != length) {
    uint32_t xx = 0;
    memcpy(&xx, in_bytes + i / 8, 2);
    uint64_t x1 = xx & 0xfff;
    *out64++ = nr_64qam_mod_table[x1];
  }
  return;
/* For 64-QAM, every 6 bits map to a complex symbol. Processes 64 bits (uint64_t) at a time, containing multiple 6-bit groups, generating multiple 64-QAM symbols. Uses bit-shift operations (>>) and &mask to extract 12 bits (two 6-bit symbols), queries nr_64qam_mod_table, and each 64-QAM symbol is represented by two int16_t values, stored in out. */
```

- **Layer Mapping (`nr_layer_mapping`, `nr_ue_layer_mapping`)**: Distributes modulated symbols to multiple transmission layers, supporting MIMO (Multiple Input Multiple Output).
  - **Parameters**:
    - `nbCodes`: Number of codewords, typically 1 or 2, representing independent data flows.
    - `encoded_len`: Symbol length of each codeword.
    - `mod_symbs`: Modulated complex symbols in `c16_t` format (16-bit integers for real and imaginary parts, Q15 fixed-point), containing `nbCodes` codewords, each with `encoded_len` symbols.
    - `n_layers`: Number of transmission layers (1 to 4), corresponding to MIMO configuration.
    - `layerSz`: Number of symbols per layer.
    - `n_symbs`: Total number of symbols, representing the data volume to process.
    - `tx_layers`: Output layer data, in `c16_t` array format, with each layer storing `layerSz` symbols.
  - **Example with 4 Layers**: Data bits are distributed across four layers, each mapped to different antennas for MIMO transmission.
    - **Base Station (gNB)**: Uses `nr_layer_mapping` for downlink transmission (e.g., PDSCH), supporting 1 to 4 layers of MIMO, optimized with SIMD instructions (AVX512, AVX2) for high data rates and multi-antenna allocation.
    - **UE**: Uses `nr_ue_layer_mapping` for uplink transmission (e.g., PUSCH), supporting simpler mappings, typically 1 or 2 layers due to antenna and power constraints.

- **DFT (`nr_dft`)**: Performs DFT transformation on uplink data (applicable to DFT-s-OFDM, SC-FDMA).
  - Converts layer-mapped time-domain modulated symbols (e.g., QPSK or 16-QAM symbols) into frequency-domain data, generating a single-carrier-like signal to reduce PAPR. Provides frequency-domain data for subcarrier mapping, ensuring correct DFT-s-OFDM waveform generation.
  - **Parameters**:
    - `d`: Input time-domain data, in `int32_t` format (typically `c16_t`, i.e., 16-bit integers for real and imaginary parts in Q15 fixed-point format). These are modulated symbols from `nr_ue_layer_mapping`'s `tx_layers[l]`.
    - `z`: Output frequency-domain data, also in `int32_t` (`c16_t`) format, storing the DFT transformation results.
    - `Msc_PUSCH`: DFT size, equal to the number of allocated subcarriers, typically `N_RB × 12` (each RB contains 12 subcarriers).

- **Symbol Rotation (`perform_symbol_rotation`, `init_symbol_rotation`, `init_timeshift_rotation`)**: Corrects phase offsets in the frequency or time domain.
  - **`perform_symbol_rotation()`**:
    Calculates a complex rotation factor (phase compensation) = `e^(j2πf0t)` for each OFDM symbol, corresponding to TX/RX frequency offsets. Stored as integer-format complex numbers in `symbol_rotation[]`, representing the rotation compensation value for the l-th OFDM symbol, multiplied into each symbol at TX or RX.
  - **`init_symbol_rotation()`**:
    Calls `perform_symbol_rotation()` once for DL and UL, calculating rotation values for each symbol in an entire frame (or subframe).
  - **`init_timeshift_rotation()`**:
    At the receiver, the FFT starting point may not align perfectly with the actual symbol start, causing time-domain offsets that result in phase shifts in the frequency domain.

- **Precoding (`nr_layer_precoder`, `nr_layer_precoder_cm`, `nr_layer_precoder_simd`)**: Applies precoding matrices to adapt to multi-antenna transmission.
  - Performs Layer-to-Antenna transformation (linear precoding) based on the Precoding Matrix Indicator (PMI), weighting and summing each layer's data to allocate to each antenna.
  - **`c16_t nr_layer_precoder()`**:
    Uses a simple character matrix (e.g., '1', '-1', 'j', '-j') to weight and combine each layer's data, suitable for simplified precoding, such as 2x1 MIMO where layer 0 uses '1' and layer 1 uses 'j'.
  - **`c16_t nr_layer_precoder_cm()`**:
    Performs multiplication and accumulation based on complete complex weights (from PMI PDU). Each layer corresponds to a set of weights (Re + jIm), performing complex multiplication of each layer’s symbols with its weights, then summing results across all layers.
  - **`void nr_layer_precoder_simd()`**:
    Accelerates Layer-to-Antenna precoding using SIMD vectorized instructions.

## nr_dlsch_demodulation.c

```c
nr_rx_pdsch()
│
├─ nr_dlsch_channel_level()           ← Executed only for the first symbol of the entire PDSCH, calculates the average channel strength for each layer+antenna, used for subsequent scaling and combining.
│
├─ nr_dlsch_channel_compensation()    ← Equalization core: Multiplies frequency-domain symbols by the channel conjugate + calculates magnitude (|H|²) and divides for QAM scaling →
|                                        Output is rxdataF_comp[]: Compensated frequency-domain data for subsequent processing.
│
├─ nr_dlsch_detection_mrc()           ← If MRC is used, performs multi-antenna combining (non-spatial multiplexing).
│
├─ Modulation LLR Calculation: Calls based on modulation type
│   ├─ nr_dlsch_qpsk_llr()
│   ├─ nr_dlsch_qam16_llr()
│   └─ nr_dlsch_qam64_llr()
│    (Calculates soft-decision LLRs for the transport layer buffer based on modulation type: QPSK, 16QAM, 64QAM, or 256QAM.)
└─ LLR Output → Softbuffer → LDPC Decoding
```

```markdown
| Stage                   | Input Data                              | Output Data                              |
|-------------------------|----------------------------------------|-----------------------------------------|
| RB Extraction & Channel Estimation | `rxdataF`, `dl_ch_estimates`         | `rxdataF_ext`, `dl_ch_estimates_ext`    |
| Channel Scaling         | `dl_ch_estimates_ext`                 | Scaled channel estimate                 |
| Channel Compensation    | `rxdataF_ext`, `dl_ch_estimates_ext`  | `rxdataF_comp`, `dl_ch_mag*`            |
| MRC/MMSE                | `rxdataF_comp`, `dl_ch_mag*`, `rho`   | Demodulated stream, MMSE parameters      |
| LLR Calculation         | `rxdataF_comp`, `dl_ch_mag*`          | Per-symbol LLR buffer (`layer_llr`)      |
| Layer Mapping & Output  | `layer_llr`, TB config                | `llr[CW0]`, `llr[CW1]`                  |
```

```c
NR_DL_UE_HARQ_t *dlsch0_harq, *dlsch1_harq;
dlsch0_harq = &ue->dl_harq_processes[0][harq_pid];
```

- **Determines Active Transport Block (TB) Based on HARQ PID**:
  - If both HARQ processes are active, sets `codeword_TB0/1`.
  - Otherwise, processes only one TB.

```c
nr_dlsch_extract_rbs()
```

- Extracts samples corresponding to PDSCH RBs from the received `rxdataF` and extends them to:
  - `rxdataF_ext[]`: Received symbol samples.
  - `dl_ch_estimates_ext[]`: Extended channel estimations.

```c
nr_dlsch_channel_level()
```

- Calculates the energy value for each channel (each TX-RX antenna pair).
- Results are used for:
  - MRC combining.
  - LLR normalization.
  - `log2_maxh`: Maximum channel gain, used to adjust quantization bit depth.

```c
nr_dlsch_channel_compensation()
```

- Compensates the received signal (`rxdataF_ext`) using channel estimates.
- Stores results in `rxdataF_comp`: Compensated data, passed to LLR calculations.

```c
static void nr_dlsch_llr(uint32_t rx_size_symbol,
                         int nbRx,
                         uint sz,
                         int16_t layer_llr[][sz],
                         int32_t rxdataF_comp[][nbRx][rx_size_symbol * NR_SYMBOLS_PER_SLOT],
                         int32_t dl_ch_mag[rx_size_symbol],
                         int32_t dl_ch_magb[rx_size_symbol],
                         int32_t dl_ch_magr[rx_size_symbol],
                         NR_DL_UE_HARQ_t *dlsch0_harq,
                         NR_DL_UE_HARQ_t *dlsch1_harq,
                         unsigned char symbol,
                         uint32_t len,
                         NR_UE_DLSCH_t dlsch[2],
                         uint32_t llr_offset_symbol)
{
  switch (dlsch[0].dlsch_config.qamModOrder) {
    case 2 :
      for(int l = 0; l < dlsch[0].Nl; l++)
        nr_qpsk_llr(&rxdataF_comp[l][0][symbol * rx_size_symbol], layer_llr[l] + llr_offset_symbol, len);
      break;

    case 4 :
      for(int l = 0; l < dlsch[0].Nl; l++)
        nr_16qam_llr(&rxdataF_comp[l][0][symbol * rx_size_symbol], dl_ch_mag, layer_llr[l] + llr_offset_symbol, len);
      break;

    case 6 :
      for(int l=0; l < dlsch[0].Nl; l++)
        nr_64qam_llr(&rxdataF_comp[l][0][symbol * rx_size_symbol], dl_ch_mag, dl_ch_magb, layer_llr[l] + llr_offset_symbol, len);
      break;

    case 8:
      for(int l=0; l < dlsch[0].Nl; l++)
        nr_256qam_llr(&rxdataF_comp[l][0][symbol * rx_size_symbol], dl_ch_mag, dl_ch_magb, dl_ch_magr, layer_llr[l] + llr_offset_symbol, len);
      break;

    default:
      AssertFatal(false, "Unknown mod_order!!!!\n");
      break;
  }
}
```

- **Selects LLR Calculation Method Based on Received Modulation**:
  - Functions like `nr_256qam_llr` are stored in `nr_phy_common.c`.

