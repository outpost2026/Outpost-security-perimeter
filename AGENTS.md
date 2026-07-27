# AGENTS.md — outpost_security_perimeter (Outpost IoT Hub)

## Scope
Off-grid IoT monitoring, security perimeter, and automation for Outpost 2026.
Covers: DS18B20 temp monitoring, Modbus telemetry, JK BMS, PIR+weight detection, GCP cloud pipeline.

## Directory structure
```
/                # README.md, LICENSE, .gitignore, AGENTS.md, .ai_state.json
/docs/           # Dokumentace (z Docs branch + analýza + plán)
/firmware/       # ESP32/ESP8266 firmware moduly
   ds18b20_monitor/    # T02: DS18B20 teplota heatsinku
   modbus_telemetry/   # T03: Modbus přes MAX3232
   security_perimeter/ # T01: PIR + weight sensor + deep sleep
   bms_monitor/        # T05: JK BMS přes JST PH TTL
   climate_station/    # T06: BMP180 teplota/tlak
/hardware/        # Schemata, BOM, pinouty
/cloud/           # GCP infrastruktura
/tests/           # Testy
/data/            # Vzorová data
/images/          # Fotky, schemata
```

## Task management
- Tento soubor je hlavní agentní konfigurace pro tento repozitář.
- Tasky (T01-T07) jsou definovány v README.md a .ai_state.json.
- Každý task musí mít binary MVP criterion.
- Po dokončení tasku: update .ai_state.json + commit.

## Import rules
- Importy z jiných repozitářů jsou definovány v revitalizacni_plan_v1.0.docx §5.
- Všechny importované artefakty patří do /docs/ nebo /hardware/.
- Source_raw/Outpost_kontext_master je RAG context source (není v tomto repu).
- IOT/ již neexistuje — jeho obsah je v historii nebo v KB.
- `docs/crop_IoT_monitoring.md` — IoT příručka pro polní kulturu (DS18B20, SHT30, BH1750, soil moisture, VPD, GCP pipeline).

## Communication
- Czech primary, English for code comments.
- Report progress in AGENTS.md after each phase milestone.
