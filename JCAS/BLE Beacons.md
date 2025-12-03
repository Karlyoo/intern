## BLE Beacons for Indoor Positioning at an Interactive IoT-Based Smart Museum
[BLE Beacons for Indoor Positioning at an
Interactive IoT-Based Smart Museum](https://arxiv.org/pdf/2001.07686)

### Introduction
This paper presents an indoor localization system designed to enhance the museum visitor experience using the Internet of Things (IoT). The system utilizes Bluetooth Low Energy (BLE) beacons to provide location-based cultural content and gather visitor analytics (such as retention time). Experimental results compare performance in a laboratory versus a corridor, demonstrating that while RSSI (Received Signal Strength Indicator) is prone to noise, the use of Kalman filtering significantly improves distance estimation and localization accuracy.

### System Architecture
The proposed system consists of three main service components operating on a visitor's smartphone and a museum server.

1. System Services
- Proximity Service: Beacons broadcast messages continuously. When a visitor approaches an exhibit, they receive a notification on their smartphone.
- Localization Service: The system estimates the visitor's location using signals from multiple beacons to guide navigation through the museum.
- Data Analytics Service: The system tracks visitor paths and retention times to provide recommendations and help museum management optimize exhibit flow.
2. Hardware & Software Components
- Beacons: The system uses Gimbal Series 21 beacons.
  - Battery Life: Approx. 18 months on 4 AA batteries.
  - Protocol: Uses the iBeacon protocol (advertising mode only).
  - Transmission: Configurable interval (100 ms to 10 s) and power (-23 dBm to 0 dBm).
- Mobile Device: An Android application was developed to run in the background, collect RSSI data, and perform localization calculations locally (or via cloud if Wi-Fi is available).

### Methodology and Algorithms
1. Ranging Technology (Path Loss Model)The distance between the beacon and receiver is calculated using the Log-Distance Path Loss model based on RSSI values.
   
   The formula used isRSSI = -10nlogd + A
   - n: Signal propagation constant (environment-dependent)
   - d: Distance.
   - A: Received signal strength at 1 meter.
     
In the presence of noise (shadowing), the distance calculation is adjusted using a Gaussian random variable.

2. Localization (Trilateration)
  -  The system uses Trilateration to estimate position based on distances from three known points (beacons).
  -  If the beacon coordinates and radii  derived from RSSI are known, the intersection point (x, y) of the three circles indicates the user's location.
  -  Mean Square Error (MSE) is used to calculate the deviation between the estimated and real location.

3. Filtering
Since raw RSSI data is highly susceptible to noise and interference, the application implements a Kalman filter.

The filter smooths the RSSI values to minimize the effect of environmental noise before distance calculation.

### Experimental Evaluation
Experiments were conducted in two environments: a Laboratory (open space, Line-of-Sight) and a Corridor (narrow, concrete walls, limited space)

Experiment A: Path Loss Analysis
- Environment Impact: The corridor proved more challenging due to concrete walls and people blocking signals.
- Path Loss Component (n): Calculated as 2.208 for the Laboratory and 2.341 for the Corridor, indicating higher attenuation in the corridor.
- Distance vs. Noise: RSSI variation increases significantly as the distance from the beacon increases.

Experiment B: Distance Estimation Accuracy
- Raw vs. Filtered: Using raw data, the estimation error was < 3m (Lab) and < 3.5m (Corridor) for 95% of the time.
- Kalman Improvement: Applying the Kalman filter reduced the error to within 2m (Lab) and 2.5m (Corridor)

Experiment C: Localization & Detection Accuracy
- Topology: Three beacons were arranged in a triangle. Best accuracy was achieved when beacons were closer together.
- Interference: Accuracy drops when the receiver is equidistant from all three beacons or when beacons are too close to one another, likely due to signal interference.
- Proximity Detection: Detection is highly accurate when within 50 cm of a beacon but drops as the user moves away or if neighboring beacons are within 1 meter
