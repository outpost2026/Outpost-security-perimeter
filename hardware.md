# Návrh hardwaru (Hardware Design)

Tento adresář bude obsahovat veškerou dokumentaci k hardwaru, schémata a diagramy zapojení pinů (pinout) poté, co bude prototyp úspěšně ověřen v terénu.

> [!IMPORTANT]
> **Stav:** Aktuálně prázdné. Dokumentace bude doplňována průběžně po validaci PoC (Proof of Concept).

### Plánovaný obsah:

* `schematics/` – Soubory Fritzing / KiCad pro váhovou podložku a propojení s ESP32.
* `pinout.md` – Detailní přiřazení GPIO pinů pro ESP32 (PIR, HX711, kamera, MOSFETy).
* `calibration.md` – Postup kalibrace tenzometrických snímačů krok za krokem.
* `enclosure.md` – Rozměry boxu s krytím IP67 a pokyny pro montáž.

---

## Předběžné technické informace

Aktuální verze PoC (Proof of Concept) využívá následující komponenty:

| Komponenta | Specifikace a role v systému |
| :--- | :--- |
| **ESP32 (Libre)** | Podpora `deep-sleep`, buzení externím přerušením (interrupt) z PIR senzoru. |
| **PIR senzor AM312** | Miniaturní detektor pohybu s velmi nízkým proudovým odběrem. |
| **HX711 + 4× 50 kg** | Tenzometrické snímače (load cells) zapojené pro měření celkové hmotnosti. |
| **ESP32-CAM (OV2640)** | Kamerový modul pro vizuální verifikaci, napájený selektivně přes MOSFET. |
| **MOSFET moduly (IRF520)** | Slouží ke spínání 5V větve pro HX711 a kameru (úspora energie). |
| **DC-DC měnič 24V→5V** | Hlavní napájecí modul pro celý systém. |

### Umístění a ochrana
Veškerá řídicí elektronika je umístěna v **suchém IP67 boxu** uvnitř objektu. Vnějšímu prostředí (vlivům počasí) je vystavena pouze samotná váhová podložka a PIR senzor.

---

*Tato sekce bude rozšířena po dokončení testování v terénu (předpokládaný termín: Q2 2026).*
