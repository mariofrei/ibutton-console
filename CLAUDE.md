# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

iButton Console is a browser app for reading and programming Maxim/Analog Devices iButton temperature/humidity data loggers through a DS9490R USB reader over **WebUSB**. The whole app is the single file `ibutton-console.html` (inline CSS + one inline `<script>`), with no build step, package manager, dependencies, or tests. The only external resource is Google Fonts.

## Running

Open `ibutton-console.html` directly in Chrome or Edge (WebUSB is needed for real hardware). Other browsers work only in Demo mode. To check a change, open the page, turn on Demo mode, and try Read / Start / Stop / Clear, CSV open/export, and history. Real-hardware paths can only be verified with a DS9490R and a logger attached.

## Architecture (inside the `<script>` IIFE)

The script is split into sections marked `/* ---------- name ---------- */`. Global UI state lives in a single `state` object; `$` is `document.getElementById`.

**Device abstraction.** The UI only talks to `state.device`, which has `kind` (`"demo"`, `"usb"`, or file/history), `rom`, `model`, `family`, `spec`, `capacity`, and async `read()`, `start(cfg)`, `stop()`, `clear()`. `read()` returns `{ status, samples: [{i, t, v, h?}], mission, clock }`, where `status` is one of `idle|running|waiting|done|full`. Any new data source has to return this same shape, and `applyData(res)` renders it.

- **Demo:** `makeDemoDevice(spec, scen)` simulates a logger. `SPECS` holds per-model datasheet limits (range, resolution options with memory capacity, interval limits, `hasHum`, `noRollover`). `SCENARIOS` are application profiles: each has an `f(t, i, seed)` temperature function plus an optional `rh` humidity model.
- **Real hardware**, layered as follows:
  1. `DS2490` is the USB to 1-Wire bridge driver (vendor 0x04FA, product 0x2490). It handles control transfers on EP0, status on EP1 and bulk on EP2/EP3. `Reader` methods (`reset`, `block`, `search`, `select`, `byteSpu`) are serialized through `rd.lock()`.
  2. Shared 1-Wire memory helpers: `owReadMem` (Read Memory with per-page CRC16 and retries), `owScratch`/`owWrite`, `owCommand`. ROM IDs appear in two byte orders: display order (CRC first, family last) and bus order. Convert with `romToBus`/`busToRom`.
  3. Per-family drivers `DS21` (DS1921x), `DS41` (DS1922L/T/E, DS1923), `DS53` (DS1925, with XPC commands and 32-bit epoch timestamps), registered in `LOGGERS` by family code. Each implements `regs`, `info`, `read`, `start`, `stop`, `clear`. Register addresses and decoding come from the datasheets and are noted in comments next to the code.
  4. `makeUsbDevice` wraps a driver into the device interface. `findLogger` searches the bus and picks the first supported family.

**Rendering.** `renderAll()` runs `renderStats`, `renderChart` (hand-built SVG, with draggable alarm threshold lines), `renderTable` (capped at `ROW_CAP` rows) and `fitFonts`. Alarm thresholds come from the Mission setup form through `thr()` and are shared by the chart, stats, table and mission start.

**Persistence (browser only).** `localStorage` keys are prefixed `ib_` (`ib_fnName`, `ib_hist_fnName`, `ib_missionPresets`, `ib_fold`, `ib_mode`, `ib_fbTo`). Every access is wrapped in try/catch with an in-memory fallback. Read history goes to IndexedDB (`ibutton-console` DB, `reads` store); an explicit Read saves an entry, and the code falls back to a `Map` when IndexedDB is unavailable.

**Import/export.** `csvOf` writes one row per sample with the device/mission metadata repeated in columns. `loadCsv`/`parseCsv` read that format back, or any CSV that has timestamp and temperature columns. File names come from templates such as `{id}_{datetime:YYYYMMDD-HHmm}`. `makeZip` is a small built-in ZIP writer (CRC32) for exporting several files at once.

**WebMCP.** When `document.modelContext` exists (Chrome 146+ with `chrome://flags/#enable-webmcp-testing`, or the origin trial; `navigator.modelContext` is accepted for older polyfills), the WebMCP section registers agent tools with `registerTool`. The tools call the same functions the buttons use (`doRead`, `doStart`, `doStop`, `doClear`, `setDemo`, `openHistory`). `guarded` resolves to `""` on success or to the error message. Tools return a string, and a failure is returned as `"Error: ..."` text rather than thrown, because Chrome replaces a thrown error with a generic message. `start_mission` writes into the Mission setup form (`fillForm`) and then goes through the same limit checks as the Start button. Start, stop and clear carry `consequentialHint` and ask the user with `window.confirm`. When you add a button action, add a matching tool too.

## Conventions

- Keep everything in the one HTML file and add no libraries or build tooling.
- Match the existing style: dense code, often a whole function on one line; `"use strict"`; DOM built with `textContent` or escaped strings.
- UI and error messages are plain, user-facing English sentences that tell the user what to do (e.g. "Reseat it and try again."). Errors thrown inside actions reach the user through `guarded(name, fn)`.
- Hardware limits are enforced before writing to a logger (interval range per spec, low < high alarm, no rollover on DS1925). Password-protected family 41h loggers are refused rather than supported.
