Slothbot-W
```mermaid
flowchart TD
    Start([Wake Up / Power On]) --> InitHardware[Setup Relays, Motors, Sensors]
    InitHardware --> Wait5s[Wait 5 Seconds]
    Wait5s --> SensorRelayOn[Turn ON Sensor Relay]
    
    %% 40-SECOND WAIT
    SensorRelayOn --> Wait40s[Wait 40 Seconds for Sensors to Stabilize]
    Wait40s --> ReadBatt[Read Bot Battery Voltage]
    
    %% DEEP SLEEP CALCULATION
    ReadBatt --> CalculateSleep{Calculate Sleep Duration}
    CalculateSleep -->|< 9.0V| Sleep3h[3 Hours]
    CalculateSleep -->|< 10.5V| Sleep2h[2 Hours]
    CalculateSleep -->|10.5V - 11.9V| Sleep1h[1 Hour]
    CalculateSleep -->|>= 12.0V| Sleep30m[30 Mins]
    CalculateSleep -->|No Battery| SleepDefault[1 Hour]
    
    Sleep3h --> ReadSensors
    Sleep2h --> ReadSensors
    Sleep1h --> ReadSensors
    Sleep30m --> ReadSensors
    SleepDefault --> ReadSensors
    
    %% SENSOR READING & INITIAL UPLOAD
    ReadSensors[Read All Sensors & Compute Values] --> InitialUpload[Upload Initial Telemetry to CH1-CH4]
    
    %% READ TOGGLES
    InitialUpload --> ReadToggles[Read CH2 F8: Deep Sleep Override<br/>Read CH4 F2: Drone Charging]
    
    %% 1. DRONE CHARGING LOOP
    ReadToggles --> CheckCharging{Is Drone Charging ON?<br/>CH4 F2 == 1}
    
    CheckCharging -->|Yes| ChargingLoopStart([Enter Drone Charging Loop])
    ChargingLoopStart --> UpdateF3_On[Set Charging Status CH4 F3 = 1]
    
    UpdateF3_On --> UploadTelemetry30s[Read Sensors & Upload Telemetry]
    UploadTelemetry30s --> Wait30s[Wait 30 Seconds]
    
    Wait30s --> ReadChargeToggle[Read CH4 F2 from Dashboard]
    ReadChargeToggle --> CheckChargingLoop{Is Drone Charging OFF?<br/>CH4 F2 == 0}
    
    CheckChargingLoop -->|No, Still ON| UploadTelemetry30s
    CheckChargingLoop -->|Yes, Turned OFF| UpdateF3_Off[Set Charging Status CH4 F3 = 0]
    
    %% 2. DEEP SLEEP OVERRIDE LOOP
    UpdateF3_Off --> CheckManual
    CheckCharging -->|No| CheckManual{Is Deep Sleep Override ON?<br/>CH2 F8 == 1}
    
    CheckManual -->|Yes| ManualLoopStart([Enter Deep Sleep Override Loop])
    ManualLoopStart --> PollControls[Check Dashboard Controls:<br/>• Motor: CH4 F4<br/>• Drone Power: CH4 F6<br/>• Surveillance: CH4 F5]
    
    PollControls --> Wait2s[Delay 2 Seconds]
    Wait2s --> ReadManualToggle[Read CH2 F8 from Dashboard]
    
    ReadManualToggle --> CheckManualLoop{Is Override OFF?<br/>CH2 F8 == 0}
    
    CheckManualLoop -->|No, Still ON| PollControls
    CheckManualLoop -->|Yes, Turned OFF| DeepSleepSequence
    
    %% DEEP SLEEP SEQUENCE
    CheckManual -->|No| DeepSleepSequence
    DeepSleepSequence[Disconnect WiFi<br/>Turn OFF Sensor Relay<br/>Delay 500ms] --> SleepNow([Enter Deep Sleep])
    
    SleepNow -->|Wake Up via Timer| Start

```
```mermaid
flowchart LR
    %% POWER SUBSYSTEM
    subgraph Power_System [Power Management Subsystem]
        direction TB
        Batt[Main Battery Pack] --> VoltSense[Voltage Sensor / Divider]
        Batt --> MainReg[Main Power Regulation]
        Batt --> RelayBoard[Relay Control Module]
    end

    %% CORE CONTROLLER
    subgraph Controller [Central Processing & Control]
        direction TB
        MCU[Microcontroller / MCU<br/>• Deep Sleep Timer<br/>• State Machine Logic]
        WiFi[Wi-Fi Transceiver]
        MCU <--> WiFi
    end

    %% SENSING SUBSYSTEM
    subgraph Sensors [Environmental & System Sensing]
        direction TB
        SensorRelay[Sensor Power Relay]
        SensorArray[Sensor Suite<br/>• Environmental Sensors<br/>• Telemetry Monitors]
        SensorRelay -->|Switched Power| SensorArray
    end

    %% ACTUATION & PAYLOADS
    subgraph Actuators [Actuation & Payload Subsystems]
        direction TB
        MotorCtrl[Locomotion / Motor Driver]
        DroneCharge[Drone Charging Interface]
        DronePwr[Drone Power Switch]
        SurvSys[Surveillance Payload]
    end

    %% CLOUD & DASHBOARD
    subgraph Cloud [Cloud Telemetry & Remote Dashboard]
        direction TB
        TS_Data[Telemetry Channels<br/>CH1 – CH4]
        TS_Ctrl[Control & Status Registers<br/>• CH2 F8: Deep Sleep Override<br/>• CH4 F2/F3: Drone Charge Ctrl/Status<br/>• CH4 F4: Motor Control<br/>• CH4 F5: Surveillance Control<br/>• CH4 F6: Drone Power Control]
    end

    %% INTER-SUBSYSTEM CONNECTIONS
    MainReg -->|3.3V / 5V Logic Power| MCU
    VoltSense -->|Battery Voltage| MCU
    
    MCU -->|Relay Trigger| SensorRelay
    SensorArray -->|Sensor Data| MCU
    
    RelayBoard -->|Switched Power| Actuators
    MCU -->|Control Signals| MotorCtrl
    MCU -->|Enable / Status| DroneCharge
    MCU -->|Power Toggle| DronePwr
    MCU -->|Enable Signal| SurvSys

    WiFi <-->|HTTP / API Telemetry Upload| TS_Data
    WiFi <-->|Poll Control Toggles & Update Status| TS_Ctrl
```


