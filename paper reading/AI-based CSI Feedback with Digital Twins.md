# AI-based CSI Feedback with Digital Twins:Real-World Validation and Insights

**Authors**: Tzu-Hao Huang, Chao-Kai Wen, Shang-Ho Tsai, Trung Q. Duong  
**Source**: arXiv:2505.00660v2 [cs.IT], 2 May 2025  
**Focus**: Digital twins (DTs) and deep learning (DL) for channel state information (CSI) feedback in MIMO systems, with real-world (RW) experiments to assess performance and the role of online learning (OL).


## 1. Introduction & Context
- **Objective**: Enhance spectrum efficiency in 5G/6G MIMO systems through accurate CSI feedback, critical for base station (BS) performance.
- **Challenge**: Traditional codebook-based CSI feedback (e.g., 5G NR Type II) struggles with large-scale MIMO due to high overhead and inefficiency.
- **Solution**: DL-based CSI feedback, using autoencoders to compress and reconstruct precoding matrices, offers higher accuracy and compatibility with 5G NR.
- **Role of Digital Twins (DTs)**: DTs generate site-specific datasets for training DL models, reducing the need for costly RW data collection.
- **Gap Addressed**: Most studies rely on simulations; this paper validates DT effectiveness in RW scenarios using ray-tracing and OL.

## 2. System Model and DT Framework
- **MIMO System**: Single-user MIMO with \( N_t \) transmit antennas at BS, \( N_r \) receive antennas at UE, using OFDM with \( N_c \) subcarriers. The UE computes \( N_s \) eigenvectors (precoders) for feedback.
- **DT Framework**:
  - **Real World(RW) Channel Model**: **H = g(E, A)** , where \( E \) is environmental conditions, \( A \) are antenna effects, and \( g( - ) \) is propagation phenomena (傳播現象（如反射、繞射)） define the channel.
  - **Virtual Channel**:Modeling E, A, and g(·) is challenging. Thus,we approximate ~H   with 3D models (~E), antenna properties (~A), and propagation simulation (~g(-)).
  - **Virtual Channel Equation**: 
   <img width="442" height="109" alt="image" src="https://github.com/user-attachments/assets/fd8f516d-5b2f-45cf-a20e-b5108bee779f" />

    where
    - L_h is the number of paths,
    - ( theta_{AoD_l}, theta_{AoA_l} ) are angles of departure/arrival of the l-th path,both including azimuth(方位角) and elevation(仰角) components.
    - The function G(·) ∈ C (Nr×Nt) is the complex gain matrix determined by the AoD and AoA.
    - tau_l  is path delay.
      
## 3. DL Model
- due to modeling limitations in reflections, diffractions, environmental noise, DTs cannot fully replicate RW channels. To fix it,e a hybrid approach that integrates DL with DT data.
  
  <img width="875" height="323" alt="image" src="https://github.com/user-attachments/assets/3b2a5ab3-7067-491f-9b9f-f470bd893ff0" />

- **DL Model**: EVCsiNet(Compress and feedback  CSI based on DL to reduce caiculation complexity delay) compresses precoding matrix ( W ) into a quantized codeword at the UE and reconstructs it at the BS.
- **Input Data**: The precoding matrix W is complex-valued. Since neural networks work with real numbers, the real and imaginary parts need to be separated.
- **Encoding and Decoding Functions**:
  - Encoder: 𝑓𝑒(⋅;Θ𝐸),with parameters ΘE.
     - Compresses the virtual precoding matrix W into a V-dimensional vector.The vector is quantized into a B-bit codeword and sent to the BS.
  - Decoder: 𝑓𝑑(⋅;Θ𝐷), with parameters ΘD.
     - Reconstructs W from the received codeword.
  - Entire autoencoder: 
    - **𝑓𝑎(𝑊;Θ)=𝑓𝑑(𝑄(𝑓𝑒(𝑊;Θ𝐸));Θ𝐷)**
    - Q(⋅) is the quantizer that discretizes the encoded vector into a B-bit codeword for transmission.
    - Θ=(Θ𝐸,Θ𝐷) are the overall model parameters.
