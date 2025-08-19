# UE Tx and RX
## Architecture
```
【 Uplink: UE TX (Simulation Transmitter) 】       【 Downlink: UE RX (Simulation Receiver) 】

Input: MAC PDU (bytes)                      ┌──────────────────────────────┐
        ↓                                   │ OFDM FFT + CP remove         │
┌──────────────────────────────┐            │ Input: RX samples            │
│ CRC attachment (crc_byte.c)  │            │ Output: freq-domain symbols  │
│ Output: bits + CRC           │            └────────────┬─────────────────┘
└────────────┬─────────────────┘                         ↓
             ↓                            ┌──────────────────────────────┐
┌──────────────────────────────┐         │ channel_estimation           │
│ Segmentation (nr_segmentation.c)│       │ Output: equalized symbols    │
│ Output: LDPC segments (bits)  │         └────────────┬─────────────────┘
└────────────┬─────────────────┘                         ↓
             ↓                            ┌──────────────────────────────┐
┌──────────────────────────────┐         │ Demodulation + LLR calculate │
│ LDPC encode (nr_dlsch_coding.c)│       │ Output: soft bits (LLRs)     │
│ Output: encoded bits          │         └────────────┬─────────────────┘
└────────────┬─────────────────┘                         ↓
             ↓                            ┌──────────────────────────────┐
┌──────────────────────────────┐         │ Rate Dematching              │
│ Rate Matching + Interleaving │         │ Input: LLRs                  │
│ Output: RM bits              │         │ Output: code blocks          │
└────────────┬─────────────────┘         └────────────┬─────────────────┘
             ↓                                                    ↓
┌──────────────────────────────┐         ┌──────────────────────────────┐
│ Scrambling                   │         │ LDPC decoding                │
│ Output: RM bits              │         │ (nr_dlsch_decoding.c)        │
└────────────┬─────────────────┘         │ Output: decoded bits         │
             ↓                           └────────────┬─────────────────┘
┌──────────────────────────────┐                      ↓
│ Modulation: QPSK, 16QAM       │       ┌──────────────────────────────┐
│ Output: complex symbols (IQ) │       │ CRC CHECK + merge segments    │
└────────────┬─────────────────┘       │ Output: MAC SDU (bytes)       │
             ↓                         └──────────────────────────────┘
┌──────────────────────────────┐
│ Layer Mapping                │
│ Output: complex symbols (IQ) │
└────────────┬─────────────────┘
             ↓
┌────────────────────────────────────────┐
│ Insert DMRS (UE uplink reference signal) │
│ Output: complex symbols (IQ)             │
└────────────┬────────────────────────────┘
             ↓
┌────────────────────────────────────────┐
│ DFT-s-OFDM IFFT + CP (ofdm_mod.c)      │
│ Output: time-domain samples            │
└────────────┬────────────────────────────┘
             ↓
┌──────────────────────────────┐
│ Simulate channel             │
│ Input: TX samples            │
│ Output: RX samples           │
└──────────────────────────────┘
```

## openairinterface5g/openair1/PHY
| Subfolder                      | Function                                               | Notes                                                                 |
| ------------------------------ | ------------------------------------------------------ | --------------------------------------------------------------------- |
| `NR_TRANSPORT/`                | 5G NR gNB Transport                                    |                                                                       |
| `NR_UE_TRANSPORT/`             | UE Transport functions                                 |                                                                       |
| `NR_REFSIG/`                   | NR reference signal module                             | Includes **DMRS, PTRS, PRACH, SSB** waveform generation and insertion |
| `MODULATION/`                  | OFDM IFFT/FFT, MODULATION, MAPPING                     |                                                                       |
| `TOOLS/`                       | Channel estimation, vector ops, FFT tools, phase noise |                                                                       |
| `INIT/`                        | Layer 1 variable initialization                        | Called in `phy_init_nr_ue()`                                          |
| `CODING/`                      | LDPC, Polar ENCODING/DECODING, CRC, SEGMENT            |                                                                       |
| `defs.h`, `extern.h`, `vars.h` | Global definitions and variable references             | Shared across modules, depends on these three files                   |

| Function                         | Specification                                                       | OAI Function / File                                                      |
| -------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Channel Estimation**           | TS 38.211 §7.4.1.1 (DM-RS)                                          | `nr_pbch_channel_estimation.c`<br>`nr_pdsch_channel_estimation.c`        |
| **Demodulation + LLR calculate** | TS 38.211 §7.3.1.4 (modulation)<br>TS 38.212 §7.1.4 (LLR calculate) | `nr_dlsch_llr_computation.c`<br>`nr_qpsk_llr.c`, `nr_qam16_llr.c`        |
| **Rate Dematching**              | TS 38.212 §5.4.1 (DL)                                               | `nr_rate_matching_ldpc.c`<br>`nr_dlsch_decoding.c`                       |
| **LDPC decoding**                | TS 38.212 §5.3.2                                                    | `nrLDPC_decoder.c`<br>`ldpc_decode.c`<br>called in `nr_dlsch_decoding.c` |
| **CRC check + merge Segments**   | TS 38.212 §5.1 (CRC)<br>§5.2.2 (Code block segmentation merge)      | `crc_byte.c`, `check_crc.c`<br>Merged inside `nr_dlsch_decoding.c`       |

## openair1/PHY/NR_UE_TRANSPORT/ //NR UE transport channel procedures are here

