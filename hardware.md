\# Hardware Design



This directory will contain all hardware‑related documentation, schematics, and pinout diagrams after the prototype has been validated in the field.



\*\*Currently empty.\*\* Planned content:



\- `schematics/` – Fritzing / KiCad files for the load‑cell pad and ESP32 connections.

\- `pinout.md` – Detailed GPIO assignments for ESP32 (PIR, HX711, camera, MOSFETs).

\- `calibration.md` – Step‑by‑step weight calibration procedure.

\- `enclosure.md` – IP67 box dimensions, mounting instructions.



\## Preliminary Information



The current PoC uses:



\- \*\*ESP32 (Libre)\*\* – deep‑sleep, interrupt wake‑up from PIR.

\- \*\*AM312 PIR sensor\*\* – low‑current motion detector.

\- \*\*HX711 + 4× 50 kg load cells\*\* – weight measurement.

\- \*\*ESP32‑CAM (OV2640)\*\* – camera module, powered via MOSFET.

\- \*\*MOSFET modules (IRF520)\*\* – to switch 5V to HX711 and camera.

\- \*\*DC‑DC converter 24V→5V\*\* – system power supply.



All electronics are housed in a dry IP67 box inside the cottage; only the weight pad and PIR sensor are exposed.



\*This section will be expanded after field testing.\*

