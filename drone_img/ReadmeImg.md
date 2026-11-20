flowchart TD
    %% Academic Styling (Clean borders, high contrast for print/PDF export)
    classDef term fill:#e9ecef,stroke:#212529,stroke-width:2px,font-weight:bold,color:#000;
    classDef proc fill:#ffffff,stroke:#343a40,stroke-width:1.5px,color:#000;
    classDef dec fill:#f1f5f9,stroke:#1e3a8a,stroke-width:1.5px,color:#000;
    classDef mode fill:#f8fafc,stroke:#475569,stroke-width:1.5px,stroke-dasharray: 4 3,color:#000;

    Start(["<b>Wake-Up / Power-On Reset</b><br/><i>(RTC Timer Interrupt)</i>"]):::term

    subgraph Phase1 ["<b>1. Initialization & Telemetry Phase</b>"]
        direction TB
        Init["<b>Hardware Setup & Sensor Stabilization</b><br/>Enable Relays & Stabilize Environmental Sensors"]:::proc
        Batt["<b>Battery Evaluation & Sleep Scheduling</b><br/>Measure V<sub>bat</sub> & Assign T<sub>sleep</sub> (30m to 3h)"]:::proc
        Upload["<b>Initial Telemetry & Control Polling</b><br/>Acquire Sensor Data, Upload to Cloud & Read Mode Flags"]:::proc
        
        Init --> Batt --> Upload
    end

    Start --> Init

    subgraph Phase2 ["<b>2. Cloud-Triggered Operational Modes</b>"]
        direction TB
        CheckCharge{"<b>Drone Charging<br/>Requested?</b>"}:::dec
        ChargeMode["<b>Drone Charging Mode</b><br/>• Enable Charging Status Flag<br/>• Continuous Telemetry Upload (30s Cycle)"]:::mode
        
        CheckOverride{"<b>Manual Override<br/>Active?</b>"}:::dec
        ManualMode["<b>Remote Control Mode</b><br/>• Poll Actuator Commands (2s Cycle)<br/>• Control Motors, Drone Power & Camera"]:::mode

        CheckCharge -- "Yes" --> ChargeMode
        ChargeMode -- "While Active" --> ChargeMode
        ChargeMode -- "Deactivated" --> CheckOverride
        CheckCharge -- "No" --> CheckOverride

        CheckOverride -- "Yes" --> ManualMode
        ManualMode -- "While Active" --> ManualMode
    end

    Upload --> CheckCharge

    subgraph Phase3 ["<b>3. Power-Saving Termination</b>"]
        direction TB
        Shutdown["<b>Peripheral Shutdown</b><br/>Disconnect Wi-Fi & Power Off Sensor Relay"]:::proc
        Sleep(["<b>Enter Adaptive Deep Sleep (T<sub>sleep</sub>)</b>"]):::term
        
        Shutdown --> Sleep
    end

    ManualMode -- "Deactivated" --> Shutdown
    CheckOverride -- "No" --> Shutdown
    Sleep -.->|"Wake Cycle"| Start