| File name                 | Function                                                                        |
| ------------------------- | ------------------------------------------------------------------------------- |
| `nr_dci.c`                | Handles DCI format (e.g., format 1\_0/1\_1) and packing                         |
| `cic_filter_nr.c`         | Cascaded Integrator-Comb and FIR filter for decimation, reduces sample rate     |
| `nr_dlsch_decoding.c`     | UE downlink data reception: ldpc decode, HARQ update, CRC verification          |
| `nr_dlsch_demodulation.c` | UE downlink data reception: demapping, channel compensation, LLR computation    |
| `nr_initial_sync.c`       | UE initial synchronization, detect gNB                                          |
| `nr_initial_sync_sl.c`    | UE initial synchronization for SideLink mode                                    |
| `nr_prach.c`              | Generate PRACH preamble and matched filter reception                            |
| `nr_psbch_rx.c`           | UE reception and decoding of PSBCH (Physical Sidelink Broadcast Channel)        |
| `nr_psbch_tx.c`           | UE transmission of PSBCH (Physical Sidelink Broadcast Channel)                  |
| `nr_pbch.c`               | MIB packaging and PBCH reference insertion                                      |
| `pss_nr.c`                | Detect PSS (Primary Synchronization Signal, time sync)                          |
| `sss_nr.c`                | Detect SSS (Secondary Synchronization Signal) and decode                        |
| `nr_ue_rf_helpers.c`      | Configure RF card frequency and gain                                            |
| `nr_ulsch_coding.c`       | Implements ULSCH encoding: CRC, segmentation, LDPC encode, rate matching        |
| `nr_ulsch_ue.c`           | Handles UE ULSCH PHY procedures (encoding, modulation, etc.)                    |
| `nr_transport_proto_ue.h` | Declares functions related to DLSCH, ULSCH, PUCCH, PBCH, PRACH, PSBCH in PHY.   |
| `nr_transport_ue.h`       | Data structures related to UE transport.                                        |
| `pucch_nr.c`              | Handles UCI (e.g., SR, HARQ-ACK, CSI)                                           |
| `pucch_nr.h`              | Defines UE-side PUCH-related data structures and functions for UCI transmission |
| `srs_modulation_nr.c`     | Implements SRS generation and processing at UE side                             |
| `srs_modulation_nr.h`     | Defines UE-side SRS-related data structures and functions                       |

## UE UPLINK
### Mapping to 3GPP specifications and OAI functions
| Function           | 3GPP Spec Section        | OAI Function                                      |
| ------------------ | ------------------------ | ------------------------------------------------- |
| CRC attachment     | 38.212 §5.1              | crc24a()、crc24b().....                            |
| Segmentation       | 38.212 §5.2              | nr\_segmentation()                                |
| LDPC encoding      | 38.212 §5.2              | LDPCencoder(), nr\_ulsch\_encoding()              |
| Scrambling         | 38.211 §6.3.1.1          | `nr_pusch_codeword_scrambling()`                  |
| Modulation         | 38.211 §6.3.2            | `nr_modulation()`                                 |
| Layer Mapping      | 38.211 §6.3.1.2          | `nr_ue_layer_mapping`                             |
| Channel Estimation | 38.211 §6.4.1.1.2 (DMRS) | `nr_pbch_channel_estimation()`                    |
| LLR Calculate      | 38.212 attachment A      | `nr_dlsch_qpsk_llr()`, `nr_dlsch_16qam_llr()` ... |

## openairinterface5g/openair1/PHY/CODING
**crc_byte.c**

Defines 24 or other bit length CRC
<img width="889" height="253" alt="image" src="https://github.com/user-attachments/assets/184d8388-9041-4d15-8cf6-1f072b9850fe" />

Defines functions to attach CRC
<img width="749" height="345" alt="image" src="https://github.com/user-attachments/assets/f59d8d42-349b-402c-aae0-2f8081bc944f" />

**nr_segmentation.c**
 - Implements segmentation per 38.212 §5.2 for LDPC encoding

## LDPC Encoding
[ldpc_encoding](https://github.com/Karlyoo/LDPCinOAI/blob/main/ldpc_encodeanddecode.md#ldpc-encoder-code-in-oai-5gnr)

## Modulation
[modulation](https://github.com/Karlyoo/LDPCinOAI/blob/main/modulation%26demodulation.md#modulation)

## openair1/PHY/NR_UE_TRANSPORT/nr_ulsch_ue.c
- Implements PHY transmission procedures for 5G NR UE Uplink Shared Channel (ULSCH)
- Data preparation:
  - Receives transport block from MAC, extracts PUSCH config (resource allocation, DMRS/PTRS config, modulation, etc.).
- Encoding and scrambling:
  - CRC, segmentation, LDPC encoding, rate matching (nr_ulsch_encoding). Scrambling using pseudo-random sequence (nr_pusch_codeword_scrambling).
    
```
void nr_pusch_codeword_scrambling(uint8_t *in, uint32_t size, uint32_t Nid, uint32_t n_RNTI, bool uci_on_pusch, uint32_t* out)
void nr_pusch_codeword_scrambling_uci(uint8_t *in, uint32_t size, uint32_t Nid, uint32_t n_RNTI, uint32_t* out)
*If uci_on_pusch is true (i.e., PUSCH carries UCI), then `nr_pusch_codeword_scrambling_uci` is used; otherwise general `nr_codeword_scrambling` is used. LDPC encoding uses `nr_ulsch_encoding`.
```

- Modulation and layer mapping:
   - Map scrambled data to complex symbols (nr_modulation). Assign to multiple transport layers (nr_ue_layer_mapping).
- Transform precoding (optional):
   - If DFT-s-OFDM enabled, apply DFT (nr_dft) and use low PAPR DMRS sequences.
- Resource mapping:
   - Map data, DMRS, PTRS to PUSCH resource grid (map_symbols, map_current_symbol). Handles DMRS type (1/2), PTRS position, DC carrier special cases.
- Precoding and antenna mapping:
   - Apply precoding matrix, map layers to antennas (nr_layer_precoder). Output to freq-domain buffer txdataF.