- The model is trained offline using virtual channel data:
  
<img width="395" height="53" alt="image" src="https://github.com/user-attachments/assets/8974274c-8de2-4377-9e4c-5b09c476d1c7" />

   - L(⋅) is the mean squared error (MSE) loss.
   - 𝐸𝐻𝑒[⋅] denotes averaging over virtual channel samples.
- Online Learning:
   - Real-World Fine-Tuning(微調)
   - There is a mismatch between simulated channels and real wireless channels, so the model needs fine-tuning.
   - The UE has limited battery and computational power. Therefore, only the decoder is fine-tuned at the BS, while the encoder parameters remain fixed.
   - Fine-Tuning Objective:
   <img width="281" height="46" alt="image" src="https://github.com/user-attachments/assets/97e0630a-a19e-4a94-8d6b-f921d15abd0d" />

## 4. Experimental Setup
- **RW Scenario**: Indoor corridor at National Sun Yat-sen University (NSYSU), 19.7m x 5.93m, with BS (8-patch antenna) and UE (8-patch antenna). Operates at 3.8 GHz, 100 MHz bandwidth, 1,620 subcarriers, 1024-QAM, and 2 spatial streams.
- **DT Scenario**: Replicated using Wireless InSite® ray-tracing, with accurate 3D models, antenna properties, and UE positions/orientations. An outdoor DT dataset was also created (NSYSU campus).
- **Fidelity Check**: DT and RW channels compared via angle of arrival (AoA) similarity (inner product \( \eta \)) and precoder cosine similarity. Results show high fidelity (e.g., \( \eta \geq 0.85 \) for primary/secondary paths).
- **OL Setup**: 30% of RW measurement points used, with 1,620 subcarriers grouped into 60 subbands to generate 60 precoding matrices per point.


## 5. Key Findings
- **DT-Trained Model Performance**:
  - **Indoor EVCsiNet**: Achieves \( \rho = 0.93 \) (cosine similarity) in DT, \( \rho = 0.67 \) in RW without OL, and \( \rho = 0.82 \) with OL, using 32-bit feedback.
  - **Comparison**: Outperforms Outdoor EVCsiNet (\( \rho = 0.64/0.75 \)) and CDL EVCsiNet (\( \rho = 0.44/0.74 \)), and matches Type II codebook (3 beams, \( \rho = 0.86 \), 58 bits) with lower overhead.
  - **Throughput**: OL increases Indoor EVCsiNet throughput by 17 Mbps, comparable to Type II (3 beams), and outperforms Type II (2 beams) due to balanced precoder reconstruction.
- **Robustness**:
  - **Environmental Mismatch**: Indoor EVCsiNet outperforms Outdoor/CDL models due to better alignment with RW conditions.
  - **Antenna Effects**: BS antenna changes degrade performance significantly (\( \rho = 0.57 \)) compared to UE changes (\( \rho = 0.94 \)). OL with 5% new BS antenna data restores \( \rho = 0.84 \).
  - **Interference/Mobility**: Performance remains stable, suggesting these factors need not be explicitly modeled in DT.
- **OL Importance**: Essential to bridge DT-RW gaps, with minimal RW data (5–30%) sufficient for significant performance gains.


## 6. Conclusions
- **DT Effectiveness**: DTs enable effective pretraining for DL-based CSI feedback, but discrepancies with RW environments require OL for robust performance.
- **OL Benefits**: Fine-tuning with minimal RW data mitigates mismatches in environment and antenna effects, achieving performance close to high-overhead codebook methods.
- **Key Insight**: A dedicated DT tailored to the RW scenario is critical, even with OL, to ensure high performance in practical deployments.

#### **References Highlights**
- **MIMO and CSI**: Importance of accurate CSI for massive MIMO (Lu et al., 2014; Rao et al., 2014).
- **DL for CSI Feedback**: Pioneering work on DL-based feedback (Wen et al., 2018; Liu et al., 2021).
- **DT Applications**: Recent advancements in DTs for MIMO (Nugroho et al., 2024; Jiang et al., 2024; Lin et al., 2025).
