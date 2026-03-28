# Title: SONiC Static Routing (IPv4/IPv6) Test Plan

## Table of Contents
- [1. Overview](#1-overview)
- [2. Scope](#2-scope)
- [3. Test Setup Environment](#3-test-setup-environment)
    - [3.1 Components](#31-components)
    - [3.2 Required Services on DUT](#32-required-services-on-dut)
    - [3.3 Observability / Evidence](#33-observability--evidence)
- [4. Test Cases](#4-test-cases)
    - [4.1 IPv4-BASE Baseline Configuration](#41-ipv4-base-baseline-configuration)
    - [4.2 IPv6-BASE Baseline Configuration](#42-ipv6-base-baseline-configuration)
    - [4.3 RT-VRF Virtual Routing and Forwarding](#43-rt-vrf-virtual-routing-and-forwarding)
    - [4.4 RT-ADV Advanced Routing Logic (AD/ECMP)](#44-rt-adv-advanced-routing-logic-adecmp)
    - [4.5 RT-GRD Guardrails and Negative Tests](#45-rt-grd-guardrails-and-negative-tests)
    - [4.6 RT-RES Resiliency and Persistence](#46-rt-res-resiliency-and-persistence)
    - [4.7 RT-SCALE Scalability and Stress](#47-rt-scale-scalability-and-stress)
- [5. Cleanup](#5-cleanup)
- [6. Reference](#6-reference)

---

## 1. Overview
Static routing is the manual definition of paths in the routing table. Unlike dynamic protocols, static routes are predictable and low-overhead. In SONiC, these routes are managed via the uCLI/CLI, stored in the Redis `CONFIG_DB`, and programmed into the ASIC (Spectrum 4) via the SAI (Switch Abstraction Interface). This plan validates IPv4 and IPv6 static routing capabilities for the Vega 8/SkyForce platforms.

## 2. Scope
This test plan covers:
- IPv4 and IPv6 static route configuration (Next-hop IP, Interface, or both).
- Multi-tenancy support via VRF isolation.
- Route preference control using Administrative Distance (AD).
- Load balancing via Equal-Cost Multi-Path (ECMP).
- System guardrails (unreachable next-hops, down interfaces).
- Configuration persistence across reboots.

## 3. Test Setup Environment

### 3.1 Components
- **DUT:** Vega 8 / SkyForce 9000 Switch running Super SONiC.
- **Neighbor A:** Upstream Router/Host (192.168.1.1 / 2001:db8:aaaa::1).
- **Neighbor B:** Upstream Router/Host (192.168.2.1 / 2001:db8:bbbb::1).
- **Traffic Generator:** For verifying data plane forwarding and ECMP hashing.

### 3.2 Required Services on DUT
- **fpmsyncd:** Syncs routes from FRR to Redis DB.
- **neighsyncd:** Manages ARP/Neighbor discovery for next-hop validation.
- **orchagent:** Programs the ASIC based on DB entries.

### 3.3 Observability / Evidence
- **Control Plane:** `show ip route static`, `vtysh -c "show ip route"`.
- **Database:** `redis-cli -n 4 HGETALL "STATIC_ROUTE_TABLE|..."`.
- **Data Plane:** `ping`, `traceroute`, ASIC-level table dumps (e.g., `bcmcmd "l3 defip show"`).

---

## 4. Test Cases

### 4.1 IPv4-BASE Baseline Configuration

#### 4.1.1 IPv4-BASE-01 Next-hop as IP
- **Steps:** `ip route 10.10.10.0/24 192.168.1.1`
- **Verification:** `show ip route 10.10.10.0`. Check if next-hop is 192.168.1.1.
- **Result:** Route installed in RIB/FIB; Ping to 10.10.10.1 succeeds.

#### 4.1.2 IPv4-BASE-02 Next-hop as Interface
- **Steps:** `ip route 10.20.20.0/24 Ethernet1`
- **Verification:** `show ip route`. Ensure route points to the egress interface.

#### 4.1.3 IPv4-BASE-03 Combined Interface and IP
- **Steps:** `ip route 10.30.30.0/24 Ethernet1 192.168.1.5`
- **Verification:** System validates 192.168.1.5 is reachable only via Ethernet1.

### 4.2 IPv6-BASE Baseline Configuration

#### 4.2.1 IPv6-BASE-01 Basic IPv6 Static Route
- **Steps:** `ipv6 route 2001:db8:cccc::/64 2001:db8:aaaa::1`
- **Expected Result:** `show ipv6 route` displays the static entry.

### 4.3 RT-VRF Virtual Routing and Forwarding

#### 4.3.1 RT-VRF-01 Tenant Isolation
- **Steps:** - Create VRF: `vrf instance Tenant_A`
    - Add route: `ip route vrf Tenant_A 10.1.1.0/24 172.16.1.1`
- **Verification:** - `ping 10.1.1.1` (from Default VRF) -> **Expected: FAIL**
    - `ping vrf Tenant_A 10.1.1.1` -> **Expected: SUCCESS**

### 4.4 RT-ADV Advanced Routing Logic (AD/ECMP)

#### 4.4.1 RT-ADV-01 Floating Static Route (AD)
- **Steps:**
    - **Primary:** `ip route 10.200.1.0/24 192.168.1.1` (AD 1)
    - **Backup:** `ip route 10.200.1.0/24 192.168.1.2 200` (AD 200)
- **Verification:** Only .1.1 is in RIB. Shut Ethernet1; verify .1.2 is installed.

#### 4.4.2 RT-ADV-02 ECMP Load Balancing
- **Steps:**
    - **Path A:** `ip route 10.77.77.0/24 Ethernet1 192.168.1.1`
    - **Path B:** `ip route 10.77.77.0/24 Ethernet2 192.168.2.1`
- **Verification:** `show ip route 10.77.77.0`. Both next-hops marked with `*`. Verify ASIC programs both paths in the hardware table via `show ip route vrf default 10.77.77.0/24`.

### 4.5 RT-GRD Guardrails and Negative Tests

#### 4.5.1 RT-GRD-01 Next-hop Reachability
- **Steps:** Configure route to an unreachable IP.
- **Expected Result:** Command accepted; route status is "Inactive" in `show ip route`.

#### 4.5.2 RT-GRD-02 Interface Down Behavior
- **Steps:** `ip route 5.5.5.0/24 Ethernet10`. Shutdown Ethernet10.
- **Expected Result:** Route removed from active RIB. Perform `no shut`; route returns.

### 4.6 RT-RES Resiliency and Persistence

#### 4.6.1 RT-RES-01 Reload Persistence
- **Steps:** `write memory`, `sudo reboot`.
- **Expected Result:** All routes restored upon boot. Check `config_db.json`.

### 4.7 RT-SCALE Scalability and Stress

#### 4.7.1 RT-SCALE-01 Max Routes
- **Steps:** Inject 1,000 unique static routes via script.
- **Expected Result:** CPU stable; all 1,000 routes visible in `show ip route summary`.

#### 4.7.2 RT-SCALE-02 Max ECMP Fan-out
- **Steps:** Configure 16 different next-hops for 10.100.100.0/24.
- **Expected Result:** All 16 paths active; traffic hashes across all 16 egress ports.

---

## 5. Cleanup
- `no ip route <prefix> <next-hop>` for all test entries.
- `no vrf instance Tenant_A`.
- `config save`.

---

## 6. Reference
- **JIRA:** ASV2-1971
- **EOS User Manual:** Chapter 14.1 (IPv4/IPv6 Routing)
- **SONiC Community Design Doc:** Static Route HLD
