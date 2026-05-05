# Architecture & Wiring Notes

Diagrams here are rendered with Mermaid.js. Most Markdown previewers (GitHub, VS Code with extension, Obsidian) render these natively.

## System Topology

```mermaid
graph LR
    subgraph Power
        BAT[18650 Cell]
        TP[TP4056 Charger]
        LDO[HT7333-A 3.3V LDO]
        SOLAR[5V Solar Panel\nfuture]
        BAT --> TP
        SOLAR -.->|charge input| TP
        TP --> LDO
    end

    subgraph ESP32-C3 Node
        MCU[ESP32-C3]
        LDO -->|3.3V| MCU
        SENS[Capacitive Sensor v1.2\nonboard LDO bridged]
        MCU -->|3.3V| SENS
        SENS -->|AOUT| MCU
    end

    MCU -->|Wi-Fi MQTT| BROKER[Mosquitto\nMQTT Broker]
    BROKER --> HA[Home Assistant]
```

## Multiplexer Wiring (Front Railing)

```mermaid
graph TD
    PSU[5V PSU] -->|18-20 AWG trunk| BUS_VCC[VCC Bus]
    PSU -->|18-20 AWG trunk| BUS_GND[GND Bus]

    BUS_VCC --> S1[Sensor 1 VCC]
    BUS_VCC --> S2[Sensor 2 VCC]
    BUS_VCC --> S10[Sensor 10 VCC]
    BUS_GND --> S1G[Sensor 1 GND]
    BUS_GND --> S2G[Sensor 2 GND]
    BUS_GND --> S10G[Sensor 10 GND]

    S1 -->|AOUT| MUX_CH0[MUX CH0]
    S2 -->|AOUT| MUX_CH1[MUX CH1]
    S10 -->|AOUT| MUX_CH9[MUX CH9]

    subgraph CD74HC4067
        MUX_CH0 & MUX_CH1 & MUX_CH9 --> SIG[SIG out]
        SEL[S0-S3 select]
    end

    ESP[ESP32-S3] -->|GPIO11-14| SEL
    SIG -->|GPIO1 ADC1| ESP
    ESP -->|Wi-Fi MQTT| HA[Home Assistant]
```

## Deep Sleep Power Budget (per mobile node)

| State | Current Draw | Duration |
|---|---|---|
| Active (Wi-Fi + ADC) | ~80 mA | 30 s |
| Deep Sleep | ~10 µA | 15 min |
| **Avg over cycle** | **~2.8 mA** | — |

A 2500 mAh 18650 cell at this average gives roughly **37 days** of runtime without solar.
With a 100 mA solar panel in 4 peak sun hours/day, the node runs indefinitely.

## Sensor Bypass Detail

```
Sensor v1.2 PCB (top view, schematic):

VCC_IN ──[LDO 5V→3.3V]──> VCC_3V3 ──> 555 Timer ──> CD ──> AOUT
                              ↑
                           [bridge this with solder]
                              |
VCC_IN ────────────────────────

Bridging connects VCC_IN directly to the 555 timer supply,
bypassing the LDO. Feed 3.3V to VCC_IN; the timer sees 3.3V.
```

## Pin Assignment Summary

### ESP32-C3 (Mobile Nodes)

| GPIO | Function |
|---|---|
| GPIO0 | Reserved (boot strapping) |
| GPIO1 | ADC1 — Sensor 1 AOUT |
| GPIO2 | ADC1 — Sensor 2 AOUT (apple tree only) |
| GPIO3 | ADC1 — Battery voltage divider |
| GPIO4 | ADC1 — spare |

### ESP32-S3 (Raised Bed)

| GPIO | Function |
|---|---|
| GPIO1 | ADC1 — Blueberries |
| GPIO2 | ADC1 — Carrots |
| GPIO3 | ADC1 — Peppers |
| GPIO4 | ADC1 — Basil |
| GPIO5 | ADC1 — Zucchini |

### ESP32-S3 (Front Railing + Multiplexer)

| GPIO | Function |
|---|---|
| GPIO1 | ADC1 — MUX SIG input |
| GPIO11 | Digital OUT — MUX S0 |
| GPIO12 | Digital OUT — MUX S1 |
| GPIO13 | Digital OUT — MUX S2 |
| GPIO14 | Digital OUT — MUX S3 |
