<mark>__*Work In Progress*__</mark>

### 6G Dynamic Cell On/Off Utilizing Upink WUS(Wakeup Signal)
* **Dynamic Cell ON/OFF with Carrier Separation** – Separating coverage carriers from capacity carriers and operating them independently helps reduce unnecessary power consumption and improve network energy efficiency, by dynamically turning on/off the capacity carriers when necessary. This technology maximizes the deep sleep periods of network nodes and user equipment (UE), significantly enhancing the energy efficiency of the network by reducing unnecessary active states and optimizing power consumption during idle periods. Both Network-initiated and UE-initiated/assisted strategies for dynamic cell on/off can be exploited
* **Low-Power Wake-up Signals by Necessity** – Low-power wake-up radio (LP-WUR) and wake-up signal (WUS) technologies allow UEs and gNodeBs to remain in deep sleep states until absolutely necessary, further reducing power consumption. Beyond 5G’s downlink LP-WUS, for 6G, this technology can be further extended and optimized to support both downlink and uplink transmission/ reception of wake-up signals. By enabling bidirectional WUS communication, 6G can enhance network responsiveness and energy efficiency simultaneously by extending the UE and base station’s deep sleep durations. Additionally, integrating LP-WUS with other advanced technologies like AI-driven traffic prediction and on-demand signaling can create a more cohesive, contextual and intelligent energy-saving ecosystem.
* Also check **Booster carrier sleep x.0** for NR cell functionality, and how that can help in 6G or still same functionality can be inherited to 6G as well.

# Call Flow: Dynamic Cell On/Off Utilizing Uplink WUS (RAN Internal Stack)

This call flow illustrates the internal protocol stack interactions (L3/RRC, L2/MAC-RLC, L1/PHY) within a disaggregated or multi-carrier RAN node (gNB) to support dynamic Network Energy Saving (NES).

## 1. Network Nodes & Protocol Layers
* **UE**: User Equipment (supports 5G-Advanced / 6G low-power UL WUS features).
* **PCell (Coverage Cell)**: An always-on macro cell providing basic coverage and RRC control.
* **SCell (Capacity Cell)**: A small or directional cell that switches ON/OFF dynamically to save power.
* **L3 (RRC/RRM)**: Handles radio resource configuration, measurement controls, and energy-saving decision policies.
* **L2 (MAC/RLC/PDCP)**: Manages scheduling, MAC Control Elements (CE), and fast cell activation hooks.
* **L1 (PHY)**: Handles physical signal transmission/reception (PRACH-like or sequence-based UL WUS, SRS, SSB, and CSI-RS).

---

## 2. Sequence Diagram: Cell Off (Deactivation Phase)

```
  UE              SCell L1        SCell L2        SCell L3         PCell L3

  |                  |               |               |                |
  |                  |               |        1. Monitor Load         |
  |                  |               |---------------|                |
  |                  |               |   (Low Traffic / Idle UEs)     |
  |                  |               |               |                |
  |                  |               |     2. Initiate Deactivation   |
  |                  |               |               |--------------->|
  |                  |               |               |  (Cell Off req)|
  |                  |               |               |                |
  |                  |      3. Prepare UL WUS Config |                |
  |                  |<------------------------------|                |
  |                  | (Program WUS sequence/window) |                |
  |                  |               |               |                |
  |                  |               |               | 4. Sync Config |
  |                  |               |               |<==============>|
  |                  |               |               |                |
  |                  | 5. Transmit RRC Reconfiguration                |
  |<------------------------------------------------------------------|
  |             (Contains UL WUS resources & SCell Deactivate)        |
  |                  |               |               |                |
  |                  |         6. Enter Deep Sleep (RF Off)           |
  |                  |<------------------------------|                |
  |                  |    (L1 RX leaves only low-power WUS correlator)|
```

---

## 3. Sequence Diagram: Cell On (Uplink WUS Triggered Phase)

