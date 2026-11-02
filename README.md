# Replicated ESP32 Circuit (REVIVE-TERY)

A battery monitoring dashboard with a **software replica of the ESP32 bench circuit**, so the whole system runs with no hardware. It reads voltage, current and temperature, calculates State of Charge (SOC) and State of Health (SOH), and screens a battery for a second-life use. The same code reads a real ESP32 over USB.

> **Read this first.** The bundled battery data is **synthetic**, and the simulator models a **made-up cell**. Neither describes a real battery, and nothing here is fit for engineering, safety, warranty or compliance decisions. The real-hardware path has been tested only against a simulated serial port. The firmware compiles for an ESP32, but it has not yet run on a physical board.

## Quick start (no hardware)

```bash
python -m pip install -r src/requirements.txt
python src/app.py
```

Open http://127.0.0.1:8050 and choose **Simulated ESP32 (no hardware needed)**, then **Start test**. Tested on Python 3.15.

With the defaults (a 1.8 Ah cell rated 2.0 Ah, 1 A load, 60× speed), a test takes about two minutes and measures an SOH of about 88%, which gives **Stationary Storage**. Try a true capacity of 1.0 Ah for about 49% and **Recycling**.

## What's in the dashboard

- **Circuit Studio:** the bench circuit with every pin and wire. Tap a wire to see what it connects. Tap the battery to reload it from DuckDB (the database holds one battery, so it reloads that one).
- **Live gauges:** a voltmeter and a thermometer that update during a test, plus voltage and SOC traces.
- **Three sources:** a virtual battery that replays recorded telemetry from DuckDB, a simulated ESP32, and a real ESP32 on a COM port.
- **Decision:** the secondary-use tier, the SOH it rests on, and where that SOH came from.
- **Fault alerts:** the firmware's own messages appear at the top of the page. Gauges go blank when readings stop, so a frozen dial can't pass for a steady reading.

The page never reloads and never changes URL.

## The simulator

`src/esp32_sim.py` replaces the serial port. It models a 1S Li-ion cell, a constant-current test load, an INA219 and a DS18B20, and makes the same decisions as the firmware: the same JSON lines, `#` messages, limits and retry timing. It runs through the real hardware code path, so SOC and SOH, fault handling and logging are all exercised.

You can inject a fault at any time during a test: a loose INA219 wire, a disconnected DS18B20, +22 V on the sensor inputs, a 3.5 A load, or the ESP32 going silent. Clear it and readings resume.

Simulated readings are labelled **SIMULATED** on the page, in the result, and in the database, and are never stored as measured data.

## Using a real ESP32

**Wiring** (low voltage only):

| From | To |
|---|---|
| Battery + | INA219 VIN+ |
| INA219 VIN− | Test load + |
| Test load − | Battery − |
| Battery − | ESP32 GND (common ground) |
| INA219 VCC / GND | ESP32 3V3 / GND |
| INA219 SDA / SCL | ESP32 GPIO21 / GPIO22 |
| DS18B20 VDD / GND | ESP32 3V3 / GND |
| DS18B20 DQ | ESP32 GPIO4, with a 4.7 kΩ pull-up to 3V3 |

