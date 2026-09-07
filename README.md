<mark>__*Work In Progress*__</mark>

### 6G Dynamic Cell On/Off Utilizing Upink WUS(Wakeup Signal)
* **Dynamic Cell ON/OFF with Carrier Separation** – Separating coverage carriers from capacity carriers and operating them independently helps reduce unnecessary power consumption and improve network energy efficiency, by dynamically turning on/off the capacity carriers when necessary. This technology maximizes the deep sleep periods of network nodes and user equipment (UE), significantly enhancing the energy efficiency of the network by reducing unnecessary active states and optimizing power consumption during idle periods. Both Network-initiated and UE-initiated/assisted strategies for dynamic cell on/off can be exploited
* **Low-Power Wake-up Signals by Necessity** – Low-power wake-up radio (LP-WUR) and wake-up signal (WUS) technologies allow UEs and gNodeBs to remain in deep sleep states until absolutely necessary, further reducing power consumption. Beyond 5G’s downlink LP-WUS, for 6G, this technology can be further extended and optimized to support both downlink and uplink transmission/ reception of wake-up signals. By enabling bidirectional WUS communication, 6G can enhance network responsiveness and energy efficiency simultaneously by extending the UE and base station’s deep sleep durations. Additionally, integrating LP-WUS with other advanced technologies like AI-driven traffic prediction and on-demand signaling can create a more cohesive, contextual and intelligent energy-saving ecosystem.
* Also check **Booster carrier sleep x.0** for NR cell functionality, and how that can help in 6G or still same functionality can be inherited to 6G as well.

# Call Flow: Dynamic Cell On/Off Utilizing Uplink WUS (RAN Internal Stack)

This call flow illustrates the internal protocol stack orchestration (L3/RRC, L2/MAC, L1/PHY) for **Dynamic Cell On/Off utilizing Uplink Wake-up Signals (UL WUS)** based on 3GPP concepts (Release 18+ Network Energy Saving). It includes standard-aligned **Information Elements (IEs)**, protocol log parameters, and a **Machine Learning (ML)** framework for optimizing the cell-activation threshold.

## 1. Network Nodes & Protocol Layers
* **UE**: User Equipment (supports 5G-Advanced / 6G low-power UL WUS features) i.e. supporting low-power UL WUS sequences.
* **PCell (Coverage Cell)**: An always-on macro cell providing basic coverage and RRC control.
* **SCell (Capacity Cell)**: The target cell transitioning dynamically between **Active (On)** and **Deep Sleep (Off)**.
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


## 5. Comprehensive Call Flow Diagram

```
  UE                    SCell L1               SCell L2               SCell L3               PCell L3
  |                        |                      |                      |                      |
  |                        |                      |             1. Monitor Load                 |
  |                        |                      |----------------------|                      |
  |                        |                      |   (Traffic < threshold)                     |
  |                        |                      |                      |                      |
  |                        |                      |             2. Initiate Deactivation        |
  |                        |                      |-------------------------------------------->|
  |                        |                      |                      | [CellDeactivationReq]|
  |                        |                      |                      |                      |
  |                        |             3. Prepare UL WUS Hardware      |                      |
  |                        |<--------------------------------------------|                      |
  |                        |    [Config: wus-SequenceId, wus-TimeWindow] |                      |
  |                        |                      |                      |                      |
  |                        |                      |                      | 4. Sync Configuration|
  |                        |                      |                      |<====================>|
  |                        |                      |                      |  [Xn-AP / F1-AP Msg] |
  |                        |                      |                      |                      |
  |             5. RRCReconfiguration (via PCell)                        |                      |
  |<--------------------------------------------------------------------------------------------|
  |  [IE: SCellToAddModList-v18, ul-WUS-Config-r18, wus-ResourceConfig]  |                      |
  |                        |                      |                      |                      |
  |                        |               6. Enter Deep Sleep (RF Off)  |                      |
  |                        |<--------------------------------------------|                      |
  |                        |  (Leaves only low-power correlator awake)   |                      |
  |                        |                      |                      |                      |
  |============================================================================================|
  |                        |             WAKE-UP SIGNAL ACTIVATION PHASE                        |
  |============================================================================================|
  |                        |                      |                      |                      |
  | 7. Burst Data Arrives  |                      |                      |                      |
  |---                     |                      |                      |                      |
  |  | (Buffer > Threshold)|                      |                      |                      |
  |<--                     |                      |                      |                      |
  |                        |                      |                      |                      |
  | 8. Transmit UL WUS     |                      |                      |                      |
  |----------------------->|                      |                      |                      |
  |  [Sequence/Preamble]   |                      |                      |                      |
  |                        | 9. Match & Detect    |                      |                      |
  |                        |---                   |                      |                      |
  |                        |  | (RSSI > Metric)   |                      |                      |
  |                        |<--                   |                      |                      |
  |                        |                      |                      |                      |
  |                        | 10. HW Interrupt     |                      |                      |
  |                        |--------------------->|                      |                      |
  |                        |                      | 11. Escalate Power-On|                      |
  |                        |                      |--------------------->|                      |
  |                        |                      |                      |                      |
  |                        |              12. RF HW Power-Up Circuit     |                      |
  |                        |<--------------------------------------------|                      |
  |                        |    (Synthesizers & Power Amps stabilized)   |                      |
  |                        |                      |                      |                      |
  |                        |                      |                      | 13. State Active Sync|
  |                        |                      |                      |=====================>|
  |                        |                      |                      |  [SCellStateNotify]  |
  |                        |                      |                      |                      |
  | 14. Receive MAC CE (SCell Activation)         |                      |                      |
  |<----------------------------------------------|---------------------------------------------|
  |   [Octet 1: Bi-Map Active, LCID: SCell Act]   |                      |                      |
  |                        |                      |                      |                      |
  | 15. DL Sync & Meas     |                      |                      |                      |
  |<-----------------------|                      |                      |                      |
  |  [SSB / CSI-RS Burst]  |                      |                      |                      |
  |                        |                      |                      |                      |
  | 16. Fast Channel Report|                      |                      |                      |
  |----------------------->|                      |                      |                      |
  |  [CSI Feedback / SRS]  |                      |                      |                      |
  |                        |                      |                      |                      |
  | 17. High-Throughput User Data plane Active    |                      |                      |
  |<======================>|<====================>|                      |                      |
```

