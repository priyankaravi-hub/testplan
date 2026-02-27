# Vega 8 Interoperability Test Plan

## 1. Overview
This test plan defines the interoperability verification for the Vega 8 platform. It ensures that the switch ASIC, SONiC software, and various physical media (DAC, AEC, AOC, and Optical Transceivers) work together without signal degradation or protocol instability.

## 2. Test Hardware Matrix (Inventory)
### 2.1 Copper & Active Cables(DAC,AEC, AOC)The following media combinations are required for full coverage. Each is tested against the procedures in Section 3.

| Speed |  SerDes | Lengths to Test |
|:---|:---|:---|
| 200G | 2/100, 4/50, 8/25 | 1m, 3m, 5m, 7m, 10m |
| 400G | 8/50, 4/100 | 1m, 3m, 5m, 7m, 10m |
| 800G | 2/400, 4/200, 8/100| 1m, 3m, 5m, 7m, 10m  |


### 2.2 Optical Transceivers
Optical testing focuses on fiber types (SMF/MMF) and connector stability.

| Speed |  Fiber / Reach | Connector |
|:---|:---|:---|
| 1.6T | SMF / 500m | Dual MPO-12, MPO-16 |
| 1.6T | SMF / 2km | Dual Duplex LC |
| 800G | MMF / 100m | MPO-16 |
| 800G | SMF / 500m | MPO-12, MPO-16 |
| 800G | SMF / 2km | Duplex LC |
| 800G | SMF / 10km | Duplex LC |
| 400G | MMF / 100m | MPO-12 |
| 400G | SMF / 500m | MPO-12 |
| 400G | SMF / 2km | Duplex LC |
| 400G | SMF / 10km | Duplex LC |

## 3. Test Cases

### 3.1 Physical Layer && EEPROM Validation
**Objective:** Verify that the switch correctly recognizes the hardware and reads its internal identity without errors
- **Step 1:**
- - **Action:** Insert the transceiver/cable into the target port.
- - **Verification:** Checksum must be **Valid**. Vendor and Part Number must match the hardware.

### 3.2 Link Training & Stability (Soak Test)
**Objective:** Ensure 0% packet loss over a 24-hour period under thermal load.
- **Action:** Inject 100% line-rate traffic. Establish OSPF adjacency.
- **Verification:** - `show interfaces status`: Oper Status is UP.
  - `show ip ospf neighbor`: Neighbor is FULL for 24 hours.
  - `show interface counters`: CRC and Symbol Errors must be **0**.

### 3.3 Physical Stress (Hot-Plug)
**Objective:** Verify the software (pmon/sfputil) can handle 100 rapid insertion cycles.
- **Action:** Physically pull and re-insert the cable 100 times.
- **Verification:** Link must reach "Up" state within 3 seconds of every insertion.

### 3.4 Lane Alignment & Skew
**Objective:** Confirm all SerDes lanes are synchronized.
- **Action:** Run `show interfaces transceiver eeprom <port> --dom`.
- **Verification:** **PCS Lock** must be OK. **Lane Skew** must be < 500ps.
