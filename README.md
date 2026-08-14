# stm32-si5351

HAL-based Si5351 driver for STM32. For ESP32 platform there is a fork [osmanovv/esp32-si5351](https://github.com/osmanovv/esp32-si5351).

Si5351 is a I2C-programmable 8 kHz - 160 MHz clock generator made by Silicon Labs. It has 3 ports (or more depending on modification) with 50 Ohm output impedance. The signal level can be changed in ~2-11 dBm range and the phase shift between channels is configurable.

## Usage Guide

### Regular Mode

```c
const int32_t correction = 978;
si5351_Init(correction, SI5351_CRYSTAL_LOAD_10PF, false);

si5351PLLConfig_t pll_conf;
si5351OutputConfig_t out_conf;
int32_t Fclk = 7000000; // 7 MHz

// Calculate and apply PLL and MultiSynth divider configurations
si5351_Calc(Fclk, &pll_conf, &out_conf);
si5351_SetupPLL(SI5351_PLL_A, &pll_conf);

si5351_SetupChannel(0, SI5351_PLL_A, SI5351_DRIVE_STRENGTH_4MA, &out_conf, 0);
si5351_ResetPLL(SI5351_PLL_A);
si5351_EnableOutputs(1<<0);
```

### Routing an output directly to the crystal/XO

The library now supports routing `CLK0..CLK2` either to the multisynth (PLL/MS)
or directly to the crystal/XO. Routing directly to XO bypasses multisynth dividers.

```c
// Enable fanout
const int32_t correction = 978;
si5351_Init(correction, SI5351_CRYSTAL_LOAD_10PF, true);

// Route CLK0 to PLLA/MS, CLK1 directly to XO, CLK2 to PLLB/MS
si5351_SetupPLL(SI5351_PLL_A, &pll_conf);
si5351_SetupPLL(SI5351_PLL_B, &pll_conf_b);

si5351_SetupChannel(0, SI5351_PLL_A, SI5351_DRIVE_STRENGTH_4MA, &out_conf0, 0);
si5351_SetupChannelBypass(1, SI5351_DRIVE_STRENGTH_4MA);
si5351_SetupChannel(2, SI5351_PLL_B, SI5351_DRIVE_STRENGTH_4MA, &out_conf2, 0);

si5351_ResetPLL(SI5351_PLL_A);
si5351_ResetPLL(SI5351_PLL_B);
si5351_EnableOutputs((1<<0) | (1<<1) | (1<<2));
```

Notes:
- When routing an output directly to XO you cannot use multisynth dividers on that
  channel.

### I/Q-Mode

```c
const int32_t correction = 978;
si5351_Init(correction, SI5351_CRYSTAL_LOAD_10PF, false);

si5351PLLConfig_t pll_conf;
si5351OutputConfig_t out_conf;
int32_t Fclk = 7000000; // 7 MHz

si5351_CalcIQ(Fclk, &pll_conf, &out_conf);

/*
 * `phaseOffset` is a 7bit value, calculated from Fpll, Fclk and desired phase shift.
 * To get N° phase shift the value should be round( (N/360)*(4*Fpll/Fclk) )
 * Two channels should use the same PLL to make it work. There are other restrictions.
 * Please see AN619 for more details.
 *
 * si5351_CalcIQ() chooses PLL and MS parameters so that:
 *   Fclk in [1.4..100] MHz
 *   out_conf.div in [9..127]
 *   out_conf.num = 0
 *   out_conf.denom = 1
 *   Fpll = out_conf.div * Fclk.
 * This automatically gives 90° phase shift between two channels if you pass
 * 0 and out_conf.div as a phaseOffset for these channels.
 */
uint8_t phaseOffset = (uint8_t)out_conf.div;
si5351_SetupChannel(0, SI5351_PLL_A, SI5351_DRIVE_STRENGTH_4MA, &out_conf, 0);
si5351_SetupChannel(2, SI5351_PLL_A, SI5351_DRIVE_STRENGTH_4MA, &out_conf, phaseOffset);
si5351_SetupPLL(SI5351_PLL_A, &pll_conf);
si5351_ResetPLL(SI5351_PLL_A); // Reset PLL to establish lock and correct phase offset
si5351_EnableOutputs((1<<0) | (1<<2));
```

### Disabling a Channel

To disable a clock channel completely (powering it down and clearing its parameters to safe defaults):

```c
si5351_DisableChannel(0); // Disables CLK0 and clears its Multisynth parameters/phase offset registers
```

More comments are in the code. See also examples/ directory.

This library was forked from [ProjectsByJRP/si5351-stm32](https://github.com/ProjectsByJRP/si5351-stm32) which in it's turn is a port of [adafruit/Adafruit_Si5351_Library](https://github.com/adafruit/Adafruit_Si5351_Library). Both libraries are licensed under BSD.
