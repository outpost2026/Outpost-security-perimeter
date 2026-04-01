\# Firmware



This directory will contain the ESP32 firmware (Arduino / ESP‑IDF) for the security perimeter.



\*\*Currently empty.\*\* Planned content:



\- `outpost\_poc/` – Main source code with deep‑sleep, HX711 driver, camera control.

\- `config.h.template` – Template for Wi‑Fi credentials, weight thresholds, etc.

\- `README.md` – Compilation and flashing instructions.



\## Preliminary Design



The firmware implements a finite state machine:



