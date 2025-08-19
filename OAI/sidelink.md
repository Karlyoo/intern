# Sidelink(UE-UE)
**openair1/ — PHY layer**

| file                   |                                             |
| -------------------- | --------------------------------------------- |
| `nr_psbch_tx.c`      | PSBCH transmitter :encoding、modulation、RE mapping                    |
| `nr_psbch_rx.c`      | PSBCH receiver: demodulation、unscrambling、decoding                            |

**The current focus of the official OAI 5G implementation includes:**
- Deployment of uplink/downlink communication for gNB and UE
- O-RAN architecture with CU/DU separation
- Support for NR NSA/SA modes
- Integration of uplink modulation and Layer 1/Layer 2 functions
- **On the other hand, 5G Sidelink has not yet been included in the main development branch.**

## nr_psbch_tx.c
#### main: nr_tx_psbch()
- Update the PSBCH payload with the provided `frame_tx` and `slot_tx` from the UE (filling in the DFN and slot number).
- Retrieve the PSS, SSS, and DMRS sequences.
- Call `sl_map_pss_or_sss() `to map the PSS and SSS.
- Call `sl_generate_and_map_psbch()` to complete the PSBCH encoding,modulation and DMRS mapping.

#### sl_psbch_scrambling()
- Performs Gold sequence scrambling of PSBCH data according to TS 38.211.
- Generates the Gold sequence using `lte_gold_generic()`.
- Scrambles the entire input,output buffer (which is the **polar encoded** bit sequence) in 32-bit chunks.

#### sl_generate_and_map_psbch()
- Handles the complete PSBCH data processing flow, including:
- Polar encoding (TS 38.212):`polar_encoder_fast()`
  - SL-MIB CRC attachment
  - Polar encoding
  - Bit interleaving and rate matching
 ```
   polar_encoder_fast(&psbch_a_reversed,
                     (void *)encoder_output,
                     0,
                     0,
                     SL_NR_POLAR_PSBCH_MESSAGE_TYPE,
                     SL_NR_POLAR_PSBCH_PAYLOAD_BITS,
                     SL_NR_POLAR_PSBCH_AGGREGATION_LEVEL);
#ifdef SL_DEBUG
  for (int i = 0; i < SL_NR_POLAR_PSBCH_E_DWORD; i++)
    printf("encoderoutput[%d]: 0x%08x\t", i, encoder_output[i]);
  printf("\n");
#endif
  /// 38.211 Scrambling
  if (cp) { // EXT Cyclic prefix
    sl_psbch_scrambling(encoder_output, id, SL_NR_POLAR_PSBCH_E_EXT_CP); // for Extended Cyclic prefix
    num_psbch_modsym = SL_NR_POLAR_PSBCH_E_EXT_CP / mod_order;
    numsym = SL_NR_NUM_SYMBOLS_SSB_EXT_CP;
    AssertFatal(1 == 0, "EXT CP is not yet supported\n");
  } else { // Normal CP
    sl_psbch_scrambling(encoder_output, id, SL_NR_POLAR_PSBCH_E_NORMAL_CP); // for Cyclic prefix
    num_psbch_modsym = SL_NR_POLAR_PSBCH_E_NORMAL_CP / mod_order;
    numsym = SL_NR_NUM_SYMBOLS_SSB_NORMAL_CP;
  }
  
```
- QPSK modulation
- Mapping the modulated data symbols and DMRS symbols onto txF
  - One DMRS symbol every 4 REs (at RE indexes where index % 4 == 0)
  - Data symbols fill the other REs

## nr_psbch_rx.c 
#### main : nr_rx_psbch()
- Performs the complete PSBCH reception and decoding process.
- Initialization:
  - Allocates buffers for soft bits `psbch_e_rx` and unclipped data `psbch_unClipped` with extra space to handle polar decoder requirements.
  - Determines the number of symbols (numsym) based on the cyclic prefix (normal or extended).
