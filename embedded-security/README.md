# Embedded Security

Status: Planned.

Security and architecture analyses of embedded systems, telemetry
pipelines, and the boundaries between physical, network, and application
layers. Each report in `reports/<target>/` will be self-contained.

## Candidate targets

- Lunabotics rover server protocol and telemetry link (related repository:
  `Lunabotics-ServerDev`)
- Consumer IoT device on hand (router, smart plug, BLE peripheral)
- Microcontroller-based system (firmware UART/JTAG attack surface)
- CAN bus or OBD-II protocol analysis

## Index

No reports published yet.

## Scope notes

Embedded analyses focus on the physical-to-network-to-application boundary,
which assumptions break when an attacker has hardware access, and the
end-to-end attack surface including the supply chain.
