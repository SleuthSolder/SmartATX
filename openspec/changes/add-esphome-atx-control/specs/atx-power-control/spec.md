## ADDED Requirements

### Requirement: PSU Switch Entity
The device SHALL expose the ATX power supply to Home Assistant as a single switch entity
over the native ESPHome API. No cloud service, no local emulation of a third-party device
protocol, and no HTTP/SSDP discovery path SHALL be used for control.

#### Scenario: Turning the supply on from Home Assistant
- **WHEN** the switch entity is turned on in Home Assistant
- **THEN** the device drives the control GPIO low
- **AND** the ATX supply's main rails come up (3.3 V orange, 5 V red, 12 V yellow)
- **AND** the switch entity reports state `on`

#### Scenario: Turning the supply off from Home Assistant
- **WHEN** the switch entity is turned off in Home Assistant
- **THEN** the device releases the control GPIO to a high-impedance state
- **AND** the ATX supply's main rails drop to 0 V
- **AND** the switch entity reports state `off`

#### Scenario: No Alexa emulation remains
- **WHEN** the firmware is inspected or the device is scanned on the LAN
- **THEN** no Hue-bulb emulation, SSDP responder, or FauxmoESP code is present

### Requirement: Fail-Safe Open-Drain Control
The control GPIO SHALL be configured as an open-drain output. Asserting ON MUST drive the
pin low; asserting OFF MUST release the pin to high impedance and let the supply's internal
pull-up hold `PS_ON` high. The firmware MUST NOT actively drive `PS_ON` high at any time.

#### Scenario: Supply stays off through a device reboot
- **WHEN** the ESP32 resets, is reflashed, or loses and regains power while the supply is on
- **THEN** the control GPIO returns to high impedance during boot
- **AND** the ATX supply turns off and stays off until commanded on again

#### Scenario: Supply stays off when WiFi or Home Assistant is unavailable
- **WHEN** the device boots with no WiFi network or no Home Assistant reachable
- **THEN** the supply remains off
- **AND** the device continues retrying its connection without changing the supply state

#### Scenario: Boot state is always off
- **WHEN** the device completes boot after any restart cause
- **THEN** the switch entity's restored state is `off` regardless of its state before the restart

### Requirement: Unchanged Physical Wiring
The configuration SHALL operate with the 2020 build's wiring unmodified: ESP32 `Vin` from
`+5VSB` (ATX pin 9, violet), `GND` from ATX pin 19 (black), and GPIO18 to `PS_ON`
(ATX pin 16, green).

#### Scenario: Existing unit is reflashed without rework
- **WHEN** the ESPHome firmware is flashed onto the existing hardware
- **THEN** it controls the supply correctly with no changes to the wiring

### Requirement: Control Pin Voltage Verified Against Device Rating
The `PS_ON` idle voltage SHALL be measured on the actual supply and recorded. Because
ESP32 GPIOs are not 5 V tolerant, the measured value MUST be reported in the published
documentation together with its implication, rather than presented as a verified-safe
design. Where the measurement exceeds the ESP32's rated input, at least one in-spec
alternative MUST be documented.

#### Scenario: Idle voltage is measured before publication
- **WHEN** the supply is connected to mains and in standby, with the ESP32 disconnected from PS_ON
- **THEN** the voltage on the green `PS_ON` wire is measured with a meter
- **AND** the measured value is recorded in the documentation, not estimated or assumed

#### Scenario: Measurement exceeds the ESP32 input rating
- **WHEN** the measured `PS_ON` idle voltage is above the ESP32's rated maximum input
- **THEN** the documentation states plainly that the as-built wiring is out of spec
- **AND** documents at least one in-spec alternative (series resistor or MOSFET buffer)