---
## 6. Draft Call Flow for Booster Carrier Sleep x.5

```text
  UE             Primary Carrier (Anchor)         eNodeB/gNodeB          Booster Carrier
  |                         |                           |                       |
  |================== ACTIVE STATE =====================|                       |
  |                         |                           |                       |
  |--- Uplink/Downlink Data |                           |                       |
  |    Traffic              |                           |                       |
  |<----------------------->|                           |                       |
  |                         |                           |                       |
  |                         |=== Traffic Monitored =====|                       |
  |                         |    (Load below threshold) |                       |
  |                         |                           |                       |
  |                         |====================== SLEEP INITIATION ===========|
  |                         |                           |                       |
  |                         |                           |--- Deactivate Cell -->|
  |                         |                           |    (Booster Sleep 1.5)|
  |                         |                           |<----------------------|
  |                         |<-- Update Available RIDs -|                       |
  |                         |                           |                       |
  |================== SLEEP STATE (Power Saved) ========|                       |
  |                         |                           |                       |
  |--- Sudden Traffic Burst |                           |                       |
  |    (High Buffer/Load)   |                           |                       |
  |<----------------------->|                           |                       |
  |                         |                           |                       |
  |                         |=== Threshold Exceeded ====|                       |
  |                         |                           |                       |
  |                         |====================== RAPID WAKE-UP SEQUENCE =====|
  |                         |                           |                       |
  |                         |                           |--- Fast Resume ------>|
  |                         |                           |    (Transmitter ON)   |
  |                         |                           |<----------------------|
  |                         |<-- Cell Activated --------|                       |
  |                         |                           |                       |
  |================== DUAL CONNECTIVITY / CA RESUMED ===|                       |
  |                         |                           |                       |
  |--- CA Configuration/ ---|                           |                       |
  |    Activation Command   |                           |                       |
  |<------------------------|                           |                       |
  |                         |                           |                       |
  |<----------------------- High-Speed Data Flow (CA Active) ------------------>|
```

## 7. 3GPP-Aligned Information Elements & Log Parameters

### 7.1 RRC Configuration Structure (ASN.1 Representation)
When configuring the UE to use UL WUS for an energy-saving secondary cell, the network injects specific fields into the `SCellToAddModList` within the `RRCReconfiguration` message:

```asn
SCellToAddModList-v1800 ::= SEQUENCE (SIZE (1..maxSCells)) OF SCellToAddMod-v1800

SCellToAddMod-v1800 ::= SEQUENCE {
    sCellIndex-r18              SCellIndex,
    ul-WUS-Config-r18           CHOICE {
        release                 NULL,
        setup                   SEQUENCE {
            wus-SequenceId-r18      INTEGER (0..1007),
            wus-TimeWindowConfig    SEQUENCE {
                periodicity-r18         ENUMERATED {sf10, sf20, sf40, sf80, sf160},
                offset-r18              INTEGER (0..159)
            },
            wus-FrequencyResource   SEQUENCE {
                startPRB-r18            INTEGER (0..274),
                numPRBs-r18             ENUMERATED {n4, n8, n12, n16}
            },
            detectionThreshold-r18  INTEGER (-140..-40) -- Configured in dBm
        }
    }
}
```

