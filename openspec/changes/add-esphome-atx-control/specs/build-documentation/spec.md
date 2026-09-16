## ADDED Requirements

### Requirement: Reproducible Build Instructions
The repository SHALL document the build well enough that a reader with a scrap ATX supply
and an ESP32 can reproduce it without consulting the original 2020 post. Documentation MUST
cover the wiring, the `+5VSB` rationale, flashing, and Home Assistant setup.

#### Scenario: A stranger reproduces the build
- **WHEN** a reader follows the repository documentation from the top
- **THEN** they can identify the three ATX connections, flash the firmware, and see a working
  switch entity in Home Assistant
- **AND** no step requires information found only in the original blog post

#### Scenario: Wiring is stated as a table with colours and pin numbers
- **WHEN** a reader looks for the connections
- **THEN** they find ESP32 pin, ATX pin number, and wire colour for all three connections

### Requirement: Documented Safety Section
The documentation SHALL include a safety section covering the hazards of the 12 V rail's
current capability and of shorting rails during bench work. It MUST state hazards plainly
rather than omitting or minimising them.

#### Scenario: Safety is addressed before wiring instructions
- **WHEN** a reader reaches the wiring steps
- **THEN** they have already been shown the safety section

### Requirement: No Unverified Figures
Every published electrical figure MUST be measured, cited to a datasheet or specification,
or explicitly marked as unverified. This SHALL apply to currents, voltages, load capacities,
and board identification alike. Figures MUST NOT be estimated and then presented as measured.

#### Scenario: An unmeasured figure is requested
- **WHEN** documentation would state a per-rail current or load figure that has not been measured
- **THEN** the figure is either measured first, or marked as not yet verified
- **AND** it is never presented as a confirmed value

#### Scenario: Board identification is presented with its basis
- **WHEN** the documentation names the ESP32 board
- **THEN** it states that the identification is inferred, links the supporting evidence,
  and gives the confidence level

### Requirement: Commented Configuration
The ESPHome YAML SHALL carry inline comments explaining the non-obvious choices — at minimum
the open-drain control mode, the always-off boot state, and the `+5VSB` power arrangement —
because the configuration is published as teaching material.

#### Scenario: A reader adapts the config to another board
- **WHEN** a reader opens the YAML intending to change the board or control pin
- **THEN** comments identify which values are hardware-specific and what constraints apply to them

### Requirement: Credentials Excluded From Version Control
Secrets SHALL live in an ignored `secrets.yaml`. The repository MUST ship a committed example
file and MUST NOT contain real network credentials or API keys in any commit.

#### Scenario: Repository is prepared for publication
- **WHEN** the repository is reviewed before being made public
- **THEN** no WiFi credentials, API keys, or OTA passwords appear in tracked files or history
- **AND** a `secrets.yaml.example` documents every key the configuration requires

### Requirement: Toolchain Setup Record
The repository SHALL keep a step-by-step record of installing the ESPHome toolchain and
building the firmware, suitable for a reader reproducing it and for use as article source
material. Commands and their output MUST be transcribed from runs that actually happened,
and pitfalls encountered MUST be recorded rather than smoothed over.

#### Scenario: A reader installs the toolchain from scratch
- **WHEN** a reader follows the setup guide on a clean machine
- **THEN** they can install ESPHome, validate the config, and produce a flashable binary
- **AND** each step shows the command and the output it produces

#### Scenario: A pitfall is encountered during setup
- **WHEN** a step fails, produces an unexpected result, or requires a non-obvious fix
- **THEN** the guide records what happened, why, and the fix
- **AND** the record is not rewritten to imply the step succeeded first time

#### Scenario: Steps that need a screen capture are identified
- **WHEN** a step cannot be conveyed by a command transcript, such as a GUI interaction
- **THEN** the guide names the screenshot required and what it should show
