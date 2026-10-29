# ESPHome RIKA GSM emulator

This fork documents an ESPHome setup that emulates the GSM module used
by a compatible RIKA pellet stove. It allows the stove to be controlled
through its serial interface and exposes the status messages sent by the
stove to Home Assistant.

Tested setup: - **Stove:** RIKA Como, first generation - **Controller:**
ESP32-C3 Super Mini - **Serial adapter:** MAX3232 RS232-to-TTL -
**Serial speed:** 38400 baud - **Framework:** ESPHome / ESP-IDF

This is a community project, not an official RIKA product. Only the
setup described above has been tested; other stove models and revisions
may differ.

## Features

-   Emulates the stove's GSM/SMS exchange through the `rika_gsm` ESPHome
    component.
-   Sends commands through `rika_gsm.send_command`.
-   Exposes the latest stove status as a text sensor.
-   Exposes a binary sensor for the GSM option status.
-   Provides a Home Assistant `climate` entity for on/off and target
    temperature.
-   Parses status messages such as
    `STOVE ON - ROOM 21 - PLEASE SAVE TEL NUMBER`.

`ROOM XX` is interpreted as the **target temperature**, not as a
measured room temperature.

## Hardware

The GSM module shares the DA15 connector at the rear of the stove with
the ambient temperature sensor. Use a suitable split cable or connector.
**Do not disconnect or disturb the ambient-temperature probe** used by
the stove for room-temperature regulation.

The original diagram is retained:

`<img src="schema/esp_rika_stove.png" alt="Original RIKA stove wiring diagram">`{=html}

> **Important:** the level shifter drawn in the original diagram is not
> the adapter used in this installation. This setup uses a MAX3232
> RS232-to-TTL transceiver: the RS232 side connects to the stove and the
> TTL side to the ESP32-C3. Check the pin labels and signal directions
> on your exact module; arrow markings and board layouts can differ.

### Wiring used in the tested setup

  -----------------------------------------------------------------------
  Connection                          Destination
  ----------------------------------- -----------------------------------
  Stove DA15 pin 1 (+5 V)             ESP32-C3 `5V` input

  Stove DA15 pin 5 (GND)              ESP32-C3 `GND`

  ESP32-C3 GPIO0 (UART TX)            MAX3232 TTL-side input (`IN`,
                                      toward the adapter)

  ESP32-C3 GPIO1 (UART RX)            MAX3232 TTL-side output (`OUT`,
                                      from the adapter)

  Stove DA15 pin 2 (TX in this setup) MAX3232 RS232-side input (`IN`,
                                      toward the adapter)

  Stove DA15 pin 3 (RX in this setup) MAX3232 RS232-side output (`OUT`,
                                      from the adapter)

  MAX3232 GND                         Common GND
  -----------------------------------------------------------------------

These are the connections that worked in this installation. Verify the
pinout and direction for your own stove and adapter before powering the
circuit. Do not connect RS232 signals directly to ESP32 GPIOs.

### MAX3232 power and ground

The tested adapter was powered at **3.3 V**. Check the specification of
your exact board before powering it: some adapter boards use different
transceivers or supply arrangements.

On the tested board, the positive supply rails on both sides were
electrically common. Therefore, connect only one appropriate supply
point; **do not connect 3.3 V and 5 V to separate `+`/`VCC` terminals if
they are tied together**.

A single ESP32-C3 GND pin is sufficient. The stove, ESP32-C3 and MAX3232
must share a common ground reference.

## ESPHome configuration

### External component

This fork stores the component locally:

``` yaml
external_components:
  - source: esphome-rika-gsm/components
```

### UART and component

The tested ESP32-C3 Super Mini uses GPIO0 for TX and GPIO1 for RX:

``` yaml
uart:
  id: rika_uart_bus
  tx_pin: GPIO0
  rx_pin: GPIO1
  baud_rate: 38400
  debug:
    direction: BOTH
    dummy_receiver: false
    after:
      timeout: 100ms
    sequence:
      - lambda: UARTDebug::log_hex(
          direction, bytes, 16, esphome::StringRef());

rika_gsm:
  id: ria_gsm_component_1
  pin: !secret rika_gsm_pin
  phone_number: !secret rika_gsm_phone_number
```

Configuration options: - `pin` (string): the four-digit PIN configured
for the stove's GSM option. - `id` (optional): component ID. -
`phone_number` (optional string): number value used during the emulated
SMS exchange. This component does not send real SMS messages through a
cellular network. Use secrets or a dummy value; do not commit a personal
phone number.

### Text sensor and binary sensor

``` yaml
text_sensor:
  - platform: rika_gsm
    raw_status:
      name: "Status poêle"
      id: rika_status

binary_sensor:
  - platform: rika_gsm
    gsm_status:
      name: "Status GSM"
```

-   `raw_status` exposes the latest status/SMS text received from the
    stove.
-   `gsm_status` indicates whether the stove currently considers the GSM
    option active.

