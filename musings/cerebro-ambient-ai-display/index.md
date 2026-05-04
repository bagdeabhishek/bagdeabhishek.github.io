---
layout: default
title: Cerebro — Ambient AI Display on ESP32-S3
permalink: /musings/cerebro-ambient-ai-display/
---

# Cerebro — Ambient AI Display on ESP32-S3

_AI-assisted research synthesis. Verify critical claims with primary sources._
Status: Completed
Last updated: 2026-05-01T20:00:00+05:30
Mode: explanatory

## Summary
- A push-based ambient AI display is **worth building** as a hobby project
- Best delivery: skill + MCP server (stdio transport, agent-agnostic)
- Any agent can use: Hermes, Claude Code, Codex, OpenCLAIDE, or direct terminal
- Architecture: Agent → MCP/CLI → Python daemon → SQLite + retry → HTTP POST → ESP32-S3
- Device is intentionally passive: no notifications, no touch, no scrolling, no dopamine loop
- 12 customer flows audited, 27 gaps identified, 4 must-fix before v0, 15 technical risks

## Overview
Cerebro is a calm, always-on, glanceable screen that answers one question: *"Do I need to check anything right now?"* Instead of repeatedly opening Gmail, Calendar, Slack, and news apps, an AI agent summarizes important items into a bounded daily brief and pushes it to a dedicated ESP32-S3 LCD screen. The device is passive by design — no notifications, no feed, no interactivity. It works with any AI agent via a standard MCP server, installed as a systemd daemon.

## Background
The problem is real: ADHD-friendly alternatives to compulsive phone checking are poorly served by existing products. Smart displays (Echo Show, Nest Hub) are cloud-dependent and designed for commerce, not peace. Dedicated info screens (DakBoard, Aura) lack AI pipelines. DIY ESP32 dashboards are user-configured, polling-based, and scroll endlessly. Cerebro fills a gap: AI decides what matters, pushes to a calm screen, and nothing else happens.

The target hardware is a Waveshare ESP32-S3 7-inch LCD board (800x480, RGB565, Wi-Fi/BLE, up to 16MB flash, 8MB PSRAM). The display stack is LVGL. The server side is Python: a FastAPI daemon with SQLite retry queue, HSDL validation, template-based layout engine, and an MCP server stdio interface.

## Core Analysis

### Architecture

```
Any AI Agent (Hermes / Claude Code / Codex / OpenCLAIDE)
   |
   +-- [via MCP / direct shell] -> cerebro-mcp (Python stdio MCP server)
   |                                  |
   |                                  +-- cerebro-daemon (FastAPI + retry worker)
   |                                  |       +-- SQLite (pending payloads queue)
   |                                  |       +-- template layout engine (5 templates)
   |                                  |       +-- [optional] Claude API layout generation
   |                                  |       +-- strict HSDL validator
   |                                  |       +-- HTTP POST to ESP32
   |                                  |
   |                                  +-- CLI modes:
   |                                        cerebro-mcp         -> MCP server mode (stdio)
   |                                        cerebro-mcp status  -> quick terminal check
   |                                        cerebro-mcp push    -> manual push
   |                                        cerebro-mcp init    -> first-time setup
```

### HSDL — Cerebro Screen Description Language
HSDL v0 is a restricted JSON layout format with exactly 3 primitives: label, line, rect. Full screen updates only (no partial push in v0). Max 30 objects, 16KB payload, coordinates within 800x480, colors as #RRGGBB, 5 Montserrat font sizes. Server validates before push; ESP32 re-validates before render.

### DisplayPayload Format
The semantic payload an agent produces before layout generation:

```json
{
  "timestamp": "Mon 28 Apr 07:42",
  "mode": "morning",
  "headline": "No urgent fires. Two things need attention.",
  "sections": [
    {"title": "TODAY", "items": ["10am standup", "3pm 1:1"], "priority": 1},
    {"title": "EMAIL", "items": ["HR: leave update"], "priority": 2}
  ],
  "footer": "Cerebro checked email, calendar, news and transcripts."
}
```

Priority rules: 1 = always visible, 2 = visible if space allows, 3 = optional.

### Template-Based Layout (v0)
5 deterministic templates: morning, afternoon, urgent, quiet, error. Each maps DisplayPayload to HSDL using fixed positioning. Display works without an LLM. Claude is an optional flavor layer later.

### Provisioning
First-time setup on bare ESP32:
1. Hardcode Wi-Fi in firmware (simplest for desk use)
2. If Wi-Fi fails, ESP32 falls back to soft AP mode with captive portal at 192.168.4.1
3. User connects phone to cerebro-XXXX, enters credentials, ESP32 saves and reboots
4. Daemon discovers ESP32 via hardcoded IP in .env (v0) or ESP32 announces itself (v1)

### Device Discovery

| Method | Viability |
|--------|-----------|
| Hardcoded IP + DHCP reservation | v0 default - simplest |
| ESP32 announces IP to daemon | v1 - best UX, requires daemon URL in firmware |
| mDNS | Avoid - ESP32 mDNS unreliable |
| LAN scan | Avoid - slow, fragile |

### Customer Flows (12 audited)

| Flow | Status |
|------|--------|
| A: First-time setup | Mostly covered (Gap A: PlatformIO docs) |
| B: Daily morning brief | Good |
| C: Afternoon refresh | Minor (mode diff, mid-glance swap) |
| D: Device reboot/recovery | Good |
| E: Agent/daemon failures | Gap F: stale-content nudge missing |
| F: Manual push from terminal | Gap G: must fix |
| G: Firmware brick recovery | Document recovery steps |
| H: Wi-Fi network change | Dual reconfig: ESP32 + daemon |
| I: Night mode / display dim | Gap K: must fix |
| J: Health check / status | Good |
| K: Multiple users | Out of scope v0 |
| L: First-time ESP32 user | Add docs |