### 7.2 MAC Control Element (CE) Format
Once L1 detects the signal, the network confirms activation using a standardized **SCell Activation/Deactivation MAC CE**. For a system with up to 7 secondary cells, a single octet format is utilized:

```
+----+----+----+----+----+----+----+----+
| R  | C7 | C6 | C5 | C4 | C3 | C2 | C1 |  Octet 1
+----+----+----+----+----+----+----+----+
```
* **Ci Fields**: Set to `1` to confirm that SCell Index $i$ is immediately shifted into the **Active State** following the UL WUS interrupt pipeline.
* **LCID (Logical Channel ID)**: Identifies this specific payload configuration as an Activation payload structure.

### 7.3 Target L1 Protocol Log Fields
Engineers inspecting cell logs at the physical layer look for these structural fields during trace validation:
* `WUS_DETECTION_STATUS`: `DETECTED` / `FALSE_ALARM` / `NONE`
* `WUS_MEASURED_RSSI`: Expected measurement target range between `-125 dBm` and `-70 dBm`.
* `WUS_CORRELATION_PEAK`: Metric mapping the mathematical score output against the detection threshold floor.
* `WAKEUP_INTERRUPT_LATENCY_US`: Microsecond runtime duration tracking the physical trigger edge to the L2 layer cascade.

---

## 8. Machine Learning Module for Adaptive Cell Activation

To maximize network power savings while preventing latency drops, the gNB-CU/DU runs an **Intelligent Energy Management Loop**. It uses Q-Learning (Reinforcement Learning) to dynamically decide when to transition the cell to deep sleep or wake it up based on user traffic velocity and buffer status.

Below is the complete executable simulation script. It maps historical traffic surges, runs the environment simulation loop, tracks the rewards achieved by the policy matrix, and outputs the optimization metrics.

```python
import numpy as np

class RanEnergyEnvironment:
    def __init__(self):
        # States: 0 = Low Traffic, 1 = Medium Traffic, 2 = High Burst
        self.num_states = 3
        # Actions: 0 = Keep Deep Sleep (Off), 1 = Trigger Fast Wake-up (On)
        self.num_actions = 2
        self.current_state = 0
        
    def step(self, action):
        # Simulate traffic transitions based on random network user demands
        next_state = np.random.choice([0, 1, 2], p=[0.5, 0.3, 0.2] if self.current_state == 0 else [0.2, 0.5, 0.3])
        
        reward = 0
        if action == 0: # Choose Deep Sleep
            if self.current_state == 0:
                reward = 15  # Optimal: High power savings, zero performance penalty
            elif self.current_state == 1:
                reward = 2   # Minor buffer delay build up
            else:
                reward = -25 # Bad Quality of Service: Missing high data burst
        else: # Choose Wake-Up
            if self.current_state == 0:
                reward = -10 # Energy waste: Active hardware with zero loading
            elif self.current_state == 1:
                reward = 10  # Good buffer preparation
            else:
                reward = 25  # Optimal performance alignment for high capacity
                
        self.current_state = next_state
        return next_state, reward

def train_ran_policy(episodes=1000, alpha=0.1, gamma=0.9, epsilon=0.2):
    env = RanEnergyEnvironment()
    # Initialize Q-Table with zeros
    q_table = np.zeros((env.num_states, env.num_actions))
    
    print("Starting Machine Learning Optimization Routine...")
    print(f"Initial Q-Table Map:\n{q_table}\n")
    
    for ep in range(episodes):
        state = env.current_state
        if np.random.rand() < epsilon:
            action = np.random.choice(env.num_actions) # Explore
        else:
            action = np.argmax(q_table[state]) # Exploit
            
        next_state, reward = env.step(action)
        
        # Bellman Equation Update formula
        best_next_action = np.argmax(q_table[next_state])
        q_table[state, action] += alpha * (reward + gamma * q_table[next_state, best_next_action] - q_table[state, action])
        
    print(f"Optimized Policy Q-Table (Trained over {episodes} cycles):")
    print("Columns correspond to Actions -> [0: Stay Asleep, 1: Wake Up]")
    print(f"{q_table}\n")
    
    # Derived logic verification rules
    states_desc = ["Low Traffic", "Medium Traffic", "High Burst Traffic"]
    for s in range(env.num_states):
        best_act = np.argmax(q_table[s])
        act_desc = "Keep Deep Sleep (Off)" if best_act == 0 else "Trigger Fast Wake-up (On)"
        print(f"Optimal Choice for State [{states_desc[s]}]: -> {act_desc}")

if __name__ == '__main__':
    # Ensure reproducible outcomes for standard tracking validation
    np.random.seed(42)
    train_ran_policy()
```




