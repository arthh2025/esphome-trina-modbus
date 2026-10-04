# Trina Nexeos Modbus RTU Integration for ESPHome and Home Assistant

Read operating data from a Trina Nexeos hybrid inverter through its local RS485 interface, publish the values through ESPHome, and use them in Home Assistant without relying on a cloud API.

> **Project status:** Experimental but working for the listed read-only values. The base RS485 communication, inverter telemetry, battery telemetry, PV telemetry, and live smart-meter power have been tested. Native grid-import and grid-export energy counters are not yet resolved; the example configuration therefore calculates these counters locally from the live smart-meter power.

## Background and Purpose

The Nexeos inverter exposes a documented Modbus RTU interface, but there was no ready-to-use ESPHome configuration for the combination of a Nexeos inverter and the LILYGO T-CAN485 board.

This project provides:

- a tested LILYGO T-CAN485 hardware configuration;
- a local Modbus RTU connection to the inverter;
- automatic discovery of the ESPHome device in Home Assistant;
- PV, inverter, battery, status, and smart-meter entities;
- locally integrated grid-import and grid-export energy counters;
- writable upper and lower battery state-of-charge limits;
- a documented starting point for further register research.

The entity names in the supplied YAML are intentionally written in German. This keeps the names consistent with a German-language Home Assistant installation. The ESPHome IDs, comments, and documentation are in English so that the configuration remains understandable and maintainable for an international audience.

### Motivation

Getting the integration up and running required a considerable amount of time, so I wanted to make the process faster and easier for others. I hope the example provided here helps you start reading inverter data more easily than I did, especially because the Modbus configuration and the initial ESP32 setup were the most time-consuming parts.

Once configured correctly, the communication has been stable and reliable. AI tools were used extensively to support the creation of the YAML configuration and this documentation. The resulting configuration was then tested against the actual inverter data and refined iteratively.

## Disclaimer and Safety

This is an unofficial community project and is not affiliated with or endorsed by Trina Solar, LILYGO, ESPHome, or Home Assistant.

- Verify the RJ45 pin numbering from the inverter manual before connecting the board.
- Use write-enabled Modbus entities carefully. Incorrect setpoints can change battery operation.
- Start with read-only values and validate each value against the inverter app or display.
- Use this project at your own risk.

## Tested Hardware and Software

- Trina Nexeos hybrid inverter (TRH 10K-T3 10kW) with an RS485/COM2 interface
  - According to the Modbus manual, other inverter brands or models may use the same protocol or register structure. Compatibility has not been verified and should not be assumed without testing.
- LILYGO TTGO T-CAN485 based on ESP32
  - Integrated MAX13487E RS485 transceiver
- Home Assistant with the ESPHome Device Builder
- ESPHome 2026.8.x during development

Other Nexeos firmware revisions and inverter variants may expose different registers or scaling factors.

## Repository Structure

```text
esphome-nexeos-modbus/
├── README.md
├── LICENSE
├── nexeos-tcan485.yaml
└── secrets.example.yaml
```

## Hardware Connection

### LILYGO T-CAN485 Pins

| Function | ESP32 GPIO |
|---|---:|
| RS485 TX | GPIO22 |
| RS485 RX | GPIO21 |
| MAX13487E receive/autodirection enable | GPIO17 |
| MAX13487E shutdown disable | GPIO19 |
| RS485/CAN 5 V boost enable | GPIO16 |
| WS2812B status LED | GPIO4 |

GPIO16, GPIO19, and GPIO17 must be driven physically HIGH. The supplied YAML uses non-inverted GPIO outputs and explicitly turns all three outputs on during boot.

### Inverter Connection

Connect the LILYGO board to the inverter's documented RS485/COM2 port:

<img width="221" height="159" alt="pastedImage" src="https://github.com/user-attachments/assets/8ac42cc3-adb1-4432-aa94-3d49ba18e9f6" />

<img width="507" height="105" alt="pastedImage" src="https://github.com/user-attachments/assets/27014048-797d-4ef2-96d4-0bd20abb5ce1" />


```text
LILYGO RS485 A  -> inverter RS485 A
LILYGO RS485 B  -> inverter RS485 B
LILYGO GND      -> inverter communication GND
```

Use a twisted pair for A/B. Keep the cable away from mains and high-current conductors. If no communication is received, verify the RJ45 pin orientation with a continuity tester before swapping signals. I took a standard RJ45 patch cable and connected it to the Trina Nexeos inverter. The other side I cut off and inserted the stripped of cable strands into the LILIGO connectors. 

Do not connect the RS485 wires to the board's CAN terminals.

## Modbus Configuration

The tested configuration is:

```text
Protocol:       Modbus RTU
Baud rate:      9600
Data format:    8 data bits, no parity, 1 stop bit (8N1)
Slave address:  3
Input read:     Function code 0x04
Holding read:   Function code 0x03
```

The Nexeos register notation must be converted for ESPHome:

1. Remove the `3x` or `4x` prefix.
2. Subtract one.

Examples:

```text
31002 -> 1002 - 1 -> ESPHome address 1001
31622 -> 1622 - 1 -> ESPHome address 1621
41154 -> 1154 - 1 -> ESPHome address 1153
```

## Flashing the ESP32 and Adding It to Home Assistant

### 1. Install ESPHome Device Builder

In Home Assistant, install and start the ESPHome Device Builder, then open its web interface.

### 2. Copy the Configuration

Copy `esphome/nexeos-tcan485.yaml` into the ESPHome configuration directory.

Copy the keys from `esphome/secrets.example.yaml` into your local ESPHome `secrets.yaml` and replace every placeholder with your own values.

Never commit your real `secrets.yaml`, Wi-Fi password, fallback-hotspot password, API encryption key, or OTA password to GitHub.

