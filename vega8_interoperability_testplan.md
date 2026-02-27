# Vega 8 Interoperability Test Plan

## 1. Overview
This test plan defines the interoperability verification for the Vega 8 platform. It ensures that the switch ASIC, SONiC software, and various physical media (DAC, AEC, AOC, and Optical Transceivers) work together without signal degradation or protocol instability.

## 2. Test Hardware Matrix (Inventory)
The following media combinations are required for full coverage. Each is tested against the procedures in Section 3.

| ID | Category | Speed | Media/Reach | SerDes | Expected FEC |
|:---|:---|:---|:---|:---|:---|
| **M-01** | Optical | 1.6T | SMF / 500m | 8x200G | RS-FEC |
| **M-02** | Optical | 800G | MMF / 100m | 8x100G | RS-FEC |
| **M-03** | DAC | 400G | Copper / 1m | 8x50G | RS-FEC |
| **M-04** | DAC | 200G | Copper / 3m | 4x50G | RS-FEC |
| **M-05** | AEC | 800G | Active / 5m | 8x100G | RS-FEC |
| **M-06** | AOC | 400G | Fiber / 10m | 4x100G | RS-FEC |

## 3. Test Cases

### 3.1 EEPROM Identification & Verification
**Objective:** Confirm the switch correctly identifies the cable/module properties.
- **Action:** Insert media and run `sfputil read-eeprom -p <port>`.
- **Verification:** Checksum must be **Valid**. Vendor and Part Number must match the hardware.

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
