# PinForge — Wokwi-style STM32 (Blue Pill) simulator with an in-browser C workflow

Web app + small compile server. You write C in the browser, the server builds it with `arm-none-eabi-gcc`,
and the page runs the result on an emulated Cortex-M3 / STM32F103 wired to a virtual circuit.

```
public/index.html   the simulator (emulator, circuit canvas, editor) — one self-contained file
server.js           zero-dependency Node server: static files + POST /api/compile + GET /api/health
toolchain/          startup code, linker script and the stm32f1.h register header used for every build
Dockerfile          node:20-slim + gcc-arm-none-eabi
render.yaml         one-click blueprint for Render (free plan)
```

## Run locally
```
docker build -t pinforge . && docker run -p 3000:3000 pinforge     # then open http://localhost:3000
# or, without Docker: install arm-none-eabi-gcc, then
node server.js
```

## Deploy for free (Render)
1. Create a GitHub repository and push this folder to it.
2. On render.com: **New → Blueprint** (uses `render.yaml`) or **New → Web Service → Docker**, choose the **Free** plan.
3. Wait for the build. Your URL looks like `https://pinforge-xxxx.onrender.com`.

Free-tier notes (check current terms; they change): Render's free web service has 512 MB RAM and a small CPU share,
sleeps after about 15 minutes without traffic and takes roughly a minute to wake. Hugging Face Docker Spaces and
Fly.io no longer have a free option as of the sources checked in late 2026. If you only need the simulator
without the compile button, `public/index.html` also works on any static host (GitHub Pages, Netlify, Cloudflare Pages);
users then load firmware files built on their own machine.

## Using it
* Pick an example, edit `main.c`, press ▶ (or Ctrl+Enter). The page builds and runs it.
* Without a compile server the editor is read-only; use **Load firmware…** with a `.bin`, `.elf` or `.hex`.
* Build flags: `-mcpu=cortex-m3 -mthumb -Os|-O0|-O1|-O2 -Wall -Wextra -nostdlib -ffreestanding`, linked with
  `toolchain/startup.c` and `link.ld` (64 KB Flash at 0x08000000, 20 KB RAM at 0x20000000).
* Allowed includes: `<stdint.h>`, `<stdbool.h>`, `<stddef.h>`, `"stm32f1.h"`.

## Security notes (read before exposing publicly)
Compiling strangers' C code is risky. Protections in `server.js`: source size cap; the code is preprocessed first and the
files actually included are checked (only the toolchain headers are allowed); inline assembly is limited to a few
instructions (wfi, wfe, nop, sev, cpsid/cpsie i, dmb, dsb, isb, bkpt); `#pragma`, `##` and `.incbin`-style tricks are rejected;
the compiler runs with CPU/memory/file-size limits (`prlimit`), a timeout, in a throw-away temp directory, as a non-root user;
builds are rate-limited per IP and at most two run at once. This is defence in depth, not a full sandbox — keep secrets
out of the container and let the platform's container isolation be the final barrier.

## Emulator coverage
Cortex-M3 (Thumb/Thumb-2 incl. IT blocks, UDIV/SDIV, 64-bit multiply, LDRD/STRD, TBB/TBH, bitfields), exceptions/NVIC,
SysTick, GPIO A/B/C, RCC clocks (HSI, HSE, PLL → up to 72 MHz), EXTI/AFIO, TIM2–4 (update interrupt, PWM), USART1–3 (TX/RX),
ADC1 (software-triggered conversions). Not modelled yet: SPI, I2C, DMA, watchdogs, RTC, flash/power control registers
(so ST's HAL start-up code may hang), timer capture, USB, CAN.
