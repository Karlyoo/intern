# OAI CODE

## openairinterface5g/openair1/PHY/NR_UE_TRANSPORT/csi_rx.c 
Estimating channel conditions and reporting metrics like :
- Reference Signal Received Power (RSRP)
- Rank Indicator (RI)
- Precoding Matrix Indicator (PMI)
- Channel Quality Indicator (CQI)
to the MAC layer.

KEY functions:

1. **nr_det_A_MF_2x2**
   - **Purpose**: Computes the determinant of a 2x2 matrix for a MIMO system, used in RI estimation.
   - **Input**: Four 32-bit integer pointers (`a_mf_00`, `a_mf_01`, `a_mf_10`, `a_mf_11`) representing matrix elements, a determinant output pointer (`det_fin`), and the number of resource blocks (`nb_rb`).
   - **Operation**:
     - Performs complex multiplications for the real part of the determinant using SIMD instructions (`simde__m128i`).
     - Computes `ad - bc` for the determinant in Q30 format.
     - Iterates over `3 * nb_rb` resource blocks, updating pointers for each iteration.
   - **Output**: Stores the absolute determinant value in `det_fin`.

2. **nr_squared_matrix_element**
   - **Purpose**: Calculates the squared magnitude of matrix elements.
   - **Input**: Input array (`a`), output array (`a_sq`), and number of resource blocks (`nb_rb`).
   - **Operation**:
     - Uses SIMD to compute the squared magnitude of complex numbers (`a_128[0] * a_128[0]`).
     - Iterates over `3 * nb_rb` resource blocks.
   - **Output**: Stores squared values in `a_sq`.

3. **nr_numer_2x2**
   - **Purpose**: Computes the numerator for RI estimation by summing squared matrix elements.
   - **Input**: Four squared matrix element arrays (`a_00_sq`, `a_01_sq`, `a_10_sq`, `a_11_sq`), output array (`num_fin`), and `nb_rb`.
   - **Operation**:
     - Adds squared elements (`a_00_sq + a_11_sq` and `a_01_sq + a_10_sq`) using SIMD.
     - Iterates over `3 * nb_rb` resource blocks.
   - **Output**: Stores the sum in `num_fin`.

4. **is_csi_rs_in_symbol**
   - **Purpose**: Determines if a CSI-RS is present in a given symbol based on 3GPP TS 38.211 Table 7.4.1.5.3-1.
   - **Input**: CSI-RS configuration (`csirs_config_pdu`) and symbol index (`symbol`).
   - **Operation**:
     - Checks the `row` field of the configuration to map symbol locations.
     - Returns `true` if the symbol contains CSI-RS, otherwise `false`.
     - Handles different row cases (1–18) with varying symbol counts (1, 2, or 3 symbols).
   - **Output**: Boolean indicating CSI-RS presence.

5. **nr_get_csi_rs_signal**
   - **Purpose**: Extracts CSI-RS signals and computes RSRP.
   - **Input**: UE context, process context, CSI-RS configuration, CSI info, mapping parameters, CDM group size, received signal (`rxdataF`), and output arrays for CSI-RS signal and RSRP.
   - **Operation**:
     - Loops over receive antennas, resource blocks, CDM groups, and resource elements.
     - Accounts for frequency density (e.g., 0.5 for even/odd RBs).
     - Copies received signal to output and computes RSRP as the average power.
     - Converts RSRP to dBm with adjustments for gain and normalization.
   - **Output**: Populates `csi_rs_received_signal`, `rsrp`, and `rsrp_dBm`.

6. **calc_power_csirs**
   - **Purpose**: Calculates the power of a CSI-RS signal.
   - **Input**: Array of signal values (`x`) and CSI-RS configuration.
   - **Operation**:
     - Computes the variance of the signal (`sum_x2 / size - (sum_x / size)^2`).
     - Accounts for frequency density to skip certain RBs.
   - **Output**: Returns the computed power.

7. **nr_csi_rs_channel_estimation**
   - **Purpose**: Performs Least Squares (LS) channel estimation and interpolation for CSI-RS.
   - **Input**: Frame parameters, process context, CSI-RS configuration, CSI info, generated and received signals, mapping parameters, CDM group size, memory offset, and output arrays for channel estimates and noise power.
   - **Operation**:
     - **LS Estimation**: Computes channel estimates by multiplying conjugated transmitted and received signals.
     - **Interpolation**: Applies filters (`filt24_start`, `filt24_middle`, `filt24_end`) based on subcarrier position.
     - **Noise Power**: Calculates noise as the difference between LS and interpolated estimates.
     - Uses SIMD for efficient processing.
   - **Output**: Populates channel estimate arrays, `log2_re`, `log2_maxh`, and `noise_power`.

8. **nr_csi_rs_ri_estimation**
   - **Purpose**: Estimates the Rank Indicator (RI) for a 2x2 MIMO system.
   - **Input**: UE context, CSI-RS configuration, CSI info, number of ports, memory offset, channel estimates, `log2_maxh`, and output rank indicator.
   - **Operation**:
     - Computes the Hermitian matrix product (`Hh x H`) and its determinant and numerator.
     - Calculates the condition number in dB and compares it to a threshold (`cond_dB_threshold = 5`).
     - Increments/decrements a counter based on the condition number to determine RI (0 or 1).
   - **Output**: Sets `rank_indicator` (0 for rank 1, 1 for rank 2).

