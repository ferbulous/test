external_components:
  - source: github://EspMeshMesh/esphome-meshmesh@main
    components: [ meshmesh, meshmesh_direct, network, socket, esphome ]

preferences:
    flash_write_interval: 30sec

substitutions:
  # remote_node_addr: 0x7559BE
  remote_node_addr: 0x4A5728 #for esp32s3 dev board
  remote_node_addr2: 0xDCA2E8 #for esp32s3 zero
  # remote_node_addr: 0x57B02A #for esp8266
  # remote_switch_hash: 0x0642 #for esp8266

esphome:
  name: testmesh2

esp32:
  board: esp32dev
  framework:
    type: esp-idf

logger:
  level: VERY_VERBOSE
  baud_rate: 115200

api:
  reboot_timeout: 0s

mdns:
  disabled: True

socket:
  implementation: meshmesh_esp32

meshmesh:
  baud_rate: 0
  rx_buffer_size: 0
  tx_buffer_size: 0
  password: !secret meshmesh_password
  channel: 3
  use_starpath: True

meshmesh_direct:
  id: meshmesh_direct_id
  # on_receive:
  #   then:
  #     - lambda: |-
  #           ESP_LOGD("lambda", "on_receive %06X data %s", from,  format_hex_pretty(data, size).c_str());

ota:
  platform: esphome
######
globals:
  - id: counter
    type: int
    restore_value: False
    initial_value: "0"

  - id: bool_dim_or_bright #false = dim, true = brighten
    type: bool
    restore_value: no
    initial_value: "false"  
########

output:
  - platform: template
    id: dummy_output
    type: float
    write_action:
      - lambda: return;

light:
  - platform: rgbww
    id: internal_light
    color_interlock: true
    cold_white_color_temperature: 6500 K
    warm_white_color_temperature: 2700 K
    red: dummy_output
    green: dummy_output
    blue: dummy_output
    cold_white: dummy_output
    warm_white: dummy_output
    on_state:
      then:
        - lambda: |-
            auto state = id(internal_light).current_values;

            bool is_on = state.is_on();

            float r = state.get_red();
            float g = state.get_green();
            float b = state.get_blue();

            float brightness = state.get_brightness();
            float ct = state.get_color_temperature();

            uint8_t power = is_on ? 1 : 0;
            uint8_t red   = (uint8_t)(r * 255.0f);
            uint8_t green = (uint8_t)(g * 255.0f);
            uint8_t blue  = (uint8_t)(b * 255.0f);
            uint8_t bri   = (uint8_t)(brightness * 255.0f);

            uint16_t ct_val = (uint16_t)ct;

            const uint8_t data[] = {
              power,
              red,
              green,
              blue,
              bri,
              (uint8_t)(ct_val >> 8),
              (uint8_t)(ct_val & 0xFF)
            };

            id(meshmesh_direct_id)->unicastSendCustom(data, 7, ${remote_node_addr});


binary_sensor:

#for button
  - platform: gpio
    pin:
      number: GPIO16
      mode: INPUT_PULLUP
      inverted: True
    name: Button 2
    id: button2
    on_multi_click:
      # single click
      - timing:
          - ON for at most 1s
          - OFF for at least 0.5s
        then:
          - light.toggle: internal_light

      # double click to switch to red, green, blue
      - timing:
          - ON for at most 1s
          - OFF for at most 1s
          - ON for at most 1s
          - OFF for at least 0.2s
        then:
          - lambda: |-
              auto call = id(internal_light).turn_on();

              if (id(counter) == 0) {   // RED
                call.set_rgb(1.0, 0.0, 0.0);
              }

              else if (id(counter) == 1) {  // GREEN
                call.set_rgb(0.0, 1.0, 0.0);
              }

              else if (id(counter) == 2) {  // BLUE
                call.set_rgb(0.0, 0.0, 1.0);
              }

              call.set_brightness(1.0);

              // Cycle counter 0 → 1 → 2 → 0
              if (id(counter) < 2) {
                id(counter) += 1;
              } else {
                id(counter) = 0;
              }

              call.perform();

    #long press for dimming
    on_press:
      then:
        - if:
            condition:
              lambda: |-
                return id(bool_dim_or_bright);
            # When above condition evaluates to true - brighter function else dimmer
            then:
              - delay: 0.5s
              - while:
                  condition:
                    binary_sensor.is_on: button2
                  then:
                    - light.dim_relative:
                        id: internal_light
                        relative_brightness: 5%
                        transition_length: 0.1s
                    - delay: 0.1s
              - lambda: |-
                  id(bool_dim_or_bright) = (false);
            else:
              - delay: 0.5s
              - while:
                  condition:
                    and:
                      - binary_sensor.is_on: button2
                  then:
                    - light.dim_relative:
                        id: internal_light
                        relative_brightness: -5%
                        transition_length: 0.1s
                    - delay: 0.1s
              - lambda: |-
                  id(bool_dim_or_bright) = (true);


#####for basic 2 way switching
# switch:
#   - platform: gpio
#     pin: 15
#     id: my_switch
#     name: "Blue LEDS"
#   - platform: meshmesh_direct
#     id: remote_switch
#     target: ${remote_switch_hash}
#     address: ${remote_node_addr}

# binary_sensor:
#   - platform: gpio
#     pin:
#       number: GPIO16
#       mode: INPUT_PULLUP
#       inverted: True
#     name: "Control"
#     on_press:
#       then:
#         - lambda: id(remote_switch)->toggle();

# interval:
#   - interval: 20sec
#     then:
#       - logger.log: "interval 1"
#       - switch.toggle: my_switch
#       - lambda: id(remote_switch)->toggle();
#   - interval: 20sec
#     startup_delay: 10sec
#     then:
#       - logger.log: "interval 2"
#       - switch.toggle: my_switch
#       - lambda: id(remote_switch)->toggle();