```mermaid
flowchart TD
    %% CLOUD & DASHBOARD (TOP LEVEL)
    subgraph Cloud [Cloud Telemetry & Remote Dashboard]
        direction LR
        TS_Data[Telemetry Channels<br/>CH1 – CH4]
        TS_Ctrl[Control & Status Registers<br/>• CH2 F8: Deep Sleep Override<br/>• CH4 F2/F3: Drone Charge Ctrl/Status<br/>• CH4 F4: Motor Control<br/>• CH4 F5: Surveillance Control<br/>• CH4 F6: Drone Power Control]
    end

    %% CORE CONTROLLER (MIDDLE LEVEL)
    subgraph Controller [Central Processing & Control]
        direction LR
        WiFi[Wi-Fi Transceiver] <--> MCU[Microcontroller / MCU<br/>• Deep Sleep Timer<br/>• State Machine Logic]
    end

    %% HARDWARE SUBSYSTEMS (BOTTOM LEVEL)
    subgraph Hardware [Hardware & Power Subsystems]
        direction LR
        
        subgraph Power_System [Power Management]
            direction TB
            Batt[Main Battery Pack] --> VoltSense[Voltage Sensor / Divider]
            Batt --> MainReg[Main Power Regulation]
            Batt --> RelayBoard[Relay Control Module]
        end

        subgraph Sensors [Environmental & System Sensing]
            direction TB
            SensorRelay[Sensor Power Relay] -->|Switched Power| SensorArray[Sensor Suite<br/>• Environmental Sensors<br/>• Telemetry Monitors]
        end

        subgraph Actuators [Actuation & Payloads]
            direction TB
            MotorCtrl[Locomotion / Motor Driver]
            DroneCharge[Drone Charging Interface]
            DronePwr[Drone Power Switch]
            SurvSys[Surveillance Payload]
        end
    end

    %% TOP-DOWN INTER-SUBSYSTEM CONNECTIONS
    TS_Data <-->|HTTP / API Telemetry Upload| WiFi
    TS_Ctrl <-->|Poll Control Toggles & Update Status| WiFi

    MainReg -->|3.3V / 5V Logic Power| MCU
    VoltSense -->|Battery Voltage| MCU

    MCU -->|Relay Trigger| SensorRelay
    SensorArray -->|Sensor Data| MCU

    RelayBoard -->|Switched Power| Actuators
    MCU -->|Control Signals| MotorCtrl
    MCU -->|Enable / Status| DroneCharge
    MCU -->|Power Toggle| DronePwr
    MCU -->|Enable Signal| SurvSys