- Symbol Processing Loop:
  - Iterates over PSBCH-carrying symbols (0, 5–12, skipping SL-PSS and SL-SSS symbols 1–4).
- For each symbol:
  - Calls nr_psbch_extract to extract PSBCH data and channel estimates.
  - Computes the channel power `nr_pbch_channel_level` for symbol 0 to determine the scaling factor `log2_maxh`.
  - Performs channel compensation `nr_pbch_channel_compensation` to equalize the received signal using channel estimates.
```
  nr_pbch_channel_compensation(rxdataF_ext,
                                 dl_ch_estimates_ext,
                                 nb_re,
                                 rxdataF_comp,
                                 frame_parms,
                                 log2_maxh); // log2_maxh+I0_shift

    nr_pbch_quantize(psbch_e_rx + psbch_e_rx_idx, (short *)rxdataF_comp[0], SL_NR_NUM_PSBCH_DATA_BITS_IN_ONE_SYMBOL);
```
- Descrambling:
  - Applies descrambling `nr_pbch_unscrambling` to the soft bits using the provided slss_id.
- Polar Decoding:
  - Uses a polar decoder `polar_decoder_int16` to decode the descrambled soft bits into the SL-MIB payload.
```    
nr_pbch_unscrambling(psbch_e_rx, slss_id, 0, 0, psbch_e_rx_idx, 0, 0, 0, NULL);
  // polar decoding de-rate matching
  uint64_t tmp = 0;
  decoderState = polar_decoder_int16(psbch_e_rx,
                                     (uint64_t *)&tmp,
                                     0,
                                     SL_NR_POLAR_PSBCH_MESSAGE_TYPE,
                                     SL_NR_POLAR_PSBCH_PAYLOAD_BITS,
                                     SL_NR_POLAR_PSBCH_AGGREGATION_LEVEL);
```
- SL-MIB Extraction:
  - Extracts the Direct Frame Number (DFN) and slot offset from the decoded payload using bit masking.
  - Logs the SL-MIB contents and DFN/slot information.
- Return Value:
  - Returns 0 on successful decoding, or the decoder state (decoderState) if decoding fails.
#### nr_psbch_extract
- Input Validation: Ensures the symbol index is valid (PSBCH data is not carried in symbols 1–4, which are reserved for SL-PSS and SL-SSS).
- Antenna Loop: Processes each receive antenna (aarx).
- Carrier Offset Calculation: Determines the starting subcarrier offset (rx_offset) based on the SSB start subcarrier and frame parameters.
- Resource Block (RB) Processing:
  - Iterates over the number of resource blocks (nb_rb) allocated for PSBCH (defined as SL_NR_NUM_PSBCH_RBS_IN_ONE_SYMBOL).
  - For each RB, it extracts every subcarrier except those used for Demodulation Reference Signals (DMRS), which occur every 4th subcarrier (i % 4 != 0).
 ```
    for (rb = 0; rb < nb_rb; rb++) {
      j = 0;

      for (i = 0; i < NR_NB_SC_PER_RB; i++) {
        if (i % 4 != 0) {
          rxF_ext[j] = rxF[rx_offset];
          dl_ch0_ext[j] = dl_ch0[i];

#ifdef DEBUG_PSBCH

          LOG_I(PHY,
                "rxF ext[%d] = (%d,%d) rxF [%u]= (%d,%d)\n",
                (9 * rb) + j,
                rxF_ext[j].r,
                rxF_ext[j].i,
                rx_offset,
                rxF[rx_offset].r,
                rxF[rx_offset].i);
          LOG_I(PHY,
                "dl ch0 ext[%d] = (%d,%d)  dl_ch0 [%d]= (%d,%d)\n",
                (9 * rb) + j,
                dl_ch0_ext[j].r,
                dl_ch0_ext[j].i,
                i,
                dl_ch0[i].r,
                dl_ch0[i].i);
#endif
          j++;
        }
```
  - Copies the received signal (rxF) and channel estimates (dl_ch0) to the respective output buffers (rxF_ext, dl_ch0_ext).