### 3. Validate the YAML

Open the device in ESPHome and select **Validate**. Correct any version-specific warnings before flashing.

### 4. Perform the First Installation over USB

1. Connect the T-CAN485 board with a USB-C data cable.
2. Select **Install** in ESPHome.
3. Choose the locally connected USB/serial device.
4. If the board does not enter flashing mode, hold **BOOT**, briefly press **RESET**, and retry.

The first installation normally requires USB. Later updates can be installed over Wi-Fi using OTA.

### 5. Add the Device to Home Assistant

After the board connects to Wi-Fi, Home Assistant should discover the ESPHome device. If it does not, add the ESPHome integration manually and enter the device hostname or IP address.

The default hostname in this project is:

```text
nexeos-tcan485.local
```

## Configuration Overview

The main configuration is available at [`esphome/nexeos-tcan485.yaml`](esphome/nexeos-tcan485.yaml).

### Polling Groups

The YAML is organized into two polling groups:

- **Frequent values, every 10 seconds:** live AC power, smart-meter power, PV voltage/current/power, battery voltage/current/power, and warning code.
- **Slow values, every 60 seconds:** state codes, temperatures, SOC, SOH, current limits, daily energy values, and diagnostic values.

The Modbus controller runs every 10 seconds. Slow values use `skip_updates: 5`, producing one read every sixth controller cycle.

### Main Entities

The names below appear in German in Home Assistant. The English translation is shown after each name.

#### AC and Grid

- **Netzfrequenz** — Grid frequency
- **AC-Wirkleistung** — AC active power
- **Netzleistung roh** — Raw grid power
- **Netzbezug Leistung** — Grid import power
- **Netzeinspeisung Leistung** — Grid export power
- **Netzbezug heute** — Grid import energy today
- **Netzeinspeisung heute** — Grid export energy today
- **Netzbezug gesamt** — Total grid import energy
- **Netzeinspeisung gesamt** — Total grid export energy
- **Netzverbindung** — Grid connection status

#### PV

- **PV1 Spannung / Strom** — PV1 voltage / current
- **PV2 Spannung / Strom** — PV2 voltage / current
- **PV-Leistung** — PV power
- **PV-Ertrag heute** — PV energy today
- **PV-Ertrag gesamt** — Total PV energy

#### Battery

- **Batteriespannung** — Battery voltage
- **Batteriestrom** — Battery current
- **Batterieleistung** — Battery power
- **Batterietemperatur** — Battery temperature
- **Batterie SOC** — Battery state of charge
- **Batterie SOH** — Battery state of health
- **Ladestromlimit** — Charge current limit
- **Entladestromlimit** — Discharge current limit
- **Batterieladung heute** — Battery charge energy today
- **Batterieentladung heute** — Battery discharge energy today
- **Batteriestatus** — Battery status

#### Diagnostics

- **Modbus-Adresse** — Modbus address
- **Gerätestatus** — Device status
- **Fehlercode** — Error code
- **Warncode** — Warning code
- **Batteriekommunikation Code** — Battery communication code
- **Smart Meter Status Code** — Smart meter status code

### Grid Energy Calculation

The live grid power is read from the inverter's smart-meter power register. Import and export are split by sign and integrated locally in ESPHome:

```text
positive smart-meter power -> grid import
negative smart-meter power -> grid export
```

Verify the sign convention on your installation. If import and export are reversed, swap the two template lambdas in the YAML.

The locally calculated lifetime counters begin when this firmware is installed. They are not the inverter's historical totals.

## Known Limitations and Open Work

### Native Grid-Energy Counters

The inverter app displays grid import and export energy, but the investigated documented Modbus registers currently return zero on the tested inverter:

- AC-side daily consumption and generation registers;
- native daily and total grid-charge registers;
- smart-meter import/export registers, tested with both 32-bit word orders.

The current workaround integrates the working live smart-meter power locally in ESPHome. Further research is needed to determine whether the app uses undocumented registers, cloud-side integration, or a firmware-specific register map.

The most reliable solution is to read the utility meter directly through its optical interface, where available. The utility meter is the authoritative source for billed grid import and credited grid export.

### Validation Still Needed

- Verify the smart-meter power sign convention during actual export.
- Decode detailed error and warning registers into human-readable text.
- Compare locally calculated daily grid energy with the inverter app over complete days.
- Validate the configuration on additional Nexeos models and firmware revisions.

## Troubleshooting

### No Modbus Response

Check all of the following:

- GPIO22 is TX and GPIO21 is RX.
- GPIO16, GPIO19, and GPIO17 are physically HIGH.
- The baud rate is 9600.
- The data format is 8N1.
- The slave address is 3.
- The board is connected to RS485, not CAN.
- RJ45 A, B, and GND are mapped correctly.
- The Smart Meter status should normally report `10` when online.

### SOC or SOH Is Off by a Factor of 100

The tested inverter returns SOC and SOH directly as whole percentages. The supplied YAML therefore does not apply a `0.01` multiplier to those two read-only sensors.

### SOC Limits Show 8500 or 2000

The writable SOC limits use a raw scale of 100. The YAML converts these values to 85% and 20% for Home Assistant and multiplies by 100 before writing.

### A Signed Value Shows -32768

For a signed 16-bit register, `-32768` is `0x8000`, which the profile uses as an invalid or unavailable value. It is not a real power setpoint.

## Contributing

Contributions are welcome, especially:

- native grid-energy register findings;
- results from other Nexeos models and firmware versions;
- pull requests improving entity naming, scaling, YAML organization, and documentation.

When reporting a result, please include the inverter model, firmware version, ESPHome version, expected value from the inverter app, and the corresponding sanitized Modbus value or log output.

## License

This project is licensed under the MIT License. See [`LICENSE`](LICENSE).
