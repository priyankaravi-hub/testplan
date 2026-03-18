# Static Route

## 1. Overview
This test plan defines the test cases for the static routing feature on Scale-Out platform

## 2. Platform
Vega 8 switches, Spectrum 4 ASIC 

## 3. Test Cases

### 3.1 IPv4 Configuration Test Cases
**Objective:** 
- **Scenario 1:**
  - Configure `ip route <prefix> <next-hop>`
  - **Expected result:** Route is installed in the routing table; next-hop is reachable.
- **Scenario 2:**
  - Configure `ip route <prefix> <interface name>`
  - **Expected result:** Route is resolved via the specified interface.
- **Scenario 3**
  - Configure route with both interface and next-hop
  - **Expected Result:** System validates that next hop is reachable via that specific interface
- **Scenario 4**
  - Configure IPv4 route within a specific VRF.
  - **Expected Result:** Route is isolated to tenant VRF and does not appear un the default VRF.
- **Scenario 5**
  - Configure route with a custom administrative distance.
  - **Expected Result:** Route is configured with the specified distance; lower distance is preferred.

### 3.2 IPv6 Configuration Test Cases
**Objective:** 
- **Scenario 6:**
  - Configure `ipv6 route <prefix> <interface-name>`
  - **Expected Result:** Route is installed in IPv6 routing table
- **Scenario 7:**
  - Configure `ipv6 route <prefix> <interface-name>`
  - **Expected Result:** Next-hop is resolved via the interface.
- **Scenario 8:**
  - Configure IPv6 route in tenant VRF
  - **Expected Result:** VRF isolation is maintained; VRF must exist for successful config.
- **Scenario 9:**
  - Set custom administrative distance for IPv6.
  - **Expected Result:** Distance is applied correctly. 

### 3.3 VRF
**Objective:** Confirm that multi-lane signals(400G/800G) are synchronized and physically healthy.
- **Step 1:**
  - **Action:** Create VRF - `vrf instance Tenant_A`
  - **Verification:** Check `show vrf`
- **Step 2:**
  - **Action:** Add route - `ip route vrf Tenant_A 10.1.1.0/24 172.16.1.1`
  - **Verification:** Check `show ip route vrf Tenant_A`
- **Step 3:**
  - **Action:** Verify isolation - Ping `10.1.1.1` from the Default VRF.
  - **Verification:** Ping fails.
- **Step 4:**
  - **Action:** Verify routing - Ping `10.1.1.1` from Tenant_A VRF.
  - **Verification:** Ping succeeds.


### 3.4 Operational and Guardrail test cases
**Objective:** Verify the 'Show' commands, system safety, and persistence
- **Visibility**
  - **Action:** Execute `show ip route static` and `show ipv6 route static`.
  - **Verification:** Ensure only static routes are displayed.
  - **Action:** Execute `show ip route <prefix>` to verify detailed infomration for a specific route.
  - **Verification:** Ensure it shows the lineage of the route - the next-hop, the interface and the Administrative Distance.
    
- **Guardrails**
  - **Action:** Attempt to configure a next-hop that is not reachable.
  - **Verification:** The command maybe accepted into the config but the route should NOT appear in the active routing table.
  - **Action:** Configure a route using an interface that is "Down" or non-existent.
  - **Verification:** The command should be accepted, but the route must remian inactive until you send 'no shut' command to the interface.
  - **Action:** Verify if no distance is specified, the default administrative distance is 1.
  - **Verification:** If you type `ip route 1.1.1.1/32 2.2.2.2` and check it with `show ip route 1.1.1.1`, the distance must show 1

- **Acceptance Criteria**
  - **Action:** Save the configuration and reboot the switch 
  - **Verification:** Verify all static routes are active immediately after boot.
  - **Action:** Verify traffic is forwarded correctly to the configured next-hop
  - **Verification:** Ping must succeed
  - **Action:** Ensure the static routes operate independently of dynamic protocols like BGP or OSPF
  - **Verification:** Static routes should not need BGP or OSPF to be turned on to function



  