**Flash** with [arduino-cli](https://arduino.github.io/arduino-cli/):

```bash
arduino-cli core update-index --additional-urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
arduino-cli core install esp32:esp32 --additional-urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
arduino-cli lib install "Adafruit INA219" OneWire DallasTemperature
arduino-cli compile --fqbn esp32:esp32:esp32 firmware/revive_tery_esp32
arduino-cli board list
arduino-cli upload -p COMx --fqbn esp32:esp32:esp32 firmware/revive_tery_esp32
```

Use the port that `board list` shows for your ESP32 in place of `COMx`. Windows also lists Bluetooth ports, which are never the board. Then choose **ESP32 hardware** in the dashboard, pick that port, and enter the rated capacity, starting SOC and cutoff voltage. The dashboard holds the port during a test, so close any Serial Monitor first.

Compare the voltage and current with a multimeter before trusting the SOH figures. They are uncalibrated.

## Safety limits

The system is **monitor-only**. The dashboard never writes to the ESP32, the firmware never reads from the USB port, and nothing in this project can disconnect a battery. **Your own fuse and physical disconnect are the protection.**

| Limit | Firmware behaviour | What you do |
|---|---|---|
| Above 26 V (the INA219's rating) | Withholds the reading and sends `# OVER 26 V` | Disconnect immediately |
| Above 3.2 A (the sensor's range) | Withholds the reading and sends a `#` message | Reduce the load |

Never connect this to an EV pack. A dashboard alert can take about 1.5 seconds to appear, so it is a warning and not a protection.

## Fault messages

| What you see | Cause | Action |
|---|---|---|
| `# INA219 not found` or `read failed` | SDA, SCL, VCC or GND wiring | Fix the wire. It reconnects by itself within about 5 seconds |
| Temperature blank, voltage still updating | DS18B20 wiring or the 4.7 kΩ pull-up | Test continues without temperature |
| "No reading for N s" | ESP32 silent: cable, board or wrong port | Check the USB cable and the port |
| "No readings yet" | Wrong port (often Bluetooth) or firmware not flashed | Choose the right port |
| Negative current under load | VIN+ and VIN− swapped | Swap them, or untick "Discharge current reads positive" |

## How SOC and SOH are calculated

- **SOC** uses Coulomb counting: `SOC(t) = SOC(t-1) − (Current(t) × Δt) / Total_Capacity`, with Δt in hours and a result in percent, kept between 0 and 100. Discharge current counts as positive.
- **SOH** uses `SOH = (Measured_Usable_Capacity / Original_Rated_Capacity) × 100`.

Usable capacity is only truly measured by discharging down to the cutoff voltage. Until then the live SOH carries a label saying how far it can be trusted, such as "at least this much". The decision uses the strongest evidence available: a full discharge in this test, then a capacity test recorded in the database, then nothing. With no evidence it says **Insufficient data**.

| SOH | Secondary use |
|---|---|
| Above 80% | Stationary Storage |
| 60% to 80% | Low-speed EVs |
| Below 60% | Recycling |

This is a capacity-only screening rule. Real reuse needs safety, resistance and cell-level testing by a qualified engineer.

## Data

`data/database/revive_tery.duckdb` holds 8 tables for one fictional pack, `DEMO-EV-PACK-001`: battery asset, modules, telemetry (12 rows), health history, charge sessions, safety events, materials and data sources. It is synthetic, which the data itself states. The dashboard flags places where it disagrees with itself, such as regenerative braking current having the same sign as driving.

Each test run adds readings to a `test_run_log` table in the database, labelled by source and data class. The source tables are never modified.

To rebuild the database from the original CSV files, put them in `data/csv_bundle/` and run `python src/build_relational_db.py`. The CSVs are not included here.

## Files

| File | Purpose |
|---|---|
| `src/app.py` | Circuit Studio dashboard (Dash and Cytoscape) |
| `src/bms_engine.py` | SOC and SOH maths, read-only ESP32 reader, DuckDB access and logging |
| `src/test_bench.py` | Virtual battery replay and the secondary-use rules |
| `src/esp32_sim.py` | The software ESP32 and circuit |
| `src/build_relational_db.py` | Builds the DuckDB database from the CSV files |
| `src/backend_connector.py` | Read-only queries for one battery |
| `firmware/revive_tery_esp32/` | The ESP32 sketch |

## Limitations

- The ESP32 path has been tested against a simulated port only. A first run on a real board may reveal wiring, driver or calibration issues.
- The database holds one battery, so battery switching reloads the same one.
- Faults are shown on the page and in the final result, but are not stored in DuckDB.
- Sensor readings are uncalibrated, and the simulated cell's curve, resistance and heating are simple assumptions.
- The dashboard has no login and listens on 127.0.0.1 only.

## License

MIT. See `LICENSE`.
