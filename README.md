# two_i2c_sensor_ardunio_slave_verilog

# Dual I2C Master — FPGA to Two Arduino Slaves (Parallel Operation)

## Overview

This project implements **two independent I2C master state machines** on an Intel Cyclone 10 LP FPGA, each driving its own I2C bus to a separate Arduino Uno acting as a slave. Both masters operate **in parallel**, triggered by a shared 1-second timer, but each has its own control FSM, its own bus (SDA/SCL pair), and its own data source.

- **I2C Master 1** sends an 8-bit free-running counter (increments by 1 each successful transfer).
- **I2C Master 2** sends the next value from a 10-entry lookup array (`data_array[0:9]`, values 1–10), cycling and wrapping back to index 0 after the last entry.

Both masters share a single custom `i2c_master` module (instantiated twice) and a single 1-second interval timer derived from a PLL-generated 4 MHz internal clock.

## Hardware

| Signal      | Description                          |
|-------------|---------------------------------------|
| `clk`       | 25 MHz board input clock              |
| `rst_n`     | Active-low external reset button      |
| `sda1/scl1` | I2C bus 1 → Arduino slave #1          |
| `sda2/scl2` | I2C bus 2 → Arduino slave #2          |
| `led1_out`  | Toggles on each transaction attempt, bus 1 |
| `led2_out`  | Toggles on each transaction attempt, bus 2 |

Both Arduinos are configured as I2C slaves at address `0x08` and simply print the received byte.

## Architecture

```
                ┌────────────┐
   25 MHz ──────►    PLL     ├──► clk_4m (4 MHz)
                └────────────┘
                       │
              ┌────────┴────────┐
              │  1-second timer │
              └────────┬────────┘
                        │ one_second_flag
          ┌─────────────┴─────────────┐
          ▼                           ▼
   FSM 1 (counter)             FSM 2 (array index)
          │                           │
   i2c_master #1               i2c_master #2
          │                           │
    sda1/scl1 ──► Arduino #1    sda2/scl2 ──► Arduino #2
```

Each `i2c_master` instance implements a full I2C write transaction (START → address+W → ACK → data byte → ACK → STOP) using a 4x-oversampled internal pulse generator (100 kHz I2C clock derived from the 4 MHz system clock).

## Status: Work in Progress

This design is **not yet synthesizable**. Known issue:

- **Duplicate driver on `state` register.** The top module currently declares a single `state` register but has two separate `always` blocks (one per FSM) both driving it with two different state encodings (`IDLE1/...` vs `IDLE2/...`). This causes a Quartus elaboration error (`Error (10028): Can't resolve multiple constant drivers`). Fix in progress: splitting into `state1` and `state2`.
- Minor implicit-net warnings on `OP_WRITE` (should be `OP_WRITE1`) and `done` (undeclared at top level) also need cleanup before this is called "done."

This section will be updated once the design compiles and has been verified on hardware.

## Repository Structure

```
├── top.v          -- top-level module: timer, two FSMs, two I2C master instances
├── pll.v           -- PLL wrapper (Cyclone 10 LP)
├── testbench/      -- (add simulation testbenches here)
└── README.md
```

## Next Steps

- Fix the `state` register conflict (split to `state1`/`state2`)
- Resolve implicit net warnings
- Simulate both FSMs in Icarus/ModelSim to confirm correct interleaving
- Verify on hardware with logic analyzer capture of both I2C buses
- Add SVA/functional coverage for both masters running concurrently
