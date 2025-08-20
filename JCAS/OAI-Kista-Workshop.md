# note of https://openairinterface.org/wp-content/uploads/2025/05/OAI-Kista-Workshop-NEU-1.pdf

<img width="1387" height="574" alt="image" src="https://github.com/user-attachments/assets/d82a9163-0615-4431-8c42-8327e5cb02a1" />

- UE (User Equipment)：like cellphone.
- gNB (next gen node B)：5G base station.
- LMF (Localization Management Function)：Requests from the UE or the network, calculates the UE's geographic location. As the central node in the positioning process, the LMF is responsible for interacting with multiple network components and external systems.
- AMF (Access Mobility Function)：Responsible for handling UE connection, registration, authentication, and mobility management. It serves as the gateway between the UE and the core network, ensuring seamless UE handover between different base stations or networks and supporting various service requirements, including positioning-related collaboration.

protocal:
- NR-Uu (UE ↔ gNB): the radio interface. Carries the actual positioning reference signals and reports, e.g., DL-PRS/CSI-RS/SSB (downlink) and SRS/PRACH (uplink).
- NG-C (N2) (gNB ↔ AMF): control-plane interface that transports RRC/NAS signaling.
- LPP (UE ↔ LMF): LTE/NR Positioning Protocol (extended NR in Rel-16 ver.).
   - Measurement configuration, assistance data, measurement reports, and results between UE and LMF.
- NRPPa (gNB ↔ LMF): NR Positioning Protocol-A (Rel-15 ver.).
   - LMF↔RAN coordination. The LMF requests/provides assistance (e.g., PRS scheduling) and measurement tasks; the gNB returns results (time/angle/power, etc.).
- Nls / Nlmf_Location (AMF ↔ LMF): a 5GC service-based interface used for the AMF to forward LPP and exchange location-related context/events with the LMF.

<img width="1387" height="261" alt="image" src="https://github.com/user-attachments/assets/5cf4a1a6-93f1-45e3-a9b5-bf07cfb53a51" />

- **Rel 15**: Enhanced cell ID, UL-TDoA, U-L-AoA
  - Enhanced cell ID: A method that estimates the UE's location using the geographic position of the serving base station and signal strength, suitable for coarse positioning.
  - UL-TDoA (Uplink Time Difference of Arrival): Determines the UE's position by measuring the time difference of arrival of uplink signals from the UE to multiple base stations, requiring coordinated measurements.
  - U-L-AoA (Uplink Angle of Arrival): Estimates the UE's location based on the angle of arrival of uplink signals received at multiple base stations.

- **Rel 16**: DL-TDoA, DL-AoD, Multi-RTT (Round-trip-time)
  - DL-TDoA (Downlink Time Difference of Arrival): Calculates the UE's position by measuring the time difference of arrival of downlink Positioning Reference Signals (PRS) from multiple base stations.
  - DL-AoD (Downlink Angle of Departure): Uses the angle of departure of downlink signals from base stations, combined with UE measurements, to determine location.
  - Multi-RTT (Multi-Round-Trip-Time): Improves accuracy by measuring the round-trip time between the UE and multiple base stations, leveraging multi-point measurements.

- **Rel 17**: RRC_INACTIVE positioning, on-demand PRS, improvements in accuracy and latency
  - RRC_INACTIVE positioning: Enables positioning even when the UE is in an RRC (Radio Resource Control) inactive state, reducing power consumption for low-power devices.
  - On-demand PRS: Triggers Positioning Reference Signals (PRS) on demand, enhancing flexibility and reducing continuous resource usage.
  - Improvements in accuracy and latency: Enhances positioning precision and reduces delay through optimized algorithms and signal processing.

- **Rel 18**: RedCap, Carrier aggregation, low power, sidelink
  - RedCap (Reduced Capability): A positioning technique designed for low-cost, low-power devices (e.g., IoT devices), minimizing hardware requirements and optimizing power usage.
  - Carrier aggregation: Combines signals from multiple carrier frequencies to increase bandwidth and improve positioning accuracy.
  - Low power: Optimizes the positioning process to minimize UE energy consumption, ideal for long-term applications.
  - Sidelink: Allows direct communication between UEs for positioning (e.g., vehicle-to-vehicle in V2X), enabling high-precision applications without relying on base stations.

<img width="1415" height="706" alt="image" src="https://github.com/user-attachments/assets/a02e81bd-a34f-413e-855e-ecfcd8270a8a" />

1. **UE Transmitting UL-Sounding Reference Signal (SRS)**:
   - The User Equipment (UE) transmits UL-Sounding Reference Signals (SRS) to multiple gNBs over the NR-Uu interface. These signals are used by the gNBs to measure the time of arrival of the UE's uplink transmission.

2. **Synchronized gNBs Estimate Time of Arrival (ToA)**:
   - The gNBs, which are synchronized (likely via a common time reference), estimate the Time of Arrival (ToA) of the SRS signals from the UE. Each gNB calculates the time it takes for the SRS to reach it, producing individual ToA values.

3. **gNBs Send ToAs over NRPPa to LMF**:
   - The gNBs send their estimated ToA values to the Location Management Function (LMF) via the New Radio Positioning Protocol A (NRPPa) over the NG-C interface. This data is transmitted through the AMF (Access Mobility Function), which acts as an intermediary in the control plane.

4. **LMF Converts ToAs to TDoAs and Estimates UE Coordinates**:
   - The LMF receives the ToA values from multiple gNBs and converts them into Time Differences of Arrival (TDoA) by comparing the ToA values pairwise (e.g., ToA differences between gNBs). Using these TDoA values and the known positions of the gNBs, the LMF calculates the UE's coordinates (position) using triangulation or similar algorithms.

5. **LMF Sends UE Coordinates to an External Location Service API**:
   - The LMF sends the estimated UE position coordinates to an external Location Service (LCS) via the NLS (Network Location Service) interface, typically through an API (e.g., HTTP). This allows the position data to be used by external applications or services, such as emergency response systems.
     
*LCS refers to a set of functions and services within the network that provide location-related information for various applications.In this figure,it  acts as an external service or interface that utilizes the calculated position data,receiving the UE's position coordinates from the Location Management Function (LMF) via an API (e.g., HTTP).

6. **TDoA Calculation with Beacons (Optional Context)**:
   - The diagram includes beacons (Beacon 1, 2, and 3) and distance differences (Δd), which represent the geometric relationships used in TDoA. The LMF uses these distance differences (derived from TDoA) between the UE and the beacons/gNBs to pinpoint the UE's location. For example, Δd_1,2 is the difference in distance from the UE to Beacon 1 and Beacon 2.


