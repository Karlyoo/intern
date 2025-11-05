##  Sounding Reference Signal (SRS) in OAI and O-RAN E2SM-LLC

###  Objective

**check point** :Understand how **Sounding Reference Signal (SRS)** is defined and handled in both **3GPP NR specifications** and **O-RAN E2 Service Model (E2SM-LLC)**, and connect this knowledge to the **OAI (OpenAirInterface)** implementation.

---

###  Checkpoints and Deadlines

| Task                                                                     | Description                                                                                                           | Reference                                | Deadline   |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- | ---------- |
| **1. Define SRS in 3GPP NR**                                             | Study the technical definition, generation, and mapping of SRS. Summarize how it is used for uplink channel sounding. | 3GPP TS 38.211 §6.4.1.4, 38.213 §9       | **Nov 7**  |
| **2. Explore SRS Implementation in OAI**                                 | Locate and summarize key functions in `nr_srs.c`. Understand how SRS is generated, mapped, and sent in OAI.           | OAI `openair1/PHY/NR_TRANSPORT/nr_srs.c` | **Nov 9**  |
| **3. Identify SRS-related Information Elements (IEs) in O-RAN E2SM-LLC** | From the specification, list the IEs that reference SRS and their semantic roles.                                     | O-RAN.WG3.TS.E2SM-LLC v1.0               | **Nov 11** |
| **4. Map SRS info flow between layers**                                  | Draw a diagram linking 3GPP → OAI PHY layer → O-RAN E2 interface → Near-RT RIC.                                       | Own synthesis                            | **Nov 13** |

---


## define SRS in 3GPP NR (TS 38.211 6.4.1.4)
https://github.com/Karlyoo/LDPCinOAI/blob/main/3GPP%20TR%2038.211.md#6414-srs-sounding-reference-signal

DIRECTION : UE-->gNB

By analyzing the received SRS, the gNodeB can determine the quality of the signal path from the UE to the gNodeB and the amount of interference.

- 1. Frequency Domain: Defines where in the frequency band the SRS is sent.
  - Comb Structure: The signal is transmitted on a "comb" of subcarriers (e.g., comb-2 or comb-4) to save UE power and allow multiple users to share resources (FDM).
  - Frequency Hopping: The signal's frequency position can "hop" over time, allowing the gNodeB to measure the entire channel bandwidth.
- 2. Time Domain: Defines when the SRS is sent.
  - Periodicity & Offset: How often the signal is repeated (e.g., "every 20 slots") and when it starts.
  - Symbol Location: Which specific OFDM symbol(s) within a slot are used to carry the SRS (usually the last 1, 2, or 4 symbols).
- 3. Code Domain: Defines what the signal is and how users are separated.
  - Base Sequence: Uses a Zadoff-Chu (ZC) sequence. This special sequence has excellent mathematical properties for precise timing measurements and power efficiency.
  - Cyclic Shift (CS): This is the key to multi-user separation (CDM). Multiple UEs can use the exact same time/frequency resource, but each applies a unique "cyclic shift" to the ZC sequence, allowing the gNodeB to tell them apart.
- 4. Spatial Domain: Defines which antenna sends the signal.
  - Antenna Ports: The SRS can be configured to transmit from 1, 2, or 4 different antenna ports on the UE.
  - Purpose: This is essential for MIMO and beamforming. It allows the gNodeB to measure the unique spatial channel from each UE antenna, enabling it to separate multiple data streams.

###  Key Findings from O-RAN E2SM-LLC v1.0

- Its official short name is "ORAN-E2SM-LLC". The specification's scope is to define the E2 interface for interactions with Layer 1 (L1) and Layer 2 (L2) of the RAN.
- The E2SM-LLC model enables an xApp to interact with the gNodeB (E2 Node) primarily through two services: REPORT and CONTROL.
  
**REPORT Service (Monitoring)**
- The xApp can subscribe to receive reports from the gNodeB.
- Style 1: Lower Layers Information (LLI) Copy 
  - This style is used to get a direct copy of L1 signals received from the UE.
  - The xApp can request the Lower Layers Information Type to be:
    - SRS: To get the raw Sounding Reference Signal data.
    - CSI: To get the Channel State Information.
- Style 2: Lower Layers Measurements 
  - This style is used to get periodic L2 traffic statistics.
  - The xApp can request the Lower Layers Measurement Type to include:
    - DL_RLC_Buffer_Status: Reports the buffer occupancy (how much data is queued) and the "Head of Line" time (how long data has been waiting).
    - DL_PDCP_Buffer_Status: Provides similar buffer metrics for the PDCP layer.
    - DL_HARQ_Statistics: Reports the counts of ACK, NACK, and DTX.

**CONTROL Service (Controlling)**
- The xApp can actively send commands to the gNodeB to control L2 functions.
- Style 1: Logical Channels Handling Control 
  - Used to take control of scheduling for specific logical channels.
  - The xApp can send a list of logical channels to add to its control or release back to the gNodeB's (O-DU's) control.
- Style 2: Scheduling Parameters Control 
  - This is the most granular level of control, allowing the xApp to dictate the exact scheduling grant per slot.
  - The xApp sends a DL Scheduling Control message  that specifies precise L1/L2 parameters, including:
    - UE ID 
    - Logical Channel ID and number of bytes to send 
    - Freq Domain Resources 
    - Time Domain Resources 
    - MCS (Modulation and Coding Scheme)
      
This style also allows the xApp to send an SRS Request to trigger an aperiodic SRS from the UE.
#### **1. Lower Layers Information Type (Section 8.3.15)**

* Defines which lower-layer information is triggered or reported.
* **ENUMERATED (SRS, CSI, …)**

  * `SRS` refers to received Sounding Reference Signals (38.211 §6.4.1.4).
  * `CSI` refers to Channel State Information (38.213 §9).

#### **2. SRS IE Definition (Section 8.3.19)**

Defines the **structure of the SRS information** carried via E2SM:

* **List of SRS Receive Antennas** – identifies antennas used for SRS reception (O-RAN WG4.CUS §12.5.4).
* **List of SRS Symbols** – raw received SRS symbols, starting from symbol position *l₀* per 38.211 §6.4.1.4.1.
* **SRS Compression Header (optional)** – defines IQ compression method if static SRS configuration is used.
* **Raw SRS** – OCTET STRING containing IQ samples of received SRS (O-RAN WG4.CUS Table 8.3.2-1).

#### **3. ASN.1 Representation (Section 8.4.2)**

```asn1
LowerLayers-Info-Type ::= ENUMERATED { srs, csi, ... }
```

Used to specify the PHY-layer signal type in E2 messages.

#### **4. Reporting Mechanism (Section 8.2.1.4.1)**

* **E2SM-LLC Indication Message Format 1** includes:

  * `Slot Time Stamp`
  * `CHOICE Lower Layers Information Type` → `SRS (8.3.19)` or `CSI (8.3.20)`
* This allows the E2 Node to report **received SRS data** to the Near-RT RIC.

---