In the tested setup, the stove polls for commands approximately every
five seconds. If no request arrives for around ten seconds, the GSM
status may become inactive until the next stove request. Timing may
vary.

### Climate entity

The climate entity sends `rXX` to set the target temperature and `OFF`
to stop the stove:

``` yaml
climate:
  - platform: template
    id: rika_climate
    name: "Poêle RIKA"
    icon: mdi:stove
    optimistic: true
    restore_mode: RESTORE
    initial_state:
      mode: "OFF"
      target_temperature: 19 °C
    supported_modes:
      - "OFF"
      - HEAT
    visual:
      min_temperature: 15 °C
      max_temperature: 30 °C
      temperature_step: 1 °C
    set_target_temperature_action:
      - rika_gsm.send_command:
          command: !lambda |-
            return "r" + std::to_string((int) x);
      - climate.template.publish:
          id: rika_climate
          mode: HEAT
          target_temperature: !lambda "return x;"
    set_mode_action:
      - if:
          condition:
            lambda: 'return x == climate::CLIMATE_MODE_OFF;'
          then:
            - rika_gsm.send_command:
                command: "OFF"
          else:
            - rika_gsm.send_command:
                command: !lambda |-
                  return "r" + std::to_string(
                    (int) id(rika_climate).target_temperature
                  );
```

Add the following automation to the `raw_status` text sensor to
synchronize the climate entity from received messages:

``` yaml
text_sensor:
  - platform: rika_gsm
    raw_status:
      name: "Status poêle"
      id: rika_status
      on_value:
        then:
          - lambda: |-
              std::string s = x;

              if (s.find("STOVE OFF") != std::string::npos) {
                id(rika_climate).set_mode(
                  climate::CLIMATE_MODE_OFF);
                id(rika_climate).publish_state();
                return;
              }

              auto pos = s.find("ROOM ");
              if (s.find("STOVE ON") != std::string::npos &&
                  pos != std::string::npos) {
                int temp = atoi(s.c_str() + pos + 5);

                if (temp >= 15 && temp <= 30) {
                  id(rika_climate).set_mode(
                    climate::CLIMATE_MODE_HEAT);
                  id(rika_climate).set_target_temperature(temp);
                  id(rika_climate).publish_state();
                }
              }
```

Messages handled: - `STOVE ON - ROOM 16 - PLEASE SAVE TEL NUMBER` → mode
`HEAT`, target 16 °C. - `STOVE ON - ROOM 21 - PLEASE SAVE TEL NUMBER` →
mode `HEAT`, target 21 °C. - `STOVE OFF - PLEASE SAVE TEL NUMBER` → mode
`OFF`.

The stove does not send a spontaneous status SMS whenever its setpoint
is changed locally. The Home Assistant climate entity can therefore show
an outdated target until a later status message arrives.

### API actions

Optional API actions for other ESPHome integrations or automations:

``` yaml
api:
  encryption:
    key: !secret api_encryption_key
  actions:
    - action: turn_on
      then:
        - rika_gsm.send_command:
            command: "ON"
    - action: turn_off
      then:
        - rika_gsm.send_command:
            command: "OFF"
    - action: send_command
      variables:
        command: string
      then:
        - rika_gsm.send_command:
            command: !lambda "return command;"
```

A command can also be sent from an ESPHome automation:

``` yaml
on_...:
  then:
    - rika_gsm.send_command:
        command: "r21"
```

`command` is required, accepts a string and is templatable.

## Logging and troubleshooting

The tested configuration uses:

``` yaml
logger:
  level: DEBUG
  baud_rate: 0
```

The logger is disabled on the UART hardware so it does not interfere
with the stove serial link. The UART debug block logs both directions in
hexadecimal and is useful for diagnosis. It can be removed if quieter
logs are preferred.

-   **No UART traffic:** check power, common GND, TX/RX direction, the
    MAX3232 pinout and the 38400 baud rate.
-   **Only one direction works:** recheck both sides of the transceiver.
    Do not rely on arrow markings alone.
-   **GSM status becomes inactive:** the stove may have stopped polling
    temporarily; check whether it becomes active again on the next
    request.
-   **Climate temperature does not reflect a local adjustment:** the
    stove does not appear to send a status SMS for every local setpoint
    change.
-   **Compile errors:** check the ESPHome version and compatibility of
    the external component.

## Known limitations

-   Tested only on one RIKA Como, first generation.
-   Other models and serial speeds are unverified.
-   The climate state is updated from received status messages; local
    changes may not be reported immediately.
-   The component emulates the GSM/SMS exchange and does not provide
    real cellular connectivity.

## Credits

The original component and initial wiring documentation are from
[n-serrette/esphome_rika_gsm](https://github.com/n-serrette/esphome_rika_gsm).
This fork documents the ESP32-C3 Super Mini and MAX3232 setup tested
with a RIKA Como first-generation stove.
