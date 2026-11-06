# Choosing a laser altimeter (laser rangefinder) for a drone

A practical guide to picking, mounting and tuning a **single-point laser rangefinder as a UAV altimeter** for terrain following, precision landing, obstacle avoidance and survey AGL hold. Written from bench and flight work with ArduPilot and PX4; vendor numbers come from the [laser-rangefinder-module-database](https://github.com/Tommyapi/laser-rangefinder-module-database).

## 1. The range number needs a target and conditions

Every datasheet says "range: 100 m" or "3 km". That number was measured on a specific target (often a white 90 % reflective board, sometimes a building) under specific light. Over grass, dark soil, wet asphalt or water the same module reaches a fraction of it. Rules of thumb:

- Over natural ground (grass, soil, 10–30 % reflectivity) expect roughly **40–60 %** of the quoted white-board range.
- Full sunlight on the target costs another 10–30 % on 905 nm modules; 1535 nm modules suffer less because solar background is lower at that wavelength.
- Water, glass and polished metal can return nothing at a slant angle. Plan a fallback (barometer blend) for over-water flights.

Pick a module whose *worst-case* range over your surface is at least 1.5× your maximum intended AGL.

## 2. 905 nm vs 1535 nm

| | 905 nm | 1535 nm |
|---|---|---|
| Detector | silicon APD, cheap | InGaAs, more expensive |
| Eye safety at a given power | Class 1 needs low pulse energy, so range is limited | Eye-safe at much higher pulse energy, so long range in a small module |
| Typical range class | 10 m – 1.5 km | 1 km – 10+ km |
| Sunlight | more affected | less affected |
| Typical weight | 1–20 g | 10–100 g |

For a sub-250 g drone holding 1–40 m AGL, a 905 nm module under 10 g is the usual answer. For a mapping or inspection aircraft that must read ground at 300–3000 m, 1535 nm.

## 3. Update rate, latency and accuracy

- Terrain following at 15 m/s with a 10 Hz sensor gives a reading every 1.5 m of travel; 50–100 Hz is better for low, fast flight.
- Accuracy quoted as ±1 m matters less than *consistency*; a noisy sensor that jitters ±0.5 m at 2 m AGL will make a landing controller hunt. Check the datasheet for accuracy at short range, not only at max range.
- Beam divergence: 0.3 mrad means a 3 cm spot at 100 m (clean ground return); 3° (50 mrad) means a 5 m spot that averages bushes, poles and ground.

## 4. Interfaces

- **UART/TTL 3.3 V** is the common case: ArduPilot `SERIALx_PROTOCOL = 9` (Rangefinder) with the matching `RNGFND1_TYPE`; PX4 uses the driver for the module's protocol on a `SENS_` serial port.
- **I2C** is fine for short cables (< 30 cm) and one or two sensors; use UART for anything longer or noisy (ESC wiring).
- **CAN / DroneCAN** modules are the cleanest on larger aircraft.
- Some modules need a measurement-trigger command and some stream continuously; a streaming module at the wrong baud produces zero readings with no error. Verify the frame with a decoder ([lrf-uart-protocol-decoder](https://github.com/Tommyapi/lrf-uart-protocol-decoder)) before blaming the flight controller.

## 5. Mounting

- Point straight down, away from props and landing gear; never through the prop disc.
- Keep the optical window clear of the fuselage shadow line; a few millimetres of body in the beam edge returns a false 0.1 m reading.
- Soft-mount (foam tape or rubber grommets); vibration smears the pulse timing on some modules.
- Set `RNGFND1_POS_Z` (ArduPilot) or the equivalent offset so AGL is measured from the landing gear, not the sensor.
- Minimum range: most modules cannot read below 0.1–1 m. Set `RNGFND1_MIN` accordingly, otherwise EKF rejects landing-phase data.

## 6. ArduPilot quick setup

```
SERIAL2_PROTOCOL  9      # Rangefinder
SERIAL2_BAUD      115    # match the module
RNGFND1_TYPE      <driver id for the module protocol>
RNGFND1_MIN       0.5    # metres, check the datasheet
RNGFND1_MAX       <0.6 × white-board range>  # conservative
RNGFND1_ORIENT    25     # down
RNGFND1_GNDCLEAR  <cm from sensor to ground when landed>
EK3_SRC1_POSZ     1      # baro; use 2 (rangefinder) only for low, flat work
EK3_RNG_USE_HGT   70     # % of RNGFND1_MAX below which EKF uses the rangefinder for height
```

Then check `rangefinder1` in the Mission Planner status tab at 1 m, 5 m and max usable height before enabling terrain following (`TERRAIN_FOLLOW`) or `RTL_ALT` tweaks. A complete worked example with bench logs is in [erdi-lrf-pixhawk-ardupilot](https://github.com/Tommyapi/erdi-lrf-pixhawk-ardupilot).

## 7. PX4 quick setup

Enable the driver for the module's protocol on the serial port (`SENS_<DRIVER>_CFG = TELEM2`, for example), set `EKF2_RNG_A_HMAX` to a conservative max, `EKF2_RNG_A_VMAX` to the speed you want range-aided height to work at, and leave `EKF2_HGT_REF` on barometer unless you fly only low and flat. Watch `distance_sensor` in the MAVLink inspector.

## 8. Modules that reach at least 100 m, lightest first

Values from manufacturer pages; see the database for source links and capture dates.

| Brand | Model | Max range | Weight | Interface | Public price |
|---|---|---|---|---|---|
| ERDI LASER (ERDI TECH LTD) | [SPD1200N0](https://erdilrf.com/models/spd1200n0/) | 1500 m | 5 g | UART-TTL (customizable) | $35 |
| ERDI LASER (ERDI TECH LTD) | [SPD1200N2](https://erdilrf.com/models/spd1200n2/) | 1500 m | 5 g | UART-TTL (customizable) | $35 |
| ERDI LASER (ERDI TECH LTD) | [SPD1200N4](https://erdilrf.com/models/spd1200n4/) | 1500 m | 5 g | UART-TTL (customizable) | $35 |
| ERDI LASER (ERDI TECH LTD) | [SPD1200ZG](https://erdilrf.com/models/spd1200zg/) | 1200 m | 5 g | UART-TTL; customizable | $80 |
| ERDI LASER (ERDI TECH LTD) | [SPD1200Z2](https://erdilrf.com/models/spd1200z2/) | 1500 m | 6 g | UART-TTL, Customizable | $150 |
| JRT | [PTGL-12X-1](https://www.jrt-measure.com/905nm-laser-rangefinder-module/68788230.html) | 1200 m | 6 g | UART (TTL_3.3V) | $49 |
| JRT | [PTGL2-12X-1](https://www.jrt-measure.com/905nm-laser-rangefinder-module/69008485.html) | 1200 m | 6 g | UART (TTL 3.3V) | quote |
| ERDI LASER (ERDI TECH LTD) | [SPD1200N3](https://erdilrf.com/models/spd1200n3/) | 1000 m | 6 g | UART, 3.3 V TTL; customizable | $36 |
| LightWare | [SF20/C](https://lightwarelidar.com/shop/sf20-c/) | 100 m | 7.5 g | USB, Serial UART and I2C (3.3 V TTL, 5 V tolerant) | $319 |
| LightWare | [SF20/HA](https://lightwarelidar.com/shop/sf20-ha/) | 100 m | 7.5 g | USB, Serial UART and I2C (3.3 V TTL, 5 V tolerant) | $359 |
| ERDI LASER (ERDI TECH LTD) | [LR1500E2](https://erdilrf.com/models/lr1500e2/) | 1500 m | 10 g | UART-TTL | $40 |
| ERDI LASER (ERDI TECH LTD) | [LR1500E4](https://erdilrf.com/models/lr1500e4/) | 1500 m | 10 g | UART-TTL | $40 |
| ERDI LASER (ERDI TECH LTD) | [SPD1200S2](https://erdilrf.com/models/spd1200s2/) | 1500 m | 10 g | UART-TTL, Customizable | $150 |
| ERDI LASER (ERDI TECH LTD) | [LR1200E2](https://erdilrf.com/models/lr1200e2/) | 1200 m | 10 g | UART-TTL | $35 |
| ERDI LASER (ERDI TECH LTD) | [LR1200E4](https://erdilrf.com/models/lr1200e4/) | 1200 m | 10 g | UART-TTL | $35 |
| ERDI LASER (ERDI TECH LTD) | [LR1000E2](https://erdilrf.com/models/lr1000e2/) | 1000 m | 10 g | UART-TTL | $30 |
| ERDI LASER (ERDI TECH LTD) | [LR1000E4](https://erdilrf.com/models/lr1000e4/) | 1000 m | 10 g | UART-TTL | $30 |
| Benewake | [TFA170-L](https://en.benewake.com/TFA170L/index.html) | 170 m | 10 g | RS-232 | quote |
| Benewake | [TFA300 Series](https://en.benewake.com/TFA300/index.html) | 290 m | 10.5 g | UART, CAN | quote |
| LightWare | [GRF-500](https://lightwarelidar.com/shop/grf-500/) | 500 m | 10.7 g | USB, Serial UART and I2C (3.3 V TTL, 5 V tolerant) | $459 |
| LightWare | [GRF-250](https://lightwarelidar.com/shop/grf-250/) | 250 m | 10.7 g | USB, Serial UART and I2C (3.3 V TTL, 5 V tolerant) | $399 |
| Lumispot Tech | [DLRF-C2 2KM Module](https://www.lumispot-tech.com/2km-laser-rangefinder-module-3-product/) | 2000 m | 11 g | UART (TTL_3.3V) | quote |
| Lumispot Tech | [DLRF-C1 1.5KM Module](https://www.lumispot-tech.com/1-5km-laser-rangefinder-module-product/) | 1500 m | 11 g | UART (TTL_3.3V) | quote |
| Lumispot Tech | [DLRF-C1 1.2KM High Repetition Module](https://www.lumispot-tech.com/1-2km-high-repetition-laser-rangefinder-module-product/) | 1200 m | 11 g | UART (TTL_3.3V) | quote |
| ERDI LASER (ERDI TECH LTD) | [LRF3000A1](https://erdilrf.com/models/lrf3000a1/) | 3000 m | 12 g | UART, 3.3 V TTL | $80 |

## 9. Checklist before first flight

1. Module reads correctly on the bench at 1 m, 5 m, 20 m against the real surface (grass, not a wall).
2. Zero readings do not appear when the aircraft is tilted 20°.
3. Min and max parameters match the datasheet, not the marketing page.
4. Barometer remains the primary height source for anything above the rangefinder's usable range.
5. Logged `RFND` messages show no dropouts at cruise speed.

## Related

- [laser-rangefinder-module-database](https://github.com/Tommyapi/laser-rangefinder-module-database)
- [awesome-laser-rangefinder](https://github.com/Tommyapi/awesome-laser-rangefinder)
- [ArduPilot rangefinder setup](https://ardupilot.org/copter/docs/common-rangefinder-landingpage.html)
- [PX4 distance sensors](https://docs.px4.io/main/en/sensor/rangefinders.html)

## Support / 打赏

If this project saved you time, you can buy the author a coffee in crypto. Thank you!
如果这个项目帮到了你，欢迎用加密货币打赏支持，谢谢！

| Network | Asset | Address |
|---|---|---|
| TRON (TRC20) | USDT | `TNGdTwE6opH6rmUyoG1JUR7FSyEb4k273R` |
| Ethereum / BNB Chain / Polygon | USDT | `0x59c43144a862b351fa1086ae842d1327c9d91613` |
| Ethereum / Arbitrum / Base | ETH | `0x59c43144a862b351fa1086ae842d1327c9d91613` |
| Bitcoin | BTC | `1RS6tWXHregLfEsiPzVUCUBuhgDg5uTSV` |
| Solana | USDT | `HZguEXfTtLChNdJTxAYCbcGscaqikz8JYrEpUhEyeVd9` |

Please send only the listed asset on the listed network — other tokens or networks may be lost.
请按表中对应的网络转对应的币种，转错币种或网络可能无法找回。

Built by tommyapi.com

## License

MIT — see [LICENSE](LICENSE).