```
  UE              SCell L1        SCell L2        SCell L3         PCell L3

  |                  |               |               |                |
  |   7. Burst Data arrives          |               |                |
  |----|             |               |               |                |
  |    |             |               |               |                |
  |<---|             |               |               |                |
  |                  |               |               |                |
  | 8. Transmit UL WUS (L1 Sequence) |               |                |
  |----------------->|               |               |                |
  |                  |               |               |                |
  |                  | 9. Detect WUS |               |                |
  |                  |----|          |               |                |
  |                  |    |          |               |                |
  |                  |<---|          |               |                |
  |                  |               |               |                |
  |                  | 10. L1 Wakeup Interrupt       |                |
  |                  |-------------->|               |                |
  |                  |               | 11. Escalate Power-On          |
  |                  |               |-------------->|                |
  |                  |               |               |                |
  |                  |         12. RF HW Power Up    |                |
  |                  |<------------------------------|                |
  |                  |    (Synthesizers & PAs active)|                |
  |                  |               |               |                |
  |                  |               |               | 13. Notify Active
  |                  |               |               |===============>|
  |                  |               |               |                |
  |                  | 14. Transmit MAC CE (SCell Activation)         |
  |<------------------------------------------------------------------|
  |                  |               |               |                |
  | 15. DL Sync & Meas (SSB / CSI-RS) |               |                |
  |<-----------------|               |               |                |
  |                  |               |               |                |
  | 16. Fast CSI / SRS reporting     |               |                |
  |----------------->|               |               |                |
  |                  |               |               |                |
  | 17. High-Throughput User Data plane Active       |                |
  |<================>|<=============>|               |                |
```

---

## 4. Detailed Step-by-Step Description

### Phase A: SCell Deactivation & Sleep Entry
1. **Load Monitoring (SCell L3)**: The Radio Resource Management (RRM) layer inside **SCell L3** detects that the cell traffic load has fallen below a certain threshold (e.g., zero active data bearers, connected UEs are idle).
2. **Deactivation Request**: **SCell L3** determines it can safely turn off to save energy. It alerts the **PCell L3** coordination engine to coordinate traffic steering or handling.
3. **Configure UL WUS Detector**: **SCell L3** programmatically configures its local **L1 (PHY)** to allocate specific physical time/frequency resources (resembling a low-overhead PRACH preamble or specialized sequence) dedicated for an Uplink Wake-up Signal.
4. **Config Sync**: **SCell L3** synchronizes this configuration details (frequency, sequence ID, wake-up window timing) with **PCell L3** over internal messaging interfaces (e.g., F1/Xn interfaces or intra-gNB control bus).
5. **RRC Reconfiguration**: The **PCell** transmits an `RRCReconfiguration` message to the **UE**. This message provisions the UE with the explicit **UL WUS parameters** for the SCell and instructs the UE to treat the SCell as deactivated.
6. **Deep Sleep State**: **SCell L3** commands **SCell L1** to power down the Power Amplifiers (PAs) and Radio Frequency (RF) front-end chains. Only a highly specialized, low-power hardware correlator circuit in **L1** stays active to sniff for the pre-configured UL WUS sequence.

### Phase B: Dynamic Wake-up (Cell On)
7. **Traffic Trigger**: The **UE** receives a sudden burst of high-priority or high-throughput uplink data in its buffer while connected to the PCell.
8. **UL WUS Transmission**: Per its RRC configuration, instead of going through a slow, high-overhead secondary cell activation process via network signaling, the **UE L1** directly fires the low-power physical **UL WUS** sequence directly targeted at the sleeping SCell.
9. **L1 Detection**: The **SCell L1** low-power correlator circuit detects the energy sequence matching the wake-up profile.
10. **L1 Interrupt**: **SCell L1** fires an ultra-fast hardware interrupt directly up to the **L2 (MAC)** scheduling layer to bypass normal pipeline latencies.
11. **Escalation**: **SCell L2** immediately triggers the **SCell L3** supervisor loop to report an unscheduled wake-up request.
12. **RF Hardware Power-Up**: **SCell L3** commands the RF chain, local oscillators, and PAs to transition from deep sleep to the fully active state.
13. **Status Notification**: **SCell L3** notifies **PCell L3** that its physical layers are fully operational and ready to process traffic.
14. **MAC CE Confirmation**: **PCell L2** issues a fast **SCell Activation MAC Control Element (MAC CE)** to the UE, providing definitive confirmation that the capacity layer is hot.
15. **Reference Signaling**: The **SCell L1** begins streaming on-demand Synchronization Signal Blocks (SSB) and Channel State Information Reference Signals (CSI-RS).
16. **Channel Profiling**: The **UE** performs fast downlink synchronization and measures the SCell, immediately returning CSI feedback or sounding reference signals (SRS) back to **SCell L1**.
17. **Data Flow**: The **SCell L2** scheduler opens up dynamic resource allocation loops. High-throughput data planes are now fully active on the capacity cell.