### Gaps Summary (27 total, A-Z)

**Must-fix before v0:**
- Gap G: Manual push from terminal (cerebro-mcp push --payload)
- Gap F: Stale-content nudge after 24h inactivity
- Gap K: Backlight dimming or clear_display dark state
- Gap B: Welcome auto-push after first init

**High-risk tech gaps:**
- Gap N: Waveshare board variants differ — pin exact SKU
- Gap O: Flash size may exceed 8MB with 5 fonts
- Gap Z: Backlight may be I2C-controlled, not simple PWM GPIO

### Risk Table (15 risks)

| # | Risk | Level | Mitigation |
|---|------|-------|------------|
| R1 | Claude on critical path | High | Template layout first |
| R2 | HTTP + LVGL concurrency | Medium | Lightweight handler, dirty flag |
| R3 | First-time LVGL/ESP32 | Med-High | Run Waveshare demo first |
| R4 | HSDL validation duplication | Low | Correct design |
| R5 | Value vs smartphone | Medium | Physically separate screen |
| R6 | Hermes-only MCP → agent-agnostic | Resolved | Stdio MCP server |
| R7 | LAN-only security | Low | Bearer token, locked URL |
| R8 | Touch hardware unused | Medium | Single-user, acceptable |
| R9 | No scrolling beyond 800x480 | Low | 30-object limit, priority pruning |
| R10 | Board variant differences | High | Pin exact SKU |
| R11 | Flash size overflow | High | Measure binary, require 16MB |
| R12 | Backlight unknown | Medium | Verify board schematic |
| R13 | No persistent RTC | Medium | NTP every boot |
| R14 | Dual-core not pinned | Medium | Pin LVGL + WiFi to separate cores |
| R15 | Flash wear from writes | Low | Skip flash for unchanged content |

## Evidence and Sources

### Alternatives Landscape
- Smart displays (Echo Show, Nest Hub): cloud-dependent, designed for commerce, not pushable from custom AI
- Info screens (DakBoard, Aura): photo-frame, subscription-based, limited API, no AI curation
- DIY ambient displays (HA dashboards, ESP32 panels): polling-based, user-configured, infinite scroll

**Gap:** No existing product combines AI-decides-what-matters, push delivery, calm content, and notification-free UX.

### Hardware Feasibility
- Waveshare provides PlatformIO-based LVGL demo projects for their 7-inch boards
- 800x480 RGB565 framebuffer = ~768KB, PSRAM handles this comfortably
- ESP32-S3 as HTTP server + LVGL renderer is standard for this class
- Max payload 16KB is manageable for WiFi
- Risk areas: board variants, flash size with fonts, core affinity, backlight, power draw

### Cross-Agent Compatibility
- Hermes: native MCP client via config.yaml
- Claude Code: claude mcp add supports stdio MCP servers
- Codex: can subprocess the MCP server binary
- OpenCLAIDE: standard MCP client
- Terminal: cerebro-mcp with push or status flags
- No separate CLI wrapper needed -- MCP server binary does both

## Uncertainties and Competing Views

### High-Confidence Claims
- Technically feasible with standard ESP32-S3 + LVGL + Python
- Template-based layout better than LLM generation for v0
- MCP server + skill is the right delivery mechanism
- Push-based (ESP32 as HTTP server) is correct architecture for calm display

### Medium-Confidence Claims
- Value proposition holds for ADHD-friendly use; unproven for broader audience
- Waveshare display config will likely work without major issues
- Flash wear from repeated writes is not a real problem in practice

### Unresolved Questions
- Exact Waveshare SKU not pinned (changes pinout, PSRAM, backlight)
- Whether 5 Montserrat fonts fit within 8MB or need 16MB variant
- Whether backlight is PWM GPIO or I2C-controlled LED driver
- Dual-core stability under simultaneous HTTP + LVGL needs testing
- Whether behavioral change actually sticks without habit support

## Practical Takeaways

### Build Order
1. Daemon + fake device (Python-only, test push/retry/validation)
2. Real ESP32 firmware (PlatformIO, LVGL, HTTP server)
3. End-to-end integration (fake agent pushes to real ESP32)
4. MCP server + skill documentation
5. Real digest pipeline (Gmail + Calendar + RSS)

### Before Buying Hardware
- Pin exact Waveshare SKU (ESP32-S3-Touch-LCD-7 or ESP32-S3-LCD-7)
- Verify PSRAM is 8MB or 16MB variant
- Check if backlight is PWM GPIO or I2C
- Buy a 5V/2A USB-C charger

### Before Writing Firmware
- Run Waveshare official LVGL demo unchanged first (Milestone 0)
- Measure binary size after embedding Montserrat fonts
- Pin LVGL renderer to core 1, WiFi to core 0
- Skip flash writes for unchanged HSDL to avoid wear
- Add NTP sync on boot for timestamp context

### First Agent Setup

```
pip install cerebro
cerebro-mcp init
systemctl --user start cerebro-daemon
```

For Hermes: add to config.yaml under mcp_servers with stdio transport pointing to cerebro-mcp.
For Claude Code: claude mcp add cerebro -- cerebro-mcp

## References
1. [Original plan — Codex implementation brief](https://github.com/bagdeabhishek/cerebro) — community
2. PlatformIO Docs — primary
3. LVGL Documentation — primary
4. [Waveshare ESP32-S3 7-inch LCD](https://www.waveshare.com/esp32-s3-touch-lcd-7.htm) — primary hardware reference
5. ESP32-S3 Datasheet (Espressif) — primary
6. MCP Python SDK — primary
7. DeepSeek API Docs — secondary