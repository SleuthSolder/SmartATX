## ADDED Requirements

### Requirement: Local State Indicator
The onboard LED SHALL mirror the ATX supply's commanded state, giving an at-a-glance
indication at the bench without Home Assistant.

#### Scenario: LED follows the supply
- **WHEN** the supply is switched on
- **THEN** the onboard LED is lit
- **WHEN** the supply is switched off
- **THEN** the onboard LED is off

### Requirement: Health Telemetry
The device SHALL publish diagnostic entities to Home Assistant: uptime, WiFi signal
strength, and free heap. These entities MUST be marked as diagnostic so they do not clutter
the primary device controls.

#### Scenario: Diagnostics available in Home Assistant
- **WHEN** the device is added to Home Assistant
- **THEN** uptime, WiFi signal, and free-heap entities appear under the device's diagnostic section
- **AND** they update without user interaction

#### Scenario: Diagnostics never affect the supply
- **WHEN** any diagnostic entity is read or updated
- **THEN** the ATX supply's state is unchanged

### Requirement: Remote Restart
The device SHALL expose a restart button so the ESP32 can be rebooted from Home Assistant
without physical access.

#### Scenario: Restart leaves the supply off
- **WHEN** the restart button is pressed while the supply is on
- **THEN** the ESP32 reboots
- **AND** the supply turns off and remains off after the reboot completes

### Requirement: Wireless Update Path
The device SHALL support over-the-air updates, so firmware can be revised while the ESP32
remains wired into the supply.

#### Scenario: OTA update of a wired-in device
- **WHEN** a new build is pushed over the air
- **THEN** the device updates and reconnects without being physically removed or re-cabled
