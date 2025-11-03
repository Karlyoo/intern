##  Sounding Reference Signal (SRS) in OAI and O-RAN E2SM-LLC

###  Objective

**check point** :Understand how **Sounding Reference Signal (SRS)** is defined and handled in both **3GPP NR specifications** and **O-RAN E2 Service Model (E2SM-LLC)**, and connect this knowledge to the **OAI (OpenAirInterface)** implementation.

---

### ✅ Checkpoints and Deadlines

| Task                                                                     | Description                                                                                                           | Reference                                | Deadline   |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- | ---------- |
| **1. Define SRS in 3GPP NR**                                             | Study the technical definition, generation, and mapping of SRS. Summarize how it is used for uplink channel sounding. | 3GPP TS 38.211 §6.4.1.4, 38.213 §9       | **Nov 7**  |
| **2. Explore SRS Implementation in OAI**                                 | Locate and summarize key functions in `nr_srs.c`. Understand how SRS is generated, mapped, and sent in OAI.           | OAI `openair1/PHY/NR_TRANSPORT/nr_srs.c` | **Nov 9**  |
| **3. Identify SRS-related Information Elements (IEs) in O-RAN E2SM-LLC** | From the specification, list the IEs that reference SRS and their semantic roles.                                     | O-RAN.WG3.TS.E2SM-LLC v1.0               | **Nov 11** |
| **4. Map SRS info flow between layers**                                  | Draw a diagram linking 3GPP → OAI PHY layer → O-RAN E2 interface → Near-RT RIC.                                       | Own synthesis                            | **Nov 13** |

---

### 🧠 Key Findings from O-RAN E2SM-LLC v1.0

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

## define SRS in 3GPP NR (TS 38.211 6.4.1.4)
https://github.com/Karlyoo/LDPCinOAI/blob/main/3GPP%20TR%2038.211.md#6414-srs-sounding-reference-signal