9. **nr_csi_rs_pmi_estimation**
   - **Purpose**: Estimates the Precoding Matrix Indicator (PMI) and precoded SINR.
   - **Input**: UE context, CSI-RS configuration, CSI info, number of ports, memory offset, channel estimates, interference plus noise power, rank indicator, `log2_re`, and output arrays for PMI (`i1`, `i2`) and SINR.
   - **Operation**:
     - For SISO (N_ports = 1): Computes average signal power and SINR.
     - For MIMO (2x2): Tests four precoding vectors and selects the one with the highest SINR for rank 1, or compares pairs for rank 2.
     - Converts SINR to dB.
   - **Output**: Sets `i1`, `i2`, and `precoded_sinr_dB`.

10. **nr_csi_rs_cqi_estimation**
    - **Purpose**: Maps precoded SINR to CQI based on 3GPP TS 38.214 Table 5.2.2.1-2.
    - **Input**: Precoded SINR and output CQI pointer.
    - **Operation**: Assigns CQI values (4–15) based on SINR ranges for a 0.1 BLER AWGN channel.
    - **Output**: Sets `cqi`.

11. **nr_csi_im_power_estimation**
    - **Purpose**: Estimates interference plus noise power using CSI-IM.
    - **Input**: UE context, process context, CSI-IM configuration, received signal, and output power pointer.
    - **Operation**:
       - Loops over symbols, antennas, and subcarriers specified in the CSI-IM configuration.
       - Computes the variance of real and imaginary parts to estimate power.
    - **Output**: Sets `interference_plus_noise_power`.

12. **nr_ue_csi_im_procedures**
    - **Purpose**: Main function for CSI-IM processing.
    - **Input**: UE context, process context, received signal, and CSI-IM configuration.
    - **Operation**:
       - Logs configuration details (if debug enabled).
       - Calls `nr_csi_im_power_estimation` to compute interference plus noise power.
       - Marks CSI-IM measurement as computed.
    - **Output**: Updates `ue->nr_csi_info->interference_plus_noise_power`.

13. **nr_ue_csi_rs_procedures**
    - **Purpose**: Main function for CSI-RS processing.
    - **Input**: UE context, process context, received signal, and CSI-RS configuration.
    - **Operation**:
       - Logs configuration details (if debug enabled).
       - Skips processing for TRS (csi_type = 0) or ZP CSI-RS (csi_type = 2).
       - Generates CSI-RS signals using `nr_generate_csi_rs`.
       - Performs signal extraction (`nr_get_csi_rs_signal`), channel estimation (`nr_csi_rs_channel_estimation`), RI estimation (`nr_csi_rs_ri_estimation`), PMI estimation (`nr_csi_rs_pmi_estimation`), and CQI estimation (`nr_csi_rs_cqi_estimation`) based on the measurement bitmap.
       - Sends measurements (RSRP, RI, PMI, CQI) to the MAC layer via `nr_fill_dl_indication` and `nr_fill_rx_indication`.
    - **Output**: Logs results and sends measurements to MAC.

#### Key Structures and Types
- **fapi_nr_dl_config_csirs_pdu_rel15_t**: Contains CSI-RS configuration (e.g., start RB, number of RBs, frequency density, symbol locations, CDM type).
- **fapi_nr_dl_config_csiim_pdu_rel15_t**: Contains CSI-IM configuration (e.g., start RB, number of RBs, subcarrier and symbol indices).
- **csi_mapping_parms_t**: Defines CSI-RS mapping parameters (e.g., ports, kprime, lprime).
- **c16_t**: 16-bit complex number structure with real (`r`) and imaginary (`i`) components.
- **simde__m128i**: SIMD vector type for optimized processing.
- **NR_DL_FRAME_PARMS**: Frame parameters (e.g., number of antennas, OFDM symbol size, samples per slot).

#### Key Features
- **SIMD Optimization**: Uses `simde__m128i` for vectorized computations in matrix operations and signal processing.
- **Frequency Density Handling**: Supports densities values (0.5, 1, 3) to process even/odd RBs or all RBs.
- **Measurement Bitmap**: Controls which measurements (RSRP, RI, PMI, CQI) are computed (e.g., bitmap 27 for all).
- **Debug Logging**: Extensive logging under `NR_CSIRS_DEBUG` and `NR_CSIIM_DEBUG` for signal values, channel estimates, and results.
- **3GPP Compliance**: Follows TS 38.211 (CSI-RS mapping) and TS 38.214 (CQI table).

#### Workflow
1. **CSI-IM Processing**:
   - Estimate interference plus noise power using `nr_csi_im_power_estimation`.
   - Store results in `ue->nr_csi_info`.

2. **CSI-RS Processing**:
   - Generate CSI-RS signals.
   - Extract received signals and compute RSRP.
   - Perform LS channel estimation and interpolation.
   - Estimate RI, PMI, and CQI based on the measurement bitmap.
   - Send results to the MAC layer.

#### Notes
- The code assumes a 2x2 MIMO system for RI and PMI estimation; other configurations are not fully supported.
- Memory alignment (e.g., `__attribute__((aligned(32)))`) is used for SIMD efficiency.
- Error handling includes assertions for invalid configurations (e.g., invalid CSI-RS row, zero RBs).
- The code skips processing for Tracking Reference Signals (TRS) and Zero-Power (ZP) CSI-RS.
