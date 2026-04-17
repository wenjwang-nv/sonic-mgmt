# Dual VXLAN Tunnel Test Plan

- [Overview](#overview)
  - [Test Purpose](#test-purpose)
  - [Scope](#scope)
  - [Testbed](#testbed)
  - [Setup Configuration](#setup-configuration)
- [Test Cases](#test-cases)
- [Open Questions](#open-questions)

## Overview
This test plan verifies support for two VXLAN tunnels coexisting on the same DUT:
- one tunnel with outer IPv4
- one tunnel with outer IPv6

The validation targets SPC3 platforms in T1 topology.

Target branches:
- `master`
- `202511`
- `202505`

The primary control-plane test implementation will be based on `sonic-mgmt/tests/vxlan/test_vxlan_multi_tunnel.py`.
Dataplane packet validation will reuse and extend the traffic-checking approach used in existing VXLAN test cases.

### Test Purpose
This test plan validates dual VXLAN tunnel support on a single DUT in T1 topology, with a focus on:
- coexistence of one outer-IPv4 tunnel and one outer-IPv6 tunnel
- successful VNET and route programming on top of the two tunnels
- basic VXLAN dataplane behavior before and after warm reboot

### Scope
This test covers:
- T1 topology only, on SPC3 platforms
- positive dual-tunnel creation scenarios, including different creation orders and VNI modes
- negative tunnel creation scenarios, including unsupported tunnel combinations
- VXLAN tunnel, VNET, and route state verification after setup and after warm reboot
- basic VXLAN dataplane validation (`encap`/`decap`) for both tunnels before and after warm reboot

### Testbed
The test will run on SPC3 platforms in the T1 role.

Platforms in scope:
- `SN4600C`
- `SN4700`
- `SN4280`

### Setup Configuration
Each scenario starts from a clean baseline.

For each successful positive scenario, configure the VNET and route setup required to run basic traffic checks through both tunnels:
- one traffic path using the outer-IPv4 tunnel
- one traffic path using the outer-IPv6 tunnel
- VNI mode selected by test parameter:
  - same VNI across the two traffic paths
  - different VNIs across the two traffic paths

Traffic validation in this phase will extend the packet-checking approach used by existing VXLAN test cases. For both encap and decap directions, the test verifies the outer IP family, VNI, TTL, QoS, and payload. After warm reboot, the test rechecks the state and repeats the same traffic validation.

Cleanup:
- remove both tunnels and related forwarding state
- restore the agreed baseline

## Test Cases

### 1. Positive Flow Test
This test validates successful dual-tunnel setup, basic VXLAN traffic, and post-warm-reboot state retention. The test is parameterized by the following setup inputs:

Tunnel creation order:
- IPv4 then IPv6
- IPv6 then IPv4

VNI mode:
- same VNI across the two traffic paths
- different VNIs across the two traffic paths

| Step | Action | Check |
|------|--------|-------|
| 1 | Set up the baseline. | The DUT is ready for the selected positive scenario. |
| 2 | Create the VXLAN tunnels. | The expected tunnel objects are present in `APP_DB` under `VXLAN_TUNNEL_TABLE`. |
| 3 | Create the VNETs and routes. | The expected VNET objects are present in `APP_DB` under `VNET_TABLE` with the expected `vxlan_tunnel` and `vni`; the expected route entries are present in `ASIC_DB`. |
| 4 | Run the encap traffic test. | Traffic direction is `T0 -> T2`; packets egress through the expected tunnel with the expected outer IP family, VXLAN UDP destination port, TTL, QoS, and VNI; the inner payload remains correct. |
| 5 | Run the decap traffic test. | Traffic direction is `T2 -> T0`; packets enter through the expected tunnel with the expected outer IP family, TTL, QoS, and VNI, are decapsulated successfully, and are forwarded to the expected T0-side port; the inner payload remains correct. |
| 6 | Perform warm reboot. | The DUT reboots and recovers successfully. |
| 7 | Recheck state after warm reboot. | Results are consistent with steps 2 and 3. |
| 8 | Rerun the encap and decap traffic tests. | Results are consistent with steps 4 and 5. |
| 9 | Clean up. | Tunnel, VNET, and route objects are removed from `APP_DB` and `ASIC_DB`, and the DUT returns to baseline. |

### 2. Negative Tunnel Creation Test
This test validates invalid tunnel creation combinations. The test is parameterized by the following setup inputs:
- two IPv4 tunnels
- two IPv6 tunnels
- more than two tunnels

| Step | Action | Check |
|------|--------|-------|
| 1 | Set up the baseline. | The DUT is ready for the selected negative scenario. |
| 2 | Attempt invalid tunnel creation. | The invalid tunnel creation request is rejected. |
| 3 | Verify DB state. | Rejected tunnel objects are not present in `APP_DB` under `VXLAN_TUNNEL_TABLE`; valid tunnel objects, if any, remain consistent. |
| 4 | Verify system stability. | No unexpected critical error is observed, and no unexpected object is programmed into `ASIC_DB`. |
| 5 | Clean up. | Any remaining configuration is removed, and the DUT returns to baseline. |

## Open Questions
- In our use case, we use the default TTL and QoS settings only. Do we still want to validate mismatched or dynamically updated TTL/QoS configurations?
- Is the recreate case necessary?